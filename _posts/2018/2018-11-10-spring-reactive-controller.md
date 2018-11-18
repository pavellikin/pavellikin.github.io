---
layout: post
title:  "Реактивный контроллер в Spring"
date:   2018-11-18 13:20:34 +0300
categories: rus java spring-boot open-api
---
В spring boot, начиная с версии 2.0, добавили возможность использовать реактивный подход для некоторых сценариев. Например, стало возможным использовать реактивные драйверы для некоторых баз данных и для REST контроллеров. В сети полно примеров, где показано как реализовать REST контроллер самому. Однако, бывают ситуации, когда контроллер и модель должны удовлетворять какому-либо контракту, например, описанному с помощью [open api](https://github.com/OAI/OpenAPI-Specification). В этой статье я покажу, как сгенерировать реактивный контроллер при помощи [openapi-generator-maven-plugin](https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin).

Сгененрируем скелет проекта с помощью [spring initializr](https://start.spring.io/). В своем проекте я использовал maven, java 11, spring boot 2.1.0 и зависимость на reactive web. В качестве дополнительной зависимости придется добавить swagger-annotations, чтобы компиляция сгенерированных классов не приводила к ошибке.
``` xml
<dependency>
    <groupId>io.swagger</groupId>
    <artifactId>swagger-annotations</artifactId>
    <version>1.5.21</version>
</dependency>
```

Подготовим описание простого REST итерфейса с помощью open api.
``` yaml
openapi: 3.0.1
info:
  description: 'Simple open api file'
  title: 'Simple open api file'
  version: 1.0.0
servers:
- url: http://localhost:8080/v1
paths:
  /hello:
    get:
      description: hello world!
      operationId: helloWorld
      responses:
        200:
          content:
            application/json:
              schema:
                type: string
          description: successful operation
``` 

А теперь настроим openapi-generator-maven-plugin.
``` xml
<plugin>
	<groupId>org.openapitools</groupId>
	<artifactId>openapi-generator-maven-plugin</artifactId>
	<version>3.3.2</version>
	<executions>
		<execution>
			<goals>
				<goal>generate</goal>
			</goals>
			<configuration>
				<inputSpec>${project.basedir}/src/main/resources/swagger/simple-open-api.yaml</inputSpec>
				<output>${generated.sources.basedir}</output>
				<generatorName>spring</generatorName>
				<apiPackage>${default.package}.controller</apiPackage>
				<ignoreFileOverride>${project.basedir}/.openapi-generator-ignore</ignoreFileOverride>
				<configOptions>
					<delegatePattern>true</delegatePattern>
					<reactive>true</reactive>
				</configOptions>
			</configuration>
		</execution>
	</executions>
</plugin>
```

Коротко опишу используемые ключи:
1. inputSpec - путь до yaml файла с описанием контракта.
2. output - директория, куда поместить генерируемые классы.
3. generatorName - имя генератора, с помощью которого будут генерироваться классы.
4. apiPackage - пути, по которым сгенерировать классы для API и модели соответсвенно.
5. ignoreFileOverride - путь до ignore файла, где можно указать, какие классы нужно исключить из кодогенерации.
6. configOptions.delegatePattern - использование класса делегата для API. Эта опция помогает скрыть swagger аннотации и сосредоточиться на логике.
7. configOptions.reactive - использовать реактивное API для возвращаемых результатов.

Теперь самое время показать наш контроллер, который будет реализовывать сгенерированный интерфейс.
``` java
@Controller
public class HelloController implements HelloApiDelegate {
	@Override
	public Mono<ResponseEntity<String>> helloWorld(ServerWebExchange exchange) {
		return Mono.just(
				ResponseEntity.ok("World!")
		);
	}
}
```

Вот и все! Полный код доступен по [ссылке](https://github.com/burnout171/spring-reactive-controller)
