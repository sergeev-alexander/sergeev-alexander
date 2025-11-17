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

| Метод (сигнатура)                 | Назначение                                             | Пример                                                   |
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









</details>