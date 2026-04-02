# JsonPath

## Содержание

1. [Основы](#1-основы)
2. [Синтаксис и операторы](#2-синтаксис-и-операторы)
3. [Примеры запросов](#3-примеры-запросов)
4. [Best Practices](#4-best-practices)

---

# 1. Основы

> JsonPath — инструмент для работы с JSON-ответами API в Rest Assured.
>
> Позволяет извлекать данные из JSON-структур, используя выражения, похожие на XPath для XML.

## Что такое JsonPath

- **Назначение** — Извлечение значений из JSON-ответов без ручного парсинга
- **Синтаксис** — Похож на XPath (для XML) и JavaScript-обращение к полям
- **Контекст использования** — В основном в автотестах API с Rest Assured

## Основные возможности

- **Извлечение значений** — Получение данных по ключам из JSON-объектов
- **Работа с массивами** — Доступ к элементам по индексу и фильтрация
- **Фильтры** — Поиск объектов по условиям
- **Агрегации** — Подсчёт элементов, поиск мин/макс значений, суммы

## Быстрый старт с Rest Assured

> Rest Assured включает поддержку JsonPath из коробки. Отдельная зависимость не требуется.

## Зависимость

- Достаточно подключить только `rest-assured`
- JsonPath встроен в основную библиотеку

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

## Получение JsonPath из Response

- Вызовите метод `jsonPath()` на объекте `Response`
- Возвращает объект `JsonPath` для навигации по JSON

```java
Response response = get("/api/users");
JsonPath json = response.jsonPath();
```

## Методы извлечения

- `getString(String path)` — извлечение строки
- `getInt(String path)` — извлечение целого числа
- `getLong(String path)` — извлечение `long`
- `getDouble(String path)` — извлечение `double`
- `getFloat(String path)` — извлечение `float`
- `getBoolean(String path)` — извлечение булевого значения
- `getList(String path)` — извлечение списка (`List<Object>` или `List<T>`)
- `getMap(String path)` — извлечение объекта как `Map<String, Object>`
- `getObject(String path, Class<T> clazz)` — маппинг на Java-объект через Jackson
- `read(String path)` — универсальное чтение, возвращает `Object` (требует приведения типа)
- `exists(String path)` — проверка существования пути (возвращает `boolean`)
- `keys(String path)` — получение списка ключей объекта

## Примеры использования

```java
// Базовые типизированные методы
String name = json.getString("users[0].name");
int id = json.getInt("users[0].id");
boolean active = json.getBoolean("users[0].active");
double price = json.getDouble("orders[0].amount");

// Извлечение коллекций
List<String> emails = json.getList("users.email");
Map<String, Object> meta = json.getMap("meta");

// Маппинг на Java-объект (требуется класс с геттерами/сеттерами)
User user = json.getObject("users[0]", User.class);

// Универсальное чтение с приведением типа
Object value = json.read("users[0].name");
String name = (String) value; // требует проверки типа

// Проверка существования поля перед извлечением
if (json.exists("users[0].middleName")) {
    String middleName = json.getString("users[0].middleName");
}

// Получение ключей динамического объекта
List<String> keys = json.keys("metadata");
for (String key : keys) {
    System.out.println(key + ": " + json.getString("metadata." + key));
}
```

---

## Зачем использовать JsonPath в тестах

✅ Преимущества:

- **Читаемость** — Выразительный синтаксис, понятный без комментариев
- **Гибкость** — Поддержка вложенных структур, массивов и фильтров
- **Интеграция** — Встроен в Rest Assured, не требует дополнительных библиотек
- **Типобезопасность** — Методы getInt(), getString(), getList() для корректного приведения

❌ Ограничения:

- Только для чтения данных (не для модификации JSON)
- Требует валидного JSON (ошибки парсинга выбрасывают исключения)
- Сложные фильтры могут быть менее производительными на больших ответах

## Ключевые понятия

- **JsonPath выражение** - Строка запроса (например, `users[0].name`)
- **Response** - Объект ответа Rest Assured, из которого извлекается JsonPath
- **Фильтр** - Условие для выборки элементов массива (`findAll { it.age > 25 }`)
- **Агрегация** - Операция над коллекцией (size(), max(), min())

---

# 2. Синтаксис и операторы

> Синтаксис JsonPath похож на обращение к полям объекта в JavaScript.
>
> Поддерживает навигацию по дереву, фильтрацию массивов и агрегацию данных.

## Основные операторы

- `$` — Корневой элемент (начало пути) (`"$"`)
  В библиотеке RestAssured (как и во многих других реализациях JsonPath) символ $ подразумевается по умолчанию. 

  Вы можете его опустить, и парсер сам добавит его неявно.

  ```java
  // Эти выражения эквивалентны:
  jsonPath.getString("name")           // Неявный $
  jsonPath.getString("$.name")         // Явный $
  jsonPath.getString("$['name']")      // Альтернативный синтаксис
  
  // $ становится обязательным в некоторых ситуациях:
  
  // 1. При работе с вложенными JsonPath
  String author = jsonPath.getString("$.store.book[0].author");

  // 2. При использовании сложных фильтров
  jsonPath.getList("$.store.book[?(@.price < 10)]");

  // 3. При рекурсивном поиске (без $ не сработает)
  jsonPath.getList("$..author");  // Найти все поля author на любом уровне

  // 4. В некоторых реализациях JsonPath (например, Jayway)
  JsonPath.read(json, "$.address.city");  // Явный $ обязателен
  ```
- `.` — Доступ к полю объекта через точку (`"$.store.name"`)
- `[]` — Доступ к полю через квадратные скобки (`"$['store']"`)
- `[*]` — Все элементы текущего массива или объекта (`"$.store.book[*]"`)
- `[n]` — Элемент по индексу (0-based) (`"store.book[0].price"`)
- `[-1:]` — Последний элемент массива (`"$.store.book[-1:]"`)
- `[start:end]` — Срез массива (как в Python) (`"$.store.book[1:3]"`)
- `[0,1]` — Несколько конкретных индексов (`"$.store.book[0,1]"`)
- `..` — Рекурсивный поиск (глубокий скан всех уровней) (`"$..author"`)
- `?()` — Фильтр по условию (`"$.store.book[?(@.price < 10)]"`)
- `@` — Ссылка на текущий элемент внутри фильтра (`"[?(@.id == 5)]"`)

## Groovy vs Стандартные фильтры в RestAssured

### Стандартный JsonPath

- Синтаксис: `$[?(@.field == 'value')]`
- Ссылка на элемент: `@`
- Где используется: Внутри `?()` фильтров
- Пример: `jsonPath.getList("$[?(@.age >= 18)]")`

### Groovy-выражения (расширение RestAssured)

- Синтаксис: collection.find { it.field == 'value' }
- Ссылка на элемент: it
- Где используется: Методы `find`, `findAll`, `collect`, `sum`, `max` и др.
- Пример: jsonPath.getInt("users.find { it.age >= 18 }.name")

### Что нельзя смешивать

```java
// @ с Groovy
.find { @.age == 18 }   // ❌ Ошибка!

// it с ?()
$[?(it.age == 18)]      // ❌ Ошибка!
```

## Логические операторы в фильтрах

Используются внутри выражений `?()` для построения сложных условий.

> JsonPath синтаксически вдохновлен JavaScript, но имеет свои правила. 
> 
> Внутри выражения `?()` строки должны быть заключены в одинарные кавычки, чтобы парсер мог отличить их от синтаксиса самого JsonPath
>
> Использование двойных кавычек при сравнении строк внутри `?()` приведет к ошибке — используйте одинарные!
> 
> ```java
> String status = "active";
>
> // ❌ Ошибка - status интерпретируется как поле JSON
> jsonPath.getList("$[?(@.status == status)]");
> 
> // ✅ Правильно - подставляем значение переменной, но оборачиваем одинарными кавычками
> jsonPath.getList("$[?(@.status == '" + status + "')]");
> 
> // ✅ Правильно - явно строим строку с использованием String.format() 
> String query = String.format("$[?(@.status == '%s')]", status);
> jsonPath.getList(query);
> ```

- `==` — Равно (`"[?(@.status == 'active')]"`)
- `!=` — Не равно (`"[?(@.type != 'admin')]"`)
- `<, >, <=, >=` — Числовое сравнение (`"[?(@.age >= 18)]"`)
- `&&` — Логическое И (`"[?(@.age > 18 && @.active == true)]"`)
- `||` — Логическое ИЛИ (`"[?(@.role == 'admin' || @.role == 'manager')]"`)
- `=~` — Соответствие регулярному выражению (`"[?(@.email =~ /.*@gmail.com/)]"`)
  
  С регулярными выражениями нужно быть внимательным:
  ```java
  // ✅ Правильно
  jsonPath.getList("$[?(@.email =~ /.*@gmail\\.com/)]");

  // ✅ Строка внутри регулярки - тоже одинарные кавычки не нужны, используется слеш /
  jsonPath.getList("$[?(@.name =~ /^John/)]");

  // ❌ Ошибка - кавычки внутри регулярки не нужны
  jsonPath.getList("$[?(@.name =~ '/^John/')]");
  ```

## Функции агрегации, которые **входят в стандарт JsonPath**

Позволяют выполнять вычисления над массивами прямо в выражении JsonPath.

- `size()` или `length()` — Количество элементов в массиве (`"$.users.size()"`)
- `min()` — Минимальное значение в массиве (`"$.prices.min()"`)
- `max()` — Максимальное значение в массиве (`"$.prices.max()"`)
- `avg()` — Среднее значение (`"$.prices.avg()"`)
- `sum()` — Сумма всех значений (`"$.orders.sum()"`)

## Функции агрегации, которые **доступны только в Groovy-синтаксисе RestAssured**

- `first()` — Первый элемент массива (`"$.users.first()"`)
- `last()` — Последний элемент массива (`"$.users.last()"`)
- `head()` — Синоним `first()` (`"$.users.head()"`)
- `tail()` — Все элементы, кроме первого (`"$.users.tail()"`)
- `count()` — Синоним size() (`"$.users.count()"`)
- `isEmpty()` — Проверка на пустоту (true/false) (`"$.users.isEmpty()"`)
- `contains(value)` — Проверка наличия значения (`"$.users*.id.contains(3)"`)
- `flatten()` — Преобразование вложенных списков в плоский (`"$.nestedArrays.flatten()"`)
- `unique()` — Уникальные значения (без дубликатов) (`"$.users*.role.unique()"`)
- `sort()` — Сортировка значений (`"$.orders.amount.sort()"`)
- `reverse()` — Обратный порядок (`"$.users*.name.reverse()"`)
- `collect()` — Преобразование элементов (`"$.users.collect { it.name.toUpperCase() }"`)
- `find()` — Поиск первого подходящего элемента (`"$.users.find { it.age > 25 }"`)
- `findAll()` — Поиск всех подходящих элементов (`"$.users.findAll { it.active == true }"`)
- `findResult()` — Поиск и возврат конкретного поля (`"$.users.findResult { it.role == 'admin' ? it.name : null }"`)
- `groupBy()` — Группировка по ключу (`"$.orders.groupBy { it.status }"`)
- `each()` — Итерация по элементам (итератор для действий, не для извлечения данных) (`"$.users.each { println it.name }"`)

## Примеры использования в Rest Assured

```java
// Извлечение всех авторов рекурсивно
List<String> authors = response.jsonPath().getList("$..author"); // ($..author) найдет все поля author на любом уровне вложенности JSON.

// Фильтрация пользователей по возрасту
List<String> adults = response.jsonPath().getList("users.findAll { it.age >= 18 }.name");

// Поиск конкретного элемента
String admin = response.jsonPath().getString("users.find { it.role == 'admin' }.name"); // Если элементов, удовлетворяющих условию несколько, то вернёт первого.

// Агрегация: подсчёт количества заказов
int count = response.jsonPath().getInt("orders.size()");

// Агрегация: максимальная сумма заказа
int maxOrder = response.jsonPath().getInt("orders.max { it.amount }.amount");

// Пробелы только для читаемости. Можно писать: "users.find{it.role=='admin'}.name"
```

---

## Ключевые выводы

1. **`$` всегда начинает путь** — это корень JSON-документа
2. **`..` мощный, но медленный** — используйте рекурсивный поиск только когда структура неизвестна
3. **Фильтры `findAll` и `find`** — основной инструмент для работы с коллекциями в Rest Assured
4. **Агрегации экономят код** — не нужно выгружать список и считать размер в Java

---

# 3. Примеры запросов

> Все примеры приведены в контексте Rest Assured.
>
> Исходный JSON для примеров — ответ API с пользователями и заказами.

## Исходные данные для примеров

```json
{
  "users": [
    {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com",
      "age": 30,
      "active": true,
      "role": "admin"
    },
    {
      "id": 2,
      "name": "Jane Smith",
      "email": "jane@example.com",
      "age": 25,
      "active": false,
      "role": "user"
    },
    {
      "id": 3,
      "name": "Bob Wilson",
      "email": "bob@gmail.com",
      "age": 35,
      "active": true,
      "role": "user"
    }
  ],
  "orders": [
    {"id": 101, "amount": 150, "status": "completed"},
    {"id": 102, "amount": 75, "status": "pending"},
    {"id": 103, "amount": 200, "status": "completed"}
  ],
  "meta": {
    "total": 3,
    "page": 1
  }
}
```

---

## Базовые запросы

### Извлечение простых полей

```java
// Извлечение целого числа
int userId = response.jsonPath().getInt("users[0].id");

// Извлечение строки
String userName = response.jsonPath().getString("users[0].name");

// Извлечение булевого значения
boolean isActive = response.jsonPath().getBoolean("users[0].active");

// Извлечение вложенного поля
int totalPages = response.jsonPath().getInt("meta.page");
```

### Доступ к вложенным структурам

```java
// Точечная нотация для вложенных объектов
String email = response.jsonPath().getString("users[1].email");

// Комбинированный доступ
int age = response.jsonPath().getInt("users.find { it.id == 3 }.age");
```

### Работа с массивами

```java
// Первый элемент массива
String firstName = response.jsonPath().getString("users[0].name");

// Последний элемент массива
String lastName = response.jsonPath().getString("users[-1:].name");

// Все значения поля из массива
List<String> allNames = response.jsonPath().getList("users.name");

// Конкретные индексы
List<String> someNames = response.jsonPath().getList("users[0,2].name");

// Срез массива
List<String> rangeNames = response.jsonPath().getList("users[1:3].name");
```

---

## Фильтрация

### Фильтр findAll — все совпадения

```java
// Все активные пользователи
List<String> activeUsers = response.jsonPath().getList(
    "users.findAll { it.active == true }.name"
);

// Пользователи старше 28 лет
List<String> olderUsers = response.jsonPath().getList(
    "users.findAll { it.age > 28 }.name"
);

// Пользователи с ролью admin
List<String> admins = response.jsonPath().getList(
    "users.findAll { it.role == 'admin' }.name"
);

// Пользователи с email на gmail.com
List<String> gmailUsers = response.jsonPath().getList(
    "users.findAll { it.email =~ /.*@gmail.com/ }.name"
);
```

### Фильтр find — первое совпадение

```java
// Первый активный пользователь
String firstActive = response.jsonPath().getString(
    "users.find { it.active == true }.name"
);

// Пользователь с конкретным ID
String userName = response.jsonPath().getString(
    "users.find { it.id == 2 }.name"
);
```

### Сложные условия фильтрации

```java
// Активные пользователи старше 30 лет
List<String> filtered = response.jsonPath().getList(
    "users.findAll { it.active == true && it.age > 30 }.name"
);

// Админы или пользователи старше 35
List<String> complex = response.jsonPath().getList(
    "users.findAll { it.role == 'admin' || it.age > 35 }.name"
);

// Неактивные пользователи
List<String> inactive = response.jsonPath().getList(
    "users.findAll { it.active != true }.name"
);
```

---

## Агрегация

### Подсчёт элементов

```java
// Количество пользователей
int userCount = response.jsonPath().getInt("users.size()");

// Количество заказов
int orderCount = response.jsonPath().getInt("orders.size()");

// Количество активных пользователей
int activeCount = response.jsonPath().getInt(
    "users.findAll { it.active == true }.size()"
);
```

### Мин/Макс значения

```java
// Минимальный возраст
int minAge = response.jsonPath().getInt("users.min { it.age }.age");

// Максимальный возраст
int maxAge = response.jsonPath().getInt("users.max { it.age }.age");

// Максимальная сумма заказа
int maxAmount = response.jsonPath().getInt("orders.max { it.amount }.amount");

// Минимальная сумма заказа
int minAmount = response.jsonPath().getInt("orders.min { it.amount }.amount");
```

### Сумма и среднее

```java
// Сумма всех заказов
int totalAmount = response.jsonPath().getInt("orders.sum { it.amount }");

// Средняя сумма заказа (может потребоваться приведение типа)
double avgAmount = response.jsonPath().getDouble("orders.avg { it.amount }");
```

### Дополнительные примеры с методами Groovy для работы с коллекциями прямо в JsonPath выражениях. 

```java
// Исходные данные: [[1, 2], [3, 4], [5]]
List<Integer> flatList = response.jsonPath().getList("nestedArrays.flatten()");
// Результат: [1, 2, 3, 4, 5]

// Исходные данные: пользователи с ролями ["admin", "user", "admin", "guest"]
List<String> roles = response.jsonPath().getList("users*.role.unique()");
// Результат: ["admin", "user", "guest"]

// Сортировка объектов по полю
List<Map<String, Object>> sortedUsers = response.jsonPath().getList("users.sort { it.age }");

// Исходные данные: заказы со статусами ["completed", "pending", "completed"]
Map<String, List<Object>> grouped = response.jsonPath().getMap("orders.groupBy { it.status }");
// Результат: { "completed": [...], "pending": [...] }

// Вывод имен всех пользователей в консоль при тесте
response.jsonPath().getList("users.each { println it.name }");
```

---

## Интеграция с утверждениями Rest Assured

### Прямая валидация в then()

```java
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

get("/api/users")
    .then()
    .statusCode(200)
    .body("users.size()", equalTo(3))
    .body("users[0].name", equalTo("John Doe"))
    .body("users.findAll { it.active == true }.size()", greaterThan(0))
    .body("orders.max { it.amount }.amount", equalTo(200))
    .body("meta.total", equalTo(3));
```

### Извлечение с последующей валидацией

```java
Response response = get("/api/users");

List<String> names = response.jsonPath().getList("users.name");
Assert.assertTrue(names.contains("John Doe"));

int maxOrder = response.jsonPath().getInt("orders.max { it.amount }.amount");
Assert.assertEquals(200, maxOrder);
```
---

## Рекурсивный поиск

### Поиск по всем уровням вложенности

```java
// Найти все поля 'name' независимо от уровня вложенности
List<String> allNames = response.jsonPath().getList("$..name");

// Найти все поля 'id' в документе
List<Integer> allIds = response.jsonPath().getList("$..id");

// Найти все id заказов со статусом 'completed' в любой вложенности
List<String> completed = response.jsonPath().getList(
    "$..orders.findAll { it.status == 'completed' }.id"
);
```

> Рекурсивный поиск удобен, когда структура JSON может меняться или данные находятся на неизвестном уровне вложенности.

---

## Ключевые выводы

1. **`getInt()`, `getString()`, `getList()`** — основные методы для извлечения данных в Rest Assured
2. **`findAll { }` возвращает список**, `find { }` возвращает первый элемент
3. **Агрегации (`size()`, `max()`, `sum()`)** позволяют считать данные без выгрузки в Java
4. **Валидация в `then().body()`** — самый чистый способ проверки ответов API
5. **Рекурсивный поиск (`..`)** используйте с осторожностью — может быть медленным на больших ответах

---

# 4. Best Practices

> Рекомендации по эффективному и безопасному использованию JsonPath в Rest Assured тестах.

## ✅ Делайте

- Используйте конкретные пути вместо рекурсивных (`..`), если структура JSON известна и стабильна
- Применяйте `find { }` вместо `findAll { }` когда нужен только первый элемент (быстрее)
- Валидируйте ответы через `then().body()` — это чище и даёт лучшие сообщения об ошибках
- Кэшируйте `JsonPath` объект при множественных извлечениях из одного ответа
- Проверяйте наличие поля перед извлечением в динамических структурах
- Используйте типизированные методы (`getInt()`, `getString()`) для явного приведения типов
- Группируйте связанные утверждения в одном `then()` блоке для читаемости

## ❌ Не делайте

- Не полагайтесь на порядок ключей в JSON-объектах (они неупорядочены по спецификации)
- Не используйте сложные фильтры в продакшен-коде (только в тестах)
- Не забывайте про обработку `PathNotFoundException` при работе с опциональными полями
- Не применяйте рекурсивный поиск (`..`) на больших ответах без необходимости (медленно)
- Не извлекайте весь JSON в Java-объект только для проверки одного поля
- Не дублируйте логику фильтрации — выносите сложные пути в константы

---

## Распространённые ошибки

| Ошибка                      | Причина                             | Решение                                                                      |
|:----------------------------|-------------------------------------|------------------------------------------------------------------------------|
| `PathNotFoundException`     | Поле отсутствует или путь неверен   | Проверьте структуру JSON, используйте `find()` вместо прямого доступа        |
| `ClassCastException`        | Несоответствие типа метода и данных | Используйте правильный метод (`getInt()` для чисел, `getString()` для строк) |
| Пустой список вместо данных | Фильтр не нашёл совпадений          | Проверьте условие фильтра, убедитесь что данные существуют                   |
| Медленные тесты             | Рекурсивный поиск на больших JSON   | Замените `..` на конкретный путь, если структура известна                    |

---

## Советы по отладке

### Вывод JSON в консоль

```java
Response response = get("/api/users");
System.out.println(response.jsonPath().prettyPrint());
```

### Проверка существования поля

```java
boolean hasField = response.jsonPath().exists("users[0].email");
if (hasField) {
    String email = response.jsonPath().getString("users[0].email");
}
```

### Сохранение JsonPath для повторного использования

```java
JsonPath json = response.jsonPath();
String name = json.getString("users[0].name");
int age = json.getInt("users[0].age");
String email = json.getString("users[0].email");
```

---