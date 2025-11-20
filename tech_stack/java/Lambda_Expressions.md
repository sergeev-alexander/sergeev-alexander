## Lambda Expressions

<details>
    <summary>
        <b>Проблема, которую решают лямбды</b>
    </summary>

### До Java 8: Анонимные классы (когда блок кода нужно передать как параметр)

```java
// Пример: сортировка списка строк по длине ДО Java 8

List<String> names = Arrays.asList("John", "Alice", "Bob", "Chris");

// Используем анонимный класс Comparator
Collections.

sort(names, new Comparator<String>() {
    @Override
    public int compare (String s1, String s2){
        return Integer.compare(s1.length(), s2.length());
    }
});

        System.out.

println(names); // [Bob, John, Alice, Chris]
```

Проблемы анонимных классов:

- Много boilerplate кода - только одна строка логики, но 5 строк обертки
- Сложно читать - много синтаксического шума
- Неэффективно - каждый анонимный класс создает новый .class файл

### Решение: Lambda Expressions

```java
// Тот же пример с лямбда-выражением

List<String> names = Arrays.asList("John", "Alice", "Bob", "Chris");

// Лямбда-выражение вместо анонимного класса
Collections.

sort(names, (s1, s2) ->Integer.

compare(s1.length(),s2.

length()));

        System.out.

println(names); // [Bob, John, Alice, Chris]
```

**Лямбда** - это анонимная функция (функция без имени), которую можно:

- ✅ Сохранять в переменной
- ✅ Передавать как параметр
- ✅ Возвращать из метода

### Базовый синтаксис

```java
(parameters)->

expression

// ИЛИ        

(parameters) ->{statements; }
```

Простые примеры

```java
// 1. Без параметров
Runnable task = () -> System.out.println("Hello Lambda!");

// 2. Один параметр (скобки можно опустить)
Consumer<String> printer = message -> System.out.println(message);

// 3. Несколько параметров
Comparator<Integer> comparator = (a, b) -> a.compareTo(b);

// 4. Многострочное тело
Runnable complexTask = () -> {
    System.out.println("Starting task...");
    System.out.println("Task completed!");
};
```

</details>

<details>
    <summary>
        <b>Базовые формы синтаксиса</b>
    </summary>

### Четыре основные формы лямбд

```java
// 1. Без параметров
()->expression

// 2. Один параметр (можно без скобок)
parameter ->

expression

// 3. Несколько параметров  
(parameter1, parameter2) ->

expression

// 4. Многострочное тело
(parameters) ->{
statement1;
statement2;
    return result;
}
```

Без параметров

```java
// Функциональный интерфейс
@FunctionalInterface
interface Generator {

    int generate();
}

public class NoParameters {

    public static void main(String[] args) {
        // Лямбда без параметров - скобки ОБЯЗАТЕЛЬНЫ
        Generator randomGenerator = () -> (int) (Math.random() * 100);
        Generator constantGenerator = () -> 42;

        System.out.println("Случайное: " + randomGenerator.generate()); // например, 73
        System.out.println("Константа: " + constantGenerator.generate()); // 42

        // Многострочное тело
        Runnable complexTask = () -> {
            System.out.println("Начинаем выполнение...");
            System.out.println("Выполняем работу...");
            System.out.println("Завершаем выполнение.");
        };

        complexTask.run();
    }
}
```

Один параметр

```java
// Функциональный интерфейс
@FunctionalInterface
interface Transformer {

    String transform(String input);
}

@FunctionalInterface
interface Validator {

    boolean isValid(String value);
}

public class SingleParameter {

    public static void main(String[] args) {
        // Один параметр - скобки НЕОБЯЗАТЕЛЬНЫ
        Transformer toUpper = str -> str.toUpperCase();
        Transformer addPrefix = s -> "PRE_" + s;

        // Но можно и со скобками (для consistency)
        Transformer toLower = (str) -> str.toLowerCase();

        System.out.println(toUpper.transform("hello")); // HELLO
        System.out.println(addPrefix.transform("test")); // PRE_test

        // Булевые проверки
        Validator isNotEmpty = s -> s != null && !s.trim().isEmpty();
        Validator isEmail = email -> email != null && email.contains("@");

        System.out.println(isNotEmpty.isValid("  ")); // false
        System.out.println(isEmail.isValid("test@mail.com")); // true
    }
}
```

Несколько параметров

```java
// Функциональные интерфейсы
@FunctionalInterface
interface Calculator {

    int calculate(int a, int b);
}

@FunctionalInterface
interface StringCombiner {

    String combine(String s1, String s2, String s3);
}

public class MultipleParameters {
    public static void main(String[] args) {

        // Два параметра - скобки ОБЯЗАТЕЛЬНЫ
        Calculator adder = (a, b) -> a + b;
        Calculator multiplier = (x, y) -> x * y;

        System.out.println("Сумма: " + adder.calculate(5, 3)); // 8
        System.out.println("Произведение: " + multiplier.calculate(5, 3)); // 15

        // Три параметра
        StringCombiner concatenator = (s1, s2, s3) -> s1 + " " + s2 + " " + s3;
        System.out.println(concatenator.combine("Hello", "from", "Java")); // Hello from Java

        // С явным указанием типов (редко нужно)
        Calculator typedAdder = (int a, int b) -> a + b;
    }
}
```

Многострочное тело

```java

@FunctionalInterface
interface ComplexProcessor {

    String process(String input);
}

@FunctionalInterface
interface NumberAnalyzer {

    String analyze(int number);
}

public class MultiLineBody {

    public static void main(String[] args) {
        // Многострочное тело с return
        ComplexProcessor processor = input -> {
            if (input == null) {
                return "NULL_INPUT";
            }
            String trimmed = input.trim();
            String result = trimmed.toUpperCase();
            return "RESULT: " + result;
        };

        System.out.println(processor.process("  hello world  ")); // RESULT: HELLO WORLD
        System.out.println(processor.process(null)); // NULL_INPUT

        // Сложная логика анализа числа
        NumberAnalyzer analyzer = number -> {
            StringBuilder analysis = new StringBuilder();
            analysis.append("Число: ").append(number);

            if (number % 2 == 0) {
                analysis.append(" (четное)");
            } else {
                analysis.append(" (нечетное)");
            }

            if (number > 0) {
                analysis.append(", положительное");
            } else if (number < 0) {
                analysis.append(", отрицательное");
            } else {
                analysis.append(", ноль");
            }

            return analysis.toString();
        };

        System.out.println(analyzer.analyze(10)); // Число: 10 (четное), положительное
        System.out.println(analyzer.analyze(-5)); // Число: -5 (нечетное), отрицательное
    }
}
```

### Указание типов параметров

```java

@FunctionalInterface
interface NumberProcessor {
    int process(int number);
}

@FunctionalInterface
interface TwoNumberProcessor {
    int process(int a, int b);
}

@FunctionalInterface
interface ComplexProcessor {
    String process(String prefix, int number, boolean uppercase);
}

public class AllVariations {
    public static void main(String[] args) {
        // ✅ 1 параметр без типа, без скобок
        NumberProcessor square = x -> x * x;

        // ✅ 1 параметр без типа, со скобками
        NumberProcessor cube = (x) -> x * x * x;

        // ✅ 1 параметр с типом (скобки ОБЯЗАТЕЛЬНЫ)
        NumberProcessor negate = (int x) -> -x;

        // ✅ 2 параметра без типов
        TwoNumberProcessor adder = (a, b) -> a + b;

        // ✅ 2 параметра с типами
        TwoNumberProcessor multiplier = (int a, int b) -> a * b;

        // ✅ 3 параметра с типами
        ComplexProcessor formatter = (String prefix, int number, boolean uppercase) -> {
            String result = prefix + number;
            return uppercase ? result.toUpperCase() : result;
        };

        System.out.println(square.process(5));      // 25
        System.out.println(cube.process(3));        // 27
        System.out.println(negate.process(10));     // -10
        System.out.println(adder.process(7, 3));    // 10
        System.out.println(multiplier.process(7, 3)); // 21
        System.out.println(formatter.process("Num: ", 42, true)); // NUM: 42
    }
}
```

```java

@FunctionalInterface
interface MultiParam {

    String process(String a, String b);
}

public class WhenParenthesesRequired {

    public static void main(String[] args) {

        // ❌ Один параметр без типа - скобки НЕ обязательны
        Greeter g1 = name -> ... ✅РАБОТАЕТ

        // ❌ Один параметр с типом - скобки ОБЯЗАТЕЛЬНЫ
        Greeter g2 = String name -> ... ❌ОШИБКА !
                Greeter g3 = (String name) -> ... ✅РАБОТАЕТ

        // ❌ Два+ параметра - скобки ОБЯЗАТЕЛЬНЫ всегда
        MultiParam m1 = a, b -> ... ❌ОШИБКА !
                MultiParam m2 = (a, b) -> a + b; ✅РАБОТАЕТ
        MultiParam m3 = (String a, String b) -> a + b; ✅РАБОТАЕТ

        // ❌ Ноль параметров - скобки ОБЯЗАТЕЛЬНЫ
        Runnable r1 = () -> System.out.println("Hello"); ✅РАБОТАЕТ
        Runnable r2 = ->System.out.println("Hello"); ❌ОШИБКА !
    }
}
```

### Лямбды с исключениями

```java

@FunctionalInterface
interface FileProcessor {

    void process(String filename) throws IOException;
}

public class ExceptionHandling {

    public static void main(String[] args) {
        // Лямбда может бросать исключения как обычный метод
        FileProcessor fileProcessor = filename -> {
            // Симуляция работы с файлом
            if (filename == null) {
                throw new IOException("Filename cannot be null");
            }
            System.out.println("Обрабатываем файл: " + filename);
        };

        try {
            fileProcessor.process("data.txt");
            fileProcessor.process(null); // выбросит исключение
        } catch (IOException e) {
            System.out.println("Ошибка: " + e.getMessage());
        }
    }
}
```

</details>

<details>
    <summary>
        <b>Лямбды + Функциональные интерфейсы</b>
    </summary>

## Функциональные интерфейсы

Итерфейс с ровно одним абстрактным методом (SAM - Single Abstract Method)

```java
// Классический пример - Runnable
@FunctionalInterface
public interface Runnable {

    void run(); // единственный абстрактный метод
}

// Еще пример - Comparator
@FunctionalInterface
public interface Comparator<T> {

    int compare(T o1, T o2); // единственный абстрактный метод

    default Comparator<T> reversed() { // default методы НЕ считаются!
        return Collections.reverseOrder(this);
    }
}
```

@FunctionalInterface

```java

@FunctionalInterface
interface MyFunctionalInterface {

    void execute(); // единственный абстрактный метод

    // Можно иметь default методы
    default void helper() {
        System.out.println("Helper method");
    }

    // Можно иметь static методы  
    static void utility() {
        System.out.println("Utility method");
    }
}
```

Что дает `@FunctionalInterface`:

- Компилятор проверяет, что интерфейс действительно функциональный
- Явно указывает на предназначение интерфейса
- Защищает от случайного добавления второго абстрактного метода

## Лямбды + Функциональные интерфейсы

```java
// Функциональный интерфейс
@FunctionalInterface
interface StringProcessor {

    String process(String input);
}

public class Lambda {
    public static void main(String[] args) {

        // Лямбда-выражение АВТОМАТИЧЕСКИ реализует функциональный интерфейс
        StringProcessor toUpper = str -> str.toUpperCase();
        StringProcessor toLower = str -> str.toLowerCase();
        StringProcessor addExclamation = str -> str + "!";

        // Использование
        System.out.println(toUpper.process("hello")); // HELLO
        System.out.println(toLower.process("WORLD")); // world
        System.out.println(addExclamation.process("Wow")); // Wow!
    }
}
```

Как это работает под капотом?

```java
// Компилятор видит это:
StringProcessor processor = (str) -> str.toUpperCase();

// И преобразует в нечто похожее на:
StringProcessor processor = new StringProcessor() {
    @Override
    public String process(String str) {
        return str.toUpperCase();
    }
};
```

Но есть важные отличия:

- Лямбды не создают новые .class файлы
- Более эффективное использование памяти
- Лучшая производительность

Практический пример: Обработка пользователей

```java
import java.util.*;

// Функциональный интерфейс для фильтрации
@FunctionalInterface
interface UserFilter {
    boolean test(User user);
}

class User {
    private String name;
    private int age;
    private boolean active;

    public User(String name, int age, boolean active) {
        this.name = name;
        this.age = age;
        this.active = active;
    }

    // геттеры
}

public class UserProcessor {

    // Метод принимает функциональный интерфейс как параметр
    public static List<User> filterUsers(List<User> users, UserFilter filter) {
        List<User> result = new ArrayList<>();
        for (User user : users) {
            if (filter.test(user)) {
                result.add(user);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<User> users = Arrays.asList(
                new User("Alice", 25, true),
                new User("Bob", 17, true),
                new User("Charlie", 30, false),
                new User("Diana", 22, true));

        // Разные фильтры через лямбды
        List<User> adults = filterUsers(users, user -> user.getAge() >= 18);
        List<User> activeUsers = filterUsers(users, user -> user.isActive());
        List<User> activeAdults = filterUsers(users, user -> user.getAge() >= 18 && user.isActive());
    }
}
```

## Функциональные интерфейсы из java.util.function

### `Predicate<T>`: Проверка условия (утверждение)

- Абстрактный метод: `boolean test(T t)`
- Назначение: Принимает объект типа T и возвращает true или false.
- Типичное использование: Фильтрация, проверка условий.
- Полезные методы по умолчанию (default): `and`, `or`, `negate` (не), `isEqual`.
  <br>
  `isEqual` — это статический метод, а не метод по умолчанию. Он возвращает Predicate.

```java
// Один предикат
Predicate<String> isLongerThan5 = str -> str.length() > 5;
        
System.out.

println(isLongerThan5.test("Hello"));   // false
        System.out.

println(isLongerThan5.test("Hello World")); // true

// Композиция предикатов
Predicate<String> isShorterThan10 = str -> str.length() < 10;
Predicate<String> between5And10 = isLongerThan5.and(isShorterThan10);

System.out.

println(between5And10.test("Hello")); // false (длина 5)
        System.out.

println(between5And10.test("Network")); // true (длина 7)
```

```java
public class PredicateExample {
    public static void main(String[] args) {

        // Пример: числа больше 10
        Predicate<Integer> greaterThan10 = x -> x > 10;

        // Пример: числа меньше 100
        Predicate<Integer> lessThan100 = x -> x < 100;

        // and
        Predicate<Integer> between10And100 = greaterThan10.and(lessThan100);
        System.out.println(between10And100.test(50)); // true
        System.out.println(between10And100.test(5));  // false

        // or
        Predicate<Integer> lessOrGreater = greaterThan10.or(x -> x < 0);
        System.out.println(lessOrGreater.test(-5)); // true
        System.out.println(lessOrGreater.test(5));  // false

        // negate
        Predicate<Integer> notGreaterThan10 = greaterThan10.negate();
        System.out.println(notGreaterThan10.test(5));  // true
        System.out.println(notGreaterThan10.test(15)); // false

        // isEqual (статический метод)
        Predicate<String> isHello = Predicate.isEqual("hello");
        System.out.println(isHello.test("hello")); // true
        System.out.println(isHello.test("world")); // false
    }
}
```

### `Function<T, R>`: Преобразование объекта (функция)

- Абстрактный метод: `R apply(T t)`
- Назначение: Принимает объект типа `T` и возвращает результат типа `R`.
- Типичное использование: Преобразование одного объекта в другой (например, String в Integer), маппинг.
- Полезные методы по умолчанию:
    - `andThen(Function<R, V> after)`<br>Применяет текущую функцию, затем применяет `after` (f(x) -> g(f(x)))
    - `compose(Function<V, T> before)`<br>Сначала применяет `before`, потом текущую функцию (f(g(x)))

```java
// Преобразует String в его длину (Integer)
Function<String, Integer> stringLengthFunction = s -> s.length();
System.out.

println(stringLengthFunction.apply("Java")); // 4

// Цепочка функций: toUpperCase -> length
Function<String, String> toUpperCase = String::toUpperCase;
Function<String, Integer> afterUpperCase = toUpperCase.andThen(stringLengthFunction);

System.out.

println(afterUpperCase.apply("lambda")); // 6 (LAMBDA -> 6)
```

```java
public class FunctionExample {
    public static void main(String[] args) {
        // Функция: умножить на 2
        Function<Integer, Integer> multiplyBy2 = x -> x * 2;

        // Функция: прибавить 5
        Function<Integer, Integer> add5 = x -> x + 5;

        // andThen: сначала умножаем на 2, потом прибавляем 5
        Function<Integer, Integer> mult2ThenAdd5 = multiplyBy2.andThen(add5);
        System.out.println(mult2ThenAdd5.apply(10)); // (10 * 2) + 5 = 25

        // compose: сначала прибавляем 5, потом умножаем на 2
        Function<Integer, Integer> add5ThenMult2 = multiplyBy2.compose(add5);
        System.out.println(add5ThenMult2.apply(10)); // (10 + 5) * 2 = 30
    }
}
```

### `Consumer<T>`: Выполнение действия (потребитель)

- Абстрактный метод: `void accept(T t)`
- Назначение: Принимает объект типа T, выполняет с ним какое-либо действие и ничего не возвращает (void).
- Типичное использование: Вывод на печать, модификация объекта, запись в базу/файл.
- Полезные методы по умолчанию: `andThen`

```java
Consumer<String> printer = s -> System.out.println(">> " + s);
printer.

accept("Message"); // Вывод: >> Message

List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
// Классическое использование forEach
names.

forEach(printer); // Выведет все имена с ">> "

// Или напрямую лямбдой:
names.

forEach(name ->System.out.

println("Name: "+name)); // Выведет все имена с ">> "

Consumer<String> printer = s -> System.out.println(">> " + s);
Consumer<String> logger = s -> System.out.println("LOG: " + s);

// Комбинируем два действия: сначала печать, потом логирование
Consumer<String> combined = printer.andThen(logger);

combined.

accept("Test message"); // Вывод: >> Test message LOG: Test message
```

### `Supplier<T>`: Поставщик объектов (поставщик)

- Абстрактный метод: `T get()`
- Назначение: Не принимает аргументов, но возвращает объект типа T.
- Типичное использование: Ленивая инициализация, генерация данных, фабрики.

```java
Supplier<Double> randomSupplier = () -> Math.random();
System.out.

println(randomSupplier.get()); // Например, 0.548412...

Supplier<LocalDateTime> timeSupplier = () -> LocalDateTime.now();
System.out.

println("Task executed at: "+timeSupplier.get());
```
</details>

<details>
    <summary>
        <b>Основные функциональные интерфейсы</b>
    </summary>

### Ключевые отношения

- UnaryOperator<T> = Function<T, T>
- BinaryOperator<T> = BiFunction<T, T, T>
- Runnable = Supplier без возвращаемого значения
- Callable = Supplier с возможностью исключений

### Паттерны именования

- XXXFunction - принимает примитив XXX, возвращает объект
- ToXXXFunction - принимает объект, возвращает примитив XXX
- XXXToYYYFunction - преобразует примитив XXX в YYY
- XXXConsumer - принимает примитив XXX
- XXXSupplier - возвращает примитив XXX
- XXXPredicate - проверяет примитив XXX
- ObjXXXConsumer - принимает (объект, примитив XXX)

### `Function<T, R>` - преобразование (1 вход → 1 выход)
```text
Function<T, R>           T → R
├─ UnaryOperator<T>      T → T
│  ├─ IntUnaryOperator      int → int
│  ├─ LongUnaryOperator     long → long
│  └─ DoubleUnaryOperator   double → double

IntFunction<R>           int → R
LongFunction<R>          long → R  
DoubleFunction<R>        double → R

ToIntFunction<T>         T → int
ToLongFunction<T>        T → long
ToDoubleFunction<T>      T → double

IntToLongFunction        int → long
IntToDoubleFunction      int → double
LongToIntFunction        long → int
LongToDoubleFunction     long → double
DoubleToIntFunction      double → int
DoubleToLongFunction     double → long
```

### `BiFunction<T, U, R>` - преобразование (2 входа → 1 выход)
```text
BiFunction<T, U, R>      (T, U) → R
├─ BinaryOperator<T>     (T, T) → T
│  ├─ IntBinaryOperator     (int, int) → int
│  ├─ LongBinaryOperator    (long, long) → long
│  └─ DoubleBinaryOperator  (double, double) → double

ToIntBiFunction<T, U>    (T, U) → int
ToLongBiFunction<T, U>   (T, U) → long
ToDoubleBiFunction<T, U> (T, U) → double
```

### `Consumer<T>` - потребление (1 вход → нет выхода)
```text
Consumer<T>              T → void
├─ BiConsumer<T, U>      (T, U) → void

ToIntBiFunction<T, U>    (T, U) → int
ToLongBiFunction<T, U>   (T, U) → long
ToDoubleBiFunction<T, U> (T, U) → double
```

### `Supplier<T>` - поставщик (нет входа → 1 выход) 
```text
Supplier<T>              () → T

IntSupplier              () → int
LongSupplier             () → long
DoubleSupplier           () → double
BooleanSupplier          () → boolean
```

### `Predicate<T>` - проверка (1 вход → boolean)
```text
Predicate<T>             T → boolean
├─ BiPredicate<T, U>     (T, U) → boolean

IntPredicate             int → boolean
LongPredicate            long → boolean
DoublePredicate          double → boolean
```
</details>

<details>
    <summary>
        <b>Захват переменных (Variable Capture)</b>
    </summary>

### Локальные переменные: требование effectively final

- Лямбда-выражение может захватывать и использовать только те локальные переменные из охватывающей области видимости (
  `enclosing scope`),
  которые являются `final` или `effectively final`.
- `Effectively final` может быть не объявлена как final, но никогда не меняет своего значения после инициализации.
  Компилятор сам определяет такие переменные.

Почему это правило существует

- Безопасность многопоточности: Лямбда может быть выполнена в другом потоке, чем тот, где была объявлена локальная
  переменная.
  Если бы переменная могла меняться, это привело бы к проблемам с видимостью изменений между потоками (data race).
- Семантика захвата: Лямбда захватывает значение локальной переменной, а не саму переменную. Это похоже на передачу по
  значению.

```java
public class VariableCaptureExample {

    public static void main(String[] args) {
        int normalLocalVar = 10;
        final int finalLocalVar = 20; // final
        int effectivelyFinalVar = 30; // Никогда не меняется -> effectively final

        // ОШИБКА КОМПИЛЯЦИИ! Попытка захвата изменяемой переменной
        Runnable errorLambda = () -> System.out.println(normalLocalVar);

        // Правильно: захват final и effectively final переменных
        Runnable correctLambda = () -> {
            System.out.println("finalLocalVar: " + finalLocalVar);
            System.out.println("effectivelyFinalVar: " + effectivelyFinalVar);
        };
        correctLambda.run();

        // Попытка изменить переменную после создания лямбды
        // normalLocalVar = 100; // Если раскомментировать, effectivelyFinalVar перестанет быть таковой,
        // и лямбда ниже вызовет ошибку компиляции

// Если переменная не изменяется до объявления лямбды - она считается effectively final, и лямбда скомпилируется.
// Если ты изменишь переменную после объявления лямбды - лямбда продолжит работать, т.к. в момент компиляции переменная была effectively final.
// Лямбда захватывает значение, а не переменную, и не видит изменений, происходящих после её объявления.
    }
}
```

Это правило введено именно для лямбд и также распространяется на анонимные классы, начиная с Java 8.

### Поля класса (instance и static): полная свобода

Правило: Лямбда-выражение имеет полный доступ к:

- Статическим полям (static fields) класса.
- Нестатическим полям (instance fields) объекта, в контексте которого она создана.

Важно: Поля класса можно свободно изменять внутри лямбды. Ограничение effectively final на них не распространяется.

```java
public class FieldAccessExample {

    private String instanceField = "Instance field";
    private static String staticField = "Static field";
    private int counter = 0;

    public void testLambda() {
        // Лямбда захватывает контекст текущего объекта (this)
        Runnable fieldAccessLambda = () -> {
            // Можем читать и изменять поля
            System.out.println(instanceField); // Доступ к instance полю
            System.out.println(staticField);   // Доступ к static полю
            counter++; // И даже изменять их! Это разрешено.
            System.out.println("Counter: " + counter);
        };

        fieldAccessLambda.run();

        // Меняем статическое поле и поле экземпляра после создания лямбды - лямбда "увидит" новые значения
        staticField = "Modified static field";
        instanceField = "Modified instance field";

        fieldAccessLambda.run();

        // Поля класса (instance и static): доступны через this, могут изменяться, и лямбда "увидит" изменения при последующем выполнении.
    }

    public static void main(String[] args) {
        new FieldAccessExample().testLambda();
    }
}
```

### Ключевое слово this в лямбдах: отличие от анонимных классов

В лямбда-выражении:

- `this` ссылается на экземпляр окружающего класса, в котором лямбда была объявлена.
- Лямбда не создает своего собственного контекста (`this`).

В анонимном классе:

- `this` ссылается на экземпляр самого анонимного класса.
- Чтобы получить доступ к внешнему классу, используется синтаксис `OuterClass.this.`

```java
public class ThisMeaningExample {

    private String value = "Outer class value";

    public void anonymousClassDemo() {
        Runnable anonymousRunnable = new Runnable() {

            private String value = "Anonymous class value";

            @Override
            public void run() {
                // this ссылается на экземпляр анонимного класса
                System.out.println("Inside anonymous class: " + this.value);
                // Чтобы получить доступ к внешнему классу, нужно явно указать его имя
                System.out.println("Accessing outer from anonymous: " + ThisMeaningExample.this.value);
            }
        };
        anonymousRunnable.run();
    }

    public void lambdaDemo() {
        Runnable lambdaRunnable = () -> {
            // this ссылается на экземпляр ThisMeaningExample!
            System.out.println("Inside lambda: " + this.value);
            // У лямбды нет своего собственного 'this'. Здесь 'this' - это тот же 'this', что и в методе lambdaDemo().
        };
        lambdaRunnable.run();
    }
}
```

Почему так важно это отличие?

Лямбды ведут себя более "естественно" и предсказуемо.
Они являются частью метода, в котором объявлены, и используют его контекст.
`this` в лямбде - это `this` окружающего объекта, а не самой лямбды.
Лямбда не создает новой области видимости для `this`.

Анонимные классы — это отдельные сущности со своим контекстом.
</details>

<details>
    <summary>
        <b>Ссылки на методы (Method References)</b>
    </summary>

Сокращенный синтаксис для лямбда-выражений, которые просто вызывают уже существующий метод.

### Ссылки на статические методы: `ClassName::staticMethod`

- Контекст: Лямбда вызывает статический метод класса.
- Соответствие лямбде: `(args) -> ClassName.staticMethod(args)`

```java
public class MethodReferenceExample {

    public static void main(String[] args) {

        // Пример c Function: преобразование строки в число
        Function<String, Integer> parser1 = s -> Integer.parseInt(s);
        Function<String, Integer> parser2 = Integer::parseInt; // ✅ OK
        System.out.println(parser2.apply("123")); // 123

        // Пример с Predicate: проверка, является ли строка пустой
        Predicate<String> isEmpty1 = s -> s.isEmpty();
        Predicate<String> isEmpty2 = String::isEmpty; // ✅ OK
        System.out.println(isEmpty2.test("")); // true
    }
}
```

### Ссылки на методы экземпляра конкретного объекта: `object::instanceMethod`

- Контекст: Лямбда вызывает метод у конкретного существующего объекта.
- Соответствие лямбде: `(args) -> object.instanceMethod(args)`

```java
// Пример: Проверка, начинается ли строка с определенного префикса
String prefix = "Mr.";
Predicate<String> startsWith1 = s -> s.startsWith(prefix);
Predicate<String> startsWith2 = prefix::startsWith; // Вызываем startsWith у конкретного объекта prefix
System.out.

println(startsWith2.test("Mr. Smith")); // true

// Пример: Вывод в разном формате
Consumer<String> printer1 = s -> System.out.println(s);
Consumer<String> printer2 = System.out::println; // Классический пример
```

### Ссылки на методы произвольного объекта конкретного типа: `ClassName::instanceMethod`

- Контекст: Первый параметр лямбды становится объектом, у которого вызывается метод.
- Соответствие лямбде: `(obj, args) -> obj.instanceMethod(args)`

```java
// Пример: Сравнение строк без ссылки на метод
Comparator<String> comparator1 = (s1, s2) -> s1.compareTo(s2);

// Со ссылкой на метод - первый параметр (s1) становится объектом для compareTo
Comparator<String> comparator2 = String::compareTo;

// Пример: Преобразование к верхнему регистру
Function<String, String> upperCase1 = s -> s.toUpperCase();
Function<String, String> upperCase2 = String::toUpperCase; // s становится неявным получателем

// Пример: Получение длины строки
Function<String, Integer> lengthGetter1 = s -> s.length();
Function<String, Integer> lengthGetter2 = String::length;
```

### Ссылки на конструкторы: `ClassName::new`

- Контекст: Лямбда создает новый объект.
- Соответствие лямбде: `(args) -> new ClassName(args)`

```java
// Пример: Создание строк из других объектов
Function<String, String> stringCreator1 = s -> new String(s);
Function<String, String> stringCreator2 = String::new;

// Пример: Создание списка заданного размера
Function<Integer, List<String>> listCreator1 = size -> new ArrayList<>(size);
Function<Integer, ArrayList<String>> listCreator2 = ArrayList::new;

// Более сложный пример с Supplier
Supplier<List<String>> supplier1 = () -> new ArrayList<>();
Supplier<List<String>> supplier2 = ArrayList::new;
```

```java
// ПЛОХО: Лямбда делает больше, чем просто вызов метода
list.stream()
.

map(String::toUpperCase) // Хорошо - просто вызов метода
.

map(s ->"Prefix: "+s) // Плохо - лямбда делает не просто вызов
        .

forEach(System.out::println);
```

```java
public class MethodReferencesExamples {

    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "bob", "charlie", "david");

        // Цепочка обработки с использованием ссылок на методы
        List<String> processedNames = names.stream()
                .filter(String::isEmpty)       // Статическая ссылка (String.isEmpty())
                .map(String::toUpperCase)      // Ссылка на метод произвольного объекта
                .sorted(String::compareTo)     // Ссылка на метод произвольного объекта  
                .collect(ArrayList::new,       // Ссылка на конструктор
                        ArrayList::add,        // Ссылка на метод экземпляра
                        ArrayList::addAll);

        // Альтернатива - с лямбдами (более многословно)
        List<String> processedNamesLambda = names.stream()
                .filter(s -> s.isEmpty())
                .map(s -> s.toUpperCase())
                .sorted((s1, s2) -> s1.compareTo(s2))
                .collect(() -> new ArrayList<>(),
                        (list, item) -> list.add(item),
                        (list1, list2) -> list1.addAll(list2));
    }
}
```

</details>


<details>
    <summary>
        <b>Практическое применение лямбда-выражений</b>
    </summary>

## Работа с коллекциями

### `List.forEach()`

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// Старый подход
for(
String name :names){
        System.out.

println(name);
}

// Новый подход с лямбдой
        names.

forEach(name ->System.out.

println(name));

// Или со ссылкой на метод
        names.

forEach(System.out::println);
```

### `List.removeIf()`

```java
List<String> list = new ArrayList<>(Arrays.asList("apple", "banana", "cherry", "date"));

// Удалить все элементы, начинающиеся на "c"
list.

removeIf(element ->element.

startsWith("c"));

        System.out.

println(list); // [apple, banana, date]

// Удалить все строки длиной менее 5 символов
list.

removeIf(s ->s.

length() < 5);

        System.out.

println(list); // [banana]
```

### `Map.forEach()`

```java
Map<String, Integer> ages = new HashMap<>();
ages.

put("Alice",25);
ages.

put("Bob",30);
ages.

put("Charlie",35);

// Старый подход
for(
Map.Entry<String, Integer> entry :ages.

entrySet()){
        System.out.

println(entry.getKey() +": "+entry.

getValue() +" years old");
        }

// Новый подход
        ages.

forEach((name, age) ->System.out.

println(name +": "+age+" years old"));
```

## Многопоточность: Runnable и Callable

Лямбды кардинально упрощают создание потоков.

### Runnable с лямбдой

```java
// Старый подход с анонимным классом
Thread oldThread = new Thread(new Runnable() {

            @Override
            public void run() {
                System.out.println("Running in thread (old way)");
            }
        });

// Новый подход с лямбдой
Thread newThread = new Thread(() -> {
    System.out.println("Running in thread (lambda way)");

    // Можно выполнять любую сложную логику
    for (int i = 0; i < 3; i++) {
        System.out.println("Working... " + i);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});
```

### Callable с лямбдой для возврата результата

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

// Старый подход
Future<String> oldFuture = executor.submit(new Callable<String>() {
    @Override
    public String call() throws Exception {
        Thread.sleep(1000); // Имитация долгой операции
        return "Result from old way";
    }
});

// Новый подход
Future<String> newFuture = executor.submit(() -> {
    Thread.sleep(1000); // Имитация долгой операции
    return "Result from lambda";
});

// Использование результата
try{
String result = newFuture.get();
    System.out.

println("Got result: "+result);
}catch(
Exception e){
        e.

printStackTrace();
}

        executor.

shutdown();
```

## Обработка событий: Swing и JavaFX

Идеально подходит для обработчиков событий, где обычно создавалось много boilerplate кода.

### Swing пример

```java
public class SwingLambdaExample {

    public static void main(String[] args) {
        JFrame frame = new JFrame("Lambda Swing Example");
        JButton button = new JButton("Click me!");
        JTextField textField = new JTextField(20);

        // Старый подход с анонимным классом
        button.addActionListener(new ActionListener() {

            @Override
            public void actionPerformed(ActionEvent e) {
                textField.setText("Button clicked (old way)");
            }
        });

        // Новый подход с лямбдой
        button.addActionListener(e -> {
            textField.setText("Button clicked (lambda way)");
            System.out.println("Action performed at: " + java.time.LocalTime.now());
        });

        // Разные обработчики для разных кнопок
        JButton exitButton = new JButton("Exit");
        exitButton.addActionListener(e -> System.exit(0));

        JPanel panel = new JPanel();
        panel.add(button);
        panel.add(exitButton);
        panel.add(textField);

        frame.add(panel);
        frame.setSize(300, 200);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setVisible(true);
    }
}
```

## Сортировка: Comparator с лямбдами

Создание компараторов становится чрезвычайно лаконичным.

```java
    List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Anna", "Alex");

// Сортировка по длине строки
    names.

sort((s1, s2) ->s1.

length() -s2.

length());

        // Сортировка в обратном порядке
        names.

sort((s1, s2) ->s2.

compareTo(s1));

        // Более сложная сортировка: сначала по длине, затем лексикографически
        names.

sort((s1, s2) ->{
int lengthCompare = s1.length() - s2.length();
        if(lengthCompare !=0){
        return lengthCompare;
        }
                return s1.

compareTo(s2);
    });
```

## Условное выполнение (Conditional Execution)

Полезный паттерн для выполнения кода только при определенных условиях.

```java
public class ConditionalExecutor {

    public static void executeIf(boolean condition, Runnable task) {
        if (condition) {
            task.run();
        }
    }

    public static void main(String[] args) {
        boolean isDebug = true;
        boolean isProduction = false;

        // Выполнить только в debug режиме
        executeIf(isDebug, () -> {
            System.out.println("Debug information...");
            System.out.println("Current time: " + java.time.LocalTime.now());
        });

        // Выполнить только в production
        executeIf(isProduction, () -> {
            System.out.println("Production task running...");
            // Отправка метрик, логирование и т.д.
        });

        // Более практичный пример - валидация
        String userInput = "some value";
        executeIf(userInput != null && !userInput.isEmpty(), () -> {
            System.out.println("Processing input: " + userInput);
            // Дополнительная обработка
        });
    }
}
```

### Работа с временными интервалами (таймеры)

```java
// Timer - НЕ совместим с лямбдами (требует анонимный класс)
Timer timerOld = new Timer();
timerOld.

schedule(new TimerTask() {
    @Override
    public void run () {
        System.out.println("Timer task executed (anonymous class)");
    }
},1000);

// ScheduledExecutorService - совместим с лямбдами (использует Runnable/Callable)
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

// Простая задача
scheduler.

schedule(() ->System.out.println("Simple task with lambda"), 1,TimeUnit.SECONDS);

// Сложная задача
scheduler.schedule(() -> {
        System.out.println("Complex task started at: "+java.time.LocalTime.now());
        // Имитация работы
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Complex task finished");
},2,TimeUnit.SECONDS);

// Периодическая задача
scheduler.scheduleAtFixedRate(() -> {
        System.out.println("Periodic task executed");
},0,3,TimeUnit.SECONDS);
```
</details>

<details>
    <summary>
        <b>Внутренние механизмы и паттерны использования лямбд</b>
    </summary>

### Замыкания (Closures)

В Java лямбды являются замыканиями только по значению (capture-by-value), а не по ссылке.

- Замыкание - это функция, которая "запоминает" окружение, в котором была создана.
- Лямбда в Java захватывает копии значений локальных переменных, а не сами переменные.

```java
public class ClosureExample {
    public static void main(String[] args) {
        String message = "Hello"; // effectively final
        int counter = 0;          // effectively final
        
        Runnable closure = () -> {
            // Захватываем КОПИИ значений message и counter
            System.out.println(message); // "Hello"
            // System.out.println(counter++); // Ошибка, если бы мы попытались изменить counter
        };
        
        // Изменение переменных после создания лямбды НЕ влияет на захваченные значения
        // message = "Changed"; // Если раскомментировать - лямбда не скомпилируется
        
        closure.run();
    }
    
    // Практический пример: фабрика задач
    public static Runnable createTask(String taskName, int delay) {
        // taskName и delay захватываются лямбдой
        return () -> {
            try {
                Thread.sleep(delay);
                System.out.println("Task '" + taskName + "' completed after " + delay + "ms");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
    }
}

// Использование
Runnable task1 = ClosureExample.createTask("Data Processing", 1000);
Runnable task2 = ClosureExample.createTask("File Cleanup", 500);
new Thread(task1).start();
new Thread(task2).start();
```

### Ленивые вычисления: отложенное выполнение

- Лямбды позволяют реализовать паттерн "ленивых вычислений" - код выполняется только когда результат действительно нужен.

```java
public class LazyEvaluation {
    
    // Ленивый логгер: сообщение вычисляется только если уровень логирования включен
    public static void logIf(Level level, Supplier<String> messageSupplier) {
        if (isLevelEnabled(level)) {
            // messageSupplier.get() вызывается только здесь!
            String message = messageSupplier.get();
            System.out.println("[" + level + "] " + message);
        }
        // Если уровень выключен - messageSupplier никогда не вызовется
    }
    
    private static boolean isLevelEnabled(Level level) {
        // В реальном приложении здесь проверка настроек логирования
        return level == Level.DEBUG; // Например, debug включен
    }
    
    // Дорогая операция, которую не хотим выполнять без необходимости
    private static String expensiveOperation() {
        System.out.println(">>> Выполняется дорогая операция!");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "Результат дорогой операции";
    }
    
    public static void main(String[] args) {
        // Старый подход: дорогая операция выполняется ВСЕГДА
        // String result = expensiveOperation(); // Выполнится сразу
        // logIf(Level.INFO, result);           // А может и не понадобиться
        
        // Новый подход: дорогая операция выполняется ТОЛЬКО при необходимости
        logIf(Level.DEBUG, () -> expensiveOperation()); // Выполнится
        logIf(Level.INFO, () -> expensiveOperation());  // НЕ выполнится!
    }
}

enum Level { DEBUG, INFO, WARN, ERROR }
```

### Композиция функций: andThen(), compose()

- Функциональные интерфейсы предоставляют методы для создания цепочек операций.

```java
public class FunctionComposition {
    
    public static void main(String[] args) {
        // Базовые функции
        Function<String, String> toUpper = String::toUpperCase;
        Function<String, String> addGreeting = s -> "Hello, " + s;
        Function<String, Integer> getLength = String::length;
        
        // Композиция: andThen - выполнить ПОСЛЕ
        Function<String, String> upperThenGreet = toUpper.andThen(addGreeting);
        System.out.println(upperThenGreet.apply("world")); // "Hello, WORLD"
        
        // Композиция: compose - выполнить ДО
        Function<String, String> greetThenUpper = toUpper.compose(addGreeting);
        System.out.println(greetThenUpper.apply("world")); // "HELLO, WORLD"
        
        // Более сложная цепочка
        Function<String, Integer> complexPipeline = 
            addGreeting        // "Hello, world"
            .andThen(toUpper)  // "HELLO, WORLD"  
            .andThen(getLength); // 13
        
        System.out.println(complexPipeline.apply("world")); // 13
        
        // Композиция предикатов
        Predicate<String> isLong = s -> s.length() > 3;
        Predicate<String> startsWithA = s -> s.startsWith("A");
        
        Predicate<String> longAndStartsWithA = isLong.and(startsWithA);
        Predicate<String> longOrStartsWithA = isLong.or(startsWithA);
        Predicate<String> notLong = isLong.negate();
        
        System.out.println(longAndStartsWithA.test("Alice")); // true
        System.out.println(longAndStartsWithA.test("Bob"));   // false
    }
}
```

### Отладка лямбд: stack traces, логирование

- Лямбды могут усложнять отладку из-за анонимности

```java
public class LambdaDebugging {
    
    // 1. Именованные лямбды (через присваивание)
    private static final Consumer<String> DEBUG_PRINTER = message -> {
        System.out.println("DEBUG: " + message);
        // Точка останова здесь будет иметь понятное имя
    };
    
    // 2. Helper методы для логирования
    private static <T> Consumer<T> debugConsumer(Consumer<T> original, String name) {
        return item -> {
            System.out.println("[" + name + "] Processing: " + item);
            original.accept(item);
        };
    }
    
    // 3. Обертка для лямбды с логированием времени
    private static Runnable timed(Runnable task, String taskName) {
        return () -> {
            long start = System.currentTimeMillis();
            try {
                task.run();
            } finally {
                long duration = System.currentTimeMillis() - start;
                System.out.println("Task '" + taskName + "' took " + duration + "ms");
            }
        };
    }
    
    public static void main(String[] args) {
        List<String> data = Arrays.asList("apple", "banana", "cherry");
        
        // Сложно отлаживать
        data.forEach(s -> {
            // Точка останова здесь показывается как "lambda$main$0"
            if (s.length() > 5) {
                System.out.println(s.toUpperCase());
            }
        });
        
        // Легче отлаживать
        data.forEach(debugConsumer(s -> {
            // Теперь видно контекст в stack trace
            if (s.length() > 5) {
                System.out.println(s.toUpperCase());
            }
        }, "StringProcessor"));
        
        // Отладка производительности
        Runnable expensiveTask = timed(() -> {
            try {
                Thread.sleep(500);
                System.out.println("Work completed");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "Expensive Operation");
        
        expensiveTask.run();
    }
    
    // 4. Исключения в лямбдах - правильно обрабатываем
    public static void processWithExceptionHandling(List<String> items) {
        items.forEach(item -> {
            try {
                // Код, который может бросить исключение
                processItem(item);
            } catch (Exception e) {
                System.err.println("Failed to process " + item + ": " + e.getMessage());
                // Логируем, но не прерываем обработку остальных элементов
            }
        });
    }
    
    private static void processItem(String item) {
        if (item.equals("error")) {
            throw new RuntimeException("Simulated error");
        }
        System.out.println("Processed: " + item);
    }
}
```

### Каррирование (Currying)

- Каррирование — это преобразование функции от нескольких аргументов в цепочку функций от одного аргумента

```java
// Вместо:
f(a, b) → результат

// Получаем:
f(a)(b) → результат
```
```java
// Обычная функция двух аргументов
static BiFunction<Integer, Integer, Integer> simpleAdd = 
    (a, b) -> a + b;
```
- `simpleAdd` — обычная функция:
- Тип: `BiFunction<Integer, Integer, Integer>`
- Принимает два аргумента типа Integer
- Возвращает Integer
- Использование: simpleAdd.apply(5, 3) → 8

```java
// Каррированная версия
static Function<Integer, Function<Integer, Integer>> curriedAdd = 
    a -> b -> a + b;
```
- `curriedAdd` — каррированная функция:
- Тип: `Function<Integer, Function<Integer, Integer>>`
- Принимает один аргумент a (тип Integer)
- Возвращает другую функцию: `Function<Integer, Integer>`, которая:
- Принимает один аргумент b (тип Integer)
- Возвращает Integer (результат a + b)

`a -> b -> a + b` - это лямбда, которая возвращает другую лямбду:
- a — захватывается
- Возвращается функция, которая принимает b и возвращает a + b

```java
public class CurryingExample {
    
    // Обычная функция двух аргументов
    static BiFunction<Integer, Integer, Integer> simpleAdd = 
        (a, b) -> a + b;
    
    // Каррированная версия
    static Function<Integer, Function<Integer, Integer>> curriedAdd = 
        a -> b -> a + b;
    
    public static void main(String[] args) {
        // Обычное использование
        System.out.println(simpleAdd.apply(5, 3)); // 8
        
        // Каррированное использование
        
        // Вызывает внешнюю функцию с a = 5
        // Возвращает новую функцию: b -> 5 + b
        // Сохраняется в add5
        Function<Integer, Integer> add5 = curriedAdd.apply(5);

        // Вызывает функцию b -> 5 + b (с b = 3)
        System.out.println(add5.apply(3)); // 5 + 3 = 8
        
        // Вызывает функцию b -> 5 + b (с b = 10) 
        System.out.println(add5.apply(10)); // 5 + 10 = 15
        
        
        // Практическое применение: создание специализированных функций
        Function<String, Predicate<String>> startsWith = 
            prefix -> text -> text.startsWith(prefix);
            
        Predicate<String> startsWithA = startsWith.apply("A");
        Predicate<String> startsWithB = startsWith.apply("B");
        
        System.out.println(startsWithA.test("Alice")); // true
        System.out.println(startsWithB.test("Alice")); // false
    }
    
    // Каррирование для функций трех аргументов
    static Function<Integer, Function<Integer, Function<Integer, Integer>>> 
        curriedTriFunction = a -> b -> c -> a + b + c;
}
``` 
</details>
