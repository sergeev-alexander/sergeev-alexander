# Sealed Classes (Java 17+)

> Цель: явно контролировать иерархию наследования - кто может наследоваться, и как.

## ✅ Базовый синтаксис

```java
// В одном файле (обычно):
public sealed class Shape
        permits Circle, Square, Rectangle { }  // <- permits = явный список наследников

final class Circle extends Shape { }          // ✅ final - лист иерархии
sealed class Rectangle extends Shape          // ✅ может иметь подтипы
        permits FilledRectangle { }            // <- свои разрешённые подтипы
final class FilledRectangle extends Rectangle { }

non-sealed class Square extends Shape { }     // ✅ «снять печать» - открыт для любых наследников
```

### `permits` можно опустить, если все подтипы объявлены в том же файле - тогда компилятор выводит их автоматически.
### ⚠️ Подтипы должны быть `final`, `sealed` или `non-sealed` - никаких «обычных» (public class X extends Shape)!

### 3 варианта для подтипов:

- `final` - Никто не может наследоваться от этого класса.
- `sealed` - Может иметь подтипы - но только перечисленные в `permits`.
- `non-sealed` - Снимает ограничение - дальше иерархия открыта (как в обычных классах). Допустим только для подтипов sealed-классов, не для корневых.

## ✅ Sealed interfaces

### Работают точно так же - и даже гибче:

```java
// 1. Объявляем ЗАПЕЧАТАННЫЙ интерфейс - как "сумма типов"
//    → "Выражение - это ЛИБО Const, ЛИБО Add, ЛИБО Mul. Больше ничего."
public sealed interface Expr
        permits Const, Add, Mul { }  // ← явно разрешаем ТОЛЬКО эти 3 типа

// 2. Const - это константа (лист дерева)
//    → record автоматически даёт private final int value + геттер value() + equals/hashCode/toString
record Const(int value) implements Expr { }

// 3. Add - бинарная операция "сложение"
//    → содержит два ПОДВЫРАЖЕНИЯ (рекурсивно!)
record Add(Expr left, Expr right) implements Expr { }
//    Пример: Add(Const(2), Mul(Const(3), Const(4))) → 2 + (3 * 4)

// 4. Mul - бинарная операция "умножение"
record Mul(Expr left, Expr right) implements Expr { }
```

Как использовать: исчерпывающий switch (Java 21+)

```java
// Вычисление значения выражения
double eval(Expr expr) {
    return switch (expr) {
        // 1. Сопоставление с образцом: если expr - Const → приводим к Const и связываем с c
        case Const c -> c.value();  // просто возвращаем число

        // 2. Если expr - Add → связываем с a, рекурсивно вычисляем подвыражения
        case Add a -> eval(a.left()) + eval(a.right());

        // 3. Если expr - Mul
        case Mul m -> eval(m.left()) * eval(m.right());

        // ❗ default НЕ НУЖЕН!
        // Компилятор ЗНАЕТ: Expr может быть ТОЛЬКО Const | Add | Mul
        // -> switch исчерпывающий -> код безопасен!
    };
}
```

Допустим, вы решили добавить деление:

```java
record Div(Expr left, Expr right) implements Expr { } // <- но ЗАБЫЛИ добавить в permits!
// → ОШИБКА КОМПИЛЯЦИИ:
//   "class Div is not allowed to extend sealed class Expr"
```

## ⚠️ Ограничения
| Ограничение                                                       | Причина                                                                                                  |
|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Подтипы должны быть в том же модуле                               | `sealed` работает на уровне модуля (module-info.java)                                                    |
| Подтипы должны быть в том же пакете, если не указано иное         | При permits можно указывать полные имена из других пакетов, но они всё равно должны быть в том же модуле |
| `permits` можно опустить только если все подтипы - в том же файле | Иначе - обязательно перечислить                                                                          |
| Нельзя сделать sealed class `final`                               | Противоречие: `final` = нельзя наследовать, `sealed` = можно, но ограниченно                             |

## ✅ Best Practices

- Комбинируйте с `record` - идеально для ADT (алгебраических типов данных):
- Используйте `non-sealed` осторожно - вы теряете контроль над иерархией ниже.
- Избегайте глубоких иерархий - `sealed` лучше всего работает на 1–2 уровнях.

```java
// 1. Result<T> - "результат операции"
//    → Может быть УСПЕХ с данными типа T, или ОШИБКА со строкой (можно заменить на Exception)
public sealed interface Result<T>
        permits Success, Failure { }  // ← только два варианта

// 2. Успех: обёртка вокруг полезных данных
//    → T - обобщённый тип: может быть String, User, List<Item> и т.д.
record Success<T>(T value) implements Result<T> { }

// 3. Ошибка: обёртка вокруг описания ошибки
record Failure<T>(String error) implements Result<T> { }
//    T здесь - "заглушка", чтобы тип Result<T> был параметризован.
//    Например: Result<String> может быть Success<String> ИЛИ Failure<String>.
```

Как использовать:

```java
// Функция, которая может завершиться ошибкой - но не кидает исключение
Result<Integer> parseToInt(String s) {
    try {
        return new Success<>(Integer.parseInt(s));  // ✅ успех
    } catch (NumberFormatException e) {
        return new Failure<>("Не число: " + s);     // ❌ ошибка
    }
}

// Обработка результата
void demo() {
    Result<Integer> res = parseToInt("123");

    // Вариант 1: switch (исчерпывающе!)
    String message = switch (res) {
        case Success<Integer> s -> "Успех: " + s.value();
        case Failure<Integer> f -> "Ошибка: " + f.error();
        // default не нужен!
    };

    System.out.println(message);

    // Вариант 2: методы-хелперы (можно добавить в интерфейс Result)
    res = res.map(v -> v * 2)           // Success(123) → Success(246); Failure → Failure
             .recover(e -> 0);          // Failure → Success(0)

    // Вариант 3: if instanceof (Java 17+)
    if (res instanceof Success<Integer> s) {
        System.out.println("Значение: " + s.value());
    } else if (res instanceof Failure<Integer> f) {
        System.err.println("Ошибка: " + f.error());
    }
}
```