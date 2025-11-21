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

| Сигнатура метода                                                                         | Назначение                                                          | Пример                                       |
|:-----------------------------------------------------------------------------------------|:--------------------------------------------------------------------|:---------------------------------------------|
| `default Stream<E> stream()`                                                             | Создание последовательного потока из коллекции                      | `collection.stream()`                        |
| `default Stream<E> parallelStream()`                                                     | Создание параллельного потока из коллекции                          | `collection.parallelStream()`                |
| `static <T> Stream<T> of(T... values)`                                                   | Создание потока из набора значений                                  | `Stream.of("a", "b", "c")`                   |
| `static <T> Stream<T> ofNullable(T t)`                                                   | Создание потока из одного элемента или пустого потока, если `null`  | `Stream.ofNullable(nullableObject)`          |
| `static <T> Stream<T> empty()`                                                           | Создание пустого потока                                             | `Stream.empty()`                             |
| `static <T> Stream<T> generate(Supplier<T> s)`                                           | Создание бесконечного потока с помощью поставщика                   | `Stream.generate(() -> "element").limit(10)` |
| `static <T> Stream<T> iterate(T seed, UnaryOperator<T> f)`                               | Создание бесконечного потока с итеративным применением функции      | `Stream.iterate(0, n -> n + 2).limit(5)`     |
| `static <T> Stream<T> iterate(T seed, Predicate<? super T> hasNext, UnaryOperator<T> f)` | Создание конечного потока с условием остановки (Java 9+)            | `Stream.iterate(0, n -> n < 10, n -> n + 2)` |
| `static <T> Stream<T> stream(T[] array)`                                                 | Создание потока из массива для ссылочных типов                      | `Arrays.stream(array)`                       |
| `static IntStream stream(int[] array)`                                                   | Создание потока из массива для примитивных типов                    | `Arrays.stream(array)`                       |
| `static Stream<String> lines(Path path)`                                                 | Создание потока строк из файла                                      | `Files.lines(Paths.get("file.txt"))`         |
| `static IntStream range(int startInclusive, int endExclusive)`                           | Создание потока целых чисел в диапазоне [start, end)                | `IntStream.range(0, 5)` // 0,1,2,3,4         |
| `static IntStream rangeClosed(int startInclusive, int endInclusive)`                     | Создание потока целых чисел в диапазоне [start, end]                | `IntStream.rangeClosed(0, 5)` // 0,1,2,3,4,5 |
| `static <T> Stream<T> concat(Stream<? extends T> a, Stream<? extends T> b)`              | Конкатенация двух потоков в один                                    | `Stream.concat(stream1, stream2)`            |
| `static <T> Stream.Builder<T> builder()`                                                 | Создание потока через построитель                                   | `Stream.builder().add("a").add("b").build()` |
</details>

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
</details>

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
</details>

<details>
    <summary>
        <b>Метод collect() и коллекторы</b>
    </summary>

### Основная сигнатура: `<R, A> R collect(Collector<? super T, A, R> collector)`

- `T` - Тип элементов в потоке. Это тип объектов, которые проходят через `Stream<T>`. Например, если у вас `Stream<String>`, то `T` - это `String`.
- `A` - Тип аккумулятора. Это внутреннее состояние или контейнер, используемый коллектором для сбора элементов. Это может быть `StringBuilder` (для `joining`), `List` (для `toList`), `Map` (для `groupingBy`), `Integer` (для подсчета) или любой другой тип, который необходим для промежуточного хранения данных во время процесса сбора. Этот тип не виден напрямую в вызове `collect()`, но он определен внутри реализации `Collector`.
- `R` - Тип результата. Это конечный тип объекта, который будет возвращен методом `collect()`. Например, `List<String>`, `Map<K, V>`, `String`, `Integer` и т.д.

`Collector<? super T, A, R>:` - Это интерфейс, который определяет, как собирать элементы потока в конечный результат.

- `? super T`: Коллектор может принимать элементы типа `T` или его супертипы. Это обеспечивает ковариантность.
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
// Результат: "a,b,c," (можно обрезать в конце в реальном коде)
```
### Коллекторы (Collectors)

| Сигнатура метода | Назначение | Пример |
|:-----------------|:-----------|:-------|
| `static <T> Collector<T, ?, List<T>> toList()` | Сбор элементов в `List` (в JDK 16+ возвращаемый список неизменяем) | `stream.collect(Collectors.toList())` |
| `static <T> Collector<T, ?, List<T>> toUnmodifiableList()` | Сбор элементов в неизменяемый `List` | `stream.collect(Collectors.toUnmodifiableList())` |
| `static <T> Collector<T, ?, Set<T>> toSet()` | Сбор элементов в `Set` (в JDK 16+ возвращаемый сет неизменяем) | `stream.collect(Collectors.toSet())` |
| `static <T> Collector<T, ?, Set<T>> toUnmodifiableSet()` | Сбор элементов в неизменяемый `Set` | `stream.collect(Collectors.toUnmodifiableSet())` |
| `static <T, C extends Collection<T>> Collector<T, ?, C> toCollection(Supplier<C> collectionFactory)` | Сбор элементов в коллекцию, созданную через `collectionFactory` | `stream.collect(Collectors.toCollection(LinkedList::new))` |
| `static <T, K> Collector<T, ?, Map<K, List<T>>> groupingBy(Function<? super T, ? extends K> classifier)` | Группировка элементов по классификатору в `Map<K, List<T>>` | `stream.collect(Collectors.groupingBy(Person::getCity))` |
| `static <T, K, A, D> Collector<T, ?, Map<K, D>> groupingBy(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream)` | Группировка элементов по классификатору, с дополнительной обработкой (`downstream`) для значений | `stream.collect(Collectors.groupingBy(Person::getCity, Collectors.counting()))` |
| `static <T, K, D, A, M extends Map<K, D>> Collector<T, ?, M> groupingBy(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream)` | Группировка элементов по классификатору, с фабрикой карты и дополнительной обработкой значений | `stream.collect(Collectors.groupingBy(Person::getCity, LinkedHashMap::new, Collectors.toList()))` |
| `static <T, K> Collector<T, ?, ConcurrentMap<K, List<T>>> groupingByConcurrent(Function<? super T, ? extends K> classifier)` | Группировка элементов по классификатору в `ConcurrentMap<K, List<T>>` | `stream.collect(Collectors.groupingByConcurrent(Person::getCity))` |
| `static <T, K, A, D> Collector<T, ?, ConcurrentMap<K, D>> groupingByConcurrent(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream)` | Группировка элементов по классификатору в `ConcurrentMap`, с дополнительной обработкой значений | `stream.collect(Collectors.groupingByConcurrent(Person::getCity, Collectors.counting()))` |
| `static <T, K, D, A, M extends ConcurrentMap<K, D>> Collector<T, ?, M> groupingByConcurrent(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream)` | Группировка элементов по классификатору в `ConcurrentMap`, с фабрикой карты и дополнительной обработкой значений | `stream.collect(Collectors.groupingByConcurrent(Person::getCity, ConcurrentHashMap::new, Collectors.toList()))` |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper)` | Сбор элементов в `Map`, где ключ и значение определяются функциями | `stream.collect(Collectors.toMap(Person::getId, Person::getName))` |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper, BinaryOperator<U> mergeFunction)` | Сбор элементов в `Map`, с функцией слияния значений для дубликатов ключей | `stream.collect(Collectors.toMap(Person::getId, Person::getName, (name1, name2) -> name1))` |
| `static <T, K, U, M extends Map<K, U>> Collector<T, ?, M> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper, BinaryOperator<U> mergeFunction, Supplier<M> mapFactory)` | Сбор элементов в `Map`, с фабрикой карты, функцией слияния значений для дубликатов ключей | `stream.collect(Collectors.toMap(Person::getId, Person::getName, (name1, name2) -> name1, LinkedHashMap::new))` |
| `static <T, K, U> Collector<T, ?, Map<K, U>> toUnmodifiableMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper)` | Сбор элементов в неизменяемую `Map` | `stream.collect(Collectors.toUnmodifiableMap(Person::getId, Person::getName))` |
| `static Collector<CharSequence, ?, String> joining()` | Объединение строковых представлений элементов в строку | `Stream.of("a", "b", "c").collect(Collectors.joining())` // результат: "abc" |
| `static Collector<CharSequence, ?, String> joining(CharSequence delimiter)` | Объединение строковых представлений элементов в строку с разделителем | `Stream.of("a", "b", "c").collect(Collectors.joining(","))` // результат: "a,b,c" |
| `static Collector<CharSequence, ?, String> joining(CharSequence delimiter, CharSequence prefix, CharSequence suffix)` | Объединение строковых представлений элементов в строку с разделителем, префиксом и суффиксом | `Stream.of("a", "b", "c").collect(Collectors.joining(",", "[", "]"))` // результат: "[a,b,c]" |
| `static <T, A, R> Collector<T, A, R> collectingAndThen(Collector<T, A, R> downstream, Function<R, R> finisher)` | Применение дополнительной функции к результату другого коллектора | `stream.collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList))` |
| `static <T> Collector<T, ?, T> reducing(T identity, BinaryOperator<T> op)` | Свёртка элементов с начальным значением и бинарным оператором | `stream.collect(Collectors.reducing(0, Integer::sum))` |
| `static <T> Collector<T, ?, Optional<T>> reducing(BinaryOperator<T> op)` | Свёртка элементов с бинарным оператором, возвращает `Optional` | `stream.collect(Collectors.reducing(Integer::max))` |
| `static <T, U> Collector<T, ?, U> reducing(U identity, Function<? super T, ? extends U> mapper, BinaryOperator<U> op)` | Свёртка элементов, преобразованных через `mapper`, с начальным значением и оператором | `stream.collect(Collectors.reducing(0, String::length, Integer::sum))` |
| `static <T> Collector<T, ?, Optional<T>> minBy(Comparator<? super T> comparator)` | Поиск минимального элемента по компаратору | `stream.collect(Collectors.minBy(Comparator.naturalOrder()))` |
| `static <T> Collector<T, ?, Optional<T>> maxBy(Comparator<? super T> comparator)` | Поиск максимального элемента по компаратору | `stream.collect(Collectors.maxBy(Comparator.naturalOrder()))` |
| `static <T, A, R> Collector<T, ?, R> mapping(Function<? super T, ? extends A> mapper, Collector<A, ?, R> downstream)` | Преобразование элементов с последующей обработкой `downstream` коллектором | `stream.collect(Collectors.mapping(Person::getName, Collectors.toList()))` |
| `static <T> Collector<T, ?, Long> counting()` | Подсчёт количества элементов | `stream.collect(Collectors.counting())` |
| `static <T, U, A, R> Collector<T, ?, R> flatMapping(Function<? super T, ? extends Stream<? extends U>> mapper, Collector<? super U, A, R> downstream)` | Преобразование элементов в потоки, их плоское объединение и последующая обработка `downstream` коллектором | `stream.collect(Collectors.flatMapping(str -> Stream.of(str.split(" ")), Collectors.counting()))` |
| `static <T> Collector<T, ?, Integer> summingInt(ToIntFunction<? super T> mapper)` | Вычисление суммы `int` значений, полученных из элементов через `mapper` | `stream.collect(Collectors.summingInt(Person::getScore))` |
| `static <T> Collector<T, ?, Long> summingLong(ToLongFunction<? super T> mapper)` | Вычисление суммы `long` значений, полученных из элементов через `mapper` | `stream.collect(Collectors.summingLong(Person::getAge))` |
| `static <T> Collector<T, ?, Double> summingDouble(ToDoubleFunction<? super T> mapper)` | Вычисление суммы `double` значений, полученных из элементов через `mapper` | `stream.collect(Collectors.summingDouble(Person::getSalary))` |
| `static <T> Collector<T, ?, Double> averagingInt(ToIntFunction<? super T> mapper)` | Вычисление среднего значения `int`, полученных из элементов через `mapper` | `stream.collect(Collectors.averagingInt(Person::getScore))` |
| `static <T> Collector<T, ?, Double> averagingLong(ToLongFunction<? super T> mapper)` | Вычисление среднего значения `long`, полученных из элементов через `mapper` | `stream.collect(Collectors.averagingLong(Person::getAge))` |
| `static <T> Collector<T, ?, Double> averagingDouble(ToDoubleFunction<? super T> mapper)` | Вычисление среднего значения `double`, полученных из элементов через `mapper` | `stream.collect(Collectors.averagingDouble(Person::getHeight))` |

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