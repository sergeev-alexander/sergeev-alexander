# CSS Selectors

## Содержание

1. [Введение](#1-введение)
2. [Базовый синтаксис и селекторы](#2-базовый-синтаксис-и-селекторы)
3. [Комбинаторы и навигация по DOM](#3-комбинаторы-и-навигация-по-dom)
4. [Псевдо-классы и функции для автоматизации](#4-псевдо-классы-и-функции-для-автоматизации)
5. [Типовые шаблоны локаторов для Selenium/Selenide](#5-типовые-шаблоны-локаторов-для-seleniumselenide)
6. [Оптимизация: производительность и стабильность](#6-оптимизация-производительность-и-стабильность)

---

# 1. Введение

> CSS Selectors — язык запросов, который определяет правила для выбора одного или нескольких HTML-элементов на веб-странице на основе их атрибутов, структуры, состояния или положения в DOM-дереве.

- **Производительность** — нативный движок браузера `querySelectorAll()` обрабатывает CSS-селекторы быстрее, чем парсинг XPath-выражений, что критично при частых поисках элементов в динамических интерфейсах.
- **Лаконичность синтаксиса** — типовой локатор `input[name='email']` короче и читаемее эквивалента XPath `//input[@name='email']`, что упрощает поддержку тестов.
- **Ограничения навигации** — в отличие от XPath, CSS не поддерживает оси `parent::`, `preceding-sibling::` и поиск по текстовому содержимому узла, что требует альтернативных подходов.
- **Интеграция в инструменты**:
    - Selenium: `driver.findElement(By.cssSelector("div.container > input"))`
    - Selenide: `$(By.cssSelector("div.container > input"))` или `$("div.container > input")`
- **Best Practice**: выбирайте CSS для простых и стабильных локаторов по атрибутам/классам; используйте XPath, когда нужна навигация к родителям или поиск по тексту.

## Когда CSS предпочтительнее XPath

| Критерий             | CSS Selectors                   | XPath                             |
|:---------------------|---------------------------------|-----------------------------------|
| Скорость выполнения  | Быстрее (нативный движок)       | Медленнее (парсинг выражения)     |
| Синтаксис            | Лаконичный, интуитивный         | Более многословный                |
| Поиск по тексту      | ❌ Не поддерживается             | ✅ `//button[text()='Отправить']`  |
| Навигация к родителю | ❌ Требует обходных путей        | ✅ `/parent::div`                  |
| Поддержка в браузере | ✅ `document.querySelectorAll()` | ❌ Требует библиотеку/движок       |

## Примеры базовых селекторов

```html
<!-- Поиск по тегу: все <input> на странице -->
<!-- CSS: input -->
<!-- XPath: //input -->
<form>
    <input name="login">
    <input name="password">
</form>

<!-- Поиск по классу: элементы с классом 'btn-primary' -->
<!-- CSS: .btn-primary -->
<!-- XPath: //*[@class='btn-primary'] (хрупко!), лучше //*[contains(@class, 'btn-primary')] -->
<button class="btn btn-primary">Отправить</button>
<div class="btn-primary disabled">Загрузка...</div>

<!-- Поиск по ID: уникальный элемент -->
<!-- CSS: #submit-btn -->
<!-- XPath: //*[@id='submit-btn'] -->
<button id="submit-btn" type="submit">Готово</button>

<!-- Поиск по атрибуту: точное совпадение -->
<!-- CSS: input[type='email'] -->
<!-- XPath: //input[@type='email'] -->
<input type="email" name="user_email" required>

<!-- Поиск по атрибуту: частичное совпадение (содержит слово) -->
<!-- CSS: [class~='error'] — элемент имеет класс 'error' среди прочих -->
<!-- XPath: //*[contains(concat(' ', @class, ' '), ' error ')] -->
<span class="field error highlight">Неверный формат</span>

<!-- Поиск по префиксу/суффиксу атрибута -->
<!-- CSS: [id^='user_'] — id начинается с 'user_' -->
<!-- CSS: [href$='.pdf'] — ссылка на PDF-файл -->
<!-- XPath: //*[starts-with(@id, 'user_')] / //*[substring(@href, string-length(@href)-3) = '.pdf'] -->
<a id="user_123_profile" href="/docs/report.pdf">Скачать</a>
```

## Ограничения CSS: что нельзя сделать напрямую

- **Нет навигации к родительскому элементу** — если нужно найти `<form>` по вложенному `<input>`, в CSS придётся строить селектор «сверху вниз», а в XPath достаточно `//input[@name='q']/ancestor::form`.
- **Нет поиска по текстовому содержимому** — селектор `button:contains('Отправить')` не валиден в нативном CSS; в XPath это `//button[text()='Отправить']`.
- **Нет осей `following-sibling::` / `preceding-sibling::` с произвольным смещением** — только соседние элементы через `+` и `~`.

```html
<!-- Задача: найти <label>, связанный с <input id='pwd'> -->
<!-- XPath (просто): //input[@id='pwd']/preceding-sibling::label -->
<!-- CSS (обходной путь): ищем по атрибуту for или используем :has() в новых браузерах -->
<div class="field">
    <label for="pwd">Пароль</label>
    <input type="password" id="pwd" name="password">
</div>

<!-- Решение на CSS: поиск по атрибуту for -->
<!-- CSS: label[for='pwd'] -->
<!-- Альтернатива с :has() (Selenium 4.2+, современные браузеры): -->
<!-- CSS: div:has(> input#pwd) > label -->
```

## Best Practices для выбора стратегии локаторов

- **Приоритет стабильных атрибутов**: `#unique-id` > `[data-testid='...']` > `[name='...']` > `.class`. Избегайте зависимостей от порядка элементов или динамических классов.
- **Комбинируйте селекторы для точности**: `form.login input[type='submit']` точнее, чем просто `.submit`, и не требует избыточной вложенности.
- **Используйте `:not()` для исключения состояний**: `button:not([disabled])` надёжнее, чем фильтрация результатов в коде теста.
- **Документируйте сложные локаторы**: если селектор требует пояснения (например, обход ограничения «нет родителей»), добавьте комментарий в коде теста или в самом HTML через `data-*` атрибуты.
- **Тестируйте локаторы в DevTools**: перед внедрением проверьте `$$('ваш_селектор')` в консоли браузера — это покажет количество и точность совпадений.

---

## 2. Базовый синтаксис и селекторы

> Фундаментальные токены для построения локаторов. 
> Эти базовые конструкции покрывают 80% задач автоматизации и служат основой для более сложных выражений.

- **Селектор по тегу** — поиск всех элементов указанного типа; полезен для массовых операций, но требует уточнения контекстом.
- **Селектор по классу (`.class`)** — обращение к элементам по CSS-классу; помните, что у элемента может быть несколько классов.
- **Селектор по ID (`#id`)** — самый быстрый и точный способ найти уникальный элемент; используйте, когда ID стабилен.
- **Атрибутные селекторы** — гибкая фильтрация по значениям атрибутов: точное совпадение, префикс, суффикс, содержание слова или подстроки.
- **Комбинаторы вложенности** — пробел (` `) для любого уровня вложенности, `>` для прямых потомков; позволяют строить точные цепочки без осей.
- **Экранирование спецсимволов** — классы с двоеточиями, точками или другими спецсимволами требуют экранирования обратным слэшем.

В CSS можно использовать и двойные, и одинарные кавычки — разницы нет. Это дело вкуса и стиля.

---

## Селекторы по тегу, классу и ID

```html
<!-- Поиск по тегу: все <button> на странице -->
<!-- CSS: button -->
<!-- XPath: //button -->
<div class="toolbar">
    <button>Новый</button>
    <button>Сохранить</button>
    <button disabled>Удалить</button>
</div>

<!-- Поиск по классу: элементы с классом 'btn-primary' -->
<!-- CSS: .btn-primary -->
<!-- XPath: //*[@class='btn-primary'] (хрупко!) или //*[contains(concat(' ', @class, ' '), ' btn-primary ')] -->
<button class="btn btn-primary">Отправить</button>
<span class="btn-primary icon">✓</span>

<!-- Поиск по нескольким классам: элемент должен иметь ОБА класса -->
<!-- CSS: .btn.btn-primary — элемент с классами 'btn' И 'btn-primary' -->
<!-- XPath: //*[contains(@class, 'btn') and contains(@class, 'btn-primary')] -->
<button class="btn btn-primary btn-lg">Большая кнопка</button>

<!-- Поиск по ID: уникальный элемент -->
<!-- CSS: #submit-btn -->
<!-- XPath: //*[@id='submit-btn'] -->
<button id="submit-btn" type="submit">Готово</button>

<!-- Комбинация: тег + класс -->
<!-- CSS: input.error — <input> с классом 'error' -->
<!-- XPath: //input[contains(@class, 'error')] -->
<input type="text" class="error" name="username">
```

## Атрибутные селекторы: полный справочник

```html
<!-- [attr] — наличие атрибута (любое значение) -->
<!-- CSS: input[required] -->
<!-- XPath: //input[@required] -->
<input type="text" name="email" required>

<!-- [attr=value] — точное совпадение значения -->
<!-- CSS: input[type='password'] -->
<!-- XPath: //input[@type='password'] -->
<input type="password" name="pwd">

<!-- [attr~=word] — атрибут содержит слово (разделённое пробелами), полезно для классов -->
<!-- CSS: [class~='btn'] — элемент имеет класс 'btn' среди прочих -->
<!-- XPath: //*[contains(concat(' ', @class, ' '), ' btn ')] -->
<button class="btn btn-primary disabled">Кнопка</button>

<!-- [attr^=start] — значение начинается с подстроки -->
<!-- CSS: [id^='user_'] — id начинается с 'user_' -->
<!-- XPath: //*[starts-with(@id, 'user_')] -->
<div id="user_123_profile">Профиль</div>
<div id="user_456_settings">Настройки</div>

<!-- [attr$=end] — значение заканчивается подстрокой -->
<!-- CSS: [href$='.pdf'] — ссылка на PDF -->
<!-- CSS: [src$='.png'] — изображение PNG -->
<!-- XPath: //*[substring(@href, string-length(@href)-3) = '.pdf'] -->
<a href="/docs/manual.pdf">Инструкция</a>
<img src="/img/logo.png" alt="Logo">

<!-- [attr*=substring] — значение содержит подстроку в любом месте -->
<!-- CSS: [class*='error'] — класс содержит 'error' (например, 'field-error', 'error-msg') -->
<!-- XPath: //*[contains(@class, 'error')] -->
<span class="field-error">Неверный формат</span>
<div class="error-msg highlight">Ошибка сервера</div>

<!-- [attr|='value'] — значение равно 'value' или начинается с 'value-' (для языковых кодов) -->
<!-- CSS: [lang|='en'] — lang='en' или lang='en-US' -->
<!-- XPath: //*[starts-with(@lang, 'en')] -->
<html lang="en-US">
```

## Комбинаторы вложенности: пробел и `>`

```html
<!-- Пробел ( ) — любой уровень вложенности (descendant) -->
<!-- CSS: form input — любой <input> внутри <form>, на любой глубине -->
<!-- XPath: //form//input -->
<form id="login">
    <div class="field">
        <input name="username"> <!-- Найден: form input -->
        <div class="nested">
            <input name="token"> <!-- Также найден: form input -->
        </div>
    </div>
</form>

<!-- > — только прямые потомки (child) -->
<!-- CSS: form > input — только <input>, которые являются прямыми детьми <form> -->
<!-- XPath: //form/input -->
<form id="register">
    <input name="email"> <!-- Найден: form > input -->
    <div>
        <input name="confirm"> <!-- Не найден: не прямой потомок -->
    </div>
</form>

<!-- Комбинация: точный путь к элементу -->
<!-- CSS: div.container > ul > li.active — прямой потомок с классом -->
<!-- XPath: //div[@class='container']/ul/li[contains(@class, 'active')] -->
<div class="container">
    <ul>
        <li>Обычный элемент</li>
        <li class="active">Активный шаг</li> <!-- Найден -->
    </ul>
</div>

<!-- Разница между пробелом и > на практике -->
<!-- CSS: .modal button — любая кнопка внутри модального окна -->
<!-- CSS: .modal > button — только кнопка, лежащая напрямую в .modal -->
<!-- XPath: //div[@class='modal']//button -->
<!-- XPath: //div[@class='modal']/button -->
<div class="modal">
    <button class="close">✕</button> <!-- Найден обоими селекторами -->
    <div class="content">
        <button>Действие</button> <!-- Найден только .modal button -->
    </div>
</div>
```

## Экранирование спецсимволов в селекторах

```html
<!-- Классы с двоеточием: требуют экранирования обратным слэшем -->
<!-- CSS: .w\:text-center — экранируем двоеточие -->
<!-- XPath: //*[@class='w:text-center'] -->
<div class="w:text-center">Центрированный текст</div>

<!-- Классы с точкой: точка внутри имени класса тоже экранируется -->
<!-- CSS: .version\.2\.0 -->
<!-- XPath: //*[@class='version.2.0'] -->
<span class="version.2.0">Версия 2.0</span>

<!-- Классы с пробелами: пробел разделяет классы, поэтому 'my class' — это два класса -->
<!-- CSS: .my.class — элемент с классами 'my' И 'class' -->
<!-- Для класса с буквальным пробелом (редко) используйте экранирование: .my\ class -->
<div class="my class">Два отдельных класса</div>

<!-- ID с точкой или двоеточием: тоже требуют экранирования -->
<!-- CSS: #user\:123 -->
<!-- XPath: //*[@id='user:123'] -->
<div id="user:123">Профиль пользователя</div>
```

Практический совет: если экранирование становится сложным, используйте атрибутные селекторы

Квадратные скобки `[]` делают селектор атрибутным

```html
<!-- CSS: [class='w:text-center'] вместо .w\:text-center -->
<!-- CSS: [id='user:123'] вместо #user\:123 -->
```

## Сводная таблица базовых селекторов

| Тип селектора         | Синтаксис CSS         | Пример             | Эквивалент XPath                                      |
|:----------------------|-----------------------|--------------------|-------------------------------------------------------|
| По тегу               | `tag`                 | `input`            | `//input`                                             |
| По классу             | `.class`              | `.btn-primary`     | `//*[contains(@class, 'btn-primary')]`                |
| По ID                 | `#id`                 | `#submit`          | `//*[@id='submit']`                                   |
| По атрибуту (наличие) | `[attr]`              | `[required]`       | `//*[@required]`                                      |
| По атрибуту (точное)  | `[attr=val]`          | `[type='text']`    | `//*[@type='text']`                                   |
| По слову в атрибуте   | `[attr~=word]`        | `[class~='btn']`   | `//*[contains(concat(' ',@class,' '),' btn ')]`       |
| По префиксу           | `[attr^=start]`       | `[id^='user_']`    | `//*[starts-with(@id,'user_')]`                       |
| По суффиксу           | `[attr$=end]`         | `[href$='.pdf']`   | `//*[substring(@href,string-length(@href)-3)='.pdf']` |
| По подстроке          | `[attr*=sub]`         | `[class*='error']` | `//*[contains(@class,'error')]`                       |
| Прямой потомок        | `parent > child`      | `form > input`     | `//form/input`                                        |
| Любой потомок         | `ancestor descendant` | `form input`       | `//form//input`                                       |

---

## Best Practices для базовых селекторов

- **Приоритет уникальности**: `#id` > `[data-testid]` > `[name]` > `.class` — чем уникальнее атрибут, тем стабильнее локатор.
- **Избегайте хрупких классов**: классы вида `btn-primary disabled active` могут меняться; предпочитайте `data-*` атрибуты для тестов.
- **Используйте `~=` для классов**: `[class~='btn']` надёжнее, чем `[class*='btn']`, так как не совпадёт с `submit-btn`.
- **Тестируйте в консоли**: перед внедрением проверьте селектор через `$$('ваш_селектор')` в DevTools — это покажет точное количество совпадений.
- **Документируйте сложные случаи**: если селектор требует экранирования или неочевидной комбинации, добавьте комментарий в коде теста.

```java
// Пример использования в Selenium
WebElement emailInput = driver.findElement(By.cssSelector("input[name='email']:not([disabled])"));

// Пример в Selenide: цепочка селекторов
$(By.cssSelector("form#login")).$(By.cssSelector("input[name='password']")).setValue("secret");

// Использование data-testid для стабильности
$(By.cssSelector("[data-testid='submit-button']")).click();
```

# 3. Комбинаторы и навигация по DOM

> Как строить цепочки поиска без осей. 
> 
> В отличие от XPath, CSS не поддерживает прямую навигацию «вверх» или «назад», но компенсирует это чёткими комбинаторами, 
> которые работают быстрее за счёт нативной оптимизации браузерных движков.

## Комбинаторы вложенности: пробел и `>`

- ` ` (пробел) — descendant combinator. Находит элемент на любом уровне вложенности внутри родительского. Аналог `//` в XPath.
- `>` — child combinator. Находит только непосредственных дочерних элементов. Аналог `/` в XPath.

В CSS между комбинаторами пробелы можно ставить или не ставить — на работу селектора это не влияет:

```css
form>input       /* без пробелов — ок */
form > input     /* с пробелами — ок */
form> input      /* пробел только справа — ок */
form >input      /* пробел только слева — ок */
```

```html
<!-- Поиск по любому уровню вложенности (descendant) -->
<!-- CSS: form input -->
<!-- XPath: //form//input -->
<form id="login">
    <div class="field">
        <input name="username"> <!-- Найден: form input -->
        <div class="nested">
            <input name="token"> <!-- Также найден: form input -->
        </div>
    </div>
</form>

<!-- Поиск только прямых потомков (child) -->
<!-- CSS: form > input -->
<!-- XPath: //form/input -->
<form id="register">
    <input name="email"> <!-- Найден: form > input -->
    <div>
        <input name="confirm"> <!-- Не найден: вложен в div, а не в form -->
    </div>
</form>
```

## Комбинаторы соседей: `+` и `~`

- `+` (adjacent sibling) — выбирает первый элемент, идущий сразу после указанного. Аналог `following-sibling::*[1]` в XPath.
- `~` (general sibling) — выбирает все последующие элементы одного уровня вложенности. Аналог `following-sibling::*` в XPath.

```html
<!-- Точный следующий сосед (adjacent sibling) -->
<!-- CSS: label[for='pwd'] + input -->
<!-- XPath: //label[@for='pwd']/following-sibling::input[1] -->
<div class="field-row">
    <label for="pwd">Пароль:</label>
    <input type="password" id="pwd"> <!-- Найден -->
    <span class="hint">Подсказка</span> <!-- Не найден -->
</div>

<!-- Любой последующий сосед (general sibling) -->
<!-- CSS: label[for='email'] ~ * -->
<!-- XPath: //label[@for='email']/following-sibling::* -->
<div class="field-row">
    <label for="email">Email:</label>
    <input type="email" id="email"> <!-- Найден -->
    <span class="error">Неверный формат</span> <!-- Найден -->
    <button type="submit">Отправить</button> <!-- Найден -->
</div>

<!-- Практический кейс: поиск поля по лейблу -->
<!-- CSS: .form-group > label[for='fname'] + input -->
<div class="form-group">
    <label for="fname">Имя</label>
    <input id="fname" name="first_name"> <!-- Найден -->
    <div class="extra">Лишний элемент</div> <!-- Не найден -->
</div>
```

## Компенсация отсутствия оси `parent::`

В CSS нет прямой навигации «вверх» по DOM. Для решения этой задачи используются три стратегии:
- Поиск «сверху вниз» через известные родительские контейнеры.
- Использование псевдо-класса `:has()` (современные браузеры, Selenium 4.2+).
- Компенсация в коде автотеста через методы `.parent()` / `.closest()` (Selenide):
  
  ```java
  // Находим кнопку, затем поднимаемся к ближайшему предку с классом "card"
  SelenideElement card = $("button.delete-btn").closest(".card");

  // Или через parent() — только на один уровень вверх
  SelenideElement parentDiv = $("button.delete-btn").parent();
  ```

```html
<!-- Задача: найти контейнер <div class="card"> по кнопке внутри него -->
<!-- XPath (просто): //button[text()='Удалить']/ancestor::div[@class='card'] -->
<!-- CSS (через :has()): div.card:has(> button.delete-btn) -->
<div class="card">
    <h3>Товар А</h3>
    <button class="delete-btn">Удалить</button> <!-- Контекст -->
</div>

<!-- Задача: найти родительскую форму по полю ввода -->
<!-- XPath: //input[@name='q']/ancestor::form -->
<!-- CSS (обходной путь): form:has(> input[name='q']) -->
<form action="/search">
    <div class="search-box">
        <input name="q"> <!-- Контекст -->
    </div>
</form>

<!-- Задача: найти соседний элемент в другом контейнере -->
<!-- CSS не позволяет прыгнуть на уровень вверх, поэтому ищем от общего предка -->
<!-- CSS: .container > .panel:first-child + .panel -->
<!-- XPath: //div[@class='container']/div[@class='panel'][1]/following-sibling::div[@class='panel'] -->
<div class="container">
    <div class="panel left">...</div>
    <div class="panel right">Цель поиска</div>
</div>
```

## Сравнение комбинаторов CSS и XPath

| Тип навигации        | CSS синтаксис | XPath аналог                      | Поведение                             |
|:---------------------|---------------|-----------------------------------|---------------------------------------|
| Любой потомок        | `A B`         | `A//B`                            | Рекурсивный поиск на всех уровнях     |
| Прямой потомок       | `A > B`       | `A/B`                             | Только один уровень вложенности       |
| Следующий сосед      | `A + B`       | `A/following-sibling::B[1]`       | Первый элемент того же уровня         |
| Все следующие соседи | `A ~ B`       | `A/following-sibling::B`          | Все элементы того же уровня после `A` |
| Родитель             | ❌ Нет         | `B/parent::A` или `B/ancestor::A` | Требуется `:has()` или обход в коде   |

## Best Practices

- **Избегайте `> ` без необходимости**: селектор `.container button` работает быстрее и устойчивее к изменениям вёрстки, чем `.container > div > button`, если структура может меняться.
- **Используйте `+` для связанных элементов**: идеален для пар `label` + `input`, `input` + `span.error`, `button` + `tooltip`.
- **Проверяйте `~` на точность**: поиск всех соседей может вернуть много элементов; уточняйте селектором класса или атрибута, например `label ~ span.error`.
- **Компенсируйте `parent::` на уровне фреймворка**: в Selenide используйте `$(By.cssSelector("input#pwd")).parent()`, а не пытайтесь эмулировать это сложным CSS, если браузер не поддерживает `:has()`.
- **Тестируйте комбинаторы в DevTools**: введите `$$('A + B')` в консоли, чтобы убедиться, что выбран только целевой элемент, а не случайный сосед.

```java
// Пример использования комбинаторов в Selenium
WebElement submitBtn = driver.findElement(By.cssSelector("form#checkout > button[type='submit']"));

// Пример в Selenide: навигация от соседа к родителю
SelenideElement passwordField = $("label[for='pwd']").sibling(1); // input рядом
SelenideElement formContainer = passwordField.closest("form"); // компенсация parent::

// Использование :has() для поиска родителя (Selenium 4.2+, Chrome 105+)
By cardSelector = By.cssSelector("div.card:has(> button.delete-btn)");
WebElement card = driver.findElement(cardSelector);
```

## 4. Псевдо-классы и функции для автоматизации

> Встроенные возможности для фильтрации состояния и позиции. 
> 
> Псевдо-классы позволяют выбирать элементы на основе их положения в DOM, статуса взаимодействия или валидации, что особенно полезно при тестировании динамических интерфейсов и форм.

## Структурные псевдо-классы

Позволяют выбирать элементы по их порядковому номеру среди братьев (siblings) или исключать определённые узлы.

```html
<!-- :first-child — первый дочерний элемент своего родителя -->
<!-- CSS: ul.steps li:first-child -->
<!-- XPath: //ul[@class='steps']/li[1] -->
<ul class="steps">
    <li>Шаг 1</li> <!-- Найден -->
    <li>Шаг 2</li>
    <li>Шаг 3</li>
</ul>

<!-- :last-child — последний дочерний элемент своего родителя -->
<!-- CSS: ul.steps li:last-child -->
<!-- XPath: //ul[@class='steps']/li[last()] -->
<ul class="steps">
    <li>Шаг 1</li>
    <li>Шаг 2</li>
    <li>Шаг 3</li> <!-- Найден -->
</ul>

<!-- :nth-child(n) — элемент на определённой позиции (1-based индекс) -->
<!-- CSS: table tbody tr:nth-child(3) -->
<!-- XPath: //table/tbody/tr[3] -->
<table>
    <tr><td>Строка 1</td></tr>
    <tr><td>Строка 2</td></tr>
    <tr><td>Строка 3</td></tr> <!-- Найден -->
</table>

<!-- :not() — отрицание, исключает элементы, подходящие под селектор -->
<!-- CSS: input:not([disabled]) -->
<!-- XPath: //input[not(@disabled)] -->
<form>
    <input type="text" name="login"> <!-- Найден -->
    <input type="text" name="promo" disabled> <!-- Исключён -->
    <input type="email" name="email"> <!-- Найден -->
</form>

<!-- Комбинация: исключение по классу или атрибуту -->
<!-- CSS: button.btn:not(.btn-outline) -->
<!-- XPath: //button[contains(@class,'btn') and not(contains(@class,'btn-outline'))] -->
<button class="btn btn-primary">Основная</button> <!-- Найден -->
<button class="btn btn-outline">Контурная</button> <!-- Исключён -->
```

## Псевдо-классы состояний форм

Фильтруют элементы ввода в зависимости от их текущего статуса: выбран, заблокирован, прошёл валидацию и т.д.

```html
<!-- :checked — выбранные чекбоксы или радиокнопки -->
<!-- CSS: input[type='checkbox']:checked -->
<!-- XPath: //input[@type='checkbox' and @checked] -->
<label>
    <input type="checkbox" name="agree" checked> Согласен с условиями
</label>
<label>
    <input type="checkbox" name="newsletter"> Подписка <!-- Не найден -->
</label>

<!-- :disabled / :enabled — состояние доступности поля -->
<!-- CSS: input:disabled -->
<!-- XPath: //input[@disabled] -->
<input type="text" name="readonly_field" disabled value="Только для чтения">
<input type="text" name="editable_field" value="Можно редактировать"> <!-- Не найден -->

<!-- :focus — элемент в фокусе (полезно для UI-тестов, но требует явного фокуса в Selenium) -->
<!-- CSS: input:focus -->
<!-- XPath: не имеет прямого аналога, требует JS-скрипта -->
<input type="text" id="search" placeholder="Введите запрос...">

<!-- :valid / :invalid — результат встроенной валидации HTML5 -->
<!-- CSS: input:invalid -->
<!-- XPath: нет нативного аналога, проверка через JS -->
<input type="email" name="user_email" required> <!-- Станет :invalid, если пусто или неверно -->
<input type="email" name="user_email" value="test@example.com"> <!-- Станет :valid -->
```

## Интерактивные псевдо-классы и ограничения в Selenium

```html
<!-- :hover / :active / :visited — псевдо-классы, зависящие от состояния мыши или истории браузера -->
<!-- CSS: a:hover -->
<!-- Selenium/Selenide: НЕ поддерживаются напрямую через findElement, так как браузер не хранит состояние hover в DOM -->
<a href="/profile" class="nav-link">Профиль</a>

<!-- Как проверить стили hover/active в автотестах -->
<!-- 1. Симулировать наведение через Actions API (Selenium) -->
<!-- 2. Использовать JavaScript для получения computed styles -->
<!-- Пример: проверка изменения цвета при hover -->
<style>
    .btn:hover { background-color: #0056b3; }
</style>
<button class="btn">Наведи на меня</button>
```

## :has() — эмуляция «родительского» и условного поиска

Появился в CSS Level 4 и поддерживается современными браузерами (Chrome 105+, Firefox 121+, Safari 15.4). Selenium 4.2+ нативно поддерживает `:has()`.

```html
<!-- :has() позволяет выбрать элемент, содержащий определённых потомков -->
<!-- CSS: div.card:has(> img.featured) -->
<!-- XPath: //div[@class='card' and .//img[contains(@class,'featured')]] -->
<div class="card">
    <img class="featured" src="hero.jpg">
    <h2>Избранный товар</h2> <!-- Родитель найден -->
</div>
<div class="card">
    <img src="thumb.jpg">
    <h2>Обычный товар</h2> <!-- Не найден -->
</div>

<!-- :has() с селекторами соседей: поиск родителя по наличию соседа -->
<!-- CSS: label:has(+ input:invalid) -->
<!-- XPath: //label[following-sibling::input[1][@invalid]] (условно) -->
<form>
    <label>Пароль:</label>
    <input type="password" required> <!-- Если invalid, label будет найден -->
    <span class="error">Обязательное поле</span>
</form>

<!-- Ограничения :has() -->
- Работает только в современных браузерах; в старых версиях Selenium Grid может вызвать `InvalidSelectorException`.
- Не поддерживает псевдо-элементы (`::before`, `::after`) внутри `:has()`.
- Требует строгой проверки совместимости перед использованием в CI/CD.
```

## Почему :contains() не работает в CSS (и чем заменить)

В XPath существует функция `text()` и `contains()`, позволяющая искать элементы по содержимому: `//button[contains(text(), 'Отправить')]`. В нативном CSS нет эквивалента `:contains()` по соображениям производительности и безопасности (сложность парсинга текстовых узлов).

```html
<!-- Задача: найти кнопку по тексту "Отправить" -->
<!-- XPath: //button[text()='Отправить'] или //button[contains(., 'Отправить')] -->
<!-- CSS: НЕ ВОЗМОЖНО напрямую -->
<button class="action-btn">Отправить заявку</button>

<!-- Альтернативы в автотестах -->
<!-- 1. Использовать data-* атрибуты для машинного поиска -->
<!-- CSS: button[data-action='submit'] -->
<button class="action-btn" data-action="submit">Отправить заявку</button>

<!-- 2. Искать по родительскому контексту + CSS, а затем фильтровать текст в коде теста -->
<!-- Selenide: $$("button.action-btn").filter(Condition.text("Отправить")) -->

<!-- 3. XPath остаётся лучшим выбором для поиска по тексту -->
<!-- WebDriver: driver.findElement(By.xpath("//button[contains(text(), 'Отправить')]")) -->
```

## Сводная таблица псевдо-классов

| Псевдокласс CSS      | Когда срабатывает                            | CSS пример                | XPath                                                    |
|:---------------------|:---------------------------------------------|:--------------------------|:---------------------------------------------------------|
| `:hover`             | Курсор мыши наведён на элемент               | `button:hover`            | - (нет аналога, только JS)                               |
| `:focus`             | Элемент в фокусе (готов принимать ввод)      | `input:focus`             | - (нет аналога, только JS)                               |
| `:active`            | Элемент активирован (зажат клик мыши)        | `button:active`           | -                                                        |
| `:visited`           | Ссылка уже была посещена в истории браузера  | `a:visited`               | -                                                        |
| `:link`              | Ссылка, которую ещё не посещали              | `a:link`                  | -                                                        |
| `:enabled`           | Элемент доступен для редактирования/кликов   | `input:enabled`           | `//input[not(@disabled)]`                                |
| `:disabled`          | Элемент заблокирован (атрибут `disabled`)    | `input:disabled`          | `//input[@disabled]`                                     |
| `:read-only`         | Поле только для чтения (`readonly`)          | `input:read-only`         | `//input[@readonly]`                                     |
| `:read-write`        | Поле доступно для редактирования             | `input:read-write`        | `//input[not(@readonly or @disabled)]`                   |
| `:checked`           | Радио-кнопка или чекбокс отмечены            | `input:checked`           | `//input[@checked]`                                      |
| `:indeterminate`     | Чекбокс в неопределённом состоянии           | `input:indeterminate`     | -                                                        |
| `:required`          | Поле обязательно для заполнения (`required`) | `input:required`          | `//input[@required]`                                     |
| `:optional`          | Поле необязательное (нет `required`)         | `input:optional`          | `//input[not(@required)]`                                |
| `:valid`             | Значение прошло HTML5-валидацию              | `input:valid`             | - (нет аналога, JS)                                      |
| `:invalid`           | Значение НЕ прошло HTML5-валидацию           | `input:invalid`           | - (нет аналога, JS)                                      |
| `:in-range`          | Число в `input[type=number]` внутри min/max  | `input:in-range`          | -                                                        |
| `:out-of-range`      | Число вне диапазона min/max                  | `input:out-of-range`      | -                                                        |
| `:placeholder-shown` | Плейсхолдер виден (поле пустое)              | `input:placeholder-shown` | `//input[@placeholder and not(text())]` (приблизительно) |
| `:empty`             | Элемент не содержит текста и дочерних узлов  | `div:empty`               | `//div[not(node())]`                                     |
| `:not(sel)`          | Логическое НЕ (исключение)                   | `input:not([disabled])`   | `//input[not(@disabled)]`                                |
| `:has(sel)`          | Содержит внутри указанный элемент            | `div:has(> img)`          | `//div[.//img]`                                          |
| `:first-child`       | Первый дочерний элемент родителя             | `ul li:first-child`       | `//ul/li[1]`                                             |
| `:last-child`        | Последний дочерний элемент родителя          | `ul li:last-child`        | `//ul/li[last()]`                                        |
| `:nth-child(n)`      | Элемент на позиции n (1-based)               | `tr:nth-child(2)`         | `//tr[2]`                                                |
| `:nth-of-type(n)`    | n-ный элемент своего типа                    | `p:nth-of-type(2)`        | `//p[2]`                                                 |
| `:contains()`        | Поиск по тексту                              | Не поддерживается в CSS   | `//button[contains(., 'text')]`                          |

## Best Practices для псевдо-классов

- **Избегайте `:nth-child()` в динамических списках**: если порядок элементов меняется или добавляется скрытая разметка, позиция сместится. Лучше использовать `:nth-of-type()` или уникальные `data-*` атрибуты.
- **`:not()` упрощает фильтрацию**: вместо поиска `input` и последующей проверки `!element.isEnabled()` в коде, сразу используйте `input:not([disabled])`.
- **Проверяйте поддержку `:has()`**: если ваш тест ранится в разных браузерах, добавьте fallback на XPath или JS-поиск для стабильности.
- **Не полагайтесь на `:hover`/`:active` в локаторах**: они не отражаются в DOM. Для проверки состояний используйте `Actions` в Selenium или получайте `computedStyle` через JS.
- **Комбинируйте с атрибутами**: `input[name='email']:valid:enabled` даёт точный и быстрый селектор без лишней вложенности.

```java
// Пример: фильтрация активных элементов через :not()
List<WebElement> activeInputs = driver.findElements(By.cssSelector("input:not([disabled]):not([readonly])"));

// Пример: Selenide + :has() для поиска карточки с активным чекбоксом
// Требуется Selenium 4.2+ и современный браузер
By cardSelector = By.cssSelector("div.product-card:has(input:checkbox:checked)");
SelenideElement selectedCard = $(cardSelector);

// Пример: обход отсутствия :contains() через Selenide Conditions
SelenideElement submitBtn = $$("button.action").filter(Condition.text("Отправить")).first();

// Пример: проверка валидности поля через CSS-состояние и JS-фоллбек
boolean isValid = driver.findElement(By.cssSelector("input#email:valid")).isDisplayed();
```

# 5. Типовые шаблоны локаторов для Selenium/Selenide

> Готовые паттерны под частые задачи автоматизации. 
> 
> Эти конструкции охватывают 90% сценариев поиска элементов в веб-приложениях и обеспечивают баланс между точностью, 
> скоростью выполнения и устойчивостью к изменениям вёрстки.

## Поиск по уникальным идентификаторам и data-атрибутам

- **Уникальный `id`** — самый быстрый и стабильный способ. Используйте, когда `id` генерируется предсказуемо или назначается разработчиками явно.
- **`data-testid` / `data-qa`** — рекомендуемый паттерн для автотестов. Отделяет тестовую логику от визуальных классов и бизнес-логики, снижая риск поломки тестов при рефакторинге UI.

```html
<!-- Поиск по стабильному ID -->
<!-- CSS: #submit-btn -->
<!-- XPath: //*[@id='submit-btn'] -->
<button id="submit-btn" type="submit">Отправить</button>

<!-- Поиск по data-testid (рекомендуется для CI/CD) -->
<!-- CSS: [data-testid='user-profile-card'] -->
<!-- XPath: //*[@data-testid='user-profile-card'] -->
<div class="card" data-testid="user-profile-card">
    <h2>Профиль пользователя</h2>
</div>

<!-- Комбинация тега и data-атрибута для точности -->
<!-- CSS: input[data-testid='login-email'] -->
<!-- XPath: //input[@data-testid='login-email'] -->
<input type="email" data-testid="login-email" placeholder="Email">
```

## Частичное совпадение атрибутов

- **Префикс `[attr^=val]`** — полезен для динамических ID или классов, начинающихся с известного шаблона.
- **Суффикс `[attr$=val]`** — подходит для расширений файлов или фиксированных окончаний строк.
- **Подстрока `[attr*=val]`** — универсальный поиск, но требует осторожности из-за риска ложных срабатываний.

```html
<!-- Поиск по префиксу: динамические ID -->
<!-- CSS: [id^='modal_'] -->
<!-- XPath: //*[starts-with(@id, 'modal_')] -->
<div id="modal_8472_confirm">Подтверждение действия</div>

<!-- Поиск по суффиксу: типы файлов или версии -->
<!-- CSS: a[href$='.pdf'] -->
<!-- XPath: //*[substring(@href, string-length(@href)-3) = '.pdf'] -->
<a href="/docs/annual_report_2023.pdf">Скачать отчёт</a>

<!-- Поиск по подстроке: классы с динамическими состояниями -->
<!-- CSS: [class*='error-state'] -->
<!-- XPath: //*[contains(@class, 'error-state')] -->
<span class="input-field error-state highlight">Неверный формат</span>

<!-- Ограничение: [class*='error'] может найти 'no-error' -->
<!-- Используйте [class~='error'] для точного совпадения по слову -->
```

## Логические комбинации и исключения

- **Сочетание условий** — объединяет несколько атрибутов в одном селекторе без пробелов.
- **Исключение через `:not()`** — убирает элементы с определённым состоянием прямо на уровне движка браузера.

```html
<!-- Точное сочетание атрибутов -->
<!-- CSS: input[type='text'][name='username'][required] -->
<!-- XPath: //input[@type='text'][@name='username'][@required] -->
<input type="text" name="username" required>

<!-- Исключение заблокированных полей -->
<!-- CSS: button.btn:not([disabled]):not(.invisible) -->
<!-- XPath: //button[contains(@class,'btn') and not(@disabled) and not(contains(@class,'invisible'))] -->
<button class="btn" disabled>Неактивная</button>
<button class="btn invisible">Скрытая</button>
<button class="btn">Активная кнопка</button> <!-- Найден -->

<!-- Фильтрация по видимому состоянию (если элемент имеет CSS-класс скрытия) -->
<!-- CSS: .panel:not(.collapsed) -->
<div class="panel collapsed">...</div>
<div class="panel">Раскрытый контент</div> <!-- Найден -->
```

## Навигация от известного соседа

- **`+` (adjacent sibling)** — ищет элемент, идущий сразу после указанного. Идеален для пар `label` + `input`.
- **`~` (general sibling)** — ищет все последующие элементы на том же уровне. Полезен для поиска сообщений об ошибках после поля ввода.

```html
<!-- Поиск поля ввода по тексту лейбла -->
<!-- CSS: label[for='pwd'] + input -->
<!-- XPath: //label[@for='pwd']/following-sibling::input[1] -->
<div class="form-row">
    <label for="pwd">Пароль</label>
    <input type="password" id="pwd"> <!-- Найден -->
    <span class="hint">Минимум 8 символов</span>
</div>

<!-- Поиск сообщения об ошибке после поля -->
<!-- CSS: input[name='email'] + span.error -->
<!-- XPath: //input[@name='email']/following-sibling::span[contains(@class,'error')] -->
<input type="email" name="email">
<span class="error">Неверный email</span> <!-- Найден -->

<!-- Поиск всех последующих элементов-соседей -->
<!-- CSS: .step.active ~ .step -->
<!-- XPath: //div[contains(@class,'step') and contains(@class,'active')]/following-sibling::div[contains(@class,'step')] -->
<div class="step active">Шаг 1</div>
<div class="step">Шаг 2</div> <!-- Найден -->
<div class="step">Шаг 3</div> <!-- Найден -->
```

## Специфика Selenide: компенсация ограничений CSS

В CSS нет прямой навигации «вверх» (`parent::`, `ancestor::`). Selenide решает эту задачу через цепочки методов, 
которые работают с найденным элементом как с контекстом, сохраняя читаемость тестов.

- `.parent()` — возвращает непосредственного родителя в DOM.
- `.closest("selector")` — ищет ближайшего предка, соответствующего селектору.
- `.sibling(index)` / `.preceding(index)` — навигация к соседним элементам по индексу.

```html
<!-- HTML-контекст для демонстрации навигации -->
<div class="card-container">
    <div class="card" id="item_1">
        <h3>Товар 1</h3>
        <button class="delete">Удалить</button>
    </div>
    <div class="card" id="item_2">
        <h3>Товар 2</h3>
        <button class="delete">Удалить</button>
    </div>
</div>
```

```java
// Selenide: поиск родительской карточки по кнопке удаления
// Аналог XPath: //button[@class='delete']/ancestor::div[@class='card']
SelenideElement card = $(".card button.delete").closest(".card");
card.shouldBe(visible); // Проверяем, что нашли нужную карточку

// Selenide: переход к родителю
// Аналог XPath: //input[@id='pwd']/..
SelenideElement formGroup = $("#pwd").parent();

// Selenide: поиск соседа по индексу
// Аналог XPath: //label[for='pwd']/following-sibling::input
SelenideElement input = $("label[for='pwd']").sibling(1);
input.setValue("my_secret");

// Selenide: цепочка от родителя к потомку
// Находим форму, затем ищем внутри неё кнопку
SelenideElement submitBtn = $("form#checkout").$("button[type='submit']");
submitBtn.click();
```

## Сравнение подходов: чистый CSS vs Selenide-цепочки

| Задача                  | Чистый CSS        | Selenide API                     | Примечание                           |
|:------------------------|-------------------|----------------------------------|--------------------------------------|
| Найти родителя          | Не поддерживается | `.parent()` / `.closest()`       | CSS требует `:has()` (Selenium 4.2+) |
| Найти соседа            | `A + B` / `A ~ B` | `.sibling(n)`                    | CSS быстрее, если сосед фиксирован   |
| Найти по тексту         | Не поддерживается | `.filter(Condition.text("..."))` | CSS требует `data-*` атрибутов       |
| Поиск внутри найденного | `parent child`    | `.$("child")`                    | Selenide сохраняет контекст поиска   |

## Best Practices

- **Всегда используйте `data-testid`**, если есть возможность договориться с разработчиками. Это устраняет зависимость от изменений CSS-классов и структуры вёрстки.
- **Избегайте `[class*='...']` без проверки контекста**: селектор может найти неожиданные элементы, если подстрока встречается в других словах класса.
- **Комбинируйте атрибуты для уникальности**: `[name='email'][type='text']` надёжнее, чем просто `[name='email']`, если имена могут дублироваться в разных формах.
- **Не полагайтесь на порядок `:nth-child` в динамических списках**: если элемент может быть скрыт или добавлен программно, позиция сместится. Лучше использовать `:nth-of-type()` или уникальные идентификаторы.
- **Используйте Selenide-цепочки для компенсации `parent::`**: это читаемее и поддерживаемее, чем попытки эмулировать восходящий поиск через сложный CSS или XPath.
- **Проверяйте селекторы в DevTools перед коммитом**: команда `$$('ваш_селектор')` в консоли мгновенно покажет количество найденных элементов и предотвратит `ElementNotFoundException` в CI.

```java
// Пример интеграции паттернов в Page Object
public class LoginPage {
    
    private static final String LOGIN_FORM = "form#login";
    private static final String EMAIL_INPUT = "input[data-testid='email-field']:not([disabled])";
    private static final String SUBMIT_BTN = "button[type='submit']:not(.loading)";

    public void login(String email, String password) {
        $(LOGIN_FORM).shouldBe(visible);
        $(EMAIL_INPUT).setValue(email);
        $(LOGIN_FORM).$("input[data-testid='password-field']").setValue(password);
        $(SUBMIT_BTN).click();
    }
}
```

---

## 6. Оптимизация: производительность и стабильность

> Как писать локаторы, которые не ломаются и работают быстро. 
> 
> Фокус на выборе атрибутов, длине выражений и балансе между точностью и устойчивостью к изменениям UI.

## Приоритет атрибутов: от стабильных к хрупким

- `#id` — абсолютный лидер по скорости и уникальности. Браузеры оптимизируют поиск по ID на уровне движка.
- `[data-testid]` / `[data-qa]` — семантически нейтральные маркеры для тестов. Не влияют на верстку и не меняются при рефакторинге стилей.
- `[name]`, `[type]`, `[action]` — стандартные атрибуты форм. Обычно стабильны, но могут дублироваться в разных секциях страницы.
- `[class]` — подвержены изменениям при обновлении дизайн-систем или использовании CSS-модулей. Используйте только как дополнительный фильтр.
- `:nth-child()` / порядковые индексы — самый хрупкий вариант. Ломается при добавлении, удалении или скрытии элементов в DOM.

```html
<!-- Идеальный: уникальный ID -->
<!-- CSS: #submit-form -->
<button id="submit-form" type="button">Отправить</button>

<!-- Рекомендуемый: data-testid -->
<!-- CSS: [data-testid='product-card-42'] -->
<div class="card" data-testid="product-card-42">Товар #42</div>

<!-- Допустимый: стандартные атрибуты формы -->
<!-- CSS: input[name='email'][type='text'] -->
<input name="email" type="text" placeholder="Введите email">

<!-- Хрупкий: зависимость от классов -->
<!-- CSS: .btn.btn-primary.btn-lg -->
<!-- Риск: классы могут измениться при обновлении дизайн-системы -->
<button class="btn btn-primary btn-lg">Купить</button>

<!-- Критически хрупкий: порядковый индекс -->
<!-- CSS: .user-list > div:nth-child(3) -->
<!-- Риск: добавление одного элемента выше сместит весь индекс -->
<div class="user-list">
    <div>Юзер 1</div>
    <div>Юзер 2</div>
    <div>Юзер 3</div> <!-- Цель поиска -->
</div>
```

## Длина селектора и влияние на `querySelectorAll`

- Движок браузера парсит CSS-селекторы справа налево. Чем правее часть селектора уникальна, тем быстрее отработает поиск.
- Избыточная вложенность (например, `div.wrapper > section.main > article > ul > li > a`) замедляет поиск и делает локатор чувствительным к структуре.
- Оптимальная длина: 2-4 токена. Обычно достаточно тега + уникального атрибута или класса.

```html
<!-- Медленно и избыточно -->
<!-- CSS: body > div.container > section.content > form.login > input#username -->
<!-- Улучшение: оставить только уникальную часть -->
<!-- CSS: #username или form.login > #username -->
<div class="container">
    <section class="content">
        <form class="login">
            <input type="text" id="username" name="login">
        </form>
    </section>
</div>

<!-- Быстро и точно -->
<!-- CSS: .product-grid [data-id='123'] -->
<!-- Правая часть ([data-id]) уникальна, левая ограничивает контекст поиска -->
<div class="product-grid">
    <div class="item" data-id="123">Товар A</div>
</div>
```

## Баланс специфичности: избегаем ложных срабатываний

- **Слишком общий селектор** (например, `.btn`) вернет десятки элементов, требуя дополнительной фильтрации в коде теста.
- **Слишком специфичный селектор** (например, `#app > main > div:nth-child(2) > form > button.submit`) сломается при малейшем изменении DOM.
- **Золотая середина**: контекст + уникальный маркер. `form#checkout button[type='submit']`

```html
<!-- Пример ложного совпадения -->
<!-- CSS: .error -->
<!-- Результат: найдет все элементы с классом 'error' на странице (заголовок, иконка, текст, поле) -->
<h2 class="error-title">Ошибка загрузки</h2>
<span class="error-icon">⚠</span>
<input class="error-field">
<p class="error-text">Сервер не отвечает</p>

<!-- Исправление: конкретизация контекста и типа -->
<!-- CSS: input.error-field -->
<!-- Или: .form-group input[class~='error'] -->
<div class="form-group">
    <input class="error-field" name="email"> <!-- Найден точно -->
</div>
```

## Паттерны для динамических интерфейсов

- **Shadow DOM**: стандартный `querySelector` не проникает в тень. Используйте `shadowRoot.querySelector()` или специальные команды Selenide/Selenium.
- **Динамические классы/ID**: часто встречаются в React/Vue (например, `css-1abc2def`). Игнорируйте их, опирайтесь на `data-*` или структуру.
- **Виртуализация списков**: элементы рендерятся только при скролле. `:nth-child()` может не сработать, если элемент не в DOM. Используйте ленивую загрузку или поиск по видимым атрибутам.

```html
<!-- Динамические CSS-модули (React/CSS Modules) -->
<!-- Класс: _container_1a2b3_c4d5e -->
<!-- НЕ ИСПОЛЬЗУЙТЕ в локаторах! -->
<div class="_container_1a2b3_c4d5e">...</div>

<!-- Стабильная альтернатива -->
<!-- CSS: [data-role='dashboard-widget'] -->
<div class="_container_1a2b3_c4d5e" data-role="dashboard-widget">
    <h2>Статистика</h2>
</div>
```

## Сравнение производительности селекторов

| Тип селектора         | Скорость (относительная) | Стабильность          | Рекомендация                       |
|:----------------------|--------------------------|-----------------------|------------------------------------|
| `#id`                 | Максимальная             | Высокая               | Использовать всегда, если доступен |
| `[data-testid]`       | Высокая                  | Высокая               | Стандарт для автотестов            |
| `tag[attr]`           | Средняя                  | Высокая               | Отличный баланс для форм/ссылок    |
| `.class`              | Средняя                  | ️Средняя              | Допустимо в комбинациях            |
| `ancestor descendant` | Медленная                | ️Зависит от структуры | Только при необходимости контекста |
| `:nth-child(n)`       | Средняя                  | Низкая                | Избегать в динамических списках    |
| Динамические классы   | Быстрая                  | Критически низкая     | Никогда не использовать            |

## Best Practices оптимизации

- **Правило «двух токенов»**: стремитесь к локаторам вида `tag[unique-attr]` или `parent > child[unique-attr]`. Избегайте цепочек длиннее 3-4 элементов.
- **Проверка уникальности в консоли**: перед добавлением в тест выполните `$$('ваш_селектор').length`. Если > 1, уточните селектор. Если == 0, проверьте тени/фреймы.
- **Отказ от порядковых индексов**: вместо `.list-item:nth-child(3)` используйте `.list-item[data-index='3']` или поиск по тексту/контенту.
- **Изоляция поиска**: ограничьте контекст ближайшим стабильным контейнером (`modal:has(> form) input`), чтобы не искать по всему DOM.
- **Документирование решений**: если пришлось использовать сложный или неочевидный селектор из-за ограничений фронтенда, добавьте комментарий `// TODO: refactor locator after UI update` или используйте Page Object паттерн с централизованным управлением локаторами.

```java
// Пример: оптимизированный поиск в Page Object
// Плохо: драйвер ищет по всей странице, зависит от верстки
By badLocator = By.cssSelector("div.app-main > section.content > form > input:nth-child(2)");

// Хорошо: точный, быстрый, устойчивый к изменениям
By goodLocator = By.cssSelector("form#checkout input[name='billing_address']");

// Пример проверки уникальности в коде (Selenide)
$$("button.submit").shouldHave(CollectionCondition.size(1));
// Если размер != 1, тест упадет с понятной ошибкой до попытки клика
```