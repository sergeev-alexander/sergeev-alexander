# Rest Assured

## Содержание

1. [Введение и Подключение](#1-введение-и-подключение)
2. [Ядро DSL и Статические импорты](#2-ядро-dsl-и-статические-импорты)
3. [Глобальная конфигурация (RestAssured.* и RestAssuredConfig)](#3-глобальная-конфигурация-restassured-и-restassuredconfig)
4. [Конструктор запроса (RequestSpecification)](#4-конструктор-запроса-requestspecification)
5. [Аутентификация (AuthConfig)](#5-аутентификация-authconfig)
6. [POJO-интеграция (Фокус на RA-обвязке)](#6-pojo-интеграция-фокус-на-ra-обвязке)
7. [Обработка и извлечение ответа (Response / ExtractableResponse)](#7-обработка-и-извлечение-ответа-response--extractableresponse)
8. [Валидация (Интеграция с вашими шпаргалками)](#8-валидация-интеграция-с-вашими-шпаргалками)
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

