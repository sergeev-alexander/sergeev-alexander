**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Selenide

## Содержание

1. [Введение, настройка и конфигурация](#1-введение-настройка-и-конфигурация)
2. [Core API: поиск и действия с элементами](#2-core-api-поиск-и-действия-с-элементами)
3. [Ожидания, проверки и коллекции](#3-ожидания-проверки-и-коллекции)
4. [Формы, контексты и браузерные данные](#4-формы-контексты-и-браузерные-данные)
5. [Архитектура тестов: Page Object и кастомизация](#5-архитектура-тестов-page-object-и-кастомизация)
6. [Расширенные сценарии и паттерны автоматизации](#6-расширенные-сценарии-и-паттерны-автоматизации)
7. [Отладка, диагностика и отчётность](#7-отладка-диагностика-и-отчётность)
8. [Интеграция с API, тестовые данные и безопасность](#8-интеграция-с-api-тестовые-данные-и-безопасность)
9. [Инфраструктура, CI/CD и кросс-браузерное тестирование](#9-инфраструктура-cicd-и-кросс-браузерное-тестирование)
10. [Оптимизация, поддержка и жизненный цикл проекта](#10-оптимизация-поддержка-и-жизненный-цикл-проекта)
11. [Псевдоклассы CSS](#8-Псевдоклассы-CSS)

---

# 1. Введение, настройка и конфигурация

> Философия Selenide и базовая настройка окружения для автоматизации UI-тестов: подключение, интеграция с фреймворками, 
> управление конфигурацией.

## Философия Selenide

- **Простой API** — интуитивные методы для работы с элементами, минимизация boilerplate
- **Автоматические ожидания** — встроенный polling-механизм, не требует явных `Thread.sleep()` или `WebDriverWait`
- **Архитектура над WebDriver** — абстракция, скрывающая сложность нативного Selenium API
- **Fluent-интерфейс** — цепочки вызовов для читаемых и поддерживаемых тестов
- **Стабильность по умолчанию** — защита от `StaleElementReferenceException`, автоматические повторные попытки

---

## Подключение зависимостей

### Maven (pom.xml):

```xml
<dependency>
    <groupId>com.codeborne</groupId>
    <artifactId>selenide</artifactId>
    <version>7.2.3</version>
    <scope>test</scope>
</dependency>
```

### Gradle (build.gradle):

```groovy
dependencies {
    testImplementation 'com.codeborne:selenide:7.2.3'
}
```

---

## Интеграция с тест-фреймворками

### JUnit 5 (Extensions)

```java
class LoginTest {

    @RegisterExtension
    static ScreenShooterExtension screenshotEmAll = new ScreenShooterExtension()
            .to("build/reports/tests")
            .forFailedTestsOnly();

    @RegisterExtension
    static TextReportExtension report = new TextReportExtension();

    @Test
    void validLogin() {
        open("/login");
        $("#username").setValue("user");
        $("#password").setValue("pass");
        $("button[type=submit]").click();
        $(".welcome").should(appear);
    }
}
```

### TestNG

```java
public class AuthTest {

    @BeforeMethod
    void setUp() {
        Configuration.baseUrl = "https://app.example.com";
        open("/");
    }

    @Test(groups = {"smoke", "auth"})
    void successfulLogin() {
        $("#username").setValue("testuser");
        $("#password").setValue("secret");
        $("button.submit").click();
        $(".dashboard-header").should(appear);
    }

    @AfterMethod
    void tearDown() {
        Selenide.closeWebDriver();
    }
}
```

### JUnit 4

```java
public class LegacyTest {

    @Rule
    public ScreenShooter screenShooter = ScreenShooter
            .to("build/reports")
            .forFailedTestsOnly();

    @Test
    void legacyScenario() {
        open("/page");
        $("button").click();
    }
}
```

---

## Справочник конфигурации

### Основные параметры

| Параметр               | Описание                                       | Пример / Значение по умолчанию                |
|:-----------------------|------------------------------------------------|-----------------------------------------------|
| `browser`              | Имя браузера                                   | `"chrome"`, `"firefox"`, `"edge"`, `"safari"` |
| `baseUrl`              | Базовый URL приложения                         | `"https://staging.example.com"`               |
| `timeout`              | Глобальный таймаут ожидания элементов (мс)     | `4000`                                        |
| `headless`             | Запуск без графического интерфейса             | `true` / `false`                              |
| `remote`               | URL Selenium Grid / Selenoid                   | `"http://grid:4444/wd/hub"`                   |
| `proxyEnabled`         | Включение proxy для перехвата запросов         | `false`                                       |
| `fastSetValue`         | Ускоренный ввод через JS (без эмуляции клавиш) | `false`                                       |
| `driverManagerEnabled` | Автозагрузка драйверов через WebDriverManager  | `true`                                        |

### CI-специфичные флаги

- `-Dselenide.headless=true` — headless-режим для CI
- `-Dselenide.browser=chrome` — явное указание браузера
- `-Dselenide.remote=http://selenoid:4444/wd/hub` — удалённый запуск
- `-Dselenide.reportsFolder=build/reports` — кастомная папка отчётов

---

## Источники конфигурации и приоритеты

Приоритет применяется сверху вниз (наивысший → низший):

1. **Код** — прямое присваивание `Configuration.browser = "firefox";`
2. **System properties** — параметры, переданные JVM при запуске через `-Dключ=значение` - `-Dselenide.browser=chrome`
3. **selenide.properties** — файл в `src/test/resources`
4. **Environment variables** — переменные операционной системы, в которой запущена JVM - `set SELENIDE_BROWSER=edge`

### Пример selenide.properties

```properties
browser=chrome
baseUrl=https://staging.example.com
timeout=10000
headless=true
reportsFolder=build/reports/tests
screenshot=true
savePageSource=true
```

### Чтение конфигурации в коде

```java
import static com.codeborne.selenide.Configuration.*; 
// Selenide сам парсит selenide.properties при инициализации класса Configuration (при первом обращении к любому статическому полю)

public class ConfigHelper {
    
    // Загружаются автоматически, без ручного использования класса java.util.Properties
    public static void printActiveConfig() {
        System.out.println("Browser: " + browser);
        System.out.println("Base URL: " + baseUrl);
        System.out.println("Timeout: " + timeout + "ms");
        System.out.println("Headless: " + headless);
    }
}
```

> Поля загружаются один раз при первом обращении к классу Configuration. 
> Поэтому `ConfigHelper.printActiveConfig()` покажет именно те значения, которые были вычислены с учетом файла, 
> переменных окружения и системных свойств на момент запуска JVM.

---

## Динамическое изменение конфигурации

```java
class DynamicConfigTest {

    @Test
    void testWithCustomTimeout() {
        // Сохраняем оригинальное значение
        long originalTimeout = timeout;
        
        // Меняем конфигурацию для конкретного сценария
        timeout = 30000;
        open("/slow-loading-page");
        $(".heavy-component").should(appear);
        
        // Восстанавливаем конфигурацию
        timeout = originalTimeout;
    }

    @Test
    void isolatedBrowserConfig() {
        // Для изолированных тестов можно запускать отдельный драйвер
        WebDriver customDriver = new ChromeDriver();
        Selenide.open("about:blank", customDriver);
        // ... тестовая логика ...
        customDriver.quit();
    }
}
```

> ⚠️ Изменение `Configuration` влияет на все последующие тесты в том же потоке. 
> 
> Для параллельного выполнения используйте `ThreadLocal<Configuration>` или изолируйте тесты по браузерным экземплярам.

---

## Best Practices

- Выносите конфигурацию в `selenide.properties` для согласованности между разработчиками и окружениями
- Используйте системные свойства для CI/CD: `-Dselenide.headless=true -Dselenide.browser=chrome`
- Не меняйте `Configuration` в середине теста без явной необходимости — это влияет на все последующие операции
- Для параллельных тестов в JUnit 5/TestNG используйте изоляцию через `@RegisterExtension` или `ThreadLocal`-обёртки
- Документируйте кастомные флаги конфигурации в README проекта
- Используйте `Configuration.assertionMode = SOFT` для накопления ошибок в отчётах, если это соответствует стратегии тестирования
- Включайте `savePageSource = true` для просмотра и анализа падающих тестов
- Жёсткое кодирование `Configuration` в каждом тесте — нарушает централизованное управление
- Не используйте `Thread.sleep()` вместо встроенных ожиданий Selenide
- Не изменяйте глобальную конфигурацию без восстановления — приводит к "утечке" состояния между тестами
- Игнорирование `headless`-режима в CI — увеличивает время прогона и потребление ресурсов
- Не храните чувствительные данные (пароли, токены) в `selenide.properties` — используйте ENV или secrets-менеджеры

---

# 2. Core API: поиск и действия с элементами

> Core API Selenide предоставляет лаконичный Fluent-интерфейс для поиска, взаимодействия и проверки веб-элементов, 
> скрывая сложность нативного Selenium WebDriver и обеспечивая встроенные автоматические ожидания.
> 
> - Цепочки вызовов — последовательное применение методов без промежуточных переменных
> - Lazy evaluation — элементы не ищутся до момента вызова действия или проверки

## Поиск элементов и селекторы

### Глобальный поиск

- `$(String cssSelector)` / `$$(String cssSelector)` — поиск одного элемента / коллекции по CSS
- `$(By)` / `$$(By)` — поиск через нативный `Selenium By`
- `$x(String xpath)` / `$$x(String xpath)` — поиск по XPath
- `$(WebElement)` / `$(SelenideElement)` — обёртка над существующим элементом
- `$$(Collection<WebElement>)` — обёртка над коллекцией `WebElement`

### Сокращённые CSS-алиасы

- `$("#id")` — поиск по ID
- `$(".class")` — поиск по классу
- `$("[name='value']")` — поиск по атрибуту
- `$("tag")` — поиск по тегу

### Пользовательские селекторы (`Selectors`)

- `byText(String)` — точное совпадение текста
- `withText(String)` — частичное совпадение текста
- `byName(String)` / `byValue(String)` — по атрибутам `name` / `value`
- `byAttribute(String attr, String value)` — по произвольному атрибуту
- `byTitle(String)` — по атрибуту `title`

## Навигация по DOM

### Внутренний поиск (от текущего `SelenideElement`)

- `$(selector)` / `find(selector)` — поиск дочернего элемента (синонимы)
- `$$(selector)` / `findAll(selector)` — поиск коллекции дочерних элементов (синонимы)
- `$x(xpath)` / `$$x(xpath)` — поиск по XPath внутри элемента

### Вверх по дереву

- `parent()` — непосредственный родитель
- `ancestor(cssSelector)` / `closest(cssSelector)` — ближайший предок по CSS (синонимы)
- `closest(By)` — ближайший предок через `Selenium By`
- `closest(String tagName)` — ближайший предок по тегу

### Соседние элементы

- `sibling(cssSelector)` — сосед на том же уровне по CSS
- `preceding(int index)` — предыдущий сосед (`1` — ближайший)
- `following(int index)` — следующий сосед (`1` — ближайший)

## Работа с коллекциями (`ElementsCollection`)

### Фильтрация и поиск внутри коллекции

- `filterBy(Condition)` / `filterBy(Predicate<SelenideElement>)` — оставить элементы, удовлетворяющие условию
- `excludeWith(Condition)` / `exclude(Predicate<SelenideElement>)` — исключить элементы по условию
- `findBy(Condition)` — найти первый подходящий элемент

### Доступ по индексу и состояние

- `get(int index)` — элемент по индексу (с `0`)
- `first()` / `last()` — первый / последний элемент
- `size()` / `isEmpty()` / `areDisplayed()` — размер, проверка на пустоту, проверка видимости всех элементов
- `asDynamicIterable()` / `asFixedIterable()` — ленивая итерация или снапшот коллекции

### Проверки коллекций (`should...`)

- `shouldHaveSize(int)` / `shouldBe(CollectionCondition)` / `shouldHave(CollectionCondition)` — проверка условий коллекции

### Условия коллекций (`CollectionCondition`)

- `size(int)` / `sizeGreaterThan` / `sizeGreaterThanOrEqual` / `sizeLessThan` / `sizeLessThanOrEqual` / `sizeNotEqual` — проверка размера
- `empty` — коллекция пуста
- `texts(String...)` / `textsInAnyOrder(String...)` / `exactTexts(String...)` — проверка текстов элементов
- `itemWithText(String)` — наличие элемента с текстом
- `containExactTextsCaseSensitive(String...)` — регистрозависимая проверка

### Итерация и преобразование

- `forEach(Consumer<SelenideElement>)` — действие для каждого элемента
- `iterator()` / `stream()` — обход коллекции (`stream()` отключает авто-ожидания Selenide)

### Действия коллекции (делегирование первому элементу)

- `click()` / `setValue(String)` — клик или ввод в первый элемент
- `texts()` / `attributes(String)` — получение списков текстов или атрибутов всех элементов

## Извлечение данных

### Текст и HTML

- `getText()` — возвращает ВЕСЬ текст внутри элемента, включая текст всех вложенных дочерних элементов.
- `getOwnText()` — возвращает только СВОЙ СОБСТВЕННЫЙ текст, игнорируя текст внутри детей.
- `innerText()` / `innerHtml()` / `outerHtml()` — получение через JS (`innerText`) и нативный HTML

### Атрибуты, стили и свойства

- `getAttribute(String name)` — значение произвольного (любого) атрибута
- `getCssValue(String property)` — значение CSS-свойства
- `getValue()` / `val()` — текущее значение поля (`value`)
- `getTagName()` / `getSelectedOption()` / `isSelected()` — тег, выбранная опция `<select>`, состояние выбора

### Геометрия и позиция

- `getLocation()` / `getSize()` / `getRect()` — координаты, размеры или объединённый объект `Rectangle`

## Действия с элементами

### Клики и контекстное меню

- `click()` / `doubleClick()` / `contextClick()` — левый, двойной, правый клик
- `click(int x, int y)` / `hover(int x, int y)` — клик или наведение по относительным координатам
- `*(ClickOptions options)` — все клики поддерживают модификаторы (`ctrl`, `shift`), оффсеты и тип устройства

### Клавиатура и ввод

- `pressEnter()` / `pressEscape()` / `pressTab()` — специальные клавиши
- `press(String keys)` / `press(Keys keys)` — произвольная клавиша или `Keys` enum
- `sendKeys(CharSequence... keys)` — низкоуровневая отправка символов

### Работа с полями ввода

- `setValue(String text)` / `val(String)` — установка значения (заменяет старое)
- `setValue(String, boolean clearBefore)` — установка с опциональной очисткой
- `append(String text)` — добавление текста к текущему значению
- `clear()` — очистка поля

### Чекбоксы, радиокнопки и `<select>`

- `setSelected(boolean)` — выбор/снятие флажка или радиокнопки
- `selectOption(String value)` / `selectOption(int index)` — выбор опции по значению или индексу
- `selectOptionByValue(String)` / `selectOptionContainingText(String)` — выбор опции по атрибуту или тексту

### Перетаскивание и скроллинг

- `dragAndDropTo(String/WebElement/SelenideElement)` — перетаскивание к цели
- `dragAndDrop(int x, int y)` — перетаскивание на смещение в пикселях
- `scrollTo()` / `scrollIntoView(...)` — скроллинг к элементу (с выравниванием `true/false` или `ScrollIntoViewOptions`)

### Загрузка файлов

- `uploadFile(File... / Path...)` — загрузка локальных файлов
- `uploadFromClasspath(String...)` — загрузка из `resources`

### Наведение мыши

- `hover()` / `hover(int x, int y)` — наведение курсора на элемент или его часть

## Проверки и ожидания

### Условия и проверки

- `should(Condition...)` / `shouldBe(Condition...)` / `shouldHave(Condition...)` — синонимы, проверка с авто-ожиданием
- `waitUntil(Condition, long timeoutMs)` / `waitWhile(Condition, long timeoutMs)` — явное ожидание условия с таймаутом

### JavaScript и низкоуровневое взаимодействие

- `executeJavaScript(String jsCode, Object... args)` — выполнение JS в контексте элемента
- `click(usingDefaultSelenium())` — клик через нативный WebDriver (без JS-обёртки Selenide)

### Скриншоты
- `screenshot()` — сохранение скриншота конкретного элемента

```java
// ПОИСК ЭЛЕМЕНТОВ И СЕЛЕКТОРЫ

// --- Глобальный поиск ---

// Поиск одного элемента по CSS-селектору
SelenideElement loginBtn = $("#login-button");

// Поиск коллекции элементов по XPath с фильтрацией
ElementsCollection items = $$x("//div[@class='product']").filterBy(visible);

// --- Сокращённые CSS-алиасы ---

// Поиск по ID (аналог $(By.id("header")))
SelenideElement header = $("#header");

// Поиск по классу с вложенным поиском
$(".card").find(".title").shouldHave(text("Заголовок"));

// --- Пользовательские селекторы (Selectors) ---

// Поиск кнопки с точным текстом "Отправить"
$(byText("Отправить")).click();

// Поиск input по атрибуту name с частичным совпадением текста
$("[name='search']").shouldBe(appear);
```

```java
// НАВИГАЦИЯ ПО DOM

// --- Внутренний поиск (от текущего SelenideElement) ---

// Поиск дочернего элемента внутри найденного родителя
$(".user-card").find(".avatar").shouldBe(visible);

// Поиск коллекции параграфов внутри статьи по XPath
$$x(".//p", $(".article")).shouldHaveSize(3);

// --- Вверх по дереву ---

// Получение родителя и проверка его класса
$("#child-element").parent().shouldHave(cssClass("container"));

// Поиск ближайшего предка с классом "modal"
$(".close-btn").closest(".modal").shouldBe(visible);

// --- Соседние элементы ---

// Клик по следующему соседнему элементу после текущего
$(".active-item").following(1).click();

// Проверка текста предыдущего элемента в списке
$(".selected").preceding(1).shouldHave(text("Предидущий"));
```
```java
// РАБОТА С КОЛЛЕКЦИЯМИ (ElementsCollection)

// --- Фильтрация и поиск внутри коллекции ---

// Фильтрация видимых элементов и выбор первого
$$(".product").filterBy(visible).first().click();

// Исключение элементов с классом "disabled" и поиск по тексту
$$("li").excludeWith(cssClass("disabled")).findBy(text("Активный")).shouldBe(enabled);

// --- Доступ по индексу и состояние ---

// Получение третьего элемента (индекс с 0) и проверка
$$("tr").get(2).shouldHave(text("Данные"));

// Проверка, что коллекция не пуста и все элементы отображаются
$$(".notification").shouldNotBe(empty).and(areDisplayed);

// --- Проверки коллекций (should...) ---

// Ожидание, что в списке появится ровно 5 элементов
$$("ul > li").shouldHaveSize(5);

// Проверка коллекции на соответствие условию
$$(".price").shouldBe(textsInAnyOrder("100", "200", "300"));

// --- Условия коллекций (CollectionCondition) ---

// Проверка точных текстов элементов в указанном порядке
$$("td").shouldHave(exactTexts("Имя", "Возраст", "Город"));

// Проверка, что хотя бы один элемент содержит указанный текст
$$("option").shouldHave(itemWithText("Выберите значение"));

// --- Итерация и преобразование ---

// Выполнение действия для каждого видимого элемента
$$(".checkbox").filterBy(visible).forEach(el -> el.setSelected(true));

// Получение списка текстов всех элементов коллекции (без ожиданий при итерации)
List<String> titles = $$("h2").texts();

// --- Действия коллекции (делегирование первому элементу) ---

// Клик по первому элементу коллекции и ввод текста
$$("input[type='text']").setValue("Первое значение").click();

// Получение всех значений атрибута href из коллекции ссылок
List<String> links = $$("a").attributes("href");
```

```java
// ИЗВЛЕЧЕНИЕ ДАННЫХ

// --- Текст и HTML ---

// Получение видимого текста и текста без учёта дочерних элементов
String fullText = $(".article").getText();
String ownText = $(".article").getOwnText();

// Получение внутреннего HTML для парсинга
String htmlContent = $(".template").innerHtml();

// --- Атрибуты, стили и свойства ---

// Чтение атрибута data-id и CSS-свойства цвета
String itemId = $(".product").getAttribute("data-id");
String color = $(".button").getCssValue("background-color");

// Получение значения input и проверка выбранной опции в select
String email = $("input[name='email']").getValue();
String selected = $("select").getSelectedOption().getText();

// --- Геометрия и позиция ---

// Получение данных о геометрии элемента
Point location = $(".modal").getLocation(); // координаты X, Y
Dimension size = $(".modal").getSize(); // ширина, высота

// Получение полного прямоугольника элемента (позиция + размеры)
Rectangle rect = $(".header").getRect();
```

```java
// ДЕЙСТВИЯ С ЭЛЕМЕНТАМИ

// --- Клики и контекстное меню ---

// Обычный клик и клик с модификатором клавиши
$("#submit").click();
$(".item").click(ClickOptions.usingMethod(JS).withCtrl());

// Вызов контекстного меню (правый клик)
$(".table-row").contextClick();

// --- Клавиатура и ввод ---

// Последовательное нажатие специальных клавиш
$("input").pressEnter().pressEscape();

// Отправка комбинации клавиш (выделить всё + удалить)
$("textarea").press(Keys.chord(Keys.CONTROL, "a")).press(Keys.DELETE);

// --- Работа с полями ввода ---

// Установка значения с предварительной очисткой и последующим добавлением текста
$("#search").setValue("запрос", true).append(" уточнение");

// Очистка поля и проверка, что оно пустое
$("#comment").clear();
String current = $("#comment").val(); // ""

// --- Чекбоксы, радиокнопки и <select> ---

// Выбор чекбокса и радиокнопки
$("#agree").setSelected(true);
$("input[value='premium']").setSelected(true);

// Выбор опции в выпадающем списке по тексту и по индексу
$("select#country").selectOptionContainingText("Россия");
$("select#role").selectOption(2);

// --- Перетаскивание и скроллинг ---

// Перетаскивание элемента к цели по селектору
$(".draggable").dragAndDropTo(".drop-zone");

// Плавный скроллинг к элементу с выравниванием по центру
$(".footer-link").scrollIntoView(ScrollIntoViewOptions.withBehavior("smooth").withBlock("center"));

// --- Загрузка файлов ---

// Загрузка файла из classpath и локального пути
$("input[type='file']").uploadFromClasspath("test.pdf");
$("input[type='file']").uploadFile(new File("/path/to/doc.docx"));

// --- Наведение мыши ---

// Наведение для отображения тултипа и клик по появившемуся элементу
$(".menu-item").hover();
$(".submenu").shouldBe(appear).click();
```

```java
// ✅ ПРОВЕРКИ И ОЖИДАНИЯ

// --- Условия и проверки ---

// Цепочка проверок с авто-ожиданием (до 4 секунд по умолчанию)
$("#status").shouldHave(text("Успешно"), cssClass("green"), be(visible));

// Явное ожидание с кастомным таймаутом
$(".loader").waitWhile(visible, 10000); // Ждать исчезновения лоадера до 10 сек

// --- JavaScript и низкоуровневое взаимодействие ---

// Выполнение произвольного JS-кода над элементом
$("#canvas").executeJavaScript("arguments[0].getContext('2d').clearRect(0,0,200,200)", $("#canvas"));

// Клик через нативный Selenium (в обход JS-обёртки Selenide)
$(".overlay").click(usingDefaultSelenium());

// --- Скриншоты ---

// Сохранение скриншота конкретного элемента при ошибке или для отчёта
if ($(".error-banner").isDisplayed()) {
    $(".error-banner").screenshot();
}
```

## Best Practices

- Используйте CSS-селекторы вместо XPath, когда это возможно — они быстрее и стабильнее
- Опирайтесь на `data-*` атрибуты (например, `data-testid`) для изоляции тестов от изменений вёрстки
- Избегайте жестких привязок к индексам в коллекциях — используйте фильтрацию по тексту или атрибутам
- Не используйте `Thread.sleep()` — Selenide автоматически ожидает появления и готовности элементов
- Для ввода текста в большие формы используйте `fastSetValue = true` в конфигурации, но учитывайте, что это не эмулирует реальные события `oninput`/`onchange`
- При работе с динамическими коллекциями всегда вызывайте `$$()` заново, а не кешируйте `ElementsCollection` надолго
- Не полагайтесь на абсолютные XPath (`/html/body/div[2]/...`) — они ломаются при малейшем изменении DOM
- Не используйте `getValue()` для получения текста неинпутовых элементов — применяйте `getText()` или `getOwnText()`
- Не забывайте про `hover()` перед взаимодействием с элементами, которые появляются только при наведении
- Избегайте смешивания нативного `WebElement` и `SelenideElement` без явной необходимости — это нарушает единую модель ожиданий

---









# 11. Псевдоклассы CSS

| Псевдокласс CSS | Когда срабатывает | Проверка в Selenide |
|:----------------|:------------------|:--------------------|
| `:hover` | Курсор мыши наведён на элемент | `hover()` (эмуляция) |
| `:focus` | Элемент в фокусе (готов принимать ввод) | `$(el).shouldBe(focused)` |
| `:active` | Элемент активирован (зажат клик мыши) | - |
| `:visited` | Ссылка уже была посещена в истории браузера | - |
| `:link` | Ссылка, которую ещё не посещали | - |
| `:enabled` | Элемент доступен для редактирования/кликов | `$(el).shouldBe(enabled)` |
| `:disabled` | Элемент заблокирован (атрибут `disabled`) | `$(el).shouldBe(disabled)` |
| `:read-only` | Поле только для чтения (`readonly`) | `$(el).shouldHave(attribute("readonly"))` |
| `:read-write` | Поле доступно для редактирования | - |
| `:checked` | Радио-кнопка или чекбокс отмечены | `$(el).shouldBe(checked)` |
| `:indeterminate` | Чекбокс в неопределённом состоянии | `$(el).executeJavaScript("return this.indeterminate")` |
| `:required` | Поле обязательно для заполнения (`required`) | `$(el).shouldHave(attribute("required"))` |
| `:optional` | Поле необязательное (нет `required`) | - |
| `:valid` | Значение прошло HTML5-валидацию | `$(el).is(":valid")` |
| `:invalid` | Значение НЕ прошло HTML5-валидацию | `$(el).is(":invalid")` |
| `:in-range` | Число в `input[type=number]` внутри min/max | - |
| `:out-of-range` | Число вне диапазона min/max | - |
| `:placeholder-shown` | Плейсхолдер виден (поле пустое) | - |
| `:empty` | Элемент не содержит текста и дочерних узлов | `$(el).shouldBe(empty)` |
| `:not(selector)` | Логическое НЕ (исключение) | `$(el).is(":not(.class)")` |
| `:has(selector)` | Содержит внутри указанный элемент | `$(el).find(selector).exists()` |
| `:first-child` | Первый дочерний элемент родителя | - |
| `:last-child` | Последний дочерний элемент родителя | - |
| `:nth-child(n)` | Элемент на позиции n (1-based) | - |
| `:nth-of-type(n)` | n-ный элемент своего типа | - |