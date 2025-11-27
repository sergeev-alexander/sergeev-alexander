# Jackson

<details>
    <summary>
        <b>ObjectMapper</b>
    </summary>

### `ObjectMapper` Основной инструмент для сериализации и десериализации

`ObjectMapper` — это **сердце Jackson**. Он отвечает за:

- преобразование Java-объектов в JSON (**сериализация**),
- преобразование JSON в Java-объекты (**десериализация**),
- настройку глобального поведения (формат дат, обработка null, стратегии именования и т.д.).

> 💡 В Spring Boot `ObjectMapper` создаётся автоматически и доступен через DI. В standalone-приложениях — создаётся вручную.

### Основные методы

| Метод                                            | Назначение                                                | Пример                                                                        |
|:-------------------------------------------------|:----------------------------------------------------------|:------------------------------------------------------------------------------|
| `writeValueAsString(Object value)`               | Сериализует объект в JSON-строку                          | `String json = mapper.writeValueAsString(user);`                              |
| `writeValue(OutputStream out, Object value)`     | Записывает JSON в поток (например, в файл или HTTP-ответ) | `mapper.writeValue(response.getOutputStream(), user);`                        |
| `readValue(String content, Class<T> valueType)`  | Десериализует JSON-строку в объект                        | `User user = mapper.readValue(json, User.class);`                             |
| `readValue(InputStream src, Class<T> valueType)` | Читает JSON из потока                                     | `User user = mapper.readValue(fileStream, User.class);`                       |
| `configure(Feature, boolean)`                    | Включает/отключает глобальные флаги                       | `mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);` |
| `setSerializationInclusion(...)`                 | Устанавливает стратегию включения полей по умолчанию      | `mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);`             |
| `setPropertyNamingStrategy(...)`                 | Задаёт глобальную стратегию именования                    | `mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);`      |
| `registerModule(...)`                            | Подключает расширения (например, для Java Time)           | `mapper.registerModule(new JavaTimeModule());`                                |

### Типичная настройка `ObjectMapper` (standalone)

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.MapperFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

public class ObjectMapperConfig {

    public static ObjectMapper createConfiguredMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // 1. Отключить падение при неизвестных полях (важно для совместимости)
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        // 2. Не падать, если JSON пустой или null
        mapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false);
        mapper.configure(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT, true);

        // 3. Форматировать даты как строки (а не timestamp)
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        // 4. Поддержка Java 8 Date/Time (LocalDateTime и др.)
        mapper.registerModule(new JavaTimeModule());

        // 5. Игнорировать null-поля по умолчанию
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

        // 6. Использовать snake_case глобально (если не переопределено в классе)
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

        // 7. Не требовать сеттеры для final-полей (для immutable-объектов)
        mapper.enable(MapperFeature.ALLOW_FINAL_FIELDS_AS_MUTATORS);

        return mapper;
    }
}
```

### Практические советы
- Не создавайте ObjectMapper на каждый запрос! 
  
  В Spring Boot ObjectMapper создаётся автоматически (если в classpath есть Jackson — а он есть по умолчанию при старте Spring Boot Web), и вы можете внедрять его через DI в любой компонент: контроллер, сервис, репозиторий и т.д.
  
  Он потокобезопасен (thread-safe) и дорогой по ресурсам. Используйте singleton:
  - В Spring: внедряйте через @Autowired.
  - В standalone: создайте один экземпляр и переиспользуйте.

- Отключайте FAIL_ON_UNKNOWN_PROPERTIES в продакшене

  Иначе обновление JSON-схемы на стороне клиента вызовет падение сервера.

- Всегда регистрируйте JavaTimeModule для Java 8+

  Без него LocalDateTime, ZonedDateTime и др. не будут работать корректно.

- Используйте ObjectReader и ObjectWriter для повторяющихся операций:

```java
ObjectWriter writer = mapper.writerWithDefaultPrettyPrinter()
.with(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS);

String json = writer.writeValueAsString(user);
```

- Для тестов — настраивайте отдельный ObjectMapper

  Можно включить SerializationFeature.INDENT_OUTPUT для читаемости:

```java
@Test
void testSerialization() {
    ObjectMapper testMapper = new ObjectMapper();
    testMapper.enable(SerializationFeature.INDENT_OUTPUT);
    testMapper.setSerializationInclusion(JsonInclude.Include.ALWAYS);
    
    String json = testMapper.writeValueAsString(user);
    // красивый, читаемый JSON для отладки
}
```

Если вы создадите собственный бин ObjectMapper, Spring Boot не будет использовать свой, а возьмёт ваш:

```java
@Configuration
public class JacksonConfig {

    @Bean
    @Primary // ← обязательно, если вы хотите заменить стандартный
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        // ... другие настройки
        return mapper;
    }
}
```
</details>

<br>

<details>
    <summary>
        <b>Базовые аннотации</b>
    </summary>

## `@JsonProperty`

### Сопоставление полей с JSON свойствами

```java
public class User {

    @JsonProperty("user_id")                                    // Изменение имени в JSON
    private Long id;

    @JsonProperty(value = "username", required = true)          // Обязательное поле
    private String username;

    @JsonProperty(value = "email", access = Access.READ_ONLY)   // Только для чтения
    private String email;

    @JsonProperty(value = "priority", defaultValue = "1")       // Значение по умолчанию
    private int priority;

    @JsonProperty(index = 1)                                    // Порядок в JSON
    private String firstName;
}
```

### Атрибуты:

- `value` - имя свойства в JSON
- `required` - обязательно ли поле (по умолчанию false)
- `defaultValue` - значение по умолчанию
- `access` - уровень доступа (READ_ONLY, WRITE_ONLY, READ_WRITE)
- `index` - порядок сериализации

## `@JsonIgnore`

### Игнорирование полей

```java
public class SecureData {

    private String publicInfo;

    @JsonIgnore                                         // Игнорировать при сериализации и десериализации
    private String secretKey;

    @JsonIgnoreProperties({"internalId", "debugInfo"})  // Игнорировать несколько полей
    public class ApiResponse {

        private String data;
        private String internalId;
        private String debugInfo;
    }

    @JsonIgnoreType                                     // Игнорировать весь тип
    public class InternalConfig {

        private String secret;
    }
}
```

## `@JsonView`

### Контроль видимости полей для разных сценариев

```java
// Определение View классов
public class Views {

    public static class Public {
    }

    public static class Internal extends Public {
    }

    public static class Admin extends Internal {
    }
}

public class Employee {

    @JsonView(Views.Public.class)
    private Long id;

    @JsonView(Views.Public.class)
    private String name;

    @JsonView(Views.Internal.class)
    private String email;

    @JsonView(Views.Admin.class)
    private BigDecimal salary;

    @JsonView({Views.Internal.class, Views.Admin.class})
    private String phone;
}

// Использование в контроллере
@RestController
public class EmployeeController {

    @GetMapping("/public/{id}")
    @JsonView(Views.Public.class)
    public Employee getPublicInfo() { /* ... */ }

    @GetMapping("/internal/{id}")
    @JsonView(Views.Internal.class)
    public Employee getInternalInfo() { /* ... */ }
}
```

## `@JsonInclude`

### Управление включением полей в JSON

```java
// Уровень класса: исключать все null-поля
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Product {

    private Long id;
    private String name;

    // NON_BLANK: исключает null, "", и строки из одних пробелов
    @JsonInclude(JsonInclude.Include.NON_BLANK)
    private String description;

    // NON_EMPTY: для коллекций - исключает null и пустые (размер = 0)
    @JsonInclude(JsonInclude.Include.NON_EMPTY)
    private List<String> tags;

    // NON_DEFAULT: исключает значения, равные Java-умолчанию (0 для int)
    // ⚠️ осторожно: если 0 - валидное значение, лучше использовать Integer + NON_NULL
    @JsonInclude(JsonInclude.Include.NON_DEFAULT)
    private int retries = 0;

    // CUSTOM: пример - исключать "чувствительные" значения по бизнес-правилу
    // Например: не показывать поле, если оно содержит слово "internal"
    @JsonInclude(
            value = JsonInclude.Include.CUSTOM,
            valueFilter = InternalDataFilter.class
    )
    private String metadata;
}

// Фильтр для CUSTOM: исключает значение, если оно содержит "internal"
class InternalDataFilter {
    @Override
    public boolean equals(Object obj) {
        // Jackson вызывает filter.equals(actualValue)
        // Если вернём true - поле будет ИСКЛЮЧЕНО из JSON
        if (obj == null) return false;
        if (!(obj instanceof String)) return false;
        return ((String) obj).toLowerCase().contains("internal");
    }
}
```

### Значения Include:

- `ALWAYS` - всегда включать (по умолчанию)
- `NON_NULL` - исключать null
- `NON_ABSENT` - исключать null и Optional.empty()
- `NON_EMPTY` - исключать null, пустые коллекции, пустые строки
- `NON_DEFAULT` - исключать значения по умолчанию
- `NON_BLANK` - исключать null, "", и строки из одних пробелов

## `@JsonFormat`

### Форматирование дат и чисел

```java
public class Event {

    @JsonFormat(shape = JsonFormat.Shape.STRING,
            pattern = "yyyy-MM-dd HH:mm:ss",
            timezone = "Europe/Moscow")
    private LocalDateTime eventTime;

    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd")
    private LocalDate eventDate;

    @JsonFormat(shape = JsonFormat.Shape.NUMBER)  // timestamp в миллисекундах
    private Date created;

    @JsonFormat(shape = JsonFormat.Shape.STRING)
    private BigDecimal price;
}
```

## `@JsonSerialize` / `@JsonDeserialize`

### Кастомные сериализаторы и десериализаторы

```java
// Пример: сериализация даты в человекочитаемом формате
public class Event {

    private String name;

    // Сериализуем LocalDateTime как "28 ноября 2025, 15:30"
    @JsonSerialize(using = HumanReadableDateTimeSerializer.class)
    @JsonDeserialize(using = HumanReadableDateTimeDeserializer.class)
    private LocalDateTime eventTime;

    // Простой пример с конвертером: BigDecimal → строка с "USD"
    @JsonSerialize(converter = UsdAmountWriter.class)
    @JsonDeserialize(converter = UsdAmountReader.class)
    private BigDecimal price;
}

// === Сериализатор: LocalDateTime → строка на русском ===
public class HumanReadableDateTimeSerializer extends JsonSerializer<LocalDateTime> {

    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        if (value == null) {
            gen.writeNull();
            return;
        }
        String formatted = value.format(DateTimeFormatter.ofPattern("d MMMM yyyy, HH:mm", new Locale("ru")));

        gen.writeString(formatted);
    }
}

// === Десериализатор: строка на русском → LocalDateTime ===
public class HumanReadableDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {

    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getText();
        if (text == null || text.isBlank()) {
            return null;
        }
        // Простой парсинг - в реальности может потребоваться гибкий парсер
        return LocalDateTime.parse(text,
                DateTimeFormatter.ofPattern("d MMMM yyyy, HH:mm", new Locale("ru")));
    }
}

// === Конвертер для записи: BigDecimal → "123.45 USD" ===
public class UsdAmountWriter extends StdConverter<BigDecimal, String> {

    @Override
    public String convert(BigDecimal value) {
        return value != null ? value.setScale(2, RoundingMode.HALF_UP) + " USD" : null;
    }
}

// === Конвертер для чтения: "123.45 USD" → BigDecimal ===
public class UsdAmountReader extends StdConverter<String, BigDecimal> {

    @Override
    public BigDecimal convert(String value) {
        if (value == null || value.isBlank()) return null;
        // Убираем " USD" и парсим число
        String numberPart = value.replace(" USD", "").trim();
        return new BigDecimal(numberPart);
    }
}
```

## `@JsonCreator`

### Указание конструктора/фабричного метода для десериализации

```java
/**
 * Класс Person - неизменяемый (immutable): все поля final.
 * Jackson не может использовать обычный конструктор без аргументов,
 * потому что его нет, а сеттеров - тоже нет.
 *
 * @JsonCreator говорит Jackson: «Вот метод, через который можно создать объект при десериализации».
 */
public class Person {

    private final Long id;
    private final String name;

    /**
     * Основной конструктор - помечен @JsonCreator.
     * Jackson будет вызывать его при чтении JSON вида:
     *
     * {
     *   "id": 123,
     *   "name": "Alex"
     * }
     *
     * @JsonProperty связывает JSON-поля с параметрами конструктора.
     */
    @JsonCreator
    public Person(@JsonProperty("id") Long id,
                  @JsonProperty("name") String name) {
        this.id = id;
        this.name = name;
    }

    // Геттеры (необязательны для десериализации, но нужны для сериализации)
    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    /**
     * Альтернатива: фабричный метод с @JsonCreator.
     * Jackson также может использовать статические методы!
     * Полезно, если:
     * - нужно валидировать данные перед созданием
     * - имя поля в JSON отличается от внутреннего (например, "person_id")
     * - вы хотите централизовать логику создания
     *
     * Пример JSON для этого метода:
     * {
     *   "person_id": 123,
     *   "person_name": "Alex"
     * }
     *
     * ⚠️ Важно: Jackson может использовать **только один** @JsonCreator!
     * Если оставить оба - будет ошибка. Нужно выбрать один или использовать
     * @JsonCreator(mode = ...) + @JsonProperty для разрешения неоднозначности.
     */

    @JsonCreator
    public static Person create(@JsonProperty("person_id") Long id,
                                @JsonProperty("person_name") String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Имя обязательно");
        }
        return new Person(id, name.trim());
    }
}
```

## `@JsonSetter`

### Настройка сеттеров для десериализации

```java
public class Product {

    private String name;
    private BigDecimal price;

    @JsonSetter("product_name")                         // Сопоставление с JSON свойством
    public void setName(String name) {
        this.name = name;
    }

    @JsonSetter(value = "price", nulls = Nulls.SKIP)    // Пропускать null
    public void setPrice(BigDecimal price) {            // Если в JSON пришёл null для поля "price", то метод setPrice(null) НЕ будет вызван.
        this.price = price;
    }

    @JsonSetter(value = "tags", nulls = Nulls.AS_EMPTY) // Заменять null на пустую коллекцию
    public void setTags(List<String> tags) {
        this.tags = tags != null ? tags : new ArrayList<>();
    }
}
```

Это особенно полезно при частичном обновлении (PATCH):

- Клиент хочет изменить только name, но не трогать price
- Он отправляет {"name": "New"}, но иногда - {"name": "New", "price": null}
- Чтобы избежать случайного обнуления полей, вы говорите Jackson: "Если пришёл null - просто проигнорируй это поле"

Другие варианты Nulls.*:

- `Nulls.SKIP` - Не вызывать сеттер - значение не меняется
- `Nulls.FAIL` - Выбросить исключение
- `Nulls.AS_EMPTY` - Заменить null на «пустое» значение (например, пустой список, 0, "")
- `Nulls.SET` (по умолчанию) Вызвать сеттер с null

## `@JsonAnySetter`

### Обработка неизвестных свойств

```java
import com.fasterxml.jackson.annotation.JsonAnySetter;

import java.util.HashMap;
import java.util.Map;

/**
 * Класс FlexibleObject - "гибкий" объект, который может принимать
 * ЛЮБЫЕ дополнительные поля из JSON, даже если они не описаны явно в классе.
 *
 * Это полезно, когда:
 * - структура JSON может меняться (например, API принимает "расширения"),
 * - вы не знаете все возможные поля заранее (например, пользовательские метаданные),
 * - вы хотите сохранить "лишние" данные для дальнейшей обработки.
 */
public class FlexibleObject {

    // Основное хранилище для "неизвестных" полей
    private Map<String, Object> properties = new HashMap<>();

    /**
     * Метод с @JsonAnySetter - это "ловушка" для всех полей JSON,
     * которые НЕ соответствуют известным полям класса.
     *
     * Как работает:
     * 1. Jackson сначала пытается сопоставить каждое поле JSON с известными сеттерами/полями.
     * 2. Если поле не найдено - он ищет метод с @JsonAnySetter.
     * 3. Вызывает этот метод, передавая имя поля (key) и его значение (value).
     * 4. Вы сами решаете, что с этим делать - обычно кладёте в Map.
     */
    @JsonAnySetter
    public void addProperty(String key, Object value) {
        properties.put(key, value);
    }

    // Геттер, чтобы получить все "динамические" свойства
    public Map<String, Object> getProperties() {
        return properties;
    }

    // Пример известного поля (оно НЕ попадёт в properties!)
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

⚠️ Важно

- @JsonAnySetter работает только при десериализации (чтение JSON → объект).
- Для сериализации (объект → JSON) используется @JsonAnyGetter - чтобы записать содержимое Map обратно в JSON.
- Поля, которые явно описаны в классе (например, name), никогда не попадают в @JsonAnySetter.

## `@JsonFormat`

### Работа с датами и временем

```java
public class TimeData {

    // Java 8 Date/Time API
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;

    @JsonFormat(pattern = "HH:mm:ss")
    private LocalTime workStart;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private LocalDateTime created;

    // Старые Date классы
    @JsonFormat(pattern = "yyyy-MM-dd")
    private Date oldDate;

    // Timezone handling
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss Z", timezone = "GMT+3")
    private Date zonedDate;

    // Duration и Period
    @JsonFormat(shape = JsonFormat.Shape.STRING)
    private Duration duration;

    @JsonFormat(shape = JsonFormat.Shape.STRING)
    private Period period;
}
```

## `@JsonNaming`

### Стратегии именования

```java

@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class NamingExample {
    private Long userId;          // → user_id
    private String firstName;     // → first_name
    private String lastName;      // → last_name

    @JsonProperty("customName")   // Переопределяет стратегию
    private String specialField;  // → customName
}
```

Доступные стратегии:

- `SnakeCaseStrategy` - camelCase → snake_case
- `LowerCaseStrategy` - все в нижнем регистре
- `KebabCaseStrategy` - camelCase → kebab-case
- `UpperCamelCaseStrategy` - первая буква заглавная

## `@JsonTypeInfo`

### Обработка полиморфных типов

```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

/**
 * ЗАЧЕМ НУЖЕН @JsonTypeInfo?
 *
 * При сериализации подкласса (например, Car) в JSON Jackson "теряет" информацию о типе:
 * → сериализует только поля Car и Vehicle, но не знает, КАКИМ классом это было.
 *
 * При десериализации он не может понять: это Car, Truck или что-то ещё?
 *
 * @JsonTypeInfo решает эту проблему: он добавляет в JSON специальное
 * "поле-дискриминатор", которое указывает, какой именно подкласс использовать.
 */

public class Car extends Vehicle { /* ... */
}

public class Truck extends Vehicle { /* ... */
}

public class Motorcycle extends Vehicle { /* ... */
}

// === Основной пример: дискриминатор как отдельное поле ===
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,             // Тип идентификатора: логическое имя (не полное имя класса)
        include = JsonTypeInfo.As.PROPERTY,     // Где хранить: как отдельное поле в JSON
        property = "type"                       // Имя поля-дискриминатора
)
@JsonSubTypes({
        @JsonSubTypes.Type(value = Car.class, name = "car"),                // Car → "car"
        @JsonSubTypes.Type(value = Truck.class, name = "truck"),            // Truck → "truck"
        @JsonSubTypes.Type(value = Motorcycle.class, name = "motorcycle")   // Motorcycle → "motorcycle"
})
public abstract class Vehicle {
    private String brand;
    private String model;
    // геттеры/сеттеры...
}

/**
 * Пример JSON после сериализации Car:
 * {
 *   "type": "car",          ← поле-дискриминатор
 *   "brand": "Toyota",
 *   "model": "Camry",
 *   "doors": 4,
 *   "fuelType": "gasoline"
 * }
 *
 * При десериализации Jackson:
 * 1. Читает поле "type" → видит "car"
 * 2. Ищет @JsonSubTypes.Type с name = "car"
 * 3. Создаёт экземпляр Car.class
 */

// === Вариант 1: использовать полное имя класса ===
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
public abstract class Animal {
}

/**
 * JSON будет содержать полное имя класса:
 * { "@class": "com.example.Dog", "name": "Buddy" }
 *
 * Минусы:
 * - Зависимость от внутренней структуры пакетов (переименование класса = ломает совместимость)
 * - Длинные строки в JSON
 *
 * Плюсы:
 * - Не нужно вручную перечислять подтипы
 */

// === Вариант 2: использовать сокращённое имя класса (с "/") ===
@JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)
public abstract class Plant {
}

/**
 * JSON будет содержать сокращённое имя:
 * { "@c": "com/example/Rose" → становится "@c": "/Rose" }
 *
 * Используется редко, в основном для совместимости со старыми системами.
 */

// === Вариант 3: использовать существующее поле как дискриминатор ===
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.EXISTING_PROPERTY,  // ← Дискриминатор - это уже существующее поле!
        property = "category"                         // Имя этого поля
)
@JsonSubTypes({
        @JsonSubTypes.Type(value = Admin.class, name = "ADMIN"),
        @JsonSubTypes.Type(value = User.class, name = "USER")
})
public abstract class Person {
    // Это поле служит ДВОЯКОЙ цели:
    // 1. Хранит бизнес-значение ("ADMIN")
    // 2. Используется Jackson как дискриминатор типа
    private String category;

    public String getCategory() {
        return category;
    }

    public void setCategory(String category) {
        this.category = category;
    }
}

/**
 * JSON:
 * { "category": "ADMIN", "name": "Alice", "permissions": [...] }
 *
 * При сериализации:
 * - Jackson не добавляет новое поле - использует уже существующее "category"
 * - Значение "category" должно совпадать с name в @JsonSubTypes.Type
 *
 * При десериализации:
 * - Jackson читает "category" → выбирает класс по совпадению с name
 *
 * ⚠️ Важно: поле category ДОЛЖНО быть установлено правильно при создании объекта!
 */
```

## `@JsonFilter`

### Динамическая фильтрация полей

```java
import com.fasterxml.jackson.annotation.JsonFilter;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;

/**
 * ЗАЧЕМ НУЖЕН @JsonFilter?
 *
 * Иногда вы хотите **динамически** (во время выполнения) решать,
 * какие поля объекта сериализовать в JSON, а какие - скрыть.
 *
 * Например:
 * - Показывать пароль только в режиме отладки,
 * - Скрывать email для анонимных пользователей,
 * - Отдавать разные наборы полей для разных ролей (админ vs гость).
 *
 * @JsonFilter позволяет сделать это гибко, без дублирования классов.
 */

// 1. Помечаем класс фильтром с именем "userFilter"
@JsonFilter("userFilter")
public class User {
    private Long id;
    private String username;
    private String password;  // ← чувствительное поле
    private String email;

    // конструкторы, геттеры, сеттеры...
    public User(Long id, String username, String password, String email) {
        this.id = id;
        this.username = username;
        this.password = password;
        this.email = email;
    }

    // Геттеры (обязательны для Jackson)
    public Long getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }

    public String getEmail() {
        return email;
    }
}

/**
 * 2. Как использовать фильтр при сериализации
 */
public class JsonFilterExample {
    public static void main(String[] args) throws JsonProcessingException {
        User user = new User(1L, "alice", "secret123", "alice@example.com");

        ObjectMapper mapper = new ObjectMapper();

        // === Сценарий 1: Скрыть пароль (обычный случай) ===
        FilterProvider filters1 = new SimpleFilterProvider()
                .addFilter("userFilter",                     // имя фильтра - должно совпадать с @JsonFilter
                        SimpleBeanPropertyFilter.serializeAllExcept("password") // ← скрыть ТОЛЬКО "password"
                );

        String json1 = mapper.writer(filters1).writeValueAsString(user);
        // Результат: {"id":1,"username":"alice","email":"alice@example.com"}
        // Пароль отсутствует!

        // === Сценарий 2: Показать ВСЁ (например, для админа или логов) ===
        FilterProvider filters2 = new SimpleFilterProvider()
                .addFilter("userFilter",
                        SimpleBeanPropertyFilter.serializeAll() // ← ничего не скрывать
                );

        String json2 = mapper.writer(filters2).writeValueAsString(user);
        // Результат: {"id":1,"username":"alice","password":"secret123","email":"alice@example.com"}

        // === Сценарий 3: Скрыть несколько полей ===
        FilterProvider filters3 = new SimpleFilterProvider()
                .addFilter("userFilter",
                        SimpleBeanPropertyFilter.serializeAllExcept("password", "email")
                );

        String json3 = mapper.writer(filters3).writeValueAsString(user);
        // Результат: {"id":1,"username":"alice"}

        System.out.println("Без пароля: " + json1);
        System.out.println("Полный вывод: " + json2);
        System.out.println("Без пароля и email: " + json3);
    }
}
```

### ⚠️ Важные ограничения

- При сериализации вы обязаны предоставить реализацию этого фильтра через ObjectMapper.writer(filters) - нельзя просто
  вызвать writeValueAsString(obj)
- Если вы забудете передать фильтр - Jackson выбросит исключение:
  `JsonMappingException: No filter configured with id 'userFilter'`
- Фильтр применяется динамически - одна и та же модель может сериализоваться по-разному в зависимости от контекста.
- В Spring Boot: для использования в контроллерах нужно настраивать ObjectMapper глобально или использовать @JsonView
  как альтернативу.
- Работает только при сериализации (объект → JSON), не при десериализации.

## `@JsonUnwrapped`

### "Разворачивание" вложенных объектов

Он устраняет ненужный уровень вложенности, делая структуру данных более плоской и удобной — без изменения модели в коде.

```java
import com.fasterxml.jackson.annotation.JsonUnwrapped;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * ЗАЧЕМ НУЖЕН @JsonUnwrapped?
 *
 * Иногда у вас есть вложенный объект, но вы **не хотите**, чтобы он сериализовался
 * как отдельный вложенный блок в JSON. Вместо этого вы хотите "развернуть" его поля
 * прямо на верхний уровень.
 *
 * Это улучшает читаемость JSON и избавляет от лишнего уровня вложенности.
 */

// Вспомогательный класс — адрес
class Address {
    private String street;
    private String city;
    private String zipCode;

    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }

    // Геттеры (обязательны для Jackson)
    public String getStreet() {
        return street;
    }

    public String getCity() {
        return city;
    }

    public String getZipCode() {
        return zipCode;
    }
}

// Основной класс — пользователь
class User {

    private String name;

    // Без @JsonUnwrapped: адрес был бы вложенным объектом {"address": {...}}
    // С @JsonUnwrapped: поля address "распаковываются" в корень JSON
    @JsonUnwrapped
    private Address address;

    public User(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public String getName() {
        return name;
    }

    public Address getAddress() {
        return address;
    }
}

/**
 * Пример использования
 */
public class JsonUnwrappedExample {

    public static void main(String[] args) throws JsonProcessingException {
        Address addr = new Address("Baker St", "London", "NW1 6XE");
        User user = new User("Sherlock Holmes", addr);

        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(user);

        // Результат БЕЗ @JsonUnwrapped:
        // {"name":"Sherlock Holmes","address":{"street":"Baker St","city":"London","zipCode":"NW1 6XE"}}
        //
        // Результат С @JsonUnwrapped:
        System.out.println(json);
        // → {"name":"Sherlock Holmes","street":"Baker St","city":"London","zipCode":"NW1 6XE"}

        // Десериализация тоже работает!
        User parsed = mapper.readValue(json, User.class);
        System.out.println("Город: " + parsed.getAddress().getCity()); // → London
    }
}
```

### ⚠️ Важные нюансы

- Имена полей не должны конфликтовать!
- Если и User, и Address имеют поле id — будет ошибка или перезапись.
- Работает и при десериализации
- Jackson умеет "запаковывать" плоский JSON обратно во вложенный объект.

Можно задать префикс/суффикс (если нужно избежать конфликтов):

```java

@JsonUnwrapped(prefix = "addr_")
private Address address;
// → {"addr_street": "...", "addr_city": "..."}
```

</details>