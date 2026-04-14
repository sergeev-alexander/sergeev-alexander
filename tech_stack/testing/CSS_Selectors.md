# CSS Selectors

## Содержание

1. [Введение](#1-введение)
2. [Базовый синтаксис и селекторы](#2-базовый-синтаксис-и-селекторы)
3. [Комбинаторы и навигация по DOM](#3-комбинаторы-и-навигация-по-dom)
4. [Псевдо-классы и функции для автоматизации](#4-псевдо-классы-и-функции-для-автоматизации)
5. [Типовые шаблоны локаторов для Selenium/Selenide](#5-типовые-шаблоны-локаторов-для-seleniumselenide)
6. [Оптимизация: производительность и стабильность](#6-оптимизация-производительность-и-стабильность)
7. [Отладка и инструменты валидации](#7-отладка-и-инструменты-валидации)

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