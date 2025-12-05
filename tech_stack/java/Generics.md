# Generics

<details>
    <summary>
        <b>Ковариантность (covariance)</b>
    </summary>

> Суть ковариантности - это сохранение иерархии наследования в прямом направлении для составных типов (коллекций, массивов, обобщений).

Если `Child` - подтип `Parent`:

- Ковариантность: Контейнер<Child> - подтип Контейнер<Parent>
- Контравариантность (обратное): Контейнер<Parent> — подтип Контейнер<Child>
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
- ### `Both` — если и то, и другое, не используйте `wildcards`
</details>