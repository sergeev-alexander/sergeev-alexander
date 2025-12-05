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

public class CovarianceExample {
    
    public static void main(String[] args) {
        // ✅ Ковариантность массивов
        Dog[] dogs = new Dog[3];
        dogs[0] = new Dog();
        
        // Можно присвоить массив собак переменной типа "массив животных"
        Animal[] animals = dogs; // Ковариантность!
        
        // Проблема: компилятор это разрешает, но может привести к исключению
        try {
            animals[1] = new Cat(); // ❌ ArrayStoreException в runtime!
        } catch (ArrayStoreException e) {
            System.out.println("Ошибка: " + e.getMessage());
        }
        
        // ✅ Работает корректно
        animals[2] = new Dog(); // OK, Dog подходит для массива Dog
    }
}
```
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
    }
}
```

## 3. Ковариантность в возвращаемых типах методов

### Java поддерживает ковариантность возвращаемых типов при переопределении методов:

```java
class AnimalFactory {
    
    Animal create() {
        return new Animal();
    }
}

class DogFactory extends AnimalFactory {
    
    // ✅ Ковариантный возвращаемый тип
    @Override
    Dog create() {  // Возвращаем Dog вместо Animal
        return new Dog();
    }
}

public class ReturnTypeCovariance {
    
    public static void main(String[] args) {
        DogFactory factory = new DogFactory();
        Dog dog = factory.create(); // Получаем Dog, не нужно кастовать
        
        // Или через родительский тип
        AnimalFactory animalFactory = new DogFactory();
        Animal animal = animalFactory.create(); // На самом деле Dog
        animal.sound(); // "Woof!"
    }
}
```

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

## PECS принцип (Joshua Bloch)

- ### `Producer Extends` - если коллекция производит элементы, используйте `? extends T`
- ### `Consumer Super` - если коллекция потребляет элементы, используйте `? super T`
</details>