---
layout: post
title:  "Динамическая конфигурация bean в spring"
date:   2018-02-04 16:00:34 +0300
categories: rus java spring-boot
---
Иногда необходимо, чтобы приложение имело возможность динамически инстанциировать бины опираясь на файл конфигурации. В частности, это может быть полезно, когда в конфигурационном файле у нас описаны настройки для разных версий клиентов, которые будут обращатся к внешним сервисам по одному и тому-же протоколу, но через разные URL.

Прежде всего объявим класс клиента. Он содержит имя, по которому можно будет отличить один клиент от другого, URL точки назначения и ссылку на бин RestTemplate, через который мы будем отправлять запросы. Класс имеет очень простую функцию `send()`, которая логирует имя текущего клиента и пытается получить объект путем вызова `restTemplateюgetForEntity()`.
```java
public class Client {

    private static final Logger log = LoggerFactory.getLogger(Client.class);

    private final String name;
    private final URI url;
    private final RestTemplate restTemplate;

    Client(final String name, final String url, final RestTemplate restTemplate) {
        this.name = name;
        this.url = URI.create(url);
        this.restTemplate = restTemplate;
    }

    public String send() {
        log.info(name + ".send");
        return restTemplate.getForEntity(url, String.class).getBody();
    }
}
```

Чтобы вклинится в процесс создания бинов в Spring, необходимо будет создать класс, реализующий интерфейс BeanDefinitionRegistryPostProcessor, который позволит регестировать свои BeanDefinition. Для такого простого класса как Client, который был объявлен выше, можно создать BeanDefinition работая напрямую с классом Client. Однако, в реальных ситуациях объект может содержать много зависимостей и содержать сложную логику по созданию экземпляра. Для решения такой проблемы можно воспользоватся классом ClientBuilder, который будет реализовывать Spring интерфейс FactoryBean.

Ниже представлен пример FactoryBean для класса Client. Он содержит ссылки на RestTemplate и объект ClientProperties, инкапсулирующий все свойства клиента.
```java
public class ClientBuilder implements FactoryBean<Client> {

    private ClientProperties properties;
    private RestTemplate restTemplate;

    @Override
    public Client getObject() throws Exception {
        return new Client(properties.getName(), properties.getUrl(), restTemplate);
    }

    @Override
    public Class<?> getObjectType() {
        return Client.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }

    public ClientProperties getProperties() {
        return properties;
    }

    public void setProperties(ClientProperties properties) {
        this.properties = properties;
    }

    public RestTemplate getRestTemplate() {
        return restTemplate;
    }

    public void setRestTemplate(final RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
}
```

Класс ClientBuilder реализует следующие методы интерфейса FactoryBean:
1. `isSingleton()` - возвращает true, если объект Singleton (должна существовать только 1 копия объекта), и false в противном случае.
2. `getObjectType()` - возвращает тип объекта, который должен быть построен в результате.
3. `getObject()` - содержит логику по созданию объекта.

Теперь разберем класс ClientsRegisterPostProcessor, который реализует BeanDefinitionRegistryPostProcessor. 
```java
@Configuration
public class ClientsRegisterPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware {

    private final Map<String, ClientProperties> clients = new HashMap<>();

    @Override
    public void postProcessBeanDefinitionRegistry(final BeanDefinitionRegistry registry) {
        for(Map.Entry<String, ClientProperties> entry: clients.entrySet()) {
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(ClientBuilder.class)
                    .addPropertyValue("properties", entry.getValue())
                    .addPropertyReference("restTemplate", "restTemplate");
            registry.registerBeanDefinition(entry.getKey(), builder.getBeanDefinition());
        }
    }

    @Override
    public void postProcessBeanFactory(final ConfigurableListableBeanFactory beanFactory) {
    }

    @Override
    public void setEnvironment(final Environment environment) {
        String allClients = environment.getProperty("clients.all");
        if (Objects.nonNull(allClients)) {
            String[] clientNames = allClients.split(",");
            fillClientProperties(environment, clientNames);
        }
    }

    private void fillClientProperties(final Environment environment, final String[] clientNames) {
        for (String name : clientNames) {
            String trimmed = name.trim();
            ClientProperties properties = new ClientProperties()
                    .setName(trimmed)
                    .setUrl(environment.getProperty(String.format("clients.%s.url", trimmed)));
            clients.put(trimmed, properties);
        }
    }
}
```

Необходимо отметить, что во время работы BeanDefinitionRegistryPostProcessor бины еще не построены, поэтому парсить файл конфигурации нам придется самим. Ниже приведен пример файла настроек для клиента. 
```yml
clients:
  all: alfa, beta
  alfa:
    url: http://127.0.0.1:8081/
  beta:
    url: http://127.0.0.1:8082/
```

Самый простой способ получить доступ к свойствам клиентов - добавить реализацию Spring интерфейса EnvironmentAware. Метод `setEnvironment()` получает копию ссылки на Environment, достает имена всех клиентов, которые нужно будет создать (содержится по самостоятельному пути в файле конфигурации), вычитывает свойства каждого клиента по фиксированным путям и сохраняет результат в Map.

Построение BeanDefinition происходит в методе `postProcessBeanDefinitionRegistry()` с помощью данных, которые мы сохранили в Map. Важно понимать, что ClientProperties добавляется в BeanDefinition через метод `addPropertyValue()` по ссылке на сохраненный объект, однако бины еще не построены, поэтому мы можем только сослаться на объект restTemplate, который будет создан позже, через метод `addPropertyReference()`.

Регистрация BeanDefinition происходит через объект BeanDefinitionRegistry, где в качестве имени используем имя клиента, а в качестве значения полученный BeanDefinition.

Файл для конфигурации RestTemplate показан ниже.
```java
@Configuration
public class DynamicBeansConfiguration {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

Чтобы получить цельное приложение, добавим класс ClientRouter, который будет получать на вход имя клиента и вызывать метод `send()` у конкретного Client. Ссылки на Client он будет хранить в Map.
```java
@Component
public class ClientRouter {

    private final Map<String, Client> clients;

    @Autowired
    public ClientRouter(Map<String, Client> clients) {
        this.clients = clients;
    }

    public String route(final String clientName) {
        Client client = clients.get(clientName);
        if (Objects.nonNull(client)) {
            return client.send();
        }
        return null;
    }
}
```

Полный код приложения доступен по [ссылке](https://github.com/burnout171/spring-boot-example).

Источники:
1. <https://dzone.com/articles/how-create-your-own-dynamic>
2. <https://spring.io/blog/2011/08/09/what-s-a-factorybean>
