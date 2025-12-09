# Анонимные классы (Anonymous Classes)

> Локальный класс без имени, объявляемый прямо при создании экземпляра - обычно для реализации интерфейса или расширения класса «на лету».

- ### Отсутствие имени:
  - Анонимные классы не имеют явного имени и создаются непосредственно в месте своего использования.
- ### Реализация интерфейсов или расширение классов:
  - Анонимные классы часто используются для реализации интерфейсов или расширения абстрактных классов.
- ### Ограниченная область видимости:
  - Анонимные классы обычно создаются в контексте конкретного использования и могут использоваться только в этом месте.
- ### Краткость и удобство:
  - Они обеспечивают краткий синтаксис для создания объектов с минимальным объемом кода.

## ✅ Базовый синтаксис

```java
// 1. Реализация интерфейса (самый частый случай)
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Анонимный Runnable");
    }
};
// Компилятор создаёт файл OuterClass$1.class, наследующий Runnable / 


// 2. Расширение класса (редко)
Thread t = new Thread() {
    @Override
    public void run() {
        System.out.println("Анонимный Thread");
    }
};
t.start(); // Вывод появится асинхронно, в отдельном потоке.
// Компилятор создаёт файл OuterClass$1.class, наследующий Thread.
```

## Когда ещё используются (практически)
| Сценарий                                              | Пример                                                                             | Комментарий                                                         |
|-------------------------------------------------------|------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| Инициализация коллекций (устаревшее, но встречается)  | new HashSet<>() {{ add("a"); add("b"); }}                                          | «Double-brace initialization» - опасно (утечки памяти, сериализация) |
| Локальные захваты final / effectively final переменных | int x = 42; Runnable r = new Runnable() { void run() { System.out.println(x); } }; | ✅ работает - x effectively final                                    |
| Переопределение одного метода у класса с конструктором | Button b = new Button("OK") { @Override protected void onClick() { ... } };        | Если нет лямбд / удобнее, чем наследование                          |
| Unit-тесты (Mockito, TestNG - редко)                  | doAnswer(new Answer<Void>() { ... })                                               | В основном вытеснено лямбдами и when().thenReturn()                 |

## Расширение абстрактного класса с помощью анонимного класса

```java
abstract class Animal {
    // Может иметь и реализованные, и нереализованные методы
    public void breathe() {
        System.out.println("Дышит");
    }

    // Абстрактный метод — должен быть реализован в подклассе
    public abstract void makeSound();
}
```

От Animal нельзя создать объект напрямую:

```java
new Animal(); // ❌ Ошибка: "Animal is abstract; cannot be instantiated"
```

Но можно создать подкласс, реализующий все abstract-методы:

```java
class Dog extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Гав!");
    }
}

new Dog(); // ✅
```

Вместо того чтобы писать именованный class Dog, можно создать подкласс прямо «на месте»:

```java
Animal cat = new Animal() {  // <- обратите внимание: new Animal() { ... }
    @Override
    public void makeSound() {
        System.out.println("Мяу!");
    }
};

cat.makeSound(); // -> Мяу!
cat.breathe();   // -> Дышит (унаследовано от Animal)
```
Компилятор создаёт анонимный класс YourClass$1 extends Animal, реализующий makeSound().

## ⚠️ Подводные камни (частые ошибки)

### Захват изменяемых переменных - запрещён

```java
int counter = 0;
Runnable r = new Runnable() {
    @Override
    public void run() {
        counter++; // ❌ ОШИБКА: "Variable used in lambda/anonymous class should be final or effectively final"
    }
};
```

Решение: используйте AtomicInteger, массив [0], или делать именованный класс.

### `this` внутри анонимного класса - ссылается на НЕГО, а не на внешний класс

```java
class Outer {
    void method() {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(this);           // Ссылается на экземпляр Runnable
                System.out.println(Outer.this);     // Так можно добраться до внешнего экземпляра Outer
            }
        };
    }
}
```

### Утечки памяти при double-brace и нестатических ссылках

```java
public class Leaky {
    private final List<String> data = new ArrayList<>();

    public List<String> getCopy() {
        return new ArrayList<>() {{        // ← анонимный класс
            addAll(data);                  // ← захватывает this (Leaky)
        }};
    }
}
```

Допустим, вы делаете так:

```java
Leaky leaky = new Leaky();
List<String> list = leaky.getCopy();
cache.put("key", list); // ← кладём в долгоживущий кэш
leaky = null;           // ← "забываем" leaky
```

Экземпляр анонимного класса держит ссылку на Leaky -> если список «утечёт» в кэш - Leaky не соберётся GC.

Объект leaky не может быть собран GC, потому что:

- `list` - это `Leaky$1`
- `Leaky$1` содержит ссылку `this$0` -> на `leaky`
- `leaky` живёт, пока жив `list`.

## Современная альтернатива лямбдам - только если:

- Нужно переопределить >1 метода
- Нужен доступ к `protected` методам суперкласса
- Нужен явный тип для перегрузки методов