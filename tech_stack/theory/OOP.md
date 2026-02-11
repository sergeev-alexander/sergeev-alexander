# OOP

ООП (Объектно-Ориентированное Программирование) — это парадигма программирования, в которой программа структурируется как совокупность взаимодействующих объектов, каждый из которых является экземпляром определенного класса. 

В основе лежат четыре основных принципа (столпа): Инкапсуляция, Наследование, Полиморфизм и Абстракция.

- `Encapsulation` (Инкапсуляция): Связывание данных и методов, работающих с ними, в единый объект, и сокрытие внутреннего состояния объекта от прямого доступа извне.
- `Inheritance` (Наследование): Возможность создания нового класса на основе существующего, с заимствованием его свойств и поведения и добавлением новых.
- `Polymorphism` (Полиморфизм): Возможность объектов с одинаковой спецификацией иметь различную реализацию. Один интерфейс - множество форм.
- `Abstraction` (Абстракция): Сокрытие сложной реализации и выделение только существенных характеристик объекта.

## Ключевые термины

### Класс (Class)

- Шаблон для создания объектов.
- Определяет состояние (поля) и поведение (методы), которые будут у объектов.
- Это концепция на этапе компиляции.

### Объект / Экземпляр (Object / Instance)

- Конкретная реализация класса, созданная в памяти во время выполнения программы (Runtime).
- Обладает конкретными значениями полей (состоянием).
- Термины "объект" и "экземпляр" часто используются как синонимы.

### Интерфейс (Interface)

- Это контракт, который определяет, что класс должен делать, но не как он это делает.
- Класс, реализующий интерфейс (implements), обязан предоставить реализацию всех его методов (если они не default).

---

<details>
    <summary>
        <b>4 основных столпа ООП</b>
    </summary>

## Encapsulation (Инкапсуляция)

- Объединяет данные (поля) и методы в единый блок - класс. Это делает класс "автономной единицей" с собственным состоянием и поведением.
- Ограничивает прямой доступ к внутренним данным (полям) извне класса и достигается с помощью модификаторов доступа.
- Обеспечивает **контролируемое взаимодействие с внутренним состоянием класса** только через публичные методы (геттеры/сеттеры) или другие бизнес-методы. Это позволяет:
  - Контролировать изменение данных: В сеттере можно добавить проверки перед установкой значения. В геттере можно вернуть не само поле, а его преобразованное значение.
  - Скрыть сложную внутреннюю реализацию: Внешний код не зависит от того, как внутри устроен класс. Это позволяет изменять внутреннюю логику без изменения кода, который **использует** класс, если публичный интерфейс остается прежним.
  - Повысить надежность и безопасность: Предотвращает случайное или намеренное изменение состояния объекта извне, нарушая бизнес-логику.
  - Упростить отладку: Легче отследить, где и когда изменяется состояние, если доступ идет только через методы.
  
## Inheritance (Наследование)

- Позволяет дочернему классу наследовать структуру и поведение родительского класса.
- Дочерний класс автоматически получает доступ к полям и методам родителя, при этом он может как использовать их без изменений, так и переопределять, чтобы изменить или расширить их функциональность. 
- Наследование устанавливает иерархическую связь между классами, способствуя повторному использованию кода и логической организации структуры программы.

### Поведение членов класса при наследовании в Java
- Выбор скрытого поля зависит от типа ссылки, через которую происходит доступ. Этот механизм называется Сокрытие (Field Hiding).
- Доступ к статическим членам (методам и полям) определяется типом ссылки или именем класса, через которое происходит вызов.
- Выбор переопределенного метода зависит от типа объекта, на котором вызывается метод во время выполнения (dynamic dispatch). Этот механизм называется Переопределение (Method Overriding).

### Поля (Fields)
- Механизм: Сокрытие (Field Hiding).
- Если дочерний класс объявляет поле с тем же именем, что и у родительского класса, это поле скрывает (не переопределяет!) поле родителя.
- Решение о выборе поля определяется во время компиляции на основе типа ссылочной переменной.
- Выбор скрытого поля зависит от типа ссылки, через которую происходит доступ.

```java
class Parent {
    String field = "Значение в Parent";
}
class Child extends Parent {
    String field = "Значение в Child";
}

Parent refToParent = new Parent();       // refToParent.type = Parent, object.type = Parent
Child refToChild = new Child();          // refToChild.type = Child, object.type = Child
Parent refToChildObject = new Child();   // refToChildObject.type = Parent, object.type = Child

System.out.println(refToParent.field);      // Вывод: "Значение в Parent" (type = Parent)
System.out.println(refToChild.field);       // Вывод: "Значение в Child" (type = Child)
System.out.println(refToChildObject.field); // Вывод: "Значение в Parent" (type = Parent!)
```

### Методы (Instance Methods)
- Механизм: Переопределение (Method Overriding).
- Описание: Если дочерний класс объявляет метод с той же сигнатурой (имя, тип и количество параметров), что и у родительского класса, он переопределяет родительский метод.
- Решение о выборе метода определяется во время выполнения (dynamic dispatch) на основе типа реального объекта в куче.
- Выбор переопределенного метода зависит от типа объекта, на котором вызывается метод.

```java
class Parent {
    void method() { System.out.println("Метод в Parent"); }
}
class Child extends Parent {
    
    @Override
    void method() { System.out.println("Метод в Child"); }
}

Parent refToParent = new Parent();      // refToParent.type = Parent, object.type = Parent
Child refToChild = new Child();         // refToChild.type = Child, object.type = Child
Parent refToChildObject = new Child();  // refToChildObject.type = Parent, object.type = Child

refToParent.method();      // Вывод: "Метод в Parent" (object = Parent)
refToChild.method();       // Вывод: "Метод в Child" (object = Child)
refToChildObject.method(); // Вывод: "Метод в Child" (object = Child!)
```

### Статические методы и переменные (Static Methods and Fields)
- Механизм: Связывание (и сокрытие/переопределение) во время компиляции.
- Статические методы не переопределяются: 
  - Если дочерний класс объявляет статический метод с той же сигнатурой, он скрывает (hides) родительский статический метод.
  - Статические поля могут быть скрыты (hided) аналогично обычным полям.
- Решение о выборе статического члена определяется во время компиляции на основе типа ссылочной переменной (или класса, через который вызов написан напрямую).
- Доступ к статическим членам (методам и полям) определяется типом ссылки или именем класса, через которое происходит вызов, а не типом объекта. Они не участвуют в механизме динамического полиморфизма.

```java
class Parent {
    static String staticField = "Статическое поле в Parent";
    static void staticMethod() { System.out.println("Статический метод в Parent"); }
}
class Child extends Parent {
    static String staticField = "Статическое поле в Child";
    static void staticMethod() { System.out.println("Статический метод в Child"); }
}

Parent refToChildObject = new Child();             // refToChildObject.type = Parent, object.type = Child

System.out.println(refToChildObject.staticField);  // Вывод: "Статическое поле в Parent" (type = Parent!)
refToChildObject.staticMethod();                   // Вывод: "Статический метод в Parent" (type = Parent!)

// Прямой вызов через имя класса
Parent.staticMethod(); // Вывод: "Статический метод в Parent"
Child.staticMethod();  // Вывод: "Статический метод в Child"
```

## Polymorphism (Полиморфизм)

Polymorphism (Полиморфизм) — это возможность объектов с одинаковой спецификацией (один интерфейс или родительский класс) иметь различную реализацию. 
Это позволяет одному и тому же коду работать с объектами разных типов.

### Java полиморфизм реализуется с использованием следующих элементов:

### Перегрузка методов (Method Overloading):

Позволяет определить несколько методов с одним и тем же именем в одном классе. Методы отличаются по количеству параметров, типам параметров или их порядку.

### Переопределение методов (Method Overriding):

Позволяет подклассу предоставить свою реализацию метода, который уже определен в его суперклассе. Метод в подклассе имеет тот же сигнатурный тип, что и метод в суперклассе.

### Интерфейсы и полиморфизм через интерфейсы:

Интерфейсы в Java позволяют создавать абстрактные типы данных. Полиморфизм через интерфейсы позволяет объектам разных классов реализовывать общий интерфейс и быть использованными по этому интерфейсу.

### Абстрактные классы:

Абстрактные классы могут содержать абстрактные методы, которые подклассы обязаны реализовать. Подклассы могут использоваться по типу абстрактного класса.

### Обобщенное программирование (Generics):

Позволяет создавать обобщенные (параметризованные) классы и методы, которые могут работать с объектами различных типов.

### Использование класса Object:

В Java все классы неявно наследуются от класса Object. Это позволяет использовать объекты разных типов через общий интерфейс класса Object.

## Типы полиморфизма:
- Static Polymorphism (Статический полиморфизм)
  - Разрешается на этапе компиляции (Compile-time)
  - Достигается через Method Overloading (Перегрузку методов)
  - Компилятор определяет, какой метод вызывать

- Dynamic Polymorphism (Динамический полиморфизм)
  - Разрешается во время выполнения (Runtime)
  - Достигается через Method Overriding (Переопределение методов)
  - JVM определяет, какой метод вызывать, основываясь на типе объекта

### Static Polymorphism (Статический полиморфизм)

Перегрузка методов (Method Overloading) - это возможность в одном классе объявить несколько методов с одинаковым именем, но различающихся по количеству, типу или порядку параметров. 

Выбор конкретного метода происходит во время компиляции на основе переданных аргументов.

```java
class MathOperations {
    // Перегрузка методов - статический полиморфизм
    public int add(int a, int b) {
        return a + b;
    }
    
    public double add(double a, double b) {
        return a + b;
    }
    
    public String add(String a, String b) {
        return a + b;
    }
}

// Использование - компилятор выбирает нужный метод
MathOperations math = new MathOperations();
math.add(5, 10);             // Вызывается add(int, int)
math.add(5.5, 10.2);         // Вызывается add(double, double)  
math.add("Hello", "World");  // Вызывается add(String, String)
```

### Dynamic Polymorphism (Динамический полиморфизм)

Динамический полиморфизм - это механизм, при котором вызов переопределенного метода определяется во время выполнения на основе типа реального объекта, а не типа ссылочной переменной.

```java
class PaymentProcessor {
    public void processPayment(double amount) {
        System.out.println("Processing generic payment: " + amount);
    }
}

class CreditCardProcessor extends PaymentProcessor {
    
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing credit card payment: " + amount);
    }
}

class PayPalProcessor extends PaymentProcessor {
    
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing PayPal payment: " + amount);
    }
}

// Динамический полиморфизм в действии
public class Main {
    
    public static void main(String[] args) {
        PaymentProcessor processor;
        
        // Runtime decision - JVM определяет какой метод вызывать
        processor = new CreditCardProcessor();
        processor.processPayment(100.0); // CreditCardProcessor.processPayment()
        
        processor = new PayPalProcessor();  
        processor.processPayment(200.0); // PayPalProcessor.processPayment()
        
        processor = new PaymentProcessor();
        processor.processPayment(300.0); // PaymentProcessor.processPayment()
    }
}
```

### Virtual Method Invocation (Виртуальные методы)

Java использует внутренний механизм, основанный на Virtual Method Table (VTable), для реализации динамического полиморфизма.

- VTable - это внутренняя таблица (по одной на каждый класс), где хранятся ссылки на реальные методы, которые должны быть вызваны. Для каждого метода, который может быть переопределен, в VTable содержится указатель на самую последнюю переопределенную версию этого метода в иерархии наследования.
- Когда компилятор видит вызов переопределенного метода через ссылку (например, processor.processPayment(50.0)), он генерирует код, который во время выполнения использует VTable объекта, на который указывает ссылка. JVM находит нужный метод в VTable конкретного объекта и вызывает его.
- Это позволяет одной и той же строке кода (например, processor.processPayment(...)) вести себя по-разному, в зависимости от типа реального объекта в куче, обеспечивая динамический полиморфизм.
- Этот механизм работает автоматически для всех методов, которые не объявлены как private, static или final (если они не переопределяются в подклассе).

```java
List<PaymentProcessor> processors = Arrays.asList(
                new CreditCardProcessor(),
                new PayPalProcessor(),
                new CryptoProcessor()
        );

// Динамический полиморфизм - каждый объект ведет себя соответственно своему типу
for (PaymentProcessor processor : processors) {
        // JVM использует VTable объекта, чтобы определить,
        // какую конкретно реализацию processPayment() вызвать.
        processor.processPayment(50.0);
}
```

### Модификаторы доступа при переопределении методов

При переопределении метода уровень доступности в подклассе должен быть таким же или более открытым, чем в родительском классе.

- `public`: Может быть переопределен методом с модификатором `public`
  - `protected` -> `public` (разрешено)
  - `package-private` (default) -> `public` (разрешено)
  - `private` -> `public` (нельзя, так как `private` не наследуется)

- `protected`: Может быть переопределен методом с модификатором `protected` или `public`.
  - `package-private` (default) -> `protected` (разрешено)
  - `private` -> `protected` (нельзя, так как `private` не наследуется)
    
- `package-private` (default): Может быть переопределен методом с модификатором `package-private` (default), `protected` или `public`, но только если подкласс находится в **том же пакете**. 
Если подкласс в другом пакете — переопределить нельзя, так как метод не наследуется.

- `private`: Не наследуется, следовательно, не может быть переопределен.

```java
class Parent {
    public void publicMethod() { }        // 1
    protected void protectedMethod() { }  // 2
    void packageMethod() { }              // 3 (default/package-private)
    private void privateMethod() { }      // 4 (не наследуется)
}

class Child extends Parent {
    
    // 1. Переопределение public метода
    @Override public void publicMethod() { } // OK

    // 2. Переопределение protected метода
    @Override protected void protectedMethod() { }  // OK
    @Override private void protectedMethod() { }    // ОШИБКА: нельзя уменьшить доступность

    // 3. Переопределение package-private метода (в том же пакете)
    @Override void packageMethod() { }              // OK (если Child в том же пакете, что и Parent)
    // Если Child в другом пакете, packageMethod() не наследуется, и переопределить его нельзя.

    // 4. privateMethod() не наследуется, переопределить нельзя.
    @Override private void privateMethod() { } // ОШИБКА компиляции
}
```

---

## Abstraction (Абстракция)

Абстракция — это принцип объектно-ориентированного программирования, направленный на выделение существенных характеристик объекта и сокрытие сложных деталей реализации. 
Это позволяет работать с объектами на уровне их назначения, а не внутреннего устройства.

В Java абстракция реализуется с помощью абстрактных классов (abstract class) и интерфейсов (interface).

### Абстрактный класс (abstract class)

- Не может быть создан напрямую (new запрещён).
- Может содержать как абстрактные методы (без тела, с ключевым словом `abstract`), так и обычные (конкретные) методы с реализацией.
- Наследуемый (конкретный) класс обязан реализовать все абстрактные методы, если сам не объявлен как abstract.
- Поддерживает наследование состояния (поля) и поведения (методы), что делает его удобным для создания «частично реализованных» шаблонов.

### Интерфейс (interface)

- Определяет контракт — набор методов, которые реализующий класс обязан предоставить.
- До Java 8: содержал только публичные абстрактные методы и константы (`public static final`).
- Начиная с Java 8:
  - Появились `default`-методы (`default void method() { … }`), позволяющие добавлять реализацию без нарушения обратной совместимости.
  - Появились статические методы (`static void method() { … }`).
- Начиная с Java 9: разрешено объявлять `private`-методы внутри интерфейса (для рефакторинга общего кода в `default`/`static` методах).
- Класс может реализовывать несколько интерфейсов (множественное наследование поведения).
- Все поля в интерфейсе неявно являются `public static final`.

### Ключевые различия: abstract class vs interface

| Характеристика          | abstract class                                                         | interface                                    |
|:------------------------|:-----------------------------------------------------------------------|:---------------------------------------------|
| Назначение              | Моделирует сущность («является»)                                       | Моделирует поведение / роль («умеет»)        |
| Наследование            | Одиночное (extends)                                                    | Множественное (implements)                   |
| Поля                    | Любые: private", "protected", "public", "static", "final" с состоянием | Только public static final (константы)       |
| Конструкторы            | Да (вызываются при создании подкласса)                                 | Нет                                          |
| Абстрактные методы      | Да (abstract void method();)                                           | Да (неявно public abstract)                  |
| Конкретные методы       | Да (обычные методы с реализацией)                                      | Да: default", "static", "private (с Java 8+) |
| Инициализация состояния | Возможна (через поля и конструкторы)                                   | Невозможна                                   |

### Поля в абстрактном классе и интерфейсе

Абстрактный класс может хранить состояние — обычные поля, которые наследуются подклассами:

```java
abstract class Animal {
    protected String name;           // поле с состоянием
    protected int age;
    
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public abstract void makeSound(); // абстрактный метод
}
```

Интерфейс может содержать только константы (неявно public static final):

```java
interface Flying {
    double MAX_ALTITUDE = 10000.0; // неявно public static final
    
    void fly(); // неявно public abstract
}
```

### Пример иерархии

```java
// Абстрактный класс — общая сущность
abstract class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public abstract void makeSound(); // каждый вид издаёт свой звук
    public void sleep() {
        System.out.println(name + " спит.");
    }
}

// Более специализированный абстрактный класс
abstract class Bird extends Animal {
    protected double wingSpan;

    public Bird(String name, int age, double wingSpan) {
        super(name, age);
        this.wingSpan = wingSpan;
    }

    // Птицы по умолчанию не плавают — если нужно, подкласс реализует
    public void swim() {
        System.out.println(name + " не умеет плавать.");
    }
}

// Интерфейсы — роли/поведения
interface Flying {
    double MAX_ALTITUDE = 10000.0;
    
    void fly(); // абстрактный метод
    
    default void takeOff() {
        System.out.println("Взлетаю!");
    }
    
    static void describeAbility() {
        System.out.println("Летающие существа могут подниматься в воздух.");
    }
    
    private void logFlight(double altitude) {
        if (altitude > MAX_ALTITUDE) {
            System.out.println("Предупреждение: слишком высоко!");
        }
    }
    
    default void flyTo(double altitude) {
        takeOff();
        logFlight(altitude); // вызов private-метода внутри интерфейса
    }
}

interface Swimming {
    void swim(); // переопределяет метод из Bird при необходимости
}

interface Walking {
    default void walk() {
        System.out.println("Иду по земле.");
    }
}

class Duck extends Bird implements Flying, Swimming, Walking {
    public Duck(String name, int age, double wingSpan) {
        super(name, age, wingSpan);
    }

    @Override
    public void makeSound() {
        System.out.println(name + " говорит: Кря!");
    }

    @Override
    public void fly() {
        System.out.println(name + " летит с размахом крыльев " + wingSpan + " м.");
    }

    @Override
    public void swim() {
        System.out.println(name + " плавает как утка!");
    }

    // walk() используется из default-метода интерфейса Walking
}

public class Main {
    public static void main(String[] args) {
        Duck duck = new Duck("Дональд", 3, 0.8);

        duck.makeSound();      // Кря! (реализация в Duck)
        duck.sleep();          // Дональд спит. (унаследовано от Animal)
        duck.fly();            // Дональд летит...
        duck.swim();           // Дональд плавает как утка!
        duck.walk();           // Иду по земле. (default из Walking)

        duck.flyTo(5000);      // Использует default + private методы из Flying

        Flying.describeAbility(); // Статический метод интерфейса
        System.out.println("Макс. высота: " + Flying.MAX_ALTITUDE);
    }
}
```
</details>