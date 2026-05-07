**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# JDI Dark

## Содержание

1. [Введение](#1-введение)
2. [Быстрый старт](#2-быстрый-старт)
3. [Основные абстракции](#3-основные-абстракции)
4. [HTTP-методы и параметры](#4-http-методы-и-параметры)
5. [Данные и сериализация](#5-данные-и-сериализация)
6. [Заголовки и авторизация](#6-заголовки-и-авторизация)
7. [Валидация и Asserts](#7-валидация-и-asserts)
8. [Интеграция с экосистемой](#8-интеграция-с-экосистемой)
9. [Продвинутые возможности](#9-продвинутые-возможности)

---

# 1. Введение

> JDI Dark — это декларативный, типобезопасный HTTP-клиент для Java, созданный специально для упрощения написания,
> поддержки и масштабирования API-тестов.
>
> Он абстрагирует низкоуровневую работу с сетью, предоставляя удобный DSL на базе аннотаций и POJO.

## Ключевые отличия от аналогов

> **JDI Dark** — использует декларативный подход через аннотации (`@Endpoint`, `@Path`, `@Query`),
> автоматически сериализует/десериализует данные, валидирует ответы встроенными ассертами и генерирует детальные логи
> без дополнительных настроек.

| Параметр             | JDI Dark                       | RestAssured                          | Java HTTP Client            |
|:---------------------|--------------------------------|--------------------------------------|-----------------------------|
| Стиль написания      | Декларативный (`@Endpoint`)    | Императивный (Builder-цепочки)       | Императивный (ручной)       |
| Маппинг JSON ↔ POJO  | Автоматический (Jackson/Gson)  | Ручной через `.extract().as()`       | Ручной через `ObjectMapper` |
| Валидация ответов    | Встроенные `Assert`s + матчеры | `.body()` + Hamcrest                 | Ручные `assertEquals`       |
| Логирование & Allure | Готово из коробки, авто-аттачи | Требует слушателей и кастомных хуков | Отсутствует                 |

## Когда выбирать JDI Dark

1. **Командная разработка API-тестов** — декларативный API гарантирует единый стиль контрактов, упрощает ревью и снижает
   порог входа для новых инженеров.
2. **Сложные сценарии оркестрации** — встроенная поддержка интерцепторов, динамических токенов и параметризированных
   запросов ускоряет интеграцию с OAuth2, GraphQL и legacy-сервисами.
3. **Быстрый онбординг и CI/CD** — минимальный бойлерплейт, встроенное логирование и нативная поддержка Allure/JUnit 5
   сокращают время настройки пайплайнов.

## Когда стоит отказаться

- Если проект требует прямого контроля над `Socket`-соединениями, WebSocket или gRPC.
- При миграции огромной кодовой базы с RestAssured, где рефакторинг займёт больше времени, чем принятие рисков текущей
  архитектуры.
- Если используется ограниченный runtime без поддержки reflection (JDI Dark активно использует аннотации для маппинга).

---

# 2. Быстрый старт

> Раздел посвящён первичной настройке проекта, добавлению зависимостей и базовой конфигурации JDI Dark для быстрого запуска API-тестов.
>
> Все примеры ниже используют актуальную стабильную ветку `2.4.x` и предполагают наличие JDK 11+.

## Подключение зависимостей

- **Maven** — стандартный выбор для Java-проектов, управляется через `pom.xml`
- **Gradle** — современный инструмент сборки, удобен для мульти-модульных проектов

### Maven (pom.xml):

```xml
<dependencies>
    <!-- Ядро JDI Dark -->
    <dependency>
        <groupId>com.epam.jdi</groupId>
        <artifactId>jdi-dark</artifactId>
        <version>2.4.1</version>
        <scope>test</scope>
    </dependency>

    <!-- Jackson для автоматической сериализации JSON ↔ POJO -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle (build.gradle.kts):

```kotlin
dependencies {
    // Ядро фреймворка
    testImplementation("com.epam.jdi:jdi-dark:2.4.1")
    // Библиотека сериализации
    testImplementation("com.fasterxml.jackson.core:jackson-databind:2.15.2")
}
```

## Минимальная конфигурация клиента и базового URL

- `JDIHttpClient` — основной класс для отправки запросов, поддерживает пул соединений
- `JDISettings` — глобальный реестр настроек (таймауты, заголовки, логи)

```java
import com.epam.jdi.http.JDIHttpClient;
import com.epam.jdi.http.settings.JDISettings;

public class ApiConfig {
    // Базовый URL вынесен в константу для переиспользования
    private static final String BASE_URL = "https://api.example.com/v1";

    // Глобальный экземпляр клиента
    public static final JDIHttpClient CLIENT = new JDIHttpClient(BASE_URL);
    // Статическое поле безопасно в многопоточной среде при условии, что: 
    // 1. настройки конфигурации устанавливаются только в static-блоке
    // 2. в рантайме конфигурация не меняется

    static {
        // Установка глобального таймаута подключения и чтения (мс)
        JDISettings.settings().setConnectionTimeout(5000);
        JDISettings.settings().setReadTimeout(10000);

        // Включение полного логирования запросов/ответов в консоль
        JDISettings.settings().setLogAll(true);

        // Добавление заголовка по умолчанию для всех исходящих запросов
        JDISettings.settings().addDefaultHeader("Accept", "application/json");
        JDISettings.settings().addDefaultHeader("User-Agent", "JDI-Dark-Tests/1.0");
    }

    // Геттер для получения пред-настроенного клиента
    public static JDIHttpClient getClient() {
        return CLIENT;
    }
}
```

## Типовая структура тестового проекта

- Разделение по слоям упрощает навигацию и масштабирование
- Контракты API изолированы от тестовой логики

```text
src/
└── test/
    ├── java/
    │   └── com.yourproject.api/
    │       ├── client/          # Конфигурации, интерцепторы, кастомные клиенты
    │       │   └── ApiConfig.java
    │       ├── models/          # DTO для сериализации (POJO)
    │       │   ├── request/     # Объекты тела запросов
    │       │   └── response/    # Объекты тела ответов
    │       ├── endpoints/       # Декларативные классы с @Endpoint
    │       │   └── UserEndpoint.java
    │       └── tests/           # Тестовые классы (JUnit 5 / TestNG)
    │           └── UserApiTest.java
    └── resources/
        ├── test.properties      # URL, креды, флаги окружения
        └── allure.properties    # Настройки генерации отчётов
```

## Best Practices

- **Используйте `scope=test`** — зависимости JDI Dark предназначены только для тестового кода, не включайте их в продакшн-артефакты
- **Храните `BASE_URL` в `*.properties`** — выносите URL и чувствительные данные во внешние файлы, используйте `System.getenv()`
  для безопасной работы в CI/CD пайплайнах
- **Один клиент на сьют** — переиспользуйте один экземпляр `JDIHttpClient` через `static` блок или `@BeforeAll`,
  чтобы не пересоздавать `ConnectionPool` и не тратить ресурсы на handshake
- **Отключайте `setLogAll(true)` в CI** — подробные логи занимают много места в отчётах Allure,
  оставляйте их только для локальной отладки, в CI используйте `JDISettings.settings().setLogLevel(LogLevel.WARN)`
- Используйте `JDIHttpClient` как единый инстанс в рамках тестового сьюта, чтобы переиспользовать `ConnectionPool`
  и глобальные конфигурации.
- Выносите базовые URL, таймауты и credentials во внешние `*.properties` или environment-переменные, не хардкодьте их в аннотациях.
- Держите `@Endpoint` классы чистыми от бизнес-логики — они должны описывать только контракт API,
  а валидацию и ассерты выносите в отдельные `@Test` методы.

---

# 3. Основные абстракции

> JDI Dark использует аннотации для декларативного описания HTTP-запросов, автоматически преобразуя Java-интерфейсы в готовые HTTP-вызовы.
>
> Этот подход отделяет контракт API от тестовой логики, упрощает рефакторинг, обеспечивает типобезопасность и снижает дублирование кода.

## Декларативные аннотации

Аннотации в JDI Dark делятся на две категории: **интерфейсного уровня** (применяются ко всему сервису)
и **методного уровня** (переопределяют или дополняют настройки для конкретного эндпоинта).

## Аннотации интерфейсного уровня

Применяются к интерфейсу, помеченному `@Service`. Задают глобальную конфигурацию для всех запросов сервиса.

- `@Service` — базовая аннотация, регистрирующая интерфейс как источник декларативных эндпоинтов. Фреймворк сканирует только помеченные интерфейсы при инициализации.
  - `value` / `name` — логическое имя сервиса для логирования и отладки
  - `url` — переопределение базового URL (если не используется `@ServiceDomain`)
  - `headers` — массив `@Header`, применяемых ко всем запросам сервиса
  - `connectionTimeout` / `readTimeout` — таймауты для этого сервиса

- `@ServiceDomain` — задаёт базовый URI сервиса.

  Поддерживает подстановку значений из `test.properties` через синтаксис `${key}`.

  Позволяет легко переключаться между окружениями (dev/stage/prod) без изменения кода.
    ```java
    @ServiceDomain("${api.base}") // из test.properties: api.base=https://api.example.com
    ```
- `@QueryParameter` — декларативно добавляет query-параметр ко всем запросам сервиса.

  Можно указывать несколько раз или использовать контейнер `@QueryParameters`.
    ```java
    @QueryParameter(name = "api_version", value = "v2") // добавит ?api_version=v2 ко всем запросам
    @QueryParameters({
        @QueryParameter(name = "locale", value = "ru"),
        @QueryParameter(name = "timezone", value = "UTC+3")
    })
    ```

- `@Header` (на интерфейсе) — глобальный заголовок для всех методов сервиса.

  Удобно для `Accept`, `X-Client-ID`, `User-Agent`.
    ```java
    @Header(name = "Accept", value = "application/json")
    @Header(name = "X-Client", value = "mobile-app-v2")
    ```

- `@Cookie` — добавляет cookie ко всем запросам сервиса. Значение может резолвиться из переменных окружения.
    ```java
    @Cookie(name = "session_id", value = "${SESSION_ID}")
    ```

---

## Аннотации методного уровня

Применяются к методам внутри `@Service`-интерфейса. Переопределяют или дополняют настройки интерфейсного уровня.

- `@Endpoint` — определяет путь ресурса относительно `@ServiceDomain`. Применяется непосредственно к методу интерфейса.
    ```java
    @Endpoint("/users") // базовый путь: /users
    ```
- `@Path` — задаёт динамическую часть URL с поддержкой плейсхолдеров `{param}`. Значение подставляется из аргумента метода с аннотацией `@Path("param")`.
    ```java
    @Endpoint("/{userId}/orders") // при userId=42 → /users/42/orders
    ```
- `@Query` — добавляет параметр в строку запроса. Поддерживает плейсхолдеры и резолвинг из конфига.
    ```java
    @Query("details") // ?details={значение_аргумента}
    @Query("api_key=${API_KEY}") // ?api_key=значение_из_переменной_окружения
    ```
- `@QueryParameter` (на методе) — локальное дополнение или переопределение интерфейсных query-параметров.
    ```java
    @QueryParameter(name = "filter", value = "active") // добавит ?filter=active только к этому методу
    ```
- `@Header` (на методе) — локальный заголовок, имеет приоритет над интерфейсным. Позволяет передавать кастомные токены или флаги трассировки.
    ```java
    @Header(name = "Authorization", value = "Bearer ${auth_token}")
    ```
- `@Form` — флаг на методе, указывающий, что запрос использует `application/x-www-form-urlencoded`. Параметры метода с `@FormParameter` будут сериализованы в форму.
- `@FormParameter` — помечает аргумент метода как поле формы. Имя поля задаётся в `name`.
    ```java
    @FormParameter(name = "username") String login
    ```
- `@FormParameters` — контейнер для группировки нескольких форм-параметров (альтернатива повторению аннотации).
- `@Body` — определяет тело запроса. Поддерживает четыре режима:
  - **POJO-объект** (автосериализация в JSON через Jackson/Gson):
      ```java
      @Body UserCreateRequest newUser // → {"name":"Alex","email":"a@b.com"}
      ```
  - **Сырая строка** (для кастомного JSON или XML):
      ```java
      @Body String rawPayload // ← "{\"name\":\"Ivan\"}"
      ```
  - **Шаблон с плейсхолдерами** (значения подставляются из аргументов метода):
      ```java
      @Body("{\"name\": \"{userName}\", \"email\": \"{userEmail}\"}")
      // при вызове: create("{\"name\": \"{userName}\"}", "Alex", "a@b.com")
      ```
  - **Файл** (для `multipart` или бинарных запросов):
      ```java
      @Body File image // отправляется как application/octet-stream или multipart
      ```
- `@Multipart` — флаг на методе, указывающий, что запрос будет `multipart/form-data`. Автоматически выставляет `Content-Type` с `boundary`.
  - `@FilePart` — бинарная часть multipart-запроса (файл). Требует указания `controlName` и `fileName`.
      ```java
      @FilePart(controlName = "avatar", fileName = "photo.jpg") File image
      ```
  - `@Part` — текстовая часть multipart-запроса (метаданные, поля формы).
      ```java
      @Part(name = "description") String desc
      ```
- `@Timeout` — переопределение таймаутов для конкретного запроса (в миллисекундах).
    ```java
    @Timeout(connection = 5000, read = 30000)
    ```
- `@ContentType` — принудительно задаёт `Content-Type` (обычно определяется автоматически по контексту: `@Body POJO` → `application/json`).
    ```java
    @ContentType("application/xml") // для SOAP или legacy-API
    ```
- `@RetryOnFailure` — декларативная настройка повторных попыток при сетевых ошибках или определённых статус-кодах.
    ```java
    @RetryOnFailure(maxAttempts = 3, statusCodes = {502, 503, 504})
    ```

---

## Пример конфигурации интерфейса

```java
// @Service регистрирует интерфейс как источник эндпоинтов
// @ServiceDomain подставляет базовый URL из test.properties (${api.users})
// @QueryParameter добавляет глобальный параметр api_version=v2 ко всем запросам
// @Header задаёт глобальный Accept-заголовок
@Service
@ServiceDomain("${api.users}")
@QueryParameter(name = "api_version", value = "v2")
@Header(name = "Accept", value = "application/json")
public interface UserApi {

    // GET /users/{id}?api_version=v2&details=full
    // @Endpoint задаёт путь с плейсхолдером {userId}
    // @Query добавляет параметр details из аргумента метода
    @GET
    @Endpoint("/users/{userId}")
    @Query("details")
    Response<UserProfile> getUserById(
            @Path("userId") String id,        // подставляется в {userId}
            @Query("details") String expand   // добавляется как ?details=full
    );

    // POST /users?api_version=v2&source=mobile
    // @Body сериализует POJO в JSON
    // @Header переопределяет Authorization только для этого запроса
    @POST
    @Endpoint("/users")
    @QueryParameter(name = "source", value = "mobile")
    @Header(name = "Authorization", value = "Bearer ${auth_token}")
    Response<UserProfile> createUser(
            @Body UserCreateRequest payload   // автосериализация в JSON
    );

    // PUT /users/{id}/avatar — загрузка файла через multipart
    // @Multipart выставляет Content-Type: multipart/form-data
    // @FilePart описывает бинарную часть, @Part — текстовые метаданные
    @PUT
    @Endpoint("/users/{userId}/avatar")
    @Multipart
    Response<UploadResult> uploadAvatar(
            @Path("userId") String id,
            @FilePart(controlName = "file", fileName = "avatar.jpg") File image,
            @Part(name = "description") String description
    );

    // DELETE /users/{id} с кастомным таймаутом и повторами при 502/503
    @DELETE
    @Endpoint("/users/{userId}")
    @Timeout(connection = 3000, read = 15000)
    @RetryOnFailure(maxAttempts = 2, statusCodes = {502, 503})
    Response<Void> deleteUser(
            @Path("userId") String id
    );
}
```

---

## Приоритеты и разрешение конфликтов

> JDI Dark разрешает конфликты конфигурации по принципу **наиболее специфичный переопределяет общий**.
>
> Непересекающиеся параметры объединяются.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ЦЕПОЧКА ПРИОРИТЕТОВ (от низшего к высшему)                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ГЛОБАЛЬНЫЕ НАСТРОЙКИ (JDISettings)                                  │
│     │                                                                   │
│     └─► JDISettings.settings().addDefaultHeader("X-Global", "1")        │
│         Применяется ко ВСЕМ запросам во всём проекте                    │
│                                                                         │
│  2. @Service (уровень интерфейса)                                       │
│     │                                                                   │
│     ├─► @Service(headers = {@Header("X-Service", "2")})                 │
│     └─► Применяется ко всем методам внутри UserApi                      │
│                                                                         │
│  3. @ServiceDomain / @QueryParameter (интерфейсный уровень)             │
│     │                                                                   │
│     ├─► @ServiceDomain("${api.base}")                                   │
│     ├─► @QueryParameter(name="v", value="2")                            │
│     └─► Применяется ко всем @Endpoint-методам интерфейса                │
│                                                                         │
│  4. @Endpoint (уровень метода)                                          │
│     │                                                                   │
│     ├─► @Endpoint("/users") + @Header("X-Resource", "3")                │
│     └─► Применяется к конкретному методу интерфейса                     │
│                                                                         │
│  5. @Header / @QueryParameter / @Path (уровень аргумента метода)        │
│     │                                                                   │
│     ├─► @Header("Authorization", "Bearer ${token}")                     │
│     ├─► @QueryParameter(name="debug", value="true")                     │
│     └─► Переопределяет все предыдущие значения с тем же именем          │
│                                                                         │
│  ИТОГ:                                                                  │
│  ┌─────────────────────────────────────────────────┐                    │
│  │ Запрос: GET /users/42?v=2&debug=true            │                    │
│  │ Headers:                                        │                    │
│  │   X-Global: 1          ← из JDISettings         │                    │
│  │   X-Service: 2         ← из @Service            │                    │
│  │   X-Resource: 3        ← из @Endpoint           │                    │
│  │   Authorization: Bearer xyz ← из @Header (метод)│                    │
│  └─────────────────────────────────────────────────┘                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Ключевые правила:**

- **Заголовки**: аргумент метода > метод > интерфейс > глобальные. При совпадении имён — переопределение, при разных — объединение.
- **Query-параметры**: метод > интерфейс. Дубликаты имён **не объединяются** — локальный параметр заменяет глобальный.
- **Плейсхолдеры** `${key}` резолвятся в порядке: `System.getProperty()` → `test.properties` → переменные окружения.
- **Таймауты**: `@Timeout` на методе переопределяет настройки из `@Service` и глобальные `JDISettings`.

---

## Request и Response объекты

- `Request` — иммутабельная обёртка над `HttpRequest.Builder`, содержащая тело, заголовки, параметры и метаданные перед отправкой.

  Создаётся автоматически при декларативном вызове или вручную через `Request.builder()`.

- `Response<T>` — типобезопасная обёртка над `HttpResponse`, хранящая статус-код, заголовки, распарсенное тело и метрики выполнения.
- Поддержка дженериков позволяет автоматически десериализовать JSON в POJO через подключённый сериализатор (Jackson по умолчанию, Gson при настройке).

### Работа с типами ответов

```java
// Response<T> автоматически парсит тело ответа в указанный тип при вызове .getBody()
// Если T = String — возвращается сырой JSON/текст без десериализации
// Если T = POJO — вызывается десериализатор из подключённой библиотеки
Response<UserProfile> profileResponse = userApi.getUserById("123", "full");

// Получение метаданных ответа
int statusCode = profileResponse.getCode();                     // 200
String contentType = profileResponse.getHeader("Content-Type"); // application/json
long executionTime = profileResponse.getTime();                 // время выполнения в мс

// Получение распарсенного тела (типобезопасно)
UserProfile userProfile = profileResponse.getBody();            // готовый POJO
String userName = userProfile.getName();                        // доступ к полям

// Fallback: ручная обработка сырого тела (для сложных схем или частичного JSON)
String rawJson = profileResponse.getRawBody();                  // "{\"id\":123,\"name\":\"Alex\"}"
// Кастомная десериализация через утилиту проекта
UserProfile customParsed = JsonUtils.fromJson(rawJson, UserProfile.class);

// Проверка статуса через встроенные хелперы (читаемее, чем сравнение кодов)
if (profileResponse.isSuccess()) {      // true для 200–299
    // обработка успешного ответа
}
if (profileResponse.isCreated()) {      // true для 201
    // обработка создания ресурса
}
```

---

## JDIHttpClient и низкоуровневый доступ

- `JDIHttpClient` — центральный компонент фреймворка, управляющий пулом соединений, таймаутами, прокси и стратегиями повторных попыток.
- При декларативном вызове `@Endpoint` фреймворк автоматически инстанциирует и кэширует клиента. Явная передача требуется только для кастомной логики.
- Для динамических сценариев (генерация URL в рантайме, сложная модификация запроса) можно работать с `JDIHttpClient` напрямую, минуя аннотации.

### Прямой вызов через клиент

```java
// Создание клиента с явной конфигурацией (альтернатива декларативной инициализации)
JDIHttpClient client = new JDIHttpClient.Builder()
        .baseUrl("https://api.example.com")                     // базовый домен
        .connectionTimeout(5000)                                // таймаут соединения, мс
        .readTimeout(30000)                                     // таймаут чтения, мс
        .addDefaultHeader("X-Client", "test-runner")            // глобальный заголовок
        .build();

// Ручное формирование запроса без аннотаций (полезно для динамических путей)
Request req = Request.builder()
        .method("POST")                                         // HTTP-метод
        .path("/v1/orders")                                     // путь относительно baseUrl
        .header("Content-Type", "application/json")             // заголовок
        .header("X-Request-Id", UUID.randomUUID().toString())   // динамический заголовок
        .body("{\"productId\": 42, \"quantity\": 2}")           // сырое тело (JSON-строка)
        .queryParam("source", "mobile")                         // query-параметр
        .build();

// Отправка запроса с указанием типа десериализации ответа
// Если указать String.class — тело вернётся как сырая строка
// Если указать POJO.class — произойдёт автоматическая десериализация
Response<OrderConfirmation> res = client.call(req, OrderConfirmation.class);

// Извлечение метрик и данных ответа
System.out.println("Status: " + res.getCode());                         // 201
System.out.println("Execution time: " + res.getTime() + " ms");         // 245 ms
System.out.println("Location header: " + res.getHeader("Location"));    // /v1/orders/789

// Работа с распарсенным телом
if (res.isSuccess()) {
    OrderConfirmation order = res.getBody();
    System.out.println("Order ID: " + order.getId());           // 789
}
```

---

## Best Practices

- **Выносите `@Service`-интерфейсы в отдельный пакет** `com.project.api.endpoints` и не смешивайте их с логикой тестов — это упрощает навигацию и рефакторинг.
- **Используйте `Response<T>` вместо `void` или `String`** — это сохраняет типобезопасность, даёт доступ к заголовкам и метрикам выполнения.
- **Используйте `@Path` с плейсхолдерами** вместо ручной сборки строк `"/users/" + id` — фреймворк проверяет корректность путей на этапе инициализации и защищает от опечаток.
- **Избегайте хардкода чувствительных значений** в `@Header` или `@QueryParameter` — передавайте токены и ключи через параметры методов или переменные окружения `${API_KEY}`.
- **Для сложной модификации запроса** (динамические заголовки, условная логика) используйте `JDIHttpClient.call()` напрямую или кастомные интерцепторы, а не перегружайте аннотации.
- **Группируйте связанные эндпоинты** в отдельные `@Service`-интерфейсы (например, `UserApi`, `OrderApi`) — это улучшает структуру кода и автодополнение в IDE.
- **Документируйте плейсхолдеры** в `@Body`-шаблонах через JavaDoc параметров метода — это помогает новым членам команды понимать, какие аргументы куда подставляются.

### Антипаттерны

- **Хардкод доменов** в `@ServiceDomain("http://...")` вместо `${api.base}` — усложняет переключение между окружениями и требует изменения кода при деплое.
- **Дублирование `@QueryParameter`** на интерфейсе и методе с одинаковым именем — создаёт путаницу в приоритетах; используйте явное переопределение с комментарием.
- **Использование `@Body String` для простых объектов** вместо POJO — отключает автосериализацию, увеличивает риск ошибок в ручном формате JSON.
- **Игнорирование `@Timeout`** для долгих эндпоинтов — приводит к преждевременным `SocketTimeoutException` в CI-средах с ограниченной пропускной способностью.
- **Смешивание `@Form` и `@Body`** в одном методе — вызывает конфликт `Content-Type`; выберите один способ передачи данных.

# 4. HTTP-методы и параметры

> Раздел охватывает поддержку всех стандартных HTTP-методов и способы передачи параметров в JDI Dark.
>
> Фреймворк унифицирует работу с методами через аннотации, автоматически обрабатывая маршрутизацию,
> кодирование и формирование тела запроса.

## Поддерживаемые HTTP-методы

JDI Dark предоставляет аннотации-маркеры для каждого метода, что позволяет декларативно описывать контракты API
без ручного конструирования `HttpRequest`.

- `@GET` — извлечение данных, параметры передаются только через URL или заголовки
- `@POST` — создание ресурсов, поддерживает тело в формате JSON, Form или Multipart
- `@PUT` — полная замена ресурса, требует передачи полного тела запроса
- `@PATCH` — частичное обновление, отправляются только изменённые поля
- `@DELETE` — удаление ресурса, может принимать `@Path` или `@Query` для идентификации

### Пример декларативного интерфейса

```java
// Аннотация @Service активирует сканирование интерфейса JDI Dark
@Service
public interface OrderApi {

    // Базовый путь для всех методов вложенного класса
    // Относительно BASE_URL клиента полный путь: /api/v2/orders
    @Endpoint("/orders")
    public class Orders {

        // GET-запрос без тела, возвращает список заказов
        // @Query автоматически кодирует параметры в URL: /orders?status=active&page=1
        @GET
        @Query({"status", "page"})
        Response<List<OrderDto>> getOrders(@Query("status") String status, @Query("page") int page);

        // POST-запрос с JSON-телом
        // Фреймворк автоматически сериализует объект CreateOrderRequest через Jackson
        @POST
        Response<OrderDto> createOrder(@Body CreateOrderRequest payload);

        // PATCH-запрос для частичного обновления
        // Передаётся только DTO с изменёнными полями, Content-Type: application/json
        @PATCH
        @Path("/{orderId}")
        Response<OrderDto> updateOrder(@Path("orderId") String id, @Body UpdateOrderRequest patchData);

        // DELETE-запрос с динамическим путём
        // Возвращает статус 204 No Content, тело ответа игнорируется
        @DELETE
        @Path("/{orderId}")
        Response<Void> deleteOrder(@Path("orderId") String id);
    }
}
```

## Работа с параметрами запроса

- Path-параметры — подставляются в плейсхолдеры `{param}` внутри URL
- Query-параметры — добавляются после `?` с автоматическим URL-кодированием
- Form-параметры — отправляются как `application/x-www-form-urlencoded`
- Multipart-параметры — поддерживают загрузку файлов и смешанные типы данных

### Form и Multipart примеры

```java
@Service
@Endpoint("/api")
public interface FileApi {

    // Отправка данных формы: Content-Type устанавливается автоматически
    // @Form аннотация принимает массив строк-ключей или маппит имя параметра
    @POST
    @Path("/login")
    @Form
    // @Form({"username", "password"}) // ← принудительный порядок
    Response<AuthResult> loginForm(@Form("username") String user, @Form("password") String pass);
    // (@Form String user, @Form String pass) -> если имена полей не указывать, то они беруться из параметров -> user, pass

    // Multipart-запрос для загрузки файла с метаданными
    // JDI Dark автоматически формирует boundary и заголовок Content-Type
    @POST
    @Path("/documents")
    @Multipart
    Response<DocumentMeta> uploadDocument(
            // @Part для обычного текстового поля внутри multipart
            @Part("category") String category,
            // @FilePart для бинарных данных, указывается имя файла и MIME-тип
            @FilePart(value = "file", filename = "report.pdf", contentType = "application/pdf") File reportFile
    );
}
```

## Комбинирование параметров и тело запроса

В сложных сценариях допустимо комбинировать `@Path`, `@Query` и `@Body` в одном методе.
Порядок следования аргументов в Java-методе не влияет на порядок их применения в HTTP-запросе.

```java
@Service
@Endpoint("/analytics")
public interface AnalyticsApi {

    // Сложный POST-запрос: динамический путь + фильтры в URL + тело запроса
    @POST
    @Path("/{region}/export")
    @Query({"format", "dateRange"})
    Response<ExportTask> exportReport(
            @Path("region") String region,           // Подставляется в URL: /analytics/eu-west-1/export
            @Query("format") String format,          // Добавляется в строку: ?format=csv
            @Query("dateRange") String dateRange,    // Добавляется в строку: &dateRange=2026-01-01_2026-05-07
            @Body ExportFilters filters              // Сериализуется в JSON-тело запроса
    );
}
```

## Best Practices

- Используйте `@Path` для обязательных идентификаторов ресурса, а `@Query` — для фильтрации, пагинации или сортировки
- Для `@Multipart` всегда указывайте `contentType` в `@FilePart`, чтобы сервер корректно интерпретировал бинарные данные
- Избегайте передачи чувствительных данных через `@Query` — URL логируются прокси-серверами и попадают в историю браузера/CI-логов
- При работе с `@Form` используйте кодировку UTF-8 по умолчанию, но проверяйте поддержку на стороне legacy-сервисов
- Не смешивайте `@Body` и `@Form`/`@Multipart` в одном запросе — это нарушает спецификацию HTTP и может привести к 400 Bad Request
- Валидируйте обязательность параметров на этапе компиляции, добавляя проверки `Objects.requireNonNull()`
  или используя Bean Validation (`@NotNull`), если проект интегрирован с Jakarta Validation

---

# 5. Данные и сериализация

> JDI Dark берёт на себя преобразование Java-объектов в HTTP-пейлоады и обратно, минимизируя ручной парсинг.
>
> Фреймворк делегирует сериализацию подключённым библиотекам (Jackson или Gson) и предоставляет хуки для кастомной логики.

## Автоматический маппинг JSON ↔ POJO

- При передаче POJO в аннотации `@Body` фреймворк вызывает `ObjectMapper` для конвертации объекта в JSON-строку.
- При получении `Response<T>` тело ответа автоматически десериализуется в указанный дженерик-тип.
- Поддержка аннотаций Jackson/Gson позволяет гибко управлять именами полей, игнорированием null-значений и форматами дат.

```java
// POJO для тела запроса/ответа
public class OrderPayload {

    // @JsonProperty сопоставляет Java-поле с JSON-ключом, если они различаются
    // Это критично при работе с API, использующими snake_case
    @JsonProperty("order_id")
    private String id;

    // Стандартное маппирование без аннотаций работает camelCase ↔ camelCase
    private String status;

    // @JsonIgnore исключает поле из сериализации/десериализации
    // Поле остаётся в памяти для тестовой логики, но не уходит в сеть
    @JsonIgnore
    private String internalDebugId;

    // @JsonFormat задаёт паттерн для дат, что критично для совместимости с legacy-API
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSZ")
    private LocalDateTime createdAt;

    // геттеры/сеттеры
}
```

## Обход автоматической сериализации

- Иногда сервер требует XML, CSV, сырой JSON с нестандартной структурой или GraphQL-запрос.
- JDI Dark позволяет передавать `String`, `byte[]` или `File` напрямую, отключая POJO-конвертер.

```java
@Service
@Endpoint("/api")
public interface RawDataApi {

    // Отправка сырого JSON-тела как String
    // Content-Type можно задать явно через @Header или оставить дефолтным
    @POST
    @Path("/graphql")
    @Header("Content-Type", "application/json")
    Response<GraphQlResponse> sendRawGraphql(@Body String graphqlPayload);

    // Отправка XML-строки с явным указанием MIME-типа
    @POST
    @Path("/import/xml")
    @Header("Content-Type", "application/xml")
    Response<ImportStatus> uploadXml(@Body String xmlContent);

    // Загрузка бинарного файла (byte[]) без сериализации в текст
    // Фреймворк автоматически выставит Content-Length и application/octet-stream
    @PUT
    @Path("/backup/{name}")
    Response<BackupMeta> uploadBinary(@Path("name") String filename, @Body byte[] data);
}
```

## Кастомные сериализаторы и мапперы

- Если стандартный Jackson не справляется с полиморфными схемами, вложенными массивами или кастомными форматами,
  можно подключить свой маппер.
- JDI Dark поддерживает регистрацию `ObjectMapper` через `JDISettings` или использование утилиты `JsonUtils` для ручного контроля.

```java
// Конфигурация кастомного ObjectMapper на старте тестового сьюта
public class SerializationConfig {

    static {
        // Создаём экземпляр маппера с нужными фичами
        ObjectMapper customMapper = new ObjectMapper()
                // Включаем поддержку Java 8 Date/Time API (LocalDateTime, Instant и т.д.)
                .registerModule(new JavaTimeModule())
                // Включить форматирование для читаемости логов (только локально)
                .enable(SerializationFeature.INDENT_OUTPUT)
                // Игнорировать неизвестные поля, чтобы тесты не падали при расширении API
                .disable(SerializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
                // Принудительно сериализовать null-поля как "null" в JSON
                .enable(SerializationFeature.WRITE_NULL_MAP_VALUES);

        // Регистрируем маппер в глобальных настройках JDI Dark
        // Все последующие @Body и Response<T> будут использовать его
        JDISettings.settings().setObjectMapper(customMapper);
    }
}
```

## Best Practices

- Всегда используйте POJO для сложных JSON-структур, чтобы получить compile-time проверку типов и автодополнение в IDE
- Для полей, которые могут отсутствовать в ответе сервера, используйте `Optional<T>` или примитивы с дефолтными значениями,
  чтобы избежать `NullPointerException` при десериализации
- Выносите `@JsonFormat` и кастомные десериализаторы в отдельные DTO-пакеты, не смешивайте их с тестовой логикой
- При работе с большими файлами (>50MB) передавайте `InputStream` или используйте `@Multipart` с чанками, чтобы не переполнять heap
  `@FilePart(value = "file", filename = "backup.zip") InputStream fileStream`
- Включайте `FAIL_ON_UNKNOWN_PROPERTIES = false` только на время миграции или для черновиков API; в стабильных контрактах
  это скрывает ошибки версионирования
- Логируйте сырые пейлоады только в `DEBUG`-режиме; в CI-средах отключайте `INDENT_OUTPUT`, чтобы сократить размер отчётов Allure

# 6. Заголовки и авторизация

> Управление заголовками и механизмами авторизации в JDI Dark выстроено на двух уровнях: глобальном (для всего сьюта)
> и локальном (для конкретных эндпоинтов).
>
> Фреймворк предоставляет гибкие хуки и интерцепторы для динамической генерации токенов, ротации секретов и безопасной
> передачи credentials.

## Глобальные vs локальные заголовки

- **Глобальные** — применяются ко всем запросам, инициированным через настроенный `JDIHttpClient`.
  Удобны для общих заголовков (`Accept`, `User-Agent`, глобальные API-ключи).
- **Локальные** — задаются на уровне аннотации `@Header` или передаются как параметры метода.
  Переопределяют глобальные значения для конкретного вызова.

Глобальная конфигурация (один раз при старте):

```java
public class ApiConfig {
    public static final JDIHttpClient CLIENT = new JDIHttpClient("https://api.example.com");

    static {
        // Заголовки, которые нужны для ВСЕХ запросов
        JDISettings.settings().addDefaultHeader("Accept", "application/json");
        JDISettings.settings().addDefaultHeader("User-Agent", "MyApp/2.0");
        JDISettings.settings().addDefaultHeader("X-API-Key", "global-key-12345");
    }
}

@Service
@Endpoint("/api/v1")
public interface ProductApi {

    // УРОВЕНЬ 1: Заголовок на весь эндпоинт
    @Endpoint("/products")
    @Header(name = "X-Module", value = "catalog")  // ← применится ко ВСЕМ методам
    class Products {

        // Запрос 1: Только глобальные + X-Module: catalog
        @GET
        Response<List<Product>> getAll();

        // Запрос 2: Переопределяем Accept только для этого метода
        @GET
        @Path("/{id}")
        @Header(name = "Accept", value = "application/xml")  // ← вместо "application/json"
        Response<Product> getAsXml(@Path("id") String id);

        // Запрос 3: Динамический заголовок из параметра метода
        @POST
        @Header(name = "X-Request-Id")  // ← имя фиксировано
        Response<Product> create(
                @Body Product newProduct,
                @Header("X-Request-Id") String requestId  // ← значение из теста
        );

        // Или динамическая передача кастомных заголовков через Map 
        @POST
        Response<Product> create(
                @Body Product newProduct,
                @Header Map<String, String> headers  // ← все кастомные заголовки
        );
    }
}
```

## Basic, Bearer, OAuth2 и динамические токены

- **Basic Auth** — кодирует `username:password` в Base64 и передаёт в заголовке `Authorization`.
- **Bearer Token** — стандарт для JWT/OAuth2, требует передачи токена в формате `Bearer <token>`.
- **OAuth2 / Динамические токены** — в тестах токены часто имеют короткий TTL.
  JDI Dark позволяет инжектировать их на лету через интерцепторы или хелперы.

```java
import com.epam.jdi.http.annotations.*;
import com.epam.jdi.http.Response;
import java.util.Base64;

@Service
@Endpoint("/api")
public interface AuthApi {

    // Пример Basic Auth через динамический заголовок
    // В реальном проекте лучше вынести кодирование в утилитный метод
    @GET
    @Path("/users/me")
    @Header("Authorization")
    Response<UserProfile> getMyProfile(@Header("Authorization") String basicAuthHeader);

    // Пример Bearer Token с динамической передачей
    // Тест генерирует или подтягивает токен из Vault/CI-переменных
    @POST
    @Path("/orders")
    @Header("Authorization")
    Response<OrderDto> createOrder(@Body OrderPayload order, @Header("Authorization") String bearerToken);
}

// Хелпер для генерации заголовков авторизации
public class AuthHeaders {

    // Генерация Basic Auth заголовка
    public static String basic(String username, String password) {
        // Формируем строку "user:pass" и кодируем в Base64 согласно RFC 7617
        String raw = username + ":" + password;
        return "Basic " + Base64.getEncoder().encodeToString(raw.getBytes());
    }

    // Генерация Bearer заголовка
    public static String bearer(String token) {
        return "Bearer " + token;
    }
}
```

## Интерцепторы для модификации запросов/ответов

- Интерцепторы позволяют прозрачно изменять запрос перед отправкой или анализировать ответ до передачи в тест.
- Идеальны для: автоматического обновления токенов (401 → refresh → retry), маскирования чувствительных данных в логах, 
  добавления трейсинг-ID.

```java
// Интерцептор запросов: выполняется ДО отправки HTTP-запроса
public class AuthRefreshInterceptor implements RequestInterceptor {
    
    @Override
    public Request intercept(Request request) {
        // Проверяем, есть ли уже токен в заголовках текущего запроса
        if (request.getHeader("Authorization") == null) {
            // Если токена нет, запрашиваем его у OAuth-провайдера (псевдокод вызова)
            String freshToken = TokenProvider.getFreshToken();
            // Добавляем заголовок Bearer к текущему запросу
            request.addHeader("Authorization", "Bearer " + freshToken);
        }
        // Возвращаем модифицированный запрос для дальнейшей отправки в сеть
        return request;
    }
}

// Интерцептор ответов: выполняется ПОСЛЕ получения ответа от сервера
public class LoggingResponseInterceptor implements ResponseInterceptor {
    
    @Override
    public Response intercept(Response response) {
        // Пример: логирование только метаданных, скрывая тело ответа для приватных данных
        if (response.getCode() >= 400) {
            System.out.println("Error Response Headers: " + response.getHeaders());
            // Можно выбросить кастомное исключение или модифицировать ответ перед возвратом
        }
        return response;
    }
}

// Регистрация интерцепторов в клиенте
public class ApiConfig {
    
    private static final String BASE_URL = "https://api.example.com";
    public static final JDIHttpClient CLIENT;

    static {
        CLIENT = new JDIHttpClient(BASE_URL);

        // Глобальные настройки через JDISettings
        JDISettings.settings().setConnectionTimeout(5000);
        JDISettings.settings().setReadTimeout(10000);
        JDISettings.settings().addDefaultHeader("Accept", "application/json");
        // ...

        // Добавляем интерцепторы в цепочку обработки (порядок важен!)
        CLIENT.addInterceptor(new AuthRefreshInterceptor());
        CLIENT.addResponseInterceptor(new LoggingResponseInterceptor());
    }

    public static JDIHttpClient getClient() {
            return CLIENT;
    }
}
```

## Best Practices

- Никогда не харкодьте токены или пароли в аннотациях `@Header` — используйте переменные окружения или secure vaults, 
  чтобы секреты не попали в VCS
- Для Basic/Bearer авторизации создавайте выделенные методы-хелперы, чтобы избежать дублирования строки `"Bearer " + token` 
  по всему проекту
- Интерцепторы должны быть stateless (без сохранения изменяемого состояния - не использовать поля в классах интерцепторов), 
  иначе параллельный запуск тестов приведёт к гонкам данных и непредсказуемым ошибкам авторизации
- При работе с OAuth2 реализуйте кэширование токенов на уровне `TokenProvider` с проверкой `exp`, 
  чтобы не дергать auth-сервер на каждый запрос и не попасть в rate-limit
- Используйте `X-Request-Id` или `X-Correlation-ID` для трассировки запросов в распределённых системах — это упрощает дебаг 
  в Allure и логах бэкенда
- Отключайте логирование заголовков авторизации в `setLogAll(true)` на этапе CI, чтобы не оставлять секреты в артефактах пайплайна; 
  используйте маскировку вроде `Authorization: Bearer ***`

---

# 7. Валидация и Asserts

> Валидация ответов — критический этап API-тестирования. 
> 
> JDI Dark предоставляет декларативный и fluent-интерфейс для проверок, глубоко интегрированный с Hamcrest, 
> JUnit / TestNG и встроенным логгером. 
> 
> Подход ориентирован на читаемость, быстрый фидбэк и лёгкую отладку.

## Проверка статус-кодов и заголовков

JDI Dark оборачивает HTTP-ответ в объект `Response`, который содержит готовые методы для быстрой проверки статуса и 
HTTP-заголовков без необходимости ручного парсинга.

- `isSuccess()` — проверка кодов `2xx`
- `isCreated()` — проверка кода `201`
- `assertStatusCode(int)` — жёсткое сравнение с ожидаемым кодом
- `assertHeader(String, Matcher)` — валидация конкретного заголовка через матчер
- `assertAll()` — пакетная проверка (используется в комбинации с SoftAsserts, если подключён)

#### Пример проверки статус-кодов и заголовков

```java
// 1. Выполняем запрос через прокси-интерфейс эндпоинта
Response response = UserEndpoint.getUser.call("123");

// 2. Быстрая проверка успешности (200–299)
response.isSuccess(); // бросит AssertionError, если статус != 2xx

// 3. Точная проверка конкретного кода
response.assertStatusCode(200); // строгое сравнение

// 4. Валидация заголовка с использованием Hamcrest-матчеров
// Проверяем, что Content-Type содержит application/json
response.assertHeader("Content-Type", containsString("application/json"));

// 5. Проверка наличия и значения кастомного заголовка
// X-RateLimit-Remaining должен быть равен "99"
response.assertHeader("X-RateLimit-Remaining", equalTo("99"));

// 6. Проверка нескольких заголовков одной цепочкой
response.assertHeader("Server", containsString("nginx"))
        .assertHeader("Connection", equalTo("keep-alive"));
```

**Ключевые особенности:**

- Методы `assert...` немедленно прерывают тест при неуспехе (Hard Assert)
- Для мягких проверок (Soft Assert) рекомендуется оборачивать вызовы в `SoftAssert` из JDI Light или TestNG
- Заголовки регистронезависимы по стандарту HTTP, но JDI Dark нормализует их на уровне клиента

---

## Валидация тела ответа и JSONPath

> JDI Dark предоставляет встроенный парсер JSON, работающий поверх стандартной библиотеки JSONPath. 
>
> Это позволяет извлекать и проверять вложенные структуры, массивы и примитивы без ручного маппинга в POJO.

- `assertBody(String path, Matcher)` — проверка значения по JSONPath
- `body().get(String path)` — прямое извлечение значения для дальнейшей обработки
- `body().jsonPath()` — доступ к полному API JSONPath для сложных запросов
- Поддержка синтаксиса: `$.field`, `$.[0].name`, `$..price`, `?(@.active == true)`

#### Пример валидации JSON через JSONPath и матчеры

```java
Response response = ProductEndpoint.searchProducts.call();

// 1. Проверка корневого массива на непустоту
// $.length() возвращает количество элементов в массиве
response.assertBody("$.length()", greaterThan(0));

// 2. Проверка первого элемента в массиве
// Извлекаем id первого продукта и сравниваем с ожидаемым
response.assertBody("$[0].id", equalTo(1001));

// 3. Валидация вложенного объекта
// Проверяем, что currency внутри price равен "USD"
response.assertBody("$[0].price.currency", equalTo("USD"));

// 4. Поиск по всему документу (рекурсивный спуск)
// Находим все поля status в любом уровне вложенности
response.assertBody("$..status", hasItem("ACTIVE"));

// 5. Проверка типов данных
// Убеждаемся, что createdAt — строка, а не null или число
response.assertBody("$[0].createdAt", instanceOf(String.class));

// 6. Динамическое извлечение значения для использования в следующих шагах
String token = response.body().get("$.auth.token");
// Теперь token можно передать в заголовок следующего запроса
```

**Важные замечания по JSONPath в JDI Dark:**

- Пути всегда начинаются с `$` (корень документа)
- Массивы индексируются с `0`, поддержка отрицательных индексов (`$[-1]` — последний элемент) зависит от версии JSONPath-движка
- При работе с большими ответами рекомендуется фильтровать ненужные поля на уровне сервера (`fields=...`), чтобы снизить нагрузку на парсер

---

## Hamcrest-матчеры и кастомные ассерты

> JDI Dark полностью совместим с `org.hamcrest.Matchers`. 
> 
> Это даёт гибкость при построении сложных условий проверки. 
> Кроме стандартных матчеров, фреймворк позволяет писать собственные `BaseMatcher` для бизнес-логики.

- `equalTo()`, `is()`, `not()` — базовое сравнение
- `hasItem()`, `hasItems()` — проверка вхождения в коллекцию
- `containsString()`, `startsWith()`, `endsWith()` — работа со строками
- `nullValue()`, `notNullValue()` — проверка на null
- `allOf()`, `anyOf()` — логические комбинации условий

#### Пример кастомного матчера для проверки формата даты

```java
// 1. Создаём кастомный матчер для валидации формата ISO-8601 (YYYY-MM-DD)
public class IsValidDateMatcher extends TypeSafeMatcher<String> {

    @Override
    protected boolean matchesSafely(String item) {
        if (item == null) return false;
        try {
            LocalDate.parse(item); // пытается распарсить дату
            return true;
        } catch (DateTimeParseException e) {
            return false;
        }
    }

    @Override
    public void describeTo(Description description) {
        description.appendText("должна быть валидной датой в формате ISO-8601");
    }

    // Статический фабричный метод для удобного вызова
    public static TypeSafeMatcher<String> isValidDate() {
        return new IsValidDateMatcher();
    }
}

// 2. Использование в тесте
Response response = ReportEndpoint.getDateReport.call();

// Проверяем, что поле reportDate соответствует нашему кастомному правилу
response.assertBody("$.reportDate", IsValidDateMatcher.isValidDate());
```

**Рекомендации по использованию матчеров:**

- Используйте `allOf()` для группировки проверок одного поля
- Избегайте `anyOf()` в критических проверках — это размывает диагностическую информацию при падении
- Кастомные матчеры выносите в отдельный пакет `com.yourproject.matchers` для переиспользования

---

## Логирование запросов и ответов

> JDI Dark интегрирован с подсистемой логирования JDI Light. 
> 
> Логирование настраивается централизованно через конфигурационные файлы и позволяет выводить запросы/ответы 
> в удобных форматах: `cURL`, `HTTP Raw`, `Pretty JSON`.

Процесс логирования проходит через следующие этапы:
```
┌─────────────────────────────────────────────────────────────────┐
│                    ЦИКЛ ЛОГИРОВАНИЯ JDI DARK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ИНИЦИАЛИЗАЦИЯ ЗАПРОСА                                       │
│     │                                                           │
│     ├─► Считывается конфигурация (logging.level, format)        │
│     │   ┌─────────────────────────────────────────────────┐     │
│     │   │ application.properties / jdi.properties         │     │
│     │   │ jdi.light.log.level=INFO                        │     │
│     │   │ jdi.dark.log.request=true                       │     │
│     │   └─────────────────────────────────────────────────┘     │
│     │                                                           │
│  2. ОТПРАВКА (BEFORE)                                           │
│     │                                                           │
│     ├─► Форматируется лог запроса (URL, Headers, Body)          │
│     ├─► Выводится в консоль/файл с префиксом [REQUEST]          │
│     │   ┌─────────────────────────────────────────────────┐     │
│     │   │ curl -X GET http://api/users/123 \              │     │
│     │   │ -H "Authorization: Bearer eyJ..."               │     │
│     │   └─────────────────────────────────────────────────┘     │
│     │                                                           │
│  3. ПОЛУЧЕНИЕ ОТВЕТА (AFTER)                                    │
│     │                                                           │
│     ├─► Форматируется лог ответа (Status, Headers, Body)        │
│     ├─► Выводится с префиксом [RESPONSE]                        │
│     │   ┌─────────────────────────────────────────────────┐     │
│     │   │ HTTP/1.1 200 OK                                 │     │
│     │   │ Content-Type: application/json                  │     │
│     │   │ { "id": 123, "name": "Alex" }                   │     │
│     │   └─────────────────────────────────────────────────┘     │
│     │                                                           │
│  4. ФИНАЛИЗАЦИЯ                                                 │
│     │                                                           │
│     └─► Лог прикрепляется к Allure-отчёту (если включено)       │
│         └─► Генерируется ссылка для отладки                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Уровень логирования | Что записывается                                                    | Когда использовать                                   |
|:--------------------|---------------------------------------------------------------------|------------------------------------------------------|
| `DEBUG`             | Полные заголовки, тело запроса/ответа, время выполнения, SSL-детали | Отладка интеграций, реверс-инжиниринг API            |
| `INFO`              | URL, статус-код, размер тела, ключевые заголовки                    | Стандартный прогон тестов, CI/CD                     |
| `WARN`              | Только предупреждения (таймауты, редиректы, ошибки сериализации)    | Продакшн-мониторинг, экономия дискового пространства |
| `ERROR`             | Исключения HTTP-клиента, сетевые разрывы, 5xx без тела              | Аварийные сценарии, алертинг                         |

#### Конфигурация логирования в application.properties

```properties
# Глобальный уровень логирования JDI
jdi.light.log.level=INFO

# Включение логирования запросов и ответов отдельно
jdi.dark.log.request=true
jdi.dark.log.response=true

# Формат вывода: CURL, HTTP, PRETTY
jdi.dark.log.format=PRETTY

# Маскирование чувствительных заголовков (регулярные выражения)
jdi.dark.log.mask.headers=Authorization,X-API-Key,Cookie

# Лимит размера тела для логирования (чтобы не забивать консоль)
jdi.dark.log.body.limit=2048
```

**Важные нюансы логирования:**

- Маскирование заголовков работает на уровне `String.replace` по шаблону. Для сложных структур используйте `jdi.dark.log.mask.pattern`
- В CI-средах рекомендуется выставлять `jdi.dark.log.body.limit=512` и `format=CURL`, чтобы ускорить сбор логов
- Все логи автоматически прикрепляются к шагам Allure, если подключён плагин `jdi-allure-adapter`

---

## Best Practices

- Всегда используйте `assertStatusCode()` перед валидацией тела — это экономит ресурсы парсера и даёт чёткий фидбэк о сетевой ошибке
- Группируйте проверки одного эндпоинта в один блок `assert`-вызовов для читаемости отчёта
- Настраивайте `jdi.dark.log.mask.headers` на уровне проекта, чтобы токены и ключи не попадали в CI-логи
- Выносите сложные JSONPath-выражения в константы или методы `DataBuilders` для переиспользования
- Не игнорируйте `SoftAssert` при проверке нескольких независимых полей — это позволяет увидеть все ошибки за один запуск
- Проверка `response.body().toString().contains("success")` вместо JSONPath — ведёт к ложным срабатываниям при изменении структуры JSON
- Жёсткая привязка к порядку полей в ответе — JSON не гарантирует порядок, используйте `$..fieldName` или точные пути
- Логирование `DEBUG` в продакшн-CI — замедляет пайплайн в 3–5 раз и переполняет артефакты
- Игнорирование таймаутов при валидации — приводит к `SocketTimeoutException` вместо чёткой валидации статуса
- Смешивание Hard и Soft Assert без явного разделения — делает отчёт непредсказуемым

---

# 8. Интеграция с экосистемой

> JDI Dark спроектирован как агент-надстройка, а не самостоятельный тестовый раннер.
>
> Он полностью полагается на экосистему JVM-тестирования, предоставляя прозрачные интеграции с JUnit 5 и TestNG, механизмы 
> параметризации, нативную поддержку параллельного выполнения и глубокую связку с Allure для детализированной отчётности.

## Интеграция с JUnit 5 и TestNG

> JDI Dark не переопределяет жизненный цикл тестов.
>
> Вместо этого он использует стандартные аннотации раннеров для инициализации сервисов, настройки окружения и выполнения проверок.
> 
> Фреймворк автоматически применяет конфигурацию из `test.properties` при первом вызове `ServiceInit.init()`.

| Задача                         | JUnit 5                                               | TestNG                                                  |
|:-------------------------------|-------------------------------------------------------|---------------------------------------------------------|
| Инициализация сервиса          | `@BeforeEach` + `ServiceInit.init(UserService.class)` | `@BeforeMethod` + `ServiceInit.init(UserService.class)` |
| Глобальная настройка окружения | `@BeforeAll` + загрузка `test.properties`             | `@BeforeSuite` + загрузка `test.properties`             |
| Объявление теста               | `@Test`                                               | `@Test`                                                 |
| Проверка исключений            | `assertThrows(Exception.class, () -> ...)`            | `@Test(expectedExceptions = ...)`                       |
| Группа тестов                  | `@Tag("api")`                                         | `@Test(groups = {"api"})`                               |
| Параметризация                 | `@ParameterizedTest` + `@CsvSource`                   | `@Test(dataProvider = "name")`                          |

#### Пример базовой структуры теста (JUnit 5)

```java
// Файл: src/test/java/com/project/tests/UserIntegrationTest.java

// Стандартные импорты JUnit 5
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.hamcrest.Matchers.equalTo;

// Импорты JDI Dark
import com.epam.jdi.dark.api.response.Response;
import static com.epam.http.requests.ServiceInit.init;  // ← Ключевой статический импорт

// Импорты вашего проекта (сервисы и DTO)
import com.project.api.UserService;                     // ← интерфейс с аннотациями @GET, @Path и т.д.
import com.project.dto.UserProfile;                     // ← POJO для десериализации ответа

// @Tag позволяет фильтровать тесты при запуске: mvn test -Dgroups=users-api
@Tag("users-api")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)   // Гарантирует порядок выполнения
class UserIntegrationTest {

    // Поле для экземпляра сервиса — НЕ static, чтобы обеспечить изоляцию между тестами
    private UserService userService;

    // @BeforeEach гарантирует, что каждый тест получает "чистый" экземпляр сервиса
    @BeforeEach
    void initService() {
        // init() создаёт динамическую реализацию интерфейса UserService
        // Все аннотации @ServiceDomain, @Header, @QueryParameter применяются автоматически
        // Конфигурация загружается из test.properties в classpath
        // Пример: если domain=api=https://staging.example.com → все запросы пойдут туда
        userService = init(UserService.class);
    }

    // Тест #1: Проверка успешного получения пользователя
    @Order(1)
    @Test
    @DisplayName("GET /users/{id} возвращает 200 и валидный профиль")
    void shouldGetUserProfileSuccessfully() {
        // Вызов метода интерфейса — аргументы подставляются в @Path/@Query автоматически
        // 42L → подставляется в {id}, "full" → добавляется как ?details=full
        Response<UserProfile> response = userService.getUserById(42L, "full");

        // Валидация статус-кода: встроенный метод-хелпер (читаемее, чем сравнение кодов)
        response.isOk(); // Бросает AssertionError, если статус != 200

        // Детальная проверка тела ответа через JSONPath + Hamcrest-матчеры
        // Синтаксис совместим с RestAssured для удобства миграции
        response.assertThat()
                .body("id", equalTo(42))                          // Проверка конкретного поля
                .body("status", equalTo("ACTIVE"))                // Проверка статуса пользователя
                .body("createdAt", not(emptyOrNullString()));     // Поле не должно быть пустым
    }

    // Тест #2: Обработка несуществующего ресурса (негативный сценарий)
    @Order(2)
    @Test
    @DisplayName("GET /users/{id} для несуществующего ID возвращает 404")
    void shouldReturnNotFoundForInvalidUserId() {
        // Запрос к заведомо несуществующему ресурсу
        Response<UserProfile> response = userService.getUserById(999_999L, "full");

        // Проверка кода ответа через классический JUnit-assertion
        assertEquals(404, response.getStatusCode(), 
                "Ожидаем 404 для несуществующего пользователя");

        // Опциональная проверка тела ошибки (если API возвращает структурированную ошибку)
        response.assertThat()
                .body("error.code", equalTo("USER_NOT_FOUND"))
                .body("error.message", containsString("not found"));
    }

    // В большинстве случаев JDI Dark очищает ресурсы автоматически
    @AfterAll
    static void tearDown() {
        // Опционально: если используется кастомный HttpClient с пулом соединений
        // JDIHttpClient.shutdown(); <- явное освобождение ресурсов
      
        // остальная логика @AfterAll (если есть)...
    }
}
```

#### Пример базовой структуры теста (TestNG)

```java
// Файл: src/test/java/com/project/tests/UserIntegrationTestNG.java

import org.testng.annotations.*;
import static org.testng.Assert.*;
import static org.hamcrest.Matchers.equalTo;

import com.epam.jdi.dark.api.response.Response;
import static com.epam.http.requests.ServiceInit.init;
import com.project.api.UserService;
import com.project.dto.UserProfile;

// groups позволяет запускать подмножество тестов: mvn test -Dgroups=smoke
public class UserIntegrationTestNG {

    private UserService userService;

    // @BeforeMethod выполняется перед каждым @Test-методом
    @BeforeMethod
    public void setUp() {
        // Инициализация сервиса с применением конфигурации из test.properties
        userService = init(UserService.class);
    }

    @Test(groups = {"smoke", "users"})
    public void shouldGetUserProfileSuccessfully() {
        Response<UserProfile> response = userService.getUserById(42L, "full");
        
        // Проверка статуса через TestNG assertion
        assertEquals(response.getStatusCode(), 200);
        
        // Валидация тела через JSONPath
        response.assertThat()
                .body("id", equalTo(42))
                .body("status", equalTo("ACTIVE"));
    }

    @Test(groups = {"regression"}, expectedExceptions = AssertionError.class)
    public void shouldFailOnInvalidSchema() {
        // Тест, который должен упасть, если схема ответа изменилась
        Response<UserProfile> response = userService.getUserById(42L, "full");
        // Ожидаем поле, которого нет в ответе — тест упадёт с AssertionError
        response.assertThat().body("nonExistentField", equalTo("value"));
    }

    @AfterClass
    public void tearDown() {
        // Очистка ресурсов при необходимости
    }
}
```

> **Важно**: В JDI Dark **не требуется** вызывать явный `reset()` или `shutdown()` между тестами — фреймворк 
> использует `ThreadLocal` для изоляции контекста и автоматически управляет жизненным циклом HTTP-соединений. 
> 
> Явные хуки очистки нужны только при работе с кастомными пулами соединений или при интеграции с внешними mock-серверами.

---

## Параметризация запросов

> Параметризация критична для проверки граничных условий, разных ролей пользователей и вариаций входных данных.
>
> JDI Dark полностью совместим со встроенными механизмами `@ParameterizedTest` (JUnit 5) и `@DataProvider` (TestNG).

#### Пример параметризации через JUnit 5

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

class ProductParametrizedTest {

    private ProductService productService;

    @BeforeEach
    void init() {
        // Инициализация сервиса перед каждым тестом
        // init() возвращает прокси-реализацию интерфейса ProductService
        productService = init(ProductService.class);
    }

    // CsvSource: каждая строка → отдельный запуск теста с указанными параметрами
    @ParameterizedTest(name = "ID={0}, ожидаемый статус={1}")
    @CsvSource({
        "1, 200",      // Валидный продукт → 200
        "999, 404",    // Несуществующий продукт → 404
        "-1, 400",     // Некорректный формат ID → 400
        "abc, 400"     // Не-числовой ID → 400
    })
    void shouldReturnCorrectStatusForProductId(String productId, int expectedStatus) {
        // Обработка невалидных входных данных до отправки запроса
        if (productId.matches("-?\\d+")) {
            Long id = Long.valueOf(productId);
            
            Response<Product> response = productService.getProduct(id);
            
            assertEquals(expectedStatus, response.getStatusCode());
        } else {
            // Для не-числовых ID ожидаем 400 — валидация аргументов может происходить на уровне фреймворка
            // Если сервис не валидирует входные данные, тест упадёт на этапе формирования запроса
            assertThrows(IllegalArgumentException.class, 
                () -> productService.getProduct(null));
        }
    }

    // ValueSource: проверка одного параметра с разными значениями
    @ParameterizedTest
    @ValueSource(strings = {"USD", "EUR", "RUB"})
    void shouldFilterProductsByCurrency(String currency) {
        Response<SearchResult> response = productService.searchByCurrency(currency);
        
        response.isSuccess();
        
        // Проверка, что все возвращённые продукты имеют указанную валюту
        response.assertThat()
                .body("$[0].price.currency", equalTo(currency));
    }
}
```

#### Пример параметризации через TestNG DataProvider

```java
import org.testng.annotations.DataProvider;
import org.testng.annotations.Test;
import static org.testng.Assert.assertEquals;

class OrderDataProviderTest {

    // DataProvider возвращает Object[][]: каждая строка — набор аргументов для одного запуска
    @DataProvider(name = "orderPayloads")
    public Object[][] provideOrderData() {
        return new Object[][] {
            { "user_1", "ITEM_A", 1, 200, "PENDING" },   // Успешный заказ
            { "user_2", "ITEM_B", 5, 200, "CONFIRMED" }, // Успешный заказ с другим статусом
            { null,     null,     0, 400, null }         // Некорректные данные → 400
        };
    }

    @Test(dataProvider = "orderPayloads")
    public void shouldCreateOrderWithPayload(String userId, 
                                             String itemId, 
                                             int quantity, 
                                             int expectedStatus, 
                                             String expectedOrderStatus) {
        // Формирование DTO через билдер или конструктор
        OrderRequest payload = new OrderRequest(userId, itemId, quantity);
        
        // Отправка POST-запроса с телом (автосериализация в JSON через @Body)
        Response<OrderConfirmation> response = OrderService.createOrder(payload);
        
        // Валидация статус-кода
        assertEquals(response.getStatusCode(), expectedStatus);
        
        // Если заказ создан — проверяем структуру ответа
        if (expectedStatus == 201) {
            response.assertThat()
                    .body("userId", equalTo(userId))
                    .body("status", equalTo(expectedOrderStatus))
                    .body("id", notNullValue()); // ID должен быть сгенерирован сервером
        }
    }
}
```

---

## Параллельный запуск

> JDI Dark использует `ThreadLocal` для хранения контекста запросов, ответов и конфигурации HTTP-клиента.
>
> Это гарантирует потокобезопасность при параллельном выполнении тестов без необходимости ручной синхронизации.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  СХЕМА ПАРАЛЛЕЛЬНОГО ВЫПОЛНЕНИЯ                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Тестовый раннер (JUnit 5 / TestNG)                                     │
│  │                                                                      │
│  ├─► Поток #1 (Thread-Local Context #1)                                 │
│  │   ├─► init(UserService) → новый прокси-экземпляр                     │
│  │   ├─► getUserById(100) → Response #1                                 │
│  │   └─► Валидация и логирование                                        │
│  │                                                                      │
│  ├─► Поток #2 (Thread-Local Context #2)                                 │
│  │   ├─► init(ProductService) → новый прокси-экземпляр                  │
│  │   ├─► searchByCurrency("USD") → Response #2                          │
│  │   └─► Валидация и логирование                                        │
│  │                                                                      │
│  └─► Поток #N                                                           │
│      └─► ...                                                            │
│                                                                         │
│  Важно: Каждый поток получает изолированный экземпляр сервиса и         │
│  конфигурацию. Конфликты данных исключены на уровне фреймворка.         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Конфигурация параллелизма

**JUnit 5** (`src/test/resources/junit-platform.properties`)

```properties
# Включение параллельного выполнения
junit.jupiter.execution.parallel.enabled=true

# Стратегия: concurrent для параллельного запуска
junit.jupiter.execution.parallel.mode.default=concurrent

# Фиксированное количество потоков (оптимально: ядра CPU - 2)
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4

# Изоляция на уровне класса (каждый тест-класс в отдельном потоке)
junit.jupiter.execution.parallel.mode.classes.default=concurrent
```

**TestNG** (`testng.xml`)

```xml
<!-- parallel="methods" запускает каждый @Test в отдельном потоке -->
<!-- thread-count ограничивает максимальное число одновременных потоков -->
<suite name="ParallelAPISuite" parallel="methods" thread-count="6">
    <test name="UserAndOrderTests">
        <packages>
            <package name="com.project.api.tests"/>
        </packages>
    </test>
</suite>
```

> **Рекомендация**: Для стабильности в CI-средах начинайте с `thread-count=2` и увеличивайте постепенно, 
> контролируя нагрузку на тестируемое приложение и сеть.

---

## Allure-отчёты и аттачи

> Интеграция с Allure реализуется через стандартные аннотации `io.qameta.allure`.
>
> JDI Dark автоматически прикрепляет логи запросов / ответов к шагам, если подключён `jdi-allure-adapter`.

#### Пример кастомизации шагов и аттачей

```java
import io.qameta.allure.Allure;
import io.qameta.allure.Step;
import io.qameta.allure.Description;
import com.epam.jdi.dark.api.response.Response;
import static org.hamcrest.Matchers.equalTo;

class PaymentApiSteps {

    // @Step создаёт именованный блок в отчёте Allure
    // Метод с @Step можно переиспользовать в разных тест-сценариях
    @Step("Оплата заказа {orderId} на сумму {amount} {currency}")
    @Description("Выполняет POST /payments и валидирует ответ платёжного шлюза")
    public Response<PaymentResult> processPayment(String orderId, double amount, String currency) {
        
        // Формирование запроса
        PaymentRequest body = new PaymentRequest(orderId, amount, currency);
        
        // Явный шаг с аттачем тела запроса (полезно для отладки)
        Allure.step("Подготовка платежного запроса", () -> {
            Allure.addAttachment("Request Payload", "application/json", 
                    body.toJson(), "json");
        });

        // Выполнение запроса через сервис (интерфейсный вызов)
        Response<PaymentResult> response = PaymentService.pay(body);

        // Валидация ответа в рамках шага
        Allure.step("Валидация ответа от платёжного шлюза", () -> {
            // Автоматический аттач тела ответа (если включено в jdi.properties)
            // Или ручное добавление для детализации
            Allure.addAttachment("Response Body", "application/json", 
                    response.getBodyAsString(), "json");
            
            // Проверки через Hamcrest-матчеры
            response.assertThat()
                    .statusCode(200)
                    .body("transaction.status", equalTo("COMPLETED"))
                    .body("transaction.amount", equalTo(amount));
        });

        return response;
    }

    // Метод для прикрепления лога транзакции (актуально для гибридных тестов UI+API)
    @Step("Прикрепление лога транзакции")
    public void attachTransactionLog(String logContent) {
        Allure.addAttachment("Backend Transaction Log", "text/plain", 
                logContent, "txt");
    }
}
```

**Настройка автоматического логирования в Allure** (`src/test/resources/jdi.properties`)

```properties
# Включение автоматического прикрепления запросов/ответов к шагам Allure
jdi.dark.allure.log.request=true
jdi.dark.allure.log.response=true
jdi.dark.allure.attach.body=true

# Формат аттачей: JSON (pretty) или RAW
jdi.dark.allure.format=PRETTY

# Лимит размера тела для аттача (защита от раздувания отчёта)
jdi.dark.allure.body.limit=4096
```

> **Важно**: Автоматическое логирование в Allure работает только если в проекте есть зависимость `io.qameta.allure:allure-java` 
> и настроен `AllureLifecycle`. Проверяйте наличие `allure.properties` в `src/test/resources`.

---

## Best Practices

- **Используйте `@BeforeEach` / `@BeforeMethod` для инициализации сервиса** — это гарантирует изоляцию контекста между тестами и предотвращает `DirtyContext`.
- **Выносите DTO и интерфейсы сервисов в отдельные пакеты** (`api`, `dto`) — это упрощает навигацию и рефакторинг.
- **Используйте `@Tag` / `groups` для фильтрации тестов** — позволяет запускать только smoke, regression или e2e тесты в CI.
- **Настраивайте `thread-count` под ресурсы CI-агента** — начинайте с 2–4 потоков, увеличивайте постепенно.
- **Прикрепляйте только релевантные аттачи в Allure** — большие тела ответов (>4KB) лучше логировать в текстовом виде, а не дублировать в JSON.

## Антипаттерны

- **Статическое поле сервиса + `@BeforeAll`** — приводит к совместному использованию экземпляра между тестами и `DirtyContext`:
  ```java
  // ❌ НЕПРАВИЛЬНО: общий экземпляр для всех тестов
  private static UserService userService = init(UserService.class);
  ```
  
- **Игнорирование `@BeforeEach`** — если тесты изменяют состояние сервиса (добавляют заголовки динамически), контекст "загрязняется".
- **Дублирование `init()` внутри теста** — избыточно и замедляет выполнение:
  ```java
  @Test
  void badExample() {
      UserService service = init(UserService.class); // лишняя инициализация
      // ...
  }
  ```
  
- **Использование `@Test(dependsOnMethods = "...")` в TestNG без учёта параллелизма** — создаёт скрытые блокировки и deadlocks 
  при `parallel="methods"`.
- **Размещение `Allure.addAttachment()` внутри циклов без фильтрации** — генерирует тысячи артефактов и ломает генерацию отчёта.

---