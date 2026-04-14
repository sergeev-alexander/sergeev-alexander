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

---

# 1. Введение, настройка и конфигурация

> Философия Selenide и базовая настройка окружения для автоматизации UI-тестов: подключение, интеграция с фреймворками, управление конфигурацией.

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
> Поэтому `ConfigHelper.printActiveConfig()` покажет именно те значения, которые были вычислены с учетом файла, переменных окружения и системных свойств на момент запуска JVM.

---

## Динамическое изменение конфигурации

```java
import static com.codeborne.selenide.Configuration.*;
import static com.codeborne.selenide.Selenide.*;

import org.junit.jupiter.api.Test;

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
- Использование `Thread.sleep()` вместо встроенных ожиданий Selenide
- Изменение глобальной конфигурации без восстановления — приводит к "утечке" состояния между тестами
- Игнорирование `headless`-режима в CI — увеличивает время прогона и потребление ресурсов
- Хранение чувствительных данных (пароли, токены) в `selenide.properties` — используйте ENV или secrets-менеджеры

---

# 2. Core API: поиск и действия с элементами

> Core API Selenide предоставляет лаконичный Fluent-интерфейс для поиска, взаимодействия и проверки веб-элементов, 
> скрывая сложность нативного Selenium WebDriver и обеспечивая встроенные автоматические ожидания.

## Базовый синтаксис и Fluent API

- `$()` — поиск одного элемента, возвращает `SelenideElement`
- `$$()` — поиск коллекции элементов, возвращает `ElementsCollection`
- `$x()` — поиск по XPath-выражению
- Цепочки вызовов — последовательное применение методов без промежуточных переменных
- Lazy evaluation — элементы не ищутся до момента вызова действия или проверки

### Пример базового использования

```java
// Поиск и действие в одну строку
$("#submit-btn").click(); // Поиск по CSS Selector

// Цепочка Fluent API
$(".user-card").find(".name")
        .shouldHave(text("Иванов"))
        .parent().find(".status")
        .shouldBe(visible);

// Работа с коллекцией
$$(".list-item").first().click();
```

---

## Селекторы и локаторы

- **CSS-селекторы** — приоритетный способ, быстрее и читаемее: `.class`, `#id`, `[attr=value]`
- **XPath** — используется для сложной навигации или поиска по тексту/иерархии: `$x("//div[contains(text(),'Login')]")`
- **Поиск по тексту** — встроенные методы `byText()`, `withText()`
- **Комбинированные локаторы** — объединение атрибутов и классов: `input[name='email'][type='text']`

### Примеры селекторов

```java
// CSS
$("form#login input[type='password']")

// XPath
$x("//table//tr[td[text()='Admin']]/td[2]")

// Поиск по точному тексту
$(byText("Сохранить"))

// Поиск по частичному тексту
$(withText("Загрузка..."))

// Комбинированный атрибут
$("button[data-action='submit'].btn-primary")

---

## Навигация по DOM и коллекции

- **Поиск внутри родителя** — `parent()`, `ancestor()`, `sibling()`, `closest()`
- **Фильтрация коллекций** — `filterBy()`, `excludeWith()`, `findBy()`
- **Доступ по индексу** — `first()`, `last()`, `get(int index)`
- **Счётчики и состояния** — `size()`, `isEmpty()`, `areDisplayed()`

### Навигация и фильтрация

```java
import static com.codeborne.selenide.CollectionCondition.size;
import static com.codeborne.selenide.Selenide.$$;

ElementsCollection rows = $$(".data-table tbody tr");

// Фильтрация по условию
rows.filterBy(text("Активен")).shouldHave(size(3));

// Навигация от элемента
$(".highlighted-item").ancestor(".main-container").click();

// Работа с индексами (0-based)
$$(".menu-items").get(2).hover();

---

## Извлечение данных

- `getText()` / `getOwnText()` — полный текст элемента / текст без дочерних элементов
- `getAttribute("name")` — значение атрибута
- `getCssValue("property")` — значение CSS-свойства
- `getValue()` / `getSelectedOption()` / `isSelected()` — состояния форм и элементов

### Извлечение значений

```java
// Текст
String header = $(".page-title").getText();

// Атрибут
String href = $("a.download-link").getAttribute("href");

// CSS-свойство
String color = $(".alert-danger").getCssValue("background-color");

// Значение инпута
String email = $("input[name='email']").getValue();

// Состояние чекбокса
boolean isChecked = $("input#terms").isSelected();

---

## Действия с элементами

- `click()` / `doubleClick()` / `contextClick()` — различные типы кликов
- `setValue()` / `clear()` — ввод и очистка полей
- `pressEnter()` / `pressEscape()` — отправка спецклавиш
- `hover()` — наведение курсора
- `dragAndDropTo()` — перетаскивание

### Взаимодействие и ввод

```java
import static com.codeborne.selenide.Selenide.*;

// Быстрый ввод (без эмуляции нажатий клавиш, если включён fastSetValue)
$("#username").setValue("admin");

// Очистка и ввод
$("#search").clear().setValue("Selenide").pressEnter();

// Двойной клик
$("li.item").doubleClick();

// Правый клик (контекстное меню)
$(".editable-area").contextClick();

// Наведение для появления тултипа
$(".tooltip-trigger").hover();

---

## Работа с формами

- **Выпадающие списки** — `selectOption()`, `selectOptionContainingText()`
- **Чекбоксы** — `setSelected(true)`, `setSelected(false)`
- **Радио-кнопки** — `click()` или `setValue()` для выбора
- **Загрузка файлов** — `uploadFromClasspath()`, `uploadFile()`

### Примеры работы с формами

```java
import java.io.File;
import static com.codeborne.selenide.Selenide.$;

// Выпадающий список (select)
$("#country").selectOption("Россия");
$("#role").selectOptionContainingText("Менеджер");

// Чекбокс
$("input#agree").setSelected(true);

// Радио-кнопка
$("[name='gender']").findBy(value("female")).click();

// Загрузка файла
$("#file-upload").uploadFile(new File("src/test/resources/document.pdf"));

// Загрузка из classpath
$("#avatar").uploadFromClasspath("images/profile.png");

---

## Drag-and-drop и сложные сценарии

- Перетаскивание между элементами — `dragAndDropTo()`
- Эмуляция сложного ввода через Actions API (доступен через `.toWebElement()`)
- Комбинирование клавиш и мыши для нестандартных UI-паттернов

### Пример Drag-and-Drop

```java
import static com.codeborne.selenide.Selenide.$;
import org.openqa.selenium.interactions.Actions;
import com.codeborne.selenide.WebDriverRunner;

// Простое перетаскивание
$("#draggable-item").dragAndDropTo($("#drop-zone"));

// С использованием WebDriver Actions для нестандартных сценариев
Actions actions = new Actions(WebDriverRunner.getWebDriver());
actions.dragAndDrop($("#source").toWebElement(), $("#target").toWebElement()).perform();

---

## Best Practices

- Используйте CSS-селекторы вместо XPath, когда это возможно — они быстрее и стабильнее
- Опирйтесь на `data-*` атрибуты (например, `data-testid`) для изоляции тестов от изменений вёрстки
- Избегайте жестких привязок к индексам в коллекциях — используйте фильтрацию по тексту или атрибутам
- Не используйте `Thread.sleep()` — Selenide автоматически ожидает появления и готовности элементов
- Для ввода текста в большие формы используйте `fastSetValue = true` в конфигурации, но учитывайте, что это не эмулирует реальные события `oninput`/`onchange`
- При работе с динамическими коллекциями всегда вызывайте `$$()` заново, а не кешируйте `ElementsCollection` надолго
- ❌ Не полагайтесь на абсолютные XPath (`/html/body/div[2]/...`) — они ломаются при малейшем изменении DOM
- ❌ Не используйте `getValue()` для получения текста неинпутовых элементов — применяйте `getText()` или `getOwnText()`
- ❌ Не забывайте про `hover()` перед взаимодействием с элементами, которые появляются только при наведении
- ❌ Избегайте смешивания нативного `WebElement` и `SelenideElement` без явной необходимости — это нарушает единую модель ожиданий