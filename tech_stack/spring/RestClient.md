# RestClient

> ### `RestClient` - это HTTP-клиент, введённый в Spring 6.1 (Spring Boot 3.2+) для выполнения HTTP -запросов к внешним REST API.

- **Fluent-интерфейс**: цепочка вызовов (`get().uri(...).retrieve().body()`), легко читается и пишется.
- **Типобезопасность**: встроенный механизм преобразования тел запроса/ответа через `HttpMessageConverter` (например,
  Jackson).
- **Глубокая интеграция с Spring**: автоматическое использование настроенного `ObjectMapper`, поддержка
  `@Configuration`, логирование, обработка ошибок.
- **Иммутабельность и потокобезопасность**: `RestClient` можно безопасно использовать как singleton (внедрять через
  Spring DI).
- **Лёгкая настройка**: базовый URL, заголовки по умолчанию, таймауты - всё настраивается один раз при создании.
- **Блокирующий** (в отличие от `WebClient`). **Не реактивный клиент** - подходит для Spring MVC-приложений.
- Требует зависимости `spring-web`.

<details>
    <summary>
        <b>Использование</b>
    </summary>

## Использование и поддержка HTTP-методов

`RestClient` предоставляет **единый fluent-интерфейс** для всех HTTP-методов: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` и
др.  
Все запросы строятся по схеме:

> ```### restClient.метод() → uri() → (headers/body если нужно) → retrieve() → body() или toEntity()```

### Создание `RestClient`

```java
// Простейший вариант - без настройки
RestClient client = RestClient.create();

// Или с базовым URL (рекомендуется для одного API)
RestClient client = RestClient.builder()
        .baseUrl("https://api.example.com")
        .build();
```

### GET — получение данных

```java
// Получить объект напрямую
User user = client
                .get()
                .uri("/users/{id}", 123)
                .retrieve()
                .body(User.class);

// Получить вместе со статусом и заголовками
ResponseEntity<User> response = client
        .get()
        .uri("/users/{id}", 123)
        .retrieve()
        .toEntity(User.class);
```

### POST — создание ресурса

```java
User newUser = new User("Alice", "alice@example.com");

// Отправить объект как JSON (автоматически через Jackson)
User created = client
        .post()
        .uri("/users")
        .body(newUser)            // тело запроса
        .retrieve()
        .body(User.class);

// Или с явным указанием заголовков
User created = client
        .post()
        .uri("/users")
        .header("Authorization", "Bearer abc123")
        .body(newUser)
        .retrieve()
        .body(User.class);
```

### PUT / PATCH — обновление

```java
// Полное обновление (PUT)
client.put()
    .uri("/users/{id}",123)
    .body(updatedUser)
    .retrieve()
    .toBodilessEntity(); // если ответ пустой (204 No Content)

// Частичное обновление (PATCH)
Map<String, Object> patch = Map.of("email", "new@example.com");
client.patch()
    .uri("/users/{id}",123)
    .body(patch)
    .retrieve()
    .toBodilessEntity();
```

### DELETE — удаление

```java
// Простое удаление
client.delete()
    .uri("/users/{id}",123)
    .retrieve()
    .toBodilessEntity();

// С заголовками
client.delete()
    .uri("/users/{id}",123)
    .header("If-Match","\"abc123\"")
    .retrieve()
    .toBodilessEntity();
```

#### Используйте .toBodilessEntity() для операций без тела ответа (например, DELETE, PUT), чтобы избежать исключений при пустом ответе.

</details>

<details>
    <summary>
        <b>Обработка ошибок через onStatus() и DefaultResponseErrorHandler</b>
    </summary>

## 🔹 Обработка ошибок и кастомизация через `onStatus()` и `DefaultResponseErrorHandler`

По умолчанию `RestClient` **автоматически выбрасывает `RestClientException`** (или его подклассы) при получении
HTTP-статусов **4xx и 5xx**.  
Но часто нужно **перехватить ошибку**, извлечь детали из тела ответа или выбросить **кастомное исключение** — например,
`UserNotFoundException` или `PaymentServiceUnavailableException`.

Для этого используется метод **`onStatus()`** или настройка **глобального обработчика ошибок**.

---

### По умолчанию `RestClient` **автоматически выбрасывает `RestClientException`

** (или его подклассы) при получении HTTP-статусов **4xx и 5xx**.

```java
User user = client.get()
        .uri("/users/999")  // такого пользователя нет → 404
        .retrieve()
        .body(User.class);  // Вызовет RestClientResponseException с сообщением вроде:
//404 Not Found: [no body]
```

### Кастомный обработчик через `onStatus()`

```java
User user = client.get()
        .uri("/users/{id}", userId)
        .retrieve()
        .onStatus(
                // Условие: когда вызывать обработчик
                HttpStatusCode::is4xxClientError,
                // Обработчик: (request, response) -> выбросить исключение
                (request, response) -> {
                    String errorCode = response.getHeaders().getFirst("X-Error-Code");
                    throw new ExternalApiException("Ошибка внешнего API: " + errorCode);
                }
        )
        .body(User.class);
```

- `HttpStatusCode::is4xxClientError`
- `HttpStatusCode::is5xxServerError`
- `(status, _) -> status.value() == 404`
- любые предикаты.

### Обработка с чтением тела ошибки

Если внешний API возвращает JSON с деталями ошибки, вы можете его прочитать:

```java
// DTO для ошибки
public record ApiError(String message, String code) {
}

User user = client.get()
        .uri("/users/{id}", userId)
        .retrieve()
        .onStatus(HttpStatusCode::isError, (request, response) -> {
                    try {
                        // Читаем тело ошибки как ApiError
                        ApiError error = response.getBody(ApiError.class);
                        throw new ExternalServiceException(error.code(), error.message());
                    } catch (IOException e) {
                        throw new RuntimeException("Не удалось прочитать тело ошибки", e);
                    }
                }
        )
        .body(User.class);
```

### Глобальный обработчик ошибок (через builder)

Если вы хотите единое поведение для всего клиента, настройте DefaultResponseErrorHandler:

```java
RestClient client = RestClient.builder()
        .baseUrl("https://api.example.com")
        .defaultStatusHandler(
                HttpStatusCode::isError,
                (request, response) -> {
                    // Общий обработчик для всех запросов
                    throw new ExternalApiException("API вернул ошибку: " + response.getStatusCode());
                }
        )
        .build();
```

