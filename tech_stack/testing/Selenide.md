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

