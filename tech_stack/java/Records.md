**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Records

> Механизм `Records` в Java введен в Java 14 для упрощения создания классов, предназначенных преимущественно для
> представления данных (data classes).
>
> Этот механизм позволяет создавать компактные и удобочитаемые классы, предоставляя сокращенный синтаксис и
> автоматически добавляя стандартные методы, такие как `equals()`, `hashCode()`, и `toString()`.
>
> Records также по умолчанию делают все свои поля неизменяемыми (final).

### ✅ Автоматические методы:

- Канонический конструктор - принимает все компоненты
- Методы доступа - `x()`, `y()` (не `getX()`)
- `equals()` - сравнивает все компоненты
- `hashCode()` - на основе всех компонентов
- `toString()` - в формате `Point[x=10, y=20]`

### ✅ Дополнительные особенности:

- Все поля являются private и final
- Класс является final (не может быть унаследован)
- Неявно расширяет java.lang.Record

```java
// Простой record
public record Point(int x, int y) {
}
```

```java
// Record с большим числом параметров
public record Person(
                String name,
                String email,
                LocalDate birthDate
        ) {
}
```

```java
// Параметризованный Record
record Pair<K, V>(K key, V value) {
}
```

## Конструктор в Record

> У record всегда есть канонический конструктор, принимающий все компоненты.
> Вы можете добавлять валидацию, дополнительные конструкторы или переопределять канонический, но не можете убрать
> обязательные параметры или сделать компоненты изменяемыми.

### Компактный конструктор

```java
public record Person(String name, int age) {
    // Компактный конструктор
    public Person {
        // Можно делать валидацию
        if (age < 0) {
            throw new IllegalArgumentException("Возраст не может быть отрицательным");
        }

        // Можно присвоить новые значения
        name = "Name=" + name;
        age = age > 0 ? age : 18;
        // Присвоение происходит после всего кода в компактном конструкторе
        // Это рекомендованный конструктор, если нужно присвоить новые значения
    }
}

public record Person(String name, int age) {
    // Это уже НЕ компактный конструктор
    // Присвоить новые значения можно также с явным вызовом this()
    // Но тогда нужно в this() передать другие значения  
    public Person(String name, int age) {
        // Можно делать валидацию
        if (age < 0) {
            throw new IllegalArgumentException("Возраст не может быть отрицательным");
        }

        // Вызов this() с новыми значениями  
        this("Name=" + name, Math.max(age, 18));
    }
}

public record Person(String name, int age) {
    // Это уже НЕ компактный конструктор
    // Присвоить новые значения можно также с явным вызовом this()
    // Но тогда нужно в this() передать другие значения
    public Person(String name, int age) {
        // Можно делать валидацию
        if (age < 0) {
            throw new IllegalArgumentException("Возраст не может быть отрицательным");
        }
        
        String newName = "Name=" + name; 
        int newAge = Math.max(age, 18);

        // Вызов this() с новыми значениями  
        this(newName, newAge);
    }
}

public record Person(String name, int age) {
    // Это уже НЕ компактный конструктор
    public Person(String name, int age) {
        // Можно делать валидацию
        if (age < 0) {
            throw new IllegalArgumentException("Возраст не может быть отрицательным");
        }

        // Можно изменить переприсваиванием
        age = Math.max(age, 18);

        // Затем передать измененное значение в канонический конструктор
        this(name, age);
    }
}

/*
В конструкторе record нельзя использовать this.age = ... или this.name = ....
Это запрещено компилятором - даже если поля доступны для чтения, присваивание им запрещено.        
 */

// ⚠️ ТАК НЕЛЬЗЯ:
public record Person(String name, int age) {
    public Person(String name, int age) {
        // Попытка явного присвоения:
        this.name = "Name=" + name;  // ← ОШИБКА КОМПИЛЯЦИИ!
        this.age = Math.max(age, 18); // ← ОШИБКА КОМПИЛЯЦИИ!
        // "cannot assign a value to final variable 'name'"
    }
}

// Чтение переменной через this.variableName разрешено
public record Person(String name, int age) {
    public Person {
        System.out.println("Old name: " + this.name); // ✅ разрешено (чтение)
        name = "Name=" + name;  // ✅ разрешено (изменение псевдопеременной)
        // this.name = ...;     // ❌ запрещено (запись)
    }
}
```

### Канонический конструктор

- Автоматически генерируется компилятором.
- Принимает все компоненты записи в том порядке, в котором они объявлены в заголовке.
- Инициализирует private final поля.
- Публичный (public), даже если явно не объявлен.

```java
record Person(String name, int age) {
}
// Эквивалентно:
// public Person(String name, int age) { this.name = name; this.age = age; }
```

### Конструктор без параметров

- Генерируется только если в записи нет компонентов:
- Не генерируется, если есть хотя бы один компонент.
- Нельзя объявить вручную, если есть компоненты (иначе final-поля останутся неинициализированными -> ошибка компиляции).

```java
record Empty() {
}
```

### Пользовательский канонический конструктор

- Можно переопределить, чтобы добавить валидацию, логирование и т.д.
- Должен инициализировать все компоненты (обычно через неявное присваивание или явный вызов this(...) - но в record’е
  это делается особым образом).

```java
record Person(String name, int age) {

    // Это называется компактный канонический конструктор - тело дополняет, но не заменяет инициализацию.
    public Person {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Имя не может быть пустым");
        }
        if (age < 0) {
            throw new IllegalArgumentException("Возраст не может быть отрицательным");
        }
        // Присваивание this.name = name и this.age = age происходит автоматически
    }
}
```

### Если нужно переименовать параметры

```java
record Person(String name, int age) {
    // Это НЕ канонический конструктор
    public Person(String fullName, int years) {
    // Должен вызвать канонический конструктор
        this(fullName, years);  // Вызов неявного канонического конструктора
    }
}
```

### Дополнительные (нестандартные) конструкторы

- Можно объявлять, но они обязаны вызывать канонический конструктор через this(...)

```java
record Point(int x, int y) {
    // Дополнительный конструктор
    public Point() {
        this(0, 0); // Должен вызывать основной конструктор
    }
}

record Person(String name, int age) {
    // Дополнительный конструктор
    public Person(String name) {
        this(name, 0); // Должен вызывать основной конструктор
    }
}
```

## Поля в Record

- Компоненты записи автоматически становятся `private final` полями
- Каждый параметр в заголовке record’а превращается в приватное неизменяемое поле.
- Они final, и компилятор не даст присвоить им новое значение. Это гарантирует неизменяемость данных, что является
  ключевой идеей record’а.

```java
record Box(int width, String label) {
}
// -> private final int width;
// -> private final String label;
```

### Можно объявлять дополнительные поля в теле record’а

- Такие поля не являются компонентами записи.
- Они могут быть изменяемыми (non-final), но это не рекомендуется

> 🔴 До Java 16 (включительно)
> - Нельзя объявлять никакие instance-поля в record - ни private, ни public, ни final, ни volatile.
> - Компилятор выдавал: records cannot declare instance fields
> - ✅ Разрешались только static поля.
> 
> 🟢 Начиная с Java 16+ (JEP 395 - Records, финализировано в Java 16)
> - Можно объявлять дополнительные instance-поля, в том числе private, не-final, и даже protected / public - но с оговорками.

```java
record Counter(String id) {
    // Это не компонент записи, не участвует в equals/hashCode/toString, может быть изменяемым.
    private int count = 0; // <- дополнительное поле

    public void inc() {
        count++;
    }

    public int count() {
        return count;
    }
}
```

- Дополнительные поля не участвуют в `equals()`, `hashCode()`, `toString()`
- Эти методы генерируются только на основе компонентов из заголовка.
- Поэтому, даже если `count` изменится, два объекта Counter("x") останутся равными.

```java
public record Point(int x, int y) {

    // Можно добавлять свои методы
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }

    // Можно переопределить автоматические методы
    @Override
    public String toString() {
        return String.format("Точка(%d, %d)", x, y);
    }

    // Статические методы и поля
    public static final Point ORIGIN = new Point(0, 0);

    public static Point of(int x, int y) {
        return new Point(x, y);
    }

    // ⚠️ ТЕОРЕТИЧЕСКИ можно переопределить x() и y(), но ЭТОГО СЛЕДУЕТ ИЗБЕГАТЬ:
    // - нарушается прозрачность данных
    // - equals()/hashCode() используют исходные поля, а не результаты этих методов
    // - легко получить логически несогласованные объекты

    @Override
    public int x() {
        return Math.abs(x); // пример - но не делайте так!
    }

    @Override
    public int y() {
        return Math.abs(y); // пример - но не делайте так!
    }

    // ✅ Если нужно другое поведение - создавайте отдельные методы:
    public int absoluteX() {
        return Math.abs(x);
    }

    public int absoluteY() {
        return Math.abs(y);
    }
}
```

### Допустимые случаи использования дополнительных полей

- ✅ Кэширование hashCode (lazy, thread-safe)
  - Цель: избежать повторных дорогостоящих вычислений hashCode при частом использовании в HashMap, HashSet.
- Почему допустимо:
  - Поле не влияет на логику равенства (equals зависит только от компонентов).
  - Используется volatile для потокобезопасности + double-checked locking.
  - Не нарушает иммутабельность видимого состояния.

```java
public record Email(String local, String domain) {
    // Кэш хэш-кода: ленивый, потокобезопасный, не влияет на equals/toString
    private volatile int hash;

    @Override
    public int hashCode() {
        int h = hash;
        if (h == 0 && (local != null || domain != null)) {
            h = Objects.hash(local, domain);
            hash = h;
        }
        return h;
    }

    // equals и toString остаются сгенерированными - работают только с local/domain
}
```

> 💡 Это оптимизация, а не новое состояние. Значение `hash` полностью определяется компонентами - даже если два объекта new Email("a","b") имеют разные hash (ноль vs неноль), они всё равно равны.

### ✅ Lazy-initialized derived data (производные данные)

- Цель: избежать повторного парсинга/вычисления «тяжёлых» значений при каждом вызове метода.
- Почему допустимо:
  - Данные производные от компонентов (чистые функции).
  - Вычисляются один раз и неизменны после этого (final или volatile).
  - Методы возвращают неизменяемые объекты (оборачиваем в Optional, List.copyOf и т.п.).

```java
public record LogEntry(String raw) {
    // Производные данные: парсятся однократно, неизменны, не влияют на equals
    private transient String[] parts;
    private transient String timestamp;
    private transient String message;

    // Инициализация при первом доступе
    private void ensureParsed() {
        if (parts == null) {
            synchronized (this) {
                if (parts == null) {
                    parts = raw.split(" \\| ", 3);
                    timestamp = parts.length > 0 ? parts[0] : "";
                    message = parts.length > 2 ? parts[2] : "";
                }
            }
        }
    }

    public String timestamp() {
        ensureParsed();
        return timestamp;
    }

    public String message() {
        ensureParsed();
        return message;
    }

    // Компонент raw остаётся единственным источником истины
    // equals/hashCode/toString - работают только с raw
    
    /*
    raw - каноническое состояние.
    timestamp, message - просто кэшированные представления этого состояния.
    Если raw одинаков -> все производные данные одинаковы -> equals() корректен.
     */
}
```

> Дополнительные поля в record допустимы, если они:
> - производные от компонентов
> - неизменны после инициализации
> - не влияют на `equals`/`hashCode`
> - служат для оптимизации (кэш) или удобства (парсинг)

### Best Practices

```java
// 1. Всегда защищайте mutable компоненты
public record Config(Map<String, String> settings) {
    public Config {
        settings = Map.copyOf(settings); // Защитная копия
    }

    @Override
    public Map<String, String> settings() {
        return Map.copyOf(settings); // Возвращаем копию
    }
}

// 2. Используйте статические фабричные методы для сложной логики
public record Email(String value) {
    private static final Pattern EMAIL_PATTERN = ...;

    public static Email of(String value) {
        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("Invalid email");
        }
        return new Email(value.toLowerCase());
    }
}

// 3. Локальные records (Java 16+)
public List<Point> processData(List<int[]> raw) {
    record TempResult(int x, int y, boolean valid) {}

    return raw.stream()
            .map(arr -> new TempResult(arr[0], arr[1], arr[0] > 0))
            .filter(TempResult::valid)
            .map(tr -> new Point(tr.x(), tr.y()))
            .toList();
}
```

## Примеры использования

### DTO:

```java
public record UserDTO(
        Long id,
        String username,
        String email,
        UserRole role
) implements Serializable {
}
```

### Возврат нескольких значений:

```java
// ✅ Record как типобезопасный способ вернуть несколько связанных значений из метода.
// Идеально подходит для результатов валидации, парсинга, операций с ошибками и т.п.
public record ValidationResult(
                boolean isValid,        // флаг: прошла ли проверка успешно
                List<String> errors     // список ошибок (пустой, если isValid == true)
        ) {
}

// Метод возвращает составной результат: не просто "да/нет", а ещё и причины при отказе.
public ValidationResult validate(User user) {
    List<String> errors = new ArrayList<>();

    if (user.name() == null || user.name().isBlank()) {
        errors.add("Имя не должно быть пустым");
    }
    if (user.email() == null || !user.email().contains("@")) {
        errors.add("Некорректный email");
    }

    boolean valid = errors.isEmpty();

    // Возвращаем неизменяемый, самодокументированный объект.
    // Нет необходимости писать класс ValidationResult вручную - record делает всё за нас.
    return new ValidationResult(valid, errors);
}

// Пример использования:
// ValidationResult result = validate(user);
// if (!result.isValid()) {
//     result.errors().forEach(System.err::println);
// }
```

### Ключи в Map:

```java
record CacheKey(String type, String id) {
}

Map<CacheKey, Object> cache = new HashMap<>();
cache.

put(new CacheKey("user", "123"),userData);
```

## Когда использовать Records

- ✅ Идеально для:
  - Value Objects (Money, Distance, Range)
  - DTO (API request/response, события)
  - Ключи в Map/Set
  - Возврат нескольких значений из метода
  - Временные данные в streams/lambdas

- ❌ Не подходит для:
  - JPA/Hibernate сущности (нужны сеттеры, прокси)
  - Сложные бизнес-объекты с поведением
  - Классы, требующие наследования
  - Объекты с изменяемым состоянием