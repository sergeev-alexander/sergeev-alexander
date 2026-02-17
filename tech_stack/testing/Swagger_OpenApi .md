# Swagger / OpenAPI Specification

> **Swagger** — набор инструментов для работы с **OpenAPI Specification** — стандартом описания REST API.

**OpenAPI Specification (OAS)** — формат YAML/JSON для описания REST API (endpoints, параметры, схемы данных, security).

**Swagger Tools** — инструменты от SmartBear для работы с OpenAPI:

- **Swagger UI** — интерактивная веб-документация API
- **Swagger Editor** — редактор для написания спецификаций
- **Swagger Codegen** — генерация клиентов/серверов из спецификации

**Версии:**

- OpenAPI 2.0 (= Swagger 2.0) — устаревшая
- OpenAPI 3.0 — текущая стабильная
- OpenAPI 3.1 — совместима с JSON Schema 2020-12

---

## 1. OpenAPI Specification

Пример структуры .yaml (или .json) файла:

```text
openapi.yaml  (или swagger.yaml, api-spec.yaml)
├── openapi: 3.0.3
├── info: {...}
├── servers: [...]
├── paths:
│   ├── /users: {...}
│   ├── /users/{id}: {...}
│   ├── /orders: {...}
│   └── /orders/{id}: {...}
├── components:
│   ├── schemas:
│   │   ├── User: {...}
│   │   ├── Order: {...}
│   │   └── Error: {...}
│   ├── parameters: {...}
│   ├── responses: {...}
│   └── securitySchemes: {...}
├── security: [...]
└── tags: [...]
```

### Структура документа

OpenAPI документ состоит из корневых секций:

```yaml
openapi: 3.0.3 # Версия спецификации (обязательно)
info: # Метаданные API (обязательно)
  title: My API
  version: 1.0.0
  description: API description
  contact:
    name: API Support
    email: support@example.com
  license:
    name: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0.html

servers: # Список серверов (опционально)
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server
  - url: http://localhost:8080
    description: Local development

paths: # Endpoints API (обязательно)
  /users:
    get:
      summary: Get all users
      # ...
  /users/{id}:
    get:
      summary: Get user by ID
      # ...

components: # Переиспользуемые компоненты (опционально)
  schemas: # Схемы данных
    User:
      type: object
      # ...
  parameters: # Параметры
    PageParam:
      name: page
      # ...
  responses: # Ответы
    NotFound:
      description: Resource not found
      # ...
  securitySchemes: # Схемы безопасности
    bearerAuth:
      type: http
      scheme: bearer
      # ...

security: # Глобальная безопасность (опционально)
  - bearerAuth: [ ]

tags: # Теги для группировки endpoints (опционально)
  - name: users
    description: User management
  - name: orders
    description: Order management
```

---

#### `info` — Метаданные API

Обязательная секция с информацией об API:

```yaml
info:
  title: My API                   # Название API (обязательно)
  version: 1.0.0                  # Версия API (обязательно)
  description: |                  # Описание (опционально, поддерживает Markdown)
    This is a **sample** API for demonstration.

    Features:
    - User management
    - Order processing
  termsOfService: https://example.com/terms  # Условия использования
  contact: # Контактная информация
    name: API Support Team
    email: support@example.com
    url: https://example.com/support
  license: # Лицензия
    name: MIT
    url: https://opensource.org/licenses/MIT
```

---

#### `servers` — Список серверов

Определяет базовые URL для API:

```yaml
servers:
  - url: https://api.example.com/v1
    description: Production server

  - url: https://{environment}.example.com/v1  # Переменные в URL
    description: Environment-specific server
    variables:
      environment:
        default: staging
        enum:
          - dev
          - staging
          - prod
        description: Environment name

  - url: http://localhost:{port}
    description: Local development
    variables:
      port:
        default: '8080'
        description: Server port
```

**Примечания:**

- Если `servers` не указан, используется `url: /` (относительный путь)
- Переменные заменяются в runtime (например, в Swagger UI можно выбрать значение)
- Все endpoints наследуют базовый URL из `servers`

---

#### `paths` — Endpoints API

Основная секция, описывающая все endpoints:

```yaml
paths:
  /users: # Path (начинается с /)
    get: # HTTP метод
      summary: Get all users      # Краткое описание
      description: Returns a list of users  # Подробное описание
      operationId: getUsers       # Уникальный ID операции (для кодогенерации)
      tags: # Теги для группировки
        - users
      parameters: # Параметры запроса
        - name: page
          in: query
          schema:
            type: integer
            default: 0
      responses: # Возможные ответы
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
        '400':
          description: Bad request

    post: # Другой метод на том же path
      summary: Create user
      tags:
        - users
      requestBody: # Тело запроса
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserCreate'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

  /users/{id}: # Path с параметром
    parameters: # Параметры на уровне path (для всех методов)
      - name: id
        in: path
        required: true
        schema:
          type: integer
          format: int64
    get:
      summary: Get user by ID
      tags:
        - users
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'
    delete:
      summary: Delete user
      tags:
        - users
      responses:
        '204':
          description: User deleted
        '404':
          $ref: '#/components/responses/NotFound'
```

**HTTP методы:** `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `trace`

---

#### `components` — Переиспользуемые компоненты

Секция для определения переиспользуемых элементов:

```yaml
components:
  schemas: # Схемы данных (модели)
    User:
      type: object
      required:
        - id
        - username
      properties:
        id:
          type: integer
          format: int64
          example: 1
        username:
          type: string
          minLength: 3
          maxLength: 50
          example: john_doe
        email:
          type: string
          format: email
          example: john@example.com
        createdAt:
          type: string
          format: date-time

    UserCreate:
      type: object
      required:
        - username
        - email
      properties:
        username:
          type: string
        email:
          type: string
          format: email

  parameters: # Параметры
    PageParam:
      name: page
      in: query
      description: Page number
      schema:
        type: integer
        minimum: 0
        default: 0

    LimitParam:
      name: limit
      in: query
      description: Items per page
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  responses: # Ответы
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: string
                example: Resource not found

    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: string
                example: Invalid or missing token

  securitySchemes: # Схемы безопасности
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://example.com/oauth/authorize
          tokenUrl: https://example.com/oauth/token
          scopes:
            read:users: Read user data
            write:users: Modify user data
```

**Ссылки на компоненты:** `$ref: '#/components/schemas/User'`

---

### Ключевые концепции

**1. `$ref` — Ссылки на переиспользуемые компоненты**

Ссылки могут указывать как на секции внутри файла:

```yaml
$ref: '#/components/schemas/User' # Символ "#" означает корень текущего документа
```

Так и на внешние файлы:

```yaml
$ref: './schemas/User.yaml'  # Внешняя ссылка
``` 

#### Использование:

```yaml
# Определение
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer

# Использование
paths:
  /users/{id}:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'  # Ссылка
```

**2. `operationId` — Уникальный идентификатор операции**

Используется для генерации кода (имена методов в клиентах):

```yaml
paths:
  /users:
    get:
      operationId: getAllUsers    # → client.getAllUsers()
    post:
      operationId: createUser     # → client.createUser(user)
```

**3. `tags` — Группировка endpoints**

```yaml
tags:
  - name: users
    description: User management operations
  - name: orders
    description: Order management operations

paths:
  /users:
    get:
      tags:
        - users    # Группируется под "users" в Swagger UI
```

**4. `deprecated` — Пометка устаревших endpoints**

```yaml
paths:
  /old-endpoint:
    get:
      deprecated: true
      summary: Old endpoint (use /new-endpoint instead)
```

---

## 2. Безопасность в OpenAPI

Безопасность в OpenAPI описывается через секцию `components/securitySchemes` и применяется глобально или на уровне
отдельных операций с помощью поля `security`.

### Scopes (Области доступа)

> Scopes - это конкретные разрешения, которые запрашивает приложение у пользователя.

Когда пользователь авторизуется, ему показывают экран примерно такого содержания:

```text
"Приложение запрашивает доступ к:
 - Чтению ваших профилей (read:users)
 - Редактированию ваших данных (write:users)"
```

Как это работает в OpenAPI:

```yaml
scopes:
  read:users: Read user profiles # Описание scope
  write:users: Modify user data # Описание scope
```

А в `security` секции мы указываем, какие именно из этих прав нужны для конкретного эндпоинта:

```yaml
/admin:
  get:
    security:
      - oauth2AuthCode: [ read:users ]  # Нужно только право на чтение
  post:
    security:
      - oauth2AuthCode: [ write:users ]  # Нужно право на запись
  delete:
    security:
      - oauth2AuthCode: [ read:users, write:users ]  # Нужны оба права
```

Когда вы видите `bearerAuth: [ ]` — это означает, что для данного эндпоинта не требуется конкретных `scopes`, достаточно
просто наличия валидного токена.

Зачем это нужно:

- Минимальные привилегии — клиент запрашивает только те права, которые реально нужны
- Безопасность — если токен скомпрометирован, ущерб ограничен (например, если это токен только на чтение, злоумышленник
  не
  сможет удалять данные)
- Прозрачность для пользователя — пользователь видит, что именно приложение сможет делать

Визуализация:

```text
Токен с правами [read:users, write:users]
├── GET /users/123 → разрешено (нужен read:users)
├── POST /users → разрешено (нужен write:users)
└── DELETE /users/123 → разрешено (нужны оба)

Токен только с правами [read:users]
├── GET /users/123 → разрешено
├── POST /users → ОТКАЗАНО
└── DELETE /users/123 → ОТКАЗАНО
```

---

## Типы схем безопасности

OpenAPI поддерживает четыре типа схем:

### 1. `http` — HTTP-аутентификация

Используется для стандартных HTTP-механизмов:

- **Bearer** (часто с JWT):

```yaml
  components:
    securitySchemes:
      bearerAuth:
        type: http
        scheme: bearer
        bearerFormat: JWT
```

- **Basic**:

```yaml
  components:
    securitySchemes:
      basicAuth:
        type: http
        scheme: basic
```

---

### 2. `apiKey` — API-ключ

Передаётся:

- В заголовке:

```yaml
    components:
      securitySchemes:
        apiKeyHeader:
          type: apiKey
          in: header
          name: X-API-Key
```

- В query-параметре:

```yaml
    components:
      securitySchemes:
        apiKeyQuery:
          type: apiKey
          in: query
          name: api_key
```

- Или в cookie:

```yaml
    components:
      securitySchemes:
        apiKeyCookie:
          type: apiKey
          in: cookie
          name: session_id
```

---

### 3. `oauth2` — OAuth 2.0

Поддерживает все стандартные flow.

Пример с authorization code:

```yaml
    components:
      securitySchemes:
        oauth2AuthCode:
          type: oauth2
          flows:
            authorizationCode:
              authorizationUrl: https://example.com/oauth/authorize
              tokenUrl: https://example.com/oauth/token
              scopes:
                read:users: Read user profiles
                write:users: Modify user data
```

В протоколе OAuth 2.0 с типом `authorizationCode` существует два этапа:

- Получение кода (Front-channel):

    - Пользователя перенаправляют в браузер на `authorizationUrl`.
    - Он там вводит логин/пароль и видит запрос разрешений (`scopes: read:users`).
    - После одобрения сервер авторизации через браузер (редирект) возвращает временный код.

- Обмен кода на токен (`Back-channel`):

    - Браузер отдает этот код вашему бэкенду.
    - Ваш сервер (бэкенд) теперь сам, без участия браузера пользователя, обращается по адресу `tokenUrl` (
      например, https://example.com/oauth/token), отправляя этот код + секретный ключ клиента (`client_secret`).
    - В ответ сервер авторизации выдает `access_token`.

Другие flow:

- `clientCredentials` — для machine-to-machine
- `implicit` — устаревший (не рекомендуется)
- `password` — устаревший (RFC 6749 deprecated)

---

### 4. `openIdConnect` — OpenID Connect (openIdConnect) в OpenAPI

Этот тип `security scheme` используется для аутентификации через `OpenID Connect (OIDC)` — протокол, построенный поверх
`OAuth 2.0`.

`openIdConnectUrl` — это ссылка на документ обнаружения (`discovery document`). Это стандартизированный JSON-документ,
который содержит метаданные о провайдере `OpenID Connect`.

Указывает URL discovery-документа:

```yaml
    components:
      securitySchemes:
        openId:
          type: openIdConnect
          openIdConnectUrl: https://example.com/.well-known/openid-configuration
```

В отличие от обычного `OAuth 2.0`, где мы вручную указываем `authorizationUrl` и `tokenUrl`, `OpenID Connect` использует
автоматическое обнаружение. По этому URL можно получить JSON вида:

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth/authorize",
  "token_endpoint": "https://example.com/oauth/token",
  "userinfo_endpoint": "https://example.com/oauth/userinfo",
  "jwks_uri": "https://example.com/oauth/jwks",
  "response_types_supported": [
    "code",
    "id_token"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "scopes_supported": [
    "openid",
    "profile",
    "email"
  ]
}
```

Используйте `openIdConnect`, если ваш API использует OIDC-провайдера для аутентификации пользователей. Это особенно
актуально для:

- API, работающих с identity-провайдерами (Google, Microsoft, Auth0)
- Микросервисных архитектур с единой системой аутентификации
- Приложений, где нужна информация о пользователе (не только права доступа)

---

## Применение схем безопасности

Схемы применяются через поле `security`, которое может быть указано:

- **Глобально** — в корне документа (влияет на все операции):

```yaml
  security:
    - bearerAuth: [ ]
```

- **На уровне операции** — переопределяет глобальные настройки:

```yaml
  paths:
  /admin:
  get:
  security:
    - oauth2AuthCode: [ read:users ]
  # ...
```

Можно указать несколько схем — они интерпретируются как логическое **ИЛИ**:

```yaml
    security:
      - bearerAuth: [ ]
      - apiKeyHeader: [ ]
```

Это означает: «доступ разрешён либо по токену, либо по API-ключу».

Если операция не требует аутентификации, явно укажите пустой массив:

```yaml
    paths:
      /public:
        get:
          security: [ ]
          # ...
```

### Best practices по безопасности

- **Не храните секреты** в спецификации: используйте placeholder’ы (`{token}`, `{api_key}`) только в примерах.
- **Разделяйте окружения**: dev/staging могут использовать упрощённую аутентификацию, production — строгую.
- **Ограничивайте scope-ы** в OAuth2: запрашивайте минимально необходимые права.
- **Используйте HTTPS** в production средах.
- **Документируйте требования** к формату токена (например, `bearerFormat: JWT`).
- **Тестируйте схемы безопасности** в рамках contract testing: проверяйте, что защищённые endpoint-ы действительно
  требуют credentials.

---

## 3. Валидация и качество спецификации

Качественная OpenAPI-спецификация — это не только корректный `YAML` / `JSON`, но и документ, который точно описывает
поведение
API, поддерживает тестирование и понятен другим разработчикам.

## Инструменты валидации

### `swagger-cli`

Официальный CLI-инструмент от Swagger для проверки валидности спецификации:

```text
    npx swagger-cli validate openapi.yaml
```

Возвращает ошибку, если:

- Нарушена структура `OAS`
- Используется несуществующий `$ref`
- Типы данных не соответствуют схеме

---

### `Spectral`

Правило-ориентированный линтер от Stoplight (поддерживает OpenAPI, AsyncAPI):

```text
    npx @stoplight/spectral-cli lint openapi.yaml
```

Позволяет:

- Применять встроенные правила (например, `oas3-schema`, `operation-description`)
- Писать кастомные правила (например, «все endpoint-ы должны иметь `operationId`»)
- Интегрировать в CI/CD

Пример правила в `.spectral.yaml`:

```yaml
rules:
  operation-id-unique:
    description: "operationId must be unique"
    given: "$.paths[*][*]"
    then:
      function: schema
      functionOptions:
        schema:
          required: [ operationId ]
```

---

### `speccy` (устаревает)

Ранее популярный линтер, сейчас в основном заменён Spectral.

---

### Распространённые ошибки в спецификациях

- **Неполные схемы ответов**:  
  Ответ `200` описан, но `400`, `401`, `500` — нет. Это затрудняет обработку ошибок на клиенте.

- **Отсутствие `required` в схемах**:  
  Все поля помечены как опциональные, хотя в реальности API требует обязательные.

- **Несогласованность между `requestBody` и `responses`**:  
  Например, в `POST /users` ожидается `UserCreate`, но в ответе возвращается `User` без указания разницы.

- **Избыточное использование `type: object` без `properties`**:  
  Такая схема бесполезна для генерации кода и валидации.

- **Неверные форматы дат/чисел**:  
  Указание `format: date-time` без соответствия ISO 8601 может привести к ошибкам в сериализации.

## Contract testing: соответствие спецификации и реализации

Валидация синтаксиса — недостаточна. Необходимо проверять, что **реальное поведение API совпадает со спецификацией**.

---

### Dredd

Запускает HTTP-запросы на основе OpenAPI и сравнивает фактические ответы с ожидаемыми:

```text
    dredd openapi.yaml http://localhost:8080
```

Поддерживает:

- Проверку статус-кодов
- Сравнение структуры JSON-ответов
- Hook-и для подстановки токенов, очистки БД и т.п.

---

### Prism (от Stoplight)

Может работать в двух режимах:

- **Mock server**: имитирует API на основе спецификации
- **Validation proxy**: перехватывает реальные запросы и проверяет их соответствие спецификации

Запуск в режиме прокси:

```text
npx @stoplight/prism-cli proxy openapi.yaml --target http://localhost:8080
```

Если запрос или ответ не соответствует спецификации — Prism возвращает ошибку валидации.

---

### Best practices по качеству

- **Описывайте все возможные HTTP-статусы**, особенно ошибки (`4xx`, `5xx`).
- **Используйте `example` и `examples`** — они улучшают документацию и помогают в тестировании.
- **Следите за согласованностью типов**: например, `id` всегда `integer` или всегда `string`.
- **Пишите описания на всех уровнях**: `summary`, `description` для операций, параметров, схем.
- **Автоматизируйте проверку качества** в CI: запускайте `swagger-cli validate` и `spectral lint` на каждый коммит.

---

## 4. Интеграция с Java/Spring Boot (code-first подход)

В enterprise-разработке на Java часто используется **code-first** подход: спецификация OpenAPI генерируется
автоматически из исходного кода контроллеров и моделей. Это снижает риск рассинхронизации между реализацией и
документацией.

### Выбор библиотеки

- **Springfox** — устаревший, не поддерживает Spring Boot 3+, проблемы с совместимостью.
- **springdoc-openapi** — современная, активно поддерживаемая альтернатива. Работает с:
    - Spring Web MVC
    - Spring WebFlux
    - Spring Boot 2.x и 3.x
    - Jakarta EE 9+ (в т.ч. `jakarta.annotation`)

Добавление зависимости (Maven):

```xml

<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

После запуска приложения:

- Спецификация доступна по `/v3/api-docs`
- Swagger UI — по `/swagger-ui.html`

### Аннотации для тонкой настройки

Без аннотаций `springdoc` генерирует базовую спецификацию. Для production-уровня требуется явное управление через
аннотации.

---

#### `@Operation` — описание операции

```java

@Operation(
        summary = "Get user by ID",
        description = "Returns a user if found, otherwise 404",
        operationId = "getUserById",
        tags = {"users"}
)
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { /* ... */ }
```

---

#### `@Parameter` — описание параметров

`springdoc-openapi` сканирует ваш код и ищет:

- Классы с аннотацией `@RestController` или `@Controller`
- Методы с аннотациями `@GetMapping`, `@PostMapping` и т.д.

Только в этих методах он обрабатывает `@Parameter`, `@Operation` и другие OpenAPI-аннотации

```text
public ReturnType methodName(
    @Parameter(...) 
    @PathVariable / @RequestParam / @RequestHeader Type parameterName
)
``` 

Для переменной пути/запроса:

```java

@RestController // ← обязательно должен быть @Controller или @RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUserById(
            @Parameter(
                    name = "id",
                    description = "User ID (must be positive)",
                    required = true,
                    example = "123"
            )
            @PathVariable Long id
    ) {
        // логика получения пользователя
        return userService.findUser(id);
    }
}
```

Для query-параметров:

```java


@RestController // ← обязательно должен быть @Controller или @RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/search")
    public List<User> searchUsers(
            @Parameter(
                    name = "name",
                    description = "Search by user name (partial match)",
                    example = "john"
            )
            @RequestParam String name,

            @Parameter(
                    name = "age",
                    description = "Filter by minimum age",
                    required = false,
                    example = "18"
            )
            @RequestParam(required = false) Integer age
    ) {
        // логика поиска
    }
}
```

Для header-параметров:

```java

@RestController // ← обязательно должен быть @Controller или @RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/profile")
    public UserProfile getProfile(
            @Parameter(
                    name = "X-API-Version",
                    description = "API version header",
                    required = true,
                    example = "1.0"
            )
            @RequestHeader("X-API-Version") String apiVersion
    ) {
        // логика
    }
}
```

Можно также описывать параметры на уровне метода, но это более громоздко:

```java

@RestController // ← обязательно должен быть @Controller или @RestController
@RequestMapping("/api/users")
public class UserController {

    @Operation(parameters = {
            @Parameter(name = "id", description = "User ID", required = true),
            @Parameter(name = "includeDetails", description = "Include additional info")
    })
    @GetMapping("/{id}")
    public User getUserWithParams(
            @PathVariable Long id,
            @RequestParam boolean includeDetails
    ) {
        // логика
    }
}
```

---

### `@Schema` — описание модели или поля

```java
public class User {

    @Schema(description = "Unique user identifier", example = "42")
    private Long id;

    @Schema(description = "Username (3–50 chars)", minLength = 3, maxLength = 50)
    private String username;
}
```

Также используется для описания вложенных объектов, массивов, enum-ов.

#### `@ApiResponse` — описание ответов

```java

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Operation(summary = "Create user")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "201", description = "User created",
                    content = @Content(schema = @Schema(implementation = User.class))),
            @ApiResponse(responseCode = "400", description = "Invalid input",
                    content = @Content(schema = @Schema(implementation = Error.class)))
    })
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody @Valid UserCreate dto) { /* ... */ }
```

---

### Глобальная конфигурация

Создайте `@Bean` типа `OpenApi` для настройки метаданных, серверов, security schemes:

```java

@Configuration
public class Configuration {

    @Bean
    public OpenApi customOpenApi() {
        return new OpenApi()
                .info(new Info()
                        .title("User Management API")
                        .version("1.0.0")
                        .description("Production-grade user API")
                        .contact(new Contact().name("API Team").email("api@example.com"))
                        .license(new License().name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0.html")))
                .servers(List.of(
                        new Server().url("https://api.example.com/v1").description("Production"),
                        new Server().url("http://localhost:8080").description("Local dev")
                ))
                .components(new Components()
                        .addSecuritySchemes("bearerAuth",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")));
    }
}
```

Глобальная безопасность (применяется ко всем endpoint-ам по умолчанию):

```java
    .addSecurityItem(new SecurityRequirement().

addList("bearerAuth"))
```

---

### Поддержка в IntelliJ IDEA

- Аннотации `@Operation`, `@Schema` и др. распознаются как обычные Java-аннотации.
- Нет встроенной визуализации OpenAPI, но можно запустить приложение и открыть `/swagger-ui.html` в браузере
- YAML-спецификация (если используется design-first) подсвечивается корректно

---

### Best practices для code-first

- **Не полагайтесь только на автоматическую генерацию**: всегда проверяйте, что сгенерированная спецификация содержит
  все нужные `responses`, `examples`, `security`.
- **Используйте `@Valid` + Bean Validation**: `springdoc` автоматически преобразует `@NotNull`, `@Min`, `@Pattern` в
  ограничения схемы (`required`, `minimum`, `pattern`).
- **Избегайте анонимных типов** в контроллерах — они мешают генерации читаемых схем.
- **Тестируйте сгенерированную спецификацию**: сохраняйте `/v3/api-docs` в CI и сравнивайте с эталоном (например, через
  `diff` или contract testing).

---

## 5. Генерация кода и клиентов

OpenAPI-спецификация может служить источником для автоматической генерации:

- Клиентских SDK (Java, TypeScript, Python и др.)
- Серверного шаблонного кода
- Коллекций для Postman, Insomnia, HTTPie

Это ускоряет интеграцию, снижает количество ошибок и обеспечивает согласованность между сервисами.

> Автоматическая генерация кода из OpenAPI
>
> Это процесс создания готового кода на основе вашей OpenAPI-спецификации. Вместо того чтобы писать код вручную, вы "
> скармливаете" YAML/JSON файл генератору, и он создаёт готовые классы, интерфейсы и методы.

---

### 1. Генерация клиентских SDK

Представьте, у вас есть API с эндпоинтом:

```yaml
/user/{id}:
  get:
    operationId: getUserById
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: integer
    responses:
      200:
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/User"
```

Генератор создаст готовый код, который можно сразу использовать:

Для Java:

```java
// Сгенерированный клиент
ApiClient client = new ApiClient();
UserApi userApi = new UserApi(client);

// Готовый метод для вызова API
User user = userApi.getUserById(123); // ← просто вызываем!
```

Для TypeScript:

```typescript
// Сгенерированный клиент
const api = new UserApi();
const user = await api.getUserById({ id: 123 });
```

Что именно генерируется:

- Методы для каждого эндпоинта — `getUserById()`, `createUser()`, `deleteUser()`
- Модели данных — классы/интерфейсы для `User`, `Order`, `Product` и т.д.
- Код для аутентификации — автоматическая подстановка токенов
- Обработку ошибок — исключения для разных HTTP-статусов
- Сериализацию/десериализацию — автоматическое преобразование JSON в объекты

---

### 2. Генерация серверного шаблонного кода

Для того же эндпоинта генератор создаст скелет сервера:

Для Spring Boot:

```java

@RestController
public class UserApiController implements UserApi {

    @Override
    public ResponseEntity<User> getUserById(Long id) {
        User user = userService.findById(id); // ❗ ВАМ ОСТАЁТСЯ НАПИСАТЬ ТОЛЬКО ЭТО
        return ResponseEntity.ok(user); // ❗ ВАМ ОСТАЁТСЯ НАПИСАТЬ ТОЛЬКО ЭТО
        // Весь остальной код (маппинг URL, параметры, статусы) уже готов!
    }
}
```

Что уже готово:

- Контроллеры с правильными @GetMapping, @PostMapping
- Интерфейсы с описанием методов
- DTO классы (Data Transfer Objects)
- Конфигурация для Spring/других фреймворков

---

## Выбор генератора

---

### `openapi-generator` (рекомендуется)

- Активно поддерживается (более 200 генераторов)
- Поддерживает OpenAPI 3.0/3.1
- Гибкая настройка через конфигурационные файлы
- Интеграция с Maven/Gradle, Docker, CLI

GitHub: https://github.com/OpenAPITools/openapi-generator

---

### `swagger-codegen` (устаревает)

- Оригинальный инструмент от SmartBear
- Медленнее развивается, меньше поддержки OpenAPI 3.1
- Не рекомендуется для новых проектов

---

### Генерация Java-клиента

Пример команды (CLI):

```text
    npx @openapitools/openapi-generator-cli generate \
      -i openapi.yaml \
      -g java \
      -o ./sdk/java-client \
      --additional-properties=groupId=com.example,artifactId=api-client,javaPackage=com.example.client
```

Поддерживаемые HTTP-библиотеки через опцию `library`:

- `webclient` (Spring WebFlux)
- `feign` (Spring Cloud OpenFeign)
- `retrofit2`
- `native` (на базе `java.net.http.HttpClient`)

Пример конфигурации (`config.json`):

```json
    {
  "groupId": "com.example",
  "artifactId": "user-api-client",
  "javaPackage": "com.example.user.client",
  "library": "webclient",
  "dateLibrary": "java8",
  "hideGenerationTimestamp": true
}
```

Запуск с конфигурацией:

```text
    openapi-generator generate -i openapi.yaml -g java -c config.json -o ./client
```

---

## Генерация коллекций для тестирования

### Postman

```text
    openapi-generator generate -i openapi.yaml -g postman -o ./collections
```

Результат: `.json`-файл, который можно импортировать в Postman.  
Все endpoint-ы, параметры, тела запросов и ожидаемые ответы переносятся автоматически.

#### Insomnia

Использует тот же формат или импорт напрямую через UI (Insomnia поддерживает OpenAPI из коробки).

---

### Управление версиями и breaking changes

- **Семантическое версионирование**: если API меняет контракт (удаляет поле, меняет тип), это должно отражаться в версии
  спецификации (`info.version`).
- **Сравнение спецификаций**: используйте `openapi-diff` для выявления breaking изменений:

  ```text
  npx openapi-diff old.yaml new.yaml
  ```

- **Не перезаписывайте клиенты «вручную»**: все изменения должны вноситься в спецификацию, затем — регенерировать.
- **Храните сгенерированный код в отдельном репозитории или модуле**, чтобы избежать смешения с бизнес-логикой.

---

### Best practices

- **Генерируйте клиенты как часть CI/CD**, если они используются внутренними сервисами.
- **Используйте `operationId` осмысленно**: он становится именем метода в SDK.
- **Проверяйте качество сгенерированного кода**: иногда требуется кастомизация шаблонов (через `--template-dir`).
- **Не коммитьте временные/локальные клиенты** — только production-версии с зафиксированной спецификацией.
- **Документируйте процесс генерации** в README: какие команды, версии, параметры использовались.

---

## 6. Тестирование на основе OpenAPI

> OpenAPI-спецификация — не только документация, но и **контракт**, который можно использовать для автоматизированного
> тестирования: проверки корректности реализации, генерации данных, валидации безопасности и производительности.

### Инструменты для тестирования

---

#### Dredd

Выполняет HTTP-запросы на основе спецификации и сравнивает ответы с ожидаемыми:

```text
    dredd openapi.yaml http://localhost:8080 --hookfiles=hooks.js
```

Поддержка:

- Аутентификации через hook-и (установка заголовков)
- Подготовки/очистки данных (pre/post hooks)
- Проверки структуры JSON (через JSON Schema)

Ограничение: работает только с **синхронными** REST-эндпоинтами.

---

#### Prism (Stoplight)

Работает как **валидирующий прокси**:

```text
    prism proxy openapi.yaml --target http://localhost:8080
```

Любой запрос через Prism проверяется на соответствие спецификации:

- Запрос → валидируется по `requestBody`, параметрам
- Ответ → валидируется по `responses`

Если нарушение найдено — Prism возвращает ошибку валидации вместо проксирования.

---

#### Schemathesis

Property-based testing framework на Python для OpenAPI:

```text
    schemathesis run openapi.yaml --base-url=http://localhost:8080
```

Особенности:

- Автоматически генерирует сотни комбинаций входных данных
- Проверяет не только статус-коды, но и **инварианты** (например, «ответ всегда содержит `id`»)
- Поддерживает stateful-тестирование (цепочки запросов: create → get → delete)
- Интегрируется с pytest

Для Java нет прямого аналога, но можно использовать:

- `RestAssured` + ручная генерация данных по схеме
- `TestContainers` + запуск `Schemathesis` как внешнего процесса
- `Schemathesis CLI` через `Docker` даже в Java-проектах (часто так и делают)

### Интеграция в CI/CD

Рекомендуется включать в pipeline:

1. **Валидацию спецификации**: `swagger-cli validate`
2. **Linting**: `spectral lint`
3. **Contract tests**: запуск Dredd или Schemathesis против staging-окружения
4. **Сравнение с предыдущей версией**: `openapi-diff` для выявления breaking изменений

Пример шага в GitHub Actions:

```text
- name: Run contract tests
  run: |
  docker run --network=host stoplight/spectral lint openapi.yaml
  npx dredd openapi.yaml http://localhost:8080
```

---

## 7. Поддержка и эволюция спецификации в production

> OpenAPI-спецификация — это живой артефакт, который должен развиваться вместе с API.
>
> В production-среде особенно важно управлять её жизненным циклом, чтобы избежать нарушения совместимости, сбоев
> интеграций и потери доверия со стороны клиентов.

---

### Design-first vs Code-first: стратегии применения

#### Design-first

- Спецификация пишется **до реализации**.
- Используется как контракт между командами (бэкенд, фронтенд, QA).
- Позволяет параллельную разработку: фронтенд может использовать mock-сервер на основе спецификации.
- Подходит для:
    - Новых сервисов
    - Public API
    - Строгих регуляторных требований

#### Code-first

- Спецификация генерируется **из кода** (например, через `springdoc-openapi`).
- Упрощает поддержку при частых изменениях.
- Риск: спецификация может не отражать реальное поведение, если аннотации не обновлены.
- Подходит для:
    - Internal APIs
    - Быстро меняющихся прототипов
    - Микросервисов в единой экосистеме

> **Рекомендация**: даже в code-first подходе проводите ревью сгенерированной спецификации перед мержем в main.

---

### Версионирование API и спецификации

#### Версия в URL

Наиболее надёжный и прозрачный способ:

```yaml
servers:
  - url: https://api.example.com/v1
```

Все endpoint-ы относятся к одной версии. При breaking change — создаётся `/v2`.

#### Версия в заголовке

Менее наглядно, но позволяет избежать дублирования путей:

```text
GET /users
Accept-Version: v2
```

Требует явной поддержки в коде и документации.

#### Версия в OpenAPI (`info.version`)

Это **версия спецификации**, а не API!  
Не используйте её как единственное средство управления совместимостью.

Пример:

```yaml
info:
  title: User API
  version: 1.2.0 # ← версия документа, не контракта
```

### Обработка breaking changes

Breaking change — любое изменение, которое ломает существующих клиентов:

- Удаление endpoint-а или поля
- Изменение типа (`string` → `integer`)
- Ужесточение валидации (`minLength: 3` → `minLength: 10`)

#### Рекомендуемый процесс:

- **Пометить старый endpoint как deprecated**:

```yaml
    paths:
      /old-users:
        get:
          deprecated: true
          summary: Use /v2/users instead
```

- **Задокументировать миграционный путь** в описании или changelog.
- **Поддерживать обе версии параллельно** (обычно 3–6 месяцев).
- **Уведомить клиентов** (email, developer portal, changelog).
- **Удалить старую версию** только после подтверждения отсутствия трафика.

---

### Хранение спецификации

#### Вариант 1: В том же репозитории, что и код

- Плюсы: всегда синхронизирована с реализацией, легко включить в CI.
- Минусы: сложнее для внешних потребителей.

#### Вариант 2: Отдельный репозиторий («API registry»)

- Плюсы: единая точка для всех API, удобно для design-first.
- Минусы: риск рассинхронизации с кодом.

> **Best practice**: храните спецификацию рядом с кодом, но публикуйте её в централизованном портале (например, через
> Swagger UI + CI/CD).

---

### Документирование изменений

Создайте файл `CHANGELOG.md` рядом со спецификацией:

```md
    ## [1.2.0] — 2026-02-15
    ### Added
    - POST /v1/orders — create new order
    ### Changed
    - `email` in User schema now requires format: email
    ### Deprecated
    - GET /v1/old-users → use /v1/users
```

Используйте семантическое версионирование:

- `MAJOR` — breaking changes
- `MINOR` — новые функции, обратно совместимые
- `PATCH` — исправления ошибок в документации

### Best practices

- **Никогда не вносите breaking changes без предварительного deprecation**.
- **Автоматизируйте сравнение спецификаций** между релизами (`openapi-diff`).
- **Проверяйте, что все production-эндпоинты покрыты спецификацией** (можно через лог-анализ или middleware).
- **Используйте Git tags или releases** для фиксации версий спецификации.
- **Предоставляйте стабильный URL** к актуальной спецификации (например, `/api-docs/v1.yaml`), чтобы клиенты могли
  автоматически обновлять SDK.

---

## 8. Расширенные возможности и best practices

> Помимо базовой структуры и описания endpoint’ов, OpenAPI предоставляет ряд продвинутых возможностей, которые повышают
> качество документации, упрощают тестирование и улучшают developer experience.

---

### Использование `examples` и `x-examples`

#### Встроенные примеры (`example`)

Можно указать пример значения для параметра, тела запроса или поля схемы:

```yaml
    components:
      schemas:
        User:
          properties:
            email:
              type: string
              format: email
              example: alice@example.com # Пример
```              

#### Множественные примеры (`examples`)

Позволяют показать разные сценарии (успешный, ошибочный, edge case):

```yaml
    paths:
      /users:
        post:
          requestBody:
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/UserCreate'
                examples:
                  validUser:
                    summary: Valid user
                    value:
                      username: alice
                      email: alice@example.com
                  invalidEmail:
                    summary: Invalid email format
                    value:
                      username: bob
                      email: not-an-email
```

Эти примеры отображаются в Swagger UI и могут использоваться как основа для автотестов.

> Примечание: `x-examples` — нестандартное расширение, поддерживаемое некоторыми инструментами (например, устаревшие
> версии
> Swagger UI).
>
> Предпочтительнее использовать стандартное поле `examples`.

---

### Поддержка multipart/form-data и загрузки файлов

Для endpoint-ов с загрузкой файлов:

```yaml
    paths:
      /upload:
        post:
          requestBody:
            content:
              multipart/form-data:
                schema:
                  type: object
                  properties:
                    file:
                      type: string
                      format: binary
                    description:
                      type: string
          responses:
            '201':
              description: File uploaded
```

Swagger UI автоматически отображает форму с файловым input-ом.

---

### Описание асинхронных операций

OpenAPI изначально ориентирован на синхронные REST-запросы, но можно описать асинхронное поведение:

- Ответ `202 Accepted` + заголовок `Location`:

```yaml
    responses:
      '202':
        description: Task accepted
        headers:
          Location:
            schema:
              type: string
              example: /tasks/123
```

- Дополнительно: ссылка на endpoint получения статуса задачи через `links` (OpenAPI 3.0+):

```yaml
    responses:
      '202':
        description: Task accepted
        links:
          GetTaskStatus:
            operationId: getTask
            parameters:
              taskId: '$response.header.Location'
```

> Примечание: `links` редко поддерживаются генераторами кода, но полезны для документации.

---

### Производительность и масштабируемость

- **Разбивайте большую спецификацию** на несколько файлов с помощью `$ref`:

```yaml
    paths:
      /users:
        $ref: './paths/users.yaml'
```

- **Избегайте глубокой вложенности** в схемах — это замедляет генерацию клиентов и парсинг.
- **Ограничивайте размер файла**: >10k строк затрудняют навигацию в Swagger UI.
- **Используйте lazy loading** в UI: Swagger UI поддерживает динамическую подгрузку при больших спецификациях (через
  `configUrl`).

---

### Антипаттерны

| Антипаттерн                                            | Риск                                           | Решение                              |
|--------------------------------------------------------|------------------------------------------------|--------------------------------------|
| Сплошной `type: object` без `properties`               | Невозможно сгенерировать клиент, нет валидации | Всегда описывайте структуру          |
| Все поля опциональны                                   | Клиенты не знают, что обязательно              | Указывайте `required`                |
| Отсутствие описания ошибок (`4xx`, `5xx`)              | Сложно обрабатывать исключения                 | Добавляйте все возможные `responses` |
| Дублирование одинаковых ответов                        | Трудно поддерживать                            | Выносите в `components/responses`    |
| Использование `application/octet-stream` без пояснений | Неясно, что ожидается                          | Указывайте `schema` и `example`      |

---

### Принципы production-grade спецификации:

1. **Полнота**: все endpoint-ы, параметры, статусы, ошибки.
2. **Точность**: соответствие реальному поведению API.
3. **Согласованность**: единый стиль именования, форматов, тегов.
4. **Поддерживаемость**: модульная структура, версионирование, changelog.
5. **Тестируемость**: наличие примеров, валидных и невалидных сценариев.

> Следуя этим принципам, вы превращаете OpenAPI из простой документации в **инженерный контракт**, который служит основой
  для разработки, тестирования, мониторинга и интеграции в enterprise-среде.

--- 