# Java Optional

`Optional<T>` - это контейнер-обертка для единственного значения, которое может быть null. 
Его главная цель - явно обозначать возможность отсутствия значения и заставлять разработчика осознанно обрабатывать эту ситуацию, избегая знаменитого `NullPointerException`.

<details>
    <summary>
        <b>Создание Optional</b>
    </summary>


|            Метод             |                         Описание                         |                                         Когда использовать                                         |
|:----------------------------:|:--------------------------------------------------------:|:--------------------------------------------------------------------------------------------------:|
|      `Optional.empty()`      |                Создает пустой `Optional`.                |                        Когда метод должен вернуть "отсутствие результата".                         |
|     `Optional.of(value)`     |         Создает `Optional` с не-null значением.          |    Когда вы уверены, что значение не null. Выбросит `NullPointerException`, если value = null.     |
| `Optional.ofNullable(value)` | Создает `Optional` с значением, которое может быть null. | Если значение может быть null, это безопасный способ. Если value = null, вернет `Optional.empty()` |

```java
Optional<String> emptyOpt = Optional.empty();
Optional<String> nameOpt = Optional.of("Anna"); // "Anna" не может быть null
Optional<String> maybeNameOpt = Optional.ofNullable(getName()); // getName() может вернуть null
```

- `of()` - когда уверены, что значение не `null`.
- `ofNullable()` - когда могло быть `null` (из legacy-API, БД, внешнего вызова).
</details>

<details>
    <summary>
        <b>Проверка и извлечение значений</b>
    </summary>

- Никогда не используйте get() без предварительной проверки isPresent()


|          Метод          |                                          Описание                                          |                                                    Риски                                                     |
|:-----------------------:|:------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------------:|
|      `isPresent()`      |                       	Возвращает `true`, если значение внутри есть.                       |                         	Сам по себе не очень полезен, ведет к императивному стилю.                          |
| `isEmpty()` (Java 11+)  |                       Возвращает `true`, если значение отсутствует.                        |                                          Аналогично `isPresent()`.                                           |
|         `get()`         |                            	Извлекает значение, если оно есть.                             |            КИДАЕТ ИСКЛЮЧЕНИЕ `NoSuchElementException`, если значения нет. Избегайте этого метода!            |
|  `ifPresent(Consumer)`  |                  Выполняет лямбда-выражение, если значение присутствует.                   |                      	Безопасная и рекомендуемая замена связки `isPresent() -> get()`.                       |
|    `orElse(T other)`    |               Возвращает значение, если оно есть, иначе возвращает `other`.                | 	`other` вычисляется ВСЕГДА, даже если значение есть. Используйте, когда `other` уже создан/легко вычисляем. |
|  `orElseGet(Supplier)`  | Возвращает значение, если оно есть, иначе вызывает `Supplier` и возвращает его результат.	 |              Ленивая версия `orElse()`. Используйте, когда создание запасного значения дорогое.              |
|     `orElseThrow()`     |         Возвращает значение, если оно есть, иначе кидает `NoSuchElementException`.         |                                   Улучшенная, более явная версия `get()`.                                    |
| `orElseThrow(Supplier)` |    	Возвращает значение, если оно есть, иначе кидает исключение, созданное `Supplier`.     |                             Позволяет кидать свое, более специфичное исключение.                             |

```java
// ПЛОХО: Старый, небезопасный стиль
Optional<String> opt = ...;
if (opt.isPresent()) { // Проверка вручную
    String value = opt.get(); // Опасное извлечение
    System.out.println(value);
}

// ХОРОШО: Функциональный стиль
opt.ifPresent(value -> System.out.println(value)); // Значение будет если есть

// Получение значения с fallback'ом
String value1 = opt.orElse("default"); // "default" создается всегда
String value2 = opt.orElseGet(() -> expensiveOperation()); // expensiveOperation() только если нужно
String value3 = opt.orElseThrow(() -> new MyCustomException("Data not found")); // С кастомным исключением
```
</details>

<details>
    <summary>
        <b>Преобразование и фильтрация (цепочки методов)</b>
    </summary>

- `map(Function)` - Преобразует значение внутри `Optional`, если оно есть. Если значения нет, возвращает `Optional.empty()`.
- `flatMap(Function)` - Как map, но функция преобразования уже возвращает `Optional`. Используется, чтобы избежать вложенности `Optional<Optional<T>>`.
- `filter(Predicate)` - Если значение присутствует и удовлетворяет условию-предикату, возвращает этот `Optional`, иначе - `Optional.empty()`.

```java
// Допустим, у нас есть метод, возвращающий Optional<User>
Optional<User> userOpt = findUserById(123);

// map: Получить имя пользователя (из User -> String)
Optional<String> nameOpt = userOpt.map(User::getName);

// flatMap: Если метод getUserAddress() сам возвращает Optional<Address>
Optional<Address> addressOpt = userOpt.flatMap(User::getUserAddress);

// filter: Найти пользователя только если он администратор
Optional<User> adminOpt = userOpt.filter(user -> user.isAdmin());

// Построение цепочек (частая практика)
String result = findUserById(123)
    .filter(User::isActive)
    .flatMap(User::getEmail)
    .orElse("no-email@example.com");
```
</details>

<details>
    <summary>
        <b>Типичные ошибки и лучшие практики</b>
    </summary>

### Что НЕ делать:

- Не использовать `Optional` для полей класса. Это создает избыточность. Для полей просто используйте `null`.
- Не передавать `Optional` в параметры методов. Это усложняет API и не дает преимуществ. Используйте перегрузку методов.
- Не использовать `Optional` в коллекциях. Коллекции и так могут быть пустыми. Используйте `Collection.isEmpty()`.
- Не проверять `Optional` на `null`. Метод, возвращающий `Optional`, ВСЕГДА должен возвращать экземпляр, а не `null`. `Optional.of(null)` - выбросит `NPE`, поэтому единственный способ получить `null` - это сделать это вручную.

```java
// ОЧЕНЬ ПЛОХО
public Optional<String> badMethod() {
return null; // Никогда так не делайте!
}
```
Избегайте `isPresent() -> get()`. Почти всегда эту комбинацию можно заменить на `ifPresent`, `orElse`, etc.

### Лучшие практики:

- Используйте как возвращаемый тип для явного указания на возможное отсутствие значения.
- Предпочитайте функциональные методы (`ifPresent`, `map`, `orElseGet`) императивным проверкам.
- Используйте цепочки методов для построения ясного и декларативного потока преобразований.
- Всегда предоставляйте запасное значение через `orElse` / `orElseGet` / `orElseThrow` в конце цепочки, чтобы получить реальный объект.

```java
// У тебя есть Optional<T> opt. Что делать?

// Нужно просто выполнить действие, если значение есть?
opt.ifPresent(value -> ...);

// Нужно получить значение или замену, если его нет?
T value = opt.orElse(...); // Замена простая
T value = opt.orElseGet(() -> ...); // Замена "дорогая" в создании
T value = opt.orElseThrow(() -> ...); // Без замены, только исключение

// Нужно преобразовать значение, если оно есть?
Optional<R> newOpt = opt.map(value -> ...);
Optional<R> newOpt = opt.flatMap(value -> ...); // Если твой маппер возвращает Optional

// Нужно проверить значение по условию?
Optional<T> filteredOpt = opt.filter(value -> ...);
```
</details>

<details>
    <summary>
        <b>Optional в стримах - подводные камни</b>
    </summary>

## ❌ Optional как элемент стрима

```java
List<Optional<String>> list = ...;
list.stream()
    .map(opt -> opt.get())   // <- NPE, если хоть один empty()
    .forEach(System.out::println);
```

### ✅ Как обрабатывать:

```java
list.stream()
    .filter(Optional::isPresent)    // <- убираем пустые
    .map(Optional::get)             // <- теперь безопасно
    .forEach(System.out::println);

// Или лучше - используем flatMap:
list.stream()
    .flatMap(Optional::stream)      // Java 9+: empty() -> пустой стрим, present -> 1 элемент
    .forEach(System.out::println);
```

## ❌ Возврат Optional из map, filter и др.

### Сценарий: у нас есть List<User>, и у User есть метод:

```java
class User {
    Optional<String> getEmail();  // <- возвращает Optional<String>
}
```

### ❌ Неправильно

```java
users.stream()
     .map(User::getEmail)       // getEmail() -> Optional<String>
     .forEach(System.out::println); // <- напечатает: Optional[alice@example.com], Optional.empty...
```

### ✅ Правильно:

### Вариант 1:
```java
users.stream()
     .map(User::getEmail)       // Stream<User> -> Stream<Optional<String>>
    // // Optional.stream() - возвращает пустой Stream<T> - если Optional.empty() или стрим из одного элемента - если Optional.of(value) 
     .flatMap(Optional::stream) // Stream<Optional<String>> -> Stream<String>
     .forEach(System.out::println);
```

Здесь мы сначала преобразуем всех пользователей в `Optional<String>`, получая стрим `Stream<Optional<String>>`,
а затем распаковываем этот стрим в стрим строк.

- Плюс: хорошо читается, если getEmail() - короткий метод-ссылка.
- Минус: создаётся промежуточный Stream<Optional<String>> (но JIT его оптимизирует).

### Вариант 2:

```java
users.stream()
     .flatMap(u -> u.getEmail().stream())  // Stream<User> -> Stream<String> за один шаг
     .forEach(System.out::println);
```

Здесь мы за одну операцию превращаем User в 0 или 1 строку, минуя промежуточный Optional.

- Плюс: чуть эффективнее (нет промежуточного Stream<Optional>), и удобно, если нужно использовать u несколько раз:

```java
.flatMap(u -> {
    if (u.isActive()) {
        return u.getEmail().stream();
    } else {
        return Stream.empty();
    }
})
```

- Минус: чуть длиннее, если лямбда простая.

## ❌ Optional в терминальных операциях

```java
import java.util.ArrayList;

Optional<User> user = users.stream().findFirst(); // ✅ правильно

// Но НЕЛЬЗЯ:
List<Optional<User>> users = new ArrayList<>();
Optional<User> first = users.stream().findFirst(); // <- Optional<Optional<User>> - ошибка проектирования
```

## Если у вас `List<Optional<T>>` - пересмотрите API. Лучше: `List<T>` с фильтрацией, или `Stream<T>` + `flatMap(Optional::stream)`
</details>