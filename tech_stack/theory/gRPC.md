**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# gRPC

> **RPC** (Remote Procedure Call) - это концепция: вызов функции/метода, который исполняется на удалённой машине, как
> будто
> он локальный.

> **gRPC** - это современный RPC-фреймворк с открытым исходным кодом, разработанный Google.
>
> Он позволяет сервисам вызывать
> методы друг друга **как будто это локальные функции**, даже если они работают на разных машинах, в разных языках и в
> разных сетях.

Основные особенности:

- Использует **HTTP/2** как транспорт (поддержка мультиплексирования, двунаправленного потока, сжатия заголовков)
- Сериализует данные в **Protocol Buffers (`protobuf`)** - компактный бинарный формат
- Поддерживает **четыре типа вызовов**:
    - `Unary`: один запрос → один ответ (аналог REST)
    - `Server streaming`: один запрос → поток ответов
    - `Client streaming`: поток запросов → один ответ
    - `Bidirectional streaming`: поток запросов ↔ поток ответов
- Генерирует **типобезопасный клиентский и серверный код** из единого контракта

### Зачем нужна генерация кода?

Представьте: у вас есть два микросервиса - `UserService` и `OrderService`.  
`OrderService` должен спросить у `UserService`: «Кто такой пользователь с ID=42?»

В классическом REST-подходе:

- Вы пишете HTTP-запрос вручную: `GET /users/42`
- Парсите JSON-ответ вручную
- Если структура ответа изменится - вы узнаете об этом только в runtime (ошибка `NullPointerException`)

В gRPC подходе:

1. Вы **один раз описываете интерфейс** в файле `.proto`:

```protobuf
syntax = "proto3";
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}
message GetUserRequest {
  int64 user_id = 1;
}
message GetUserResponse {
  string name = 1;
  string email = 2;
}
```   

2. Вы запускаете **генератор кода** (`protoc` + плагин для Java).

3. На выходе получаете:
    - Класс `UserServiceGrpc` с интерфейсом сервера и клиентским stub-ом
    - Классы `GetUserRequest`, `GetUserResponse` с геттерами/сеттерами
    - Всё типизировано, сериализация/десериализация - «из коробки»

Теперь в коде `OrderService` вы вызываете:

```java
GetUserResponse response = userServiceBlockingStub.getUser(
        GetUserRequest.newBuilder().setUserId(42).build()
);

String name = response.getName(); // ← компилятор знает, что getName() существует
```

Если в `.proto` изменится структура - **код не скомпилируется**, а не упадёт в production.

> **Генерация кода превращает контракт в исполняемый, типобезопасный код. Это основа надёжной интеграции.**

--- 

### Как это работает технически?

1. **`.proto`-файл** - это **единый источник истины** (single source of truth) для клиента и сервера.
2. Инструмент `protoc` (Protocol Buffer Compiler) читает этот файл.
3. Плагин (например, `protoc-gen-grpc-java`) генерирует:
    - **Серверный код**: абстрактный класс, который вы наследуете и реализуете логику
    - **Клиентский код**: stub-ы (blocking, async, reactive), которые прячут сетевую сложность
4. Обе стороны используют **один и тот же формат сериализации**, поэтому нет несовместимостей.

### Преимущества такого подхода

| Аспект              | REST + JSON                         | gRPC + protobuf                        |
|---------------------|-------------------------------------|----------------------------------------|
| Типизация           | Нет (всё Object/Map)                | Да (компилятор проверяет поля)         |
| Скорость            | Медленнее (текстовый JSON)          | Быстрее (бинарный protobuf)            |
| Размер трафика      | Больше                              | Компактнее (до 3–5× меньше)            |
| Streaming           | Нет (только через SSE/WebSocket)    | Встроен (server/client/bidi)           |
| Контракт            | Неформальный (OpenAPI - опционален) | Обязательный (`.proto` - часть сборки) |
| Безопасность ошибок | Runtime                             | Compile-time                           |

---

### Когда использовать gRPC?

✅ **Идеально подходит**:

- Для внутренних микросервисов (service-to-service communication)
- Когда важна производительность и низкая задержка
- Когда нужен streaming (например, live-уведомления, IoT-телеметрия)
- Когда вы контролируете обе стороны (клиент и сервер)

❌ **Не подходит**:

- Для public API (браузеры плохо работают с gRPC; лучше REST/gRPC-Web)
- Для простых CRUD-операций без сложной логики
- Если вы не готовы управлять `.proto`-файлами как частью CI/CD

---

### Пример: минимальный workflow в Java

1. Создаёте `src/main/proto/user_service.proto`
2. Добавляете в `pom.xml`:

```xml

<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.6.1</version>
    <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.25.0:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.60.0:exe:${os.detected.classifier}</pluginArtifact>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>compile-custom</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

3. Запускаете `mvn compile` → генерируются классы в `target/generated-sources/protobuf`

4. Реализуете сервер:

```java
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
        // ваша логика
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

5. Используете клиент:

```java
ManagedChannel channel = Grpc.newChannelBuilder("localhost:8080", InsecureChannelCredentials.create()).build();
UserServiceGrpc.UserServiceBlockingStub stub = UserServiceGrpc.newBlockingStub(channel);
GetUserResponse resp = stub.getUser(GetUserRequest.newBuilder().setUserId(42).build());
```

Этот процесс **полностью автоматизирован** и гарантирует, что клиент и сервер «говорят на одном языке».

---

# 2. Protocol Buffers (protobuf)

## Что такое Protocol Buffers?

**Protocol Buffers (protobuf)** - это:

- **Язык описания интерфейсов (IDL)** - вы описываете структуру данных и сервисы в `.proto`-файлах
- **Бинарный формат сериализации** - компактный, быстрый, кроссплатформенный
- **Система генерации кода** - из `.proto` создаются классы на Java, Go, Python и др.

В отличие от JSON/XML, protobuf:

- Не читаем человеком «в сыром виде» (но можно декодировать)
- Требует схемы (`.proto`) для интерпретации
- Гарантирует порядок полей через **нумерацию**

---

### `syntax`

Файл начинается с указания версии:

```protobuf
syntax = "proto3";
```

---

### `package` / `import`

Пакет предотвращает коллизии имён:

```protobuf
package com.example.user.v1;
```

Импорт других `.proto`-файлов:

```protobuf
import "google/protobuf/timestamp.proto"; // Well-Known Types
import "common/status.proto"; // ищется в common/status.proto относительно proto_path
```

> Стандартные пути (Well-Known Types). Эти файлы встроены в компилятор `protoc` и находятся в его инсталляции.
>
> Они всегда доступны без дополнительной настройки.

Структура проекта:

```text
project/
├── proto/
│   ├── google/
│   │   └── protobuf/          # well-known types (копия или симлинк)
│   │       ├── timestamp.proto
│   │       └── wrappers.proto
│   ├── common/
│   │   └── status.proto
│   └── user/
│       └── user.proto
```

- `Package` - это логическое пространство имен в `protobuf`
- `Import` - это физическое подключение других файлов
- `Package` влияет на generated code, но может быть переопределен опциями
- `Package` помогает версионировать и организовывать API
- Полное имя типа = `package + имя сообщения`

Таким образом, package - это способ сказать "где живет этот тип", а import - "откуда взять определения других типов".

> В Java имя пакета в `.proto` **не обязательно** совпадает с Java-пакетом - его можно переопределить через опцию
`java_package`.

---

### `message`

Описывают структуру данных:

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  bool active = 4;
}
```

Каждое поле имеет:

- **Тип** (`int32`, `int64`, `string`, `bool`, `bytes`, `enum`, другой `message`)
- **Имя**
- **Номер тега** - уникальный в пределах сообщения, используется при сериализации

Правила нумерации типов:

- Минимальный тег: 1 (для Enum - 0)
- Максимальный тег: 2^29 - 1 (536,870,911)
- Зарезервировано: 19000-19999 (для внутренних нужд protobuf)
- Теги 1-15 занимают 1 байт - используйте их для часто используемых полей, 16-2047 - 2 байта

---

### `enum`

```protobuf
enum Role {
  ROLE_UNSPECIFIED = 0; // Первое значение должно быть `= 0` (значение по умолчанию при десериализации).
  ROLE_ADMIN = 1;
  ROLE_USER = 2;
}
```

---

### `oneof` (взаимоисключающие поля)

```protobuf
message SearchRequest {
  oneof query_type {
    string text_query = 1;
    bytes binary_query = 2;
  }
}
```

В любой момент задано **только одно** поле из `oneof`.

---

### `repeated`

> Ключевое слово repeated указывает, что поле может содержать ноль или более значений (аналог List<T> в Java):

```protobuf
message UserList {
  repeated User users = 1;
}
```

Особенности:

- Порядок элементов сохраняется
- Доступ в Java: `getUsersList() → List<User>`
- Можно комбинировать с любыми типами: `repeated string tags = 2;`, `repeated int32 scores = 3;`
- Не имеет ограничений на размер - но для больших объёмов данных лучше использовать `streaming` вместо `repeated`, чтобы
  избежать переполнения памяти

---

### `map`

```protobuf
syntax = "proto3";

message Price {
  string currency = 1;
  double amount = 2;
  int64 last_updated = 3;  // timestamp
}

message StoreInfo {
  int64 store_id = 1;
  string store_name = 2;
  string address = 3;
  bool is_active = 4;
}

message MapDemonstration {

  // Map с примитивами - простейший случай
  map<string, string> string_to_string = 1;        // Map<String, String>
  map<string, int32> string_to_int = 2;             // Map<String, Integer>
  map<int64, string> long_to_string = 3;            // Map<Long, String>
  map<bool, string> bool_to_string = 4;             // Map<Boolean, String>
  map<string, bytes> string_to_bytes = 5;           // Map<String, byte[]>
  map<int32, double> int_to_double = 6;              // Map<Integer, Double>


  // Самое частое использование - ключ примитив, значение - другой message
  map<int64, Price> product_prices = 10;             // Map<Long, Price> - цена по ID товара
  map<string, Price> regional_prices = 11;           // Map<String, Price> - цена по региону
  map<string, StoreInfo> store_catalog = 12;         // Map<String, StoreInfo> - магазины по коду
  map<int64, StoreInfo> store_by_id = 13;             // Map<Long, StoreInfo>

  /*
  Вот ЭТИ варианты НЕ РАБОТАЮТ - ключом может быть только скалярный тип!
  
  map<Price, string> price_description = 20;         // ❌ ОШИБКА! Price - не скаляр
  map<StoreInfo, int64> store_visits = 21;           // ❌ ОШИБКА! StoreInfo - не скаляр
  map<Price, StoreInfo> price_to_store = 22;         // ❌ ОШИБКА! оба не скаляры
  map<Price, Price> price_mapping = 23;              // ❌ ОШИБКА! ключ не скаляр
  */


  // Вариант 4.1: Непосредственно вложенный map (синтаксически верно)
  map<string, map<string, int32>> nested_map_1 = 30;
  // Map<String, Map<String, Integer>>

  map<int64, map<string, double>> nested_map_2 = 31;
  // Map<Long, Map<String, Double>>

  map<string, map<int64, Price>> nested_map_3 = 32;
  // Map<String, Map<Long, Price>>


  map<string, string> metadata = 40;
  // В Java: map.put("key", null);  ❌ Будет исключение NullPointerException!

  map<int64, Price> prices = 41;
  // В Java: map.put(1L, null);     ❌ Тоже исключение!

}

// Правильный подход - использовать wrapper-типы или отдельный признак
message NullablePrice {
  Price price = 1;
  bool is_null = 2;  // если true, значит цены нет
}

// map<int64, NullablePrice> safe_prices = 42;  // безопасный вариант
```

- `map` - это поле в `message`
- В значении можно использовать другие `message` - очень частая практика
- В ключе можно только скалярные типы - важное ограничение
- Порядок не гарантируется - как в обычном `HashMap`
- `null` не поддерживается - учитывай при маппинге

---

### Вложенные `message`

```protobuf
message User {
  // Вложенное сообщение Settings
  message Settings {
    bool notifications_enabled = 1;
    string language = 2;
    string theme = 3;
  }

  string user_id = 1;
  string name = 2;
  Settings settings = 3;  // используем вложенное сообщение как поле
}

// Доступ в Java: `User.Settings`

message Order {
  // Здесь Settings НЕДОСТУПНО!
  // Order.Settings - ошибка! Settings живёт только внутри User
}
```

В Java это превращается в:

```java
// Доступ к вложенному классу
User.Settings settings = User.Settings.newBuilder()
                .setNotificationsEnabled(true)
                .setLanguage("ru")
                .setTheme("dark")
                .build();

// Использование в User
User user = User.newBuilder()
        .setUserId("123")
        .setName("Иван")
        .setSettings(settings)
        .build();

// Чтение вложенных полей
boolean notifications = user.getSettings().getNotificationsEnabled();
String lang = user.getSettings().getLanguage();
```

Глубокая вложенность:

```protobuf
message Company {
  message Department {
    message Team {
      message Employee {
        message Contact {
          string email = 1;
          string phone = 2;
        }

        string name = 1;
        string position = 2;
        Contact contact = 3;
      }

      string team_name = 1;
      repeated Employee employees = 2;
    }

    string dept_name = 1;
    map<string, Team> teams = 2;  // team_name -> Team
  }

  string company_name = 1;
  repeated Department departments = 2;
}
```

Вложенные enum:

```protobuf
message User {
  enum UserType {
    USER_TYPE_UNKNOWN = 0;
    USER_TYPE_REGULAR = 1;
    USER_TYPE_ADMIN = 2;
  }

  message Settings {
    enum Theme {
      THEME_LIGHT = 0;
      THEME_DARK = 1;
      THEME_AUTO = 2;
    }

    Theme theme = 1;
    bool notifications = 2;
  }

  string name = 1;
  UserType type = 2;
  Settings settings = 3;
}

// Доступ в Java
    User.UserType type = User.UserType.USER_TYPE_ADMIN;
    User.Settings.Theme theme = User.Settings.Theme.THEME_DARK;
```

Когда использовать вложенные сообщения?

✅ Используй вложенные, когда:

- Сообщение используется только внутри родительского
- Хочешь показать иерархию и группировку
- Нужно избежать конфликтов имён
- Создаёшь внутренние DTO для конкретного API

❌ Не используй вложенные, когда:

- Сообщение нужно переиспользовать в других местах
- Глубина вложенности > 3 (становится нечитаемо)
- Создаёшь общие типы для всего проекта

---

## Определение сервисов

Сервис описывает RPC-методы:

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc StreamOrders (StreamOrdersRequest) returns (stream Order);
  rpc UploadLogs (stream LogEntry) returns (UploadResult);
  rpc Chat (stream Message) returns (stream Message);
}
```

Ключевые слова:

- `stream` перед типом → потоковая передача
- Четыре комбинации:
    - Без `stream` → **unary**
    - `returns (stream ...)` → **server streaming**
    - `(stream ...) returns` → **client streaming**
    - `(stream ...) returns (stream ...)` → **bidirectional streaming**

---

### Версионирование и совместимость

Protobuf поддерживает **forward и backward compatibility**, если соблюдать правила:

✅ **Можно**:

- Добавлять новые поля с **новыми номерами тегов**
- Переименовывать поля (номер тега важнее имени)
- Делать поля необязательными (все поля в proto3 неявно optional)

❌ **Нельзя**:

- Менять номер тега у существующего поля
- Удалять поле и повторно использовать его номер для другого типа
- Менять тип поля на несовместимый (например, `string` → `int32`)

> Если поле удаляется - оставьте номер «зарезервированным»:
> reserved 5;
> reserved "old_field_name";

Примеры резервирования:

Пример 1: Удалили поле и резервируем его номер

```protobuf
// Было (версия 1)
message User {
  int64 id = 1;
  string name = 2;
  string phone = 3;  // потом решили убрать телефон
  string email = 4;
}

// Стало (версия 2) - правильно с резервированием
message User {
  int64 id = 1;
  string name = 2;
  string email = 4;  // email остался

  reserved 3;  // номер 3 теперь зарезервирован для phone
}

// Если кто-то попытается добавить поле с номером 3:
message User {
  int64 id = 1;
  string name = 2;
  string new_field = 3;  // ❌ ОШИБКА! Номер 3 зарезервирован
  string email = 4;

  reserved 3;
}
// Ошибка компиляции: field new_field uses reserved number 3
```

Пример 2: Резервирование имени поля

```protobuf
// Было
message User {
  int64 id = 1;
  string name = 2;
  string phone_number = 3;  // удаляем это поле
}

// Стало
message User {
  int64 id = 1;
  string name = 2;

  reserved "phone_number";  // имя поля зарезервировано
}

// Если кто-то попытается использовать имя phone_number снова:
message User {
  int64 id = 1;
  string name = 2;
  string phone_number = 4;  // ❌ ОШИБКА! Имя зарезервировано

  reserved "phone_number";
}
// Ошибка: field phone_number uses reserved name "phone_number"
```

Пример 3: Резервирование и номеров, и имён

```protobuf
message User {
  int64 id = 1;
  string name = 2;

  // Резервируем номера и имена удалённых полей
  reserved 3, 5, 7 to 10;        // диапазон тоже можно
  reserved "phone", "fax", "old_field";

  // Теперь это новое поле с номером 4 - ок
  string email = 4;

  // А это уже нельзя:
  // int32 phone = 3;           // ❌ номер 3 зарезервирован
  // string fax = 8;            // ❌ номер 8 зарезервирован
  // string old_field = 11;      // ❌ имя зарезервировано
}
```

---

### Best practices для .proto-файлов

- Используйте **ясные имена**: `user_id`, а не `uid`
- Все RPC-методы - **глагол + существительное**: `CreateOrder`, `ListUsers`
- Каждый метод принимает **один запросный объект** и возвращает **один ответный** (даже если сейчас нужно одно поле -
  заведите сообщение)
- Не используйте `repeated` для больших коллекций - лучше `streaming`
- Добавляйте комментарии - они попадут в сгенерированный Javadoc

---

### Интеграция с Java: как выглядит сгенерированный код?

Из сообщения:

```protobuf
message User {
  int64 id = 1;
  string name = 2;
}
```

Генерируется immutable-класс с:

- `getId()`, `getName()`
- `newBuilder()` → `setId(42).setName("Alice").build()`
- `toByteArray()` / `parseFrom(byte[])` - сериализация

```java
// Файл: UserOuterClass.java (имя зависит от названия .proto файла)

public final class UserOuterClass {

    // Вложенный финальный класс - сам не может быть унаследован
    public static final class User extends com.google.protobuf.GeneratedMessageV3  // Базовый класс от Protobuf
            implements UserOrBuilder {  // Интерфейс с геттерами

        // Поля - private final, неизменяемые после создания
        private final long id_;
        private final volatile Object name_;      // volatile для потокобезопасности

        // Геттеры (из интерфейса UserOrBuilder)
        @Override
        public long getId() {
            return id_;
        }

        @Override
        public String getName() {
            Object ref = name_;
            if (ref instanceof String) {
                return (String) ref;
            } else {
                // Если это ByteString, конвертируем в String
                ByteString bs = (ByteString) ref;
                String s = bs.toStringUtf8();
                name_ = s;  // Кешируем результат
                return s;
            }
        }

        // Сериализация в байты
        @Override
        public byte[] toByteArray() {
            try {
                final byte[] result = new byte[getSerializedSize()];
                // ... логика сериализации
                return result;
            } catch (IOException e) {
                throw new RuntimeException("Serialization error", e);
            }
        }

        // Десериализация из байтов (статический метод)
        public static User parseFrom(byte[] data) throws InvalidProtocolBufferException {
            return newBuilder().mergeFrom(data).buildPartial();
        }

        // equals сравнивает все поля
        @Override
        public boolean equals(final Object obj) {
            if (obj == this) return true;
            if (!(obj instanceof User)) return false;

            User other = (User) obj;

            boolean result = true;
            result = result && (getId() == other.getId());
            result = result && getName().equals(other.getName());
            return result;
        }

        // hashCode учитывает все поля
        @Override
        public int hashCode() {
            int hash = 41;  // Начальное значение
            hash = (19 * hash) + Long.hashCode(getId());
            hash = (29 * hash) + getName().hashCode();
            return hash;
        }

        // toString для отладки
        @Override
        public String toString() {
            return "User{" +
                    "id=" + id_ +
                    ", name=" + name_ +
                    "}";
        }

        // Builder - отдельный вложенный класс для создания объектов
        public static final class Builder extends
                com.google.protobuf.GeneratedMessageV3.Builder<Builder> {

            private long id_;
            private Object name_ = "";

            // Сеттер возвращает Builder для цепочек
            public Builder setId(long id) {
                id_ = id;
                return this;
            }

            public Builder setName(String name) {
                name_ = name;
                return this;
            }

            // Финальное создание объекта
            public User build() {
                User result = new User(this);
                return result;
            }
        }
    }
}
```

Из сервиса:

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}
```

Генерируются:

- `UserServiceGrpc` - содержит:
    - `UserServiceStub` (async)
    - `UserServiceBlockingStub` (sync)
    - `UserServiceFutureStub` (future-based)
- `UserServiceGrpc.UserServiceImplBase` - абстрактный класс для реализации сервера

```java
// Файл: UserServiceGrpc.java

@javax.annotation.Generated(
        value = "by gRPC proto compiler",
        comments = "Source: user_service.proto")
public final class UserServiceGrpc {

    // Класс с описанием метода (для внутреннего использования)
    private static final MethodDescriptor<GetUserRequest, GetUserResponse>
            METHOD_GET_USER =
            MethodDescriptor.newBuilder()
                    .setType(MethodType.UNARY)  // Один запрос - один ответ
                    .setFullMethodName(
                            MethodDescriptor.generateFullMethodName(
                                    "UserService", "GetUser"))
                    .build();

    // ----- Базовый класс для реализации сервера -----
    public abstract static class UserServiceImplBase implements BindableService {

        // Метод, который нужно переопределить в реализации
        public void getUser(GetUserRequest request,
                            StreamObserver<GetUserResponse> responseObserver) {
            // По умолчанию - не реализовано
            responseObserver.onError(
                    new io.grpc.StatusRuntimeException(Status.UNIMPLEMENTED));
        }

        // Регистрация сервиса в gRPC сервере
        @Override
        public final ServerServiceDefinition bindService() {
            return ServerServiceDefinition.builder(getServiceDescriptor())
                    .addMethod(
                            METHOD_GET_USER,
                            // Адаптер, преобразующий вызовы в наш метод
                            asyncUnaryCall(
                                    new MethodHandlers<
                                            GetUserRequest, GetUserResponse>(
                                            this, METHODID_GET_USER)))
                    .build();
        }
    }

    // ----- Блокирующий (синхронный) клиентский Stub -----
    public static final class UserServiceBlockingStub
            extends AbstractBlockingStub<UserServiceBlockingStub> {

        // Синхронный вызов - ждет ответ
        public GetUserResponse getUser(GetUserRequest request) {
            return blockingUnaryCall(
                    getChannel(), METHOD_GET_USER, getCallOptions(), request);
        }
    }

    // ----- Асинхронный клиентский Stub -----
    public static final class UserServiceStub
            extends AbstractAsyncStub<UserServiceStub> {

        // Асинхронный вызов с callback
        public void getUser(GetUserRequest request,
                            StreamObserver<GetUserResponse> responseObserver) {
            asyncUnaryCall(
                    getChannel().newCall(METHOD_GET_USER, getCallOptions()),
                    request, responseObserver);
        }
    }

    // ----- Future-клиент (для CompletableFuture) -----
    public static final class UserServiceFutureStub
            extends AbstractFutureStub<UserServiceFutureStub> {

        // Возвращает ListenableFuture
        public com.google.common.util.concurrent.ListenableFuture<GetUserResponse>
        getUser(GetUserRequest request) {
            return futureUnaryCall(
                    getChannel().newCall(METHOD_GET_USER, getCallOptions()), request);
        }
    }

    // Фабричные методы для создания Stub'ов
    public static UserServiceBlockingStub newBlockingStub(Channel channel) {
        return new UserServiceBlockingStub(channel);
    }

    public static UserServiceStub newStub(Channel channel) {
        return new UserServiceStub(channel);
    }

    public static UserServiceFutureStub newFutureStub(Channel channel) {
        return new UserServiceFutureStub(channel);
    }
}
```

> Вся сетевая логика, маршалинг, управление потоками - уже реализованы.

---

### Заключение

`.proto`-файл - это **контракт**, который:

- Читаем человеком
- Обрабатывается машиной
- Гарантирует совместимость
- Служит основой для генерации надёжного, типизированного кода

Правильно написанный `.proto` - залог стабильной и масштабируемой gRPC-системы.

---

# 3. Генерация кода и интеграция с Java

## Компиляция .proto файлов: протокол и инструменты

**`protoc`** (Protocol Buffers Compiler) - основной инструмент для генерации кода из `.proto`-файлов. Он сам по себе не
генерирует Java-код для gRPC - для этого нужны **плагины**:

- `protoc-gen-java` - генерирует классы сообщений (входит в `protobuf-java`)
- `protoc-gen-grpc-java` - генерирует сервисы и stub'ы (входит в `grpc-java`)

Процесс компиляции:

```text
proto-файл → protoc + плагин → сгенерированный Java-код
```

---

### Настройка Maven: protobuf-maven-plugin

Добавляем зависимости:

```xml

<dependencies>

    <!-- Protobuf runtime -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>${protobuf.version}</version>
    </dependency>

    <!-- gRPC core -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <!-- Для аннотаций @Generated (Java 9+) -->
    <dependency>
        <groupId>jakarta.annotation</groupId>
        <artifactId>jakarta.annotation-api</artifactId>
        <version>2.1.1</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

Конфигурация плагина:

```xml

<build>

    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>

            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>

            <configuration>
                <!-- Путь к .proto файлам -->
                <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>

                <!-- Куда генерировать код -->
                <outputDirectory>${project.build.directory}/generated-sources/protobuf/java</outputDirectory>
                <grpcOutputDirectory>${project.build.directory}/generated-sources/protobuf/grpc-java
                </grpcOutputDirectory>

                <!-- Включить генерацию gRPC кода -->
                <generateProtoCompilers>true</generateProtoCompilers>
                <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>

            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

    </plugins>
</build>
```

> `${os.detected.classifier}` определяет ОС (linux-x86_64, osx-aarch_64 и т.д.) через `os-maven-plugin`.

---

### Настройка Gradle: protobuf-gradle-plugin

В `build.gradle`:

```groovy

plugins {
    id 'java'
    id 'com.google.protobuf' version '0.9.4'
}

dependencies {
    implementation 'io.grpc:grpc-netty-shaded:1.60.0'
    implementation 'io.grpc:grpc-protobuf:1.60.0'
    implementation 'io.grpc:grpc-stub:1.60.0'
    compileOnly 'jakarta.annotation:jakarta.annotation-api:2.1.1'
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.25.0"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:1.60.0"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}

sourceSets {
    main {
        java {
            srcDirs 'build/generated/source/proto/main/grpc'
            srcDirs 'build/generated/source/proto/main/java'
        }
    }
}
```

---

## Структура сгенерированного кода

Для `.proto` файла:

```protobuf
syntax = "proto3";
package com.example.user.v1;

message GetUserRequest {
  int64 user_id = 1;
}

message GetUserResponse {
  string name = 1;
  string email = 2;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

Генерируются:

- `GetUserRequest.java`, `GetUserResponse.java` — immutable-классы сообщений
- `UserServiceGrpc.java` — контракт сервиса, stub-ы и базовый класс реализации

---

## Сообщения

Сгенерированные классы:

- Иммутабельны после создания
- Имеют `newBuilder()` для конструирования
- Реализуют `equals()`, `hashCode()`, `toString()`
- Поддерживают сериализацию через `toByteArray()` / `parseFrom()`

Пример использования:

```java
UserOuterClass.User user = UserOuterClass.User.newBuilder()
        .setId(123L)
        .setName("Alice")
        .build();

byte[] serialized = user.toByteArray();
UserOuterClass.User deserialized = UserOuterClass.User.parseFrom(serialized);
```

---

## Сервисы и stub'ы

Из `service UserService { ... }` генерируется класс `UserServiceGrpc` со следующими вложенными элементами:

### 1. `UserServiceImplBase`

Абстрактный класс для реализации сервера:

```java
public abstract static class UserServiceImplBase implements BindableService {

    public void getUser(GetUserRequest request,
                        StreamObserver<GetUserResponse> responseObserver) {
        // По умолчанию — UNIMPLEMENTED
    }
}
```

Реализация:

```java
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<GetUserResponse> responseObserver) {
        try {
            GetUserResponse response = GetUserResponse.newBuilder()
                    .setName("John Doe")
                    .setEmail("john@example.com")
                    .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException());
        }
    }
}
```

#### 2. Stub-ы для клиента

- `UserServiceBlockingStub` — синхронный, блокирующий вызов
- `UserServiceStub` — асинхронный, callback-based
- `UserServiceFutureStub` — возвращает `ListenableFuture` (из Guava)

Примеры:

```java
public class GrpcClientExamples {


    // 1. СОЗДАНИЕ КАНАЛА - соединение с сервером
    ManagedChannel channel = ManagedChannelBuilder
            .forAddress(host, port)
            .usePlaintext()  // для разработки (без TLS)
            .build();

    // 2. СОЗДАНИЕ STUB'ОВ - клиентских прокси для вызова методов
    UserServiceGrpc.UserServiceBlockingStub blockingStub = UserServiceGrpc.newBlockingStub(channel);    // синхронный
    UserServiceGrpc.UserServiceStub asyncStub = UserServiceGrpc.newStub(channel);   // асинхронный
    UserServiceGrpc.UserServiceFutureStub futureStub = UserServiceGrpc.newFutureStub(channel);  // future-based


    /**
     * ПРИМЕР 1: Блокирующий (синхронный) вызов
     * Самый простой способ - поток блокируется до получения ответа
     */
    public void blockingCallExample() {
        // Создаем запрос с помощью Builder
        GetUserRequest request = GetUserRequest.newBuilder()
                .setUserId(123)
                .build();

        try {
            // Вызов блокирует поток до получения ответа или ошибки
            GetUserResponse response = blockingStub.getUser(request);

            // Получаем данные из ответа
            User user = response.getUser(); // ...

        } catch (StatusRuntimeException e) {
            // Специфическая gRPC ошибка с кодом статуса
            System.err.rintln("gRPC ошибка: " + e.getStatus());
            System.err.println("Описание: " + e.getStatus().getDescription());
        } catch (Exception e) {
            // Другие ошибки
            System.err.println("Ошибка: " + e.getMessage());
        }
    }

    /**
     * ПРИМЕР 2: Асинхронный вызов с callback-ами
     * Не блокирует поток, ответ приходит в колбэки
     */
    public void asyncCallExample() {
        GetUserRequest request = GetUserRequest.newBuilder()
                .setUserId(456)
                .build();

        // Создаем обработчик ответа (StreamObserver)
        StreamObserver<GetUserResponse> responseObserver = new StreamObserver<GetUserResponse>() {

            @Override
            public void onNext(GetUserResponse response) {
                // Вызывается при получении каждого ответа
                // Для обычного (unary) вызова - один раз
                User user = response.getUser(); // ...
            }

            @Override
            public void onError(Throwable t) {
                if (t instanceof StatusRuntimeException statusEx) {
                    System.err.println("Статус: " + statusEx.getStatus());
                    System.err.println("Причина: " + statusEx.getStatus().getDescription());
                } else {
                    t.printStackTrace();
                }
            }

            @Override
            public void onCompleted() {
                // Вызывается после успешного завершения
                // Для стримов - после всех onNext, для unary - после одного onNext
                System.out.println("onCompleted: вызов завершен успешно");
            }
        };

        // Асинхронный вызов (не блокируется)
        System.out.println("Отправляем асинхронный запрос...");
        asyncStub.getUser(request, responseObserver);

        // Мы можем продолжать работу здесь, пока ждем ответ
        System.out.println("Ждем ответ в callback'ах...");
    }

    /**
     * ПРИМЕР 3: Future stub (ListenableFuture)
     * Компромисс между синхронным и асинхронным подходом
     */
    public void futureStubExample() throws Exception {
        GetUserRequest request = GetUserRequest.newBuilder()
                .setUserId(789)
                .build();

        // Получаем ListenableFuture (из Google Guava)
        ListenableFuture<GetUserResponse> future1 = futureStub.getUser(request);

        // СПОСОБ 1: Блокируемся и ждем (как blocking stub)
        try {
            GetUserResponse response = future1.get(5, TimeUnit.SECONDS); // таймаут 5 сек
            System.out.println("Получен: " + response.getUser().getName());
        } catch (Exception e) {
            System.err.println("Ошибка: " + e.getMessage());
        }

        // СПОСОБ 2: Добавляем callback (неблокирующий)
        // Создаем новый future для второго примера
        ListenableFuture<GetUserResponse> future2 = futureStub.getUser(GetUserRequest.newBuilder().setUserId(101112).build());

        // Добавляем колбэк, который выполнится при завершении
        Futures.addCallback(future2, new FutureCallback<GetUserResponse>() {

            @Override
            public void onSuccess(GetUserResponse response) {
                User user = response.getUser(); // ...
            }

            @Override
            public void onFailure(Throwable t) {
                t.printStackTrace();
            }
        }, MoreExecutors.directExecutor()); // выполнить в текущем потоке

        // СПОСОБ 3: Комбинируем с CompletableFuture (Java 8+)
        ListenableFuture<GetUserResponse> future3 = futureStub.getUser(request);

        CompletableFuture<User> completableFuture = new CompletableFuture<>();

        future.addListener(() -> {
            try {
                completableFuture.complete(future3.get().getUser());
            } catch (Exception e) {
                completableFuture.completeExceptionally(e);
            }
        }, MoreExecutors.directExecutor());

        // Теперь можно использовать completableFuture как обычный CompletableFuture
        completableFuture
                .thenApply(User::getName)
                .thenAccept(name -> System.out.println("Имя: " + name))
                .exceptionally(throwable -> {
                    System.err.println("Ошибка: " + throwable.getMessage());
                    return null;
                });
    }

    /**
     * ПРИМЕР 4: Обработка ошибок и deadlines
     */
    public void errorHandlingExample() {
        // Устанавливаем дедлайн для вызова
        UserServiceGrpc.UserServiceBlockingStub stubWithDeadline = blockingStub
                .withDeadline(Deadline.after(2, TimeUnit.SECONDS));

        GetUserRequest request = GetUserRequest.newBuilder()
                .setUserId(999)  // несуществующий ID
                .build();

        try {
            GetUserResponse response = stubWithDeadline.getUser(request);
            System.out.println("Успех: " + response);

        } catch (StatusRuntimeException e) {
            Status status = e.getStatus();

            // Анализируем код ошибки
            switch (status.getCode()) {
                case DEADLINE_EXCEEDED:
                    System.err.println("⏰ Таймаут: сервер не ответил вовремя");
                    break;

                case NOT_FOUND:
                    System.err.println("🔍 Пользователь не найден");
                    break;

                case UNAVAILABLE:
                    System.err.println("🔌 Сервер недоступен");
                    break;

                case PERMISSION_DENIED:
                    System.err.println("🔐 Нет доступа");
                    break;

                default:
                    System.err.println("❌ Другая ошибка: " + status.getCode());
            }

            // Можно получить дополнительные детали из метаданных
            Metadata trailers = e.getTrailers();
            if (trailers != null) {
                // Извлекаем кастомные метаданные
                System.err.println("Trailers: " + trailers.keys());
            }
        }
    }

    /**
     * ПРИМЕР 5: Cleanup - закрытие канала
     */
    public void shutdown() throws InterruptedException {
        // Инициируем graceful shutdown
        channel.shutdown();

        // Ждем завершения (максимум 5 секунд)
        if (!channel.awaitTermination(5, TimeUnit.SECONDS)) {
            System.err.println("Принудительное закрытие канала");
            channel.shutdownNow();
        }
    }
}
```

> Выбор stub'а зависит от контекста:
> - **Blocking**: простые скрипты, CLI-инструменты
> - **Async**: реактивные приложения, высокая нагрузка
> - **Future**: интеграция с CompletableFuture (через `Futures.addCallback` или `MoreExecutors`)

---

## Интеграция с Spring Boot

### Вариант 1: `grpc-spring-boot-starter` (рекомендуется)

Добавляем зависимость:

```xml

<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

Реализуем сервис как `@GrpcService`:

```java

@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
        // реализация
    }
}
```

application.yml:

```yml
grpc:
  server:
    port: 9090
    enable-reflection: true # для grpcurl / BloomRPC
```

> Автоматически регистрирует все `@GrpcService` бины в gRPC-сервере.

### Вариант 2: Ручная интеграция

Если нельзя использовать стартер:

```java

@Configuration
public class GrpcServerConfig {

    @Bean
    public Server grpcServer(UserServiceImpl userService) throws IOException {
        return ServerBuilder.forPort(9090)
                .addService(userService)
                .build()
                .start();
    }

    @PreDestroy
    public void stopGrpcServer() {
        // graceful shutdown
    }
}
```

> Требует ручного управления жизненным циклом сервера.

---

## Клиентская часть в Spring Boot

### Создание канала

```java

@Bean
public ManagedChannel grpcChannel() {
    return ManagedChannelBuilder.forAddress("localhost", 9090)
            .usePlaintext() // или .useTransportSecurity() для TLS
            .build();
}

@Bean
public UserServiceGrpc.UserServiceBlockingStub userServiceBlockingStub(ManagedChannel channel) {
    return UserServiceGrpc.newBlockingStub(channel);
}
```

Использование в сервисе:

```java

@Service
public class UserClientService {

    private final UserServiceGrpc.UserServiceBlockingStub stub;

    public UserClientService(UserServiceGrpc.UserServiceBlockingStub stub) {
        this.stub = stub;
    }

    public GetUserResponse fetchUser(long userId) {
        return stub.getUser(GetUserRequest.newBuilder().setUserId(userId).build());
    }
}
```

> Для production рекомендуется настраивать keepAlive, retry, таймауты и connection pooling.

---

## Заключение

- Генерация кода полностью автоматизирована через `protoc` + плагины
- Maven/Gradle интеграция позволяет включить генерацию в стандартный build lifecycle
- Сгенерированный код типобезопасен, иммутабелен и готов к использованию
- Spring Boot значительно упрощает запуск сервера и управление клиентами
- Выбор stub'а (blocking/async/future) должен основываться на архитектуре приложения

---

# 4. Реализация gRPC-сервисов на Java

## Базовая реализация сервиса

Серверная часть gRPC в Java строится на **наследовании** от сгенерированного абстрактного класса:

```java
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
        // Логика обработки запроса
    }
}
```

Ключевые особенности:

- Методы **не возвращают значение напрямую**, а используют `StreamObserver<T>`
- Ответ отправляется через `responseObserver.onNext(response)`
- Успешное завершение — `responseObserver.onCompleted()`
- Ошибка — `responseObserver.onError(Throwable)`

> Важно: **всегда вызывайте либо `onCompleted()`, либо `onError()`**, иначе клиент зависнет.

---

### Unary-вызов (один запрос → один ответ)

```java

@Override
public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
    try {
        if (request.getUserId() <= 0) {
            responseObserver.onError(
                    Status.INVALID_ARGUMENT.withDescription("user_id must be positive").asRuntimeException());
            return;
        }

        User user = userRepository.findById(request.getUserId());
        if (user == null) {
            responseObserver.onError(
                    Status.NOT_FOUND.withDescription("User not found").asRuntimeException());
            return;
        }

        GetUserResponse response = GetUserResponse.newBuilder()
                .setName(user.getName())
                .setEmail(user.getEmail())
                .build();

        responseObserver.onNext(response); // отправляет ответ
        responseObserver.onCompleted(); // успешное завершение

    } catch (Exception e) {
        responseObserver.onError(
                Status.INTERNAL.withDescription("Internal error").withCause(e).asRuntimeException());
    }
}
```

> Обратите внимание: даже при ошибке **не выбрасывается исключение**, а вызывается `onError()`.

---

### Server streaming (один запрос → поток ответов)

```java

@Override
public void listOrders(ListOrdersRequest request, StreamObserver<Order> responseObserver) {
    try {
        List<OrderEntity> orders = orderRepository.findByUserId(request.getUserId());
        for (OrderEntity entity : orders) {
            Order order = Order.newBuilder()
                    .setId(entity.getId())
                    .setAmount(entity.getAmount())
                    .setStatus(entity.getStatus())
                    .build();
            responseObserver.onNext(order);
        }
        responseObserver.onCompleted();
    } catch (Exception e) {
        responseObserver.onError(Status.INTERNAL.withCause(e).asRuntimeException());
    }
}
```

> Поток может содержать **ноль, один или много** сообщений. Клиент получает их по мере отправки.

---

### Client streaming (поток запросов → один ответ)

```java
private static class OrderAggregator implements StreamObserver<UploadOrderRequest> {

    private final StreamObserver<UploadOrderResponse> responseObserver;
    private final List<OrderEntity> buffer = new ArrayList<>();
    private boolean completed = false;

    public OrderAggregator(StreamObserver<UploadOrderResponse> responseObserver) {
        this.responseObserver = responseObserver;
    }

    @Override
    public void onNext(UploadOrder_protobuf.UploadOrderRequest request) {
        if (completed) return;
        // Валидация и буферизация
        buffer.add(toEntity(request));
    }

    @Override
    public void onError(Throwable t) {
        // Клиент оборвал соединение или прислал невалидные данные
        log.warn("Client stream failed", t);
        // Нет необходимости вызывать responseObserver.onError() — соединение уже разорвано
    }

    @Override
    public void onCompleted() {
        if (completed) return;
        completed = true;
        try {
            int saved = orderService.saveAll(buffer);
            UploadOrderResponse response = UploadOrderResponse.newBuilder()
                    .setSavedCount(saved)
                    .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL.withCause(e).asRuntimeException());
        }
    }
}

@Override
public StreamObserver<UploadOrderRequest> uploadOrders(StreamObserver<UploadOrderResponse> responseObserver) {
    return new OrderAggregator(responseObserver);
}
```

> Для client streaming метод возвращает `StreamObserver<Request>`, а не `void`.

---

### Bidirectional streaming (поток ↔ поток)

```java

@Override
public StreamObserver<Message> chat(StreamObserver<Message> responseObserver) {
    return new StreamObserver<Message>() {

        @Override
        public void onNext(Message message) {
            // Эхо-ответ
            responseObserver.onNext(message);
        }

        @Override
        public void onError(Throwable t) {
            log.warn("Chat stream error", t);
            // Соединение уже разорвано — ничего не отправляем
        }

        @Override
        public void onCompleted() {
            // Клиент завершил отправку
            responseObserver.onCompleted();
        }

    };
}
```

> Часто используется для чатов, подписок, long-polling замены.

---

## Обработка ошибок: Status и StatusRuntimeException

gRPC использует **коды состояния**, определённые
в [gRFC A6](https://github.com/grpc/grpc/blob/master/doc/statuscodes.md):

| Код                 | Сценарий                              |
|---------------------|---------------------------------------|
| `OK`                | Успех (не используется в исключениях) |
| `INVALID_ARGUMENT`  | Невалидные входные данные             |
| `NOT_FOUND`         | Ресурс не найден                      |
| `ALREADY_EXISTS`    | Конфликт уникальности                 |
| `PERMISSION_DENIED` | Недостаточно прав                     |
| `UNAUTHENTICATED`   | Не пройдена аутентификация            |
| `INTERNAL`          | Внутренняя ошибка сервера             |
| `UNAVAILABLE`       | Сервис временно недоступен            |

Пример:

```java
throw Status.NOT_FOUND
        .withDescription("User with id="+userId +" not found")
        .

asRuntimeException();
```

> **Никогда не используйте `INTERNAL` для бизнес-ошибок** — это маскирует реальную причину.

Можно добавлять детали через `StatusException`:

```java
Metadata metadata = new Metadata();
metadata.

put(Metadata.Key.of("error_code", Metadata.ASCII_STRING_MARSHALLER), "USER_LOCKED");

StatusRuntimeException sre = Status.PERMISSION_DENIED
        .withDescription("Account is locked")
        .asRuntimeException(metadata);
```

---

## Метаданные (headers)

Метаданные — это **key-value пары**, передаваемые вне тела сообщения. Используются для:

- Аутентификации (`Authorization: Bearer ...`)
- Tracing (`x-request-id`, `traceparent`)
- Корреляции (`correlation-id`)
- Кастомной маршрутизации

### Чтение метаданных на сервере

```java

@Override
public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
    Metadata metadata = Metadata.current();
    String token = metadata.get(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER));
    if (token == null || !isValidToken(token)) {
        responseObserver.onError(Status.UNAUTHENTICATED.asRuntimeException());
        return;
    }
    // ...
}
```

> `Metadata.current()` работает только внутри контекста gRPC-вызова.

### Отправка метаданных клиентом

```java
Metadata metadata = new Metadata();
metadata.

put(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER), "Bearer abc123");

// Для blocking stub
GetUserResponse response = blockingStub
        .withInterceptors(MetadataUtils.newAttachHeadersInterceptor(metadata))
        .getUser(request);

// Для async stub
GetUserResponse response = asyncStub
        .withInterceptors(MetadataUtils.newAttachHeadersInterceptor(metadata))
        .getUser(request, observer);
```

---

## Interceptor'ы

Interceptor'ы — аналог middleware в REST. Регистрируются **глобально** или **на уровне stub'а**.

> Interceptor'ы — это перехватчики вызовов на стороне сервера или клиента, которые выполняются до и после обработки
> запроса бизнес-логикой, но внутри gRPC-стека.

### Пример: логирование

```java
public class LoggingInterceptor implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        long start = System.nanoTime();
        String method = call.getMethodDescriptor().getFullMethodName();

        ServerCall.Listener<ReqT> listener = next.startCall(call, headers);

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(listener) {
            @Override
            public void onComplete() {
                long durationMs = (System.nanoTime() - start) / 1_000_000;
                log.info("gRPC call {} completed in {} ms", method, durationMs);
                super.onComplete();
            }

            @Override
            public void onHalfClose() {
                log.info("gRPC call {} received all requests", method);
                super.onHalfClose();
            }
        };

    }
}
```

Регистрация в Spring Boot (с `grpc-spring-boot-starter`):

```java

@GrpcGlobalInterceptor
public class LoggingInterceptor implements ServerInterceptor { /* ... */
}
```

Или программно:

```java
Server server = ServerBuilder.forPort(9090)
        .addService(ServerInterceptors.intercept(new UserServiceImpl(), new LoggingInterceptor()))
        .build();
```

### Пример: авторизация

```java
public class AuthInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_KEY =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String token = headers.get(AUTH_KEY);
        if (token == null || !validateToken(token)) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new ServerCall.Listener<>() {
            };
        }
        return next.startCall(call, headers);

    }
}
```

> Interceptor'ы выполняются **до** попадания в вашу реализацию сервиса.

---

## Интеграция с контекстом (Context)

Для передачи данных между interceptor'ами и реализацией сервиса используйте `io.grpc.Context`:

```java
// В interceptor'е
Context.Key<String> USER_ID_KEY = Context.key("user-id");

String userId = extractFromToken(token);
Context.current().withValue(USER_ID_KEY, userId).attach();

// В реализации сервиса
String userId = USER_ID_KEY.get();
```

> Это безопасный способ передачи данных без ThreadLocal (учитывает асинхронность gRPC).

---

## Заключение

- Все типы вызовов поддерживаются через единый интерфейс `StreamObserver`
- Обработка ошибок — через `Status` и `StatusRuntimeException`
- Метаданные — механизм для передачи служебной информации вне тела сообщения
- Interceptor'ы позволяют вынести кросс-функциональные задачи (логирование, auth, метрики)
- Контекст (`Context`) — предпочтительный способ передачи данных между слоями

Правильно реализованный gRPC-сервис:

- Явно обрабатывает все возможные ошибки
- Не теряет соединения (всегда вызывает `onCompleted/onError`)
- Безопасен к асинхронным вызовам
- Легко тестируется и расширяем через interceptor'ы

---