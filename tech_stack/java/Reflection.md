## Java Reflection API

Reflection — это механизм, позволяющий исследовать и модифицировать структуру и поведение классов, объектов, методов и полей во время выполнения (Runtime), а не на этапе компиляции.

### Основной класс: `java.lang.Class<T>`

Это отправная точка для любого рефлексивного действия. Получить объект Class можно несколькими способами:

```java
// 1. Через литерал класса (на этапе компиляции)
Class<String> stringClass = String.class;

// 2. Через объект (во время выполнения)
String str = "Hello";
Class<?> strClass = str.getClass();

// 3. По полному имени класса (чаще всего используется в Reflection)
Class<?> clazz = Class.forName("java.lang.String"); // Может выбросить ClassNotFoundException

// 4. Для примитивных типов и void
Class<?> intClass = int.class;
Class<?> voidClass = void.class;
```

### Создание экземпляров класса

```java
Class<?> clazz = Class.forName("com.example.MyClass");

// 1. Использование newInstance() (Устарел с Java 9)
MyClass obj = (MyClass) clazz.newInstance();

// 2. Использование Constructor.newInstance() (Предпочтительный способ)
Constructor<?> constructor = clazz.getConstructor(); // Получаем конструктор по умолчанию
MyClass obj = (MyClass) constructor.newInstance();

// 3. С передачей параметров в конструктор
Constructor<?> constructorWithParams = clazz.getConstructor(String.class, int.class);
MyClass obj = (MyClass) constructorWithParams.newInstance("Arg1", 42);
```

### Работа с полями (Fields)

```java
Class<?> clazz = MyClass.class;
MyClass obj = new MyClass();

// Получение информации о полях
Field[] publicFields = clazz.getFields(); // Все public поля (включая унаследованные)
Field[] allFields = clazz.getDeclaredFields(); // Все поля, объявленные именно в этом классе

// Получение конкретного поля
Field field = clazz.getDeclaredField("fieldName");

// Доступ к приватным полям
field.setAccessible(true); // Критически важный шаг для не-public полей!

// Чтение и запись значения поля
Object value = field.get(obj); // Прочитать значение
field.set(obj, newValue); // Установить новое значение

// Для статических полей передавайте null вместо объекта
field.set(null, staticValue);
```

### Работа с методами (Methods)

```java
Class<?> clazz = MyClass.class;
MyClass obj = new MyClass();

// Получение информации о методах
Method[] publicMethods = clazz.getMethods(); // Все public методы (включая унаследованные)
Method[] allMethods = clazz.getDeclaredMethods(); // Все методы, объявленные в этом классе

// Получение конкретного метода
Method method = clazz.getDeclaredMethod("methodName", param1Type.class, param2Type.class);

// Вызов метода
method.setAccessible(true); // Для не-public методов
Object returnValue = method.invoke(obj, arg1, arg2); // Вызов метода на объекте obj

// Вызов статического метода
returnValue = method.invoke(null, arg1, arg2);
```

### Работа с конструкторами (Constructors)

```java
Class<?> clazz = MyClass.class;

// Получение всех конструкторов
Constructor<?>[] constructors = clazz.getDeclaredConstructors();

// Получение конкретного конструктора по типам параметров
Constructor<?> constructor = clazz.getDeclaredConstructor(String.class, int.class);

// Создание экземпляра
constructor.setAccessible(true); // Для не-public конструкторов
MyClass instance = (MyClass) constructor.newInstance("test", 123);
```

### Аннотации через Reflection

```java
Class<?> clazz = MyClass.class;

// Проверка наличия аннотации
if (clazz.isAnnotationPresent(MyAnnotation.class)) {
    MyAnnotation annotation = clazz.getAnnotation(MyAnnotation.class);
    // Работа с аннотацией
}

// Аннотации на полях, методах и т.д.
Field field = clazz.getDeclaredField("fieldName");
if (field.isAnnotationPresent(MyFieldAnnotation.class)) {
    // ...
}
```
### Важные нюансы и предупреждения

- Производительность: Операции через `Reflection` значительно медленнее прямых вызовов. 
Не используйте в критичных к производительности местах.

- Безопасность: `Reflection` обходит инкапсуляцию (с помощью `setAccessible(true)`). 
Это может быть запрещено Security Manager'ом.

- Типизация: Компилятор не может проверить типы, поэтому много предупреждений и приведений типов (cast). 
Ошибки проявляются только во время выполнения.

- Исключения: Многие методы `Reflection` бросают проверяемые исключения (`ClassNotFoundException`, `NoSuchMethodException`, `IllegalAccessException` и т.д.), которые нужно обрабатывать.

### Простой пример

```java
import java.lang.reflect.*;
import java.util.*;

public class ReflectionDemo {
    public static void main(String[] args) throws Exception {
        // 1. Получаем класс
        Class<?> clazz = Class.forName("java.util.ArrayList");
        
        // 2. Создаем экземпляр
        List<String> list = (List<String>) clazz.getDeclaredConstructor().newInstance();
        
        // 3. Получаем метод add и вызываем его
        Method addMethod = clazz.getMethod("add", Object.class);
        addMethod.invoke(list, "Hello, Reflection!");
        
        // 4. Получаем метод size и вызываем его
        Method sizeMethod = clazz.getMethod("size");
        int size = (int) sizeMethod.invoke(list);
        
        // 5. Альтернатива: работа с приватным полем size
        // Field sizeField = clazz.getDeclaredField("size");
        // sizeField.setAccessible(true);
        // System.out.println("Size from field: " + sizeField.get(list));
        
        System.out.println("Содержимое списка: " + list); // [Hello, Reflection!]
        System.out.println("Размер списка через reflection: " + size);
    }
}
```
