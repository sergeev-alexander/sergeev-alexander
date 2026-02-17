# Swagger

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
    .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
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

