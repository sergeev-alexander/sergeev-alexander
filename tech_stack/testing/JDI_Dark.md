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

Аннотации в JDI Dark делятся на две категории: **классового уровня** (применяются ко всему сервису) 
и **методного уровня** (переопределяют или дополняют настройки для конкретного эндпоинта).

## Аннотации классового уровня

Применяются к интерфейсу или классу, помеченному `@Service`. Задают глобальную конфигурацию для всех запросов сервиса.

- `@Service` — базовая аннотация, регистрирующая тип как источник декларативных эндпоинтов. Фреймворк сканирует только помеченные классы при инициализации.
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

- `@Header` (на классе) — глобальный заголовок для всех методов сервиса. 

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

Применяются к методам внутри `@Endpoint`-класса. Переопределяют или дополняют настройки классового уровня.

- `@Endpoint` — определяет базовый путь ресурса относительно `@ServiceDomain`. Все методы внутри класса наследуют этот путь.
    ```java
    @Endpoint("/users") // базовый путь: /users
    ```
- `@Path` — задаёт динамическую часть URL с поддержкой плейсхолдеров `{param}`. Значение подставляется из аргумента метода с аннотацией `@Path("param")`.
    ```java
    @Path("/{userId}/orders") // при userId=42 → /users/42/orders
    ```
- `@Query` — добавляет параметр в строку запроса. Поддерживает плейсхолдеры и резолвинг из конфига.
    ```java
    @Query("details") // ?details={значение_аргумента}
    @Query("api_key=${API_KEY}") // ?api_key=значение_из_переменной_окружения
    ```
- `@QueryParameter` (на методе) — локальное дополнение или переопределение классовых query-параметров.
    ```java
    @QueryParameter(name = "filter", value = "active") // добавит ?filter=active только к этому методу
    ```
- `@Header` (на методе) — локальный заголовок, имеет приоритет над классовым. Позволяет передавать кастомные токены или флаги трассировки.
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
// Файл: src/test/java/com/project/api/endpoints/UserApi.java

// @Service регистрирует интерфейс как источник эндпоинтов
// @ServiceDomain подставляет базовый URL из test.properties (${api.users})
// @QueryParameter добавляет глобальный параметр api_version=v2 ко всем запросам
// @Header задаёт глобальный Accept-заголовок
@Service
@ServiceDomain("${api.users}")
@QueryParameter(name = "api_version", value = "v2")
@Header(name = "Accept", value = "application/json")
public interface UserApi {

    // Все методы внутри Users наследуют базовый путь /users из @Endpoint
    @Endpoint("/users")
    public class Users {

        // GET /users/{id}?api_version=v2&details=full
        // @Path подставляет userId в URL
        // @Query добавляет параметр details из аргумента метода
        @GET
        @Path("/{userId}")
        @Query("details")
        Response<UserProfile> getUserById(
                @Path("userId") String id,        // подставляется в {userId}
                @Query("details") String expand   // добавляется как ?details=full
        );

        // POST /users?api_version=v2&source=mobile
        // @Body сериализует POJO в JSON
        // @Header переопределяет Authorization только для этого запроса
        @POST
        @QueryParameter(name = "source", value = "mobile")
        @Header(name = "Authorization", value = "Bearer ${auth_token}")
        Response<UserProfile> createUser(
                @Body UserCreateRequest payload   // автосериализация в JSON
        );

        // PUT /users/{id}/avatar — загрузка файла через multipart
        // @Multipart выставляет Content-Type: multipart/form-data
        // @FilePart описывает бинарную часть, @Part — текстовые метаданные
        @PUT
        @Path("/{userId}/avatar")
        @Multipart
        Response<UploadResult> uploadAvatar(
                @Path("userId") String id,
                @FilePart(controlName = "file", fileName = "avatar.jpg") File image,
                @Part(name = "description") String description
        );

        // DELETE /users/{id} с кастомным таймаутом и повторами при 502/503
        @DELETE
        @Path("/{userId}")
        @Timeout(connection = 3000, read = 15000)
        @RetryOnFailure(maxAttempts = 2, statusCodes = {502, 503})
        Response<Void> deleteUser(@Path("userId") String id);
    }
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
│  3. @ServiceDomain / @QueryParameter (классовый уровень)                │
│     │                                                                   │
│     ├─► @ServiceDomain("${api.base}")                                   │
│     ├─► @QueryParameter(name="v", value="2")                            │
│     └─► Применяется ко всем @Endpoint внутри интерфейса                 │
│                                                                         │
│  4. @Endpoint (уровень ресурса)                                         │
│     │                                                                   │
│     ├─► @Endpoint("/users") + @Header("X-Resource", "3")                │
│     └─► Применяется ко всем методам внутри класса Users                 │
│                                                                         │
│  5. @Header / @QueryParameter / @Path (уровень метода) ← НАИВЫСШИЙ      │
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

- **Заголовки**: метод > ресурс > сервис > глобальные. При совпадении имён — переопределение, при разных — объединение.
- **Query-параметры**: метод > класс. Дубликаты имён **не объединяются** — локальный параметр заменяет глобальный.
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
Response<UserProfile> profileResponse = userApi.Users.getUserById("123", "full");

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

- **Выносите `@Endpoint`-интерфейсы в отдельный пакет** `com.project.api.endpoints` и не смешивайте их с логикой тестов — это упрощает навигацию и рефакторинг.
- **Используйте `Response<T>` вместо `void` или `String`** — это сохраняет типобезопасность, даёт доступ к заголовкам и метрикам выполнения.
- **Используйте `@Path` с плейсхолдерами** вместо ручной сборки строк `"/users/" + id` — фреймворк проверяет корректность путей на этапе инициализации и защищает от опечаток.
- **Избегайте хардкода чувствительных значений** в `@Header` или `@QueryParameter` — передавайте токены и ключи через параметры методов или переменные окружения `${API_KEY}`.
- **Для сложной модификации запроса** (динамические заголовки, условная логика) используйте `JDIHttpClient.call()` напрямую или кастомные интерцепторы, а не перегружайте аннотации.
- **Группируйте связанные эндпоинты** во вложенные классы внутри `@Service` (как `Users`, `Orders`) — это улучшает структуру кода и автодополнение в IDE.
- **Документируйте плейсхолдеры** в `@Body`-шаблонах через JavaDoc параметров метода — это помогает новым членам команды понимать, какие аргументы куда подставляются.

### Антипаттерны

- **Хардкод доменов** в `@ServiceDomain("http://...")` вместо `${api.base}` — усложняет переключение между окружениями и требует изменения кода при деплое.
- **Дублирование `@QueryParameter`** на классе и методе с одинаковым именем — создаёт путаницу в приоритетах; используйте явное переопределение с комментарием.
- **Использование `@Body String` для простых объектов** вместо POJO — отключает автосериализацию, увеличивает риск ошибок в ручном формате JSON.
- **Игнорирование `@Timeout`** для долгих эндпоинтов — приводит к преждевременным `SocketTimeoutException` в CI-средах с ограниченной пропускной способностью.
- **Смешивание `@Form` и `@Body`** в одном методе — вызывает конфликт `Content-Type`; выберите один способ передачи данных.

---

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

## 7. Валидация и Asserts

> Валидация ответов — критический этап API-тестирования. 
> 
> JDI Dark предоставляет декларативный и fluent-интерфейс для проверок, глубоко интегрированный с Hamcrest, 
> JUnit / TestNG и встроенным логгером. 
> 
> Подход ориентирован на читаемость, быстрый фидбэк и лёгкую отладку.

### Проверка статус-кодов и заголовков

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

### Валидация тела ответа и JSONPath

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

### Hamcrest-матчеры и кастомные ассерты

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

### Логирование запросов и ответов

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

### Best Practices

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