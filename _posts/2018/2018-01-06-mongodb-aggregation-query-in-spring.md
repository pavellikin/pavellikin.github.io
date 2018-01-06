---
layout: post
title:  "Агрегированные запросы в MongoDb через spring"
date:   2018-01-06 17:00:34 +0300
categories: rus java spring-boot mongoDb
---
Часто при работе с базой данных (БД) нам нужно получить данные, которые были обработаны и представлены так, как того требует логика программы. В большинстве случаев при работе с spring можно обойтись JPA репозиториями. Однако, если нам необходимо получить результат группировки, то JPA репозитории нам не подойдут, так как они не предоставляют такой [операции][1]. В простейшем случае можно обойти эту проблему проаннотировав необходимый метод репозитория аннотацией @Query, в остальных приходится писать запрос в БД вручную. В этой статье мы попробуем составить несколько таких запросов в MongoDb.

MongoDb позволяет использовать aggregation операции для вычислений в БД, а spring-data-mongodb, который является частью spring-boot-starter-data-mongodb, предоставляет абстракцию Aggregation для построения таких запросов в коде. Aggregation состоит из AggregationOperation, которые формируют конвеер обработки (pipeline), и представляют из себя операции, которые должны быть выполнены с данными на текущем шаге. Порядок операций определяется логикой вашего приложения. Результатом вычислений является AggregationResults.

Остановимся подробнее на некоторых AggregationOperation, которые будут использованы в дальнейшем:
1. MatchOperation. Фильтр для данных, которые будут выбраны. Принимает на вход Criteria.
2. GroupOperation. Операция группировки по какому-либо полю. Результатом является json вида `{"_id": "result"}`, где result это содержимое поля, по которому происходила группировка. Содержит очень много вспомогательных методов для дополнительного преобразования данных:
    * first - выбор содержимого поля, являющегося первым в списке группировки.
    * last - выбор содержимого поля, являющегося последним в списке группировки.
    * min - выбор минимального значения для поля.
    * max - выбор максимального значения для поля.
    * count - посчитать количество записей.
Вызов каждого из вспомогательных методов модифицирует результирующий json, дополняя его используемыми полями.
3. ProjectionOperation. Операция формирования представления полученного результата. С помощью нее можно указать какие поля будут использоваться в итоговом отображении.
4. SortOperation. Сортирует полученный результат.

Попробуем применить данные выше на практике. Создадим объект TextDocument, который будет контейнером для данных, хранимых в MongoDb. Для генерации сеттеров, геттеров и конструктора будем использовать lombok.
``` java
@Document(collection = "documents")
@Data
public class TextDocument {
    @Id
    private String id;
    private String name;
    private Integer version;
    private String text;
}
```

В качестве первого примера попробуем посчитать сколько документов хранится у нас в базе. Для представления результата заведем еще один объект DocumentCounter.
``` java
@Data
public class DocumentCounter {
    @JsonIgnore
    private String id;
    private Integer total;
}
```

Теперь заведем класс MongoAggregationDao, который будет формировать запрос в базу и делегировать его выполнение MongoTemplate.
``` java
@Component
public class MongoAggregationDao {

    private final MongoTemplate mongoTemplate;

    @Autowired
    public MongoAggregationDao(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public DocumentCounter getDocumentsNumberByName(final String name) {
        TypedAggregation<TextDocument> request = countDocumentsByName(name);
        return mongoTemplate.aggregateStream(request, DocumentCounter.class).next();
    }

    private TypedAggregation<TextDocument> countDocumentsByName(final String name) {
        MatchOperation match = Aggregation.match(Criteria.where("name").is(name));
        GroupOperation group = Aggregation.group("name").count().as("total");
        return Aggregation.newAggregation(TextDocument.class, match, group);
    }
}
```

Разберем подробнее запрос. MatchOperation показывает, что необходимо отобрать все поля "name" у которых значение равно переменной name. Далее используется очень простая GroupOperation, которая группирует данные по столбцу "name", считает количество сгруппированных данных и сохраняет полученный результат в столбце "total". Кроме того, был использован класс TypedAggregation, который содержит информацию о запрашиваемом типе объекта.

Для запроса в базу использовался метод `mongoTemplate.aggregateStream`, который позволяет получить результат агрегирующего запроса в MongoDb начиная с версии 3.6. Данный метод был добавлен в spring-data-mongodb начиная с версии 2.0.2. Для более поздних версий MongoDb достаточно использовать `mongoTemplate.aggregate`

Для того, чтобы протестировать запрос, добавим в приложение REST интерфейс.
``` java
@RestController
public class MongoAggregationRestController {

    private final MongoAggregationDao dao;

    @Autowired
    public MongoAggregationRestController(final MongoAggregationDao dao) {
        this.dao = dao;
    }

    @RequestMapping("getDocumentsNumber")
    public DocumentCounter getDocumentsCountByName(@RequestParam(value = "name") final String name) {
        return dao.getDocumentsNumberByName(name);
    }
}
``` 

Настроим параметры сервера в конфигурационном yml файле.
```
spring:
  name: springbootexample
  data:
    mongodb:
      host: 127.0.0.1
      port: 27017
      database: home
server:
  port: 8055
  servlet:
    context-path: /springbootexample
```

Пошлем REST запрос `curl -X GET 'http://127.0.0.1:8055/springbootexample/getDocumentsNumber?name=test'` и получим ответ вида:
``` json
{
    "total": 2
}
```

Теперь давайте попробуем выбрать документ максимальной версии из базы. Модифицируем MongoAggregationDao класс.
``` java
@Component
public class MongoAggregationDao {

    private final MongoTemplate mongoTemplate;

    @Autowired
    public MongoAggregationDao(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public TextDocument getLastDocumentByName(final String name) {
        TypedAggregation<TextDocument> request = lastDocumentByNameRequest(name);
        return mongoTemplate.aggregateStream(request, TextDocument.class).next();
    }

    private TypedAggregation<TextDocument> lastDocumentByNameRequest(final String name) {
        MatchOperation match = Aggregation.match(Criteria.where("name").is(name));
        SortOperation sort = Aggregation.sort(Sort.Direction.DESC, "version");
        GroupOperation group = Aggregation.group("name")
                .last("id").as("id")
                .last("name").as("name")
                .max("version").as("version")
                .last("text").as("text");
        ProjectionOperation projection = Aggregation.project("name", "version", "text")
                .andExpression("id").as("_id");
        return Aggregation.newAggregation(TextDocument.class, match, sort, group, projection);
    }
}
```

Разберем получившийся запрос. Формируем MatchOperation такой же, как и в примере с подсчетом документов, далее сортируем выборку по убыванию по полю "version" с помощью SortOperation. Группируем с помощью GroupOperation по полю "name", далее берем все значения полей у первой записи (запись с максимальной версией) с помощью метода first. Версию получим с помощью метода max, чтобы проиллюстрировать то, что нам нужен документ с максимальной версией, однако в данном примере его можно заменить на first. Сформируем итоговое отображение с помощью ProjectionOperation, включим в него поля "name", "version", "text", а в столбец "\_id", который получился после операции группировки, запишем значение из столбца "id", чтобы сохранить оригинальный id из БД.

Добавим еще один REST интерфейс.
``` java
@RestController
public class MongoAggregationRestController {

    private final MongoAggregationDao dao;

    @Autowired
    public MongoAggregationRestController(final MongoAggregationDao dao) {
        this.dao = dao;
    }

    @RequestMapping("getLastDocument")
    public TextDocument getLastDocumentByName(@RequestParam(value = "name") final String name) {
        return dao.getLastDocumentByName(name);
    }
}
```

Сформируем новый REST запрос `curl -X GET 'http://127.0.0.1:8055/springbootexample/getLastDocument?name=test'` и получим ожидаемый ответ:
``` json
{
    "id": "5a4e313ac151d0a6c692a4c5",
    "name": "test",
    "version": 2,
    "text": "another text"
}
```
Полный код приложения доступен по [ссылке](https://github.com/burnout171/spring-boot-example).

Источники:
1. <https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongo.aggregation>
2. <http://www.baeldung.com/spring-data-mongodb-projections-aggregations>
3. <https://www.mkyong.com/mongodb/spring-data-mongodb-aggregation-grouping-example/>

[1]: <https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation>