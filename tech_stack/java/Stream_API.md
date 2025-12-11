# Stream API в Java

### Концепция

`Stream API` - это не структура данных, а абстракция для представления последовательности элементов, над которой можно производить функциональные операции.

### Ключевые идеи:

- Не хранит данные. Поток берет данные из источника (коллекция, массив, I/O-канал).
- Не изменяет источник. Операции над потоком (например, фильтрация) возвращают новый поток, а не меняют исходную коллекцию.
- Lazy и Eager операции. Цепочка операций выполняется только при вызове терминальной операции.
- Однократное использование. Поток нельзя использовать повторно после вызова терминальной операции.

### Общая схема работы:

`Источник -> Промежуточные операции -> Терминальная операция`

<details>
    <summary>
        <b>Создание потоков (Источники)</b>
    </summary>

| Сигнатура метода                                                                         | Назначение                                                         | Пример                                       |
|:-----------------------------------------------------------------------------------------|:-------------------------------------------------------------------|:---------------------------------------------|
| `default Stream<E> stream()`                                                             | Создание последовательного потока из коллекции                     | `collection.stream()`                        |
| `default Stream<E> parallelStream()`                                                     | Создание параллельного потока из коллекции                         | `collection.parallelStream()`                |
| `static <T> Stream<T> of(T... values)`                                                   | Создание потока из набора значений                                 | `Stream.of("a", "b", "c")`                   |
| `static <T> Stream<T> ofNullable(T t)`                                                   | Создание потока из одного элемента или пустого потока, если `null` | `Stream.ofNullable(nullableObject)`          |
| `static <T> Stream<T> empty()`                                                           | Создание пустого потока                                            | `Stream.empty()`                             |
| `static <T> Stream<T> generate(Supplier<T> s)`                                           | Создание бесконечного потока с помощью поставщика                  | `Stream.generate(() -> "element").limit(10)` |
| `static <T> Stream<T> iterate(T seed, UnaryOperator<T> f)`                               | Создание бесконечного потока с итеративным применением функции     | `Stream.iterate(0, n -> n + 2).limit(5)`     |
| `static <T> Stream<T> iterate(T seed, Predicate<? super T> hasNext, UnaryOperator<T> f)` | Создание конечного потока с условием остановки (Java 9+)           | `Stream.iterate(0, n -> n < 10, n -> n + 2)` |
| `static <T> Stream<T> stream(T[] array)`                                                 | Создание потока из массива для ссылочных типов                     | `Arrays.stream(array)`                       |
| `static IntStream stream(int[] array)`                                                   | Создание потока из массива для примитивных типов                   | `Arrays.stream(array)`                       |
| `static Stream<String> lines(Path path)`                                                 | Создание потока строк из файла                                     | `Files.lines(Paths.get("file.txt"))`         |
| `static IntStream range(int startInclusive, int endExclusive)`                           | Создание потока целых чисел в диапазоне [start, end)               | `IntStream.range(0, 5)` // 0,1,2,3,4         |
| `static IntStream rangeClosed(int startInclusive, int endInclusive)`                     | Создание потока целых чисел в диапазоне [start, end]               | `IntStream.rangeClosed(0, 5)` // 0,1,2,3,4,5 |
| `static <T> Stream<T> concat(Stream<? extends T> a, Stream<? extends T> b)`              | Конкатенация двух потоков в один                                   | `Stream.concat(stream1, stream2)`            |
| `static <T> Stream.Builder<T> builder()`                                                 | Создание потока через билдер                                       | `Stream.builder().add("a").add("b").build()` |

## Несколько примеров создания стрима

### Создание стрима из существующей коллекции

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Stream;

public class CollectionStreamExample {
    
    public static void main(String[] args) {
        List<String> words = Arrays.asList("hello", "world", "java");

        Stream<String> stream = words.stream();     // Создаем стрим из коллекции

        stream.forEach(System.out::println);        // Выводим элементы стрима
    }
}
```

### Преобразование массива в стрим

```java
import java.util.stream.Stream;

public class ArrayStreamExample {
    
    public static void main(String[] args) {
        String[] words = {"hello", "world", "java"};

        Stream<String> stream = Arrays.stream(words);   // Создаем стрим из массива

        stream.forEach(System.out::println);            // Выводим элементы стрима
    }
}
```

### Использование статического метода Stream.of() для передачи элементов.

```java
import java.util.stream.Stream;

public class StreamOfExample {
    
    public static void main(String[] args) {
        Stream<String> stream = Stream.of("hello", "world", "java");    // Создаем стрим из элементов

        stream.forEach(System.out::println);                            // Выводим элементы стрима
    }
}
```

### Использование Stream.builder() для добавления элементов динамически

```java
import java.util.stream.Stream;

public class StreamBuilderExample {
    
    public static void main(String[] args) {
        Stream.Builder<String> builder = Stream.builder();

        builder.add("hello");
        builder.add("world");
        builder.add("java");

        Stream<String> stream = builder.build();    // Создаем стрим из билдера

        stream.forEach(System.out::println);        // Выводим элементы стрима
    }
}
```

### Создание бесконечного стрима на основе функции-генератора

```java
import java.util.stream.Stream;

public class StreamGenerateExample {
 
    public static void main(String[] args) {
        Stream<Double> stream = Stream.generate(Math::random);  // Создаем бесконечный стрим случайных чисел

        stream.limit(5)                                         // Ограничиваем количество элементов
              .forEach(System.out::println);                    // Выводим элементы стрима
    }
}
```

### Создание бесконечного стрима с помощью начального значения и функции-генератора

```java
import java.util.stream.Stream;

public class StreamIterateExample {
    
    public static void main(String[] args) {
        Stream<Integer> stream = Stream.iterate(0, n -> n + 2);     // Создаем бесконечный стрим четных чисел

        stream.limit(5)                                             // Ограничиваем количество элементов
              .forEach(System.out::println);                        // Выводим элементы стрима
        
        // Stream.iterate(0, n -> n < 5, n -> n + 2).forEach(System.out::println); // перегруженный метод с предикатом

    }
}
```

### Преобразование строки в стрим символов, кодовых точек или строк

```java
import java.util.stream.IntStream;

public class StringStreamExample {
    public static void main(String[] args) {
        String text = "hello";

        // Создаем стрим кодовых точек символов
        IntStream charStream = text.chars();                        // Работает только с ASCII или BMP символами
        
        IntStream codePointStream = text.codePoints();              // Работает с текстом, который может содержать эмодзи или специальные символы
        
        charStream.forEach(ch -> System.out.println((char) ch));    // Выводим символы
        
        // При использовании .codePoints() лучше использовать Character.toCars(int codePoint)
        codePointStream
                .map(Character::toChars)                            // возвращает char[] 
                .map(String::new)
                .forEach(System.out::println);
    }
}
```

### Через методы из библиотеки Files

```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.stream.Stream;

public class FilesStreamExample {
 
    public static void main(String[] args) {
        try (Stream<String> stream = Files.lines(Paths.get("example.txt"))) {   // Создаем стрим из строк файла
            stream.forEach(System.out::println);                                // Выводим строки файла
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
</details>

<br>

<details>
    <summary>
        <b>Промежуточные операции (Intermediate Operations)</b>
    </summary>

Промежуточные операции ленивые (не выполняются без терминальной операции) и возвращают новый поток. 

| Сигнатура метода                  | Назначение                                             | Пример                                                   |
|:----------------------------------|:-------------------------------------------------------|:---------------------------------------------------------|
| `filter(Predicate<T>)`            | Фильтрация элементов по условию                        | `stream.filter(s -> s.length() > 3)`                     |
| `map(Function<T, R>)`             | Преобразование каждого элемента                        | `stream.map(String::toUpperCase)`                        |
| `flatMap(Function<T, Stream<R>>)` | Преобразование элемента в поток и "склеивание" потоков | `stream.flatMap(line -> Arrays.stream(line.split(" ")))` |
| `distinct()`                      | Удаление дубликатов (через `equals`)                   | `stream.distinct()`                                      |
| `sorted()`                        | Сортировка натуральная                                 | `stream.sorted()`                                        |
| `sorted(Comparator<T>)`           | Сортировка по компаратору                              | `stream.sorted(Comparator.reverseOrder())`               |
| `peek(Consumer<T>)`               | Просмотр каждого элемента без изменения элементов      | `stream.peek(System.out::println)`                       |
| `limit(long maxSize)`             | Ограничение количества элементов                       | `stream.limit(5)`                                        |
| `skip(long n)`                    | Пропуск первых N элементов                             | `stream.skip(2)`                                         |

Промежуточные операции создают новый стрим, который можно дальше модифицировать. Операторы в стримах в Java обеспечивают эффективную, декларативную обработку данных, что делает код более читаемым и поддерживаемым.

Если стрим в Java собрать без терминального оператора, то операции промежуточные (intermediate) не будут выполнены!

Стрим будет создан, но никаких данных не будет обработано до момента вызова терминальной операции.

Промежуточные операции в Java являются ленивыми (lazy), что означает, что они не выполняются, пока не потребуется

### Открытый стрим

Открытый стрим - это стрим, к которому еще можно применять операции.

```java
List<String> strings = Arrays.asList("a", "b", "c");
Stream<String> stream = strings.stream();           // открытый стрим
stream.filter(s -> s.contains("a"));                // промежуточная операция
```

</details>

<br>

<details>
    <summary>
        <b>Терминальные операции (Terminal Operations)</b>
    </summary>

Запускают выполнение всего конвейера и возвращают результат (не поток). "Жадные" (eager).

| Сигнатура метода                                      | Назначение                                                | Пример                                                                                                            |
|:------------------------------------------------------|:----------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------|
| `void forEach(Consumer<T> action)`                    | Выполнение действия для каждого элемента                  | `stream.forEach(System.out::println)`                                                                             |
| `<R, A> R collect(Collector<T, A, R> collector)`      | Сбор элементов в коллекцию или другую структуру           | `stream.collect(Collectors.toList())`                                                                             |
| `Object[] toArray()`                                  | Сбор элементов в массив Object[]                          | `stream.toArray()`                                                                                                |
| `<A> A[] toArray(IntFunction<A[]> generator)`         | Сбор элементов в массив нужного типа                      | `stream.toArray(String[]::new)`<br>`stream.toArray(Integer[]::new)`<br>`stream.toArray(size -> new String[size])` |
| `Optional<T> reduce(BinaryOperator<T> accumulator)`   | Свертка элементов в одно значение (возвращает `Optional`) | `stream.reduce((a, b) -> a + b)`                                                                                  |
| `T reduce(T identity, BinaryOperator<T> accumulator)` | Свертка с начальным значением                             | `stream.reduce("", (a, b) -> a + b)`                                                                              |
| `Optional<T> findFirst()`                             | Найти первый элемент потока                               | `stream.findFirst()`                                                                                              |
| `Optional<T> findAny()`                               | Найти любой элемент (полезно в параллельных потоках)      | `stream.findAny()`                                                                                                |
| `boolean anyMatch(Predicate<T> predicate)`            | Проверка, что хотя бы один элемент соответствует условию  | `stream.anyMatch(s -> s.startsWith("A"))`                                                                         |
| `boolean allMatch(Predicate<T> predicate)`            | Проверка, что все элементы соответствуют условию          | `stream.allMatch(s -> s.length() > 2)`                                                                            |
| `boolean noneMatch(Predicate<T> predicate)`           | Проверка, что ни один элемент не соответствует условию    | `stream.noneMatch(s -> s.isEmpty())`                                                                              |
| `long count()`                                        | Подсчет количества элементов                              | `stream.count()`                                                                                                  |
| `Optional<T> min(Comparator<T> comparator)`           | Поиск минимального элемента                               | `stream.min(Comparator.naturalOrder())`                                                                           |
| `Optional<T> max(Comparator<T> comparator)`           | Поиск максимального элемента                              | `stream.max(Comparator.naturalOrder())`                                                                           |

Примечание: Для примитивных специализаций потоков (IntStream, LongStream, DoubleStream) методы reduce, min, max, sum возвращают примитивные значения, а не Optional.

### Закрытый стрим

Закрытый стрим - это стрим, к которому уже была применена терминальная операция, и он не может быть использован для последующих промежуточных операций.

```java
List<String> strings = Arrays.asList("a", "b", "c");
Stream<String> stream = strings.stream();                           // открытый стрим
stream.filter(s -> s.contains("a")).forEach(System.out::println);   // терминальная операция - стрим теперь закрыт
```
</details>

<br>

<details>
    <summary>
        <b>Stateless и Stateful операторы</b>
    </summary>

### `Stateless` - ленивость сохраняется - эффективно при досрочном завершении (`findFirst()`, `anyMatch()`).

### `Stateful` - ломает ленивость - может потребовать полной материализации промежуточных данных - потенциально дороже по памяти и времени на больших данных.

## Stateless (Состояние-независимые)

> `filter(Predicate<? super T> predicate)`

> `map(Function<? super T, ? extends R> mapper)`

> `flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)`

> `peek(Consumer<? super T> action)`

> `distinct()` - (в некоторых случаях может быть stateful, но обычно считается stateless)

- совместимы с ленивой (lazy) природой потоков 
- могут остановиться досрочно (например, при findFirst())
- не требуют сохранения состояния между вызовами
- не зависят от предыдущих элементов для обработки текущего
- обрабатывают каждый элемент независимо от других элементов

Stateless операции (map(), filter(), peek()) обрабатывают элементы по мере необходимости и не требуют знания о других элементах. 

```java
        Optional<Integer> result = Stream.of(1, 2, 3, 4, 5)
                .map(n -> {
                    System.out.println("  [map] обрабатывается: " + n);
                    return n * 10;
                })
                .filter(x -> {
                    System.out.println("  [filter] проверяется: " + x);
                    return x > 0;           // всегда true, но демонстрирует вызов
                })
                .findFirst();               // терминальная операция остановка после первого элемента
```

## Stateful (Состояние-зависимые)

> `sorted()`

>`distinct()` - (в некоторых случаях может быть stateful, если требуется сохранить состояние для определения уникальности элементов)

>`limit(long maxSize)`

> `skip(long n)`

</details>

<br>

<details>
    <summary>
        <b>Метод collect() и коллекторы</b>
    </summary>

### Основная сигнатура: `<R, A> R collect(Collector<? super T, A, R> collector)`

- `T` - Тип элементов в потоке. Это тип объектов, которые проходят через `Stream<T>`. Например, если у вас `Stream<String>`, то `T` - это `String`.
- `A` - Тип аккумулятора. Это внутреннее состояние или контейнер, используемый коллектором для сбора элементов. Это может быть `StringBuilder` (для `joining`), `List` (для `toList`), `Map` (для `groupingBy`), `Integer` (для подсчета) или любой другой тип, который необходим для промежуточного хранения данных во время процесса сбора. Этот тип не виден напрямую в вызове `collect()`, но он определен внутри реализации `Collector`.
- `R` - Тип результата. Это конечный тип объекта, который будет возвращен методом `collect()`. Например, `List<String>`, `Map<K, V>`, `String`, `Integer` и т.д.

`Collector<? super T, A, R>:` - Это интерфейс, который определяет, как собирать элементы потока в конечный результат.

- `? super T`: Коллектор может принимать элементы типа `T` или его предков. Это обеспечивает ковариантность.
- `A`: Тип аккумулятора.
- `R`: Тип результата.

```java
List<String> result = Stream.of("a", "b", "c")
    .collect(Collectors.toList());
// T = String (элементы потока)
// A = ArrayList (внутреннее состояние, например, в реализации toList)
// R = List<String> (конечный результат)
```

### Альтернативная (более низкоуровневая) сигнатура: `<R> R collect(Supplier<R> supplier, BiConsumer<R, ? super T> accumulator, BiConsumer<R, R> combiner)`

- `R` - Тип результата. В этой сигнатуре `R` одновременно играет роль аккумулятора и результата. То есть промежуточное состояние и конечный результат имеют один и тот же тип. Это менее гибко, чем первая сигнатура, но проще для понимания и реализации простых случаев. В контексте первой сигнатуры Collector<? super T, A, R>, если использовать teeing или mapping, A и R могут отличаться. В этой сигнатуре они одинаковы.
- `Supplier<R> supplier` - Функция, которая создает новый пустой контейнер аккумуляции (например, новый `ArrayList`, `HashMap`, `StringBuilder`). Вызывается, когда нужно начать сбор (и снова при параллельной обработке для каждого сегмента).
- `BiConsumer<R, ? super T> accumulator` - Функция, которая принимает текущий аккумулятор `R` и один элемент потока `? super T`, и изменяет аккумулятор, добавляя к нему элемент. Это основной шаг сбора.
- `BiConsumer<R, R> combiner` - Функция, которая принимает два аккумулятора `R` и `R` и объединяет их содержимое в один аккумулятор. Используется только при параллельной обработке, когда результаты от разных частей потока нужно объединить.

```java
// Сбор строк в одну строку с разделителем ","
String result = Stream.of("a", "b", "c").collect(
    StringBuilder::new,                         // Supplier<StringBuilder> supplier (R = StringBuilder)
    (sb, str) -> sb.append(str).append(","),    // BiConsumer<StringBuilder, String> accumulator (R = StringBuilder, T = String)
    (sb1, sb2) -> sb1.append(sb2)               // BiConsumer<StringBuilder, StringBuilder> combiner (R = StringBuilder)
);
// T = String
// A = StringBuilder (но неявно, так как A = R)
// R = StringBuilder (и возвращаемый тип метода collect)
// Результат: "a,b,c,"
```
### Коллекторы (Collectors)

| Сигнатура метода                                                                                                                                                                                                    | Назначение                                                                                                       | Пример                                                                                                          |
|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------|
| `static <T> Collector<T, ?, List<T>> toList()`                                                                                                                                                                      | Сбор элементов в `List` (в JDK 16+ возвращаемый список неизменяем)                                               | `stream.collect(Collectors.toList())`                                                                           |
| `static <T> Collector<T, ?, List<T>> toUnmodifiableList()`                                                                                                                                                          | Сбор элементов в неизменяемый `List`                                                                             | `stream.collect(Collectors.toUnmodifiableList())`                                                               |
| `static <T> Collector<T, ?, Set<T>> toSet()`                                                                                                                                                                        | Сбор элементов в `Set` (в JDK 16+ возвращаемый сет неизменяем)                                                   | `stream.collect(Collectors.toSet())`                                                                            |
| `static <T> Collector<T, ?, Set<T>> toUnmodifiableSet()`                                                                                                                                                            | Сбор элементов в неизменяемый `Set`                                                                              | `stream.collect(Collectors.toUnmodifiableSet())`                                                                |
| `static <T, C extends Collection<T>> Collector<T, ?, C> toCollection(Supplier<C> collectionFactory)`                                                                                                                | Сбор элементов в коллекцию, созданную через `collectionFactory`                                                  | `stream.collect(Collectors.toCollection(LinkedList::new))`                                                      |
| `static <T, K> Collector<T, ?, Map<K, List<T>>> groupingBy(Function<? super T, ? extends K> classifier)`                                                                                                            | Группировка элементов по классификатору в `Map<K, List<T>>`                                                      | `stream.collect(Collectors.groupingBy(Person::getCity))`                                                        |
| `static <T, K, A, D> Collector<T, ?, Map<K, D>> groupingBy(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream)`                                                                     | Группировка элементов по классификатору, с дополнительной обработкой (`downstream`) для значений                 | `stream.collect(Collectors.groupingBy(Person::getCity, Collectors.counting()))`                                 |
| `static <T, K, D, A, M extends Map<K, D>> Collector<T, ?, M> groupingBy(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream)`                                | Группировка элементов по классификатору, с фабрикой карты и дополнительной обработкой значений                   | `stream.collect(Collectors.groupingBy(Person::getCity, LinkedHashMap::new, Collectors.toList()))`               |
| `static <T, K> Collector<T, ?, ConcurrentMap<K, List<T>>> groupingByConcurrent(Function<? super T, ? extends K> classifier)`                                                                                        | Группировка элементов по классификатору в `ConcurrentMap<K, List<T>>`                                            | `stream.collect(Collectors.groupingByConcurrent(Person::getCity))`                                              |
| `static <T, K, A, D> Collector<T, ?, ConcurrentMap<K, D>> groupingByConcurrent(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream)`                                                 | Группировка элементов по классификатору в `ConcurrentMap`, с дополнительной обработкой значений                  | `stream.collect(Collectors.groupingByConcurrent(Person::getCity, Collectors.counting()))`                       |
| `static <T, K, D, A, M extends ConcurrentMap<K, D>> Collector<T, ?, M> groupingByConcurrent(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream)`            | Группировка элементов по классификатору в `ConcurrentMap`, с фабрикой карты и дополнительной обработкой значений | `stream.collect(Collectors.groupingByConcurrent(Person::getCity, ConcurrentHashMap::new, Collectors.toList()))` |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper)`                                                                       | Сбор элементов в `Map`, где ключ и значение определяются функциями                                               | `stream.collect(Collectors.toMap(Person::getId, Person::getName))`                                              |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper, BinaryOperator<U> mergeFunction)`                                      | Сбор элементов в `Map`, с функцией слияния значений для дубликатов ключей                                        | `stream.collect(Collectors.toMap(Person::getId, Person::getName, (name1, name2) -> name1))`                     |
| `static <T, K, U, M extends Map<K, U>> Collector<T, ?, M> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper, BinaryOperator<U> mergeFunction, Supplier<M> mapFactory)` | Сбор элементов в `Map`, с фабрикой карты, функцией слияния значений для дубликатов ключей                        | `stream.collect(Collectors.toMap(Person::getId, Person::getName, (name1, name2) -> name1, LinkedHashMap::new))` |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toUnmodifiableMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper)`                                                           | Сбор элементов в неизменяемую `Map`                                                                              | `stream.collect(Collectors.toUnmodifiableMap(Person::getId, Person::getName))`                                  |
| `static Collector<CharSequence, ?, String> joining()`                                                                                                                                                               | Объединение строковых представлений элементов в строку                                                           | `Stream.of("a", "b", "c").collect(Collectors.joining())` // результат: "abc"                                    |
| `static Collector<CharSequence, ?, String> joining(CharSequence delimiter)`                                                                                                                                         | Объединение строковых представлений элементов в строку с разделителем                                            | `Stream.of("a", "b", "c").collect(Collectors.joining(","))` // результат: "a,b,c"                               |
| `static Collector<CharSequence, ?, String> joining(CharSequence delimiter, CharSequence prefix, CharSequence suffix)`                                                                                               | Объединение строковых представлений элементов в строку с разделителем, префиксом и суффиксом                     | `Stream.of("a", "b", "c").collect(Collectors.joining(",", "[", "]"))` // результат: "[a,b,c]"                   |
| `static <T, A, R> Collector<T, A, R> collectingAndThen(Collector<T, A, R> downstream, Function<R, R> finisher)`                                                                                                     | Применение дополнительной функции к результату другого коллектора                                                | `stream.collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList))`              |
| `static <T> Collector<T, ?, T> reducing(T identity, BinaryOperator<T> op)`                                                                                                                                          | Свёртка элементов с начальным значением и бинарным оператором                                                    | `stream.collect(Collectors.reducing(0, Integer::sum))`                                                          |
| `static <T> Collector<T, ?, Optional<T>> reducing(BinaryOperator<T> op)`                                                                                                                                            | Свёртка элементов с бинарным оператором, возвращает `Optional`                                                   | `stream.collect(Collectors.reducing(Integer::max))`                                                             |
| `static <T, U> Collector<T, ?, U> reducing(U identity, Function<? super T, ? extends U> mapper, BinaryOperator<U> op)`                                                                                              | Свёртка элементов, преобразованных через `mapper`, с начальным значением и оператором                            | `stream.collect(Collectors.reducing(0, String::length, Integer::sum))`                                          |
| `static <T> Collector<T, ?, Optional<T>> minBy(Comparator<? super T> comparator)`                                                                                                                                   | Поиск минимального элемента по компаратору                                                                       | `stream.collect(Collectors.minBy(Comparator.naturalOrder()))`                                                   |
| `static <T> Collector<T, ?, Optional<T>> maxBy(Comparator<? super T> comparator)`                                                                                                                                   | Поиск максимального элемента по компаратору                                                                      | `stream.collect(Collectors.maxBy(Comparator.naturalOrder()))`                                                   |
| `static <T, A, R> Collector<T, ?, R> mapping(Function<? super T, ? extends A> mapper, Collector<A, ?, R> downstream)`                                                                                               | Преобразование элементов с последующей обработкой `downstream` коллектором                                       | `stream.collect(Collectors.mapping(Person::getName, Collectors.toList()))`                                      |
| `static <T> Collector<T, ?, Long> counting()`                                                                                                                                                                       | Подсчёт количества элементов                                                                                     | `stream.collect(Collectors.counting())`                                                                         |
| `static <T, U, A, R> Collector<T, ?, R> flatMapping(Function<? super T, ? extends Stream<? extends U>> mapper, Collector<? super U, A, R> downstream)`                                                              | Преобразование элементов в потоки, их плоское объединение и последующая обработка `downstream` коллектором       | `stream.collect(Collectors.flatMapping(str -> Stream.of(str.split(" ")), Collectors.counting()))`               |
| `static <T> Collector<T, ?, Integer> summingInt(ToIntFunction<? super T> mapper)`                                                                                                                                   | Вычисление суммы `int` значений, полученных из элементов через `mapper`                                          | `stream.collect(Collectors.summingInt(Person::getScore))`                                                       |
| `static <T> Collector<T, ?, Long> summingLong(ToLongFunction<? super T> mapper)`                                                                                                                                    | Вычисление суммы `long` значений, полученных из элементов через `mapper`                                         | `stream.collect(Collectors.summingLong(Person::getAge))`                                                        |
| `static <T> Collector<T, ?, Double> summingDouble(ToDoubleFunction<? super T> mapper)`                                                                                                                              | Вычисление суммы `double` значений, полученных из элементов через `mapper`                                       | `stream.collect(Collectors.summingDouble(Person::getSalary))`                                                   |
| `static <T> Collector<T, ?, Double> averagingInt(ToIntFunction<? super T> mapper)`                                                                                                                                  | Вычисление среднего значения `int`, полученных из элементов через `mapper`                                       | `stream.collect(Collectors.averagingInt(Person::getScore))`                                                     |
| `static <T> Collector<T, ?, Double> averagingLong(ToLongFunction<? super T> mapper)`                                                                                                                                | Вычисление среднего значения `long`, полученных из элементов через `mapper`                                      | `stream.collect(Collectors.averagingLong(Person::getAge))`                                                      |
| `static <T> Collector<T, ?, Double> averagingDouble(ToDoubleFunction<? super T> mapper)`                                                                                                                            | Вычисление среднего значения `double`, полученных из элементов через `mapper`                                    | `stream.collect(Collectors.averagingDouble(Person::getHeight))`                                                 |

### Базовые коллекции
```java
// Список
List<String> list = stream.collect(Collectors.toList());
List<String> synchronizedList = stream.collect(Collectors.toCollection(
    () -> Collections.synchronizedList(new ArrayList<>())));

// Множество
Set<String> set = stream.collect(Collectors.toSet());
Set<String> treeSet = stream.collect(Collectors.toCollection(TreeSet::new));

// Конкретные реализации
Queue<String> queue = stream.collect(Collectors.toCollection(PriorityQueue::new));
Deque<String> deque = stream.collect(Collectors.toCollection(ArrayDeque::new));
```

### Map
```java
// Простая Map
Map<Long, String> idToName = people.stream()
    .collect(Collectors.toMap(Person::getId, Person::getName));

// Map с разрешением конфликтов (берем последнее значение)
Map<String, Person> latestPersonByName = people.stream()
    .collect(Collectors.toMap(Person::getName, Function.identity(), (oldVal, newVal) -> newVal));

// Map с суммированием при конфликтах
Map<String, Double> totalSalesByProduct = sales.stream()
    .collect(Collectors.toMap(Sale::getProduct, Sale::getAmount, Double::sum));

// Map с конкретной реализацией (TreeMap с компаратором)
Map<String, Person> sortedMap = people.stream()
    .collect(Collectors.toMap(Person::getName, Function.identity(), 
        (oldVal, newVal) -> newVal, 
        () -> new TreeMap<>(String.CASE_INSENSITIVE_ORDER)));

// ConcurrentMap
ConcurrentMap<String, Person> concurrentMap = people.stream()
    .collect(Collectors.toConcurrentMap(Person::getName, Function.identity()));
```

### String
```java
// Простая конкатенация
String result = stream.collect(Collectors.joining()); // "abc"

// С разделителем
String withComma = stream.collect(Collectors.joining(", ")); // "a, b, c"

// Полный формат
String formatted = stream.collect(Collectors.joining(", ", "[", "]")); // "[a, b, c]"

// С фильтрацией null
String safeJoin = stream.filter(Objects::nonNull).collect(Collectors.joining());

// Многострочный результат
String multiline = stream.collect(Collectors.joining("\n"));
```

### Группировка
```java
// Простая группировка
Map<String, List<Person>> peopleByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity));

// Группировка с изменением типа значения
Map<String, Set<Person>> peopleByCitySet = people.stream()
    .collect(Collectors.groupingBy(Person::getCity, Collectors.toSet()));

// Группировка с подсчетом
Map<String, Long> countByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity, Collectors.counting()));

// Группировка с суммированием
Map<String, Double> totalSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment, 
             Collectors.summingDouble(Employee::getSalary)));

// Группировка с averaging
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment,
             Collectors.averagingDouble(Employee::getSalary)));

// Группировка с маппингом
Map<String, List<String>> namesByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
             Collectors.mapping(Person::getName, Collectors.toList())));

// Группировка с фильтрацией
Map<String, List<Person>> adultsByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
             Collectors.filtering(p -> p.getAge() >= 18, Collectors.toList())));

// Многоуровневая группировка
Map<String, Map<String, List<Person>>> peopleByCityAndGender = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
             Collectors.groupingBy(Person::getGender)));

// Группировка с конкретной Map
Map<String, List<Person>> sortedGrouping = people.stream()
    .collect(Collectors.groupingBy(Person::getCity, TreeMap::new, Collectors.toList()));
```

### Partitioning - разделение на две группы
```java
// Простое разделение
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() >= 18));

// Partitioning с подсчетом
Map<Boolean, Long> countPartition = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() >= 18, Collectors.counting()));

// Partitioning с группировкой
Map<Boolean, Map<String, List<Person>>> complexPartition = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() >= 18,
             Collectors.groupingBy(Person::getCity)));
```

### Агрегация и статистика
```java
// Подсчет
Long count = stream.collect(Collectors.counting());

// Суммирование
Integer totalLength = strings.stream()
    .collect(Collectors.summingInt(String::length));

Double totalSalary = employees.stream()
    .collect(Collectors.summingDouble(Employee::getSalary));

// Среднее значение
Double avgSalary = employees.stream()
    .collect(Collectors.averagingDouble(Employee::getSalary));

// Полная статистика
IntSummaryStatistics stats = strings.stream()
    .collect(Collectors.summarizingInt(String::length));
// stats.getCount(), stats.getSum(), stats.getMin(), stats.getMax(), stats.getAverage()

// Минимальный/максимальный с компаратором
Optional<Person> oldest = people.stream()
    .collect(Collectors.maxBy(Comparator.comparing(Person::getAge)));

Optional<Person> youngest = people.stream()
    .collect(Collectors.minBy(Comparator.comparing(Person::getAge)));
```

### Комбинированные коллекторы
```java
// Группировка с ограничением размера
Map<String, List<Person>> top3ByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
             Collectors.collectingAndThen(Collectors.toList(),
                 list -> list.stream().limit(3).collect(Collectors.toList()))));

// Группировка с joining
Map<String, String> namesByCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity,
             Collectors.mapping(Person::getName, 
                 Collectors.joining(", "))));

// Вложенная агрегация
Map<String, DoubleSummaryStatistics> salaryStatsByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment,
             Collectors.summarizingDouble(Employee::getSalary)));
```

### Кастомные сборки с 3-аргументным `collect()`

Трехаргументный метод stream().collect(...):

`supplier`: `Supplier<U>` - Функция, создающая новый пустой контейнер аккумуляции
`accumulator`: `BiConsumer<U, T>` - Функция, которая принимает контейнер аккумуляции и один элемент потока, и изменяет этот контейнер.
`combiner`: `BiConsumer<U, U>` - Функция, которая принимает два контейнера аккумуляции и объединяет их содержимое.

```java
// Сбор в StringBuilder
StringBuilder sb = stream.collect(
    StringBuilder::new,        // supplier
    StringBuilder::append,     // accumulator  
    StringBuilder::append      // combiner
);

// Сбор в кастомную структуру
Map<String, Integer> wordCount = words.stream().collect(
    HashMap::new,                                     // supplier
    (map, word) -> map.merge(word, 1, Integer::sum), // accumulator
    HashMap::putAll                                   // combiner
);

// Параллельная обработка
List<String> threadSafeList = stream.collect(
    () -> Collections.synchronizedList(new ArrayList<>()), // supplier
    List::add,                                            // accumulator
    List::addAll                                          // combiner
);

// Сбор с дополнительной логикой
Set<String> uniqueWords = stream.collect(
    HashSet::new,
    (set, word) -> {
        if (word != null && !word.trim().isEmpty()) {
            set.add(word.toLowerCase());
        }
    },
    Set::addAll
);
```

### Специальные коллекторы
```java
// reducing - аналог reduce() но как коллектор
Optional<String> longest = strings.stream()
    .collect(Collectors.reducing((s1, s2) -> s1.length() > s2.length() ? s1 : s2));

/*
teeing позволяет однократно пройти по потоку employees, выполнить две независимые операции агрегации (нахождение среднего и максимума) и объединить их результаты в нужную структуру данных с помощью функции. 
Это может быть эффективнее, чем проходить по потоку дважды, вызывая каждый коллектор отдельно.
 */

Map<String, Object> stats = employees.stream()
        .collect(Collectors.teeing(
                // 1. Первый коллектор
                Collectors.averagingDouble(Employee::getSalary),
                // 2. Второй коллектор
                Collectors.maxBy(Comparator.comparing(Employee::getSalary)),
                // 3. Функция объединения (finisher)
                (avg, max) -> Map.of("average", avg, "max", max.orElse(null))
        ));

 /*
- Collectors.averagingDouble(Employee::getSalary):
    Это первый "внутренний" коллектор.
    Он проходит по потоку employees и вычисляет среднюю зарплату.
    Его результат — Double (среднее значение).
- Collectors.maxBy(Comparator.comparing(Employee::getSalary)):
    Это второй "внутренний" коллектор.
    Он проходит по потоку employees и находит сотрудника с максимальной зарплатой.
    Его результат — Optional<Employee> (самый высокооплачиваемый сотрудник, или Optional.empty(), если поток пуст).
- (avg, max) -> Map.of("average", avg, "max", max.orElse(null)):
    Это функция объединения (finisher function). Она принимает два результата — avg (результат первого коллектора) и max (результат второго коллектора).
    avg — это Double.
    max — это Optional<Employee>.
    Эта функция объединяет эти два результата в один объект — Map<String, Object>, содержащий пары "ключ-значение" для средней и максимальной зарплаты.
 */

// filtering - фильтрация внутри коллектора (Java 9+)
List<String> nonEmpty = strings.stream()
    .collect(Collectors.filtering(s -> !s.isEmpty(), Collectors.toList()));

// flatMapping - flatMap внутри коллектора (Java 9+)
Map<String, List<String>> tagsByCategory = articles.stream()
    .collect(Collectors.groupingBy(Article::getCategory,
             Collectors.flatMapping(a -> a.getTags().stream(), Collectors.toList())));
```
</details>

<br>

<details>
    <summary>
        <b>Parallel Streams</b>
    </summary>

# Parallel Streams

> `parallelStream()` разбивает источник на части и обрабатывает каждую часть в отдельном потоке пула Fork/Join и потом объединяет результат (если нужно).
>
> ⚠️ `parallelStream()` не гарантирует порядок обработки и не ускоряет всё подряд. 
> Выигрыш зависит от объема данных, стоимости операций и характеристик системы.

## Создание параллельного стрима

```java
List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);

// Прямо из коллекции
Stream<Integer> parallel = data.parallelStream();

// Или переключить обычный стрим
Stream<Integer> alsoParallel = data.stream().parallel();
```

### `.parallel()` и `.sequential()` — мутирующие (меняют внутреннее состояние стрима, не возвращают копию)

### Последнее вызванное определение (parallel/sequential) — решает.

```java
// Практический пример с обработкой:
long count = data.parallelStream()  // начинаем как параллельный
    .filter(x -> x > 2)            // параллельная фильтрация
    .sequential()                  // переключаем на последовательный
    .map(x -> x * 2)               // последовательное отображение
    .count();                      // последовательный подсчет
```

> Обычно не стоит переключать режим в середине цепочки операций, так как это может запутать и не даст ожидаемых преимуществ в производительности. 
> Лучше явно определиться в начале, какой стрим вам нужен

```java
// Четко и понятно:
Stream<Integer> parallelStream = data.parallelStream();
Stream<Integer> sequentialStream = data.stream();
```

## Параллельные операции

### Большинство промежуточных и терминальных операций могут выполняться параллельно:

- `filter`, `map`, `flatMap`, `distinct`, `sorted` (с оговорками), `reduce`, `collect`, `forEach`, `forEachOrdered`, `anyMatch`, и т.д.

```java
List<String> result = data.parallelStream()
    .filter(x -> x > 2)                 // параллельная фильтрация
    .map(x -> "item-" + x)              // параллельное преобразование
    .collect(Collectors.toList());      // сбор — тоже в общем случае параллелится
```

> collect() с иммутабельными коллекторами (например, toList(), toSet()) безопасен. Коллекторы вроде Collectors.toList() — потокобезопасны внутри, но не гарантируют порядок.

### Пример с `reduce`:  

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);

// Параллельный reduce - как это работает внутри:
Integer sum = numbers.parallelStream()
    .reduce(0, (a, b) -> a + b); // identity, accumulator

/*
Как это выполняется:
1. Стрим разбивается на части (например, [1,2,3,4] и [5,6,7,8])
2. Каждая часть обрабатывается в своем потоке:
   - Поток 1: reduce(0, 1+2+3+4) = 10
   - Поток 2: reduce(0, 5+6+7+8) = 26
3. Результаты частей объединяются: 10 + 26 = 36
4. Итог: 36 (корректный результат)
*/
```

Требования к аккумулятору для parallel reduce:

- Ассоциативность: `(a ⊕ b) ⊕ c` = `a ⊕ (b ⊕ c)`
- Identity элемент: `identity ⊕ a` = `a`

### Пример неассоциативной операции:

```java
// Вычитание НЕ ассоциативно: (a - b) - c ≠ a - (b - c)
// Этот код даст разный результат при каждом запуске в параллельном режиме
int wrong = numbers.parallelStream()
                .reduce(0, (a, b) -> a - b); // НЕПРЕДСКАЗУЕМО!
```

### Пример с `collect`:

```java
List<String> words = Arrays.asList("a", "b", "c", "d", "e", "f");

// Параллельный collect в ArrayList
List<String> result = words.parallelStream()
    .collect(
        ArrayList::new,          // Supplier - создает новый контейнер
        ArrayList::add,          // Accumulator - добавляет элемент
        ArrayList::addAll        // Combiner - объединяет контейнеры
    );

/*
1. Стрим разбивается на части
2. Для каждой части:
   - Supplier создает новый ArrayList
   - Accumulator добавляет элементы в этот список
3. Combiner объединяет списки из разных потоков
*/
```

### Проблема с небезопасными коллекторами:

```java
// НЕПРАВИЛЬНО! ConcurrentModificationException гарантирован
List<String> badResult = words.parallelStream()
                .collect(
                        () -> new ArrayList<>(),
                        (list, item) -> {
                            if (item.equals("c")) list.add("C");
                            else list.add(item);
                        },
                        (list1, list2) -> list1.addAll(list2)
                );
```

```java
List<Integer> sharedList = new ArrayList<>();  // НЕ потокобезопасен!

// race condition

data.parallelStream()
    .map(x -> x * 2)
    .forEach(sharedList::add);  // ⚠️ Потенциально некорректный результат!

// ArrayList.add() внутри:
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Проблема 1: гонка за size
    elementData[size++] = e;           // Проблема 2: гонка за size и elementData
    return true;
}

// Представьте, что два потока одновременно вызывают add():
// Поток 1: size = 0, хочет записать в elementData[0]
// Поток 2: size = 0, хочет записать в elementData[0]
// Оба увеличивают size до 1, но один элемент перезапишет другой!
```

## Правильные подходы для параллельного collect

### Использовать потокобезопасные коллекторы

```java
// Collectors.toList() уже оптимизирован для параллелизма
List<String> safe1 = words.parallelStream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Collectors.groupingByConcurrent() для параллельной группировки
Map<Integer, List<String>> safe2 = words.parallelStream()
    .collect(Collectors.groupingByConcurrent(String::length));

```

```java
// Collectors.collectingAndThen() - собираем в список, затем делаем его неизменяемым
List<String> unmodifiableList = items.stream()
        .collect(Collectors.collectingAndThen(
                Collectors.toList(),           // 1. Собираем в обычный список
                Collections::unmodifiableList  // 2. Преобразуем в неизменяемый
        ));

// Эквивалент без collectingAndThen (более многословно):
List<String> tempList = items.stream()
        .collect(Collectors.toList());     // 1. Собираем
List<String> result = Collections.unmodifiableList(tempList); // 2. Преобразуем
```

```java
List<Integer> result = data.parallelStream()
        .map(x -> x * 2)
        .collect(Collectors.toCollection(ArrayList::new));  // ✅ Потокобезопасный / изолированный сбор
```

```java
// Collectors.toCollection(ArrayList::new) внутри:
Supplier<ArrayList> supplier = ArrayList::new;  // 1. Каждый поток создает СВОЙ список
BiConsumer<ArrayList, Integer> accumulator = ArrayList::add;  // 2. Добавляет в свой список
BinaryOperator<ArrayList> combiner = (list1, list2) -> {  // 3. В конце объединяет списки
    list1.addAll(list2);
    return list1;
};

// Таким образом:
// - Каждый поток работает со своим изолированным ArrayList
// - Нет гонки за ресурсы
// - В конце списки безопасно объединяются
```

### Если нужно получить мутабельную коллекцию:

### Вариант 1: Потокобезопасные коллекции в forEach

```java
// Можно использовать потокобезопасные коллекции в forEach
CopyOnWriteArrayList<Integer> safeList = new CopyOnWriteArrayList<>();
data.parallelStream()
    .map(x -> x * 2)
    .forEach(safeList::add);  // ✅ Безопасно, но НЕ ЭФФЕКТИВНО
    
// CopyOnWriteArrayList дорогой для частых записей
// Лучше использовать только если действительно нужно
```

### Вариант 2: Concurrent коллекции с коллектором

```java
// Более эффективно: использовать специальный коллектор
ConcurrentHashMap<Integer, Long> map = data.parallelStream()
                .collect(Collectors.groupingByConcurrent(
                        x -> x % 2,  // ключ: четность
                        Collectors.counting()
                ));
```

### Вариант 3: Atomic переменные для счетчиков

```java
AtomicInteger counter = new AtomicInteger(0);
data.parallelStream()
    .forEach(x -> {
        if (x > 10) {
            counter.incrementAndGet();  // ✅ Безопасно
        }
    });
```

## Порядок элементов
- `.forEach()` — не сохраняет порядок
- `.forEachOrdered()` — сохраняет порядок встречи в источнике, но снижает производительность (синхронизирует вывод).
- `.sorted().forEachOrdered()` — гарантирует упорядоченный вывод.

```java
data.parallelStream()
    .sorted()                     // сортирует (может быть частично параллельно)
    .forEachOrdered(System.out::println);  // выводит в правильном порядке
```

> ⚠️ `.sorted()` в параллельном стриме требует дополнительных шагов: каждый поток сортирует свою часть, затем происходит слияние отсортированных частей (merge). Это создает оверхед по памяти и CPU.

```java
List<Integer> data = Arrays.asList(5, 8, 1, 3, 9, 2, 7);

// ❌ Плохо: сортировка в параллельном стриме
List<Integer> slow = data.parallelStream()
    .sorted()                    // ⚠️ Каждый поток сортирует свою часть + merge
    .collect(Collectors.toList());

// ✅ Лучше: сортировать ДО параллельной обработки
List<Integer> fast = data.stream()
    .sorted()                    // Быстрая последовательная сортировка
    .parallel()                  // Затем параллельная обработка
    .map(x -> x * 2)             // Далее операции без сортировки
    .collect(Collectors.toList());
``` 

### Если нужен limit() с сортировкой, используйте Stream.of() с явным указанием последовательности операций.

```java
List<Integer> data = Arrays.asList(5, 8, 1, 3, 9, 2, 7, 4, 6);

// ❌ Плохо: limit() после parallel() - может работать неоптимально
// Сортирует все 9 элементов параллельно, затем берет 3 первых
List<Integer> suboptimal = data.parallelStream()
    .sorted()
    .limit(3)                    // ⚠️ Все равно сортируются все элементы!
    .collect(Collectors.toList());

// ✅ Лучше: сначала отсортировать, потом ограничить, потом паралеллизовать
// Stream.of() помогает явно управлять порядком операций
List<Integer> optimal = Stream.of(data)
    .flatMap(List::stream)       // Разворачиваем список
    .sorted()                    // Сортируем ПОСЛЕДОВАТЕЛЬНО
    .limit(3)                    // Берем только нужные
    .parallel()                  // Теперь можно паралеллизовать
    .map(x -> x * 2)             // Дальнейшая параллельная обработка
    .collect(Collectors.toList());
```

## Parallel Streams и Fork/Join Pool

### Как работает под капотом

> parallelStream() использует общий пул потоков ForkJoinPool.commonPool() — один на все приложение. 
> Размер обычно равен количеству ядер процессора минус 1.

```java
// Проверим размер пула по умолчанию
int poolSize = ForkJoinPool.getCommonPoolParallelism();
System.out.println("Размер commonPool: " + poolSize); // Например, 7 для 8-ядерного CPU

// Все parallelStream() делят один пул
List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
long sum = data.parallelStream()
    .map(x -> x * 2)
    .reduce(0, Integer::sum); // Использует commonPool
```

### Проблема: один пул на всех

> Что плохо? Если у вас несколько `parallelStream()` работают одновременно, они конкурируют за одни и те же потоки. 
> Это может привести к деградации производительности.

### Кастомный пул потоков

```java
// Создаем кастомный пул с 4 потоками
ForkJoinPool customPool = new ForkJoinPool(4);

// Запускаем задачу в своем пуле
int result = customPool.submit(() -> 
    data.parallelStream()               // ⚠️ Важно: этот parallelStream будет использовать customPool
        .map(x -> {                     
            System.out.println(Thread.currentThread().getName());
            return x * 2;
        })
        .reduce(0, Integer::sum)
).join(); // Ждем завершения

System.out.println("Результат: " + result);
customPool.shutdown();
```

> Когда вы запускаете `parallelStream()` внутри задачи `ForkJoinPool`, он использует тот пул, в котором выполняется задача.

### ⚠️ Вложенный параллелизм

```java
// ❌ ПРОБЛЕМА: nested parallelStream() может "утечь" в commonPool
List<List<Integer>> matrix = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5, 6),
    Arrays.asList(7, 8, 9)
);

customPool.submit(() -> 
    matrix.parallelStream()                 // Использует customPool
        .map(row -> row.parallelStream()    // ⚠️ Вложенный parallelStream() может использовать commonPool!
                       .sum()              
        )
        .forEach(System.out::println)
).join();
```

- Внутренний `parallelStream()` может начать использовать commonPool
- Потоки из `customPool` ждут завершения задач в `commonPool`
- Может возникнуть взаимная блокировка (deadlock) или деградация производительности

### Избегайте вложенного parallelStream()

```java
// ✅ Лучше: преобразовать вложенность в плоскую структуру
List<Integer> flatList = matrix.parallelStream()  // Только один parallelStream
    .flatMap(List::stream)                        // "Разворачиваем" вложенные списки
    .collect(Collectors.toList());                // Теперь обрабатываем как один поток

// ✅ Альтернатива: явно управлять параллелизмом
customPool.submit(() -> 
    matrix.stream()                     // Последовательный внешний стрим
        .map(row -> row.stream()        // Последовательный внутренний стрим
                       .sum()
        )
        .parallel()                     // Параллелизм на верхнем уровне
        .forEach(System.out::println)
).join();
```

- Один `parallelStream()` лучше нескольких — избегайте вложенного параллелизма
- Кастомный пул для изоляции — используйте для критичных по времени задач
- Не смешивать пулы — если используете `customPool`, убедитесь, что все `parallelStream()` внутри него тоже используют его
- Избегать блокирующих операций — в `parallelStream()` не должно быть I/O или синхронизации

```java
// ✅ Идеальный паттерн: один parallelStream с кастомным пулом
ForkJoinPool dedicatedPool = new ForkJoinPool(8);
dedicatedPool.submit(() -> 
    bigDataList.parallelStream()  // Только ОДИН parallelStream
        .map(this::processItem)    // Тяжелая операция
        .collect(Collectors.toList())
).join();
dedicatedPool.shutdown();
```
</details>