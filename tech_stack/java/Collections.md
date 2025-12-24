# Java Collections Framework

> Java Collections Framework — это унифицированная архитектура для представления и манипулирования группами объектов.

## Иерархия интерфейсов

```text
public interface Iterable<T>
 │
 └── public interface Collection<E> extends Iterable<E>
      │
      ├── public interface List<E> extends Collection<E>, SequencedCollection<E>
      │
      ├── public interface Set<E> extends Collection<E>
      │    │
      │    ├── public interface SortedSet<E> extends Set<E>
      │    │    │
      │    │    └── public interface NavigableSet<E> extends SortedSet<E>
      │    │     
      │    └── public interface SequencedSet<E> extends Set<E>, SequencedCollection<E>
      │
      ├── public interface Queue<E> extends Collection<E>
      │    │
      │    ├── public interface Deque<E> extends Queue<E>, SequencedCollection<E>
      │    │
      │    ├── public interface BlockingQueue<E> extends Queue<E>
      │    │
      │    └── public interface BlockingDeque<E> extends Deque<E>, BlockingQueue<E>
      │
      └── public interface SequencedCollection<E> extends Collection<E>
```

<details>
    <summary>
        <b>Основные интерфейсы</b>
    </summary>

### `Iterable`

<details>
    <summary>
        <code>Iterable&lt;T&gt;</code>
    </summary>

# `public interface Iterable<T>`

> Позволяет объекту использоваться в цикле `for-each`

- `Iterator<T> iterator()` — возвращает итератор для внешней итерации
- `default void forEach(Consumer<? super T> action)` — внутренняя итерация (через `for-each`)
- `default Spliterator<T> spliterator()` — для ленивой и параллельной итерации (используется в `Stream`)

</details>

### `Iterable -> Collection`

<details>
    <summary>
        <code>Collection&lt;T&gt; extends Iterable&lt;T&gt;</code>
    </summary>

# `public interface Collection<E> extends Iterable<E>`

> Корневой интерфейс для групп объектов (коллекций). Не гарантирует порядок, уникальность или допустимость `null` — это определяют подтипы

- `int size()` — количество элементов
- `boolean isEmpty()` — пуста ли коллекция
- `boolean contains(Object o)` — есть ли элемент (по `equals`)
- `Iterator<E> iterator()` — итератор по элементам
- `Object[] toArray()` / `<T> T[] toArray(T[] a)` — преобразование в массив
- `boolean add(E e)` — добавить элемент (возвращает `true`, если изменилась)
- `boolean remove(Object o)` — удалить первое вхождение
- `boolean containsAll(Collection<?> c)` — содержатся ли все элементы
- `boolean addAll(Collection<? extends E> c)` — добавить все элементы
- `boolean removeAll(Collection<?> c)` — удалить все совпадающие
- `boolean retainAll(Collection<?> c)` — оставить только совпадающие
- `void clear()` — удалить все элементы
- `boolean equals(Object o)` — сравнение по содержимому (обычно порядок и `equals`)
- `int hashCode()` — хэш по содержимому
- `default Stream<E> stream()` / `parallelStream()` — потоки элементов

</details>

### `Iterable -> Collection -> List`

<details>
    <summary>
        <code>List&lt;E&gt; extends Collection&lt;E&gt;</code>
    </summary>

# `public interface List<E> extends Collection<E>`

> Упорядоченная коллекция (последовательность). Допускает дубликаты и `null`. Элементы имеют индексы от `0` до `size()-1`

- `E get(int index)` — получить элемент по индексу
- `E set(int index, E element)` — заменить элемент, вернуть старый
- `void add(int index, E element)` — вставить элемент (сдвиг вправо)
- `E remove(int index)` — удалить по индексу, вернуть значение
- `int indexOf(Object o)` / `lastIndexOf(Object o)` — индекс первого / последнего вхождения (или `-1`)
- `ListIterator<E> listIterator()` — двунаправленный итератор с возможностью модификации
- `List<E> subList(int from, int to)` — *представление* части списка (изменения синхронизированы с оригиналом)
- `boolean add(E e)` — добавить в конец
- `boolean equals(Object o)` — сравнивает по порядку и `equals` элементов
- `int hashCode()` — вычисляется как `e1 * 31^(n-1) + e2 * 31^(n-2) + … + en`

</details>

### `Iterable -> Collection -> Set`

<details>
    <summary>
        <code>Set&lt;E&gt; extends Collection&lt;E&gt;</code>
    </summary>

# `public interface Set<E> extends Collection<E>`

> Коллекция без дубликатов. Элементы уникальны по `equals()`/`hashCode()`. Порядок не гарантируется (если не указан в подтипе)

- `boolean add(E e)` — добавляет, только если такого элемента ещё нет; возвращает `true`, если добавлен
- `boolean remove(Object o)` — удаляет элемент (по `equals`)
- `boolean contains(Object o)` — проверяет наличие
- `int size()` / `isEmpty()` — стандартные запросы
- `Iterator<E> iterator()` — порядок зависит от реализации
- `boolean equals(Object o)` — два `Set` равны, если содержат одни и те же элементы (порядок игнорируется)
- `int hashCode()` — сумма `hashCode()` всех элементов

</details>

### `Iterable -> Collection -> Set -> SequencedSet`

<details>
    <summary>
        <code>SequencedSet&lt;E&gt; extends Set&lt;E&gt;, SequencedCollection&lt;E&gt;</code>
    </summary>

# `public interface SequencedSet<E> extends Set<E>, SequencedCollection<E>`

> Упорядоченное множество с чётко определённым началом и концом. Порядок — порядок вставки (как в `LinkedHashSet`), а не сортировка.

- Наследует все методы `Set<E>` (уникальность по `equals`/`hashCode`)
- Наследует методы `SequencedCollection<E>`:
    - `E getFirst()` / `getLast()`
    - `void addFirst(E)` / `addLast(E)`
    - `E removeFirst()` / `removeLast()`
    - `SequencedSet<E> reversed()` — view в обратном порядке

> ⚠️ `addFirst(e)` и `addLast(e)` добавляют элемент **только если его ещё нет** (соблюдается контракт `Set`).  
> Если элемент уже присутствует, он **не дублируется**, и его позиция **не меняется** (в отличие от `Deque`, где `addFirst` всегда вставляет в начало).

</details>

### `Iterable -> Collection -> Set -> SortedSet`

<details>
    <summary>
        <code>SortedSet&lt;E&gt; extends Set&lt;E&gt;</code>
    </summary>

# `public interface SortedSet<E> extends Set<E>`

> Упорядоченное множество: элементы хранятся в отсортированном порядке (естественном или заданном `Comparator`)

- `Comparator<? super E> comparator()` — возвращает компаратор или `null`, если используется естественный порядок
- `E first()` — возвращает первый (наименьший) элемент
- `E last()` — возвращает последний (наибольший) элемент
- `SortedSet<E> headSet(E toElement)` — view на подмножество строго меньше `toElement`
- `SortedSet<E> tailSet(E fromElement)` — view на подмножество больше или равно `fromElement`
- `SortedSet<E> subSet(E fromElement, E toElement)` — view на подмножество в диапазоне `[fromElement, toElement)`

> Все view (`headSet`, `tailSet`, `subSet`) синхронизированы с оригиналом.  
> Поведение не определено при модификации через итератор, если порядок нарушается.

</details>

### `Iterable -> Collection -> Set -> SortedSet -> NavigableSet`

<details>
    <summary>
        <code>NavigableSet&lt;E&gt; extends SortedSet&lt;E&gt;</code>
    </summary>

# `public interface NavigableSet<E> extends SortedSet<E>`

> Расширяет `SortedSet`: поддержка поиска ближайших элементов и двунаправленной навигации

- `E lower(E e)` — наибольший элемент **строго меньше** `e`, или `null`
- `E floor(E e)` — наибольший элемент **меньше или равен** `e`, или `null`
- `E ceiling(E e)` — наименьший элемент **больше или равен** `e`, или `null`
- `E higher(E e)` — наименьший элемент **строго больше** `e`, или `null`
- `E pollFirst()` / `pollLast()` — удаляют и возвращают первый / последний элемент, или `null`, если пусто
- `NavigableSet<E> descendingSet()` — view в обратном порядке (изменения синхронизированы)
- `Iterator<E> descendingIterator()` — итератор в обратном порядке
- `NavigableSet<E> headSet(E toElement, boolean inclusive)` — view до `toElement` (включительно/нет)
- `NavigableSet<E> tailSet(E fromElement, boolean inclusive)` — view от `fromElement` (включительно/нет)
- `NavigableSet<E> subSet(E from, boolean incFrom, E to, boolean incTo)` — view с гибкими границами

> Все `*Set`-методы возвращают **views**, а не копии.

### Ключевое отличие от Deque: в SequencedSet, как и в любом Set, дубликаты запрещены, поэтому `addFirst` / `addLast` — это условная вставка:
 
```java
SequencedSet<String> set = LinkedHashSet.newLinkedHashSet(); // Java 21+
set.add("a"); 
set.add("b");

set.addFirst("a"); // не добавит дубликат, порядок остаётся: [a, b]
```
</details>

### `Iterable -> Collection -> Queue`

<details>
    <summary>
        <code>Queue&lt;E&gt; extends Collection&lt;E&gt;</code>
    </summary>

# `public interface Queue<E> extends Collection<E>`

> Коллекция, оптимизированная под обработку «головы»: вставка в хвост, извлечение из головы (FIFO по умолчанию, но не обязательно)

- `boolean add(E e)` — добавить в конец; выбрасывает `IllegalStateException`, если ограничена и полна
- `boolean offer(E e)` — добавить в конец; возвращает `false` при невозможности (например, очередь заполнена)
- `E remove()` — извлечь и удалить голову; выбрасывает `NoSuchElementException`, если пуста
- `E poll()` — извлечь и удалить голову; возвращает `null`, если пуста
- `E element()` — получить голову без удаления; выбрасывает `NoSuchElementException`, если пуста
- `E peek()` — получить голову без удаления; возвращает `null`, если пуста

</details>

### `Iterable -> Collection -> Queue -> Deque`

<details>
    <summary>
        <code>Deque&lt;E&gt; extends Queue&lt;E&gt;, SequencedCollection&lt;E&gt;</code>
    </summary>

# `public interface Deque<E> extends Queue<E>, SequencedCollection<E>`

> Двусторонняя очередь: поддерживает вставку и извлечение с обоих концов. Реализует `Queue` (голова = начало) и `SequencedCollection`.

- `boolean add(E e)` — добавить в конец; `IllegalStateException`, если ограничена и полна
- `boolean offer(E e)` — добавить в конец; возвращает `false`, если невозможно
- `E remove()` — извлечь и удалить начало; `NoSuchElementException`, если пуст
- `E poll()` — извлечь и удалить начало; возвращает `null`, если пуст
- `E element()` — прочитать начало без удаления; `NoSuchElementException`, если пуст
- `E peek()` — прочитать начало без удаления; возвращает `null`, если пуст

- `void addFirst(E e)` — добавить в начало; `IllegalStateException`, если ограничена и полна
- `boolean offerFirst(E e)` — добавить в начало; возвращает `false`, если невозможно
- `E removeFirst()` — извлечь и удалить начало; `NoSuchElementException`, если пуст
- `E pollFirst()` — извлечь и удалить начало; возвращает `null`, если пуст
- `E getFirst()` — прочитать начало без удаления; `NoSuchElementException`, если пуст
- `E peekFirst()` — прочитать начало без удаления; возвращает `null`, если пуст

- `void addLast(E e)` — добавить в конец; `IllegalStateException`, если ограничена и полна
- `boolean offerLast(E e)` — добавить в конец; возвращает `false`, если невозможно
- `E removeLast()` — извлечь и удалить конец; `NoSuchElementException`, если пуст
- `E pollLast()` — извлечь и удалить конец; возвращает `null`, если пуст
- `E getLast()` — прочитать конец без удаления; `NoSuchElementException`, если пуст
- `E peekLast()` — прочитать конец без удаления; возвращает `null`, если пуст

- `void push(E e)` — как `addFirst(e)`
- `E pop()` — как `removeFirst()`; `NoSuchElementException`, если пуст
- `Iterator<E> descendingIterator()` — итератор от конца к началу
- `SequencedCollection<E> reversed()` — view в обратном порядке (наследуется от `SequencedCollection`)
</details>

### `Iterable -> Collection -> Queue -> BlockingQueue`

<details>
    <summary>
        <code>BlockingQueue&lt;E&gt; extends Queue&lt;E&gt;</code>
    </summary>

# `public interface BlockingQueue<E> extends Queue<E>`

> Потокобезопасная очередь с *блокирующими* операциями: при недоступности элемента/места — ждёт, пока станет возможно.

- `boolean add(E e)` — добавить в конец; `IllegalStateException`, если ограничена и полна
- `boolean offer(E e)` — добавить в конец; возвращает `false`, если невозможно
- `void put(E e)` — добавить в конец; **блокирует**, пока не освободится место
- `boolean offer(E e, long timeout, TimeUnit unit)` — как `offer`, но с таймаутом; возвращает `false` при истечении
- `E take()` — извлечь и удалить начало; **блокирует**, пока очередь не станет непустой
- `E poll(long timeout, TimeUnit unit)` — как `poll`, но ждёт до `timeout`; возвращает `null` при истечении
- `int remainingCapacity()` — оставшаяся ёмкость (может быть `Integer.MAX_VALUE` для неограниченных)
- `boolean remove(Object o)` — удаляет *одно* вхождение; возвращает `true`, если удалено
- `boolean contains(Object o)` — проверяет наличие
- `int drainTo(Collection<? super E> c)` — переносит все доступные элементы в коллекцию (атомарно, эффективно)
- `int drainTo(Collection<? super E> c, int maxElements)` — как выше, но не более `maxElements`

> ⚠️ `put` и `take` — **могут блокироваться бесконечно**.  
> Исключения `InterruptedException` не указаны явно — они объявлены в `throws`, но не нарушают контракт (как и во всех `blocking`-методах).

### Важно:
- BlockingQueue не добавляет новых исключений - только расширяет поведение Queue блокирующими методами
- все методы потокобезопасны по контракту
- `drainTo` - предпочтительнее цикла `poll()`, особенно в producer-consumer сценариях

</details>

### `Iterable -> Collection -> Queue -> BlockingDeque`

<details>
    <summary>
        <code>BlockingDeque&lt;E&gt; extends BlockingQueue&lt;E&gt;, Deque&lt;E&gt;</code>
    </summary>

# `public interface BlockingDeque<E> extends BlockingQueue<E>, Deque<E>`

> Потокобезопасная двусторонняя очередь: объединяет возможности `BlockingQueue` и `Deque` с блокирующими операциями на обоих концах.

- Наследует все методы `Deque<E>` (`addFirst`, `pollLast`, `push`, `pop` и т.д.)
- Наследует все методы `BlockingQueue<E>` (`put`, `take`, `drainTo` и т.д.)


- Добавляет блокирующие аналоги для *обоих концов*:


- `void putFirst(E e)` — добавить в начало; **блокирует**, пока не освободится место
- `void putLast(E e)` — добавить в конец; **блокирует**, пока не освободится место
- `E takeFirst()` — извлечь и удалить начало; **блокирует**, пока не станет непустым
- `E takeLast()` — извлечь и удалить конец; **блокирует**, пока не станет непустым

- `E pollFirst(long timeout, TimeUnit unit)` — как `pollFirst()`, но с таймаутом
- `E pollLast(long timeout, TimeUnit unit)` — как `pollLast()`, но с таймаутом
- `boolean offerFirst(E e, long timeout, TimeUnit unit)` — как `offerFirst()`, но с таймаутом
- `boolean offerLast(E e, long timeout, TimeUnit unit)` — как `offerLast()`, но с таймаутом

> ⚠️ Все `put*`/`take*` — **могут блокироваться бесконечно**.  
> Все методы с таймаутом возвращают `null`/`false` при истечении времени.  
> Исключения `InterruptedException` объявлены в `throws` у блокирующих методов.
</details>

### `Iterable -> Collection -> SequencedCollection`

<details>
    <summary>
        <code>SequencedCollection&lt;E&gt; extends Collection&lt;E&gt;</code>
    </summary>

# `public interface SequencedCollection<E> extends Collection<E>`

> Коллекция с чётко определённым началом и концом.

- `E getFirst()` / `getLast()` - возвращает первый элемент
- `void addFirst(E)` / `addLast(E)` - добавляет элемент в начало / конец
- `E removeFirst()` / `removeLast()` - удаляет и возвращает первый / последний элемент
- `SequencedCollection<E> reversed()` - возвращает представление коллекции в обратном порядке (view, изменения отражаются в оригинале и наоборот)

</details>

</details>

<details>
    <summary>
        <b>Конкретные реализации</b>
    </summary>

<details>
    <summary>
        <code>ArrayList&lt;E&gt;</code>
</summary>

## `public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable`

> Стандартный выбор для списка, если нужен быстрый доступ по индексу и/или список редко изменяется в середине. 
>
> Подходит для итерации, случайного доступа, когда объём данных умеренный или предсказуем. 
> 
> Не используйте при частых вставках/удалениях в начале или середине — будет O(n).

- Основа: динамический массив
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: ДА
- Сложность:
    - `get(i)`, `set(i, e)` — O(1)
    - `add(e)` — O(1) (амортизированно)
    - `add(i, e)`, `remove(i)` — O(n)
    - `indexOf(o)`, `remove(o)`, `contains(o)` — O(n)
- Потокобезопасность: НЕТ (`Collections.synchronizedList()` / `CopyOnWriteArrayList`)
- Особенности:
    - `subList()` и `reversed()` возвращают живые view
    - итераторы — fail-fast
    - default capacity: 10
    - default load factor: 1.5 при переполнении

</details>

<details>
    <summary>
        <code>CopyOnWriteArrayList&lt;E&gt;</code>
    </summary>

## `public class CopyOnWriteArrayList<E> implements List<E>, RandomAccess, Cloneable, Serializable`

> Используется в сценариях с преобладанием чтений и редкими изменениями. 
> 
> Все модифицирующие операции (`add`, `set`, `remove`) создают полную копию внутреннего массива — отсюда название и высокая цена записи, но абсолютная безопасность чтения без блокировок. 
> 
> Итераторы не бросают `ConcurrentModificationException` и работают с «снимком» состояния на момент создания.

- Основа: неизменяемый массив (`volatile Object[]`), копируемый при записи
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: ДА
- Сложность:
    - `get(i)` — O(1)
    - `iterator()`, `forEach()` — O(1) (итерация по фиксированному массиву)
    - `add(e)`, `set(i, e)`, `remove(i)` — O(n) (копирование всего массива)
    - `contains(o)`, `indexOf(o)` — O(n)
- Потокобезопасность: ДА (все операции атомарны; итерация не требует синхронизации)
- Особенности:
    - итераторы — слабо согласованные (weakly consistent), не fail-fast
    - `subList()` и `reversed()` возвращают живые view (но view также copy-on-write)
    - не поддерживает `ListIterator.add/remove/set`
    - потребляет много памяти при частых изменениях
    - используется, например, в `EventListenerList` Swing или подписках в реактивных системах с редкими обновлениями

</details>

<details>
    <summary>
        <code>LinkedList&lt;E&gt;</code>
    </summary>

## `public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, Serializable`

> Используется, когда нужны частые вставки и удаления в начале или конце списка (например, как очередь или стек), либо когда требуется реализация `Deque`. 
> 
> Не подходит для доступа по индексу или частого вызова `contains()`/`indexOf()` — эти операции линейные. 
> 
> В большинстве случаев уступает `ArrayList` по кэш-локальности и накладным расходам на узлы.

- Основа: двусвязный список (каждый элемент — `Node` с `prev`, `next`, `item`)
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: ДА
- Сложность:
    - `get(i)`, `set(i, e)` — O(n) *(проход от ближайшего конца)*
    - `add(e)` — O(1)
    - `addFirst(e)`, `addLast(e)`, `removeFirst()`, `removeLast()` — O(1)
    - `add(i, e)`, `remove(i)` — O(n)
    - `contains(o)`, `indexOf(o)`, `remove(o)` — O(n)
- Потокобезопасность: НЕТ (`Collections.synchronizedList()` — но не даёт преимуществ `Deque`)
- Особенности:
    - реализует `Deque`, поэтому поддерживает `offer`, `poll`, `peek`, `push`, `pop`
    - `subList()` возвращает view, но не поддерживает `RandomAccess`
    - итераторы — fail-fast
    - накладные расходы: ~3× больше памяти на элемент (по сравнению с `ArrayList`)

</details>

<details>
    <summary>
        <code>Vector&lt;E&gt;</code>
    </summary>

## `public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable`

> Устаревшая синхронизированная реализация списка на основе динамического массива. Фактически — `ArrayList` с synchronized-методами. 
> 
> Из-за глобальной блокировки (`synchronized` на каждом методе) не масштабируется при высокой конкурентности. 
> 
> Используется редко — в основном для совместимости со старым кодом или в узких случаях, где нужна простая потокобезопасность и отсутствует высокая нагрузка.

- Основа: динамический массив
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: ДА
- Сложность:
    - `get(i)`, `set(i, e)` — O(1)
    - `add(e)` — O(1) (амортизированно)
    - `add(i, e)`, `remove(i)` — O(n)
    - `indexOf(o)`, `remove(o)`, `contains(o)` — O(n)
- Потокобезопасность: ДА (все публичные методы synchronized)
- Особенности:
    - поддерживает явный `capacityIncrement` (при >0 — растёт на фиксированную величину, иначе ×2)
    - итераторы — fail-fast (и при этом не потокобезопасны: `ConcurrentModificationException` возможна даже без конкурентной модификации)
    - наследует `elementCount` как size, `elementData` как массив
    - не рекомендуется для нового кода: предпочтительнее `ArrayList` + внешняя синхронизация или `CopyOnWriteArrayList`

</details>

<details>
    <summary>
        <code>Stack&lt;E&gt;</code>
</summary>

## `public class Stack<E> extends Vector<E>`

> Устаревшая реализация стека (LIFO) на основе `Vector`. 
> 
> Помечен как deprecated (начиная с Java 21, JEP 454), так как наследование от `Vector` нарушает инкапсуляцию и приводит к утечке операций списка (`get(i)`, `insertElementAt()` и т.п.). 
> 
> Вместо него рекомендуется использовать `Deque<E>`: `ArrayDeque` или `LinkedList` с методами `push()`, `pop()`, `peek()`.

- Основа: `Vector` (динамический массив + synchronized)
- Порядок: LIFO (last-in-first-out)
- Дубликаты: ДА
- `null`: ДА
- Сложность:
    - `push(e)`, `pop()`, `peek()` — O(1) *(амортизированно для `push`, если resize)*
    - `empty()`, `size()` — O(1)
    - `search(o)` — O(n) *(линейный поиск с вершины вниз, возвращает расстояние от вершины + 1)*
- Потокобезопасность: ДА (наследует синхронизацию от `Vector`)
- Особенности:
    - `search()` возвращает 1-based расстояние или -1
    - наследует всё API `Vector`, включая индексированный доступ — что ломает абстракцию стека
    - не реализует `Deque` и не совместим с современными коллекционными утилитами для стеков

</details>

<br>

<details>
    <summary>
        <code>HashSet&lt;E&gt;</code>
    </summary>

## `public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, Serializable`

> Основная реализация `Set` — неупорядоченное множество без дубликатов. 
> 
> Строится поверх `HashMap`, где элементы хранятся как ключи, а значения — заглушка (`PRESENT`). 
> 
> Быстрый доступ, вставка, удаление в среднем O(1). 
> 
> Порядок итерации не определён и может меняться при изменении. 
> 
> Не подходит, если важен порядок или стабильность итерации.

- Основа: хэш-таблица (`HashMap`)
- Порядок: не определён (зависит от хэша и размера таблицы)
- Дубликаты: НЕТ
- `null`: ДА (только один)
- Сложность:
    - `add(e)`, `remove(e)`, `contains(e)` — O(1) (амортизированно)
    - `iterator()`, `size()`, `isEmpty()` — O(1)
- Потокобезопасность: НЕТ (`Collections.synchronizedSet()` или `ConcurrentHashMap.newKeySet()`)
- Особенности:
    - default initial capacity: 16
    - default load factor: 0.75
    - при превышении capacity × load factor — rehash и resize (×2)
    - итераторы — fail-fast

</details>

<details>
    <summary>
        <code>LinkedHashSet&lt;E&gt;</code>
    </summary>

## `public class LinkedHashSet<E> extends HashSet<E> implements Set<E>, Cloneable, Serializable`

> Реализация `Set` с предсказуемым порядком итерации — порядок вставки (или доступа, если включён access-order через package-private конструктор). 
> 
> Строится на `LinkedHashMap`, добавляя к `HashSet` двусвязный список для учёта порядка. 
> 
> Используется, когда нужна уникальность + стабильный порядок обхода. 
> 
> Накладные расходы немного выше, чем у `HashSet`, но порядок итерации стабилен и понятен.

- Основа: хэш-таблица + двусвязный список (`LinkedHashMap`)
- Порядок: порядок вставки (по умолчанию); порядок доступа — только через внутренний конструктор (не публичный API)
- Дубликаты: НЕТ
- `null`: ДА (ровно один)
- Сложность:
    - `add(e)`, `remove(e)`, `contains(e)` — O(1) (амортизированно)
    - `iterator()` — O(1) на создание, O(n) на полную итерацию (но порядок фиксирован)
- Потокобезопасность: НЕТ (`Collections.synchronizedSet()`)
- Особенности:
    - наследует параметры от `HashSet`: initial capacity, load factor
    - итераторы — fail-fast
    - потребляет чуть больше памяти, чем `HashSet` (из-за ссылок `before`/`after`)
    - часто используется для дедупликации с сохранением порядка (например, в `Collectors.toCollection(LinkedHashSet::new)`)

</details>

<details>
<summary>
<code>TreeSet&lt;E&gt;</code>
</summary>

## `public class TreeSet<E> extends AbstractSet<E> implements NavigableSet<E>, Cloneable, Serializable`

> Упорядоченное множество на основе красно-чёрного дерева. 
> 
> Гарантирует сортированный порядок элементов: естественный (`Comparable`) или заданный `Comparator`. 
> 
> Поддерживает эффективный поиск ближайших элементов (`lower`, `floor`, `ceiling`, `higher`) и диапазонные операции (`subSet`, `headSet`, `tailSet` — все как живые view). 
> 
> Используется, когда важен порядок и нужны операции навигации.

- Основа: красно-чёрное дерево (реализовано через `TreeMap`, где элементы — ключи)
- Порядок: сортированный (natural order или comparator)
- Дубликаты: НЕТ
- `null`: запрещён, если используется natural order; разрешён, если `Comparator` корректно его обрабатывает
- Сложность:
    - `add(e)`, `remove(e)`, `contains(e)` — O(log n)
    - `first()`, `last()`, `lower(e)`, `floor(e)`, `ceiling(e)`, `higher(e)` — O(log n)
    - `iterator()`, `descendingIterator()` — O(1) на создание, O(n) на обход
- Потокобезопасность: НЕТ (`Collections.synchronizedSortedSet()`)
- Особенности:
    - все методы диапазонов (`subSet`, `headSet`, `tailSet`) возвращают живые view
    - `descendingSet()` — view в обратном порядке
    - итераторы — fail-fast
    - не поддерживает произвольный доступ по индексу (нет `get(i)`)

</details>

<details>
    <summary>
        <code>CopyOnWriteArraySet&lt;E&gt;</code>
</summary>

## `public class CopyOnWriteArraySet<E> extends AbstractSet<E> implements Set<E>, Cloneable, Serializable`

> Потокобезопасная реализация `Set`, построенная поверх `CopyOnWriteArrayList`. 
> 
> Подходит для сценариев с преобладанием чтений и редкими изменениями. 
> 
> Все модификации требуют полного копирования внутреннего списка, поэтому дорогостоящие при большом объёме данных. 
> 
> Гарантирует согласованность итераторов без блокировок.

- Основа: `CopyOnWriteArrayList`
- Порядок: порядок вставки (первого добавления уникального элемента)
- Дубликаты: НЕТ
- `null`: ДА (ровно один)
- Сложность:
    - `contains(o)`, `size()`, `iterator()` — O(n) *(внутренний список сканируется линейно)*
    - `add(e)`, `remove(e)` — O(n) *(копирование всего списка + линейный поиск для проверки уникальности)*
- Потокобезопасность: ДА (наследует свойства `CopyOnWriteArrayList`)
- Особенности:
    - не использует хэширование — сравнение через `equals()`, как в списке
    - итераторы — weakly consistent, не fail-fast
    - `toArray()` и `iterator()` работают с моментальным снимком
    - неэффективна при большом числе элементов или частых модификациях

</details>

<details>
    <summary>
        <code>ConcurrentSkipListSet&lt;E&gt;</code>
    </summary>

## `public class ConcurrentSkipListSet<E> extends AbstractSet<E> implements NavigableSet<E>, Cloneable, Serializable`

> Потокобезопасная реализация `NavigableSet` на основе skip list.
>
> Аналог `TreeSet`, но с поддержкой высокой конкурентности: операции чтения не блокируются, запись использует тонкую блокировку (CAS + локальные блокировки узлов).
>
> Подходит для сценариев с частыми параллельными операциями вставки/удаления/поиска при необходимости сортированного порядка.

- Основа: skip list (многоуровневый связный список)
- Порядок: сортированный (natural order или comparator)
- Дубликаты: НЕТ
- `null`: запрещён (всегда, даже с comparator)
- Сложность (амортизированная):
    - `add(e)`, `remove(e)`, `contains(e)` — O(log n)
    - `lower(e)`, `floor(e)`, `ceiling(e)`, `higher(e)` — O(log n)
    - `first()`, `last()` — O(log n)
    - `iterator()`, `descendingIterator()` — O(1) на создание, O(n) на обход
- Потокобезопасность: ДА (concurrent reads, lock-free/CAS-based writes)
- Особенности:
    - итераторы — weakly consistent: не блокируют, могут отражать часть изменений, сделанных после создания
    - не fail-fast
    - реализован поверх `ConcurrentSkipListMap` (элементы хранятся как ключи)
    - `subSet()`, `headSet()`, `tailSet()`, `descendingSet()` возвращают живые view
    - не гарантирует атомарности bulk-операций (`addAll`, `retainAll`)

</details>

<details>
    <summary>
        <code>EnumSet&lt;E extends Enum&lt;E&gt;&gt;</code>
    </summary>

## `public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E> implements Cloneable, Serializable`

> Высокоэффективная реализация `Set` для перечислений (`enum`). 
>
> Внутренне использует битовые векторы (bit vector), где каждый бит соответствует одному значению `enum`. 
>
> Все операции — O(1) или O(n_bits), что на практике константа (максимум 64 бита для одного long, или массив для >64). 
>
> Создаётся только через статические фабрики (`of`, `range`, `allOf`, `noneOf`, `complementOf`). 
>
> Не поддерживает `null`.

- Основа: битовый вектор (`long` или `long[]`)
- Порядок: порядок объявления в `enum` (`ordinal()`-порядок)
- Дубликаты: НЕТ
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `add(e)`, `remove(e)`, `contains(e)` — O(1)
    - `iterator()`, `size()`, `isEmpty()` — O(1)
    - bulk-операции (`addAll`, `retainAll`, `complement`) — O(1) или O(k/64), где k — число констант в enum
- Потокобезопасность: НЕТ (`Collections.synchronizedSet()`)
- Особенности:
    - фиксированный тип при создании — все элементы должны быть одного `enum`-типа
    - сериализуется компактно (только имена enum-констант)
    - `clone()` создаёт мелкую копию (безопасно, т.к. enum — immutable)
    - единственная реализация в JDK — пакетно-приватные классы (`RegularEnumSet`, `JumboEnumSet`)

</details>

<br>

<details>
    <summary>
        <code>ArrayDeque&lt;E&gt;</code>
    </summary>

## `public class ArrayDeque<E> extends AbstractCollection<E> implements Deque<E>, Cloneable, Serializable`

> Основная реализация `Deque` — двусторонняя очередь на основе циклического массива. 
> 
> Используется как эффективная замена `Stack` (LIFO) и `LinkedList` (FIFO) для большинства сценариев. 
> 
> Быстрее, чем `LinkedList`, за счёт лучшей кэш-локальности и отсутствия накладных расходов на узлы. 
> 
> Не допускает `null`.

- Основа: циклический массив (`Object[]`)
- Порядок: порядок вставки (от головы к хвосту)
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `addFirst(e)`, `addLast(e)`, `offerFirst(e)`, `offerLast(e)` — O(1) (амортизированно)
    - `removeFirst()`, `removeLast()`, `pollFirst()`, `pollLast()` — O(1)
    - `getFirst()`, `getLast()`, `peekFirst()`, `peekLast()` — O(1)
    - `push(e)`, `pop()`, `peek()` — O(1) (через `addFirst`/`removeFirst`/`peekFirst`)
    - `iterator()`, `descendingIterator()` — O(1) на создание
- Потокобезопасность: НЕТ (`Collections.synchronizedDeque()`)
- Особенности:
    - capacity — степень двойки (для быстрого взятия по модулю через `&`)
    - resize ×2 при переполнении
    - итераторы — fail-fast
    - `descendingIterator()` и `reversed()` возвращают view в обратном порядке
    - рекомендуется как стек или очередь вместо `Stack` и `LinkedList`, если не нужны `null` и не требуется реализация `List`

</details>

<details>
    <summary>
        <code>ConcurrentLinkedDeque&lt;E&gt;</code>
    </summary>

## `public class ConcurrentLinkedDeque<E> extends AbstractCollection<E> implements Deque<E>, Serializable`

> Потокобезопасная неблокирующая двусторонняя очередь на основе двусвязного списка. 
> 
> Все операции — lock-free (на основе CAS), что обеспечивает высокую масштабируемость при высокой конкурентности. 
> 
> Подходит для сценариев с множеством параллельных `add`/`poll` с обоих концов. Не допускает `null`.

- Основа: двусвязный список (lock-free, с CAS-обновлениями указателей)
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность (амортизированная):
    - `addFirst(e)`, `addLast(e)`, `offerFirst(e)`, `offerLast(e)` — O(1)
    - `removeFirst()`, `removeLast()`, `pollFirst()`, `pollLast()` — O(1)
    - `peekFirst()`, `peekLast()` — O(1)
    - `size()`, `contains(o)` — O(n) (требуют сканирования из-за отсутствия глобального состояния)
- Потокобезопасность: ДА (lock-free, concurrent reads & writes)
- Особенности:
    - итераторы — weakly consistent: не блокируют, не fail-fast, могут пропускать или дублировать элементы при модификации
    - `remove(o)` и `removeFirstOccurrence(o)` — O(n), не атомарны относительно других операций
    - потребляет больше памяти, чем `ArrayDeque` (из-за узлов и overhead CAS-логики)
    - не поддерживает `RandomAccess`, `Cloneable`

</details>

<details>
<summary>
<code>ConcurrentLinkedQueue&lt;E&gt;</code>
</summary>

## `public class ConcurrentLinkedQueue<E> extends AbstractQueue<E> implements Queue<E>, Serializable`

> Потокобезопасная неблокирующая очередь FIFO на основе односвязного списка. 
> 
> Реализует алгоритм использованием CAS. 
> 
> Подходит для высококонкурентных producer-consumer сценариев без ограничения ёмкости. 
> 
> Не допускает `null`.

- Основа: односвязный список (lock-free, head/tail с CAS)
- Порядок: FIFO (первый добавленный — первый извлечённый)
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность (амортизированная):
    - `offer(e)`, `add(e)` — O(1)
    - `poll()`, `peek()` — O(1)
    - `size()`, `contains(o)` — O(n) *(требуют полного обхода)*
- Потокобезопасность: ДА (lock-free, scalable reads & writes)
- Особенности:
    - итераторы — weakly consistent: не блокируют, не fail-fast, могут не включать недавно добавленные или уже удалённые элементы
    - `remove(o)` — O(n), не атомарен
    - не имеет ограничения capacity (unbounded)
    - потребляет больше памяти, чем `ArrayDeque`, но лучше масштабируется под нагрузкой
    - `drainTo()` не переопределён — использует базовую реализацию через `poll()`

</details>

<details>
<summary>
<code>PriorityQueue&lt;E&gt;</code>
</summary>

## `public class PriorityQueue<E> extends AbstractQueue<E> implements Serializable`

> Очередь с приоритетом на основе двоичной кучи (min-heap по умолчанию). 
> 
> Элемент с наименьшим значением (по natural order или comparator) извлекается первым. 
> 
> Не обеспечивает FIFO между элементами с одинаковым приоритетом. 
> 
> Не потокобезопасна. 
> 
> Не допускает `null`.

- Основа: двоичная куча в массиве (`Object[]`)
- Порядок: по приоритету (natural order или comparator); стабильность FIFO — не гарантируется
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `offer(e)`, `add(e)` — O(log n)
    - `poll()`, `peek()` — O(log n) и O(1) соответственно
    - `remove(o)`, `contains(o)` — O(n)
    - `size()`, `isEmpty()` — O(1)
- Потокобезопасность: НЕТ (`PriorityBlockingQueue` — потокобезопасный аналог)
- Особенности:
    - внутренний массив не отсортирован — только свойство кучи (родитель ≤ потомков)
    - итератор не гарантирует порядок обхода (не по приоритету)
    - итераторы — fail-fast
    - `toArray()` не возвращает элементы в порядке приоритета
    - `iterator().remove()` поддерживается

</details>

<details>
    <summary>
        <code>PriorityBlockingQueue&lt;E&gt;</code>
    </summary>

## `public class PriorityBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable`

> Потокобезопасная очередь с приоритетом, аналог `PriorityQueue`, но с поддержкой блокирующих операций (`take()`, `put()` всегда неблокирующие для `put`, но `take()` блокируется при пустой очереди). 
> 
> Основана на динамической двоичной куче с ReentrantLock для записи и условной синхронизацией для ожидания элементов. 
> 
> Не имеет ограничения capacity (за исключением `Integer.MAX_VALUE`).

- Основа: двоичная куча в массиве (`Object[]`) + `ReentrantLock`
- Порядок: по приоритету (natural order или comparator); FIFO между равными — не гарантируется
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `offer(e)`, `add(e)`, `put(e)` — O(log n)
    - `poll()`, `peek()` — O(log n) и O(1) соответственно
    - `take()` — O(log n) после пробуждения
    - `remove(o)`, `contains(o)` — O(n)
    - `size()` — O(1)
- Потокобезопасность: ДА (блокирующая, с отдельными lock для чтения/записи в критических участках)
- Особенности:
    - unbounded (capacity ограничена только `MAX_ARRAY_SIZE`)
    - `put(e)` никогда не блокируется (в отличие от других `BlockingQueue`)
    - `take()` блокируется, пока очередь не станет непустой
    - итераторы — weakly consistent, не fail-fast, не гарантируют порядок приоритета
    - `drainTo()` поддерживается и может быть эффективнее множественных `poll()`

</details>

<details>
    <summary>
        <code>SynchronousQueue&lt;E&gt;</code>
    </summary>

## `public class SynchronousQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable`

> Очередь-«рукопожатие»: каждый `put()` должен дождаться соответствующего `take()` (и наоборот). 
> 
> Внутренне не хранит элементы — передаёт их напрямую от производителя к потребителю. 
> 
> Используется в сценариях высокой пропускной способности с 1:1 передачей (например, в `ThreadPoolExecutor` с `CallerRunsPolicy` или для жёсткой синхронизации потоков).

- Основа: нет внутреннего хранилища; передача через wait/notify (fair) или CAS (non-fair)
- Порядок: FIFO при fair-режиме, иначе не определён
- Дубликаты: ДА (но каждый элемент живёт «мгновенно»)
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `put(e)`, `offer(e, timeout)`, `take()`, `poll(timeout)` — O(1) при наличии пары, иначе блокировка/ожидание
    - `offer(e)` (без таймаута) — всегда false (не может завершиться сразу, если нет ожидающего `take()`)
    - `peek()`, `size()`, `contains(o)`, `iterator()` — всегда возвращают `null`, `0`, `false`, пустой итератор
- Потокобезопасность: ДА (встроенная поддержка blocking operations)
- Особенности:
    - fair-режим включается через конструктор (`new SynchronousQueue(true)`)
    - `drainTo()` всегда возвращает 0
    - не поддерживает `remove(o)`, `toArray()` бесполезен
    - максимальная пропускная способность при балансе producer/consumer

</details>

<details>
    <summary>
        <code>ArrayBlockingQueue&lt;E&gt;</code>
    </summary>

## `public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable`

> Потокобезопасная очередь с фиксированной ёмкостью на основе циклического массива. 
> 
> Использует один `ReentrantLock` (разделяемый для всех операций) и пару `Condition` (`notEmpty`, `notFull`). 
> 
> Подходит, когда нужно ограничить потребление памяти и обеспечить предсказуемое поведение при переполнении / опустошении.

- Основа: циклический массив фиксированного размера
- Порядок: FIFO
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `offer(e)`, `put(e)`, `poll()`, `take()` — O(1)
    - `peek()`, `size()`, `remainingCapacity()` — O(1)
    - `contains(o)`, `remove(o)` — O(n)
- Потокобезопасность: ДА (блокирующая, один общий lock для всех операций)
- Особенности:
    - capacity задаётся в конструкторе — неизменяема
    - `drainTo()` эффективен (переносит до `remainingCapacity()` элементов атомарно)
    - fair-режим опционален (через конструктор) — влияет на порядок пробуждения потоков
    - потребляет меньше памяти, чем `LinkedBlockingQueue` (нет узлов)
    - возможна блокировка всех операций под одним lock’ом (менее масштабируемо при высокой нагрузке)

</details>

<details>
<summary>
<code>DelayQueue&lt;E extends Delayed&gt;</code>
</summary>

## `public class DelayQueue<E extends Delayed> extends AbstractQueue<E> implements BlockingQueue<E>`

> Потокобезопасная очередь, в которой элементы становятся доступными для извлечения только по истечении задержки (`getDelay() <= 0`). 
> 
> Основана на `PriorityQueue` (по времени истечения) + `ReentrantLock`. 
> 
> Используется для отложенных задач: кэш с TTL, retry-механизмы, scheduled-обработка без фиксированного пула.

- Основа: `PriorityQueue` (min-heap по времени истечения) + `ReentrantLock`
- Порядок: по времени истечения задержки (`Delayed.getDelay(TimeUnit.NANOSECONDS)`)
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `offer(e)`, `put(e)`, `add(e)` — O(log n)
    - `poll()`, `peek()`, `take()` — O(log n) при извлечении, O(1) при недоступности элементов
    - `take()` блокируется, пока не появится элемент с истёкшей задержкой
    - `size()`, `remainingCapacity()` — O(1) и `Integer.MAX_VALUE` соответственно
    - `remove(o)`, `contains(o)` — O(n)
- Потокобезопасность: ДА (блокирующая)
- Особенности:
    - capacity: unbounded (ограничена только памятью и `Integer.MAX_VALUE`)
    - `poll()` возвращает `null`, если ни один элемент не готов
    - `iterator()` не учитывает задержки: перебирает все элементы (включая ещё не готовые), порядок — не гарантирован
    - элементы должны реализовывать `Delayed` (методы `getDelay(TimeUnit)` и `compareTo(Delayed)`)
    - не поддерживает `drainTo()` напрямую (но унаследована реализация через `poll()`)

</details>

<details>
<summary>
<code>LinkedBlockingQueue&lt;E&gt;</code>
</summary>

## `public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable`

> Потокобезопасная очередь на основе связного списка. 
> 
> Поддерживает ограниченную или неограниченную ёмкость (по умолчанию — unbounded, capacity = `Integer.MAX_VALUE`). 
> 
> Использует два отдельных `ReentrantLock` — один для `put`, другой для `take` — что повышает параллелизм по сравнению с `ArrayBlockingQueue`.

- Основа: связный список (`Node`) с отдельными lock-ами для головы и хвоста
- Порядок: FIFO
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `offer(e)`, `put(e)`, `poll()`, `take()` — O(1)
    - `peek()`, `size()` — O(1) (size поддерживается атомарным счётчиком)
    - `contains(o)`, `remove(o)` — O(n)
- Потокобезопасность: ДА (блокирующая, с разделением lock-ов на producer/consumer)
- Особенности:
    - конструктор без параметров — unbounded (`capacity = Integer.MAX_VALUE`)
    - конструктор с capacity — bounded, `offer(e)` может вернуть `false`, `put(e)` блокируется при переполнении
    - `drainTo()` эффективен и может быть атомарным для части элементов
    - потребляет больше памяти на элемент, чем `ArrayBlockingQueue` (из-за узлов)
    - лучше масштабируется при высокой нагрузке от producers и consumers одновременно

</details>

<details>
<summary>
<code>LinkedTransferQueue&lt;E&gt;</code>
</summary>

## `public class LinkedTransferQueue<E> extends AbstractQueue<E> implements TransferQueue<E>, Serializable`

> Неблокирующая очередь с поддержкой transfer-операций: `transfer(e)` блокируется, пока другой поток не вызовет `take()` или `poll()`, позволяя передать элемент напрямую, минуя внутреннее хранилище. 
> 
> Реализует алгоритм dual queue/dual stack (на основе связного списка с CAS). 
> 
> Универсальна: работает и как обычная очередь, и как высокопроизводительный канал связи.

- Основа: связный список, lock-free (CAS), dual-структура (producer/consumer ноды)
- Порядок: FIFO
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность (амортизированная):
    - `offer(e)`, `add(e)`, `put(e)` — O(1)
    - `poll()`, `peek()`, `take()` — O(1)
    - `transfer(e)`, `tryTransfer(e)`, `tryTransfer(e, timeout)` — O(1) при наличии пары, иначе блокировка/ожидание
    - `size()`, `contains(o)` — O(n)
- Потокобезопасность: ДА (lock-free для большинства операций, без глобальных блокировок)
- Особенности:
    - unbounded — capacity не ограничена
    - `transfer(e)` гарантирует, что элемент будет получен тем, кто его запросил (не просто добавлен в очередь)
    - `tryTransfer(e)` возвращает `true`, только если элемент немедленно принят
    - итераторы — weakly consistent, не fail-fast
    - `drainTo()` поддерживается
    - единственная стандартная реализация `TransferQueue` в JDK

</details>

<details>
<summary>
<code>LinkedBlockingDeque&lt;E&gt;</code>
</summary>

## `public class LinkedBlockingDeque<E> extends AbstractQueue<E> implements BlockingDeque<E>, Serializable`

> Потокобезопасная двусторонняя блокирующая очередь на основе связного списка. 
> 
> Поддерживает ограниченную ёмкость (по умолчанию — unbounded, capacity = `Integer.MAX_VALUE`). 
> 
> Использует один `ReentrantLock` (в отличие от `LinkedBlockingQueue`, где два lock’а), что снижает параллелизм при одновременных операциях с обоих концов.

- Основа: двусвязный список (`Node`) + единый `ReentrantLock`
- Порядок: порядок вставки
- Дубликаты: ДА
- `null`: НЕТ (бросает `NullPointerException`)
- Сложность:
    - `addFirst(e)`, `addLast(e)`, `offerFirst(e)`, `offerLast(e)` — O(1)
    - `putFirst(e)`, `putLast(e)` — O(1) (блокируются при переполнении)
    - `pollFirst()`, `pollLast()`, `takeFirst()`, `takeLast()` — O(1)
    - `peekFirst()`, `peekLast()` — O(1)
    - `size()`, `remainingCapacity()` — O(1)
    - `contains(o)`, `remove(o)` — O(n)
- Потокобезопасность: ДА (блокирующая, один общий lock)
- Особенности:
    - bounded/unbounded в зависимости от конструктора (по умолчанию — unbounded)
    - реализует `BlockingDeque`, поэтому поддерживает двусторонние blocking-операции
    - `drainTo()` эффективен, но не атомарен для всей операции
    - итераторы — weakly consistent, не fail-fast
    - используется реже, чем `ArrayDeque` или `ConcurrentLinkedDeque`, из-за overhead блокировок

</details>

</details>

```text
public interface Map<K, V>
 └──┐
    ├── public interface SortedMap<K,V> extends SequencedMap<K,V>
    │    │
    │    └── public interface NavigableMap<K,V> extends SortedMap<K,V> ───┐
    │                                                                     │
    ├── public interface ConcurrentMap<K,V> extends Map<K,V> ─────────────│
    │                                                                     └── public interface ConcurrentNavigableMap<K,V> extends ConcurrentMap<K,V>, NavigableMap<K,V>
    └── public interface SequencedMap<K, V> extends Map<K, V>         
```

<details>
    <summary>
        <b>Основные Интерфейсы</b>
    </summary>

<details>
    <summary>
        <code>Map&lt;K, V&gt;</code>
    </summary>

# `public interface Map<K, V>`

> Ассоциативный контейнер «ключ → значение». 
> 
> Ключи уникальны. 
> 
> Не является `Collection` и не реализует `Iterable`.

- `int size()` — возвращает количество пар; 0, если пуста
- `boolean isEmpty()` — true, если size() == 0
- `boolean containsKey(Object key)` — true, если ключ присутствует (учитывает null, если поддерживается)
- `boolean containsValue(Object value)` — true, если значение присутствует (линейная проверка в большинстве реализаций)
- `V get(Object key)` — возвращает значение по ключу или null (null ≠ отсутствие, если null разрешён как значение)
- `V put(K key, V value)` — добавляет/заменяет пару; возвращает предыдущее значение или null
- `V remove(Object key)` — удаляет пару по ключу; возвращает предыдущее значение или null
- `void putAll(Map<? extends K, ? extends V> m)` — копирует все пары из m (перезаписывает при совпадении ключей)
- `void clear()` — удаляет все пары
- `Set<K> keySet()` — представление ключей как Set (изменяется вместе с map)
- `Collection<V> values()` — представление значений как Collection (может содержать дубликаты)
- `Set<Map.Entry<K, V>> entrySet()` — представление пар как Set<Map.Entry> (основной способ итерации и модификации)

Default-методы (начиная с Java 8):

- `default V getOrDefault(Object key, V defaultValue)` — возвращает значение или defaultValue, если ключа нет
- `default void forEach(BiConsumer<? super K, ? super V> action)` — применяет действие ко всем парам (порядок не гарантирован, кроме упорядоченных реализаций)
- `default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function)` — заменяет каждое значение результатом функции от (ключ, значение); не допускает null-результата
- `default V putIfAbsent(K key, V value)` — вставляет пару, только если ключа ещё нет
- `default boolean remove(Object key, Object value)` — удаляет пару, только если (ключ, значение) в точности совпадают (включая null)
- `default boolean replace(K key, V oldValue, V newValue)` — заменяет значение, только если текущее равно oldValue
- `default V replace(K key, V value)` — заменяет значение по ключу, если ключ существует; возвращает старое значение или null
- `default V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)` — вычисляет и сохраняет значение, если ключа нет; mappingFunction не должна модифицировать карту
- `default V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)` — обновляет/удаляет значение по ключу, если ключ есть (null-результат → удаление)
- `default V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)` — применяет функцию независимо от наличия ключа
- `default V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction)` — если ключа нет — put(key, value); если есть — вычисляет новое значение из старого и value (null-результат → удаление)

Другие:

- `boolean equals(Object o)` — две карты равны, если содержат одинаковые пары (на основе equals() ключей и значений)
- `int hashCode()` — сумма hashCode() всех entry (где entry.hashCode() = key.hashCode() ^ value.hashCode())
- `default Spliterator<Map.Entry<K, V>> spliterator()` — возвращает spliterator по entrySet() (для Stream API)
- `default Set<Map.Entry<K, V>> entrySet()` — уже объявлен как абстрактный, но в Java 8+ некоторые реализации могут переопределить с оптимизациями

</details>

<details>
    <summary>
        <code>SortedMap&lt;K, V&gt;</code>
    </summary>

# `public interface SortedMap<K, V> extends Map<K, V>`

> Карта, ключи которой упорядочены по *естественному порядку* (Comparable) или по заданному Comparator. 
> 
> Гарантирует упорядоченную итерацию.

- `Comparator<? super K> comparator()` — возвращает компаратор, использованный для упорядочения, или null, если используется естественный порядок (K implements Comparable)
- `K firstKey()` — возвращает первый (наименьший) ключ; NoSuchElementException, если карта пуста
- `K lastKey()` — возвращает последний (наибольший) ключ; NoSuchElementException, если карта пуста
- `SortedMap<K, V> headMap(K toKey)` — возвращает представление части карты с ключами < toKey; изменения отражаются в оригинале; toKey не включается
- `SortedMap<K, V> tailMap(K fromKey)` — возвращает представление части карты с ключами ≥ fromKey; fromKey включается
- `SortedMap<K, V> subMap(K fromKey, K toKey)` — возвращает представление части карты с ключами в диапазоне [fromKey, toKey); fromKey включается, toKey — нет

Примечания:
- Все представления (headMap, tailMap, subMap) — живые view: изменения в представлении влияют на оригинал и наоборот.
- Поведение неопределено при модификации карты во время итерации (ConcurrentModificationException).
- Реализации обязаны обеспечивать логарифмическую или лучшую сложность для операций firstKey(), lastKey(), headMap(), tailMap(), subMap().

</details>

<details>
    <summary>
        <code>SequencedMap&lt;K, V&gt;</code>
    </summary>

# `public interface SequencedMap<K, V> extends Map<K, V>`

> Карта, в которой пары имеют определённый порядок вставки или изменения (например, порядок вставки или LRU). Введён в Java 21.

- `SequencedMap<K, V> reversed()` — возвращает представление той же карты, но с обратным порядком итерации; изменения отражаются в обеих сторонах
- `K firstKey()` — возвращает первый ключ в порядке итерации; NoSuchElementException, если карта пуста
- `K lastKey()` — возвращает последний ключ в порядке итерации; NoSuchElementException, если карта пуста
- `Map.Entry<K, V> firstEntry()` — возвращает первую пару (ключ-значение) или null, если карта пуста
- `Map.Entry<K, V> lastEntry()` — возвращает последнюю пару или null, если карта пуста
- `Map.Entry<K, V> pollFirstEntry()` — удаляет и возвращает первую пару или null, если карта пуста
- `Map.Entry<K, V> pollLastEntry()` — удаляет и возвращает последнюю пару или null, если карта пуста
- `V putFirst(K key, V value)` — вставляет пару в начало порядка (если ключ уже есть — удаляется старая позиция и пара перемещается в начало); возвращает предыдущее значение или null
- `V putLast(K key, V value)` — вставляет пару в конец порядка (если ключ уже есть — удаляется старая позиция и пара перемещается в конец); возвращает предыдущее значение или null
- `SequencedMap<V, K> inverse()` — возвращает двустороннее представление карты с обращёнными парами (значение ↔ ключ); изменения в одном отражаются в другом; порядок в inverse() соответствует порядку значений в исходной карте.
  - требует, чтобы значения в исходной карте были уникальны.
  - удобен для обратного просмотра, но не заменяет BiMap, если уникальность значений критична.

Примечания:
- Все методы, возвращающие `Map.Entry`, обязаны возвращать *immutable* entry (попытка изменить её через setValue() выбрасывает UnsupportedOperationException).
- `putFirst`/`putLast` гарантируют, что после вызова ключ будет *именно* первым/последним — даже если он уже присутствовал.
- `LinkedHashMap` (начиная с Java 21) реализует `SequencedMap`.
- В отличие от `SortedMap`, порядок здесь *не основан на сравнении ключей*, а на истории операций (вставки, доступа, перемещения).

</details>

<details>
    <summary>
        <code>NavigableMap&lt;K, V&gt;</code>
    </summary>

# `public interface NavigableMap<K, V> extends SortedMap<K, V>`

> Расширение SortedMap с поддержкой поиска ближайших ключей и строгого контроля границ диапазонов. 
> 
> Введён в Java 6.

- `Map.Entry<K, V> lowerEntry(K key)` — наибольшая запись с ключом < key, или null, если такой нет
- `K lowerKey(K key)` — наибольший ключ < key, или null
- `Map.Entry<K, V> floorEntry(K key)` — наибольшая запись с ключом ≤ key, или null
- `K floorKey(K key)` — наибольший ключ ≤ key, или null
- `Map.Entry<K, V> ceilingEntry(K key)` — наименьшая запись с ключом ≥ key, или null
- `K ceilingKey(K key)` — наименьший ключ ≥ key, или null
- `Map.Entry<K, V> higherEntry(K key)` — наименьшая запись с ключом > key, или null
- `K higherKey(K key)` — наименьший ключ > key, или null
- `Map.Entry<K, V> firstEntry()` — первая запись (наименьший ключ), или null, если пусто
- `Map.Entry<K, V> lastEntry()` — последняя запись (наибольший ключ), или null
- `Map.Entry<K, V> pollFirstEntry()` — удаляет и возвращает firstEntry(); null, если пусто
- `Map.Entry<K, V> pollLastEntry()` — удаляет и возвращает lastEntry(); null, если пусто
- `NavigableMap<K, V> descendingMap()` — представление той же карты с обратным порядком (по убыванию); изменения отражаются в обеих сторонах
- `NavigableSet<K> navigableKeySet()` — представление ключей как NavigableSet (с поддержкой floor/ceiling и т.д.)
- `NavigableSet<K> descendingKeySet()` — то же, что navigableKeySet().descendingSet()
- `NavigableMap<K, V> subMap(K fromKey, boolean fromInclusive, K toKey, boolean toInclusive)` — представление диапазона с гибким включением границ
- `NavigableMap<K, V> headMap(K toKey, boolean inclusive)` — представление ключей < toKey (или ≤ toKey, если inclusive)
- `NavigableMap<K, V> tailMap(K fromKey, boolean inclusive)` — представление ключей > fromKey (или ≥ fromKey, если inclusive)

Примечания:
- Все представления (subMap, headMap, tailMap, descendingMap) — «живые» и отражают изменения в оригинале.
- Реализации обязаны обеспечивать логарифмическую или лучшую сложность для операций поиска ближайших ключей.
- `TreeMap` — стандартная реализация; полностью реализует этот интерфейс.

</details>

<details>
    <summary>
        <code>ConcurrentMap&lt;K, V&gt;</code>
    </summary>

# `public interface ConcurrentMap<K, V> extends Map<K, V>`

> Расширение `Map`, добавляющее условные атомарные операции.
>
> Контракт требует, чтобы перечисленные ниже методы выполнялись как единое, не прерываемое действие — даже при конкурентном доступе.

- `V putIfAbsent(K key, V value)` - Вставляет пару, только если ключ отсутствует; возвращает предыдущее значение или `null`.
- `boolean remove(Object key, Object value)` - Удаляет пару, только если `(key, value)` в точности совпадают (включая `null`); возвращает `true`, если удаление произошло.
- `boolean replace(K key, V oldValue, V newValue)` - Заменяет значение, только если текущее равно `oldValue` (проверка через `equals`, с учётом `null`); возвращает `true`, если замена выполнена.
- `V replace(K key, V value)` - Заменяет значение по ключу, только если ключ присутствует; возвращает предыдущее значение или `null`.

Примечания:
- Эти методы переопределяют одноимённые default-методы из `Map`, но с усилением контракта: реализации обязаны обеспечивать атомарность.
- Интерфейс не требует потокобезопасности для других операций (например, `size()`, `entrySet().iterator()` и т.д.) — это остаётся на усмотрение конкретной реализации.

</details>

<details>
    <summary>
        <code>ConcurrentNavigableMap&lt;K, V&gt;</code>
    </summary>

# `public interface ConcurrentNavigableMap<K, V> extends NavigableMap<K, V>, ConcurrentMap<K, V>`

> Потокобезопасное расширение `NavigableMap`, сочетающее упорядоченность по ключам с атомарными условными операциями.
>
> Гарантирует атомарность всех методов из `ConcurrentMap`, а также корректную работу навигационных и диапазонных операций в условиях конкурентного доступа.

Наследуемые от `NavigableMap` методы с усилением контракта:
- Все методы поиска ближайших ключей (`lowerEntry`, `floorKey`, `ceilingEntry`, `higherKey` и др.) должны возвращать согласованный результат в рамках одного вызова.
- Все диапазонные представления (`subMap`, `headMap`, `tailMap`, `descendingMap`) являются живыми view — изменения в представлении отражаются в оригинале и наоборот.

Добавленные/унаследованные методы с атомарными гарантиями (из `ConcurrentMap`):
- `V putIfAbsent(K key, V value)` - Вставляет пару, только если ключ отсутствует; возвращает предыдущее значение или `null`.
- `boolean remove(Object key, Object value)` - Удаляет пару, только если `(key, value)` в точности совпадают; возвращает `true`, если удаление произошло.
- `boolean replace(K key, V oldValue, V newValue)` - Заменяет значение, только если текущее равно `oldValue`; возвращает `true`, если замена выполнена.
- `V replace(K key, V value)` - Заменяет значение по ключу, только если ключ присутствует; возвращает предыдущее значение или `null`.

Примечания:
- Интерфейс не требует, чтобы диапазонные представления были *сами по себе* потокобезопасными — но любая операция над ними должна соблюдать атомарность в рамках контракта `ConcurrentMap`/`NavigableMap`.
- Порядок итерации определяется компаратором (или естественным порядком), как в `SortedMap`; конкурентные модификации не нарушают упорядоченность.
- Как и в `NavigableMap`, сложность поиска ближайших ключей и диапазонных операций должна быть логарифмической или лучше — даже в условиях конкурентного доступа.

</details>

</details>

<details>
  <summary>
    <b>Конкретные реализации</b>
  </summary>

<details>
<summary>
<code>HashMap&lt;K, V&gt;</code>
</summary>

## `public class HashMap<K, V> extends AbstractMap<K, V> implements Map<K, V>, Cloneable, Serializable`

> Неупорядоченная, непотокобезопасная хэш-таблица на основе массива бакетов и связных списков/сбалансированных деревьев (при высокой коллизии).
>
> Основан на `hashCode()` и `equals()` ключей.
>
> Оптимизирована под высокую производительность в однопоточных сценариях и умеренной нагрузке.

- Основа: массив бакетов (`Node<K,V>[]`), каждый бакет — связный список (`Node`) или `TreeNode` (красно-чёрное дерево при ≥8 элементов в бакете и размере таблицы ≥64)
- Порядок: НЕГАРАНТИРОВАН (не путать с `LinkedHashMap`)
- Дубликаты ключей: НЕТ 
- Дубликаты значений: ДА
- `null`: ДА (один `null`-ключ разрешён; `null`-значения — любое количество)
- Сложность (амортизированная):
  - `get(key)`, `put(key, value)`, `remove(key)` — O(1) в среднем, O(log n) в худшем случае (при treeification)
  - `containsKey(key)` — O(1)
  - `containsValue(value)` — O(n)
  - `size()`, `isEmpty()` — O(1)
  - `keySet()`, `values()`, `entrySet()` — O(1) (возвращают view)
- Потокобезопасность: НЕТ (конкурентная модификация без синхронизации приводит к `ConcurrentModificationException` или *corruption* (например, бесконечный цикл в связном списке))
- Особенности:
  - начальная ёмкость (`initialCapacity`, по умолчанию 16) и коэффициент загрузки (`loadFactor`, по умолчанию 0.75) управляют resize-ами
  - resize происходит, когда `size > capacity * loadFactor`; таблица удваивается
  - `treeifyThreshold = 8`, `untreeifyThreshold = 6` — пороги для перехода список ↔ дерево
  - итераторы — fail-fast (бросают `ConcurrentModificationException` при структурной модификации)
  - `clone()` — shallow copy (новая таблица, но те же ключи/значения)
  - `entrySet().iterator().remove()` поддерживается (безопасное удаление при итерации)
  - `hash()` — дополнительное перемешивание хэш-кода (защита от плохих `hashCode()`):
    - обеспечивает вовлечение старших битов `hashCode()` в выбор бакета (т.к. индекс = `(n-1) & hash` использует только младшие биты);
    - вызывается при *каждой* операции по ключу (`get`, `put`, `remove` и др.);
    - не устраняет коллизии, но защищает от их кластеризации при плохих `hashCode()`

</details>










</details>