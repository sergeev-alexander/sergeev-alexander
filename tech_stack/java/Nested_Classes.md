# Nested classes

> Nested class (вложенный класс) в Java - это класс, определенный внутри другого класса. Вложенные классы могут быть статическими (static) или нестатическими (non-static). Они используются для логической группировки классов и улучшения структуры кода

- ### Логическая организация кода:
  - Вложенные классы используются для группировки классов, которые логически связаны между собой. Это может повысить читаемость и понимание кода.
- ### Инкапсуляция:
  - Вложенные классы могут быть использованы для инкапсуляции логически связанных классов и скрытия их деталей реализации от внешнего кода.
- ### Уменьшение видимости:
  - Если вложенный класс объявлен как private, его члены будут недоступны за пределами внешнего класса, что может уменьшить видимость и предоставить лучшее управление доступом.
- ### Использование внутри другого класса:
  - Вложенные классы часто используются, когда класс имеет смысл только в контексте другого класса. Это позволяет сделать код более локализованным и уменьшить глобальное пространство имен.
- ### Упрощение структуры кода:
  - Вложенные классы могут использоваться для упрощения структуры кода и избежания создания большого количества небольших файлов классов.

## Статический вложенный класс (static nested class)
   
- Когда нужна логическая группировка, но без привязки к экземпляру внешнего класса
- Для вспомогательных классов, относящихся к внешнему (например, `Entry` в `Map`).
- В `record`/`sealed` - для кэширования, билдеров и т.д.

```java
public class Database {
    private String url;

    // Статический вложенный - не зависит от конкретного Database
    public static class Config {
        private String host;
        private int port;

        public Config(String host, int port) {
            this.host = host;
            this.port = port;
        }

        // Может обращаться к private-членам Database!
        private String buildUrl() {
            return "jdbc:postgresql://" + host + ":" + port;
        }
    }

    public Database(Config config) {
        this.url = config.buildUrl(); // ✅ доступ к private-методу!
    }
}

// Использование:
Database.Config cfg = new Database.Config("localhost", 5432);
Database db = new Database(cfg);
```

- Компактно - Config «живёт» внутри Database, но не создаёт overhead.
- Безопасно - никаких скрытых ссылок → нет риска утечки памяти.
- Полный доступ к private внешнего класса (даже если Config - public).

## Внутренний (non-static) класс (inner class)

- Когда объект внутреннего класса логически принадлежит экземпляру внешнего.
- Для итераторов, слушателей событий, состояния, зависящего от внешнего объекта.

```java
public class Car {
    private String model;
    private int fuel = 50;

    // Внутренний класс - машина "владеет" двигателем
    public class Engine {
        public void start() {
            if (fuel > 0) {
                System.out.println(model + " заводится!"); // ✅ доступ к model
                fuel -= 5;
            }
        }

        // Явный доступ к внешнему экземпляру:
        public Car getCar() {
            return Car.this; // <- ⚠️ особый синтаксис!
        }
    }

    public Engine getEngine() {
        return new Engine(); // <- ⚠️ создаётся "привязанным" к this (к экземпляру)
    }
}

// Использование:
Car car = new Car("Tesla", 50);
Car.Engine engine = car.getEngine(); // <- engine "знает", что принадлежит конкретному car
engine.start(); // -> Tesla заводится!
```

## ⚠️ Подводные камни

### Скрытая ссылка -> утечки памяти

```java
public class Heavy {
    // Большой объект - 1 МБ
    private final byte[] data = new byte[1_000_000];

    // Метод возвращает задачу (например, для пула потоков)
    public Runnable createTask() {
        // Вот здесь создаётся анонимный ВНУТРЕННИЙ класс.
        // Компилятор неявно генерирует что-то вроде:
        //   class Heavy$1 implements Runnable {
        //       
        //       private final Heavy this$0;     <- СКРЫТАЯ ССЫЛКА НА ВНЕШНИЙ ЭКЗЕМПЛЯР
        //       
        //       Heavy$1(Heavy outer) {
        //          this$0 = outer; 
        //       }
        //
        //       public void run() { 
        //           System.out.println("Done"); 
        //       }
        //   }
        return new Runnable() {
            @Override
            public void run() {
                System.out.println("Done");
            }
        };
        // → Новый объект Runnable содержит скрытое поле, ссылающееся на этот Heavy
    }
}
```

Что произойдёт при использовании?

```java
public class Cache {
    // Долгоживущий кэш (например, веб-кэш, пуле задач)
    private static final List<Runnable> TASKS = new ArrayList<>();

    public static void addTask(Runnable task) {
        TASKS.add(task); // <- кладём в долгоживущее хранилище
    }
}

// Где-то в коде:
Heavy heavy = new Heavy();              // <- создаём "тяжёлый" объект
Runnable task = heavy.createTask();     // <- task "держит" ссылку на heavy!
Cache.addTask(task);                    // <- task попадает в статический список
heavy = null;                           // <- "забываем" heavy - но он НЕ СОБЁТСЯ GC!
```

### Исправление

Лямбда (лучший способ):

```java
public Runnable createTask() {
    return () -> System.out.println("Done");
    // ✅ Лямбда НЕ создаёт внутренний класс, если не захватывает this.
    // Компилятор генерирует:
    // - либо static метод + invokedynamic (без ссылки на this),
    // - либо внутренний класс - ТОЛЬКО если вы используете this/data.
}
```

Если всё же нужен класс - сделайте его static:

```java
public Runnable createTask() {
    // Локальный static-класс (Java 16+)
    static class DoneTask implements Runnable {
        @Override
        public void run() { 
            System.out.println("Done"); 
        }
    }
    
    return new DoneTask();
}
```

Решение: сделайте статическим, если не нужен доступ к this:

```java
public Runnable createTask() {
    return new Runnable() {  // <- компилятор сделает его static-подобным (если не захватывает this)
        @Override
        public void run() { 
            System.out.println("Done"); 
        }
    };
}
// ИЛИ явно:
    return () -> System.out.println("Done"); // <- лямбда - никогда не захватывает this, если не нужно
```

### Нельзя создать без экземпляра внешнего класса

```java
Car.Engine e = new Car.Engine(); // ❌ Ошибка: "non-static class Engine cannot be referenced from a static context"
Car car = new Car("BMW", 40);
Car.Engine e = car.new Engine(); // ✅ Синтаксис: outer.new Inner()
```

### Сложно сериализовать

> Если внешний класс `Car` реализует `Serializable`, то внутренний класс `Engine` тоже должен реализовывать `Serializable`,
потому что при сериализации `Engine` требуется сериализовать и скрытую ссылку на `Car.this`.
>
> При десериализации сначала восстанавливается `Car`, потом - `Engine`,
что усложняет процесс и может привести к `InvalidClassException`, если версии не совпадают.

### ✅ Решение: используйте `static class Engine` - тогда `Engine` сериализуется независимо, без `Car`. 

## Best practice

- ### `static class` = просто класс внутри другого - без привязки
- ### `class` (без `static`) = кусочек состояния внешнего объекта
- ### Если не используете `Outer.this` - сделайте `static`.
- ### Анонимные и локальные - на крайний случай.




