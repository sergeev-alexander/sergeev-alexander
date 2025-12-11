# Generics

> Generics в Java - это механизм, позволяющий создавать классы, интерфейсы и методы, которые могут работать с различными типами данных, обеспечивая при этом безопасность типов. 
> Обобщения позволяют создавать более абстрактный и гибкий код, который может быть использован с разными типами данных, сохраняя при этом статическую типизацию.
> Generics в Java обеспечивают параметризацию типов, что позволяет создавать обобщенные классы и методы, способные работать с разными типами данных. 
> Этот механизм позволяет использовать полиморфизм при работе с объектами различных типов, сохраняя при этом статическую типизацию.

## Параметризованный полиморфизм:

- Обобщенные классы и методы могут быть параметризованы типами, что позволяет им работать с разными типами данных.
- Это подразумевает использование параметров типа (например, `T`, `E`, `K`), которые будут заменены конкретными типами при создании объектов или вызове методов.
- Интерфейсы могут также быть параметризованы, что позволяет создавать обобщенные интерфейсы, предоставляя единые сигнатуры методов для различных типов.
- Ограничения могут быть использованы для добавления ограничений на типы, которые могут быть использованы с параметризованными классами или методами.
Это позволяет использовать полиморфизм только для определенных типов, удовлетворяющих определенным критериям.

<details>
    <summary>
        <b>Ковариантность (Covariance)</b>
    </summary>

> Суть ковариантности - это сохранение иерархии наследования в прямом направлении для составных типов (коллекций, массивов, обобщений).

Если `Child` - подтип `Parent`:

- Ковариантность: Контейнер<Child> - подтип Контейнер<Parent>
- Контравариантность (обратное): Контейнер<Parent> - подтип Контейнер<Child>
- Инвариантность (отсутствие): никакого отношения подтипов между Контейнер<Parent> и Контейнер<Child>

Основные принципы:

- Массивы в Java ковариантны, но это может приводить к `ArrayStoreException`
- Дженерики инвариантны по умолчанию, но можно сделать ковариантными через `? extends T`
- Возвращаемые типы методов могут быть ковариантными при переопределении
- `? extends T` позволяет читать как `T`, но не позволяет добавлять (кроме `null`)
- Используйте ковариантность (`? extends T`), когда контейнер является `producer` (поставщиком данных) 
- Для контейнеров-потребителей (`consumer`) используйте контравариантность (`? super T`)

## 1. Ковариантность массивов в Java

### Массивы в Java ковариантны по умолчанию. Это было сделано для совместимости с ранними версиями языка, но может приводить к ошибкам времени выполнения.

```java
class Animal {
    void sound() {
        System.out.println("Some sound");
    }
}

class Dog extends Animal {
    @Override
    void sound() {
        System.out.println("Woof!");
    }
}

class Cat extends Animal {
    @Override
    void sound() {
        System.out.println("Meow!");
    }
}

class DogChild extends Dog {
    @Override
    void sound() {
        System.out.println("Bark");
    }
}

public class CovarianceExample {

    public static void main(String[] args) {
        // ✅ Ковариантность массивов
        Dog[] dogs = new Dog[3];
        dogs[0] = new Dog();

        // Можно присвоить массив собак переменной типа "массив животных"
        Animal[] animals = dogs; // Ковариантность!

        // Проблема: компилятор это разрешает, но может привести к исключению
        try {
            animals[0] = new Cat(); // ❌ ArrayStoreException в runtime!
            animals[1] = new Animal(); // ❌ ArrayStoreException в runtime!
        } catch (ArrayStoreException e) {
            System.out.println("Ошибка: " + e.getMessage());
        }

        // ✅ Работает корректно
        animals[1] = new Dog(); // ✅ OK - Dog в Dog[]
    
        animals[2] = new DogChild(); // ✅ OK - DogChild тоже Dog
    }
}
```
### Ключевое правило: в ковариантный массив можно добавлять только сам тип или его подтипы, но не супертипы

Компилятор разрешает присвоить Cat в массив, который на самом деле является Dog[], но в runtime возникает ArrayStoreException. 
Это цена ковариантности массивов.

---

## 2. Ковариантность в дженериках (Generics)

### Дженерики в Java инвариантны по умолчанию, но можно сделать их ковариантными с помощью wildcards (`? extends X`).

### Без ковариантности (инвариантность):

```java
public class GenericsExample {
    
    public static void main(String[] args) {
        List<Dog> dogs = new ArrayList<>();
        dogs.add(new Dog());
        
        // ❌ Не компилируется - дженерики инвариантны
        // List<Animal> animals = dogs; // Ошибка компиляции
        
        // ❌ Этот метод тоже не работает
        // processAnimals(dogs); // Ошибка компиляции
    }
    
    static void processAnimals(List<Animal> animals) {
        for (Animal a : animals) {
            a.sound();
        }
    }
}
```

### С ковариантностью (wildcard `? extends X`):

```java
public class CovariantGenerics {
    
    public static void main(String[] args) {
        List<Dog> dogs = new ArrayList<>();
        dogs.add(new Dog());
        dogs.add(new Dog());
        
        // ✅ Теперь работает с wildcard
        processAnimalsCovariant(dogs);
        
        List<Animal> animals = new ArrayList<>();
        animals.add(new Dog());
        animals.add(new Cat());
        processAnimalsCovariant(animals); // Тоже работает
    }
    
    // Ковариантность через wildcard
    static void processAnimalsCovariant(List<? extends Animal> animals) {
        // Компилятор НЕ ЗНАЕТ точный тип list
        // Это может быть List<Dog>, List<Cat>, List<Animal>
        
        // Можно ЧИТАТЬ элементы как Animal
        for (Animal a : animals) {
            a.sound();
        }
        
        // ❌ НЕЛЬЗЯ добавлять новые элементы (кроме null)
        // animals.add(new Dog()); // Ошибка компиляции
        // animals.add(new Animal()); // Ошибка компиляции
        // animals.add(new Cat()); // Ошибка компиляции
        
        // Почему? Потому что мы не знаем точный тип списка.
        // Это может быть List<Dog>, а мы пытаемся добавить Cat

        // ✅ Единственное безопасное значение - null
        list.add(null); // null подходит для любого типа

        // ✅ Чтение - безопасно
        Animal a = list.get(0); // Любой элемент можно привести к Animal
    }
}
```

### Глубокая причина: Типовая безопасность:

```java
class Box<T> {
    private T value;
    
    void set(T value) { this.value = value; }
    T get() { return value; }
}

public class TypeSafety {
    public static void main(String[] args) {
        Box<Dog> dogBox = new Box<>();
        dogBox.set(new Dog());
        
        // Ковариантная ссылка
        Box<? extends Animal> animalBox = dogBox;
        
        // ❌ Почему нельзя animalBox.set(new Cat())?
        // Потому что animalBox может указывать на Box<Dog>
        // И тогда в Box<Dog> окажется Cat!
        
        // ✅ Но можно читать
        Animal animal = animalBox.get(); // Безопасно - Dog является Animal
        
        // Контравариантность (? super Dog) позволяет писать:
        Box<? super Dog> dogContainer = new Box<Animal>();
        dogContainer.set(new Dog()); // OK - Dog можно положить в Animal
        // dogContainer.set(new Cat()); // ❌ Нельзя - Cat не Dog
        
        Object obj = dogContainer.get(); // Возвращает Object
    }
}
```

---

## 3. Ковариантность в возвращаемых типах методов

### Java поддерживает ковариантность возвращаемых типов при переопределении методов:

```java
class AnimalFactory {
    Animal create() {
        System.out.println("AnimalFactory создает Animal");
        return new Animal();
    }
}

class DogFactory extends AnimalFactory {
    @Override
    Dog create() {  // Ковариантный возвращаемый тип
        System.out.println("DogFactory создает Dog");
        return new Dog();
    }
}

class CatFactory extends AnimalFactory {
    @Override
    Cat create() {  // Тоже ковариантный тип
        System.out.println("CatFactory создает Cat");
        return new Cat();
    }
}

public class ReturnTypeCovarianceDemo {
    public static void main(String[] args) {
        // Вариант 1: Ссылка DogFactory, переменная DogFactory
        DogFactory dogFactory1 = new DogFactory();
        Dog dog = dogFactory1.create(); // DogFactory.create() -> Dog

        // Вариант 2: Ссылка AnimalFactory, переменная DogFactory (полиморфизм)
        AnimalFactory factory = new DogFactory();
        Animal animal = factory.create(); // Вызывается DogFactory.create()!
        // Компилятор: вызов метода create() из AnimalFactory
        // Runtime: фактический объект DogFactory, поэтому вызывается DogFactory.create()
        // Результат: Dog, но присваивается в переменную Animal

        System.out.println("Тип объекта: " + animal.getClass()); // class Dog
        animal.sound(); // "Woof!" (полиморфный вызов)
    }
}
```

- Компилятор проверяет, что у AnimalFactory есть метод create(), возвращающий Animal
- В runtime определяется фактический тип объекта и вызывается соответствующая реализация
- Ковариантность возвращаемого типа разрешена с Java 5: переопределенный метод может возвращать подтип

---

## 4. Когда использовать ковариантность?

### Ковариантность полезна, когда вы только читаете данные из контейнера (producer):

```java
public class PracticalExample {
    
    // Пример 1: Подсчет суммы чисел
    static double sum(List<? extends Number> numbers) {
        double total = 0;
        for (Number n : numbers) {
            total += n.doubleValue();
        }
        return total;
    }
    
    // Пример 2: Вывод всех животных
    static void printAllAnimals(List<? extends Animal> animals) {
        for (Animal a : animals) {
            System.out.println(a.getClass().getSimpleName());
        }
    }
    
    public static void main(String[] args) {
        // Работает с разными типами
        List<Integer> ints = List.of(1, 2, 3);
        List<Double> doubles = List.of(1.5, 2.5, 3.5);
        
        System.out.println(sum(ints));     // 6.0
        System.out.println(sum(doubles));  // 7.5
        
        List<Dog> dogList = List.of(new Dog(), new Dog());
        printAllAnimals(dogList); // Выведет: Dog, Dog
    }
}
```

---

## PECS принцип (Joshua Bloch)

- ### `Producer Extends` - если коллекция производит элементы, используйте `? extends T`
- ### `Consumer Super` - если коллекция потребляет элементы, используйте `? super T`
- ### `Both` - если и то, и другое, не используйте `wildcards`
</details>

<br>

<details>
    <summary>
        <b>Стирание типов(Type Erasure)</b>
    </summary>

> Type erasure — это механизм в Java, при котором информация о generic-типах удаляется во время компиляции и заменяется на их границы (`bounds`).
> Если границы не обозначены, то информация о типе во время компиляции заменяется на Object (верхняя граница всех классов).

```java
// Исходный код (до компиляции)
public class Box<T> {
    private T value;
    
    public void set(T value) {
        this.value = value;
    }
    
    public T get() {
        return value;
    }
}

// После компиляции (стирания)
public class Box {
    private Object value; // T заменен на Object
    
    public void set(Object value) {
        this.value = value;
    }
    
    public Object get() {
        return value;
    }
}
```

### С ограничениями (bounds):

```java
// Исходный код
public class NumberBox<T extends Number> { // Ограничение
    private T value;

    public double getDouble() {
        int i = 2;
        i++ += ++i;
        return value.doubleValue();
    }
}

// После стирания
public class NumberBox {
    private Number value; // T заменен на Number, а не Object

    public double getDouble() {
        return value.doubleValue();
    }
}
```

###

В Java можно указывать несколько ограничений для generic-параметра! Синтаксис: <T extends A & B & C>

## Мостовые методы (Bridge Methods)

> Мостовые методы — это синтетические методы, которые компилятор генерирует автоматически для сохранения полиморфизма при использовании дженериков с наследованием.

Зачем они нужны?
Проблема возникает, когда переопределенный метод в подклассе имеет другой стираемый тип, чем метод в родительском классе.

### Пример 1: Простой случай с мостовым методом

```java
// Родительский класс с generic
class Parent<T> {
    public void set(T value) {
        System.out.println("Parent.set: " + value);
    }
}

// Дочерний класс с конкретным типом
class Child extends Parent<String> {
    @Override
    public void set(String value) { // Переопределяем с конкретным типом
        System.out.println("Child.set: " + value);
    }
}
```

После стирания типов (после компиляции):

```java
// Parent после стирания
class Parent {
    public void set(Object value) { // T -> Object
        System.out.println("Parent.set: " + value);
    }
}

// Child после стирания
class Child extends Parent {
    // Метод 1: Наш оригинальный метод
    public void set(String value) {
        System.out.println("Child.set: " + value);
    }
    
    // Метод 2: Мостовой метод (сгенерирован компилятором)
    public void set(Object value) {
        set((String) value); // Вызывает наш метод с кастом
    }
}
```

Проверка:

```java
import java.lang.reflect.Method;

public class BridgeMethodDemo {
    public static void main(String[] args) {
        Child child = new Child();

        // Посмотрим методы класса Child
        System.out.println("Методы класса Child:");
        for (Method method : Child.class.getDeclaredMethods()) {
            System.out.printf("%s - bridge: %s%n",
                    method,
                    method.isBridge());
        }

        // Вызов через родительский тип
        Parent<String> parent = child;
        parent.set("Hello"); // Вызовется Child.set через мостовой метод

        // Что происходит:
        // 1. parent.set("Hello") вызывает Parent.set(Object)
        // 2. Но parent ссылается на Child
        // 3. Вызывается Child.set(Object) - мостовой метод
        // 4. Мостовой метод вызывает Child.set(String)
    }
}
```
```text
Методы класса Child:
public void Child.set(java.lang.String) - bridge: false
public void Child.set(java.lang.Object) - bridge: true
```

### Пример 2: Мостовые методы для возвращаемых значений

```java
// Родительский generic класс
abstract class Converter<T> {
    public abstract T convert(String input);
}

// Дочерний класс с конкретным типом
class IntegerConverter extends Converter<Integer> {
    @Override
    public Integer convert(String input) {
        return Integer.parseInt(input);
    }
}
```

После стирания типов (после компиляции):

```java
abstract class Converter {
    public abstract Object convert(String input); // T -> Object
}

class IntegerConverter extends Converter {
    // Метод 1: Наш метод
    public Integer convert(String input) {
        return Integer.parseInt(input);
    }
    
    // Метод 2: Мостовой метод
    public Object convert(String input) {
        return convert(input); // Автоупаковка int -> Integer
    }
}
```

### Пример 3: generic интерфейсы

```java
interface Comparable<T> {
    int compareTo(T other);
}

class StringComparable implements Comparable<String> {
    // Метод 1: Наш метод
    @Override
    public int compareTo(String other) {
        return this.toString().compareTo(other);
    }

    // Метод 2: Мостовой метод
     public int compareTo(Object other) {
         return compareTo((String) other); // Вызывает наш метод с кастом
     }
}
```

### Пример 4: Проблемы без мостовых методов

```java
import java.util.*;

public class WithoutBridgeProblem {
    public static void main(String[] args) {
        // Представьте, что мостовых методов нет:
        
        // Стирание: List<String> -> List
        // Стирание: List<Integer> -> List
        
        List<String> stringList = new ArrayList<>();
        List rawList = stringList; // Raw type
        
        // Без type erasure это было бы невозможно
        rawList.add(123); // Добавляем Integer в List<String>!
        
        // При получении - ClassCastException
        String s = stringList.get(0); // Integer нельзя кастовать к String
    }
}
```

### Практический пример с коллекциями

```java
import java.util.*;

public class CollectionsBridge {

    public static void main(String[] args) {
        // Создаем специфичную коллекцию
        class StringList extends ArrayList<String> {
            @Override
            public boolean add(String s) {
                System.out.println("Добавляем строку: " + s);
                return super.add(s);
            }
        }

        StringList stringList = new StringList();
        List<String> list = stringList;
        List rawList = stringList;

        list.add("Hello");      // Вызовет StringList.add(String)
        rawList.add(123);       // Проблема! Integer в StringList. Будет исключение при получении
    }
}
```

### Type erasure и массивы

```java
public class ErasureAndArrays {
    public static void main(String[] args) {
        // ❌ Нельзя создать generic-массив
        // List<String>[] array = new List<String>[10]; // Ошибка компиляции
        
        // Почему? Потому что после стирания:
        // List<String> -> List
        // List<Integer> -> List
        // И компилятор не может обеспечить безопасность типов
        
        // Но можно так:
        List<?>[] array = new List<?>[10]; // OK
        array[0] = new ArrayList<String>();
        array[1] = new ArrayList<Integer>();
        
        // Или с unchecked warning
        @SuppressWarnings("unchecked")
        List<String>[] array2 = (List<String>[]) new List[10]; // Warning
    }
}
```

## Обход ограничений type erasure

### 1. Передача Class<T> для сохранения типа:

```java
public class TypeToken<T> {
    
    private final Class<T> type;
    
    public TypeToken(Class<T> type) {
        this.type = type;
    }
    
    public Class<T> getType() {
        return type;
    }
}

// Использование
TypeToken<String> token = new TypeToken<>(String.class);
System.out.println(token.getType()); // class java.lang.String
```

### 2. Super Type Token (GSON/Gson, Jackson):

```java
import java.lang.reflect.*;

abstract class TypeReference<T> {
    
    private final Type type;
    
    protected TypeReference() {
        Type superClass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }
    
    public Type getType() {
        return type;
    }
}

// Использование
TypeReference<List<String>> ref = new TypeReference<List<String>>() {};
System.out.println(ref.getType()); // java.util.List<java.lang.String>
```

## Несколько ограничений (Multiple Bounds)

> в Java можно указывать несколько ограничений для generic-параметра! Синтаксис: <T extends A & B & C>

### Работа с сортируемыми числами 

- Первое ограничение — используется для стирания типа
- Класс должен быть первым, если есть (нельзя extends Comparable<T> & Number)
- Интерфейсов может быть много

```java
// T должно быть Number И Comparable
public class SortedNumberContainer<T extends Number & Comparable<T>> {
    private T value;
    
    public SortedNumberContainer(T value) {
        this.value = value;
    }
    
    // Можем использовать методы Number
    public double getDoubleValue() {
        return value.doubleValue();
    }
    
    // И методы Comparable
    public int compareTo(T other) {
        return value.compareTo(other);
    }
    
    // Пример использования
    public static void main(String[] args) {
        // Integer подходит: extends Number implements Comparable<Integer>
        SortedNumberContainer<Integer> intContainer = 
            new SortedNumberContainer<>(42);
        
        // Double тоже подходит
        SortedNumberContainer<Double> doubleContainer = 
            new SortedNumberContainer<>(3.14);
        
        // BigDecimal тоже
        SortedNumberContainer<BigDecimal> decimalContainer = 
            new SortedNumberContainer<>(new BigDecimal("123.456"));
        
        // ❌ String не подходит - не Number
        // SortedNumberContainer<String> stringContainer; // Ошибка компиляции
        
        // ❌ AtomicInteger не подходит - Number, но не Comparable
        // SortedNumberContainer<AtomicInteger> atomicContainer; // Ошибка
        
        // Используем методы
        System.out.println("Integer value: " + intContainer.getDoubleValue());
        System.out.println("Compare 42 and 10: " + 
            intContainer.compareTo(10));
    }
}

// Что происходит после стирания?
// T extends Number & Comparable<T> -> заменяется на Number (первое ограничение)
// Но компилятор помнит, что есть и Comparable
```
</details>

<br>

<details>
    <summary>
        <b>Type witness</b>
    </summary>

# Type witness

> Type witness (свидетель типа) — специальный синтаксис Java для явного указания generic-типов при вызове generic-метода.
>
> Встречается в `Collections`, `Stream API`, `Optional` и других generic-библиотеках.

```java
Comparator.<Map.Entry<String, Double>>comparing(...)
          ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
          Type witness - явное указание generic-типа
```

### Аналогия с обычными generic-классами:

```java
// Обычные generic-классы - тип в diamond operator
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();

// Generic-методы - тип перед именем метода
Collections.<String>emptyList();    // Type witness
Arrays.<Integer>asList(1, 2, 3);    // Type witness
```

## Где встречается

### Collections utility methods:

```java
// Без type witness - работает
List<String> empty1 = Collections.emptyList();

// С type witness - явное указание
List<String> empty2 = Collections.<String>emptyList();

// Когда нужен type witness:
List<?> list = Collections.<String>emptyList();  // ✅
// List<?> list = Collections.emptyList();       // ❌ Может не компилироваться
```

### 2. Arrays.asList():

```java
// Компилятор выводит тип
List<Integer> list1 = Arrays.asList(1, 2, 3);

// Явное указание (редко нужно)
List<Number> list2 = Arrays.<Number>asList(1, 2, 3.0);
```

### 3. Optional.ofNullable() с casting:

```java
// Когда нужен каст с generic-типом
Optional<String> opt = Optional.<String>ofNullable(getObject());
```

### 4. Stream API:

```java
// В Stream.of() когда нужен общий тип
Stream.<Number>of(1, 2.5, 3L);  // Все элементы как Number

// В Collectors.collectingAndThen()
Collectors.<String, ?, List<String>>collectingAndThen(...)
```

### 5. Comparator.comparing()

```java
// ❌ Ошибка: cannot infer type-variable(s) T
.sorted(Comparator.comparing(Map.Entry::getValue).reversed())

// ✅ Явно указываем, что comparing() работает с Map.Entry<String, Double>
.sorted(Comparator.<Map.Entry<String, Double>>comparing(Map.Entry::getValue).reversed())
```

## Используйте type witness, когда:

- ### Компилятор не может вывести тип (ошибка "cannot infer type-variable")
- ### Нужно явно указать тип для читаемости
- ### Работаете с wildcard-типами (`?`, `? super T`, `? extends T`)
</details>