# MongoDB

## Содержание

1. [Теория и Архитектура](#1-теория-и-архитектура)
2. [Основные Концепции](#2-основные-концепции)
3. [Работа с Document](#3-работа-с-document)
4. [CRUD Операции](#4-crud-операции)
5. [Операторы Обновления](#5-операторы-обновления)
6. [Индексы и Производительность](#6-индексы-и-производительность)
7. [Best Practices для Production](#7-best-practices-для-production)

---

## 1. Теория и Архитектура

### Что такое MongoDB?

MongoDB — документоориентированная NoSQL база данных, хранящая данные в формате **BSON** (Binary JSON).

### Ключевые особенности:

- **Схема-независимая** (schema-less) — документы в одной коллекции могут иметь разную структуру
- **Горизонтальное масштабирование** через шардирование
- **Высокая доступность** через реплика-сеты
- **Гибкие запросы** — поддержка вложенных структур, массивов, агрегаций

### Архитектурные уровни:

| Уровень | Аналог в RDBMS | Описание |
|---------|---------------|----------|
| Database | Database | Изолированное пространство данных |
| Collection | Table | Группа документов |
| Document | Row | Единица хранения данных (BSON) |
| Field | Column | Отдельное поле в документе |

### MongoDB Java Driver (современный):

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>5.2.0</version>
</dependency>
```

Или для реактивного подхода:

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-reactivestreams</artifactId>
    <version>5.2.0</version>
</dependency>
```

---

## 2. Основные Концепции

### BSON (Binary JSON)

- Бинарное представление JSON-документов
- Поддерживает больше типов данных чем JSON (Date, ObjectId, Binary и др.)
- Эффективное хранение и быстрый парсинг

### Document

Основной класс для работы с данными:

- Реализует интерфейс `Map<String, Object>`
- Представляет собой структуру «ключ-значение»
- Поддерживает вложенные документы и массивы

### Collection

Группа документов, получаемая из Database:

- Аналог таблицы в реляционных БД
- Не требует фиксированной схемы
- Документы могут иметь разную структуру

### Connection String (URI):

```url
mongodb://username:password@host:port/database?options
```

Пример подключения:

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString(
        "mongodb://localhost:27017/mydb"))
    .applyToConnectionPoolSettings(builder -> 
        builder.maxSize(50).minSize(10))
    .applyToSocketSettings(builder -> 
        builder.connectTimeoutMS(5000))
    .build();

MongoClient mongoClient = MongoClients.create(settings);
```

---

## 3. Работа с Document

### Создание Document

#### Простой документ:

```java
Document doc = new Document();
doc.put("name", "Alex");
doc.put("age", 30);
doc.put("email", "alex@example.com");
```

#### Через цепочку методов (append):

```java
Document doc = new Document("name", "Alex")
    .append("age", 30)
    .append("skills", Arrays.asList("Java", "MongoDB", "Spring"))
    .append("active", true);
```

#### Вложенные документы:

```java
Document address = new Document("city", "Moscow")
    .append("street", "Tverskaya")
    .append("building", "15");

Document user = new Document("name", "Alex")
    .append("address", address);
```

#### Документ с массивом объектов:

```java
Document order = new Document("orderId", "ORD-001")
    .append("items", Arrays.asList(
        new Document("product", "Laptop").append("qty", 1),
        new Document("product", "Mouse").append("qty", 2)
    ));
```

### Чтение данных из Document

#### Получение полей:

```java
String name = doc.getString("name");
Integer age = doc.getInteger("age");
Boolean active = doc.getBoolean("active");
```

#### Получение вложенных документов:

```java
Document address = doc.get("address", Document.class);
String city = address.getString("city");
```

#### Получение массивов:

```java
List<String> skills = doc.getList("skills", String.class);
```

#### Итерация по полям:

```java
for (Map.Entry<String, Object> entry : doc.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

### Преобразование Document

#### В JSON строку:

```java
String json = doc.toJson();
```

#### С форматированием:

```java
JsonWriterSettings settings = JsonWriterSettings.builder()
    .indent(true)
    .indentCharacters("  ")
    .build();
String prettyJson = doc.toJson(settings);
```

#### Из JSON строки:

```java
Document doc = Document.parse(jsonString);
```

### Важное предупреждение о мутациях:

Полученный из БД Document — это КОПИЯ данных в памяти.
Изменения в этом объекте НЕ сохраняются автоматически в базу.

```java
// Получили документ из БД
Document user = collection.find(eq("_id", id)).first();

// Изменили в памяти (это НЕ сохранится в БД!)
user.put("age", 35);

// Для сохранения нужно явно вызвать update:
collection.updateOne(
    eq("_id", id),
    new Document("$set", new Document("age", 35))
);
```

---

## 4. CRUD Операции

### Create (Вставка)

#### Вставка одного документа:

```java
Document user = new Document("name", "Alex")
    .append("age", 30);

collection.insertOne(user);

// После вставки в документ добавляется _id
ObjectId id = user.getObjectId("_id"); 
// ObjectId - это 12-байтовый уникальный идентификатор, который автоматически генерируется MongoDB для поля _id, если вы не указали своё значение.
```

#### Вставка нескольких документов:

```java
List<Document> users = Arrays.asList(
    new Document("name", "Alex").append("age", 30),
    new Document("name", "Bob").append("age", 25),
    new Document("name", "Carol").append("age", 35)
);
collection.insertMany(users);
```

### Read (Чтение)

#### Найти все документы:

```java
/*
FindIterable - это специальный интерфейс из MongoDB Java Driver, который представляет результат поиска в виде итерируемой коллекции.
FindIterable можно представить как "ленивую" (lazy) коллекцию результатов запроса. Она не загружает все данные сразу в память, 
а подгружает их по мере необходимости.
*/

FindIterable<Document> results = collection.find();
```

#### Пример с FindIterable:

```java
public class FindIterableExample {
    public static void main(String[] args) {
        // MongoCollection<Document> - это интерфейс из MongoDB Java Driver, который представляет собой коллекцию в базе данных MongoDB.
        MongoCollection<Document> collection = ...; 
        
        // Создаем запрос с настройками
        FindIterable<Document> results = collection
            .find(new Document("age", new Document("$gte", 18)))  // взрослые
            .sort(new Document("name", 1))                         // по алфавиту
            .limit(100)                                            // первые 100
            .projection(new Document("name", 1)                    // только имя
                .append("age", 1)                                   // и возраст
                .append("_id", 0));                                 // без _id
        
        // Обрабатываем результаты
        System.out.println("Первые 100 совершеннолетних пользователей:");
        for (Document doc : results) {
            System.out.printf("%s (%d лет)%n", 
                doc.getString("name"), 
                doc.getInteger("age"));
        }
        
        // Считаем количество (если нужно)
        long count = collection.countDocuments(new Document("age", new Document("$gte", 18)));
        System.out.println("Всего совершеннолетних: " + count);
    }
}
```

Важные особенности FindIterable:

1. "Ленивость" (Lazy Evaluation)

```java
FindIterable<Document> results = collection.find();
// На этом этапе запрос к БД еще НЕ выполнен!

for (Document doc : results) {
// Только здесь начинается выполнение запроса
// Документы подгружаются по мере необходимости
}
```
2. Одноразовость
```java
FindIterable<Document> results = collection.find();

// Первый проход - работает
for (Document doc : results) {
    System.out.println(doc);
}

// Второй проход по тому же объекту - НЕ РАБОТАЕТ!
for (Document doc : results) {  // Не вернет ничего!
    System.out.println(doc);  // Этот код не выполнится
}

// Нужно создать новый запрос
FindIterable<Document> newResults = collection.find();
```

3. Настройки курсора
```java
FindIterable<Document> results = collection.find()
   .cursorType(CursorType.Tailable)   // для "хвостовых" курсоров
   .noCursorTimeout(true)             // курсор не закрывается по таймауту
   .batchSize(100);                   // размер пакета загрузки
```

#### Найти с фильтром:

```java
/*
Bson - это интерфейс в MongoDB Java Driver, который представляет любую структуру данных в формате BSON (Binary JSON).

Bson (интерфейс)
├── Document (класс)
├── BsonDocument (класс)
├── BsonArray (класс)
├── Классы-фильтры (eq, gt, and...)
└── Классы-обновления ($set, $push...)
*/

Bson filter = eq("age", 30);
Document user = collection.find(filter).first();
```

#### Сложные фильтры:

```java
Bson filter = and(
    eq("active", true),
    gt("age", 25),
    lt("age", 40),
    in("city", Arrays.asList("Moscow", "SPb"))
);
```

#### Сортировка:

```java
collection.find()
    .sort(new Document("age", 1));  // 1 = возрастание

collection.find()
    .sort(new Document("age", -1)); // -1 = убывание
```

#### Ограничение и пропуск:

```java
collection.find()
    .skip(10)
    .limit(20);
```

#### Выбор конкретных полей (проекция):

```java
Bson projection = fields(
    include("name", "email"),
    exclude("_id")
);

collection.find().projection(projection);
```

### Update (Обновление)

#### Обновление одного документа:

```java
Bson filter = eq("_id", id);
Bson update = new Document("$set", 
    new Document("age", 35)
    .append("updated", new Date())
);
collection.updateOne(filter, update);
```

#### Обновление нескольких документов:

```java
Bson filter = gt("age", 18);
Bson update = new Document("$set", new Document("active", true));
collection.updateMany(filter, update);
```

#### Обновление с созданием если не существует:

```java
Bson filter = eq("email", "test@example.com");
Bson update = new Document("$set", 
    new Document("name", "Test")
    .append("createdAt", new Date())
);

UpdateOptions options = new UpdateOptions().upsert(true);
collection.updateOne(filter, update, options);
```

#### Заменить документ полностью:

```java
Bson filter = eq("_id", id);
Document replacement = new Document("name", "New Name")
    .append("age", 40);
collection.replaceOne(filter, replacement);
```

### Delete (Удаление)

#### Удаление одного документа:

```java
Bson filter = eq("_id", id);
collection.deleteOne(filter);
```

#### Удаление нескольких документов:

```java
Bson filter = eq("active", false);
collection.deleteMany(filter);
```

#### Удаление всех документов:

```java
collection.deleteMany(new Document());
```

---

## 5. Операторы Обновления

### Операторы полей

#### `$set` — установить значение:

```java
new Document("$set", new Document("age", 35))
```

#### `$unset` — удалить поле:

```java
new Document("$unset", new Document("inactiveField", ""))
```

#### `$rename` — переименовать поле:

```java
new Document("$rename", new Document("oldName", "newName"))
```

#### `$currentDate` — установить текущую дату:

```java
new Document("$currentDate", new Document("lastModified", true))
```

### Операторы числовых значений

#### `$inc` — инкремент:

```java
new Document("$inc", new Document("views", 1))
new Document("$inc", new Document("balance", -100)) // Прибавляется указанное число (может быть положительным или отрицательным)
```

#### `$mul` — умножение:

```java
new Document("$mul", new Document("price", 1.1))
```

#### `$min` / `$max` — мин/макс значение:

**Работает как `Math.min()` и `Math.max()`**

Эти операторы сравнивают текущее значение поля с указанным числом и обновляют поле только если указанное число соответствует условию (меньше для `$min`, больше для `$max`).

```java
// Оператор $min установит lowScore в значение 50, ЕСЛИ текущее lowScore БОЛЬШЕ 50        
new Document("$min", new Document("lowScore", 50))
        
// Оператор $max установит highScore в значение 100, ЕСЛИ текущее highScore МЕНЬШЕ 100        
new Document("$max", new Document("highScore", 100))
```

### Операторы массивов

#### `$push` — добавить элемент в массив:

```java
new Document("$push", new Document("skills", "Python"))
```

#### `$addToSet` — добавить если не существует:

```java
new Document("$addToSet", new Document("tags", "java"))
```

#### `$pop` — удалить первый/последний элемент:

```java
new Document("$pop", new Document("items", 1))   // последний
new Document("$pop", new Document("items", -1))  // первый
```

#### `$pull` — удалить по значению:

```java
new Document("$pull", new Document("skills", "COBOL"))
```

#### `$pullAll` — удалить несколько значений:

```java
new Document("$pullAll", new Document("oldTags", 
    Arrays.asList("deprecated", "legacy")))
```

#### `$each` — добавить несколько элементов:

```java
new Document("$push", new Document("scores", 
    new Document("$each", Arrays.asList(90, 85, 95))))
```

#### `$slice` — ограничить размер массива:

```java
new Document("$push", new Document("logs", 
    new Document("$each", Arrays.asList(newLog))
        .append("$slice", -10)))  // держать только 10 последних
```

#### `$position` — вставить в конкретную позицию:

```java
new Document("$push", new Document("items", 
    new Document("$each", Arrays.asList("new"))
        .append("$position", 0)))  // в начало
```

### Позиционные операторы

#### `$` — обновить первый найденный элемент массива:

```java
Bson filter = eq("items.name", "Laptop");
Bson update = new Document("$set", 
    new Document("items.$.price", 999));
```

#### `$[]` — обновить все элементы массива:

```java
Bson update = new Document("$set", 
    new Document("items.$.price", 999));
```

#### `$[<identifier>]` — обновить по условию:

```java
Bson filter = eq("items.name", "Laptop");
Bson update = new Document("$set", 
    new Document("items.$[elem].price", 999));
UpdateOptions options = new UpdateOptions();
options.arrayFilters(Arrays.asList(
    new Document("elem.name", "Laptop")
));
```

---

## 6. Индексы и Производительность

### Создание индексов

#### Простой индекс:

```java
collection.createIndex(new Document("email", 1));
```

#### Составной индекс:

```java
collection.createIndex(new Document("lastName", 1)
    .append("firstName", 1));
```

#### Уникальный индекс:

```java
// Уникальный индекс - это запрет дубликатов в конкретном поле (как UNIQUE в SQL)
collection.createIndex(new Document("email", 1), 
    new IndexOptions().unique(true));
```

Важно помнить:

- Уникальный индекс = защита от дубликатов на уровне БД
- Код ошибки дубликата - 11000 (нужно обрабатывать в коде)
- Нельзя создать, если уже есть дубликаты
- Работает и при вставке, и при обновлении

#### Индекс с TTL (автоудаление):

```java
collection.createIndex(new Document("expiresAt", 1),
    new IndexOptions().expireAfter(3600));  // 1 час
```

#### Текстовый индекс:

```java
collection.createIndex(new Document("content", "text"));
```

Текстовый индекс = встроенный поисковик в MongoDB, который позволяет:

- Искать слова в тексте

```java
// Найти документы со словами "mongodb" или "database"
collection.find(Filters.text("mongodb database"));
```
- Ранжировать результаты по релевантности

```java
// Найти и отсортировать по релевантности
collection.find(Filters.text("mongodb"))
    .sort(Sorts.metaDataScore("score"))
    .projection(Projections.metaDataScore("score"));
```

- Понимать разные формы слов

```java
// Найдет "run", "running", "ran" автоматически
collection.find(Filters.text("run"));
// MongoDB сама понимает разные формы слов
```
- Исключать ненужные слова

```java
// Найти "mongodb", но исключить "mysql"
collection.find(Filters.text("mongodb -mysql"));
```

- Искать точные фразы
```java
// Найти точную фразу "mongodb tutorial"
collection.find(Filters.text("\"mongodb tutorial\"")); 
```

### Направление индекса

| Значение | Направление | Использование |
|----------|-------------|---------------|
| 1 | Возрастание (A→Z, 0→9) | Сортировка по возрастанию |
| -1 | Убывание (Z→A, 9→0) | Сортировка по убыванию |

Важно:
- Для поиска по равенству (`eq`) направление не важно
- Для сортировки (`sort`) направление критично
- В составном индексе порядок полей важен

### Проверка использования индекса:

```java
Explainable explainable = Explainable.builder()
    .verbosity(ExplainVerbosity.EXECUTION_STATS)
    .build();

ExplainableCollection explainCollection = 
    explainable.wrapCollection(collection);

explainCollection.find(eq("age", 30)).explain();
```

Вернет:

```json
{
  "queryPlanner": {           // План выполнения запроса
    "winningPlan": {          // Выбранный план
      "stage": "IXSCAN",      // IXSCAN = использовался индекс!
      "indexName": "age_1"    // Какой индекс использован
    }
  },
  "executionStats": {         // Статистика выполнения
    "totalDocsExamined": 10,  // Сколько документов просмотрено
    "totalKeysExamined": 10,  // Сколько ключей индекса просмотрено
    "executionTimeMillis": 2  // Время выполнения (мс)
  }
}
```

### Best Practices для индексов:

- Создавайте индексы для часто используемых фильтров
- Избегайте избыточных индексов (замедляют запись)
- Используйте составные индексы для сложных запросов
- Мониторьте размер индексов и эффективность

---

## 7. Best Practices для Production

### Подключение и Pooling

#### Настройка пула соединений:

```java
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString("mongodb://host1:27017,host2:27017,host3:27017/mydb"))
    .applyToConnectionPoolSettings(builder -> 
        builder.maxSize(100)
            .minSize(10)
            .maxWaitTimeMS(5000))
    .applyToSocketSettings(builder -> 
        builder.connectTimeoutMS(5000)
            .readTimeoutMS(10000))
    .applyToServerSelectionSettings(builder -> 
        builder.serverSelectionTimeoutMS(5000))
    .applyToRetrySettings(builder -> 
        builder.maxRetries(3))
    .build();
```

### Обработка ошибок

#### Try-catch для операций:

```java
try {
    collection.insertOne(document);
} catch (MongoWriteException e) {
    // Ошибка записи (дубликат, валидация)
    logger.error("Write error", e);
} catch (MongoCommandException e) {
    // Ошибка команды БД
    logger.error("Command error", e);
} catch (MongoException e) {
    // Общая ошибка MongoDB
    logger.error("Mongo error", e);
}
```

### Транзакции (для реплика-сетов и шардированных кластеров)

```java
try (ClientSession session = mongoClient.startSession()) {
    session.startTransaction();
    try {
        collection1.insertOne(session, doc1);
        collection2.insertOne(session, doc2);
        session.commitTransaction();
    } catch (Exception e) {
        session.abortTransaction();
        throw e;
    }
}
```

### Repository Pattern

```java
public class UserRepository {
    
    private final MongoCollection<Document> collection;
    
    public UserRepository(MongoDatabase database) {
        this.collection = database.getCollection("users");
    }
    
    public Optional<Document> findById(ObjectId id) {
        return Optional.ofNullable(
            collection.find(eq("_id", id)).first()
        );
    }
    
    public void save(Document user) {
        if (user.containsKey("_id")) {
            collection.replaceOne(
                eq("_id", user.getObjectId("_id")), 
                user,
                new ReplaceOptions().upsert(true)
            );
        } else {
            collection.insertOne(user);
        }
    }
    
    public List<Document> findByAgeRange(int min, int max) {
        return collection.find(
            and(gte("age", min), lte("age", max))
        ).into(new ArrayList<>());
    }
}
```

### POJO Mapping (для типобезопасности)

```java
// Класс-модель
public class User {
    @BsonId
    private ObjectId id;
    private String name;
    private Integer age;
    private List<String> skills;
    // getters/setters
}

// Регистрация кодеков
// PojoCodecProvider - это компонент MongoDB Java Driver, который автоматически преобразует Java-объекты (POJO) в документы MongoDB и обратно.
PojoCodecProvider pojoCodecProvider = PojoCodecProvider.builder()
    .register(User.class)
    .build();

MongoClientSettings settings = MongoClientSettings.builder()
    .codecRegistry(CodecRegistries.fromRegistries(
        MongoClientSettings.getDefaultCodecRegistry(),
        fromProviders(pojoCodecProvider)
    ))
    .build();

// Использование

// Без PojoCodecProvider (работа с Document):
Document doc = new Document("name", "John")
        .append("age", 30);

collection.insertOne(doc);
// Нужно вручную создавать Document

// С PojoCodecProvider (работа с объектами):
User user = new User("John", 30);    // Просто Java-объект!

userCollection.insertOne(user);      // Автоматически конвертируется в MongoDB
```

### Логирование и мониторинг

#### Включение логирования драйвера:

```xml
<logger name="org.mongodb.driver" level="DEBUG"/>
```

#### Метрики для мониторинга:

- Размер коллекции
- Количество операций в секунду
- Время выполнения запросов
- Размер индексов
- Использование пула соединений

### Безопасность

#### Аутентификация:

```java
MongoCredential credential = MongoCredential.createCredential(
    "username", 
    "database", 
    "password".toCharArray()
);
```

#### TLS/SSL подключение:

```java
ConnectionString cs = new ConnectionString(
    "mongodb://host:27017/mydb?ssl=true&tls=true"
);
```

#### Роли и разрешения:

- Создавайте пользователей с минимальными необходимыми правами
- Используйте отдельные пользователи для app/admin/backup
- Ограничивайте доступ по IP

### Тестирование

#### TestContainers для интеграционных тестов:

```java
MongoDBContainer mongo = new MongoDBContainer("mongo:7.0")
    .withDatabaseName("testdb");
mongo.start();

String uri = mongo.getReplicaSetUrl();
// использовать в тестах

#### Очистка между тестами:
@BeforeEach
void cleanup() {
    collection.deleteMany(new Document());
}
```

---

## Приложения

### Частые фильтры (Bson helpers)

```java
eq("field", value)     // Равенство
ne("field", value)     // Не равно
gt("field", value)     // Больше
gte("field", value)    // Больше или равно
lt("field", value)     // Меньше
lte("field", value)    // Меньше или равно
in("field", values)    // В списке
nin("field", values)   // Не в списке
exists("field", bool)  // Поле существует
regex("field", pattern)// Регулярное выражение
text("searchTerm")     // Текстовый поиск
```

### Логические операторы

```java
and(filter1, filter2, ...)  // И
or(filter1, filter2, ...)   // ИЛИ
nor(filter1, filter2, ...)  // НИ
not(filter)                 // НЕ
```

### Агрегация (базовый пример)

```java
List<Bson> pipeline = Arrays.asList(
        // Шаг 1: Берем только активных пользователей
        match(eq("active", true)),                    // WHERE active = true

        // Шаг 2: Группируем по городу
        group("city",                                 // GROUP BY city
                sum("count", 1),                      // COUNT(*) as count
                avg("avgAge", "$age")),               // AVG(age) as avgAge

        // Шаг 3: Сортируем по убыванию количества
        sort(descending("count")),                    // ORDER BY count DESC

        // Шаг 4: Берем топ-10 городов
        limit(10)                                     // LIMIT 10
);

AggregateIterable<Document> result = collection.aggregate(pipeline);

// Результат: топ-10 городов с наибольшим количеством активных пользователей
```

### Полезные ссылки

- Официальная документация: https://www.mongodb.com/docs/drivers/java/
- GitHub драйвера: https://github.com/mongodb/mongo-java-driver
- MongoDB Shell: https://www.mongodb.com/docs/mongodb-shell/
- Compass (GUI): https://www.mongodb.com/products/compass

---