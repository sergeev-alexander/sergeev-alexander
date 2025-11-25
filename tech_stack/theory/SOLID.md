## SOLID: основы

**SOLID** - это аббревиатура, объединяющая **пять фундаментальных принципов объектно-ориентированного проектирования (ООП)**, сформулированных Робертом Мартином (Robert C. Martin) в начале 2000-х. Эти принципы призваны помочь разработчикам создавать программное обеспечение, которое:

- легко **поддерживать**,
- просто **расширять**,
- минимизирует **связность** между компонентами,
- максимизирует **тестируемость** и **гибкость**.

SOLID не является набором строгих правил, а скорее **руководством к размышлению** о структуре кода. Соблюдение этих принципов снижает риск возникновения «хрупкого» кода, трудного для модификации и сопровождения.

Каждая буква в SOLID означает один принцип:

- **`S`** - **Single Responsibility Principle** (Принцип единственной ответственности)
- **`O`** - **Open/Closed Principle** (Принцип открытости/закрытости)
- **`L`** - **Liskov Substitution Principle** (Принцип подстановки Барбары Лисков)
- **`I`** - **Interface Segregation Principle** (Принцип разделения интерфейсов)
- **`D`** - **Dependency Inversion Principle** (Принцип инверсии зависимостей)

Эти принципы тесно связаны между собой и часто работают в совокупности: нарушение одного может привести к проблемам с соблюдением других.

<details>
    <summary>
        <b>Single Responsibility Principle (Принцип единственной ответственности)</b>
    </summary>

> **«У класса должна быть только одна причина для изменения»** - Роберт Мартин

### Суть

Класс (или модуль) должен быть **ответственен только за одну часть функциональности** системы. 

Это означает, что каждый класс должен решать лишь одну конкретную задачу. 
Он должен быть ответственен только за одну часть функциональности, и эта ответственность должна быть полностью инкапсулирована в класс.

Зачем это нужно:
- Понимание кода: Классы становятся меньше и понятнее.
- Тестирование: Легче написать unit-тесты для одной функциональности.
- Сопровождение: Изменения в одной части программы не ломают другие.
- Снижение связности: Классы становятся менее зависимыми друг от друга.
- Переиспользование: Проще использовать класс в другом месте, когда он делает что-то одно.

Если вы можете назвать более одной причины, почему класс может измениться - значит принцип нарушен.

### ❌ Нарушение SRP
```java
// ПЛОХО: Класс имеет несколько ответственностей.
class Employee {
    
    private String name;
    private String position;
    private double salary;

    // Конструкторы, геттеры и сеттеры...

    // Ответственность 1: Управление данными сотрудника
    public void saveToDatabase() {
        // Код для сохранения сотрудника в базу данных
        System.out.println("Сохранение сотрудника в БД...");
    }

    // Ответственность 2: Генерация отчета
    public void generatePayrollReport() {
        // Код для генерации отчета по зарплате
        System.out.println("Генерация отчета по зарплате для " + this.name);
    }

    // Ответственность 3: Расчет налогов (еще одна потенциальная причина для изменения)
    public double calculateTax() {
        // Логика расчета налога
        return salary * 0.13;
    }
}
```
- Если изменится структура базы данных, придется менять класс Employee.
- Если потребуется изменить формат отчета (например, в PDF вместо консоли), придется менять класс Employee.
- Если изменятся налоговые ставки, мы снова лезем в этот же класс.

У этого класса как минимум три причины для изменения.

### ✅ Соблюдение SRP

```java
// Класс отвечает ТОЛЬКО за представление данных сотрудника.
class Employee {
    
    private String name;
    private String position;
    private double salary;

    // Конструкторы, геттеры и сеттеры...
    // Только методы, связанные с основными данными объекта.
}

// Класс отвечает ТОЛЬКО за работу с хранилищем данных.
class EmployeeRepository {
    
    public void save(Employee employee) {
        // Код для сохранения в БД, файл и т.д.
        System.out.println("Сохранение сотрудника в БД...");
    }

    public Employee findById(int id) {
        // Код для поиска сотрудника
        return null; // заглушка
    }
}

// Класс отвечает ТОЛЬКО за генерацию отчетов.
class ReportService {
    
    public void generatePayrollReport(Employee employee) {
        // Код для генерации отчета в любом формате
        System.out.println("Генерация отчета по зарплате для " + employee.getName());
    }
}

// Класс отвечает ТОЛЬКО за налоговые расчеты.
class TaxCalculator {
    
    private static final double INCOME_TAX_RATE = 0.13;

    public double calculateIncomeTax(Employee employee) {
        return employee.getSalary() * INCOME_TAX_RATE;
    }
}
```

### Как определить нарушение SRP?

- Класс имеет слишком много методов, не связанных напрямую с его основной целью.
- Один класс часто меняется по разным, несвязанным друг с другом причинам.
- При описании класса приходится использовать союз "и" (например, "этот класс отвечает за данные пользователя и за рассылку email и за логирование").
- Класс имеет зависимости от множества внешних систем (БД, сеть, файловая система, API и т.д.).
</details>

<br>

<details>
    <summary>
        <b>Open/Closed Principle (Принцип открытости/закрытости)</b>
    </summary>

> **«Программные сущности (классы, модули, функции) должны быть открыты для расширения, но закрыты для модификации»** - Бертран Мейер (уточнен Робертом Мартином)

### Суть
Код должен быть спроектирован так, чтобы **новую функциональность можно было добавлять без изменения существующего кода**. Это достигается за счёт **абстракций** (интерфейсов, абстрактных классов) и **полиморфизма**.

- **Открыт для расширения**: можно добавить новое поведение (например, новый тип отчёта или способ оплаты).
- **Закрыт для модификации**: не нужно править рабочий, протестированный код.

Этот принцип лежит в основе **гибких и масштабируемых архитектур**, особенно в системах с частыми требованиями к расширению (плагины, стратегии, обработчики).

Преимущества соблюдения OCP
- Повышение стабильности: Меньше риск сломать существующую функциональность
- Упрощение тестирования: Новый функционал добавляется через новые классы, которые тестируются отдельно
- Снижение связанности: Компоненты слабее связаны между собой
- Облегчение поддержки: Легче отслеживать, где и какие изменения происходят
- Возможность "горячей" замены: Поведение можно менять во время выполнения программы

### ❌ Нарушение OCP
```java
// ПЛОХО: При добавлении новой фигуры нужно изменять существующий класс
class AreaCalculator {
    public double calculateArea(Object shape) {
        if (shape instanceof Circle) {
            Circle circle = (Circle) shape;
            return Math.PI * circle.getRadius() * circle.getRadius();
        } else if (shape instanceof Rectangle) {
            Rectangle rectangle = (Rectangle) shape;
            return rectangle.getWidth() * rectangle.getHeight();
        }
        // Добавление новой фигуры требует изменения этого метода!
        throw new IllegalArgumentException("Unknown shape: " + shape.getClass());
    }
}

class Circle {
    private double radius;
    
    public Circle(double radius) { this.radius = radius; }
    public double getRadius() { return radius; }
}

class Rectangle {
    private double width;
    private double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    public double getWidth() { return width; }
    public double getHeight() { return height; }
}
```

Проблема: Каждый раз при добавлении новой фигуры (Треугольник, Квадрат и т.д.) нам приходится модифицировать класс AreaCalculator, добавляя новые if-else ветки.

### ✅ Соблюдение OCP через абстракцию

Создадим общий интерфейс для всех фигур и делегируем вычисление площади самим фигурам.

```java
// ХОРОШО: Новые фигуры можно добавлять без изменения существующего кода
interface Shape {
    double calculateArea();
}

class Circle implements Shape {
    
    private double radius;

    public Circle(double radius) { this.radius = radius; }

    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

class Rectangle implements Shape {
    
    private double width;
    private double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return width * height;
    }
}

// Класс AreaCalculator теперь закрыт для модификации
class AreaCalculator {
    
    public double calculateArea(Shape shape) {
        return shape.calculateArea(); // Делегируем вычисление фигуре
    }

    // Дополнительный метод для вычисления суммы площадей нескольких фигур
    public double calculateTotalArea(List<Shape> shapes) {
        return shapes.stream()
                .mapToDouble(Shape::calculateArea)
                .sum();
    }
}
```
Теперь мы можем легко добавлять новые фигуры:
```java
// Добавляем новую фигуру БЕЗ изменения AreaCalculator
class Triangle implements Shape {
    
    private double base;
    private double height;
    
    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return 0.5 * base * height;
    }
}

// Использование
public class Main {
    public static void main(String[] args) {
        AreaCalculator calculator = new AreaCalculator();
        
        List<Shape> shapes = Arrays.asList(
            new Circle(5),
            new Rectangle(4, 6),
            new Triangle(3, 4) // Новая фигура - никаких изменений в calculator!
        );
        
        double totalArea = calculator.calculateTotalArea(shapes);
        System.out.println("Total area: " + totalArea);
    }
}
```
---
### Другой пример: Система скидок:

### ❌ Нарушение OCP

```java
class DiscountCalculator {
    public double calculateDiscount(Customer customer, double amount) {
        if (customer.getType().equals("REGULAR")) {
            return amount * 0.1; // 10% скидка
        } else if (customer.getType().equals("VIP")) {
            return amount * 0.2; // 20% скидка
        } else if (customer.getType().equals("PREMIUM")) {
            return amount * 0.3; // 30% скидка
        }
        // Добавление нового типа клиента требует изменения этого метода!
        return 0;
    }
}
```

✅ Соблюдение OCP через стратегию

```java
// Интерфейс для стратегии скидок
interface DiscountStrategy {
    
    double calculateDiscount(double amount);
}

// Конкретные стратегии
class RegularDiscount implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.1;
    }
}

class VIPDiscount implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.2;
    }
}

class PremiumDiscount implements DiscountStrategy {
    
    @Override
    public double calculateDiscount(double amount) {
        return amount * 0.3;
    }
}

// Класс DiscountCalculator закрыт для модификации
class DiscountCalculator {
    
    public double calculateDiscount(DiscountStrategy strategy, double amount) {
        return strategy.calculateDiscount(amount);
    }
}

// Использование
public class Main {
    
    public static void main(String[] args) {
        DiscountCalculator calculator = new DiscountCalculator();
        
        // Легко добавляем новые типы скидок без изменения calculator
        DiscountStrategy seasonalDiscount = new DiscountStrategy() {
            
            @Override
            public double calculateDiscount(double amount) {
                return amount * 0.15; // Сезонная скидка 15%
            }
        };
        
        double discount = calculator.calculateDiscount(seasonalDiscount, 1000);
        System.out.println("Discount: " + discount);
    }
}
```

## Паттерны проектирования, помогающие соблюдать OCP

### Стратегия (Strategy)

Определяет семейство алгоритмов, инкапсулирует каждый из них и делает их взаимозаменяемыми.

- Позволяет добавлять новые варианты поведения без изменения существующего контекста
- Контекст становится закрытым для модификации - он работает с абстракцией стратегии
- Новые алгоритмы добавляются как новые классы, реализующие общий интерфейс
- Идеально для сценариев, где нужно менять поведение объекта во время выполнения

### Декоратор (Decorator)

Динамически добавляет объекту новые обязанности, являясь альтернативой порождению подклассов.

- Позволяет расширять функциональность объектов без модификации их исходного кода
- Можно комбинировать различные декораторы для создания сложного поведения
- Каждый декоратор добавляет одну конкретную функциональность
- Соответствует принципу "композиция предпочтительнее наследования"

### Наблюдатель (Observer)

Определяет зависимость "один-ко-многим" между объектами, чтобы при изменении состояния одного объекта все зависящие от него объекты уведомлялись автоматически.

- Издатель (Subject) закрыт для модификации - он не знает о конкретных подписчиках
- Новые подписчики могут добавляться без изменения кода издателя
- Издатель работает с абстракцией наблюдателей, а не с конкретными реализациями
- Отличный пример "открытости для расширения" - подписка на события

### Шаблонный метод (Template Method)

Определяет скелет алгоритма, перекладывая ответственность за некоторые его шаги на подклассы.

- Базовый класс закрыт для модификации - алгоритм фиксирован
- Подклассы могут переопределять определенные шаги алгоритма (расширять поведение)
- Структура алгоритма остается неизменной, но отдельные шаги могут варьироваться
- Защищает инварианты алгоритма от случайного нарушения

### Фабричный метод (Factory Method)

Определяет интерфейс для создания объекта, но оставляет подклассам решение о том, какой класс инстанцировать.

- Код, использующий фабрику, закрыт для модификации при добавлении новых продуктов
- Новые типы продуктов можно добавлять через создание новых фабрик
- Клиентский код зависит от абстракций (продуктов), а не от конкретных классов
- Позволяет легко расширять семейства создаваемых объектов

### Абстрактная фабрика (Abstract Factory)

Предоставляет интерфейс для создания семейств связанных или зависимых объектов без указания их конкретных классов.

- Позволяет добавлять новые семейства продуктов без изменения существующего клиентского кода
- Клиент работает только с абстрактными интерфейсами
- Замена всего семейства объектов происходит через подмену одной фабрики
- Идеален для кроссплатформенных и сменяемых тем оформления

### Посетитель (Visitor)

Описывает операцию, которая выполняется над объектами других классов, позволяя добавлять новые операции без изменения этих классов.

- Структура объектов (элементы) закрыта для модификации
- Новые операции добавляются как новые классы посетителей
- Позволяет отделить алгоритмы от структур данных, над которыми они работают
- Особенно полезен, когда нужно много различных операций над одной структурой

### Команда (Command)

Инкапсулирует запрос как объект, позволяя параметризовать клиентов с различными запросами, организовывать очередь или протокол запросов, а также поддерживать отмену операций.

- Invoker (вызывающий) закрыт для модификации - он работает с абстрактной командой
- Новые команды добавляются как новые классы, реализующие интерфейс Command
- Позволяет легко добавлять сложные операции без изменения существующего кода
- Поддерживает undo/redo функциональность

### Мост (Bridge)

Разделяет абстракцию и реализацию так, чтобы они могли изменяться независимо.

- Абстракция и реализация могут расширяться независимо друг от друга
- Новые реализации можно добавлять без изменения абстракций
- Новые абстракции можно создавать, используя существующие реализации
- Преодолевает ограничения наследования "взрывом классов"

### Цепочка ответственности (Chain of Responsibility)

Позволяет передавать запрос по цепочке потенциальных обработчиков, пока один из них не обработает запрос.

- Клиентский код закрыт для модификации - он отправляет запрос в цепочку
- Новые обработчики можно добавлять без изменения существующей цепочки
- Позволяет динамически конфигурировать цепочки обработки
- Каждый обработчик решает одну конкретную задачу

### Общие принципы, объединяющие эти паттерны:

- Инкапсуляция изменений - каждый паттерн инкапсулирует аспект, который может изменяться
- Программирование на уровне интерфейсов - зависимости строятся на абстракциях, а не на конкретных реализациях
- Композиция предпочтительнее наследования - большинство паттернов используют композицию для достижения гибкости
- Разделение ответственностей - паттерны помогают разделить различные аспекты системы
- Однонаправленность зависимостей - зависимости направлены от конкретных реализаций к абстракциям

Эти паттерны предоставляют готовые архитектурные решения для создания систем, которые легко расширять без модификации существующего, стабильного кода.

</details>

<br>

<details>
    <summary>
        <b>Liskov Substitution Principle (Принцип подстановки Барбары Лисков)</b>
    </summary>

> **«Объекты в программе должны быть заменимы экземплярами их подтипов без изменения корректности программы»** - Барбара Лисков (1987)

Если класс `B` наследуется от класса `A`, то **везде, где ожидается `A`, можно безопасно подставить `B`**, и программа продолжит работать так, как задумано - без ошибок, исключений или изменения логики.

# Ключевые правила LSP

## Контракты методов

- ### Предусловия (требования до выполнения метода) не могут быть усилены в подклассе

  - Предусловие - это требование, которое **должно быть истинным до вызова метода** (например, «аргумент не null», «значение положительное»).
  
    Суперкласс может требовать: «`amount > 0`».
    
    Подкласс **не имеет права** потребовать: «`amount > 100`» - это **усиление** предусловия.
    
    Подкласс **может ослабить** или убрать требование (например, разрешить `amount >= 0`), но **никогда не ужесточать**.


- ### Постусловия (гарантии после выполнения метода) не могут быть ослаблены в подклассе

  - Постусловие - это гарантия, которая становится истинной после выполнения метода (например, «баланс уменьшится на amount», «возвращённое значение не null»).

    Суперкласс гарантирует: «после save() объект будет в БД».
  
    Подкласс не может отменить эту гарантию.

    Подкласс может усилить постусловие (например, «и будет отправлено уведомление»), но не ослабить.


- ### Инварианты базового класса должны сохраняться в подклассе

  - Инвариант - это свойство объекта, которое всегда истинно между вызовами публичных методов (например, «баланс ≥ 0», «список отсортирован»).

    Если в Rectangle инвариант - «ширина и высота ≥ 0», то Square не может нарушать это.
    
    Все методы подкласса (включая переопределённые) должны поддерживать инварианты родителя.

## Поведенческие требования

- ### Подкласс не должен генерировать новые исключения, которые не генерируются базовым классом
  
  - Если суперкласс никогда не бросает `IOException`, подкласс не может начать его бросать в переопределённом методе.
  
    Подкласс может бросать подтипы уже объявленных исключений (если они есть в throws), но не новые типы. 

  
- ### Подкласс не должен иметь более строгих условий, чем базовый класс

  - Это обобщение предусловий: включает не только аргументы, но и состояние объекта.

    Если суперкласс позволяет вызывать `start()` в любом состоянии, подкласс не может требовать, чтобы сначала был вызван `init()`.


- Возвращаемые значения должны быть совместимыми по типу и семантике

  - По типу: в Java поддерживается ковариантность — подкласс может возвращать подтип того, что возвращает суперкласс.
  
    По семантике: значение должно соответствовать ожиданиям, заданным контрактом.

    Пример: Если в суперкласе `public User findById(Long id)` должен вернуть активного пользователя, то в подклассе нельзя возвращать заглушку или удалённого пользователя.
  
    Хотя тип возвращаемого значения (User) совпадает, семантический контракт («пользователь существует и активен») нарушен.

Нарушение LSP приводит к **непредсказуемому поведению**, особенно в коде, который работает с полиморфными ссылками.

### ❌ Классический пример: Прямоугольник и Квадрат

```java
// Базовый класс
class Rectangle {
    
    private int width;
    private int height;

    public void setWidth(int width) {
        this.width = width;
    }

    public void setHeight(int height) {
        this.height = height;
    }

    public int getArea() {
        return width * height;
    }
}

// Подкласс, нарушающий LSP
class Square extends Rectangle {
    
    @Override
    public void setWidth(int width) {
        super.setWidth(width);
        super.setHeight(width); // Нарушение поведения!
    }

    @Override
    public void setHeight(int height) {
        super.setHeight(height);
        super.setWidth(height); // Нарушение поведения!
    }
}

// Клиентский код, который ломается при подстановке
public class GeometryTest {
    
    public void testRectangle(Rectangle rectangle) {
        rectangle.setWidth(5);
        rectangle.setHeight(4);
        assert rectangle.getArea() == 20; // Для Square будет 16!
    }

    public static void main(String[] args) {
        GeometryTest test = new GeometryTest();
        test.testRectangle(new Rectangle()); // OK
        test.testRectangle(new Square());    // FAIL! Нарушение LSP
    }
}
```

Проблема: Квадрат изменяет ожидаемое поведение методов setWidth и setHeight.

### ❌ Пример с исключениями

```java
class Bird {
    
    public void fly() {
        System.out.println("Flying...");
    }
}

class Penguin extends Bird {
    
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}

// Клиентский код
public class BirdService {
    
    public void makeBirdFly(Bird bird) {
        bird.fly(); // Для Penguin выбросит исключение!
    }
}
```

Проблема: Подкласс вводит новое исключение, которое не ожидается клиентским кодом.

### ❌ Пример с ослаблением постусловий

```java
class Database {
    
    public Connection connect(String url) {
        // Гарантирует, что возвращается НЕ-null соединение
        Connection conn = createConnection(url);
        if (conn == null) {
            throw new ConnectionException("Failed to connect");
        }
        return conn;
    }
}

class MockDatabase extends Database {
    
    @Override
    public Connection connect(String url) {
        // Нарушение: может вернуть null, ослабление постусловия
        return null;
    }
}
```

### ✅ Правильные реализации с соблюдением LSP

Решение для фигур через интерфейсы
```java
// Вместо наследования используем интерфейсы
interface Shape {
    int getArea();
}

class Rectangle implements Shape {
    
    private int width;
    private int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    public void setWidth(int width) { this.width = width; }
    
    public void setHeight(int height) { this.height = height; }
    
    @Override
    public int getArea() {
        return width * height;
    }
}

class Square implements Shape {
    
    private int size;
    
    public Square(int size) {
        this.size = size;
    }
    
    public void setSize(int size) { this.size = size; }
    
    @Override
    public int getArea() {
        return size * size;
    }
}

// Клиентский код теперь работает корректно
public class GeometryService {
    
    public void printArea(Shape shape) {
        System.out.println("Area: " + shape.getArea());
    }
}
```

Решение для птиц через разделение ответственности

```java
interface Bird {
    
    void eat();
    void sleep();
}

interface FlyingBird extends Bird {
    
    void fly();
}

interface SwimmingBird extends Bird {
    
    void swim();
}

class Sparrow implements FlyingBird {
    
    public void eat() { /* ... */ }
    public void sleep() { /* ... */ }
    public void fly() { /* ... */ }
}

class Penguin implements SwimmingBird {
    
    public void eat() { /* ... */ }
    public void sleep() { /* ... */ }
    public void swim() { /* ... */ }
}

// Специализированные сервисы
class FlightService {
    
    public void makeFly(FlyingBird bird) {
        bird.fly(); // Гарантированно безопасно
    }
}

class SwimService {
    
    public void makeSwim(SwimmingBird bird) {
        bird.swim(); // Гарантированно безопасно
    }
}

// КОМПИЛЯЦИЯ УСПЕШНА
FlightService flightService = new FlightService();
flightService.makeFly(new Sparrow()); // Sparrow implements FlyingBird

// КОМПИЛЯЦИЯ ПРОВАЛИВАЕТСЯ
        flightService.makeFly(new Penguin()); // Ошибка: Penguin НЕ implements FlyingBird
```

### Признаки нарушения LSP

- Подкласс выбрасывает новые исключения
- Подкласс возвращает несовместимые значения
- Подкласс требует более строгих условий
- Подкласс ослабляет инварианты базового класса
- Клиентский код вынужден проверять конкретный тип объекта
- Подкласс не поддерживает все методы базового класса

### Практические рекомендации

- Тестируйте подстановку - пишите тесты, которые работают с базовым классом и проверяют работу с подклассами
- Избегайте наследования "is-a" в пользу "behaves-like-a" - наследуйтесь не потому что объект "является" чем-то, а потому что он "ведет себя как"
- Сомневаетесь - используйте композицию - когда не уверены в иерархии наследования, предпочитайте композицию
- Следите за исключениями - подклассы не должны вводить новые типы исключений
- Сохраняйте инварианты - если базовый класс гарантирует определенные состояния, подклассы должны их сохранять

### Ключевой вывод

`LSP` - это не просто о синтаксической совместимости, а о семантической эквивалентности. 
Наследование должно означать не просто "является разновидностью", а "может использоваться везде, где используется базовый класс, без изменения ожидаемого поведения".

Золотое правило: Если вы сомневаетесь, будет ли подкласс корректно работать вместо базового, представьте, что клиентский код написан против базового класса и не знает о существовании подклассов. 
Будет ли он работать корректно? Если нет - LSP нарушен.
</details>

<br>

<details>
    <summary>
        <b>Interface Segregation Principle (Принцип разделения интерфейсов)</b>
    </summary>

> **«Клиенты не должны зависеть от интерфейсов, которые они не используют»** - Роберт Мартин

Много специализированных интерфейсов лучше, чем один универсальный.

Интерфейсы должны быть **маленькими, узкоспециализированными и ориентированными на клиента**. 
Вместо одного «жирного» интерфейса с множеством методов лучше создать несколько **специфичных интерфейсов**, каждый из которых решает одну задачу.

### Проблема "толстого" интерфейса:

- Классы вынуждены реализовывать методы, которые им не нужны
- Появляются "пустые" реализации или методы, выбрасывающие исключения
- Изменения в интерфейсе затрагивают всех клиентов, даже тех, кто не использует изменяемый метод
- Увеличивается связность между не связанными частями системы

### Цель ISP:

- Разделить интерфейсы на более мелкие и специфичные
- Каждый интерфейс решает одну конкретную задачу
- Клиенты зависят только от того, что им действительно нужно

ISP особенно важен в языках с **обязательной реализацией всех методов интерфейса** (как Java до default-методов).

Java 8+ позволяет добавлять default-методы в интерфейсы, но это не отменяет ISP:

default-методы удобны для общей реализации, но если метод не нужен клиенту — лучше вынести его в отдельный интерфейс.

Чрезмерное использование default превращает интерфейс обратно в «жирный».

### ❌ "Божественный" интерфейс для всех устройств

```java
// ПЛОХО: Один интерфейс на все случаи жизни
interface MultiFunctionDevice {
    
    // Методы для печати
    void print(Document document);
    void scan(Document document);
    void fax(Document document);
    
    // Методы для работы с email
    void sendEmail(Email email);
    void receiveEmail();
    
    // Методы для работы с сетью
    void connectToInternet();
    void disconnectFromInternet();
}

// Реализации вынуждены делать то, что не умеют
class SimplePrinter implements MultiFunctionDevice {
    
    public void print(Document document) {
        // OK - принтер умеет печатать
    }
    
    public void scan(Document document) {
        throw new UnsupportedOperationException("This printer cannot scan!");
    }
    
    public void fax(Document document) {
        throw new UnsupportedOperationException("This printer cannot fax!");
    }
    
    public void sendEmail(Email email) {
        // Пустая реализация или исключение
    }
    
    // ... и так далее для всех методов
}

class BasicScanner implements MultiFunctionDevice {
    
    public void scan(Document document) {
        // OK - сканер умеет сканировать
    }
    
    public void print(Document document) {
        throw new UnsupportedOperationException("This scanner cannot print!");
    }
    
    // ... все остальные методы тоже бросают исключения
}
```

Проблемы:
- Классы реализуют методы, которые никогда не будут использоваться
- Нарушение принципа единственной ответственности
- Сложность тестирования
- Хрупкость системы - изменение интерфейса ломает всех
    
### ✅ Правильное решение с соблюдением ISP

Разделяем на специализированные интерфейсы
```java
// Мелкие, сфокусированные интерфейсы

interface Printer {
    void print(Document document);
}

interface Scanner {
    void scan(Document document);
}

interface FaxMachine {
    void fax(Document document);
}

interface EmailClient {
    void sendEmail(Email email);
    void receiveEmail();
}

interface NetworkDevice {
    void connectToInternet();
    void disconnectFromInternet();
}
```
Гибкие реализации устройств:
```java
// Простой принтер - только печать
class SimplePrinter implements Printer {
    
    public void print(Document document) {
        System.out.println("Printing document: " + document.getName());
    }
}

// Сканер - только сканирование
class BasicScanner implements Scanner {
    
    public void scan(Document document) {
        System.out.println("Scanning document: " + document.getName());
    }
}

// МФУ - комбинирует несколько интерфейсов
class MultiFunctionMachine implements Printer, Scanner, FaxMachine {
    
    public void print(Document document) {
        System.out.println("Printing: " + document.getName());
    }
    
    public void scan(Document document) {
        System.out.println("Scanning: " + document.getName());
    }
    
    public void fax(Document document) {
        System.out.println("Faxing: " + document.getName());
    }
}

// Умное устройство - все возможности
class SmartOfficeDevice implements Printer, Scanner, FaxMachine, EmailClient, NetworkDevice {
    // Реализация всех методов...
}
```

Специализированные сервисы для клиентов

```java
// Сервис печати работает только с принтерами
class PrintService {
    
    public void printDocument(Printer printer, Document document) {
        printer.print(document); // Гарантированно безопасно
    }
}

// Сервис сканирования работает только со сканерами
class ScanService {
    
    public void scanDocument(Scanner scanner, Document document) {
        scanner.scan(document); // Гарантированно безопасно
    }
}

// Сервис для "умных" устройств
class SmartDeviceService {
    
    public void processDocument(MultiFunctionMachine machine, Document document) {
        machine.scan(document);
        machine.print(document);
        machine.fax(document);
    }
}
```

### ❌ Нарушение ISP

```java
interface PaymentProcessor {
    
    void processCreditCard(PaymentData data);
    void processPayPal(PaymentData data);
    void processCrypto(PaymentData data);
    void processBankTransfer(PaymentData data);
    void processLoyaltyPoints(PaymentData data);
    void refundPayment(String transactionId);
    void generateInvoice(Order order);
    void sendReceipt(Customer customer);
}

class CreditCardProcessor implements PaymentProcessor {
    // Вынужден реализовывать все методы, хотя нужны только 2-3
}
```

### ✅ Соблюдение ISP

```java
// Базовый интерфейс для всех платежей
interface PaymentProcessor {
    
    void processPayment(PaymentData data);
    void refundPayment(String transactionId);
}

// Специализированные интерфейсы для разных способов оплаты
interface CreditCardPayment {
    void processCreditCard(CreditCardData data);
}

interface DigitalWalletPayment {
    void processDigitalWallet(WalletData data);
}

interface CryptocurrencyPayment {
    void processCrypto(CryptoData data);
}

// Интерфейсы для дополнительных сервисов
interface InvoiceGenerator {
    void generateInvoice(Order order);
}

interface ReceiptSender {
    void sendReceipt(Customer customer);
}

// Гибкие реализации
class BasicCreditCardProcessor implements PaymentProcessor, CreditCardPayment {
    public void processPayment(PaymentData data) {
        // Базовая обработка
    }
    
    public void refundPayment(String transactionId) {
        // Возврат средств
    }
    
    public void processCreditCard(CreditCardData data) {
        // Специфичная обработка карт
    }
}

class FullServiceProcessor implements PaymentProcessor, CreditCardPayment, 
                                      DigitalWalletPayment, InvoiceGenerator, 
                                      ReceiptSender {
    // Реализация всех возможностей
}
```

## Признаки нарушения ISP

- Пустые реализации методов - класс реализует метод "заглушкой"
- Методы, выбрасывающие UnsupportedOperationException
- Классы зависят от методов, которые не используют
- Интерфейсы требуют знание о слишком многих аспектах системы
- Изменение в интерфейсе вызывает каскад изменений в не связанных классах

## Преимущества соблюдения ISP

- ### Для разработки:

  - Упрощение тестирования - мокировать маленькие интерфейсы проще

  - Улучшение читаемости - интерфейсы ясно выражают намерения

  - Снижение связности - компоненты зависят только от необходимого

- ### Для архитектуры:

  - Гибкость - легко комбинировать поведения

  - Устойчивость к изменениям - изменения затрагивают меньше кода

  - Переиспользование - мелкие интерфейсы легче использовать в разных контекстах

- ### Для командной работы:

  - Параллельная разработка - разные разработчики могут работать над разными интерфейсами

  - Четкие контракты - каждый интерфейс определяет ясную зону ответственности

## Связь с другими принципами SOLID

### SRP (Единая ответственность)
ISP является естественным продолжением SRP на уровне интерфейсов. 
Если SRP говорит "класс должен иметь одну причину для изменения", то ISP говорит "интерфейс должен иметь одного клиента".

### LSP (Подстановка Лисков)
Мелкие интерфейсы легче правильно реализовать без нарушения контрактов.

### DIP (Инверсия зависимостей)
Легче зависеть от абстракций, когда эти абстракции маленькие и сфокусированные.


## Практические рекомендации

- Декомпозируйте по клиентам - создавайте интерфейсы для конкретных клиентов, а не для всей системы
- Избегайте "мульти функциональных интерфейсов" - если интерфейс делает "всё", скорее всего его нужно разделить
- Используйте сегрегацию на ранних этапах - легче разделить сразу, чем рефакторить потом
- Следите за "заглушками" - пустые реализации и исключения (типа UnsupportedOperationException) - явный признак нарушения ISP
- Ориентируйтесь на использование - проектируйте интерфейсы исходя из того, как они будут использоваться, а не из того, что "может понадобиться"

## Ключевой вывод

ISP учит нас, что интерфейсы должны быть сфокусированными и клиенто-ориентированными. 

Вместо создания универсальных "монолитных" интерфейсов, мы создаем множество мелких, специализированных интерфейсов, каждый из которых решает конкретную задачу для конкретного клиента.

Если вы видите, что класс реализует методы интерфейса, которые никогда не будут вызваны в данном контексте - интерфейс слишком велик и его нужно разделить.
</details>

<br>

<details>
    <summary>
        <b>Dependency Inversion Principle (Принцип инверсии зависимостей)</b>
    </summary>

> **«Модули верхних уровней не должны зависеть от модулей нижних уровней. Оба должны зависеть от абстракций.  
> Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций»** - Роберт Мартин

### Глубокое понимание

Традиционный подход (проблема):

```text
Верхний уровень → Нижний уровень → Детали реализации
```

Инверсия зависимостей (решение):

```text
Верхний уровень → Абстракции ← Нижний уровень
                      ↑
               Детали реализации
```

### Ключевые концепции:

- Модули верхнего уровня - содержат бизнес-логику, политики приложения
- Модули нижнего уровня - содержат реализации, технические детали
- Абстракции - интерфейсы, абстрактные классы, которые определяют контракт

### ❌ Прямая зависимость от деталей

```java
// Модуль нижнего уровня (деталь)
class MySQLDatabase {
    
    public void connect() {
        System.out.println("Connecting to MySQL database...");
    }
    
    public void save(User user) {
        System.out.println("Saving user to MySQL: " + user.getName());
    }
    
    public User findById(int id) {
        System.out.println("Finding user in MySQL by id: " + id);
        return new User("John Doe");
    }
}

// Модуль верхнего уровня (бизнес-логика)
class UserService {
    
    private MySQLDatabase database;  // ПРЯМАЯ зависимость от детали!
    
    public UserService() {
        this.database = new MySQLDatabase();  // Жесткая связь
        this.database.connect();
    }
    
    public void createUser(String name) {
        User user = new User(name);
        database.save(user);  // Зависимость от конкретной реализации
    }
    
    public User getUser(int id) {
        return database.findById(id);
    }
}

// Использование
public class Application {
    
    public static void main(String[] args) {
        UserService userService = new UserService();  // Жестко привязан к MySQL
        userService.createUser("Alice");
    }
}
```

Проблемы:

- Невозможно заменить БД без изменения кода UserService
- Сложно тестировать (нужна реальная БД)
- Нарушение принципа открытости/закрытости
- Высокая связность

### ✅ Правильное решение с соблюдением DIP

Создаем абстракции

```java
// Абстракция для репозитория
interface UserRepository {
    
    void save(User user);
    User findById(int id);
    List<User> findAll();
}

// Абстракция для сервиса уведомлений
interface EmailService {
    
    void sendEmail(String to, String subject, String body);
}

// Абстракция для логгера
interface Logger {
    
    void log(String message);
}
```

Реализуем детали (нижний уровень)

```java
// Конкретная реализация репозитория
class MySQLUserRepository implements UserRepository {
    
    public void save(User user) {
        System.out.println("Saving user to MySQL: " + user.getName());
    }
    
    public User findById(int id) {
        System.out.println("Finding user in MySQL by id: " + id);
        return new User("John Doe");
    }
    
    public List<User> findAll() {
        return Arrays.asList(new User("John Doe"));
    }
}

class PostgreSQLUserRepository implements UserRepository {
    
    public void save(User user) {
        System.out.println("Saving user to PostgreSQL: " + user.getName());
    }
    
    public User findById(int id) {
        System.out.println("Finding user in PostgreSQL by id: " + id);
        return new User("Jane Smith");
    }
    
    public List<User> findAll() {
        return Arrays.asList(new User("Jane Smith"));
    }
}

// Конкретная реализация email сервиса
class SMTPEmailService implements EmailService {
    
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending email via SMTP to: " + to);
    }
}

class SendGridEmailService implements EmailService {
    
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending email via SendGrid to: " + to);
    }
}

// Конкретная реализация логгера
class ConsoleLogger implements Logger {
    
    public void log(String message) {
        System.out.println("LOG: " + message);
    }
}

class FileLogger implements Logger {
    
    public void log(String message) {
        System.out.println("Writing to file: " + message);
    }
}
```

Строим бизнес-логику (верхний уровень)

```java
// Бизнес-логика зависит только от абстракций
class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final Logger logger;
    
    // Зависимости внедряются через конструктор
    public UserService(UserRepository repository, EmailService emailService, Logger logger) {
        this.userRepository = repository;
        this.emailService = emailService;
        this.logger = logger;
    }
    
    public void registerUser(String name, String email) {
        logger.log("Registering user: " + name);
        
        User user = new User(name);
        userRepository.save(user);
        
        emailService.sendEmail(email, "Welcome!", "Hello " + name + ", welcome!");
        
        logger.log("User registered successfully: " + name);
    }
    
    public User getUser(int id) {
        return userRepository.findById(id);
    }
}
```
---

### Конфигурация и компоновка

Способ 1: Ручная компоновка

```java
public class ApplicationConfig {
    
    public static UserService userService() {
        UserRepository repository = new PostgreSQLUserRepository();
        EmailService emailService = new SendGridEmailService();
        Logger logger = new FileLogger();
        
        return new UserService(repository, emailService, logger);
    }
    
    public static UserService testUserService() {
        // Для тестов используем другие реализации
        UserRepository repository = new InMemoryUserRepository();
        EmailService emailService = new MockEmailService();
        Logger logger = new ConsoleLogger();
        
        return new UserService(repository, emailService, logger);
    }
}
```

Способ 2: Использование Spring Framework

```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserRepository userRepository() {
        return new PostgreSQLUserRepository();
    }
    
    @Bean
    public EmailService emailService() {
        return new SendGridEmailService();
    }
    
    @Bean
    public Logger logger() {
        return new FileLogger();
    }
    
    @Bean
    public UserService userService(UserRepository repository, EmailService emailService, Logger logger) {
        return new UserService(repository, emailService, logger);
    }
}

// Бизнес-логика с аннотациями
@Service
class OrderService {
    
    private final UserRepository userRepository;
    private final Logger logger;
    
    @Autowired
    public OrderService(UserRepository userRepository, Logger logger) {
        this.userRepository = userRepository;
        this.logger = logger;
    }
}
```
---

## Признаки нарушения DIP

- Ключевое слово new для сервисов в конструкторах
- Статические вызовы сервисов (Database.connect(), Logger.log())
- Зависимость от конкретных классов в импортах
- Сложность тестирования без реальных сервисов
- Прямые вызовы API внешних систем

## Преимущества соблюдения DIP

- ### Гибкость и поддерживаемость:

  - Легкая замена реализаций без изменения бизнес-логики 
  - Простое тестирование через mock-объекты
  - Снижение связанности между компонентами

- ### Архитектурные benefits:

  - Возможность отложенных решений - можно начать с простых реализаций
  - Простота миграции между технологиями
  - Четкое разделение ответственности

- ### Бизнес-преимущества:

  - Быстрее время выхода на рынок - можно параллельно разрабатывать модули
  - Снижение рисков - изоляция изменений
  - Упрощение онбординга новых разработчиков

## Связь с другими принципами

### DIP + OCP

DIP делает соблюдение OCP возможным - мы можем расширять систему, добавляя новые реализации абстракций.

### DIP + ISP

Мелкие, сфокусированные интерфейсы (ISP) идеально подходят для инверсии зависимостей.

### DIP + Patterns

- Strategy - позволяет менять алгоритмы через абстракции

- Adapter - адаптирует несовместимые интерфейсы к нашим абстракциям

- Factory - создает объекты, следуя абстракциям

## Практические рекомендации

- Зависите от интерфейсов, а не от классов
- Внедряйте зависимости через конструктор
- Выносите создание объектов в компоновочный корень
- Избегайте статических связей с сервисами
- Создавайте интерфейсы на стороне клиента - интерфейс должен отражать потребности клиента, а не возможности сервиса

## Ключевой вывод
    
DIP переворачивает традиционное направление зависимостей, делая систему гибкой и устойчивой к изменениям. 

Бизнес-логика (что делает система) становится независимой от технических деталей (как система это делает).

Фундаментальная идея: Самые ценные части вашей системы - бизнес-правила и политики - не должны зависеть от менее ценных частей, таких как базы данных, фреймворки или внешние API.
</details>