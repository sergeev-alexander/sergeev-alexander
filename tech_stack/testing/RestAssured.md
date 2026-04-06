# Rest Assured

## Содержание

1. [Введение и Подключение](#1-введение-и-подключение)
2. [Ядро DSL и Статические импорты](#2-ядро-dsl-и-статические-импорты)
3. [Глобальная конфигурация (RestAssured.* и RestAssuredConfig)](#3-глобальная-конфигурация-restassured-и-restassuredconfig)
4. [Конструктор запроса (RequestSpecification)](#4-конструктор-запроса-requestspecification)
5. [Аутентификация (AuthConfig)](#5-аутентификация-authconfig)
6. [POJO-интеграция (Фокус на RA-обвязке)](#6-pojo-интеграция-фокус-на-ra-обвязке)
7. [Обработка и извлечение ответа (Response / ExtractableResponse)](#7-обработка-и-извлечение-ответа-response--extractableresponse)
8. [Валидация](#8-валидация)
9. [Спецификации и Фильтры](#9-спецификации-и-фильтры)
10. [Логирование (LogSpecification)](#10-логирование-logspecification)
11. [Асинхронность и Продвинутые фичи](#11-асинхронность-и-продвинутые-фичи)
12. [Паттерны интеграции с JUnit 5 / TestNG](#12-паттерны-интеграции-с-junit-5--testng)
13. [Типичные ошибки и Best Practices (RA 5.x)](#13-типичные-ошибки-и-best-practices-ra-5x)

---

## 1. Введение и Подключение

> Rest Assured — это Java-библиотека для тестирования REST API, предоставляющая выразительный DSL-синтаксис, вдохновлённый BDD-фреймворками. 
> 
> Она абстрагирует низкоуровневую работу с HTTP-клиентами, реализуя цепочечный вызов методов с ленивым выполнением запросов и встроенной интеграцией с Hamcrest для валидации ответов.

---

- **Место в стеке тестирования** — надстройка над Apache HttpClient, интегрируется с JUnit 5/TestNG для декларативной проверки контрактов API без написания boilerplate-кода.
- **Сравнение с низкоуровневыми клиентами** — вместо ручного создания `HttpRequest`, обработки статус-кодов и парсинга `InputStream`, RA предлагает читаемый fluent-API: `given().param(...).when().get().then().statusCode(200)`.
- **Ленивое выполнение (Lazy Evaluation)** — построение `RequestSpecification` происходит мгновенно в памяти. Фактический HTTP-вызов инициируется только при вызове методов валидации (`then()`), извлечения (`extract()`) или асинхронного ожидания.
- **DSL-цепочки** (Domain-Specific Language (предметно-ориентированный язык)) — каждый метод возвращает новый объект спецификации или ответа, что позволяет выстраивать вложенные проверки и конфигурации без промежуточных переменных.

---

### Подключение/Зависимости

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

```groovy
dependencies {
    testImplementation 'io.rest-assured:rest-assured:5.4.0'
}
```

---

### Модульная структура 5.x и Транзитивные зависимости

Начиная с 5.x, библиотека разделена на модули. Базовый артефакт `rest-assured` автоматически подтягивает:

- `json-path` и `xml-path` — декларативный парсинг ответов через GPath/Groovy.
- `groovy` и `groovy-xml` — ядро движка шаблонов (критично для работы DSL-матчеров).
- `hamcrest` и `hamcrest-core` — набор матчеров для `.assertThat()`.
- `jackson-databind` и `jackson-core` — сериализация/десериализация POJO по умолчанию.

### Решение конфликтов и Jakarta vs Javax

- RA 5.x полностью мигрировал на `jakarta.servlet-api`. Конфликты возникают при использовании старых библиотек, тянущих `javax.servlet`.
- При работе с Spring Boot 3.x или Jakarta EE 9+ конфликты отсутствуют. Для legacy-проектов используйте `<exclusions>`.

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <exclusions>
        <exclusion>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

---

## Best Practices

- Фиксируйте версию RA в `<dependencyManagement>` или `platform` BOM для синхронизации транзитивных зависимостей.
- Откажитесь от устаревшего артефакта `rest-assured-all` — он добавляет избыточный вес и дублирует модули.
- Убедитесь, что версия `groovy` (по умолчанию 4.x в RA 5.x) совместима с вашей JDK (требуется JDK 11+).
- Явно управляйте `jakarta.servlet` в зависимостях, если проект находится на этапе миграции с `javax`.
- Изолируйте зависимости RA в профиле `test`, чтобы не загрязнять production classpath.

---

# 2. Ядро DSL и Статические импорты

> Фундаментальный механизм Rest Assured, построенный на fluent-интерфейсе и статических импортах. 
> 
> DSL строго разделяет жизненный цикл HTTP-запроса на фазы подготовки, отправки и валидации, обеспечивая декларативный BDD-стиль.

---

## given() → when() → then() → extract(): потоковая семантика

- `given()` — фаза конфигурации запроса. Устанавливает заголовки, параметры, тело, куки и авторизацию. Возвращает `RequestSpecification`.
- `when()` — фаза выполнения. Определяет HTTP-метод (`get()`, `post()`, `put()`, `delete()`) и целевой URI. Инициирует ленивую отправку сетевого запроса.
- `then()` — фаза валидации ответа. Проверяет статус-код, заголовки и содержимое тела через Hamcrest. Возвращает `ValidatableResponse`.
- `extract()` — фаза извлечения данных. Прерывает цепочку валидаций, позволяя получить `Response` или конкретное значение для дальнейшего использования в тесте.

### Фазы выполнения и ленивая оценка

- Построение спецификации происходит синхронно в памяти, сетевых вызовов не выполняется.

  На этом этапе все методы `given()` и промежуточные конфигурации (параметры, заголовки, тело) просто заполняют объект `RequestSpecification` в куче JVM. 

  Никакой проверки корректности URL, доступности хоста или синтаксиса запроса не производится — это чисто Java-операции. 
  
  Такой подход позволяет гибко переиспользовать спецификации и динамически изменять их перед фактическим выполнением.

- Фактический HTTP-запрос отправляется строго при первом вызове методов валидации, извлечения или асинхронного ожидания.

  До момента вызова `then()`, `extract()` или аналогичных терминальных операций Rest Assured не открывает сокеты, не резолвит DNS и не формирует wire-представление запроса. 
  
  Ленивая инициализация позволяет конструировать сложные цепочки в условных блоках или циклах без риска многократного дублирования сетевых вызовов. 

  Кроме того, это даёт возможность подготовить запрос в одном месте теста, а выполнить его позже — после дополнительной логики или ожидания условий.

- Ошибки валидации выбрасываются сразу после получения ответа, до перехода к extract().

  Как только HTTP-ответ полностью получен и распарсен (например, JSON или XML преобразован в доступную структуру), Rest Assured последовательно применяет все объявленные в then() проверки. 
  
  Если какая-либо ассерция (статус-код, поле body, заголовок) не проходит, генерируется исключение `AssertionError`, и выполнение теста прерывается — extract() в этом случае даже не вызывается. 
  
  Такое поведение гарантирует, что извлечение данных происходит только из заведомо валидного ответа, предотвращая ложные негативные результаты и последующие ошибки парсинга.

---

## Обязательные static imports для 5.x

Без статических импортов DSL теряет читаемость и BDD-выразительность. Минимальный набор для комфортной работы:

```java
import static io.restassured.RestAssured.given;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;
import static io.restassured.matcher.RestAssuredMatchers.*;
```

---

## Разделение цепочки: переменные vs one-liner

### One-liner подход

Идеален для простых проверок контракта. Не хранит промежуточное состояние, минимизирует boilerplate.

```java
given()
    .header("Accept", "application/json")
    .queryParam("limit", 10)
.when()
    .get("/api/users")
.then()
    .statusCode(200)
    .body("size()", greaterThan(0));
```

### Разделение на переменные

Рекомендуется для сложных сценариев, где требуется извлечь данные, переиспользовать спецификацию или добавить кастомную логику между фазами.

```java
RequestSpecification spec = given()
    .contentType("application/json")
    .body("{\"username\": \"test\", \"password\": \"12345\"}");

Response response = spec
    .when()
    .post("/api/auth/login")
.then()
    .extract().response();

String token = response.jsonPath().getString("accessToken");
```

---

## JUnit 5 vs TestNG: точки инициализации

| Задача                           | JUnit 5               | TestNG          |
|:---------------------------------|:----------------------|:----------------|
| Инициализация `baseURI` / `port` | `@BeforeAll` (static) | `@BeforeClass`  |
| Сброс глобального состояния      | `@AfterAll` (static)  | `@AfterClass`   |
| Изоляция перед каждым тестом     | `@BeforeEach`         | `@BeforeMethod` |
| Очистка после каждого теста      | `@AfterEach`          | `@AfterMethod`  |
| Аннотация тест-метода            | `@Test`               | `@Test`         |

### Пример инициализации (JUnit 5)

```java
@BeforeAll
static void setup() {
    RestAssured.baseURI = "https://api.example.com";
    RestAssured.port = 443;
    RestAssured.basePath = "/v2";
    RestAssured.urlEncodingEnabled = false;
}
```

---

## Best Practices

- Всегда импортируйте `io.restassured.RestAssured.*` и `org.hamcrest.Matchers.*` статически для сохранения BDD-синтаксиса.
- Разбивайте длинные цепочки на `RequestSpecification` и `ExtractableResponse`, если необходимо извлечь данные или добавить промежуточное логирование.
- Не смешивайте валидацию и извлечение в одном `then()`: вызывайте `extract()` только после завершения всех `assertThat()`, чтобы не пропустить ошибки контракта.
- Для JUnit 5 используйте `static` методы в `@BeforeAll`, так как глобальная конфигурация `RestAssured.*` хранится в статических полях класса.
- В TestNG сбрасывайте модифицированные спецификации в `@AfterMethod`, чтобы предотвратить утечку состояния (`statefulness`) между тестовыми методами.

# 3. Глобальная конфигурация (RestAssured.* и RestAssuredConfig)

> Глобальная конфигурация определяет базовые параметры для всех запросов в рамках тестовой сессии. 
> 
> Она позволяет избежать дублирования настроек, централизовать управление соединениями, сериализацией и логированием, а также задавать строгие приоритеты переопределения на уровнях спецификаций и отдельных запросов.

---

## Статические поля `RestAssured.*`

- `baseURI` — базовый URL (схема, хост, порт), например `https://api.example.com`.
- `basePath` — базовый путь, автоматически добавляемый ко всем запросам, например `/v2/resources`.
- `port` — явное переопределение порта (по умолчанию вычисляется из схемы `baseURI`).
- `urlEncodingEnabled` — флаг включения/отключения URL-кодирования параметров (по умолчанию `true`). 

   Критично для передачи спецсимволов в `pathParam`.
- `parser` — парсер по умолчанию для ответов без явного заголовка `Content-Type`.
- `authentication` — глобальная схема аутентификации, применяемая ко всем запросам по умолчанию.

---

## RestAssured.config() и подконфиги

Метод `RestAssured.config()` возвращает объект `RestAssuredConfig`, позволяющий тонко настроить клиентское поведение и инфраструктуру HTTP-вызовов:

- `encoderConfig()` — управление кодировками запросов, дефолтным `Charset` и сжатием (`gzip`, `deflate`).
- `decoderConfig()` — настройка распаковки ответов, игнорирование неподдерживаемых кодировок, дефолтный `ContentCharset`.
- `httpClientConfig()` — параметры низкоуровневого HTTP-клиента:
    - `setParam("http.connection.timeout", int)` — таймаут установки TCP-соединения.
    - `setParam("http.socket.timeout", int)` — таймаут ожидания данных от сервера.
    - `setParam("http.max-connections-per-route", int)` — лимит пула соединений.
- `logConfig()` — конфигурация логирования: `defaultStream()`, `prettyPrintingLogging()`, `blacklistHeader()`.
- `redirectConfig()` — управление редиректами: `followRedirects(boolean)`, `maxRedirects(int)`.
- `sslConfig()` — настройка TLS/SSL: `relaxedHTTPSValidation()` (для тестовых сред), кастомные `TrustStore`/`KeyStore`.
- `objectMapperConfig()` — переключение между сериализаторами: `jackson2ObjectMapperFactory(...)` или `gsonObjectMapperFactory(...)`.

---

## Приоритет применения

Настройки применяются по принципу строгого переопределения от общего к частному:

1. **Global** (`RestAssured.*` / `RestAssured.config()`) — базовый слой, наследуемый всеми тестами в текущей JVM.
2. **Specification** (`RequestSpecBuilder` / `RestAssured.requestSpecification`) — наследует глобальные, но может переопределять их через `.setBaseUri()`, `.addHeader()`, `.config()`.
3. **Request** (локальный `given()`) — наивысший приоритет. Значения, переданные непосредственно в цепочку `given()`, полностью игнорируют глобальные и спецификационные настройки.

---

## Примеры инициализации (JUnit 5 vs TestNG)

### JUnit 5

```java
@BeforeAll
static void globalSetup() {
    RestAssured.baseURI = "https://api.staging.com";
    RestAssured.port = 8443;
    RestAssured.basePath = "/api/v1";
    RestAssured.urlEncodingEnabled = false;

    RestAssured.config = RestAssuredConfig.config()
        .httpClient(HttpClientConfig.httpClientConfig()
            .setParam("http.connection.timeout", 5000)
            .setParam("http.socket.timeout", 10000))
        .objectMapperConfig(ObjectMapperConfig.objectMapperConfig()
            .defaultObjectMapperType(ObjectMapperType.JACKSON_2))
        .sslConfig(SSLConfig.sslConfig().relaxedHTTPSValidation());
}

@AfterAll
static void teardown() {
    RestAssured.reset();
}
```

### TestNG

```java
@BeforeClass
public void globalSetup() {
    RestAssured.baseURI = "https://api.prod.com";
    RestAssured.port = 443;
    RestAssured.basePath = "/api/v2";
    RestAssured.urlEncodingEnabled = true;

    RestAssured.config = RestAssuredConfig.config()
        .logConfig(LogConfig.logConfig()
            .enableLoggingOfRequestAndResponseIfValidationFails(true))
        .decoderConfig(DecoderConfig.decoderConfig()
            .defaultContentCharset(StandardCharsets.UTF_8));
}

@AfterClass
public void cleanup() {
    RestAssured.reset();
}
```

---

## Best Practices

- Всегда вызывайте `RestAssured.reset()` в `@AfterAll` (JUnit 5) или `@AfterClass` (TestNG) для полной очистки статического состояния и предотвращения `Statefulness` утечек между тестовыми сьютами.
- Избегайте модификации `RestAssured.config` внутри отдельных тестовых методов; выносите клиентские таймауты, SSL-настройки и мапперы на глобальный или спецификационный уровень.
- Для переключения на `Gson` используйте `objectMapperConfig().defaultObjectMapperType(ObjectMapperType.GSON)`, но убедитесь, что зависимость `com.google.code.gson:gson` явно добавлена в `pom.xml` / `build.gradle`, так как RA 5.x использует Jackson по умолчанию.
- Отключайте `urlEncodingEnabled` (`false`) только при работе с уже закодированными строками в `pathParam` или при тестировании API, некорректно обрабатывающих `%20` и спецсимволы.
- Централизуйте глобальную конфигурацию в отдельном `ConfigProvider` или базовом `TestBase` классе, чтобы не дублировать логику инициализации в каждом тестовом файле.
- При параллельном запуске тестов (JUnit 5 `@Execution(CONCURRENT)`, TestNG `parallel="methods"`) избегайте мутации статических полей `RestAssured.*` в рантайме. Используйте `ThreadLocal<RequestSpecification>` или передавайте конфигурацию через изолированные экземпляры `RequestSpecBuilder`.
- При работе с микросервисной архитектурой используйте `RestAssured.config().httpClient(...)` для настройки отдельного `ConnectionPool` и `RetryHandler` под конкретные SLA сервисов, вместо глобальных `socketTimeout`.

---

## 4. Конструктор запроса (RequestSpecification)

> RequestSpecification — это изменяемый билдер, отвечающий за конфигурацию HTTP-запроса перед отправкой.
> 
> Он позволяет декларативно задавать заголовки, параметры маршрутизации, тело, прокси и таймауты, обеспечивая гибкое управление сетевым взаимодействием на уровне каждого вызова с приоритетом переопределения глобальных настроек.

---

### Заголовки (Headers)

- `header(String name, String value)` — добавление одного заголовка. Если ключ уже существует, значение добавляется в список (multi-value).
- `headers(Map<String, ?> headers)` — массовая установка заголовков из карты.
- `replaceHeader(String name, String value)` — замена существующего заголовка новым значением.
- `removeHeader(String name)` — удаление заголовка из спецификации.
- `multiValueHeader(String name, String... values)` — явная передача нескольких значений для одного ключа.

```java
RequestSpecification spec = given()
    .header("Authorization", "Bearer token123")
    .header("Accept", "application/json")
    .headers(Map.of(
        "X-Request-ID", UUID.randomUUID().toString(),
        "X-Client-Version", "2.1.0"
    ));
```

---

### Параметры (Parameters)

- `queryParam(String name, Object value)` / `queryParams(Map)` — добавление параметров в строку запроса (`?key=value`).
- `pathParam(String name, Object value)` / `pathParams(Map)` — подстановка значений в URI-шаблон (`/users/{id}`).
- `formParam(String name, Object value)` / `formParams(Map)` — кодирование данных в `application/x-www-form-urlencoded`.
- `multiPart(String controlName, Object content)` — добавление multipart-данных (файлы, поля формы).

#### GET с подстановкой пути и фильтром запроса

```java
given()
    .pathParam("userId", 105)
    .queryParam("fields", "name,email")
    .queryParam("status", "ACTIVE")
.when()
    .get("/api/v1/users/{userId}");

// Итоговый URL: /api/v1/users/105?fields=name,email&status=ACTIVE
// Метод: GET, тело отсутствует
```

#### POST с отправкой формы (Form-Data)
```java
given()
    .formParam("username", "admin")
    .formParam("password", "s3cr3t")
    .formParam("role", "editor")
.when()
    .post("/api/v1/auth/login");

// Итоговый URL: /api/v1/auth/login
// Content-Type: application/x-www-form-urlencoded
// Тело запроса: username=admin&password=s3cr3t&role=editor
```

#### POST с загрузкой файлов (MultiPart)
```java
given()
    .pathParam("ticketId", 9921)
    .multiPart("attachment", new File("logs.zip"), "application/zip")
    .multiPart("description", "Critical error dump", "text/plain")
.when()
    .post("/api/v1/support/tickets/{ticketId}/attachments");

// Итоговый URL: /api/v1/support/tickets/9921/attachments
// Content-Type: multipart/form-data; boundary=...
// Тело: содержит две части (бинарный файл "attachment" + текстовое поле "description")
```

### Тело и контент (Body & Content)

- `body(Object body)` — установка тела запроса. Поддерживает String, byte[], File, InputStream, POJO (сериализуется автоматически через Jackson/Gson).
- `contentType(String type)` — установка `Content-Type`. Поддерживает MIME-строки или `ContentType.JSON`, `ContentType.XML` и др.
- `accept(String type)` — установка заголовка `Accept`.
- `charset(String charset)` / `charset(Charset)` — переопределение кодировки тела запроса.

```java
UserDto user = new UserDto("john", "john@example.com");

given()
    .contentType(ContentType.JSON)
    .accept(ContentType.XML)
    .body(user) // Сериализуется в Json автоматически, т.к. contentType(ContentType.JSON) 
.when()
    .post("/api/users");

---
```

### Прокси (Proxy)

- `proxy(String host, int port)` — маршрутизация трафика через HTTP/HTTPS прокси.
- `proxy(ProxySpecification spec)` — расширенная конфигурация: авторизация, схемы, исключения.
- `noProxy(String... hosts)` — исключение хостов из проксирования (bypass).

```java
ProxySpecification proxy = new ProxySpecification("127.0.0.1", 8080, "http");
proxy.noProxy("localhost", "*.internal.corp");

given()
    .proxy(proxy)
.when()
    .get("/api/external-service");
```

---

### Таймауты запроса (Timeouts)

- `timeout(long millis)` — общий таймаут на выполнение запроса (соединение + чтение).
- `connectTimeout(long millis)` — макс. время на установку TCP-соединения.
- `readTimeout(long millis)` — макс. время ожидания ответа от сервера после установления соединения.

```java
given()
    .timeout(10_000)
    .connectTimeout(3_000)
    .readTimeout(7_000)
.when()
    .get("/api/slow-endpoint");
```

---

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5
class ApiRequestTest {
    
    private RequestSpecification commonSpec;

    @BeforeEach
    void initSpec() {
        commonSpec = new RequestSpecBuilder()
            .setBaseUri("https://api.staging.com")
            .setPort(443)
            .setContentType(ContentType.JSON)
            .addHeader("X-Trace-Id", UUID.randomUUID().toString())
            .build();
    }

    @Test
    void testQueryAndPathParams() {
        given()
            .spec(commonSpec)
            .pathParam("userId", 105)
            .queryParam("include", "roles,permissions")
        .when()
            .get("/api/v1/users/{userId}")
        .then()
            .statusCode(200);
    }
}
```

```java
// TestNG
public class ApiRequestTestNG {
    
    private RequestSpecification threadSafeSpec;

    @BeforeMethod
    public void setupMethod() {
        threadSafeSpec = new RequestSpecBuilder()
            .setBaseUri("https://api.prod.com")
            .setContentType(ContentType.JSON)
            .addHeader("Accept-Encoding", "gzip")
            .build();
    }

    @Test
    public void testProxyAndTimeouts() {
        given()
            .spec(threadSafeSpec)
            .proxy("proxy.corp.net", 3128)
            .connectTimeout(5000)
            .readTimeout(15000)
        .when()
            .post("/api/v2/sync-data")
        .then()
            .statusCode(is(201));
    }
}
```

---

## Best Practices

- Не модифицируйте `RequestSpecification` после передачи в `given()`: объект изменяем (mutable), но RA копирует состояние на этапе выполнения. Повторное использование одного инстанса без клонирования может привести к `Statefulness` ошибкам в параллельных тестах.
- Для массовых параметров используйте `queryParams(Map)` или `pathParams(Map)` вместо цепочки вызовов `queryParam()`, чтобы сократить boilerplate и улучшить читаемость.
- При работе с `formParam` убедитесь, что `contentType` установлен в `application/x-www-form-urlencoded`, иначе сервер может не распознать тело запроса.
- Используйте `noProxy()` для исключения локальных или внутренних сервисов из проксирования, что критично при работе с CI/CD пайплайнами и сервис-мешами.
- Таймауты запроса (`connectTimeout`, `readTimeout`) всегда должны быть ниже глобальных таймаутов HTTP-клиента, чтобы избежать блокировки потоков в пуле соединений.
- В параллельном запуске создавайте новые экземпляры `RequestSpecification` на каждый тест (через `@BeforeEach`/`@BeforeMethod` или фабрики), а не переиспользуйте статические объекты.
- При отправке бинарных данных (`byte[]`, `File`, `InputStream`) явно указывайте `contentType("application/octet-stream")` или специфичный MIME-тип, чтобы RA не пытался применить JSON/XML сериализатор.

## 5. Аутентификация (AuthConfig)

> Механизмы аутентификации в Rest Assured позволяют декларативно настраивать учетные данные на уровне запроса или глобально. 
> 
> Библиотека поддерживает стандартные HTTP-схемы, OAuth 1.0/2.0, form-based авторизацию и кастомные заголовки/cookie, автоматически управляя жизненным циклом сессий и токенов.

### Базовая аутентификация (Basic Auth)

- `basic(user, pass)` — отправка заголовка `Authorization: Basic ...` только после получения `401 Unauthorized` от сервера (challenge-response).
- `preemptive().basic(user, pass)` — немедленная отправка заголовка в первом же запросе. Рекомендуется для внутренних API или снижения сетевых round-trip.

```java
given()
    .auth().basic("admin", "s3cr3t") // Rest Assured автоматически делает второй запрос внутри одного вызова .get() после получения 401 Unauthorized
.when()
    .get("/api/v1/admin/users");
```


```java
given()
    .auth().preemptive().basic("admin", "s3cr3t")
.when()
    .get("/api/v1/admin/users");
```

### Form-Based Authentication

- `form(user, pass)` — автоматический POST на стандартный `/login` с последующим извлечением и передачей session cookie.
- `form(user, pass, authEndpoint)` — явное указание кастомного эндпоинта логина и параметров формы.

```java
given()
    .auth().form("user", "pass", FormAuthConfig.formAuthConfig()
        .withUsernameFieldName("login")
        .withPasswordFieldName("pwd")
        .withLoggingEnabled(true))
.when()
    .get("/api/v1/protected-resource");
```

### OAuth 1.0 & OAuth 2.0

- `oauth(consumerKey, consumerSecret, accessToken, tokenSecret)` — полная настройка OAuth 1.0a с HMAC-SHA1 подписью запросов.
- `oauth2(token)` — упрощенная обертка, добавляющая `Authorization: Bearer <token>` в заголовок.

```java
given()
    .auth().oauth2("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")
.when()
    .get("/api/v1/users/profile");
```

### Кастомные схемы и токены

Для современных API (JWT, OIDC, API Keys) эффективнее использовать прямую установку заголовков или cookie, минуя встроенные методы `auth()`.

- `header("Authorization", "Bearer <token>")` — ручная передача токена.
- `cookie("JSESSIONID", "abc123")` — управление сессией вручную.
- `header("X-API-Key", "key-value")` — авторизация через кастомные заголовки.

```java
given()
    .header("Authorization", "Bearer " + jwtToken)
    .cookie("X-Tracking-ID", "track_999")
    .header("X-API-Key", "sys_key_777")
.when()
    .post("/api/v1/orders");
```

### Глобальное (RestAssured.authentication) vs локальное применение

- **Глобальное** — задается через `RestAssured.authentication = ...`. Применяется ко всем запросам, если не переопределено локально.
- **Локальное** — указывается в `given().auth()...`. Имеет высший приоритет и изолирует тесты друг от друга.

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5: Глобальная настройка с переопределением в тестах
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class AuthTestJUnit5 {

    @BeforeAll
    void globalAuthSetup() {
        RestAssured.authentication = preemptive().basic("global_user", "global_pass");
        RestAssured.baseURI = "https://api.staging.com";
    }

    @AfterAll
    void resetAuth() {
        RestAssured.authentication = NONE;
    }

    @Test
    void testGlobalAuthApplied() {
        given()
            .pathParam("id", 1)
        .when()
            .get("/api/v1/resources/{id}")
        .then()
            .statusCode(200); // Использует глобальный basic auth
    }

    @Test
    void testLocalAuthOverridesGlobal() {
        given()
            .auth().oauth2("local_bearer_token_123")
            .pathParam("id", 2)
        .when()
            .get("/api/v1/resources/{id}")
        .then()
            .statusCode(200); // Переопределяет глобальный auth
    }
}

// TestNG: Глобальная настройка и изоляция через локальные переопределения
public class AuthTestNG {
    private String authToken;

    @BeforeClass
    public void setup() {
        RestAssured.baseURI = "https://api.prod.com";
        // Имитация получения токена
        authToken = given()
            .contentType(ContentType.JSON)
            .body("{\"user\":\"admin\",\"pass\":\"pass\"}")
            .post("/auth/login")
            .jsonPath().getString("access_token");
        // Глобальная установка для всех методов в классе
        RestAssured.authentication = oauth2(authToken);
    }

    @AfterClass
    public void cleanup() {
        RestAssured.authentication = NONE;
    }

    @Test
    public void testWithGlobalToken() {
        given()
            .queryParam("scope", "read")
        .when()
            .get("/api/v1/data")
        .then()
            .statusCode(200);
    }

    @Test
    public void testWithFormAuthOverride() {
        given()
            .auth().form("service_account", "svc_pass")
        .when()
            .get("/api/v1/internal/config")
        .then()
            .statusCode(200);
    }
}
```

## Best Practices

- Всегда используйте `preemptive().basic()` для внутренних API и микросервисов, чтобы избежать лишнего `401` round-trip и ускорить выполнение тестов.
- Для JWT/OIDC предпочтительнее использовать `.header("Authorization", "Bearer ...")` или кастомный `RequestFilter`, так как встроенный `oauth2()` не поддерживает автоматическое обновление (refresh) токенов.
- При работе с `form()` явно указывайте `FormAuthConfig` с именами полей логина/пароля и URL-эндпоинтом, чтобы избежать зависимости от дефолтных эвристик RA.
- В параллельных тестах избегайте мутации `RestAssured.authentication`. Передавайте учетные данные локально через `given().auth()` или используйте `ThreadLocal<String>` для динамических токенов.
- Всегда сбрасывайте глобальную аутентификацию в `RestAssured.authentication = NONE` в `@AfterAll` / `@AfterClass`, чтобы не допустить утечки credentials между тестовыми классами или сьютами.
- Для API с API-Keys используйте централизованный `Filter` или `RequestSpecBuilder`, добавляющий заголовок `X-API-Key`, вместо дублирования `.header()` в каждом тесте.
- При отладке проблем с `form()` или `basic()` включайте `.log().all()` в фазе `given()`, чтобы проверить, какие заголовки и cookie фактически отправляются на сервер.
- Храните чувствительные данные (пароли, секреты, токены) во внешних конфигурациях (`.env`, Vault, CI/CD secrets), а не в коде тестов. Подставляйте их через `System.getenv()` или DI-контейнеры.

## 6. POJO-интеграция (Фокус на RA-обвязке)

> Rest Assured предоставляет прозрачную интеграцию с механизмами сериализации/десериализации, позволяя работать с Java-объектами (POJO) вместо сырых JSON/XML строк. 
> 
> По умолчанию используется Jackson 2, но конфигурация гибко настраивается под Gson или кастомные фабрики, обеспечивая типобезопасную работу с API контрактами.

### Сериализация запроса (Request)

- `body(pojo)` — автоматическое преобразование Java-объекта в JSON/XML перед отправкой.
- `body(List<T>)` / `body(Set<T>)` / `body(Map<String, T>)` — сериализация коллекций и словарей в соответствующие JSON-структуры.
- `ObjectMapperType.JACKSON_2` — маппер по умолчанию в RA 5.x. Активируется автоматически при наличии `jackson-databind` в classpath.
- Явное указание типа: `body(pojo, ObjectMapperType.JACKSON_2)`.

### Десериализация ответа (Response)

- `response.as(Class<T>)` / `extract().as(Class<T>)` — преобразование тела ответа в указанный класс.
- `extract().path("$.field").as(Class)` — извлечение конкретного поля с последующей конвертацией.
- Ленивая оценка: десериализация происходит только при явном вызове `.as()` или `.path()`. До этого момента ответ хранится в сыром виде (byte[]/String).

### Конфигурация ObjectMapper

- `objectMapperConfig().defaultObjectMapperType(...)` — смена маппера на GSON, Jackson 1 или кастомный.
- `jackson2ObjectMapperFactory((cls, charset) -> new ObjectMapper().disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES))` — гибкая настройка десериализатора через лямбду.
- Поддержка стандартных аннотаций: `@JsonIgnore`, `@JsonProperty`, `@JsonFormat` (Jackson) или `@SerializedName` (Gson).

### Работа с коллекциями и вложенными структурами

- `extract().jsonPath().getList("$.items", UserDto.class)` — маппинг JSON-массива в `List<UserDto>`.
- `response.as(new TypeRef<List<UserDto>>() {})` — безопасное приведение к параметризованным типам с сохранением дженерик-информации.
- `response.jsonPath().getObject("$.data", ResponseWrapper.class)` — извлечение вложенного объекта по JSON-пути.

### Нюансы RA 5.x

- **Ленивая десериализация**: `Response` кэширует сырое тело после первого чтения. Повторные вызовы `.as()` работают с кэшем, что экономит ресурсы CPU.
- **Кэширование Response**: При вызове `.extract().response()` ответ материализуется. Последующие `.asString()`, `.as(Class)` читают из внутреннего буфера, а не повторно парсят сеть.
- `response.prettyPrint()` — удобная отладочная печать без прерывания цепочки валидаций. Работает даже если тело уже было извлечено.

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5: Сериализация и десериализация с кастомным Jackson-конфигом
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class PojoTestJUnit5 {

    @BeforeAll
    static void configureMapper() {
        RestAssured.config = RestAssuredConfig.config()
            .objectMapperConfig(ObjectMapperConfig.objectMapperConfig()
                .jackson2ObjectMapperFactory((clazz, charset) -> {
                    ObjectMapper mapper = new ObjectMapper();
                    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
                    mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
                    return mapper;
                }));
    }

    @Test
    @Order(1)
    void testPojoSerializationAndExtraction() {
        CreateUserRequest request = new CreateUserRequest("alice", "alice@example.com");

        CreateUserResponse response = given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(201)
            .extract()
            .as(CreateUserResponse.class);

        Assertions.assertNotNull(response.getId());
        Assertions.assertEquals("alice", response.getUsername());
    }
}
```

```java
// TestNG: Работа с коллекциями и TypeRef для безопасного приведения типов
public class PojoTestNG {

    @BeforeClass
    public void setupLogging() {
        RestAssured.config = RestAssuredConfig.config()
            .logConfig(LogConfig.logConfig().enablePrettyPrinting(true));
    }

    @Test
    public void testListDeserialization() {
        List<UserDto> users = given()
            .queryParam("role", "admin")
        .when()
            .get("/api/v1/users")
        .then()
            .statusCode(200)
            .extract()
            .response()
            .as(new TypeRef<List<UserDto>>() {});

        Assert.assertFalse(users.isEmpty(), "Admin list should not be empty");
        users.forEach(u -> Assert.assertNotNull(u.getEmail()));
    }

    @Test
    public void testNestedJsonPathExtraction() {
        String metadataJson = given()
        .when()
            .get("/api/v1/dashboard/stats")
        .then()
            .statusCode(200)
            .extract()
            .jsonPath()
            .getObject("$.metadata", String.class);

        System.out.println("Raw metadata: " + metadataJson);
        Assert.assertTrue(metadataJson.contains("generated_at"));
    }
}
```

## Best Practices

- Всегда настраивайте `FAIL_ON_UNKNOWN_PROPERTIES = false` в `ObjectMapperFactory`, чтобы тесты не падали при добавлении новых полей в ответ сервера (backward compatibility).
- Используйте `TypeRef<T>` вместо сырых `List.class` при десериализации коллекций для сохранения дженерик-информации и избежания `ClassCastException` на этапе рантайма.
- Отдавайте предпочтение `extract().as(Class)` вместо ручного парсинга через `response.asString()` + сторонний JSON-парсер, чтобы использовать встроенный кэш RA и избежать двойного десериализования.
- При работе с `@JsonIgnore` или кастомными сериализаторами убедитесь, что аннотации совместимы с выбранным `ObjectMapperType` и не конфликтуют с версией библиотеки.
- Не вызывайте `.asString()` перед `.as(Class)` без необходимости: это принудительно материализует ответ в String, что может увеличить потребление памяти при работе с большими payload.
- Используйте `response.prettyPrint()` только в отладочных режимах или связывайте с `.log().ifValidationFails()`, чтобы не зашумлять логи CI/CD пайплайнов.
- Для DTO-классов применяйте Java Records или Lombok `@Data`/`@Builder`, но следите за тем, чтобы конструкторы и геттеры соответствовали требованиям Jackson/Gson (публичные поля или аксессоры).
- Изолируйте конфигурацию `ObjectMapper` в базовом `@BeforeAll`/`@BeforeClass` или через кастомный `RequestFilter`, чтобы избежать дублирования логики инициализации в каждом тестовом классе.

## 7. Обработка и извлечение ответа (Response / ExtractableResponse)

> Интерфейсы `Response` и `ExtractableResponse` предоставляют унифицированный доступ к метаданным HTTP-ответа, заголовкам, куки и телу. 
> 
> Rest Assured абстрагирует низкоуровневую обработку потоков `InputStream`, позволяя извлекать данные декларативно через JSON-пути, XPath или прямую десериализацию в Java-объекты без ручного парсинга.

---

### Статус и Мета-данные
- `statusCode()` — возвращает числовой HTTP-статус (int).
- `statusLine()` — возвращает полную строку статуса (String), например `HTTP/1.1 200 OK`.
- `time()` — время выполнения запроса в миллисекундах (long).
- `timeIn(TimeUnit unit)` — конвертация времени в заданную единицу измерения (`SECONDS`, `MINUTES` и т.д.).

```java
Response response = given().when().get("/api/v1/status");
int code = response.statusCode();
String line = response.statusLine();
long millis = response.time();
long seconds = response.timeIn(TimeUnit.SECONDS);
```

---

### Заголовки и Cookies
- `header(String name)` / `headers()` — получение одного или всех заголовков ответа (`Headers` объект).
- `cookie(String name)` / `cookies()` — извлечение куки по имени или всех сразу (`Map<String, String>`).
- `contentAsBytes()` / `asByteArray()` — получение сырого тела ответа для бинарной обработки или хэширования.

```java
String contentType = response.header("Content-Type");
Map<String, String> allHeaders = response.headers().asMap();
String sessionId = response.cookie("JSESSIONID");
byte[] binaryData = response.asByteArray();
```

---

### Извлечение данных (Extract)
- `extract().response()` — возвращает `Response` для дальнейшей работы вне цепочки валидации.
- `extract().path("json.expression")` — извлечение значения по JSON/XPath выражению.
- `extract().jsonPath()` / `extract().xmlPath()` — возврат объекта парсера для множественных запросов к телу без повторного чтения сетевого потока.
- `extract().asString()` — получение тела ответа в виде строки (UTF-8 по умолчанию).

```java
// Извлечение через path
String token = given().post("/login").then().extract().path("access_token");

// Повторное использование JsonPath
JsonPath json = given().get("/api/data").then().extract().jsonPath();
String name = json.getString("user.name");
int age = json.getInt("user.age");
```

---

### Типизация на лету

RA автоматически определяет тип возвращаемого значения при использовании `.path()` или `.as()`, поддерживая маппинг в Map, List, POJO и примитивы.

- Примитивы/Обёртки: `.path("$.id").as(int.class)`, `.as(Boolean.class)`
- Коллекции: `.path("$.tags").as(List.class)`
- Словари: `.path("$.metadata").as(Map.class)`
- POJO: `.as(UserDto.class)` (требует настроенного ObjectMapper)

```java
// Автоматический вывод типа
List<String> tags = given().get("/api/post/1").then().extract().path("tags");
boolean isActive = given().get("/api/user/5").then().extract().path("active");
```

---

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5: Типобезопасное извлечение и проверка метаданных
@Test
void testResponseExtractionJunit5() {
    ExtractableResponse<Response> response = given()
        .pathParam("id", 42)
    .when()
        .get("/api/v1/users/{id}")
    .then()
        .extract();

    Assertions.assertEquals(200, response.statusCode());
    Assertions.assertTrue(response.timeIn(TimeUnit.MILLISECONDS) < 1000L);
    Assertions.assertEquals("application/json", response.header("Content-Type"));

    UserDto user = response.as(UserDto.class);
    Assertions.assertEquals(42, user.getId());
    Assertions.assertNotNull(user.getEmail());
}
```

```java
// TestNG: Работа с JsonPath, Cookies и примитивами
@Test
public void testResponseExtractionTestNg() {
    Response res = given().get("/api/v1/session");

    String sessionId = res.cookie("session_id");
    Assert.assertNotNull(sessionId, "Session ID must be present");

    JsonPath jp = res.jsonPath();
    String login = jp.getString("data.user.login");
    int loginCount = jp.getInt("data.user.login_count");

    Assert.assertTrue(loginCount > 0, "Login count should be positive");
    Assert.assertEquals(login, "test_user");
}
```

---

## Best Practices

- Всегда используйте `ExtractableResponse` вместо сырого `Response`, когда нужно прервать цепочку `then()` и сохранить результат для последующих проверок или передачи в другие шаги теста.
- Избегайте вызова `.asString()` или `.asByteArray()` перед валидацией статуса или заголовков: чтение тела материализует ответ в память, что может скрыть ошибки раннего завершения соединения или таймауты.
- Для множественных извлечений из одного ответа вызывайте `.jsonPath()` или `.xmlPath()` один раз и переиспользуйте полученный объект. Повторный вызов `extract().path()` инициирует повторный парсинг буфера, что снижает производительность.
- При работе с `extract().path()` RA пытается автоматически привести тип. Если путь указывает на массив, используйте `.as(List.class)`, а не `.as(String.class)`, чтобы избежать `ClassCastException`.
- Используйте `timeIn(TimeUnit)` вместо ручного деления `time() / 1000` для точных измерений производительности и избегания потери данных из-за целочисленного округления.
- При извлечении куки или заголовков используйте `Objects.requireNonNullElse()` или комбинированные матчеры, так как отсутствующие заголовки возвращают `null` без выброса исключений, что может привести к `NullPointerException` в последующей логике.
- Для бинарных файлов (изображения, PDF, архивы) используйте `contentAsBytes()` или `response.downloadTo(new File("path"))`, а не `asString()`, чтобы избежать повреждения данных из-за некорректного декодирования в UTF-8.
- Изолируйте извлечение данных от бизнес-логики: создавайте вспомогательные методы `extractUserId(response)`, `extractToken(response)`, чтобы сохранить тесты читаемыми и соответствующими принципу единственной ответственности.
- При параллельном запуске тестов не кэшируйте объекты `Response` в статических полях. Каждый HTTP-вызов должен обрабатываться в рамках своего потока, так как `Response` содержит внутренние буферы и указатели потоков.

## 8. Валидация

> Механизм валидации ответов в Rest Assured построен на глубокой интеграции с библиотекой Hamcrest. 
> 
> Он позволяет декларативно описывать ожидаемые статус-коды, заголовки и содержимое тела ответа, используя читаемые матчеры и цепочечные вызовы. 
> 
> При несоответствии контракту автоматически генерируются детализированные исключения с указанием ожидаемых и фактических значений.

### Цепочки валидации: `.and()` и `.assertThat()`

- `.and()` — опциональный разделитель для улучшения читаемости длинных цепочек проверок. Не влияет на логику выполнения, служит только синтаксическим сахаром.
- `.assertThat()` — явное начало блока валидаций. Рекомендуется использовать для разделения фаз извлечения и проверки, а также для интеграции с кастомными Hamcrest-матчерами.

### Статус-коды и Заголовки

- `statusCode(is(200))` / `statusCode(equalTo(201))` — проверка HTTP-статуса.
- `statusLine(containsString("OK"))` — валидация полной строки статуса.
- `header("Content-Type", containsString("application/json"))` — проверка заголовка.
- `headers("X-Req-Id", not(emptyOrNullString()))` — проверка на null и empty.

```java
given()
    .when()
        .get("/api/v1/status")
    .then()
        .assertThat()
        .statusCode(is(200))
        .and()
        .header("X-Response-Time", not(emptyOrNullString()));
```

### Тело + JsonPath

- `body("$.field", equalTo("value"))` — точное совпадение.
- `body("$.list", hasSize(3))` — проверка размера коллекции.
- `body("$.users.age", everyItem(greaterThan(18)))` — проверка каждого элемента списка.
- `body("data.items[0].id", notNullValue())` — проверка на null.
- Комбинирование условий: `body("id", equalTo(1), "name", startsWith("Admin"))`.

### JSON Schema Validation

- `body(matchesJsonSchemaInClasspath("schema.json"))` — валидация структуры ответа на JSON Schema.
- Требует зависимости `io.rest-assured:json-schema-validator`.
- Поддержка внешних файлов, URL или строкового представления схемы.

### Множественные проверки и Кастомные матчеры

- RA позволяет перечислять несколько path-matcher пар в одном вызове `body()`.
- Кастомные матчеры интегрируются через `assertThat(actualValue, customMatcher)`, часто используемые после `extract().path()`.
- `TypeSafeMatcher<T>` из Hamcrest позволяет создать строгую типизированную проверку с кастомным описанием ошибки.

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5: Комплексная валидация с JsonPath и JSON Schema
@Test
void testValidationChainJUnit5() {
    given()
        .queryParam("role", "admin")
    .when()
        .get("/api/v1/users")
    .then()
        .assertThat()
        .statusCode(is(200))
        .and()
        .header("Content-Type", containsString("application/json"))
        .and()
        .body("$", hasKey("data"))
        .body("data", hasSize(greaterThan(0)))
        .body("data[0].id", notNullValue())
        .body(matchesJsonSchemaInClasspath("schemas/user-list-schema.json"))
        .time(lessThan(2000L), TimeUnit.MILLISECONDS);
}
```

```java
// TestNG: Множественные проверки и кастомный Hamcrest-матчер
@Test
public void testValidationChainTestNg() {
    Response response = given().get("/api/v1/config").then().extract().response();
    String configVersion = response.jsonPath().getString("version");

    // Кастомный матчер для проверки формата версии (SemVer)
    Matcher<String> semVerMatcher = new TypeSafeMatcher<String>() {
        
        @Override
        protected boolean matchesSafely(String version) {
            return version.matches("\\d+\\.\\d+\\.\\d+");
        }

        @Override
        public void describeTo(Description description) {
            description.appendText("a valid Semantic Version string");
        }
    };

    // Интеграция с Hamcrest
    org.hamcrest.MatcherAssert.assertThat("Config version must be valid SemVer", configVersion, semVerMatcher);

    // Валидация в стиле RA
    given()
        .get("/api/v1/config")
    .then()
        .assertThat()
        .statusCode(equalTo(200))
        .body("features[0].enabled", is(true))
        .body("features", hasSize(greaterThanOrEqualTo(3)))
        .body(matchesJsonSchemaInClasspath("schemas/config-schema.json"));
}
```

## Best Practices

- Всегда используйте `.assertThat()` перед первой проверкой для явного отделения фазы выполнения от фазы валидации. Это улучшает читаемость стека вызовов при падении теста.
- Группируйте проверки логически: сначала статус-коды и заголовки, затем структура тела, и только потом содержимое полей. Это ускоряет отладку, так как ошибки структуры не будут маскировать ошибки данных.
- Для JSON-схем используйте `matchesJsonSchemaInClasspath()` с вынесенными в отдельный ресурс `.json` файлами. Не храните схемы прямо в коде теста.
- При проверке списков избегайте хардкода индексов (`body("items[0].id")`) для бизнес-логики. Используйте `hasItem()`, `everyItem()` или `collectionItemIn()` для устойчивости к изменению порядка.
- Кастомные матчеры создавайте в отдельных утилитарных классах (`MatchersUtils`), чтобы переиспользовать их между тестами и поддерживать принцип DRY.
- Не смешивайте валидацию `body("$.path", matcher)` с извлечением `extract().path()` в одной цепочке `then()`. Валидация должна завершаться до вызова `extract()`, иначе RA может материализовать ответ раньше времени.
- Для числовых сравнений всегда используйте `is()`, `greaterThan()`, `closeTo()` из `org.hamcrest.Matchers`, а не операторы `==` или `>`, чтобы получать детализированные сообщения об ошибках.
- При работе с `matchesJsonSchemaInClasspath()` убедитесь, что схема совместима с версией спецификации (Draft 4/6/7), используемой в зависимости `json-schema-validator`. Для кастомных валидаторов используйте `SchemaFactory` напрямую.
- В параллельных тестах избегайте мутации глобальных статических матчеров. Все проверки должны быть stateless или инициализироваться локально в методе теста.

## 9. Спецификации и Фильтры

> Спецификации (Specifications) и Фильтры (Filters) предоставляют мощные механизмы повторного использования конфигурации запросов/ответов и перехвата сетевого трафика. 
> 
> Они позволяют централизовать базовые настройки, внедрять кастомную логику обработки и логирования, сохраняя код тестов чистым и соответствующим принципам DRY.

### RequestSpecBuilder / ResponseSpecBuilder

- `RequestSpecBuilder` — билдер для создания `RequestSpecification`. Поддерживает: 
  - `.setBaseUri()` 
  - `.setBasePath()`
  - `.addHeader()` 
  - `.setContentType()`
  - `.addQueryParam()`
  - `.setAuth()`.
- `ResponseSpecBuilder` — билдер для создания `ResponseSpecification`. Позволяет задать ожидаемые значения по умолчанию: 
  - `.expect().statusCode(200)`
  - `.expect().contentType(ContentType.JSON)` 
  - `.expect().header("Server", "nginx")`.
- `.build()` — финализирует построение и возвращает готовую спецификацию.

```java
RequestSpecification apiSpec = new RequestSpecBuilder()
    .setBaseUri("https://api.staging.com")
    .setBasePath("/v2")
    .setContentType(ContentType.JSON)
    .addHeader("X-Client-Id", "test-client")
    .addQueryParam("version", "2.0")
    .build();

ResponseSpecification successSpec = new ResponseSpecBuilder()
    .expectStatusCode(200)
    .expectContentType(ContentType.JSON)
    .expectHeader("X-Request-Id", not(emptyString()))
    .build();
```

### Применение спецификаций

- `.spec(spec)` — применение к конкретному запросу. Переопределяет конфликтующие параметры из глобальных настроек.
- `RestAssured.requestSpecification = spec` / `RestAssured.responseSpecification = spec` — глобальное применение ко всем запросам в текущей JVM.

```java
given()
    .spec(apiSpec)
    .pathParam("id", 10)
.when()
    .get("/users/{id}")
.then()
    .spec(successSpec);
```

### Наследование и слияние

- Спецификации наследуются и объединяются. Параметры из `.spec()` мержатся с локальными вызовами `given()`.
- `.no()` — метод-префикс, отменяющий применение спецификации или сбрасывающий её параметры. Например, `no().basePath()` или `no().headers()`.
- При конфликте локальные параметры в `given()` всегда приоритетнее параметров из `.spec(spec)`.

```java
given()
    .spec(apiSpec) // содержит basePath="/v2"
    .no().basePath() // отменяет наследование basePath
    .queryParam("debug", true)
.when()
    .get("/health"); // запрос уйдет на корень baseURI, без /v2
```

- `.basePath("/new")` - Переопределяет значение (если было в spec)
- `.no().basePath()` - Сбрасывает унаследованное значение
- `.no().basePath().basePath("/new")` - Сброс + новое значение

> Используйте `.no().something()` когда нужно полностью отказаться от унаследованного параметра, а не просто переопределить его. 
> 
> Для простой замены достаточно `.basePath("/new") без .no()`.

### Фильтры (Filter Interface)

- `Filter` — интерфейс для перехвата запроса/ответа до и после отправки. Метод `Response filter(FilterableRequestSpecification requestSpec, FilterableResponseSpecification responseSpec, FilterContext ctx)`.
- `RequestFilter` / `ResponseFilter` — функциональные интерфейсы (SAM) для лаконичной записи логики только до отправки или после получения.
- `LoggingFilter` — встроенный фильтр для логирования всего запроса/ответа (альтернатива `.log().all()`).
- Порядок выполнения: фильтры применяются в порядке добавления. Цепочка: Filter 1 (Request) -> Filter 2 (Request) -> HTTP Call -> Filter 2 (Response) -> Filter 1 (Response).
- `.filter(Filter)` / `.filters(Collection<Filter>)` — добавление фильтров на уровень запроса или глобально (на все запросы во всех тестах) через `RestAssured.filters()`.
  ```java
  // 1. Глобальные применяются ко всем запросам
  RestAssured.filters(filter1, filter2);

  // 2. Локальные ДОБАВЛЯЮТСЯ к глобальным (не заменяют)
  given().filter(localFilter).get("/test"); // filter1, filter2, localFilter

  // 3. Очистка глобальных фильтров
  RestAssured.filters(); // пустой список

  // 4. Порядок: сначала глобальные, потом локальные
  RestAssured.filters(globalA, globalB);
  given().filter(localC).get("/test"); // globalA -> globalB -> localC
  ```

Примеры:

```java
// Кастомный RequestFilter для добавления динамического заголовка
RequestFilter dynamicHeaderFilter = (requestSpec, responseSpec, ctx) -> {
    requestSpec.header("X-Trace-ID", UUID.randomUUID().toString());
    return requestSpec;
};

// ResponseFilter для автоматического извлечения и логирования времени ответа
ResponseFilter perfLogger = (responseSpec, ctx) -> {
    long elapsed = responseSpec.getTime();
    if (elapsed > 1000) {
        System.out.println("WARNING: Slow response detected: " + elapsed + "ms");
    }
};

given()
    .filter(new LoggingFilter(true))
    .filter(dynamicHeaderFilter)
    .filter(perfLogger)
.when()
    .get("/api/slow-endpoint");
```

### Примеры интеграции (JUnit 5 vs TestNG)

```java
// JUnit 5: Изоляция спеков и кастомный Filter
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class SpecFilterTestJUnit5 {
    
    private RequestSpecification sharedReqSpec;
    private ResponseSpecification sharedRespSpec;

    @BeforeAll
    static void init() {
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }

    @BeforeEach
    void setupSpecs() {
        sharedReqSpec = new RequestSpecBuilder()
            .setBaseUri("https://api.test.local")
            .addHeader("Accept", "application/json")
            .build();
        sharedRespSpec = new ResponseSpecBuilder()
            .expectStatusCode(200)
            .build();
    }

    @Test
    @Order(1)
    void testSpecAndFilterApplication() {
        RequestFilter authFilter = (req, res, ctx) -> {
            req.header("Authorization", "Bearer test-token-" + System.currentTimeMillis());
            return req;
        };

        given()
            .spec(sharedReqSpec)
            .filter(authFilter)
        .when()
            .get("/v1/resource")
        .then()
            .spec(sharedRespSpec)
            .body("status", equalTo("OK"));
    }
}
```

```java
// TestNG: Глобальные фильтры и наследование спеков с .no()
public class SpecFilterTestNG {
    
    private RequestSpecification baseSpec;

    @BeforeClass
    public void initGlobalConfig() {
        baseSpec = new RequestSpecBuilder()
            .setBaseUri("https://api.prod.local")
            .setBasePath("/api/v2")
            .setContentType(ContentType.JSON)
            .addHeader("X-Env", "qa")
            .build();

        RestAssured.filters(
            new LoggingFilter(false),
            (req, res, ctx) -> {
                req.queryParam("timestamp", System.currentTimeMillis());
                return req;
            }
        );
    }

    @AfterClass
    public void cleanup() {
        RestAssured.reset();
    }

    @Test
    public void testSpecOverrideAndNoInheritance() {
        given()
            .spec(baseSpec) // наследует basePath="/api/v2"
            .no().basePath() // отменяет его для этого запроса
            .queryParam("sync", true)
        .when()
            .get("/legacy/endpoint") // запрос уйдет на корень
        .then()
            .statusCode(200);
    }
}
```

## Best Practices

- Используйте `RequestSpecBuilder` / `ResponseSpecBuilder` для выноса повторяющихся заголовков, базовых URI и ожидаемых статусов в отдельные конфигурационные классы.
- Применяйте `.no()` осторожно: он сбрасывает параметр на уровень по умолчанию, а не удаляет его. Для полной очистки используйте `new RequestSpecBuilder().build()`.
- Фильтры выполняются строго в порядке добавления. Помещайте `LoggingFilter` в конец цепочки или используйте `.log().all()`, чтобы видеть финальный запрос после всех модификаций другими фильтрами.
- Для динамической аутентификации или генерации nonce/timestamp используйте `RequestFilter`, а не хардкод в `@BeforeMethod`.
- `ResponseSpecBuilder` идеально подходит для контрактного тестирования: задавайте ожидаемые заголовки и статусы один раз, чтобы тесты падали сразу при отклонении API от SLA.
- При работе с `.spec()` помните: локальные вызовы `given()` (`header()`, `queryParam()`) всегда переопределяют спецификацию. Если нужно добавить, а не заменить, используйте кастомную логику слияния или билдеры.
- Отключайте `LoggingFilter` в production/CI пайплайнах или используйте `.log().ifValidationFails()` для экономии дискового пространства и ускорения вывода логов.
- Избегайте мутации статических `RestAssured.filters()` в параллельных тестах. Передавайте фильтры локально через `.filter()` или используйте `ThreadLocal<RequestSpecification>`.
  Если тесты могут выполняться параллельно, никогда не используйте глобальные RestAssured.filters() - только локальные через .filter() или изолированные спецификации.
  ```java
  // Для параллельных тестов:
  
  // Для большинства случаев: локальные фильтры
  given().filter(myFilter).get("/api");

  // Для повторного использования: спецификации (не глобальные)
  RequestSpecification spec = new RequestSpecBuilder()
      .addFilter(myFilter)
      .build();

  given().spec(spec).get("/api");
  ```

  Использование спецификаций вместо глобальных фильтров:

  ```java
  // Создаем разные спецификации для разных сценариев
  class TestSpecs {

      static RequestSpecification authSpec = new RequestSpecBuilder()
          .addFilter(new AuthFilter())
          .build();

      static RequestSpecification loggingSpec = new RequestSpecBuilder()
          .addFilter(new LoggingFilter())
          .build();
    
      static RequestSpecification retrySpec = new RequestSpecBuilder()
          .addFilter(new RetryFilter())
          .build();
  }

  // Использование в параллельных тестах
  @Test
  void parallelTest1() {
      given()
          .spec(TestSpecs.authSpec) // ✅ каждый поток использует свою
          .get("/user/1");
  }

  @Test
  void parallelTest2() {
      given()
          .spec(TestSpecs.loggingSpec) // ✅ независимые объекты
          .get("/user/2");
  }
  ```

