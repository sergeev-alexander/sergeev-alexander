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
11. [Псевдоклассы CSS](#11-Псевдоклассы-CSS)

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

# 3. Ожидания, проверки и коллекции

> Продвинутые паттерны работы с ожиданиями и коллекциями в Selenide: кастомные условия, сложные таймауты, динамические списки и стратегии предотвращения типичных ошибок.

## Философия fluent-ассерций и встроенные условия

- `should()` / `shouldNot()` — базовые методы для проверки произвольных условий
- `shouldBe()` / `shouldNotBe()` — проверка состояний элемента (видимость, кликабельность, выбранность)
- `shouldHave()` / `shouldNotHave()` — проверка свойств (текст, значение атрибута, CSS-свойства)
- Автоматический повторный запрос элемента до выполнения условия или истечения таймаута
- Декларативный стиль: логика проверки читается как естественное предложение

> Методы `should()`, `shouldBe()`, `shouldHave()` и их отрицательные версии возвращают тот же самый объект элемента/коллекции (`SelenideElement` или `ElementsCollection`), 
> что позволяет выстраивать цепочку проверок. 
> При этом сам вызов метода не возвращает булево значение — в случае невыполнения условия тест просто упадет с исключением.
>
> Метод `.and()` является синтаксическим сахаром и не выполняет логическую операцию `AND`, а просто продолжает цепочку вызовов для того же элемента, 
> делая код читаемее. 
> Selenide также поддерживает `.or()` для коллекций при формировании условий совпадения, но для ассерций отдельных элементов в fluent-стиле отдельного `or()`-метода нет (вместо этого пишут кастомное условие с `Condition.or`).

```java
// Проверка видимости и кликабельности
$(".btn-primary").shouldBe(visible).and(clickable);

// Проверка текста и атрибута
$("h1.title").shouldHave(text("Добро пожаловать")).and(attribute("data-loaded", "true"));

// Негативные проверки
$(".error-message").shouldNotBe(visible);
$("#loading-spinner").shouldNot(exist);

// Condition.and — оба условия должны выполниться одновременно
$(".modal").shouldBe(Condition.and("видима и доступна", visible, enabled));

// Condition.or — достаточно выполнения хотя бы одного условия
$(".status").shouldBe(Condition.or("успех или предупреждение",
                      text("Success"),
                      text("Warning")));
```

## Таймауты: глобальные и точечные

- Глобальный таймаут задаётся через `Configuration.timeout` (по умолчанию 4000 мс)
- Точечный таймаут переопределяет глобальный для конкретной проверки через `withTimeout()`
- Интервал опроса настраивается через `Configuration.pollingInterval` (по умолчанию 200 мс)
- Динамическое изменение не влияет на уже начатые ожидания

```java
// Глобальная настройка

// Configuration — это статический класс Selenide (com.codeborne.selenide.Configuration) 
// его поля задают глобальные параметры для всех последующих проверок
Configuration.timeout = 10000;
Configuration.pollingInterval = 500;

// Локальная переопределяющая проверка
$(".heavy-component").withTimeout(Duration.ofSeconds(30)).should(appear);

// Цепочка с разным таймаутом
$("#fast-btn").click();
$("#slow-result").withTimeout(Duration.ofSeconds(15)).shouldHave(text("Готово"));
```

## Кастомные условия

- Реализация интерфейса `Condition` для нестандартных проверок
- Использование лямбда-выражений через `Condition.condition()`
- Переопределение `apply()` и `test()` для валидации сложных состояний DOM
- Интеграция с `should()`, `shouldBe()` без потери fluent-синтаксиса

```java
// Базовая сигнатура с названием и Predicate
public static Condition condition(String name, Predicate<SelenideElement> predicate)

// С переопределённым таймаутом
public static Condition condition(String name, Predicate<SelenideElement> predicate, long timeoutMs)
```

```java
// Лямбда-подход
Condition hasDataAttr = Condition.condition("has-data-attr", el ->
    el.getAttribute("data-custom") != null && !el.getAttribute("data-custom").isEmpty()
);

// В тесте
$(".widget").should(hasDataAttr);
```

```java
// Полная реализация через интерфейс
class ContainsJsonCondition extends Condition {
    
    public ContainsJsonCondition() {
        super("contains-valid-json"); 
    }

    @Override
    public boolean apply(Driver driver, WebElement element) {
        try {
            new org.json.JSONObject(element.getText());
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}

// В тесте
$("#api-response").should(new ContainsJsonCondition());
```

> Когда используется `Condition.condition(name, predicate)`, внутри Selenide сам вызывает `predicate.test(element)`.
>
> А когда класс наследуешься от Condition напрямую, переопределяется абстрактный метод `apply(Driver, WebElement)`.
>
> Это два разных пути создания кастомного условия.

## Работа с коллекциями и динамическими списками

- `$$()` возвращает `ElementsCollection`, поддерживающий ленивую инициализацию
- Проверки размера, порядка элементов, текстового содержимого
- Фильтрация: `filterBy()`, `excludeWith()`, `findBy()`
- Условный доступ: `firstMatch()`, `last`, `get(index)`
- Динамические списки (ленивая подгрузка, AJAX) требуют повторного запроса `$$()` после изменений
- Кеширование `ElementsCollection` опасно при изменении DOM — приводит к `StaleElementReferenceException`

```java
// На странице изначально 3 элемента
$$(".item").shouldHave(size(3));  // Selenide ищет .item → находит 3

// Кликаем кнопку, AJAX подгружает ещё 2 элемента
$("#load-more").click();

// Коллекция НЕ кеширована — она заново ищет .item в DOM → находит 5
$$(".item").shouldHave(size(5));  // работает, без StaleElementException
```

```java
// Проверка размера
$$(".list-item").shouldHave(size(5));

// Проверка содержимого (порядок важен)
$$(".nav-link").shouldHave(exactTexts("Главная", "О нас", "Контакты"));

// Фильтрация по условию
$$(".product-card").filterBy(text("В наличии")).shouldHave(sizeGreaterOrEqual(2));

// Ожидание подгрузки элементов
$$(".paginated-item").shouldHave(size(10));
$("button.load-more").click();
$$(".paginated-item").shouldHave(size(20));
```

## Распространённые ошибки и способы их предотвращения

- `ElementNotFound` при проверке коллекции до её полной загрузки
- `StaleElementReferenceException` при кешировании коллекции и последующем изменении DOM
- Несоответствие `text()` и `exactText()` при наличии скрытых элементов или пробелов
- Игнорирование `pollingInterval`, приводящее к избыточным запросам к DOM

## Best Practices

- Используйте `shouldHave(text())` вместо `getText().equals()` — встроенные проверки автоматически ждут появления текста
- Настраивайте `Configuration.pollingInterval` в диапазоне 100–300 мс для баланса между нагрузкой и скоростью реакции
- Применяйте `withTimeout()` только для известных "медленных" мест, не раздувайте таймауты глобально
- Для динамических коллекций всегда запрашивайте `$$()` заново после AJAX-обновлений или действий пользователя
- Создавайте кастомные `Condition` для повторяющихся бизнес-проверок (например, `isValidDate()`, `isFullyLoaded()`)
- Избегайте `shouldNot(exist)` для скрытых элементов — используйте `shouldNotBe(visible)` или `shouldBe(hidden)`
- Не кешируйте `ElementsCollection` в полях Page Object — это ломает ленивую природу Selenide
- Не используйте `Thread.sleep()` или ручные `WebDriverWait` внутри проверок — это нарушает философию автоматических ожиданий
- Не проверяйте коллекции на `size(0)` для ожидания исчезновения — используйте `shouldBe(empty)` или `shouldHave(size(0))` с явным ожиданием
- Не смешивайте проверки `text()` и `exactText()` без учёта скрытых дочерних элементов и переносов строк

# 4. Формы, контексты и браузерные данные

> Управление контекстами браузера, работа с формами, выполнение JavaScript и манипуляция данными сессии 
> для создания устойчивых и изолированных UI-тестов.

## Стратегии заполнения форм

- **Пошаговое заполнение** — последовательный ввод данных с проверкой валидации после каждого поля
- **Массовый ввод** — использование `setValue()` в цепочках или кастомных методах Page Object
- **Сабмит формы** — отправка через `pressEnter()`, `submit()`, или клик по кнопке отправки
- **Обработка валидации** — проверка сообщений об ошибках через `shouldHave(text(...))` или `cssClass(...)`

```java
// Пошаговое заполнение с валидацией
$("#email").setValue("user@test.com");
$("#password").setValue("securePass123");
$("button.submit").click();

// Массовый ввод через Map
Map<String, String> formData = Map.of(
    "email", "admin@corp.com",
    "password", "adminPass",
    "role", "manager"
);
formData.forEach((key, value) -> $(String.format("[name='%s']", key)).setValue(value));
$("form").submit();
```

## Работа с окнами и вкладками

- **Переключение по индексу/хендлу** — `switchTo().window(index)` или `switchTo().window(handle)`
- **Ожидание появления** — `WebDriverRunner.getWebDriver().getWindowHandles()` с `shouldHave`
- **Закрытие и возврат** — `closeWindow()`, возврат к родительской вкладке через сохранение хендлов

```java
String originalWindow = WebDriverRunner.getWebDriver().getWindowHandle();

// Открытие новой вкладки и переключение
$("a[target='_blank']").click();
switchTo().window(1);
$(".modal-content").shouldBe(visible);

// Закрытие и возврат
closeWindow();
switchTo().window(originalWindow);
$(".main-container").should(appear);
```

## Работа с iframe

- **Переключение** — `switchTo().frame(String/WebElement/SelenideElement)`
- **Вложенные фреймы** — последовательный переход по цепочке
- **Возврат к основному контексту** — `switchTo().defaultContent()`

```java
// Переключение в iframe по селектору
switchTo().frame($("#payment-iframe"));
$("#card-number").setValue("4111 1111 1111 1111");

// Работа с вложенными фреймами
switchTo().frame("#parent-frame");
switchTo().frame("#child-frame");
$(".inner-content").click();

// Возврат в основной контекст
switchTo().defaultContent(); // выход из любого iframe обратно на главную страницу (в основной DOM)
$(".header").should(appear);
```

## Выполнение JavaScript и эмуляция событий

- **Произвольный код** — `Selenide.executeJavaScript(String, Object...)`
- **Возврат значений** — получение результатов вычислений в DOM
- **Передача элементов** — использование `arguments[0]` для работы с `WebElement`
- **Эмуляция событий** — скролл, фокус, программные клики через `dispatchEvent`

```java
// Скролл к элементу
Selenide.executeJavaScript("arguments[0].scrollIntoView({block: 'center'})", $(".footer"));

// Получение значения из localStorage
String token = Selenide.executeJavaScript("return localStorage.getItem('auth_token');");

// Программный клик (в обход стандартного WebDriver)
Selenide.executeJavaScript("arguments[0].click()", $(".hidden-overlay"));

// Эмуляция изменения события input для реактивных фреймворков
String js = "const event = new Event('input', { bubbles: true }); arguments[0].dispatchEvent(event);";
Selenide.executeJavaScript(js, $("#react-input"));
```

## Работа с модальными окнами (Alert/Confirm/Prompt)

- **Alert** — `alert().accept()`, `alert().dismiss()`, `alert().getText()`
- **Confirm/Prompt** — аналогичные методы с передачей текста в `alert().input()`
- **Ожидание появления** — встроенные механизмы Selenide автоматически ждут модального окна

```java
// Accept alert
$("button.show-alert").click();
alert().accept();

// Dismiss confirm и проверка текста
$("button.delete").click();
alert().dismiss();
alert().getText().contains("Вы уверены?");

// Ввод текста в prompt
$("button.prompt").click();
alert().input("Тестовое значение");
alert().accept();
```

> Методы `alert().accept()`, `alert().dismiss()`, `alert().shouldHave()` внутри используют тот же механизм ожидания с `Configuration.timeout`, что и `shouldBe()` для элементов. 
> 
> Selenide будет опрашивать драйвер на наличие алерта с интервалом `pollingInterval`, пока алерт не появится или не истечёт таймаут.

## Cookies и Web Storage

- **Cookies** — `WebDriverRunner.getWebDriver().manage().getCookies()`, установка/удаление через `addCookie()`/`deleteCookie()`
- **localStorage/sessionStorage** — чтение/запись через `Selenide.executeJavaScript()`
- **Изоляция данных** — очистка перед каждым тестом для предотвращения утечки состояния

```java
// Установка кастомной cookie для обхода авторизации
Cookie authCookie = new Cookie("session_id", "abc123xyz", "/", null, true, false); // org.openqa.selenium.Cookie
WebDriverRunner.getWebDriver().manage().addCookie(authCookie);

// Очистка всех cookies и localStorage
WebDriverRunner.getWebDriver().manage().deleteAllCookies();
Selenide.executeJavaScript("sessionStorage.clear(); localStorage.clear();");

// Чтение данных из sessionStorage
String cartData = Selenide.executeJavaScript("return sessionStorage.getItem('cart');");
```

## Сценарии использования: обход авторизации и предустановка состояния

- **Pre-login через cookies** — установка валидной сессии перед открытием URL
- **Мок состояния через localStorage** — инжекция данных для UI без вызова API
- **Очистка контекста** — `clearCookies()`, `clearBrowserLocalStorage()` в `@AfterMethod`

```java
@BeforeMethod
void injectSession() {
    open("about:blank"); // Требуется домен для установки cookies
    WebDriverRunner.getWebDriver().manage().addCookie(
        new Cookie("jwt_token", "mock_valid_token", "/")
    );
    Selenide.executeJavaScript("localStorage.setItem('user_prefs', '{\"theme\": \"dark\"}')");
}

@Test
void dashboardWithoutLogin() {
    open("/dashboard");
    $(".welcome-panel").should(appear);
}

@AfterMethod
void cleanupContext() {
    clearCookies();
    clearBrowserLocalStorage();
    Selenide.closeWebDriver();
}
```

## Best Practices

- Используйте `switchTo().defaultContent()` после работы с iframe для предотвращения `NoSuchElementException` в основном DOM
- Для работы с React/Vue/Angular формами используйте эмуляцию событий `input`/`change` через JS, если стандартный `setValue()` не триггерит валидацию
- Устанавливайте cookies только после перехода на `about:blank` или целевой домен, иначе браузер отклонит их по политике безопасности

  WebDriver должен иметь активную страницу с каким-либо URL (даже about:blank), иначе браузер не знает, к какому домену привязать cookie, и выбрасывает `InvalidCookieDomainException`

- Очищайте `localStorage`, `sessionStorage` и cookies в `@AfterMethod` для полной изоляции тестов
- Используйте `alert().accept()` с встроенными ожиданиями Selenide или `alert().withTimeout(Duration.ofSeconds(..)).accept()` — не добавляйте `Thread.sleep()` перед взаимодействием с модальными окнами
- Для сложных сценариев с несколькими вкладками сохраняйте `getWindowHandle()` в начале теста и восстанавливайте его в конце
- Не пытайтесь взаимодействовать с элементами внутри iframe без явного `switchTo().frame()` — WebDriver выбросит `ElementNotInteractableException`
- Не храните чувствительные cookies или токены в коде тестов — выносите их в переменные окружения или CI-secrets
- Не используйте `executeJavaScript` для обхода стандартных действий Selenide без веской причины — это ломает автоматические ожидания и логгирование
- Не забывайте про `open("about:blank")` перед установкой cookies, если тест стартует с пустого контекста

---

# 5. Архитектура тестов: Page Object и кастомизация

> Архитектурные паттерны построения поддерживаемых UI-тестов: Page Object/Component, ленивая инициализация, вынос локаторов, 
> создание кастомных условий и утилит для переиспользования логики.

## Структура Page Object

- **Поля-элементы** — декларативное объявление `SelenideElement` без мгновенного поиска в DOM
- **Методы-действия** — инкапсуляция бизнес-сценариев страницы (авторизация, навигация, отправка форм)
- **Возвращаемые страницы** — возврат `new NextPage()` для построения fluent-цепочек между тестами
- **Инкапсуляция селекторов** — скрытие локаторов внутри классов, запрет прямого вызова `$()` из тестов

```java
public class LoginPage {
    
    // Поля-элементы (ленивая инициализация)
    private final SelenideElement usernameInput = $("#username");
    private final SelenideElement passwordInput = $("#password");
    private final SelenideElement submitBtn = $("button[type='submit']");

    // Методы-действия
    public DashboardPage login(String user, String pass) {
        usernameInput.setValue(user);
        passwordInput.setValue(pass);
        submitBtn.click();
        return new DashboardPage(); // Возврат следующей страницы
    }
}
```

## Ленивая инициализация и вынос локаторов

- `SelenideElement` не ищет элемент в DOM при объявлении поля — поиск происходит только при вызове метода
- Вынос селекторов в `private static final String` или `By` константы упрощает поддержку при рефакторинге
- Использование `By` вместо строк обеспечивает type-safety и подсказки в IDE
- Хранение текстов, ошибок и констант в отдельных классах или `.properties` файлах

```java
public class UserPage {

    private static final By PROFILE_LINK = By.cssSelector("a.profile");
    private static final By EDIT_BTN = By.id("edit-user");
    private static final String SUCCESS_MSG = "Профиль успешно обновлён";

    private final SelenideElement profileLink = $(PROFILE_LINK);
    private final SelenideElement editBtn = $(EDIT_BTN);

    public boolean isProfileVisible() {
        return profileLink.isDisplayed();
    }
}
```

## Композиция и вложенные компоненты

- **Page Component** — переиспользуемые блоки интерфейса (хедер, футер, модальные окна, виджеты)
- **Композиция через поля** — включение компонентов в Page Object без глубокого наследования
- **Базовые классы** — общие методы навигации, ожиданий лоадеров, снятия скриншотов
- **Разделение ответственности** — UI-логика в компонентах, бизнес-сценарии и ассерты в тестах

> Композиция в Page Object — это сборка страницы из независимых компонентов-кирпичиков вместо наследования от базового класса со всей логикой. 
> 
> Каждый компонент отвечает только за свой кусок UI и может переиспользоваться на разных страницах.

```java
// Компонент — переиспользуемый блок, который живёт на многих страницах
public class HeaderComponent {

    private final SelenideElement logo = $(".logo");
    private final SelenideElement userMenu = $(".user-dropdown");

    public void openProfile() {
        userMenu.click();
        $("#profile-link").click();
    }
}

// Страница собирается из компонентов через поля
public class DashboardPage {

    private final HeaderComponent header = new HeaderComponent();  // композиция
    private final SelenideElement statsWidget = $(".stats");

    public HeaderComponent getHeader() {
        return header;  // отдаём компонент наружу для взаимодействия
    }
}

// Тест работает с компонентом через страницу
@Test
void navigateViaHeader() {
    DashboardPage page = new DashboardPage();  // страница готова
    page.getHeader()                           // получаем компонент
            .openProfile();                    // используем его метод
    
    // профиль открыт, тест продолжается
}
```

## Кастомные условия и обёртки элементов

- Расширение `Condition` для бизнес-специфичных проверок (например, `isValidEmail()`, `isFullyLoaded()`)
- Создание утилитных методов-обёрток над `SelenideElement` для сложных или повторяющихся действий
- Интеграция с `should()` через `Condition.condition()` или статические фабрики без потери fluent-синтаксиса

```java
// Кастомное условие
public class ValidEmailCondition extends Condition {
    
    public ValidEmailCondition() {
        super("valid-email-format"); 
    }
    
    @Override
    public boolean apply(Driver driver, WebElement element) {
        return Pattern.matches("^[\\w-.]+@([\\w-]+\\.)+[\\w-]{2,4}$", element.getText());
    }
}

// Использование
$(".user-email").should(new ValidEmailCondition());

// Метод-обёртка для сложного сценария
public static void waitAndClickWithRetry(SelenideElement element, Duration timeout) {
    element.withTimeout(timeout).click();
}
```

## Глобальные утилиты и переиспользование логики

- **BaseTest / BasePage** — общая инициализация, `@BeforeEach` настройка, автоматические скриншоты
- **SelenideUtils** — статические методы для частых операций (скролл, ожидание лоадера, массовая очистка форм)
- **Контекстные хелперы** — обёртки над `Configuration`, управление драйвером, логирование шагов
- **Изоляция тестовых данных** — генерация уникальных идентификаторов, очистка состояния между прогонами

```java
public final class SelenideUtils {
    
    private SelenideUtils() {}

    public static void waitForLoaderDisappear(Duration maxWait) {
        $(".loader").waitWhile(visible, maxWait);
    }

    public static void clearAllInputs(ElementsCollection forms) {
        forms.filterBy(visible).forEach(SelenideElement::clear);
    }

    public static void scrollToBottom() {
        Selenide.executeJavaScript("window.scrollTo(0, document.body.scrollHeight);");
    }
}
```

## Best Practices

- Используйте композицию (`HeaderComponent`, `ModalComponent`) вместо глубокого наследования для гибкости
- Объявляйте `SelenideElement` как `final` поля — ленивая инициализация не требует `null`-проверок
- Выносите локаторы и тексты в константы или внешние конфиги, чтобы изменения вёрстки требовали правки в одном месте
- Создавайте кастомные `Condition` только для повторяющихся бизнес-проверок, не дублируйте встроенные
- Возвращайте `this` или новую страницу из методов Page Object для поддержки fluent-цепочек в тестах
- Не создавайте Page Object с сотнями полей — разделяйте на логические компоненты (композиция)
- Не используйте `@FindBy` из Selenium — Selenide не требует `PageFactory`, он работает "из коробки"

  - Selenide сам делает ленивую инициализацию через $(selector)
  - PageFactory.initElements() — лишний шаг, который не даёт выигрыша
  - WebElement не умеет ждать и самовосстанавливаться, в отличие от SelenideElement
  - Код становится проще и короче без аннотаций и инициализации фабрики

- Не кешируйте `SelenideElement` через `WebDriver.findElement()` — это ломает автоматические ожидания
- Не смешивайте UI-логику страницы с ассертами тестов — тест должен вызывать методы Page Object, а проверять только конечный результат
- Не храните состояние между тестами в полях Page Object — каждый тест должен работать с новым экземпляром страницы

---

# 6. Расширенные сценарии и паттерны автоматизации

> Паттерны обработки сложных UI-сценариев: многошаговые формы, динамический контент, работа с внешними зависимостями и data-driven тестирование.

## Многошаговые формы и управление состоянием

- **Сохранение промежуточного состояния** — использование Page Object с методами-цепочками, возвращающими `this` или следующую страницу
- **Валидация на каждом шаге** — инкапсуляция проверок в методы страницы, а не в тестовый класс
- **Навигация назад/вперед** — реализация методов `goBack()`, `cancel()` с восстановлением данных формы
- **Изоляция шагов** — передача данных между шагами через контекст-объекты, а не через глобальные переменные

```java
public class RegistrationFlow {

    public RegistrationFlow fillStep1(String email, String password) {
        $("#step1-email").setValue(email);
        $("#step1-pass").setValue(password);
        $("button.next").click();
        return this; // Возврат текущего состояния для цепочки
    }

    public ProfilePage completeStep2(String name, String phone) {
        $("#step2-name").setValue(name);
        $("#step2-phone").setValue(phone);
        $("button.submit").click();
        return new ProfilePage(); // Завершение и переход
    }
}

// Использование в тесте
@Test
void fullRegistration() {
    new RegistrationFlow()
        .fillStep1("user@mail.com", "Secure123!")
        .completeStep2("Иван", "+79001234567")
        .shouldHaveHeader("Добро пожаловать");
}
```

## Динамический контент: анимации, лоадеры, ленивая подгрузка

- **Ожидание исчезновения лоадеров** — использование `waitWhile(visible, timeout)` вместо `Thread.sleep()`
- **Стабилизация после анимаций** — Selenide автоматически повторяет запросы, но для сложных CSS-анимаций может потребоваться явная проверка `display: none` или `opacity`
- **Ленивая подгрузка (Infinite Scroll)** — скролл до появления элементов, повторный запрос `$$()` после каждого рендера
- **Отслеживание сетевых запросов** — проверка завершения AJAX через ожидание специфичного DOM-состояния или использование прокси

```java
// Ожидание исчезновения индикатора загрузки
$(".loader-overlay").waitWhile(visible, Duration.ofSeconds(10)); // visible — экземпляр Condition, проверяющий, что элемент отображается на странице (не display: none, не visibility: hidden, имеет размеры)

// Бесконечная прокрутка с проверкой подгрузки
int initialSize = $$(".product-card").size();
Selenide.executeJavaScript("window.scrollTo(0, document.body.scrollHeight)");
$$(".product-card").shouldHave(sizeGreaterThan(initialSize));

// Проверка завершения анимации через CSS-свойство
$(".animated-box").shouldHave(cssValue("opacity", "1")); // cssValue — это Condition в Selenide. Проверяет, что CSS-свойство элемента имеет ожидаемое значение
```

## Всплывающие элементы и асинхронные уведомления

- **Тултипы и поп-оверы** — использование `hover()` с последующим `should(appear)`
- **Toast-уведомления** — проверка появления и автоматического скрытия через `waitWhile`
- **Системные диалоги** — работа с `alert()`, `confirm()`, `prompt()` через встроенные методы Selenide
- **Ожидание динамических модальных окон** — поиск по `data-` атрибутам или роли, проверка `isDisplayed()` перед взаимодействием

```java
// Взаимодействие с тултипом
$(".info-icon").hover();
$(".tooltip-content").shouldBe(visible);

// Проверка toast-уведомления с таймаутом на появление и скрытие
Selenide.executeJavaScript("triggerToast()"); // Эмуляция вызова
$(".toast-message").withTimeout(ofSeconds(5)).should(appear);
$(".toast-message").waitWhile(visible, ofSeconds(3)); // Ожидание авто-скрытия

// Обработка системного диалога
$("button.delete").click();
alert().accept();
$(".success-banner").should(appear);
```

## Внешние зависимости: моки, стабы и прокси

- **Proxy-перехват запросов** — встроенный `Selenide proxy` или интеграция с BrowserMob Proxy / WireMock
- **Мок API-ответов** — возврат фиктивных данных без вызова реального бэкенда
- **Стабилизация тестов** — изоляция UI-логики от нестабильных внешних сервисов
- **Верификация запросов** — проверка URL, заголовков и тела отправленных запросов

```java
// Настройка Selenide Proxy (встроенный)
Configuration.proxyEnabled = true;
Configuration.proxyHost = "localhost";

// Использование BrowserMob Proxy для моков
BrowserMobProxy proxy = new BrowserMobProxyServer();
proxy.start(0);
proxy.addResponseFilter((response, contents, messageInfo) -> {
    if (messageInfo.getOriginalUrl().contains("/api/users")) {
        contents.setText("[]"); // Возврат пустого списка
        response.setStatus(HttpResponseStatus.OK);
    }
});

// Интеграция с WireMock
wireMockServer.stubFor(get(urlEqualTo("/api/config"))
    .willReturn(aResponse()
    .withStatus(200)
    .withBody("{\"theme\":\"dark\"}")));
```

## Data-driven подход и параметризация

- **Источники данных** — CSV, JSON, YAML, БД, внешние API-фикстуры
- **JUnit 5 / TestNG** — `@ParameterizedTest`, `@CsvSource`, `@DataProvider`, `@ValueSource`
- **Генерация уникальных данных** — использование библиотек вроде `DataFaker`, `UUID`, временных меток
- **Изоляция данных** — очистка или использование транзакций для предотвращения конфликтов между параллельными запусками

```java
// JUnit 5 ParameterizedTest
@ParameterizedTest(name = "Login attempt: {0}")
@CsvSource({
    "valid_user,  correct_pass, true",
    "invalid_user, wrong_pass, false",
    "admin, admin_123, true"
})
void dataDrivenLogin(String user, String pass, boolean expectedResult) {
    open("/login");
    $("#username").setValue(user);
    $("#password").setValue(pass);
    $("button[type=submit]").click();

    if (expectedResult) {
        $(".dashboard").should(appear);
    } else {
        $(".error-message").should(appear);
    }
}

// Генерация данных через DataFaker
Faker faker = new Faker(new Locale("ru"));
String uniqueEmail = faker.internet().emailAddress();
String randomName = faker.name().fullName();
```

## Best Practices

- Используйте цепочки методов в Page Object для многошаговых сценариев, возвращая `this` или следующую страницу
- Для ожидания лоадеров применяйте `waitWhile(visible)` вместо `Thread.sleep()` — это ускоряет тесты и делает их стабильнее
- При работе с `Infinite Scroll` или ленивой подгрузкой всегда повторяйте запрос `$$()` после скролла, так как DOM меняется
- Изолируйте UI-тесты от нестабильных внешних API через моки (WireMock) или встроенный прокси Selenide
- Параметризуйте тесты через `@CsvSource` / `@DataProvider`, вынося данные в отдельные файлы для читаемости
- Генерируйте уникальные данные (`UUID`, `DataFaker`) для каждого запуска, чтобы избежать конфликтов в БД или кэше
- Не используйте `Thread.sleep()` для ожидания анимаций или сетевых запросов — это антипаттерн, ломающий стабильность
- Не передавайте состояние между шагами через `static` поля или глобальные переменные — это нарушает изоляцию тестов
- Не кешируйте элементы всплывающих окон в полях Page Object — они создаются динамически и часто меняются в DOM
- Не смешивайте мок-данные с реальными вызовами API в одном тесте без явного разделения контекста — это усложняет отладку

---

# 7. Отладка, диагностика и отчётность

> Автоматическая фиксация состояния тестов, интеграция с Allure, логирование, анализ падений и инструменты для дистанционной отладки в CI/CD.

## Автоматические скриншоты и HTML-дампы

- **Авто-скриншоты** — встраиваются через `ScreenShooterExtension` или `Configuration.screenshots = true`
- **Сохранение исходного кода страницы** — `Configuration.savePageSource = true` для постмортем-анализа DOM
- **Кастомные скриншоты** — вызов `SelenideElement.screenshot()` или `Selenide.webdriver().getScreenshotAs()` в конкретных шагах
- **Папка отчётов** — настраивается через `Configuration.reportsFolder` (по умолчанию `build/reports/tests`)

```java
class DebugTest {

    @RegisterExtension
    static ScreenShooterExtension screenshots = new ScreenShooterExtension()
            .to("build/test-reports/screenshots")
            .forFailedTestsOnly(); // или .forFailedTestsOnly(false) для всех тестов

    @Test
    void captureCustomState() {
        open("/dashboard");
        // Явное создание скриншота конкретного элемента
        $(".critical-section").screenshot();
        // Проверка с автоматическим дампом HTML при падении
        $(".dynamic-widget").should(appear);
        // Если проверка (should(appear)) не проходит — Selenide автоматически сохраняет исходный код HTML страницы в файл в тот момент, когда тест падает
    }
}
```

> Когда вы включаете Configuration.savePageSource = true (или используете ScreenShooterExtension, который это подразумевает), при каждом падении теста Selenide делает две вещи:
> - Скриншот экрана (PNG)
> - Дамп HTML — сохраняет полный HTML-код текущей страницы (.html файл)

## Видео-запись выполнения теста

- **Selenoid / Ggr** — нативная поддержка видео через `enableVideo: true` в capabilities
- **Локальная запись** — использование `org.testcontainers` или сторонних утилит (FFmpeg + VNC)
- **Интеграция с CI** — сохранение `.mp4` артефактов и привязка к Allure-отчётам
- **Управление размером** — ограничение качества и длительности для экономии места в артефактах

```properties
# selenide.properties или system properties для Selenoid
selenoid.enabled=true
selenoid.video.enabled=true
selenoid.video.frameRate=24
selenoid.video.quality=medium
```

```bash
# Пример docker-compose для Selenoid с видео
version: "3"
services:
  selenoid:
    image: aerokube/selenoid:latest
    ports: ["4444:4444"]
    volumes:
      - ./selenoid/config:/etc/selenoid
      - ./selenoid/video:/opt/selenoid/video
    environment:
      - OVERRIDES={"video":{"enabled": true, "quality": "medium"}}
```

> В Java-коде видео не нужно явно включать для каждого теста — оно записывается автоматически для всех тестов после того, 
> как вы один раз настроили конфигурацию. Но есть важные нюансы.

### Включение видео в Java-коде

Способ 1: Через capabilities (для Selenoid):

```java
public class OptimalVideoConfig {
    
    @BeforeAll
    static void optimalSetup() {
        DesiredCapabilities caps = new DesiredCapabilities();
        
        // 1. Включаем видео
        caps.setCapability("enableVideo", true);
        
        // 2. Ограничиваем качество для экономии места
        caps.setCapability("videoQuality", "medium"); // вместо "high"
        caps.setCapability("videoFrameRate", 15);     // вместо 24
        
        // 3. Записываем только падающие тесты
        caps.setCapability("videoSaveMode", "failed");
        
        // 4. Лимит размера видео (в МБ)
        caps.setCapability("videoSizeLimit", 100);

        Configuration.browserCapabilities = caps;
        /*
        Поскольку Configuration.browserCapabilities — это статическое поле, оно является единым для всех тестов, выполняющихся в рамках одного Java-процесса. 
        Если вы измените его в одном тесте, эти изменения затронут все последующие тесты
         */
    }
}
```

### Локальная конфигурация в Selenide 7.5.0+

> Начиная с Selenide 7.5.0 появилась возможность задавать конфигурацию (включая capabilities) локально для одного конкретного теста.
> 
> Вы можете передать объект SelenideConfig напрямую в метод `open()`. 
> 
> В этом случае настройки не будут влиять на глобальный Configuration

```java
public class OptimalVideoConfig {

    @Test
    void testWithLocalVideoConfig() {
        // 1. Создаем локальный объект конфигурации
        SelenideConfig config = new SelenideConfig();
        
        // 2. Создаем и настраиваем capabilities
        DesiredCapabilities caps = new DesiredCapabilities();
        caps.setCapability("enableVideo", true);
        caps.setCapability("videoQuality", "medium");
        caps.setCapability("videoFrameRate", 15);
        caps.setCapability("videoSaveMode", "failed");
        caps.setCapability("videoSizeLimit", 100);
        
        // 3. Применяем capabilities к нашей локальной конфигурации
        config.browserCapabilities(caps);
        
        // 4. Открываем браузер с этой конфигурацией
        // Глобальный Configuration при этом НЕ меняется
        open("https://example.com/dashboard", config);
        
        // ... ваши тестовые шаги
    }
}
```

## Логирование и фильтрация чувствительных данных

- **SLF4J / Logback** — интеграция через `selenide.logback.xml` или `logback-test.xml`
- **Уровни логирования** — `DEBUG` для трассировки действий, `INFO` для шагов, `WARN/ERROR` для падений
- **Маскирование данных** — фильтрация паролей, токенов, PII через кастомные аппендеры или `SelenideCommandListener`
- **Логирование шагов** — использование `SelenideLogger.begin()` и `.commit()` для ручного трекинга

```xml
<!-- logback-test.xml -->
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.codeborne.selenide" level="DEBUG"/>
    <logger name="com.codeborne.selenide.impl" level="INFO"/>
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

## Интеграция с Allure

- **Аннотации** — `@Story`, `@Feature`, `@Severity`, `@Description` для группировки и приоритизации
- **Allure Steps** — `@Step` для методов Page Object и `Allure.step()` для динамических шагов
- **Вложения** — скриншоты, логи, JSON-ответы через `Allure.addAttachment()`
- **Метаданные и теги** — `allure.label.epic`, `allure.parameter` для фильтрации в отчётах

```java
@Feature("Авторизация")
@Story("Вход в систему")
@Severity(SeverityLevel.CRITICAL)
class AuthTest {

    @Test
    @Description("Проверка успешного входа с валидными данными")
    void successfulLogin() {
        Allure.step("Открытие страницы логина", () -> Selenide.open("/login"));
        Allure.step("Ввод учётных данных", () -> {
            $("#user").setValue("admin");
            $("#pass").setValue("secret");
        });
        Allure.step("Отправка формы", () -> $("button.submit").click());
        Allure.step("Проверка перехода в дашборд", () -> $(".dashboard").should(appear));
    }
}
```

## Best Practices

- Включайте `savePageSource = true` и автоматические скриншоты только для падающих тестов в CI, чтобы не раздувать артефакты
- Используйте `Allure.step()` для логирования бизнес-действий, а не для обёртывания каждого `$().click()` — отчёт должен быть читаемым
- Маскируйте пароли и токены в логах через кастомный `SelenideCommandListener` или фильтрацию на уровне Logback
- Для удалённых запусков (Selenoid/Grid) обязательно сохраняйте видео и VNC-запись, привязывая их к Allure через `addAttachment`
- При анализе падений сначала смотрите HTML-дамп страницы, затем скриншот, и только потом стектрейс — это ускоряет локализацию причины
- Не используйте `Thread.currentThread().sleep()` в отладке — применяйте `Configuration.timeout` или `Debugger.pause()`
- Не включайте `DEBUG` логирование Selenide в продакшн-CI без ротации файлов — это быстро заполняет диск
- Не сохраняйте скриншоты всех тестов в репозиторий или базовую ветку — используйте временное хранилище с TTL
- Не пытайтесь отлаживать тесты в headless-режиме через UI-инспектор браузера без VNC или DevTools-проброса
- Не оставляйте `Allure.step()` с пустыми или неинформативными названиями — это снижает ценность отчёта для аналитиков и разработчиков

---

# 8. Интеграция с API, тестовые данные и безопасность

> Комбинирование UI- и API-тестов, управление жизненным циклом тестовых данных, обеспечение безопасности секретов и изоляции окружений.

## Интеграция с API (REST Assured + Selenide)

- **Подготовка данных** — создание пользователей, заказов, конфигураций через API перед открытием UI
- **Верификация результатов** — проверка состояния БД или ответов API после UI-действий
- **Общий контекст** — передача токенов, cookies, session-id между REST Assured и Selenide
- **Сериализация/десериализация** — использование Jackson/Gson для DTO в запросах и ответах

```java
public class ApiSetupHelper {

    public static String createUserViaApi(String username, String password) {
        return given()
            .contentType("application/json")
            .body(new UserDto(username, password))
            .when()
            .post("/api/users")
            .then()
            .statusCode(201)
            .extract()
            .jsonPath().getString("token");
    }
}

// Использование в тесте: API-подготовка + UI-валидация
@Test
void createAndVerifyUser() {
    // 1. Подготовка через API
    String token = ApiSetupHelper.createUserViaApi("testuser", "pass123");

    // 2. Установка токена в браузер
    Selenide.open("about:blank");
    WebDriverRunner.getWebDriver().manage().addCookie(
        new Cookie("auth_token", token, "/", null, true, false)
    );

    // 3. UI-сценарий
    open("/profile");
    $(".user-name").shouldHave(text("testuser"));

    // 4. Верификация через API после UI-действия
    $("button.deactivate").click();
    given()
        .header("Authorization", "Bearer " + token)
        .when()
        .get("/api/users/me/status")
        .then()
        .body("status", equalTo("INACTIVE"));
}
```

---

## Генерация и управление тестовыми данными

- **Фабрики данных** — использование `DataFaker`, `UUID`, `Instant.now()` для уникальности
- **Фикстуры** — JSON/XML шаблоны, загружаемые из `resources`
- **Очистка после теста** — стратегии `DELETE` через API, SQL-транзакции с `ROLLBACK`, или перезапуск контейнеров
- **Изоляция запусков** — префиксы для тестовых сущностей (`test_`, `qa_`), предотвращение конфликтов в параллельных прогонах

```text
Ваш проект:
├── src/
│   └── test/
│       └── resources/
│           └── fixtures/              ← Папка с тестовыми данными
│               ├── user.json          ← фикстура пользователя
│               ├── order.xml          ← шаблон заказа
│               ├── response.json      ← ожидаемый ответ API
│               └── product-data.json  ← данные для POST запроса
```

```java
public class TestDataFactory {
    
    private static final Faker faker = new Faker(new Locale("ru"));

    public static String generateUniqueEmail() {
        return faker.internet().emailAddress() + "+" + UUID.randomUUID().toString().substring(0, 8) + "@test.local";
    }

    public static String loadFixture(String fileName) throws IOException {
        // Путь к файлу всегда относительно resources/
        // "fixtures/user.json" загрузит "src/test/resources/fixtures/user.json"
        try (InputStream is = TestDataFactory.class.getClassLoader().getResourceAsStream("fixtures/" + fileName)) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}

@AfterEach
void cleanupTestData() {
    // Очистка через API-хелпер
    String testUserId = getCreatedUserId();
    if (testUserId != null) {
        ApiCleanupHelper.deleteUser(testUserId);
    }
    // Альтернатива: очистка через JDBC-коннектор в тестовой БД
    JdbcExecutor.execute("DELETE FROM users WHERE email LIKE '%@test.local'");
}
```

---

## Безопасность и защита окружений

- **Хранение секретов** — ENV-переменные, CI/CD Secrets (GitHub/GitLab), HashiCorp Vault, AWS Secrets Manager
- **Маскирование в логах** — фильтрация паролей, токенов, PII перед записью в `slf4j` или `Allure`
- **Изоляция данных** — строгое разделение `dev`, `staging`, `prod` через `Configuration.baseUrl` и ENV
- **Защита от accidental prod-impact** — валидация URL перед запуском, блокировка `prod` в `@BeforeAll`

```bash
# Передача секретов через переменные окружения в CI

# 1. Устанавливаем переменные окружения
export SELENIDE_BASE_URL=https://staging.example.com     # ← url тестируемого приложения
export API_TEST_TOKEN=eyJhbGciOiJIUzI1NiIs...            # ← токен для API-тестов

# 2. Запускаем тесты, передавая параметр для Selenoid
mvn test -Dselenide.remote=http://grid:4444/wd/hub       # ← url сервера с браузерами
```

```java
// Безопасная инициализация с проверкой окружения
@BeforeAll
static void preventProdAccidents() {
    String envUrl = System.getenv("SELENIDE_BASE_URL");
    if (envUrl != null && envUrl.contains("prod")) {
        throw new IllegalStateException("Запуск тестов на PRODUCTION запрещён!");
    }
    Configuration.baseUrl = System.getProperty("app.url", "https://dev.example.com");
}
```

---

## Негативные тесты и валидация

- **Валидация входных данных** — проверка сообщений об ошибках при вводе некорректных форматов
- **XSS/SQL-инъекции** — эмуляция опасных payloads в полях ввода, проверка экранирования
- **Обработка ошибок API** — имитация `500`, `403`, `401` ответов и проверка UI-поведения
- **Граничные значения** — пустые строки, максимальная длина, спецсимволы, emoji

```java
@ParameterizedTest
@ValueSource(strings = {"<script>alert(1)</script>", "'; DROP TABLE users; --", "A".repeat(5000)})
void negativeInputValidation(String maliciousPayload) {
    open("/comment-form");
    $("#comment-body").setValue(maliciousPayload);
    $("button.submit").click();

    // Проверка корректной обработки вредоносного ввода
    $(".error-banner").should(appear);
    $("#comment-body").shouldNotHave(attribute("value", maliciousPayload)); // Проверка экранирования/очистки
}
```

---

## Best Practices

- Используйте API-фикстуры и `REST Assured` для подготовки данных вместо ручного прохождения UI-сценариев
- Генерируйте уникальные идентификаторы (`UUID`, `DataFaker`) для каждого теста, чтобы избежать конфликтов при параллельных запусках
- Всегда очищайте созданные сущности в `@AfterEach` через API или SQL, чтобы не оставлять мусор в БД
- Храните секреты исключительно в ENV или CI/CD Vault, никогда не коммитьте токены в `selenide.properties` или код
- Блокируйте запуск тестов на `prod` окружении через проверку `baseUrl` в `@BeforeAll`
- Эмулируйте ошибки `4xx/5xx` через WireMock или моки для проверки устойчивости UI к сбоям бэкенда
- Не используйте реальные данные клиентов или продакшн-пароли в тестах
- Не полагайтесь на порядок выполнения тестов для передачи данных между ними
- Не оставляйте тяжёлые объекты (WebDriver, REST Assured Response) в статических полях без очистки
- Не игнорируйте валидацию URL перед запуском — это главная причина accidental prod-impact
- Не тестируйте XSS/SQL-инъекции на боевых базах данных — используйте изолированные контейнеры или мок-сервисы

---

# 9. Инфраструктура, CI/CD и кросс-браузерное тестирование

> Масштабирование UI-тестов: параллельный запуск, интеграция с Docker/Selenoid/Grid, настройка CI/CD пайплайнов и кросс-браузерное тестирование.

## Параллельное выполнение и многопоточность

- **JUnit 5** — настройка через `junit-platform.properties` или `@Execution(ExecutionMode.CONCURRENT)`
- **TestNG** — атрибут `parallel="methods"/"classes"/"tests"` в `testng.xml`
- **Изоляция потоков** — Selenide автоматически привязывает `WebDriver` к текущему потоку (`ThreadLocal<WebDriver>`)
- **Ограничения** — статические `Configuration` параметры применяются глобально.

  Для разных настроек браузера в одном запуске используйте `WebDriverRunner.setWebDriver()` или `@BeforeEach` с кастомной инициализацией

Пример для JUnit:

> JUnit создаёт отдельный экземпляр тестового класса для каждого тестового метода, 
> поэтому Selenide может автоматически управлять драйверами через ThreadLocal — вам достаточно просто добавить `@Execution(CONCURRENT)`.

```java
@Execution(ExecutionMode.CONCURRENT)  // JUnit запускает методы параллельно
class ParallelTests {

  @Test
  void testA() {
    open("/page1");      // ← Поток 1: свой браузер
    $("button").click(); // ← Поток 1: работает с СВОИМ браузером
  }

  @Test
  void testB() {
    open("/page2");              // ← Поток 2: свой браузер (другой экземпляр)
    $("input").setValue("test"); // ← Поток 2: работает с СВОИМ браузером
  }
}
```

Пример для TestNG (где в testng.xml настроен параллелизм на уровне методов (`parallel="methods"`):

> TestNG по умолчанию создаёт один экземпляр класса для всех методов, 
> поэтому при `parallel="methods"` вы обязаны вручную создавать драйвер и вызывать `WebDriverRunner.setWebDriver(driver)` в каждом тесте, 
> иначе потоки будут использовать один и тот же браузер.

```java
public class ParallelTests {

    @Test
    void testChrome() {
        // Создаём и устанавливаем драйвер прямо в методе
        WebDriver driver = new ChromeDriver();
        WebDriverRunner.setWebDriver(driver);
        
        open("/login");
        $(".auth-form").shouldBe(visible);
    }

    @Test
    void testFirefox() {
        // Создаём и устанавливаем драйвер прямо в методе
        WebDriver driver = new FirefoxDriver();
        WebDriverRunner.setWebDriver(driver);
        
        open("/register");
        $(".signup-form").shouldBe(visible);
    }

    @AfterMethod
    void tearDown() {
        // Закрываем драйвер после каждого теста
        WebDriverRunner.closeWebDriver();
    }
}
```

Важные нюансы:

- Статические параметры Configuration
  
  Хотя драйверы изолированы, Configuration глобален

- Общие ресурсы (база данных, API)
  
  ThreadLocal защищает только WebDriver, но не ваши данные

---

## Запуск в Docker / Selenoid / Selenium Grid

- **Удалённый WebDriver** — настройка через `Configuration.remote = "http://selenoid:4444/wd/hub"`
- **Selenoid capabilities** — передача `browserVersion`, `enableVNC`, `enableVideo`, `screenResolution`
- **Selenium Grid 4** — маршрутизация запросов через `http://<host>:4444`, поддержка Docker-образов
- **Изоляция контейнеров** — каждый тест получает чистый экземпляр браузера с временным профилем

```yaml
# docker-compose.yml для Selenoid
version: "3"
services:
  selenoid:
    image: aerokube/selenoid:latest
    ports: ["4444:4444"]
    volumes:
      - ./selenoid/config:/etc/selenoid
      - ./selenoid/video:/opt/selenoid/video
      - /var/run/docker.sock:/var/run/docker.sock
    command: ["-conf", "/etc/selenoid/browsers.json", "-video-output-dir", "/opt/selenoid/video"]
```

```bash
# Запуск пайплайна с подключением к Selenoid
docker-compose up -d selenoid
mvn test -Dselenide.remote=http://localhost:4444/wd/hub \
         -Dselenide.browser=chrome \
         -Dselenide.headless=false
```

---

## Параметры браузеров в удалённом режиме

- **Версия браузера** — `browserVersion: "115.0"`, `browserName: "chrome"`
- **Разрешение экрана** — `screenResolution: "1920x1080x24"`
- **Дополнительные флаги** — `args: ["--no-sandbox", "--disable-dev-shm-usage", "--disable-gpu"]`
- **Кеширование и профили** — передача `--user-data-dir` для сохранения состояния между сессиями (не рекомендуется для изоляции)

Пример 1: Базовая настройка Selenoid с capabilities:

```java
public class SelenoidSetup {
    
    @BeforeAll
    static void setupSelenoid() {
        // Адрес Selenoid
        Configuration.remote = "http://localhost:4444/wd/hub";
        
        // Браузер и его версия
        Configuration.browser = "chrome";
        Configuration.browserVersion = "100.0";
        
        // Отключаем headless (Selenoid не поддерживает headless режим)
        Configuration.headless = false;
        
        // Настройки Selenoid через capabilities
        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("enableVNC", true);                  // Включить VNC для просмотра
        capabilities.setCapability("enableVideo", true);                // Включить видео запись
        capabilities.setCapability("screenResolution", "1920x1080x24"); // Разрешение экрана
        capabilities.setCapability("enableLog", true);                  // Логи браузера
        capabilities.setCapability("timeZone", "Europe/Moscow");        // Часовой пояс
        
        // Применяем capabilities
        Configuration.browserCapabilities = capabilities;
    }
}
```

Пример 2: Глобальная конфигурация через static-блок:

```java
public class RemoteConfig {
    static {
        Configuration.remote = "http://selenoid:4444/wd/hub";
        Configuration.browser = "chrome";
        Configuration.browserVersion = "115.0";

        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("enableVNC", true);
        capabilities.setCapability("enableVideo", true);
        capabilities.setCapability("screenResolution", "1920x1080x24");
        capabilities.setCapability("args", List.of("--no-sandbox", "--disable-dev-shm-usage"));
        Configuration.browserCapabilities = capabilities;
    }
}
```

---

## Интеграция с CI/CD и параметризация сборок

- **GitHub Actions / GitLab CI / Jenkins** — объявление stages: `test`, `report`, `notify`
- **Матричные сборки** — параллельный прогон на разных ОС/браузерах через `strategy.matrix`
- **Передача параметров** — `env` переменные или CLI флаги `-Dapp.env=staging -Dselenide.browser=chrome`
- **Публикация артефактов** — отчёты Allure, скриншоты, видео-записи сохраняются в `artifacts`

Для GitHub Actions:

```yaml
# .github/workflows/ui-tests.yml
# Этот файл определяет CI/CD пайплайн для GitHub Actions

# Название workflow (отображается в UI GitHub)
name: UI Tests

# Триггеры запуска: при push в любую ветку и при создании pull request
on: [push, pull_request]

# Описание одной или нескольких джоб
jobs:
  # Название джобы
  e2e:
    # Стратегия матричного запуска — создаёт комбинации параметров
    strategy:
      matrix:
        # Список браузеров для тестирования
        browser: [chrome, firefox]
        # Список операционных систем (раннеров GitHub)
        os: [ubuntu-latest, macos-latest]
        # Будет создано 4 параллельные джобы:
        # 1. chrome + ubuntu-latest
        # 2. chrome + macos-latest  
        # 3. firefox + ubuntu-latest
        # 4. firefox + macos-latest

    # Для каждой комбинации используется своя ОС
    runs-on: ${{ matrix.os }}

    # Шаги выполнения джобы
    steps:
      # 1. Клонируем репозиторий с кодом тестов
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Устанавливаем JDK (Java Development Kit)
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'           # Версия Java
          distribution: 'temurin'      # Дистрибутив (Eclipse Temurin)

      # 3. Запуск тестов с параметрами из матрицы
      - name: Run Tests
        # Переменные окружения, доступные в процессе тестирования
        env:
          SELENIDE_BROWSER: ${{ matrix.browser }}   # Какой браузер использовать
          APP_ENV: staging                           # Окружение для тестов
        # Команда запуска Maven (verify выполняет интеграционные тесты)
        run: mvn verify -DskipFrontend=true

      # 4. Сохраняем отчёты Allure как артефакты сборки
      - name: Upload Allure Report
        # always() — загружаем даже если тесты упали
        if: always()
        uses: actions/upload-artifact@v4
        with:
          # Уникальное имя артефакта (включает браузер для различия)
          name: allure-report-${{ matrix.browser }}
          # Путь к результатам Allure в проекте
          path: target/allure-results/
```

Аналог для Jenkins:

```groovy
pipeline {
    agent any
    
    parameters {
        choice(name: 'BROWSER', choices: ['chrome', 'firefox'], description: 'Browser for tests')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Test environment')
    }
    
    stages {
        stage('Test') {
            steps {
                sh """
                    mvn test \
                        -Dselenide.browser=${params.BROWSER} \
                        -Dapp.env=${params.ENV}
                """
            }
        }
    }
    
    post {
        always {
            allure includeProperties: false, jdk: '17', results: [[path: 'target/allure-results']]
        }
    }
}
```

Аналог для GitLab CI:

```yaml
# .gitlab-ci.yml
ui-tests:
  parallel:
    matrix:
      - BROWSER: chrome
      - BROWSER: firefox
  variables:
    APP_ENV: staging
  script:
    - mvn test -Dselenide.browser=$BROWSER
  artifacts:
    when: always
    paths:
      - target/allure-results/
    reports:
      junit: target/surefire-reports/TEST-*.xml
```

```java
// В коде тестов параметры можно получить так:
public class TestConfig {
    
    @BeforeAll
    static void loadFromEnv() {
        // Переменная из CI окружения
        String browser = System.getenv("SELENIDE_BROWSER");
        if (browser != null) {
            Configuration.browser = browser;
        }
        
        String appEnv = System.getenv("APP_ENV");
        if ("staging".equals(appEnv)) {
            Configuration.baseUrl = "https://staging.example.com";
        }
    }
}
```

---

## Кросс-браузерное тестирование и эмуляция

- **Матрица покрытия** — приоритизация браузеров по статистике пользователей (Chrome, Safari, Firefox, Edge)
- **Headless-режим в CI** — ускорение прогона, экономия ресурсов, отключение `Xvfb`
- **Эмуляция мобильных устройств** — `--mobile-emulation`, `deviceName` в capabilities, viewport scaling
- **Выборочный запуск** — тегирование тестов (`@Smoke`, `@CrossBrowser`) и фильтрация в `testng.xml` или `@Tag`

Конфигурация эмуляции мобильного устройства через selenide-mobile.properties:

```properties
# selenide-mobile.properties
browser=chrome
mobileEmulationEnabled=true
mobileDeviceName=Pixel 5
screenResolution=1080x1920x24
headless=true
```

Настройка эмуляции Android-устройства через DesiredCapabilities:

```java
public class MobileConfig {
    static {
        browser = "chrome";
        headless = true;
        DesiredCapabilities caps = new DesiredCapabilities();
        caps.setCapability("mobileEmulation", Map.of("deviceName", "Pixel 5"));
        browserCapabilities = caps;
    }
}
```

Тегирование тестов для выборочного кросс-браузерного запуска:

```java
public class CrossBrowserTest {
    
    @Test
    @Tag("smoke")                    // Дымовые тесты
    @Tag("cross-browser")            // Требуют кросс-браузерной проверки
    void loginTest() {
        open("/login");
        $("#username").setValue("user");
        $(".auth-form").shouldBe(visible);
    }
    
    @Test
    @Tag("regression")               // Только критичные сценарии проверяем на всех браузерах
    void complexWorkflowTest() {
        open("/dashboard");
        $(".report").shouldBe(visible);
    }
    
    @Test
    @Tag("mobile-only")              // Только для мобильной эмуляции
    void mobileMenuTest() {
        open("/menu");
        $(".hamburger").click();
        $(".mobile-nav").shouldBe(visible);
    }
}
```

Запуск только определённых тегов через Maven:

```bash
# Только кросс-браузерные тесты
mvn test -Dgroups="cross-browser"

# Дымовые тесты + кросс-браузерные
mvn test -Dgroups="smoke,cross-browser"

# Исключить мобильные тесты
mvn test -DexcludedGroups="mobile-only"
```

Эмуляция разных мобильных устройств:

```java
class MobileEmulationTest {
    
    @ParameterizedTest(name = "Тест на {0}")
    @CsvSource({
        "Pixel 5, 1080x1920x24",
        "iPhone 12, 1170x2532x24",
        "iPad Pro, 1366x1024x24",
        "Samsung Galaxy S20, 1440x3200x24"
    })
    void testMobileResponsive(String deviceName, String resolution) {
        // Настройка эмуляции
        DesiredCapabilities caps = new DesiredCapabilities();
        caps.setCapability("mobileEmulation", Map.of("deviceName", deviceName));
        caps.setCapability("screenResolution", resolution);
        
        Configuration.browserCapabilities = caps;
        Configuration.browser = "chrome";
        Configuration.headless = true;
        
        // Открываем адаптивный сайт
        open("/responsive-page");
        
        // Проверяем мобильную версию
        $(".mobile-header").shouldBe(visible);
        $(".desktop-menu").shouldNotBe(visible);
    }
}
```

---

## Best Practices

- Настройте `ThreadLocal` изоляцию или используйте встроенный `WebDriverRunner` для безопасного параллельного запуска
- Выносите конфигурацию удалённых браузеров в `browsers.json` (Selenoid) или `capabilities` файл
- Используйте матричные сборки в CI для кросс-браузерного тестирования, но ограничьте `firefox` и `edge` критическими путями для экономии ресурсов
- Включайте `headless=true` и `fastSetValue=true`(устанавливает значение через JavaScript одной операцией, а не нажатие каждой клавиши с событиями) в CI-профилях для ускорения прогона
- Автоматически публикуйте артефакты (Allure, скриншоты, видео) при статусе `always()` или `failure`, чтобы не терять контекст падений
- Эмулируйте мобильные устройства через `mobileEmulation` capabilities, а не через изменение `User-Agent` вручную
- Не запускайте UI-тесты без `--no-sandbox` и `--disable-dev-shm-usage`(в Docker-контейнерах размер `/dev/shm` по умолчанию составляет всего 64 МБ; современным браузерам этого часто не хватает, 
  особенно при работе со сложными страницами) в Docker-контейнерах — это вызовет краш браузера
- Не используйте `Configuration.remote` локально без отключения прокси и видео-записи — это замедлит отладку
- Не кешируйте `WebDriver` инстансы между разными пайплайнами или раннерами — каждый запуск должен быть изолирован
- Не тестируйте все браузеры на каждом коммите — используйте триггеры по тегам или ночные прогоны для полной матрицы
- Не игнорируйте `screenResolution` в удалённых запусках — адаптивная вёрстка может ломаться при нестандартных DPI

---

# 11. Псевдоклассы CSS

| Псевдокласс CSS      | Когда срабатывает                            | Проверка в Selenide                                    |
|:---------------------|----------------------------------------------|--------------------------------------------------------|
| `:hover`             | Курсор мыши наведён на элемент               | `hover()` (эмуляция)                                   |
| `:focus`             | Элемент в фокусе (готов принимать ввод)      | `$(el).shouldBe(focused)`                              |
| `:active`            | Элемент активирован (зажат клик мыши)        | -                                                      |
| `:visited`           | Ссылка уже была посещена в истории браузера  | -                                                      |
| `:link`              | Ссылка, которую ещё не посещали              | -                                                      |
| `:enabled`           | Элемент доступен для редактирования/кликов   | `$(el).shouldBe(enabled)`                              |
| `:disabled`          | Элемент заблокирован (атрибут `disabled`)    | `$(el).shouldBe(disabled)`                             |
| `:read-only`         | Поле только для чтения (`readonly`)          | `$(el).shouldHave(attribute("readonly"))`              |
| `:read-write`        | Поле доступно для редактирования             | -                                                      |
| `:checked`           | Радио-кнопка или чекбокс отмечены            | `$(el).shouldBe(checked)`                              |
| `:indeterminate`     | Чекбокс в неопределённом состоянии           | `$(el).executeJavaScript("return this.indeterminate")` |
| `:required`          | Поле обязательно для заполнения (`required`) | `$(el).shouldHave(attribute("required"))`              |
| `:optional`          | Поле необязательное (нет `required`)         | -                                                      |
| `:valid`             | Значение прошло HTML5-валидацию              | `$(el).is(":valid")`                                   |
| `:invalid`           | Значение НЕ прошло HTML5-валидацию           | `$(el).is(":invalid")`                                 |
| `:in-range`          | Число в `input[type=number]` внутри min/max  | -                                                      |
| `:out-of-range`      | Число вне диапазона min/max                  | -                                                      |
| `:placeholder-shown` | Плейсхолдер виден (поле пустое)              | -                                                      |
| `:empty`             | Элемент не содержит текста и дочерних узлов  | `$(el).shouldBe(empty)`                                |
| `:not(selector)`     | Логическое НЕ (исключение)                   | `$(el).is(":not(.class)")`                             |
| `:has(selector)`     | Содержит внутри указанный элемент            | `$(el).find(selector).exists()`                        |
| `:first-child`       | Первый дочерний элемент родителя             | -                                                      |
| `:last-child`        | Последний дочерний элемент родителя          | -                                                      |
| `:nth-child(n)`      | Элемент на позиции n (1-based)               | -                                                      |
| `:nth-of-type(n)`    | n-ный элемент своего типа                    | -                                                      |