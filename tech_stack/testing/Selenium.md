**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Selenium

## Содержание

1. [Введение и Основы](#1-введение-и-основы)
2. [Настройка окружения и Управление драйверами](#2-настройка-окружения-и-управление-драйверами)
3. [Локаторы и Поиск элементов](#3-локаторы-и-поиск-элементов)
4. [Взаимодействие с элементами (WebElement & Actions API)](#4-взаимодействие-с-элементами-webelement--actions-api)
5. [Синхронизация и Ожидания (Waits)](#5-синхронизация-и-ожидания-waits)
6. [Управление контекстом браузера](#6-управление-контекстом-браузера)
7. [Selenium 4: Продвинутые возможности](#7-selenium-4-продвинутые-возможности)
8. [Архитектура](#8-архитектура)
9. [Отладка, Логирование и Диагностика](#9-отладка-логирование-и-диагностика)
10. [Selenium Grid 4 и Параллельное выполнение](#10-selenium-grid-4-и-параллельное-выполнение)
11. [Интеграция с тестовыми раннерами и экосистемой](#11-интеграция-с-тестовыми-раннерами-и-экосистемой)

# 1. Введение и Основы

> Selenium WebDriver — это инструмент для автоматизации взаимодействия с веб-браузерами, реализующий клиент-серверную архитектуру, где тестовый код (клиент) отправляет команды драйверу браузера (сервер) через стандартизированный протокол.

## Архитектура Selenium: Клиент-Сервер

- **Клиентская библиотека** (Java bindings) — API, с которым работает тестировщик: `WebDriver`, `WebElement`, `By`
- **Драйвер браузера** (chromedriver, geckodriver) — отдельный исполняемый файл, транслирующий команды WebDriver в нативные вызовы браузера
- **Браузер** — целевое приложение, выполняющее действия и возвращающее состояние DOM

```text
┌─────────────────┐     HTTP/JSON/W3C      ┌─────────────────┐
│   Тест на Java  │ -------------------->  │  ChromeDriver   │
│ (WebDriver API) │ <--------------------  │  (HTTP Server)  │
└─────────────────┘                        └────────┬────────┘
                                                    │
                                                    ▼
                                           ┌─────────────────┐
                                           │     Chrome      │
                                           │  (DevTools API) │
                                           └─────────────────┘
```

## Эволюция протоколов: JSON Wire → W3C Standard

| Аспект                 | Selenium 3 (JSON Wire)              | Selenium 4 (W3C WebDriver)               |
|:-----------------------|-------------------------------------|------------------------------------------|
| Протокол               | Собственный, не стандартизированный | Официальный стандарт W3C                 |
| Совместимость          | Зависела от реализации драйвера     | Единое поведение между браузерами        |
| Действия (Actions)     | `Actions` с ограничениями           | Полная поддержка W3C Actions API         |
| Capabilities           | `DesiredCapabilities` (устарело)    | `BrowserOptions` + `MutableCapabilities` |
| Относительные локаторы | Отсутствовали                       | `RelativeLocator.withTagName()`          |

## Ключевые отличия Selenium 4

### Selenium Manager (авто-управление драйверами)

- Встроен начиная с версии 4.6.0
- Автоматически определяет: ОС, архитектуру, версию браузера
- Скачивает и кэширует совместимый драйвер без ручной настройки
- Приоритет: `selenium-manager` > `WebDriverManager` > ручная настройка

### Переход на W3C Actions API

- Унифицированная работа с клавиатурой, мышью, тач-событиями
- Поддержка многопоточных действий: `PointerInput`, `Sequence`
- Пример цепочки действий:

```java
Actions actions = new Actions(driver);
actions.moveToElement(element)
       .click()
       .keyDown(Keys.CONTROL)
       .sendKeys("c")
       .keyUp(Keys.CONTROL)
       .perform();
```

### Относительные локаторы (Selenium 4+)

- Позволяют искать элементы относительно других: `above()`, `below()`, `toLeftOf()`, `near()`
- Упрощают работу с динамическими формами и адаптивной вёрсткой

```java
WebElement usernameField = driver.findElement(By.id("username"));
WebElement passwordField = driver.findElement(
    RelativeLocator.withTagName("input").below(usernameField)
);
```

### Deprecated и удаление API

- `@FindBy` и `PageFactory.initElements()` — помечены как deprecated, рекомендуется ручная инициализация
- `DesiredCapabilities` — заменено на `BrowserOptions` (`ChromeOptions`, `FirefoxOptions`)
- Кастинг к `RemoteWebElement` — больше не требуется для большинства операций

## Поддерживаемые браузеры и версии

| Браузер | Мин. версия  | Драйвер         | Примечание                                 |
|:--------|:------------:|-----------------|--------------------------------------------|
| Chrome  |     75+      | `chromedriver`  | Требуется совместимость с Chromium         |
| Firefox |     78+      | `geckodriver`   | Поддержка ESR-версий                       |
| Edge    |     79+      | `msedgedriver`  | На базе Chromium, совместим с chromedriver |
| Safari  |     13+      | Встроен в macOS | Требуется ручное включение в настройках    |
| Opera   |     60+      | `operadriver`   | Через Chromium-базу                        |

> Для актуальной матрицы совместимости используйте `selenium-manager --browser <name> --browser-version <ver>` 
> или проверяйте [официальную документацию](https://www.selenium.dev/documentation/webdriver/getting_started/).

## Ограничения Selenium WebDriver

- **CAPTCHA**: невозможно обойти программно — требует интеграции с сервисами распознавания или отключения в тестовом окружении
- **Нативные диалоги ОС**: файловые пикеры, принтеры, системные алерты — не контролируются через WebDriver
- **Flash / Java Applets**: устаревшие технологии, не поддерживаются современными браузерами
- **Canvas / SVG**: взаимодействие возможно только через координаты или JS-инъекции, нет семантического доступа к элементам
- **Biometric / 2FA**: требуют мокирования или использования тестовых токенов

## Best Practices

- Начинайте проект с Selenium 4.6+, чтобы использовать Selenium Manager и избежать ручной настройки драйверов
- Проверяйте совместимость версий: браузер ↔ драйвер ↔ Selenium bindings через `selenium-manager --resolve`
- Для кросс-браузерного тестирования используйте W3C-совместимые возможности, избегайте браузер-специфичных хаков
- Документируйте ограничения тестов: какие сценарии требуют моков, какие не могут быть автоматизированы
- При работе с динамическим контентом всегда комбинируйте локаторы с явными ожиданиями (`WebDriverWait`)

---

# 2. Настройка окружения и Управление драйверами

> Правильная конфигурация окружения — фундамент стабильных UI-тестов. 
> 
> В этом разделе рассматриваются зависимости, инициализация WebDriver, управление драйверами и критически важные аргументы запуска для локальной и CI-среды.

## Зависимости в сборщиках

### Maven (pom.xml)

- Используйте `selenium-bom` для централизованного управления версиями всех Selenium-модулей
- Добавляйте `selenium-java` как основную зависимость, `selenium-remote-driver` — для Grid
- Указывайте совместимую версию JDK в `maven-compiler-plugin`

```xml
<project>
    <properties>
        <selenium.version>4.18.1</selenium.version>
        <jdk.version>17</jdk.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.seleniumhq.selenium</groupId>
                <artifactId>selenium-bom</artifactId>
                <version>${selenium.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-remote-driver</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${jdk.version}</source>
                    <target>${jdk.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Gradle (build.gradle / build.gradle.kts)

- Используйте `platform()` для импорта BOM в Gradle
- Разделяйте `implementation` и `testImplementation` для изоляции тестовых зависимостей
- Разрешайте конфликты транзитивных зависимостей через `resolutionStrategy`

```groovy
plugins {
    id 'java'
}

ext {
    seleniumVersion = '4.18.1'
    jdkVersion = 17
}

dependencies {
    // Импорт BOM для управления версиями
    implementation platform("org.seleniumhq.selenium:selenium-bom:${seleniumVersion}")
    
    // Основные зависимости
    testImplementation 'org.seleniumhq.selenium:selenium-java'
    testImplementation 'org.seleniumhq.selenium:selenium-remote-driver'
    
    // Тестовые раннеры
    testImplementation 'org.testng:testng:7.10.0'
}

java {
    sourceCompatibility = jdkVersion
    targetCompatibility = jdkVersion
}

// Разрешение конфликтов версий
configurations.all {
    resolutionStrategy {
        force "org.seleniumhq.selenium:selenium-api:${seleniumVersion}"
    }
}
```

### Совместимость с JDK

| Selenium версия   | Минимальная JDK | Рекомендуемая JDK | Примечание                                   |
|:------------------|-----------------|-------------------|----------------------------------------------|
| 4.0 – 4.10        | 11              | 11, 17            | Полная совместимость                         |
| 4.11+             | 11              | 17, 21            | Оптимизация под виртуальные потоки           |
| 4.15+ (с Records) | 17              | 21                | Использование современных возможностей языка |

> Для проектов на JDK 8 используйте Selenium 3.141.59, но учитывайте отсутствие поддержки W3C и Selenium Manager.

---

## Инициализация WebDriver

### Базовая инициализация драйверов

- Создавайте инстанс драйвера через конструктор: `new ChromeDriver()`, `new FirefoxDriver()`
- Для кросс-браузерности используйте абстракцию `WebDriver` вместо конкретных классов

```java
public class Tset {
    
    // Простая инициализация
    WebDriver driver = new ChromeDriver();
}

// Абстракция для кросс-браузерности
public class DriverFactory {
    
    public static WebDriver create(String browserName) {
        return switch (browserName.toLowerCase()) {
            case "firefox" -> new FirefoxDriver();
            case "edge" -> new EdgeDriver();
            default -> new ChromeDriver();
        };
    }
}
```

### Конфигурация через BrowserOptions

- Используйте `ChromeOptions`, `FirefoxOptions`, `EdgeOptions` вместо устаревших `DesiredCapabilities`
- Настраивайте аргументы, расширения, предпочтения до создания инстанса драйвера

```java
ChromeOptions options = new ChromeOptions();

options.addArguments("--headless=new");
options.addArguments("--disable-gpu", "--no-sandbox");
options.setExperimentalOption("excludeSwitches", new String[]{"enable-automation"});

WebDriver driver = new ChromeDriver(options);
```

### Переменные окружения

- Используйте `System.setProperty()` только для legacy-проектов
- В современных проектах полагайтесь на Selenium Manager или внешнюю конфигурацию

```bash
# Установка путей через переменные окружения (не рекомендуется для Selenium 4.6+)
export CHROME_DRIVER_PATH=/opt/drivers/chromedriver
export GECKO_DRIVER_PATH=/opt/drivers/geckodriver
```

### RemoteWebDriver базовая настройка

- Для подключения к Selenium Grid или облачным провайдерам
- Передавайте `Options` через `MutableCapabilities` для совместимости с W3C

```java
ChromeOptions options = new ChromeOptions();
options.setPlatformName("linux");
options.setBrowserVersion("120");

WebDriver driver = new RemoteWebDriver(
    new URL("http://localhost:4444/wd/hub"),
    options
);
```

---

## Управление драйверами

### Selenium Manager (Selenium 4.6+) — РЕКОМЕНДУЕМЫЙ СПОСОБ

- Встроен в дистрибутив Selenium, не требует дополнительных зависимостей
- Автоматически определяет: ОС, архитектуру, версию браузера, скачивает совместимый драйвер
- Кэширует драйверы в `~/.cache/selenium` (Linux/Mac) или `%USERPROFILE%\.cache\selenium` (Windows)

```java
// Просто создайте драйвер — Selenium Manager сделает всё остальное
public class SimpleTest {
    
    public static void main(String[] args) {
        // Драйвер скачается автоматически при первом запуске
        WebDriver driver = new ChromeDriver();
        driver.get("https://example.com");
        
        // Работа с драйвером...
        
        driver.quit();
    }
}
```

### Проверка и отладка Selenium Manager

```bash
# Проверка доступных версий драйвера
selenium-manager --browser chrome --browser-version 120 --driver

# Просмотр кэшированных драйверов
selenium-manager --list

# Принудительное обновление драйвера
selenium-manager --browser chrome --driver-version 120.0.6099.109 --force
```

### WebDriverManager (Bonigarcia) — Альтернатива для сложных сценариев

- Используйте, если нужна тонкая настройка: прокси, кастомные зеркала, специфичные версии
- Требует отдельной зависимости в проекте

```xml
<dependency>
    <groupId>io.github.bonigarcia</groupId>
    <artifactId>webdrivermanager</artifactId>
    <version>5.8.0</version>
    <scope>test</scope>
</dependency>
```

```java
// Ручная настройка
WebDriverManager.chromedriver()
    .browserVersion("119")
    .driverVersion("119.0.6045.105")
    .proxy("http://proxy:8080")
    .setup();

WebDriver driver = new ChromeDriver();
```

### Конфигурация WebDriverManager через properties

```properties
# webdrivermanager.properties
wdm.chromeDriverVersion=120.0.6099.109
wdm.forceCache=true
wdm.useMirror=https://npm.taobao.org/mirrors/chromedriver
wdm.proxy.http=proxy.company.com:8080
wdm.proxy.https=proxy.company.com:8080
wdm.timeout=30
```

### Ручное управление (Legacy)

- Используйте только для поддержки старых проектов или в изолированных средах без доступа к интернету
- Требует явного указания пути к драйверу через `System.setProperty()`

```java
// Для Selenium < 4.6 или offline-сред
System.setProperty("webdriver.chrome.driver", "/opt/drivers/chromedriver-linux64/chromedriver");
WebDriver driver = new ChromeDriver();
```

---

## Режимы запуска и аргументы

### Headless-режимы

| Браузер | Аргумент         | Версия | Примечание                                        |
|:--------|------------------|:------:|---------------------------------------------------|
| Chrome  | `--headless=new` |  109+  | Новый режим, стабильнее, ближе к headed           |
| Chrome  | `--headless`     |  <109  | Устаревший режим, могут быть отличия в рендеринге |
| Firefox | `--headless`     |  Все   | Единственный режим, стабилен                      |
| Edge    | `--headless=new` |  109+  | Аналогично Chrome                                 |

```java
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless=new"); // Chrome 109+
// options.addArguments("--headless"); // Firefox или старый Chrome
```

### Аргументы производительности и стабильности

- `--disable-gpu` — отключает аппаратное ускорение (критично для Linux CI)

  Отключает аппаратное ускорение видео и графики через видеокарту. 
  Критически важно для Linux-серверов и CI/CD (GitHub Actions, Jenkins), 
  где нет физической видеокарты или установлены урезанные драйверы — без этого флага браузер часто падает при запуске или зависает на пустом экране.
  
- `--no-sandbox` — отключает sandbox (требуется в Docker/root-средах)

  Отключает главный защитный механизм изоляции процессов браузера от операционной системы. 
  Требуется при запуске из-под root-пользователя или внутри Docker-контейнера. 
  Sandbox ограничивает права процессу, но в контейнере права уже ограничены, и эта «двойная изоляция» вызывает ошибку запуска.
  
- `--disable-dev-shm-usage` — решает проблемы с shared memory в контейнерах

  Перенаправляет запись временных файлов вместо директории /dev/shm. 
  В Docker и Kubernetes на эту директорию по умолчанию выделяется очень мало места (64 МБ), чего не хватает для работы современных тяжелых сайтов. 
  Без этого флага браузер будет падать с ошибкой `Out of Memory` или будет зависать вкладка.
  
- `--disable-infobars` — убирает баннер "Chrome is being controlled by automated software"

  Убирает желтую плашку в верхней части окна с текстом «Chrome is being controlled by automated test software». 
  Используется чисто для эстетики и удобства отладки скриншотов, так как эта плашка может перекрывать кнопки сайта и не влияет на реальное детектирование бота.
  
- `--disable-blink-features=AutomationControlled` — обход простых детектов ботов

  Выключает внутренний флаг браузера (`navigator.webdriver = true`), который сообщает сайтам, что управление происходит программно. 
  Это базовый уровень антидетекта, который помогает обойти простые блокировки на сайтах, но не скрывает сложные метрики браузера от профессиональных систем вроде Cloudflare или Datadome.

```java
ChromeOptions options = new ChromeOptions();
options.addArguments(
    "--disable-gpu",
    "--no-sandbox",
    "--disable-dev-shm-usage",
    "--disable-infobars",
    "--window-size=1920,1080",
    "--disable-blink-features=AutomationControlled"
);
```

### Настройка профиля браузера

- `--user-data-dir` — кастомный профиль для изоляции сессий

  Указывает путь к папке, где браузер будет хранить куки, историю и локальные расширения. 
  Позволяет запустить несколько полностью независимых сессий браузера одновременно без конфликта данных.

- `--load-extension` — подключение расширений (`.crx` для Chrome, `.xpi` для Firefox)

  Загружает указанное расширение (распакованную папку или файл `.crx` / `.xpi`) прямо при старте сессии. 
  Основной способ добавить функциональность в headless-режиме, так как открыть магазин расширений и нажать «Установить» через GUI нельзя.

- `--allow-insecure-localhost` — доверие самоподписанным сертификатам

  Разрешает заходить на https://localhost или 127.0.0.1, даже если SSL-сертификат сайта просрочен или самоподписанный. 
  Избавляет от ошибки «Ваше соединение не защищено» при локальной разработке и тестировании.

```java
// Настройка профиля с расширением и кастомным user-data-dir
ChromeOptions options = new ChromeOptions();
options.addArguments("--user-data-dir=/tmp/chrome-profile-test");
options.addExtensions(new File("src/test/resources/extensions/auth-helper.crx"));
options.setAcceptInsecureCerts(true);
```

### Эмуляция мобильного устройства

- Используйте `mobileEmulation` для тестирования адаптивных интерфейсов
- Поддержка предустановленных устройств и кастомных параметров

```java
ChromeOptions options = new ChromeOptions();
Map<String, String> mobileEmulation = new HashMap<>();
mobileEmulation.put("deviceName", "iPhone 14 Pro");

// Или кастомные параметры
// mobileEmulation.put("width", "414");
// mobileEmulation.put("height", "896");
// mobileEmulation.put("pixelRatio", "3");

options.setExperimentalOption("mobileEmulation", mobileEmulation);
options.addArguments("--user-agent=Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)...");
```

---

## Capabilities vs Options

### Миграция с DesiredCapabilities на BrowserOptions

- `DesiredCapabilities` помечен как deprecated в Selenium 4
- Используйте типобезопасные `ChromeOptions`, `FirefoxOptions` и т.д.

```java
// ❌ Устаревший подход (Selenium 3)
DesiredCapabilities caps = new DesiredCapabilities();
caps.setBrowserName("chrome");
caps.setCapability("acceptInsecureCerts", true);
WebDriver driver = new RemoteWebDriver(new URL("..."), caps);

// ✅ Современный подход (Selenium 4)
ChromeOptions options = new ChromeOptions();
options.setAcceptInsecureCerts(true); // разрешает браузеру принимать недействительные (самоподписанные, просроченные) SSL-сертификаты на HTTPS-сайтах без показа предупреждения об ошибке безопасности.
options.addArguments("--headless=new");
WebDriver driver = new ChromeDriver(options);
```

### Объединение capabilities для Grid

- При работе с RemoteWebDriver используйте `merge()` для комбинирования настроек
- Платформенные capabilities передавайте отдельно для матчинга с нодами

```java
ChromeOptions browserOptions = new ChromeOptions();
browserOptions.addArguments("--headless=new");

MutableCapabilities platformCaps = new MutableCapabilities();
platformCaps.setCapability("platformName", "linux");
platformCaps.setCapability("browserVersion", "120");
platformCaps.setCapability("se:build", "CI-2024-Q1");

// Объединение
browserOptions.merge(platformCaps);

WebDriver driver = new RemoteWebDriver(
    new URL("http://grid:4444/wd/hub"),
    browserOptions
);
```

### Ключевые платформенные capabilities

| Capability            | Тип     | Описание                               | Пример                          |
|:----------------------|:--------|:---------------------------------------|:--------------------------------|
| `platformName`        | String  | ОС для матчинга с нодой                | `"linux"`, `"windows"`          |
| `browserVersion`      | String  | Версия браузера                        | `"120"`, `"119.0"`              |
| `acceptInsecureCerts` | Boolean | Доверие к самоподписанным сертификатам | `true`                          |
| `pageLoadStrategy`    | String  | Стратегия загрузки страницы            | `"normal"`, `"eager"`, `"none"` |
| `se:name`             | String  | Имя теста для отслеживания в Grid      | `"LoginTest"`                   |
| `se:build`            | String  | Сборка/версия проекта                  | `"release-2.1.0"`               |

```java
ChromeOptions options = new ChromeOptions();
options.setCapability("pageLoadStrategy", "eager"); // Ускорение за счёт пропуска ресурсов
options.setCapability("se:name", "E2E_CheckoutFlow");
options.setCapability("se:build", "2024.03.15");
```

## Best Practices

- Всегда используйте Selenium Manager для проектов на Selenium 4.6+ — это устраняет 90% проблем с драйверами
- Выносите конфигурацию браузера в отдельные классы (`BrowserConfig`, `DriverFactory`) или внешние файлы (YAML/properties)
- Никогда не смешивайте `ImplicitWait` и `ExplicitWait` на одном инстансе драйвера — это приводит к непредсказуемым таймаутам
- В CI/CD всегда запускайте браузер в режиме `--headless=new` с аргументами `--disable-gpu --no-sandbox --disable-dev-shm-usage`
- Используйте `BrowserOptions` вместо `DesiredCapabilities`; оставляйте `Capabilities` только для платформенного матчинга в Grid
- Для локальной отладки временно отключайте headless-режим и добавляйте `--remote-debugging-port=9222` для подключения DevTools
- Кэшируйте зависимости и драйверы в CI (GitHub Actions cache, GitLab CI cache) для ускорения пайплайнов
- Документируйте минимальные версии браузера и драйверов в `README.md` или `ENVIRONMENT.md`

---

# 3. Локаторы и Поиск элементов

> Локаторы — фундамент взаимодействия с веб-интерфейсом. Грамотный выбор стратегии поиска определяет стабильность, скорость выполнения и поддерживаемость автотестов в долгосрочной перспективе.

### Базовые стратегии поиска

- `By.id` — самый быстрый и надежный способ. Требует уникального `id` в DOM.
- `By.name` — оптимален для форм и полей ввода, но не гарантирует уникальность.
- `By.className` — работает только с одним классом. Составные классы вызовут `InvalidSelectorException`.
- `By.tagName` — поиск по HTML-тегу, полезен для массового выбора.
- `By.linkText` / `By.partialLinkText` — поиск по тексту ссылки `<a>`.

```java
// Уникальный идентификатор (приоритет №1)
WebElement loginInput = driver.findElement(By.id("username"));

// Поиск по атрибуту name
WebElement emailField = driver.findElement(By.name("user-email"));

// Поиск ссылки по тексту
WebElement logoutLink = driver.findElement(By.linkText("Выйти из системы"));
```

### XPath и CSS Selectors

- **XPath**: мощный язык запросов, навигация по DOM-дереву, оси, поиск по тексту.
- **CSS Selectors**: быстрее парсятся браузером, лаконичный синтаксис, не умеют искать по тексту напрямую.
- Всегда используйте относительные пути, избегайте абсолютных.

```java
// XPath примеры
WebElement btn = driver.findElement(By.xpath("//button[contains(@class, 'submit')]"));
WebElement label = driver.findElement(By.xpath("//label[normalize-space()='Email']/preceding-sibling::input"));

// CSS примеры
WebElement input = driver.findElement(By.cssSelector("input[name='password'][required]"));
WebElement item = driver.findElement(By.cssSelector("ul.menu > li:nth-child(3)"));
```

### Относительные локаторы (Selenium 4)

- Позволяют находить элементы относительно других видимых элементов.
- Основаны на визуальном расположении, зависят от CSS-рендеринга, а не структуры DOM.

- `below` / `above` — поиск по вертикали (Y-координата bounding-бокса)
- `toLeftOf` / `toRightOf` — поиск по горизонтали (X-координата bounding-бокса)
- `near` — поиск в радиусе 50px от центра элемента-ориентира

```java
import org.openqa.selenium.support.locators.RelativeLocator;

WebElement title = driver.findElement(By.tagName("h2"));
WebElement emailInput = driver.findElement(
    RelativeLocator.with(By.tagName("input")).below(title)
);
WebElement submitBtn = driver.findElement(
    RelativeLocator.with(By.tagName("button")).toRightOf(emailInput)
);
```

### Поиск в Shadow DOM

> Shadow DOM — это изолированный фрагмент DOM-дерева внутри веб-компонента, который браузер скрывает от стандартных поисковых методов. 
>
> Обычный `driver.findElement()` не видит элементы внутри такого компонента — они живут в отдельной "тени" (shadow tree). 
> 
> Это нативный механизм инкапсуляции в современных веб-приложениях (React, Angular, Lit, Vue с Custom Elements).

Два режима Shadow DOM:

- `open` — корень тени доступен через `WebElement.getShadowRoot()`. Можно автоматизировать стандартными средствами.
- `closed` — корень тени намеренно скрыт разработчиком, API возвращает null. 
  
  Требует инжекта JavaScript для обхода (пробивать через `executeScript()`).


```java
// 1. Находим хост-элемент — обычный элемент, к которому прикреплен Shadow DOM
WebElement shadowHost = driver.findElement(By.cssSelector("custom-web-component"));

// 2. Получаем корень тени. Работает ТОЛЬКО если тень открытая (mode: 'open')
SearchContext shadowRoot = shadowHost.getShadowRoot();

// 3. Внутри тени ищем элементы как обычно, но контекстом поиска выступает shadowRoot
WebElement innerButton = shadowRoot.findElement(By.cssSelector("button.submit"));
```

Когда встречается на практике:

- Сложные UI-библиотеки (например, vaadin, material-web, shoelace).
- Страницы с `<iframe>` или встроенными виджетами платежных систем.
- Приложения на современных фреймворках, активно использующих Custom Elements.

### Коллекции и фильтры

- `findElement()` — возвращает первый элемент или бросает `NoSuchElementException`.
- `findElements()` — возвращает `List<WebElement>`, безопаснее для проверок наличия. 

  Если элемент отсутствует - возвращает **пустой** список.

```java
List<WebElement> allRows = driver.findElements(By.cssSelector("table tbody tr"));
List<WebElement> enabledRows = allRows.stream()
    .filter(WebElement::isEnabled)
    .collect(Collectors.toList());
```

## Best Practices
- Приоритет локаторов: `id` > `name` > `data-testid` > `CSS` > `XPath`
- Избегать абсолютных XPath и привязки к индексу `[1]`
- Использовать кастомные атрибуты (`data-qa`, `data-testid`) для стабильности
- При динамических списках искать по видимым/активным элементам
- Логировать локатор в исключениях для быстрого дебага
- Не кешировать `WebElement` в полях классов на долгий срок, использовать локаторы или перепоиск

---

# 4. Взаимодействие с элементами (WebElement & Actions API)

> Взаимодействие с элементами — это выполнение действий: клик, ввод текста, навигация, drag&drop. 
> 
> Selenium 4 полностью перешел на W3C Actions API, что унифицирует работу с клавиатурой, мышью и тач-устройствами.

### Базовые методы WebElement

- `click()` — стандартный клик левой кнопкой мыши.

  Требует, чтобы элемент был видим (`isDisplayed() = true`) и не перекрывался другими элементами (Overlay/Popup).

  При перекрытии выбрасывает `ElementClickInterceptedException`.

- `sendKeys(CharSequence... keysToSend)` — ввод текста или спецсимволов в поле.

  Имитирует нажатия клавиш. Можно передавать `Keys.ENTER`, `Keys.TAB`, `Keys.CONTROL + "a"` для комбинаций.

  Не очищает поле перед вводом (используйте вместе с `clear()` или `Ctrl+A`).

- `clear()` — очистка текстового поля.

  Работает через эмуляцию выделения всего текста (`Ctrl+A`) и нажатия `Backspace`.

  В современных SPA-фреймворках (React, Vue) часто нестабилен. Альтернатива: `sendKeys(Keys.CONTROL + "a") + sendKeys(Keys.DELETE)`.

- `submit()` — отправка формы.

  Работает только если элемент находится внутри тега `<form>`. Эквивалент нажатия `Enter` в поле формы.

  Если элемента нет в форме — выбрасывает `NoSuchElementException`.

- `getText()` — получение видимого текста элемента.

  Возвращает текст, который пользователь видит на экране (учитывает CSS свойства `visibility: hidden` и `display: none` — текст не вернется).

  Для скрытых элементов используйте `getAttribute("textContent")`.

- `getAttribute(String name)` — возвращает значение атрибута из исходного HTML.

  Возвращает то, что было при загрузке страницы (например `value`, `class`, `href`, `placeholder`).

  Если атрибут изменился через JavaScript — значение может не совпадать с текущим.

- `getDomProperty(String name)` — возвращает текущее значение свойства DOM-объекта в памяти браузера.

  Например, `getDomProperty("value")` вернет актуальный текст в поле ввода, даже если атрибут `value` в HTML не менялся.

  Предпочтительнее `getAttribute` для проверки состояния полей форм. 

- `getDomAttribute(String name)` — возвращает текущее значение атрибута DOM.

  В отличие от `getAttribute`, который может дать значение из HTML или свойства, этот метод имеет более предсказуемое поведение.

  Особенно полезен для проверки атрибутов типа `checked`, `disabled`, `selected`. 

- `getCssValue(String propertyName)` — возвращает вычисленное значение CSS-свойства.

  Например `"color"`, `"font-size"`, `"display"`.

  Возвращает строку в формате браузера (`"rgba(0, 0, 0, 1)"`, `"16px"`).

- `isDisplayed()` — проверяет, видим ли элемент на странице.

  Возвращает `false` если элемент скрыт через CSS (`display: none`, `visibility: hidden`) или имеет нулевые размеры (`width/height = 0`).

  Не проверяет, перекрыт ли элемент другим.

- `isEnabled()` — проверяет, активен ли элемент для взаимодействия.

  Возвращает `false` для полей с атрибутом `disabled` или `readonly` (зависит от типа элемента).

  Кнопки с `disabled` не кликабельны.

- `isSelected()` — проверяет состояние "выбранности".

  Применимо к `checkbox`, `radio buttons`, `option` в `select`.

  Для `checkbox`/`radio` возвращает `true` если отмечен галочкой.

- `getLocation()` — возвращает объект `Point` с координатами `x`, `y` левого верхнего угла элемента относительно страницы (viewport).

  Полезно для отладки перекрытий и проверки позиционирования.

- `getSize()` — возвращает объект `Dimension` с `width` и `height` элемента в пикселях.

  Используется в связке с `getLocation()` для проверки кликабельности области.

- `getRect()` — возвращает объект `Rectangle`, объединяющий `x`, `y`, `width`, `height`.

  Более удобный метод чем `getLocation()` + `getSize()` по отдельности.

- `getTagName()` — возвращает имя HTML-тега элемента в нижнем регистре.

  Например `"input"`, `"div"`, `"button"`.

  Полезно для валидации типа найденного элемента.

- `findElement(By by)` — поиск первого дочернего элемента внутри текущего элемента по заданному локатору.

  Контекст поиска ограничен текущим элементом, не ищет по всей странице.

- `findElements(By by)` — поиск всех дочерних элементов внутри текущего элемента по заданному локатору.

  Возвращает пустой список, если ничего не найдено (не выбрасывает исключение).

- `getShadowRoot()` — возвращает `SearchContext` для работы с открытым Shadow DOM текущего элемента.

  Возвращает `null` если shadow root закрытый (`mode: closed`) или отсутствует.

- `getAccessibleName()` — возвращает вычисленное accessible name элемента (для скринридеров).

  Полезно для тестирования accessibility (a11y).

- `getAriaRole()` — возвращает вычисленную ARIA-роль элемента.

  Например `"button"`, `"checkbox"`, `"textbox"`.

  Также используется для accessibility-тестирования. 

- `takeScreenshot()` — делает скриншот только этого элемента (не всей страницы).

  Требует, чтобы драйвер поддерживал скриншоты (`WebDriver`, реализующий `TakesScreenshot`).

  Возвращает `byte[]`.

- `clear()` (альтернатива для SPA) — обходной путь для нестабильного `clear()` в современных фреймворках.
  ```java
  element.sendKeys(Keys.chord(Keys.CONTROL, "a")); // выделить всё
  element.sendKeys(Keys.DELETE); // удалить
  element.sendKeys("новый текст"); // ввести новое
  ```

---

```java
WebElement searchInput = driver.findElement(By.id("search"));
searchInput.sendKeys("Selenium WebDriver");
searchInput.submit();

String htmlValue = searchInput.getAttribute("value");
String currentDomValue = searchInput.getDomProperty("value");
boolean isVisible = searchInput.isDisplayed();
```

### Специфичные контролы

- `Select` — работа со стандартными `<select>` элементами. Не работает с кастомными дропдаунами.
- Checkbox/Radio — требуют проверки `isSelected()` перед кликом. Клик по `<label>` часто надежнее.
- DatePickers, Sliders, Canvas — требуют координатного взаимодействия или JS-инъекций.

```java
WebElement selectElement = driver.findElement(By.id("country"));
Select dropdown = new Select(selectElement);

// Выбор по значению, тексту или индексу
dropdown.selectByValue("US");
dropdown.selectByVisibleText("США");
dropdown.selectByIndex(2);

// Проверка мультиселекта
if (dropdown.isMultiple()) {
    dropdown.deselectAll();
}
```

### W3C Actions API (Selenium 4)

- `Actions` builder позволяет строить сложные цепочки: перемещение мыши, зажатие клавиш, drag&drop, контекстные клики.
- `perform()` автоматически выполняет `build()`, поэтому `build().perform()` избыточно.
- Координатное взаимодействие: `moveByOffset()`, `clickAndHold()`, `release()`.

```java
Actions actions = new Actions(driver);
WebElement source = driver.findElement(By.id("drag-source"));
WebElement target = driver.findElement(By.id("drop-target"));

// Drag and Drop
actions.dragAndDrop(source, target).perform();

// Hover + Context Click
actions.moveToElement(source)
       .pause(Duration.ofMillis(500))
       .contextClick()
       .perform();

// Зажатие клавиши, ввод, отпускание
actions.keyDown(Keys.SHIFT)
       .sendKeys("selenium")
       .keyUp(Keys.SHIFT)
       .perform();
```

### Клавиатурные комбинации и спецклавиши

- Enum `Keys` содержит все спецклавиши: `ENTER`, `TAB`, `ARROW_DOWN`, `ESCAPE`, `BACK_SPACE`.
- `Key.chord()` позволяет передавать комбинации в `sendKeys()`.
- `Keys.NULL` используется для сброса состояния зажатых модификаторов.
- Ввод без фокуса: отправка клавиш в `body` или родительский контейнер.

```java
WebElement input = driver.findElement(By.name("email"));
input.sendKeys(Keys.CONTROL + "a");
input.sendKeys(Keys.DELETE);
input.sendKeys("test@example.com");
input.sendKeys(Keys.TAB, Keys.NULL); // Сброс модификаторов

// Комбинация для выделения всего текста
input.sendKeys(Keys.chord(Keys.CONTROL, "a"));
```

### Тач и мультитач (мобильный контекст)

- Для эмуляции тач-событий используются `PointerInput` и `Sequence`.
- Поддержка жестов: pinch, zoom, swipe, long press.
- В десктопных браузерах работает ограниченно, требует эмуляции мобильного устройства или Appium.

```java
PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
Sequence swipe = new Sequence(finger, 0);

// Эмуляция свайпа
swipe.addAction(finger.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), 500, 1000));
swipe.addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()));
swipe.addAction(finger.createPointerMove(Duration.ofMillis(200), PointerInput.Origin.viewport(), 500, 400));
swipe.addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

driver.perform(Collections.singletonList(swipe));
```

## Best Practices

- Всегда проверяйте `isDisplayed()` или используйте `ExpectedConditions.elementToBeClickable()` перед `click()`

  ```java
  WebElement button3 = wait.until(
  ExpectedConditions.elementToBeClickable(By.id("submitBtn"))
  );
  button3.click(); // гарантированно видима, активна и не перекрыта
  ```
  
- Для отправки текста предпочитать `clear()` + `sendKeys()` или `Keys.CONTROL + "a" + Keys.DELETE` для надежности

  ```java
  input.clear();
  input.sendKeys("новое_значение");
  
  // Или
  
  input.sendKeys(Keys.chord(Keys.CONTROL, "a", Keys.DELETE));
  input.sendKeys("новое_значение");
  ```
- Избегать `Thread.sleep()` для синхронизации анимаций, использовать `Actions` с явными ожиданиями

  ```java
  // Ожидание конкретного состояния после анимации
  actions.moveToElement(menu).perform();
        
  WebElement visibleItem = wait.until(
        ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".submenu-item"))
  );
  visibleItem.click();
  ```
  
- При работе с `Select` проверять тип элемента `<select>`, для кастомных дропдаунов использовать обычные локаторы
- Логировать координаты и целевые элементы при сложных Actions-цепочках
- Использовать `getDomProperty()` вместо `getAttribute()` для получения текущего значения полей ввода в SPA
- Для стабильного drag&drop добавлять `pause()` между шагами, чтобы браузер успел обработать события

  ```java
  // drag&drop с паузами между ключевыми шагами
  actions.clickAndHold(source)                    // 1. Захватили элемент (dragStart)
         .pause(Duration.ofMillis(300))           // 2. Ждем обработки dragStart браузером
         .moveToElement(target)                   // 3. Перемещаем к цели (dragOver)
         .pause(Duration.ofMillis(300))           // 4. Ждем обработки dragover и пересчета координат
         .release()                               // 5. Отпускаем (drop)
         .perform();                              // 6. Выполняем всё
  ```
- В CI/CD тестировать тач-жесты только через эмуляцию или мобильные драйверы, десктопные браузеры не поддерживают полноценный Multi-Touch

---

# 5. Синхронизация и Ожидания (Waits)

> Синхронизация — критически важный аспект UI-автоматизации. 
> 
> Неправильное использование ожиданий приводит к нестабильным ("flaky") тестам. 
> 
> Selenium предоставляет три механизма: Implicit, Explicit и Fluent Waits, каждый из которых решает свои задачи.

### Implicit Wait (Неявное ожидание)

- Глобальная настройка на уровне драйвера: `driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10))`
- Как работает: драйвер непрерывно опрашивает (polling) DOM с заданной частотой до появления элемента или истечения таймаута
- Почему не рекомендуется: конфликтует с Explicit Waits, скрывает реальные проблемы синхронизации, применяется ко всем `findElement()` без учета контекста
- В Selenium 4 рекомендуется использовать `implicitlyWait` только как fallback или отключать полностью

```java
// Установка неявного ожидания (глобально для всей сессии)
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(5));

// Сброс до нуля (рекомендуется перед явными ожиданиями)
driver.manage().timeouts().implicitlyWait(Duration.ZERO);
```

### Explicit Wait (Явное ожидание)

- `WebDriverWait` в связке с `ExpectedConditions` — золотой стандарт синхронизации
- Базовые условия: `elementToBeClickable`, `visibilityOfElementLocated`, `presenceOfElementLocated`, `textToBePresentInElement`
- Настраиваемый таймаут и pollingInterval: `withTimeout()`, `pollingEvery()`
- Поддержка кастомных условий через лямбды или реализацию `ExpectedCondition<T>`

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));
wait.pollingEvery(Duration.ofMillis(500));

// Ожидание кликабельности элемента
WebElement submitBtn = wait.until(
    ExpectedConditions.elementToBeClickable(By.cssSelector("button[type='submit']"))
);

// Ожидание появления текста в элементе
wait.until(ExpectedConditions.textToBePresentInElement(
    By.id("status-message"), "Успешно сохранено"
));

// Кастомное условие через лямбду
WebElement dynamicElement = wait.until(driver -> {
    WebElement el = driver.findElement(By.id("dynamic-data"));
    return el.isDisplayed() && !"Loading...".equals(el.getText()) ? el : null;
});

/*
Интерфейс ExpectedCondition<T> имеет метод apply(WebDriver driver), который:
- Возвращает null для обозначения "условие не выполнено"
- Возвращает любой non-null объект (включая WebElement, Boolean, String и т.д.) для обозначения "условие выполнено"
 */
```

> `findElement()` внутри лямбды выбросит исключение NoSuchElementException, если элемента нет в DOM. 
> 
> WebDriverWait автоматически перехватит его и продолжит ожидание (считая, что условие не выполнено).

### FluentWait

- Расширенная версия Explicit Wait с тонкой настройкой
- Игнорирование исключений: `ignoring(StaleElementReferenceException.class, NoSuchElementException.class)`
- Кастомное сообщение при таймауте: `withMessage()`
- Сценарии использования: асинхронная загрузка данных, динамические таблицы, SPA-роутинг

```java
Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(20))
    .pollingEvery(Duration.ofMillis(1000))
    .ignoring(StaleElementReferenceException.class)
    .ignoring(NoSuchElementException.class)
    .withMessage("Таблица данных не загрузилась за отведенное время");

WebElement tableRow = fluentWait.until(driver -> {
    WebElement row = driver.findElement(By.cssSelector("table tr.active"));
    return row.isDisplayed() ? row : null;
});
```

### Ожидание специфичных состояний

- URL: `urlContains`, `urlMatches`, `urlToBe` — критично после редиректов и логина
- Title: `titleContains`, `titleIs` — проверка загрузки страницы
- Alert/Frame/Window: `alertIsPresent`, `frameToBeAvailableAndSwitchToIt`, `numberOfWindowsToBe`
- Ожидание завершения JS/AJAX: polling `document.readyState`, `ExpectedConditions.jsReturnsValue()`

```java
// Ожидание перехода на конкретный URL
wait.until(ExpectedConditions.urlContains("/dashboard"));

// Ожидание и переключение во фрейм
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(By.id("payment-frame")));

// Ожидание готовности AJAX (кастомное условие)
wait.until(driver -> 
    ((JavascriptExecutor) driver).executeScript("return document.readyState").equals("complete")
);
```

### Отладка и типичные ошибки

- `TimeoutException`: условие не выполнилось за отведенное время. Проверьте локатор, наличие элемента в DOM, CSS-анимации, overlay-элементы.
- `StaleElementReferenceException`: DOM обновился после получения ссылки на элемент. Решение: перепоиск локатора перед действием.
- `ElementNotInteractableException`: элемент присутствует, но невидим или перекрыт. Используйте `scrollIntoView`, `Actions.moveToElement()`, или явное ожидание видимости.
- Логирование polling-попыток: включайте `LogLevel.FINE` для `RemoteWebDriver` или используйте кастомные `Wait` с callback-логированием.

```java
// Пример отладки с логированием каждой попытки polling
Wait<WebDriver> debugWait = new FluentWait<WebDriver>(driver) {
    private int attempt = 0;
    
    // Анонимный класс может иметь только instance initializer
    {
        withTimeout(Duration.ofSeconds(10));                                // настройка таймаута
        pollingEvery(Duration.ofSeconds(1));                                // настройка интервала опроса
        withMessage("Элемент не найден после " + getTimeout() + " секунд"); // кастомное сообщение об ошибке
    }

    @Override
    public <V> V apply(Function<? super WebDriver, V> isTrue) {
        attempt++;
        System.out.println("[DEBUG] Poll attempt #" + attempt);
        return super.apply(isTrue);
    }
}.until(ExpectedConditions.presenceOfElementLocated(By.id("dynamic-content")));
```

## Best Practices

- Никогда не смешивать Implicit и Explicit Waits в одном тесте — это приводит к непредсказуемым таймаутам и замедлению выполнения
- Использовать `WebDriverWait` с минимальным адекватным polling (500ms–1s) для баланса между скоростью и нагрузкой на браузер
- Выносить wait-логику в утилитарные методы (`waitForElementVisible`, `waitForAjaxComplete`, `waitForUrlContains`)
- При возникновении `StaleElement` всегда ре-инициализировать локатор, а не кешировать `WebElement`

  ```java
  // Сохраняем локатор, а не элемент
  By statusLocator = By.id("status");

  // Каждый раз получаем свежий элемент
  String text = driver.findElement(statusLocator).getText();
  ```
  
- Документировать критические ожидания в комментариях или Allure-шагах для упрощения поддержки
- Избегать `Thread.sleep()` как основного механизма синхронизации — использовать только для отладки специфичных race-conditions
- При работе с SPA и фреймворками (React, Angular, Vue) ожидать стабилизацию фреймворка, а не просто появление элемента в DOM

## 6. Управление контекстом браузера

> Управление контекстом включает работу с несколькими окнами/вкладками, фреймами, всплывающими диалогами, хранилищами данных и механизмами загрузки файлов. 
> 
> Корректное переключение между контекстами — обязательное условие стабильной работы сложных сценариев.

### Окна и вкладки

- `driver.getWindowHandle()` — возвращает уникальный идентификатор текущего окна
- `driver.getWindowHandles()` — возвращает `Set<String>` всех открытых окон/вкладок
- Переключение: `driver.switchTo().window(handle)` — переводит фокус драйвера на указанное окно
- Открытие новой вкладки: через JS `window.open()` или комбинацию клавиш `Ctrl+Enter`
- Ожидание появления новой вкладки: проверка изменения размера `getWindowHandles().size()`

```java
String originalWindow = driver.getWindowHandle();

// Открытие новой вкладки через JavaScript
((JavascriptExecutor) driver).executeScript("window.open('https://example.com', '_blank');");

// Ожидание появления второго окна
wait.until(driver -> driver.getWindowHandles().size() == 2);

// Переключение на новую вкладку
Set<String> allWindows = driver.getWindowHandles();

for (String handle : allWindows) {
    if (!handle.equals(originalWindow)) {
        driver.switchTo().window(handle);
        break;
    }
}

// Закрытие текущей вкладки и возврат
driver.close();
driver.switchTo().window(originalWindow);
```

### Frames и Iframes

- Страница может содержать встроенные документы (`<iframe>`, `<frame>`). Для взаимодействия нужно переключить контекст драйвера внутрь фрейма.
- Переключение по: индексу (`0`, `1`...), атрибуту `name`/`id`, или найденному `WebElement`.
- Выход из фрейма: `switchTo().defaultContent()` (выход в корень страницы) или `switchTo().parentFrame()` (на уровень выше).
- Вложенные фреймы требуют последовательного переключения и явных ожиданий доступности.

```java
// Переключение по WebElement с ожиданием доступности
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(By.id("payment-frame")));

// Действия внутри фрейма
WebElement cardInput = driver.findElement(By.name("cardNumber"));
cardInput.sendKeys("4111111111111111");

// Выход из вложенного фрейма на один уровень вверх
driver.switchTo().parentFrame();

// Полный выход в основную страницу
driver.switchTo().defaultContent();
```

### Alerts, Prompts, Confirmations

- Нативные браузерные диалоги (`alert`, `confirm`, `prompt`) блокируют выполнение JS до ответа пользователя.
- Selenium предоставляет интерфейс `Alert` для взаимодействия: `accept()`, `dismiss()`, `getText()`, `sendKeys()`.
- Всегда используйте `ExpectedConditions.alertIsPresent()` перед переключением, так как диалог может появиться с задержкой.
- Системные диалоги ОС (выбор файла, печать, сохранение пароля) НЕ обрабатываются через Selenium.

```java
// Нажатие кнопки, вызывающей alert
driver.findElement(By.id("show-alert")).click();

// Ожидание и получение интерфейса алерта
Alert alert = wait.until(ExpectedConditions.alertIsPresent());
String alertText = alert.getText();

// Для prompt-окон можно ввести текст
if (alertText.contains("Введите значение:")) {
    alert.sendKeys("Тестовое значение");
}

// Подтверждение или отмена
alert.accept(); // или alert.dismiss();
```

### Cookies и Storage

- Класс `org.openqa.selenium.Cookie` позволяет управлять HTTP-куками: имя, значение, домен, путь, срок жизни, флаги `secure`/`httpOnly`.
- Методы управления: `getCookies()`, `getCookieNamed()`, `addCookie()`, `deleteCookie()`, `deleteAllCookies()`.
- `LocalStorage` и `SessionStorage` не имеют прямого API в Selenium. Работа осуществляется через `JavascriptExecutor`.
- Для корректной работы cookie домен страницы должен соответствовать домену, указанному в cookie (или `driver.get()` должен быть выполнен заранее).

#### Cookies

> Небольшие фрагменты данных (до 4 КБ), которые сервер отправляет браузеру, а браузер автоматически прикрепляет их к каждому следующему запросу к этому серверу.

Для чего используется: 

- Аутентификация — хранить `session ID`, чтобы пользователь не вводил логин/пароль на каждой странице
- Отслеживание — аналитика, рекламные идентификаторы
- Запоминание настроек — язык, регион, валюта
- Корзина интернет-магазина (устаревающий сценарий)

Ключевые особенности:

- Автоматически отправляются на сервер при каждом запросе
- Имеют срок жизни (expires/max-age)
- Ограничены доменом и путем
- Могут иметь флаги `Secure` (только по HTTPS) и `HttpOnly` (недоступны из JavaScript — защита от XSS)

```java
// Добавление куки

// 1. Переходим на сайт (обязательно!)
driver.get("https://example.com");

// 2. Создаем куку для авторизации
Cookie authCookie = new Cookie.Builder("session_token", "xyz789")
        .domain("example.com")
        .path("/dashboard")  // кука работает только на /dashboard/*
        .expiresOn(Instant.now().plusSeconds(7200)) // через 2 часа
        .isSecure(true)
        .isHttpOnly(true)
        .build();

driver.manage().addCookie(authCookie);

// 3. Обновляем страницу — кука улетит на сервер
driver.navigate().refresh();

// 4. Проверяем, что кука установилась
Cookie cookieAfter = driver.manage().getCookieNamed("session_token");
System.out.println("Значение: " + cookieAfter.getValue());

// 5. Чистим после теста
driver.manage().deleteCookieNamed("session_token");
```

#### Storage

Для чего используется:

- Хранение тем/настроек — "темная тема", размер шрифта
- Оффлайн-режим — кэширование данных для работы без интернета
- Состояние корзины (современный подход вместо кук)
- JWT-токены (хотя есть споры по безопасности)
- Черновики форм — временное сохранение ввода пользователя

Ключевые особенности:

- Не отправляются на сервер (уменьшает трафик)
- Большой объем (до 10 МБ)
- Только строки (но можно хранить JSON через `JSON.stringify`)
- Доступны через JavaScript (не защищены от XSS)

```java
// LocalStorage

// Устанавливаем значение
((JavascriptExecutor) driver)
    .executeScript("localStorage.setItem('user_theme', 'dark');");

// Читаем значение
String theme = (String) ((JavascriptExecutor) driver)
    .executeScript("return localStorage.getItem('user_theme');");
System.out.println("Тема: " + theme); // dark

// Сохраняем объект (через JSON)
((JavascriptExecutor) driver).executeScript(
    "localStorage.setItem('user_prefs', JSON.stringify({lang: 'ru', fontSize: 16}));"
);

// Получаем объект обратно
String prefsJson = (String) ((JavascriptExecutor) driver)
    .executeScript("return localStorage.getItem('user_prefs');");

// Удаляем конкретный ключ
((JavascriptExecutor) driver)
    .executeScript("localStorage.removeItem('user_theme');");

// Очищаем всё хранилище
((JavascriptExecutor) driver)
    .executeScript("localStorage.clear();");


// SessionStorage

// Сохраняем данные для текущей вкладки
((JavascriptExecutor) driver)
    .executeScript("sessionStorage.setItem('step_in_wizard', '3');");

// Читаем
String step = (String) ((JavascriptExecutor) driver)
    .executeScript("return sessionStorage.getItem('step_in_wizard');");

// После закрытия вкладки или браузера — эти данные исчезнут
```

Сравнительная таблица:

| Критерий                   | Cookies                                        | LocalStorage                                | SessionStorage                                  |
|----------------------------|------------------------------------------------|---------------------------------------------|-------------------------------------------------|
| Объем                      | ~4 КБ                                          | ~5-10 МБ                                    | ~5-10 МБ                                        |
| Отправка на сервер         | ✅ Да (автоматически)                           | ❌ Нет                                       | ❌ Нет                                           |
| Срок жизни                 | Задается сервером (expires/max-age)            | Пока не удалить вручную                     | До закрытия вкладки/окна                        |
| Доступ из JavaScript       | Только если нет флага HttpOnly                 | ✅ Да                                        | ✅ Да                                            |
| Поддержка во всех вкладках | Да (в пределах домена)                         | Да (в пределах источника)                   | Нет (только текущая вкладка)                    |
| Защита от XSS-атак         | Через флаг HttpOnly                            | Нет (уязвим)                                | Нет (уязвим)                                    |
| Автоматическое удаление    | По истечении срока или вручную                 | Только вручную или через код                | При закрытии вкладки                            |
| Использование в Selenium   | Прямой API (driver.manage().addCookie() и др.) | Через JavascriptExecutor                    | Через JavascriptExecutor                        |
| Типичные сценарии          | Сессии, трекинг, языковые настройки            | Темы оформления, оффлайн-данные, JWT-токены | Временные данные формы, состояние шагов мастера |

### Загрузка файлов

- Для стандартного `<input type="file">` достаточно отправить абсолютный путь к файлу через `sendKeys()`.
  ```java
  WebElement fileInput = driver.findElement(By.id("avatar"));
  fileInput.sendKeys("C:/Users/user/Pictures/photo.jpg");
  driver.findElement(By.id("submit")).click(); // отправка формы
  ```

  Selenium эмулирует ввод текста в поле ввода. Поскольку `input[type="file"]` принимает путь к файлу как свое значение, браузер автоматически загружает этот файл.

- В headless-режиме и при работе с `RemoteWebDriver` браузер может находиться на другой машине. Требуется передача файла на удаленный узел через `FileDetector`.

  ```java
  // 1. Приводим driver к типу RemoteWebDriver
  RemoteWebDriver remoteDriver = (RemoteWebDriver) driver;

  // 2. Включаем детектор локальных файлов
  remoteDriver.setFileDetector(new LocalFileDetector());

  // 3. Находим поле загрузки
  WebElement fileInput = driver.findElement(By.cssSelector("input[type='file']"));

  // 4. Отправляем путь к локальному файлу
  File fileToUpload = new File("src/test/resources/test_data/document.pdf");
  fileInput.sendKeys(fileToUpload.getAbsolutePath());
  ```
  
- Для кросс-ОС совместимости используйте абсолютные пути или ресурсы из `src/test/resources`, преобразуя их через `getClass().getResource()`.
  ```java
  // Работает везде: IDE, Maven, Jenkins)
  URL resourceUrl = getClass().getClassLoader().getResource("test_data/document.pdf");
  File file = new File(resourceUrl.toURI());

  fileInput.sendKeys(file.getAbsolutePath());
  ```
  Как это работает:

  - `getClass().getClassLoader().getResource()` ищет файл в `src/test/resources` (или в корне JAR)
  - Возвращает URL (например, `file:/C:/project/target/test-classes/test_data/document.pdf`)
  - Преобразуем в File и получаем абсолютный путь, корректный для текущей ОС

## Best Practices

- Всегда возвращаться в `defaultContent()` после работы с iframe, иначе дальнейшие локаторы будут искать элементы внутри фрейма и падать
- Проверять размер `getWindowHandles()` перед переключением, использовать циклы или Stream API для поиска нужной вкладки по URL/Title
- Для алертов использовать явные ожидания (`alertIsPresent`), не полагаться на мгновенное появление диалога
- Ограничивать область видимости cookies доменом и путем, очищать сессию (`driver.manage().deleteAllCookies()`) между независимыми тестами
- Для загрузки файлов использовать абсолютные пути или `File` из ресурсов проекта; в Grid всегда настраивать `LocalFileDetector`
- Избегать хранения состояния `Alert` или `SwitchContext` в полях тестового класса — переключайте контекст непосредственно в методе-шаге
- При работе с `LocalStorage`/`SessionStorage` проверять поддержку браузером через JS: `typeof(Storage) !== "undefined"`

---

# 7. Selenium 4: Продвинутые возможности

> Selenium 4 предоставляет мощные API для отладки браузера, перехвата сетевого трафика, 
> кросс-браузерного прослушивания событий и сбора метрик производительности, выходя далеко за рамки базового манипулирования DOM.

## DevTools Protocol (CDP) интеграция

> CDP (Chrome DevTools Protocol) — это протокол, который позволяет внешним инструментам управлять браузером на том же уровне, как это делают встроенные инструменты разработчика (F12).

- Интерфейс `HasDevTools` позволяет получать доступ к сессии `DevTools` для браузера на базе Chromium.
- Поддерживает эмуляцию геолокации, мобильных устройств, сетевых задержек и принудительное переключение тем оформления.
- Требует явного создания сессии через `createSession()` и её закрытия для предотвращения утечек памяти.

### Базовая настройка CDP

```java
DevTools devTools = ((HasDevTools) driver)  // приводим driver к интерфейсу HasDevTools (этот интерфейс реализуют ChromeDriver, EdgeDriver и др. для Chromium-браузеров)
        .getDevTools();                     // получаем объект для работы с DevTools Protocol 
devTools.createSession();                   // создаем отдельную сессию CDP (важно! это потребляет ресурсы)

// Эмуляция геолокации (широта, долгота, точность)
devTools.send(Emulation.setGeolocationOverride(
    Optional.of(55.7558), 
    Optional.of(37.6173), 
    Optional.of(1.0)
));

// Закрытие сессии (обязательно в teardown)
devTools.close();
```

## Перехват и модификация сетевых запросов

- Позволяет блокировать нежелательные ресурсы (аналитика, шрифты, реклама), модифицировать заголовки и мокать ответы API на лету.
- Использует команды `Network.enable()`, `Network.setBlockedURLs()`, `Fetch.enable()`.
- Критически важно для тестирования edge-cases и offline-режимов без зависимости от внешних сервисов.

### Блокировка и мок ответов

```java
// Network.enable() — активирует перехват всех сетевых событий (запросы, ответы, ошибки)
default Command enable(Optional<String> urlPattern,                 // Фильтрует, какие запросы перехватывать, по URL.
                       Optional<RequestPattern> requestPattern,     // Более сложная фильтрация по типу запроса, заголовкам и т.д.
                       Optional<String> clientId)                   // Идентификатор клиента для контекста (редко используется, в основном для отладки)
// Три аргумента, каждый обернут в Optional (может присутствовать или отсутствовать)
```

```java
devTools           // Объект сессии DevTools
    .send(         // Метод для отправки команды в браузер
        Fetch.enable(           // Команда: включить перехват fetch
            Optional.empty(),   // Параметр 1: URL-фильтр (пусто = все URL)
            Optional.empty(),   // Параметр 2: сложный фильтр (пусто = нет фильтра)
            Optional.empty()    // Параметр 3: ID клиента (пусто = не указан)
        )
    );
```

```java
// Перехват с комбинированными фильтрами

RequestPattern pattern = new RequestPattern()
        .withUrlPattern("/api/users/*")
        .withMethod("GET")
        .withResourceType(ResourceType.XHR);

devTools.send(Fetch.enable(
        Optional.empty(),               // URL уже задан в pattern
        Optional.of(pattern),
        Optional.empty()
));
```

```java
// Fetch.fulfillRequest() перехватывает запрос и возвращает свой собственный ответ, полностью игнорируя настоящий сервер.

default Command fulfillRequest(
    String requestId,                   // ID запроса (обязательный)
    int responseCode,                   // HTTP статус (200, 404, 500...)
    List<HeaderEntry> responseHeaders,  // Заголовки ответа
    Optional<String> body,              // Тело ответа (JSON, HTML, текст...)
    Optional<String> responsePhrase     // Текстовое описание статуса (опционально)
)
```

Примеры:

```java
// Включение перехвата сети
devTools.send(Network.enable(                                       // активирует перехват всех сетевых событий
        Optional.empty(), Optional.empty(), Optional.empty()));

// Блокировка аналитики и сторонних скриптов
devTools.send(Network.setBlockedURLs(                               // Network.setBlockedURLs() — указывает браузеру: "если видишь URL, подходящий под эти паттерны, даже не пытайся его загрузить"
        List.of("*://analytics.*/*", "*://ads.*/*")));

// Перехват и изменение ответа API
devTools.send(Fetch.enable(
        Optional.empty(), Optional.empty(), Optional.empty())); 

devTools.addListener(Fetch.requestPaused(), request -> { // В 90% случаев для тестов используют Fetch.enable(Optional.empty(), Optional.empty(), Optional.empty()), потому что фильтрацию удобнее делать внутри Fetch.requestPaused() обработчика
    String url = request.getRequest().getUrl();
    if (url.contains("/api/user/profile")) {
        devTools.send(Fetch.fulfillRequest( // Fetch.fulfillRequest() — это мок ответа API на уровне браузера, без реального похода на сервер
            request.getRequestId(), 
            200, 
            Collections.emptyList(), 
            Optional.of("{\"id\":1,\"role\":\"admin\"}"), 
            Optional.empty()
        ));
    } else {
        devTools.send(Fetch.continueRequest(
            request.getRequestId(), 
            Optional.empty(), 
            Optional.empty(), 
            Optional.empty(), 
            Optional.empty(), 
            Optional.empty()
        ));
    }
});
```

## BiDi API (WebDriver BiDi)

- Кросс-браузерный стандарт на базе WebSocket, пришедший на смену проприетарным расширениям.
- Подписка на события: `browsingContext`, `log`, `network`, `script`.
- Позволяет слушать `console.log`, ошибки JS и события навигации в реальном времени для Chrome, Firefox и Edge.

> Включение перехвата:
>
> В BiDi не нужно явно посылать `Network.enable` или `Fetch.enable`. 
> 
> Подписка на событие beforeRequestSent автоматически включает перехват.

Полная сигнатура конструктора ContinueRequest в Selenium WebDriver BiDi API:

```java
public ContinueRequest(
        Optional<org.openqa.selenium.bidi.network.Body> body,
        // Optional.of(Body.fromString("{\"key\":\"value\"}")); или Optional.of(Body.fromFile(Paths.get("/path/to/file")));
        Optional<List<org.openqa.selenium.bidi.network.Header>> headers,
        // Optional.of(List.of(new Header("Content-Type", "application/json")));
        Optional<org.openqa.selenium.bidi.network.CookieHeader> cookies,
        // Optional.of(new CookieHeader(List.of(new Cookie("sessionId", "abc123"))));
        Optional<org.openqa.selenium.bidi.network.Auth> authorization)
/*
// Basic auth
Auth authorization = new Auth(
    Auth.AuthType.BASIC,
    Optional.of("username"),
    Optional.of("password"),
    Optional.empty()
);

// Bearer token
Auth authorization = new Auth(
    Auth.AuthType.BEARER,
    Optional.empty(),
    Optional.empty(),
    Optional.of("token123")
);
Optional.of(authorization)
 */
```

Блокировка URL:

```java
if (url.matches(".*analytics\\..*")) {
    biDi.getNetwork().failRequest(requestId, new Network.ErrorReason("BLOCKED_BY_CLIENT"));
}
// Аналог Network.setBlockedURLs — но более гибкий через условия в коде.
```

Подмена ответа:

```java
biDi.getNetwork().continueResponse(requestId, new ContinueResponse(
                                                        Optional.of(200L),         // статус
                                                        Optional.empty(),          // заголовки
                                                        Optional.of(mockBodyBytes) // новое тело ответа
));
// Аналог Fetch.fulfillRequest.
```

Пропуск запроса:
```java
biDi.getNetwork().continueRequest(requestId, new ContinueRequest(
                                                            Optional.empty(),
                                                            Optional.empty(),
                                                            Optional.empty(),
                                                            Optional.empty()));
```

В BiDi (в отличие от CDP с Fetch.enable) каждый перехваченный запрос требует явного решения:

- `continueRequest` - продолжить (пропустить)
- `continueResponse` - подменить ответ
- `failRequest` - заблокировать

Без вызова одного из этих методов запрос "зависнет" в состоянии requestPaused.

Пример:

```java
// Подписка на логи консоли через BiDi
BiDi bidi = ((HasBiDi) driver).getBiDi();
LogInspector logInspector = new LogInspector(bidi);
BrowsingContext context = new BrowsingContext(driver, ContextId.ROOT);

logInspector.onLogEvent(logEntry -> {
    if (logEntry.getLevel().equals(Level.ERROR)) {
        System.err.println("[BiDi Error] " + logEntry.getText());
    }
});

// Навигация с ожиданием загрузки через BiDi
context.navigate("https://example.com");
```

### Сравнение CDP и BiDi

| Параметр            | CDP (Chrome DevTools Protocol)                | BiDi (WebDriver BiDi)                               |
|:--------------------|-----------------------------------------------|-----------------------------------------------------|
| Поддержка браузеров | Только Chromium-based                         | Chrome, Firefox, Edge, Safari (в процессе)          |
| Архитектура         | Проприетарный HTTP/WS                         | Стандарт W3C                                        |
| События             | `Log.entryAdded`, `Network.requestWillBeSent` | `log.entryAdded`, `network.beforeRequestSent`       |
| Статус              | Stable для Chrome/Edge                        | Active development, рекомендован для новых проектов |

## Скриншоты элементов и страницы

- `TakesScreenshot` для снимков всего экрана или видимой области.
- `WebElement.getScreenshotAs(OutputType.BYTES)` для автоматической обрезки конкретного элемента (Selenium 4).
- Конвертация в `BASE64` для прямой интеграции с Allure/Extent без сохранения временных файлов.

### Получение скриншотов

```java
// Скриншот всей страницы с обработкой оконного менеджера
// Полезно для фиксации состояния UI при падении теста
File fullPageScreenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
Path fullPagePath = Paths.get("reports/screenshots/full_page.png");
Files.copy(fullPageScreenshot.toPath(), fullPagePath, StandardCopyOption.REPLACE_EXISTING);

// Скриншот конкретного элемента с автоматической обрезкой (Selenium 4+)
// Идеально для выделения валидационных сообщений, модальных окон или кнопок
WebElement errorMessage = driver.findElement(By.cssSelector(".validation-error"));
byte[] elementScreenshot = errorMessage.getScreenshotAs(OutputType.BYTES);
Path elementPath = Paths.get("reports/screenshots/error_element.png");
Files.write(elementPath, elementScreenshot);

// Конвертация в BASE64 для Allure/Extent Reports (без временных файлов)
// Прямая интеграция с отчетом - скриншот будет встроен в HTML
String base64Screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BASE64);
Allure.addAttachment("Скриншот ошибки", "image/png", base64Screenshot, "png");

// Скриншот видимой области (viewport) с помощью JavaScript
// Когда стандартный getScreenshotAs захватывает больше, чем нужно
WebElement viewport = driver.findElement(By.tagName("body"));
byte[] viewportScreenshot = viewport.getScreenshotAs(OutputType.BYTES);
Files.write(Paths.get("reports/screenshots/viewport.png"), viewportScreenshot);

// Скриншот с маскированием конфиденциальных данных (пример)
// Для защиты персональных данных в отчетах
BufferedImage image = ImageIO.read(new ByteArrayInputStream(
        ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES)
));
Graphics2D g = image.createGraphics();
g.setColor(Color.BLACK);
g.fillRect(100, 200, 300, 50); // Закрашиваем область с email/паролем
g.dispose();
ImageIO.write(image, "png", Paths.get("reports/screenshots/masked.png").toFile());
```

## Производительность и трассировка

- Сбор метрик через `Performance` CDP-домен.
- Анализ `Navigation Timing`, `Resource Timing` для оценки скорости загрузки компонентов.
- Интеграция с Chrome Tracing для генерации `trace.json`.
- Альтернативы для глубокого аудита: Lighthouse CI, WebPageTest, k6 Browser.

### Сбор метрик загрузки

БАЗОВЫЕ МЕТРИКИ ЧЕРЕЗ PERFORMANCE CDP:

```java
// Включение Performance домена для сбора метрик
devTools.send(Performance.enable(Optional.empty()));

// Получение всех доступных метрик после загрузки страницы
List<Metric> metrics = devTools.send(Performance.getMetrics());

// Извлечение ключевых метрик загрузки страницы
Map<String, Double> performanceMetrics = new HashMap<>();
metrics.forEach(metric -> {
    switch(metric.getName()) {
        case "DomContentLoaded" -> // DOM полностью построен
            performanceMetrics.put("DOM Content Loaded (ms)", metric.getValue());
        case "FirstPaint" -> // Первая отрисовка пикселей
            performanceMetrics.put("First Paint (ms)", metric.getValue());
        case "FirstMeaningfulPaint" -> // Полезный контент появился
            performanceMetrics.put("First Meaningful Paint (ms)", metric.getValue());
        case "Load" -> // Страница полностью загружена
            performanceMetrics.put("Full Load (ms)", metric.getValue());
    }
});
```

CHROME TRACING (генерация trace.json для DevTools):

```java
// Создаем файл трассировки для анализа в chrome://tracing
try {
    // Включаем трассировку с нужными категориями
    devTools.send(Tracing.start(new Tracing.Start()
        .setCategories("devtools.timeline,v8,blink.console")
        .setOptions("record-until-full")));
    
    // Выполняем действие, которое хотим профилировать
    driver.findElement(By.id("slow-button")).click();
    
    // Останавливаем трассировку и получаем данные
    TracingEndResponse traceData = devTools.send(Tracing.end());
    String traceJson = traceData.getStream().toString();
    
    // Сохраняем для анализа в chrome://tracing
    Files.writeString(Paths.get("reports/traces/trace_%d.json".formatted(System.currentTimeMillis())), traceJson);
} catch (Exception e) {
    System.err.println("Трассировка не поддерживается в текущем драйвере");
}
```

## Best Practices

- Закрывать CDP-сессии в `@AfterMethod` или `finally`, избегать утечек памяти и конфликтов сессий
- Использовать мокирование только для нестабильных внешних API, не для покрытия бизнес-логики
- BiDi предпочтительнее CDP для кросс-браузерной поддержки и будущих версий Selenium
- Скриншоты элементов делать только после `ExpectedConditions.visibilityOf()`, чтобы избежать пустых кадров
- Логировать перехваченные запросы на уровне `DEBUG`, не хранить сырые ответы в продакшен-отчётах
- Отключать `Performance.enable()` после сбора метрик, чтобы не нагружать браузер постоянным сбором данных
- При работе с `Fetch` всегда обрабатывать `requestPaused` коллбэки, иначе браузер повиснет в ожидании ответа

---

## 8. Архитектура

> Грамотная архитектура тестового фреймворка обеспечивает масштабируемость, переиспользование кода и изоляцию изменений UI от бизнес-логики тестов. 
> 
> В данном разделе рассматриваются ключевые паттерны проектирования, управление жизненным циклом драйвера и организация пакетной структуры.

## Page Object Model (POM)

- Принцип: один класс представляет одну страницу или логический UI-компонент
- Локаторы инкапсулируются как `private`, методы взаимодействия — `public`
- Методы возвращают `this` или новую страницу для поддержки Fluent-интерфейсов
- Базовый класс `BasePage` выносит общие действия: ожидание готовности, скролл, получение URL

```java
public abstract class BasePage {
    
    protected final WebDriver driver;
    protected final WebDriverWait wait;
    protected final Actions actions;
    protected final JavascriptExecutor js;
    
    // Конфигурация таймаутов
    private static final int DEFAULT_TIMEOUT_SECONDS = 10;
    private static final int POLLING_INTERVAL_MS = 500;
    
    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT_SECONDS));
        this.wait.pollingEvery(Duration.ofMillis(POLLING_INTERVAL_MS));
        this.actions = new Actions(driver);
        this.js = (JavascriptExecutor) driver;
    }
    
    // ========== ОБЩИЕ МЕТОДЫ ОЖИДАНИЯ ==========
    
    protected WebElement waitUntilVisible(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }
    
    protected WebElement waitUntilClickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }
    
    protected boolean waitUntilInvisible(By locator) {
        return wait.until(ExpectedConditions.invisibilityOfElementLocated(locator));
    }
    
    // ========== FLUENT-МЕТОДЫ (возвращают this для цепочек) ==========
    
    // Скролл к элементу с плавной анимацией
    public BasePage scrollToElement(By locator) {
        WebElement element = waitUntilVisible(locator);
        js.executeScript("arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});", element);
        return this; // Возвращаем текущий объект для дальнейших вызовов
    }
    
    // Ожидание загрузки страницы (проверка DOM ready)
    public BasePage waitForPageLoad() {
        wait.until(driver -> js.executeScript("return document.readyState").equals("complete"));
        return this;
    }
    
    // Клик с JavaScript (когда обычный клик не работает)
    public BasePage jsClick(By locator) {
        WebElement element = waitUntilVisible(locator);
        js.executeScript("arguments[0].click();", element);
        return this;
    }
    
    // Очистка поля перед вводом
    public BasePage clearField(By locator) {
        waitUntilVisible(locator).clear();
        return this;
    }
    
    // Ввод текста с предварительной очисткой
    public BasePage type(By locator, String text) {
        WebElement element = waitUntilVisible(locator);
        element.clear();
        element.sendKeys(text);
        return this;
    }
    
    // Получение текста с защитой от null
    public String getText(By locator) {
        return waitUntilVisible(locator).getText();
    }
    
    // Проверка, отображается ли элемент
    public boolean isDisplayed(By locator) {
        try {
            return driver.findElement(locator).isDisplayed();
        } catch (NoSuchElementException e) {
            return false;
        }
    }
    
    // Наведение мыши на элемент
    public BasePage hover(By locator) {
        WebElement element = waitUntilVisible(locator);
        actions.moveToElement(element).perform();
        return this;
    }
    
    // Скриншот элемента для отчетности
    public byte[] takeElementScreenshot(By locator) {
        return waitUntilVisible(locator).getScreenshotAs(OutputType.BYTES);
    }
}
```

```java
public class LoginPage extends BasePage {
    
    // Локаторы (инкапсулированы как private)
    private final By usernameInput = By.id("user-login");
    private final By passwordInput = By.name("password");
    private final By submitBtn = By.cssSelector("button[type='submit']");
    private final By rememberMeCheckbox = By.id("remember-me");
    private final By forgotPasswordLink = By.linkText("Forgot password?");
    private final By errorMessage = By.cssSelector(".alert-danger");
    
    public LoginPage(WebDriver driver) {
        super(driver);
    }
    
    // ========== FLUENT-МЕТОДЫ (возвращают this) ==========
    
    // Ввод логина с автоматической очисткой
    public LoginPage enterUsername(String username) {
      type(usernameInput, username);
      return this; // Возвращаем this для fluent-цепочки
    }
    // Пример: new LoginPage(driver).enterUsername("user").enterPassword("pass")

    // Ввод пароля
    public LoginPage enterPassword(String password) {
        type(passwordInput, password);
        return this;
    }
    
    // Установка чекбокса "Запомнить меня"
    public LoginPage checkRememberMe() {
        WebElement checkbox = waitUntilVisible(rememberMeCheckbox);
        if (!checkbox.isSelected()) {
            checkbox.click();
        }
        return this;
    }
    
    // Снятие чекбокса "Запомнить меня"
    public LoginPage uncheckRememberMe() {
        WebElement checkbox = waitUntilVisible(rememberMeCheckbox);
        if (checkbox.isSelected()) {
            checkbox.click();
        }
        return this;
    }
    
    // Скролл к кнопке входа (сначала скроллим, потом жмем)
    public LoginPage scrollToSubmitButton() {
        scrollToElement(submitBtn);
        return this;
    }
    
    // Ожидание готовности формы входа
    public LoginPage waitForFormReady() {
        waitUntilVisible(usernameInput);
        waitUntilVisible(passwordInput);
        return this;
    }
    
    // ========== МЕТОДЫ ПЕРЕХОДА НА ДРУГИЕ СТРАНИЦЫ ==========
    
    // Успешный логин - возвращает новую страницу (DashboardPage)
    public DashboardPage loginSuccess(String username, String password) {
        enterUsername(username)
            .enterPassword(password)
            .waitForFormReady()
            .scrollToSubmitButton();
        
        waitUntilClickable(submitBtn).click();
        return new DashboardPage(driver); // Возвращаем НОВУЮ страницу
    }
    
    // Неуспешный логин - возвращаем эту же страницу (остаемся на LoginPage)
    public LoginPage loginFail(String username, String password) {
        enterUsername(username)
            .enterPassword(password)
            .scrollToSubmitButton();
        
        waitUntilClickable(submitBtn).click();
        return this; // Возвращаем текущую страницу для проверки ошибки
    }
    
    // Полный Fluent-пример с проверкой
    public DashboardPage loginWithFluentStyle(String username, String password) {
        return this.enterUsername(username)
                   .enterPassword(password)
                   .checkRememberMe()
                   .waitForFormReady()
                   .scrollToSubmitButton()
                   .clickSubmit()
                   .waitForPageLoad(); // Ждем загрузки дашборда
    }
    
    // Клик по кнопке Submit (возвращает this для промежуточных действий)
    private LoginPage clickSubmit() {
        waitUntilClickable(submitBtn).click();
        return this;
    }
    
    // ========== ВЕРИФИКАЦИОННЫЕ МЕТОДЫ ==========
    
    // Проверка отображения формы логина
    public boolean isLoginFormDisplayed() {
        return waitUntilVisible(usernameInput).isDisplayed();
    }
    
    // Получение текста ошибки (если есть)
    public String getErrorMessage() {
        if (isDisplayed(errorMessage)) {
            return getText(errorMessage);
        }
        return "";
    }
    
    // Проверка, отображается ли сообщение об ошибке
    public boolean isErrorDisplayed() {
        return isDisplayed(errorMessage);
    }
    
    // Переход на страницу восстановления пароля
    public ForgotPasswordPage goToForgotPassword() {
        waitUntilClickable(forgotPasswordLink).click();
        return new ForgotPasswordPage(driver);
    }
    
    // Очистка формы логина
    public LoginPage clearForm() {
        clearField(usernameInput);
        clearField(passwordInput);
        return this;
    }
}
```

Пример использования Fluent-интерфейса в тесте:
```java
@Test
public void testFluentLoginExample() {

    WebDriver driver = new ChromeDriver();

    // Пример 1: Простой логин с возвратом новой страницы
    DashboardPage dashboard = new LoginPage(driver)
        .enterUsername("admin")
        .enterPassword("secret")
        .checkRememberMe()
        .loginSuccess("admin", "secret");
    
    Assert.assertTrue(dashboard.isUserMenuDisplayed());
    
    // Пример 2: Ошибочный логин (остаемся на той же странице)
    LoginPage loginPage = new LoginPage(driver)
        .enterUsername("wrong")
        .enterPassword("wrong")
        .loginFail("wrong", "wrong");
    
    Assert.assertTrue(loginPage.isErrorDisplayed());
    Assert.assertEquals("Invalid credentials", loginPage.getErrorMessage());
    
    // Пример 3: Супер-fluent цепочка с промежуточными действиями
    LoginPage dashboardPage = new LoginPage(driver)
        .waitForFormReady()
        .enterUsername("user")
        .enterPassword("pass")
        .checkRememberMe()
        .scrollToSubmitButton()
        .waitForPageLoad()  // наследуется из BasePage
        .loginSuccess("user", "pass");  // переходим на Dashboard
  
    // ассерты...
}
```

---

## Управление драйвером и жизненный цикл

- `ThreadLocal<WebDriver>` гарантирует изоляцию сессий при параллельном выполнении
- Factory-паттерн (`DriverFactory`) централизует создание и конфигурацию браузеров
- Корректное завершение: `quit()` закрывает все окна и убивает процесс драйвера, `close()` — только текущее окно
- Хуки фреймворка (`@BeforeMethod`, `@AfterMethod`) управляют инициализацией и очисткой

```java
public class DriverFactory {
    
    private static final ThreadLocal<WebDriver> DRIVER_POOL = new ThreadLocal<>();

    public static WebDriver getDriver() {
        if (DRIVER_POOL.get() == null) {
            ChromeOptions options = new ChromeOptions();
            options.addArguments("--headless=new");
            DRIVER_POOL.set(new ChromeDriver(options));
        }
        return DRIVER_POOL.get();
    }

    public static void quitDriver() {
        WebDriver driver = DRIVER_POOL.get();
        
        if (driver != null) {
            driver.quit();
            DRIVER_POOL.remove();
        }
    }
}
```

---

## Паттерны и абстракции

- `PageFactory` и `@FindBy` помечены как deprecated в Selenium 4 — рекомендуется ручная инициализация или кастомные прокси
- `Screenplay Pattern` разделяет роли: `Actor` (исполнитель), `Task` (действие), `Question` (проверка), `Ability` (возможность)
- Builder для сложных сценариев: цепочечное создание тестовых данных или навигации
- Facade для высокоуровневых операций: скрытие деталей реализации за методами `LoginPage.loginAs()`, `CartPage.addItem()`

```java
public class CheckoutFlow {
    
    private final WebDriver driver;
    private final LoginPage loginPage;
    private final CartPage cartPage;

    public CheckoutFlow(WebDriver driver) {
        this.driver = driver;
        this.loginPage = new LoginPage(driver);
        this.cartPage = new CartPage(driver);
    }

    public CheckoutFlow executeGuestCheckout(String productUrl) {
        driver.get(productUrl);
        cartPage.addToCart();
        cartPage.proceedToCheckoutAsGuest();
        return this;
    }
}
```

---

## Организация кода и модульность

- Рекомендуемая пакетная структура: `pages`, `components`, `utils`, `tests`, `config`, `listeners`
- Вынос локаторов в enum `Locators` или внешние YAML/JSON файлы для централизованного управления
- Параметризация страниц через конструкторы или DI-контейнеры (Spring, Guice) для упрощения тестов
- Интеграция с логированием и отчетностью на уровне слушателей (`ITestListener`, `Extension`)

```text
src/test/java/
├── pages/          # Page Objects и компоненты UI
├── utils/          # Утилиты: Waits, FileOps, ConfigReader
├── tests/          # Тестовые сценарии и Data-Driven тесты
├── config/         # Конфигурации: BrowserConfig, EnvProperties
└── listeners/      # Слушатели: Allure, Screenshot, Retry
```

---

## Best Practices

- Не дублировать логику навигации, выносить в `BasePage` или `NavigationManager`
- Избегать `PageFactory.initElements()` в Selenium 4, использовать ручную инициализацию или кастомные прокси
- Делать методы страниц возвращающими `this` или конкретную страницу для цепочек
- Драйвер не должен передаваться напрямую в тесты, использовать менеджер или DI
- Покрывать `BasePage` юнит-тестами, проверять инициализацию локаторов на этапе компиляции
- Использовать `ThreadLocal` для потокобезопасности в параллельных прогонах
- Разделять тесты по слоям: `unit` (быстрые), `integration` (API), `e2e` (UI)

---

## 9. Отладка, Логирование и Диагностика

> Эффективная отладка и сбор диагностических данных превращают падающие тесты из «черного ящика» в понятный отчет о причине сбоя. 
> 
> Этот раздел охватывает автоматизацию скриншотов, анализ логов браузера, интеграцию с системами отчетности и техники локальной отладки.

## Скриншоты и видео

- `TakesScreenshot` интерфейс предоставляет методы `getScreenshotAs(OutputType.FILE/BYTES/BASE64)`
- Автоматический захват при падении: реализуется через `ITestListener` (TestNG) или `TestWatcher` (JUnit 5) с `alwaysRun = true`
- Скриншот элемента vs всей страницы: `WebElement.getScreenshotAs()` автоматически обрезает область видимости
- Запись видео: через Docker-контейнеры (Selenoid/Grid 4), FFmpeg или специализированные плагины

```java
public class ScreenshotManager {
    
    public static String captureScreenshot(WebDriver driver, String testName) {
        try {
            String fileName = testName + "_" + LocalDateTime.now().toString().replace(":", "-") + ".png";
            File srcFile = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            Files.copy(srcFile.toPath(), Paths.get("reports/screenshots/" + fileName));
            return fileName;
        } catch (Exception e) {
            System.err.println("Failed to capture screenshot: " + e.getMessage());
            return null;
        }
    }
}

// ИСПОЛЬЗОВАНИЕ:

@Test
public void testLoginFailure() {
    driver = new ChromeDriver();
    driver.get("https://example.com/login");

    // Действие, которое упадёт
    driver.findElement(By.id("nonexistent")).click();
}

// Автоматический скриншот при падении
@AfterMethod
public void tearDown(ITestResult result) {
    if (ITestResult.FAILURE == result.getStatus()) {
        String screenshotName = ScreenshotManager.captureScreenshot(driver, result.getName());
        System.out.println("Скриншот сохранён: " + screenshotName);
    }
    if (driver != null) {
        driver.quit();
    }
}
```

Лучшая практика — с Allure:
```java
@AfterMethod
public void takeScreenshotOnFailure(ITestResult result) {
    if (result.getStatus() == ITestResult.FAILURE && driver != null) {
        byte[] screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        Allure.addAttachment("Screenshot on failure", "image/png", screenshot, "png");
    }
}
```

## Логи браузера и сети

- `LogEntries`, `LogType.BROWSER`, `LogType.DRIVER`, `LogType.PERFORMANCE`
- Фильтрация по уровню: `Level.ALL`, `Level.FINE`, `Level.WARNING`, `Level.SEVERE`
- Парсинг JS-ошибок, CORS-предупреждений, статусов 4xx/5xx
- Вывод в консоль, запись в файлы, прикрепление к отчетам Allure/Extent

```java
LogEntries browserLogs = driver.manage().logs().get(LogType.BROWSER);
for (LogEntry entry : browserLogs) {
    if (entry.getLevel().equals(Level.SEVERE) || entry.getMessage().contains("Failed to load")) {
        System.err.println("[BROWSER ERROR] " + entry.getMessage());
    }
}
```

## Диагностика и отладка

- `driver.getPageSource()` для анализа текущего состояния DOM
- `getDomProperty()` vs `getAttribute()` (v4 изменения): первое отражает live-состояние, второе — исходный HTML
- Headed vs Headless: визуальная отладка через `--remote-debugging-port=9222`, подключение Chrome DevTools
- Интеграция с IDE: breakpoints в Selenium-коде, пошаговое выполнение, инспекция `WebElement` прокси

```bash
# Запуск Chrome с портом удаленной отладки для подключения DevTools
--remote-debugging-port=9222
--disable-extensions
--user-data-dir=/tmp/chrome-debug-profile
```

## Логирование и отчеты

> Централизованное логирование и интеграция с системами отчетности превращают сырые данные выполнения в понятную диагностическую картину. 
> 
> Конфигурация логгера, структурирование сообщений и автоматическое прикрепление артефактов критически важны для анализа падений в CI/CD.

### Конфигурация SLF4J + Logback

- **Фасад SLF4J** — унифицированный API, позволяющий менять реализацию (Logback, Log4j2) без изменения кода тестов
- **Logback** — реализация по умолчанию, поддерживает асинхронные аппендеры, фильтрацию по уровням и ротацию файлов
- **Уровни логирования** — `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`; для тестов рекомендуется `INFO` по умолчанию, `DEBUG` для отладки локаторов и ожиданий
- **Ротация файлов** — `RollingFileAppender` с политикой `SizeAndTimeBasedRollingPolicy` предотвращает переполнение диска артефактами прогонов

### Структурированные логи и трассировка

- **JSON-формат** — удобен для парсинга ELK/Splunk, включает `timestamp`, `thread`, `level`, `logger`, `message`, `correlationId`
- **Correlation ID** — уникальный идентификатор сессии теста, передается во все шаги для сквозной трассировки в распределенных окружениях
- **Step Tracing** — логирование каждого шага теста через `@Step` или кастомные обертки для восстановления последовательности действий

### Интеграция с Allure

- `@Step` — маркировка методов-действий, автоматическое отображение в дереве выполнения
- `@Attachment` — прикрепление скриншотов, логов, `pageSource` при падении или по условию
- `@Severity` — приоритизация: `BLOCKER`, `CRITICAL`, `NORMAL`, `MINOR`, `TRIVIAL`
- `@Link` / `@Issue` / `@TmsLink` — связь с задачами Jira, баг-трекерами и тест-кейсами в TestRail/Qase

### Обработка исключений

- Логирование контекста (`URL`, `current page title`, `active element`) в `catch`-блоках или `TestWatcher`
- Сохранение `pageSource` только при `ERROR/FAILED` для минимизации размера отчетов
- Использование `try-with-resources` или `finally` для гарантированной очистки временных файлов

### Конфигурация Logback с детальными комментариями

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Конфигурация Logback для тестового окружения.
  Оптимизирована для CI/CD: вывод в консоль, фильтрация шума от Selenium,
  асинхронная запись в файл с ротацией по размеру и времени.
-->
<configuration>

    <!--
      Аппендер для вывода в консоль.
      Формат включает: время, поток, уровень, имя логгера (макс. 36 символов), сообщение.
      %highlight добавляет цветовое выделение уровней в терминале.
    -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight([%thread]) %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- Фильтр: в консоль выводим только INFO и выше, чтобы не забивать вывод CI -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
    </appender>

    <!--
      Аппендер для записи в файл с ротацией.
      Логи сохраняются в папке logs/, архивируются ежедневно или при достижении 50МБ.
      Хранятся последние 30 дней архивов.
    -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/test-run.log</file>
        <encoder>
            <!-- Расширенный формат для локальной отладки с поддержкой MDC -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} | %-5level | %thread | %logger{36} | %X{correlationId} | %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/test-run.%d{yyyy-MM-dd}.%i.log.zip</fileNamePattern>
            <maxFileSize>50MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>2GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!--
      Настройка логгера для Selenium/WebDriver.
      Уровень WARN подавляет подробные отладочные сообщения драйвера,
      оставляя только критические ошибки и предупреждения.
    -->
    <logger name="org.openqa.selenium" level="WARN" />
    <logger name="org.openqa.selenium.devtools" level="WARN" />
    <logger name="io.github.bonigarcia" level="INFO" />

    <!--
      Корневой логгер проекта.
      Уровень DEBUG для захвата всех шагов тестов, включая кастомные обертки.
      Подключает консольный и файловый аппендеры.
    -->
    <root level="DEBUG">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>

</configuration>
```

## Best Practices

- Делать скриншоты только при `SEVERE` или `FAILED`, избегать захвата на каждом шаге для экономии ресурсов CI
- Парсить логи браузера через Stream API, фильтровать по регулярным выражениям для отсева ложных срабатываний
- В CI/CD пайплайнах отключать verbose-логи драйвера, оставлять только `ERROR/WARN` для читаемости консоли
- Использовать `@Attachment` для автоматической вставки скриншотов, логов и `pageSource` в отчеты
- Хранить `pageSource` только при критических падениях, ограничивать размер файлов для ускорения загрузки отчетов
- Настраивать `LogType.PERFORMANCE` только для конкретных тестов производительности, чтобы не перегружать браузер сбором метрик
- При отладке `StaleElement` всегда логировать текущий URL и timestamp, чтобы отслеживать навигацию и редиректы

---

# 10. Selenium Grid 4 и Параллельное выполнение

> Распределённое выполнение тестов на множестве браузеров и платформ. 
> 
> Grid 4 представляет собой модульную архитектуру на основе событий, заменяющую монолитный Hub/Node подход предыдущих версий.

## Архитектура Grid 4

- **Router** — единая точка входа, распределяет запросы на создание сессий
- **Distributor** — находит подходящий узел (Node) по `capabilities` и назначает сессию
- **Session Map** — хранит активные сессии и их привязку к узлам
- **Node** — исполняет тесты, регистрируется в Grid через Event Bus
- **Event Bus** — шина событий (Pub/Sub) для координации компонентов, работает на Netty

```text
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │---->│   Router    │---->│ Distributor │
│  (Test Run) │     │ (Port 4444) │     │             │
└─────────────┘     └──────┬──────┘     └──────┬──────┘
                           |                   |
                           ▼                   ▼
                   ┌──────────────┐     ┌──────────────┐
                   │  Session Map │     │   Event Bus  │
                   │  (Redis/In-  │     │  (Netty/ZMQ) │
                   │   memory)    │     │              │
                   └───────┬──────┘     └──────┬───────┘
                           |                   |
                           ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐
                    │    Node 1   │     │    Node N   │
                    │ Chrome/Fire │     │  Docker/K8s │
                    └─────────────┘     └─────────────┘
```

### Режимы запуска Grid 4

- **Standalone** — все компоненты Grid (Router, Distributor, Session Map, Event Bus, Node) запускаются в едином JVM-процессе.
  - Это упрощённый режим для локальной отладки, разработки и быстрого старта.
  - Не подходит для продакшена из-за отсутствия отказоустойчивости и горизонтального масштабирования. 


- **Hub + Node** — гибридный режим, совместимый с классической архитектурой Grid 3, но использующий внутренние механизмы Grid 4. 
  - Один процесс (Hub) объединяет Router, Distributor, Session Map и Event Bus. 
  - Отдельные процессы (Node) регистрируются в Hub и исполняют тесты. 
  - Упрощает миграцию с Grid 3, но Hub остаётся единой точкой отказа.


- **Distributed** — каждый компонент Grid (Router, Distributor, Session Map, Event Bus, Node) запускается как отдельный процесс, возможно на разных физических или виртуальных машинах. 
  - Компоненты взаимодействуют через Event Bus (по умолчанию Netty). 
  - Это production-режим для высоконагруженных систем: можно масштабировать только узкие места (например, добавить несколько Distributor), обеспечить отказоустойчивость через репликацию Session Map (Redis) и Event Bus (ZMQ). 
  - Требует настройки переменных окружения или конфигурационного файла для указания адресов компонентов.


- **Docker/Kubernetes** — специализированный режим для контейнерной оркестрации. 
  - Узлы (Node) запускаются как Docker-контейнеры с предустановленными браузерами и драйверами (например, Selenium Docker images). 
  - Grid 4 поддерживает auto-discovery через механизмы Docker Swarm или Kubernetes Services. 
  - В Kubernetes Deployment описываются отдельно Router, Distributor, Session Map (Redis), Event Bus и реплицируемые узлы. 
  - Особенности: динамическое добавление/удаление узлов, изоляция тестов, автоматическое восстановление после сбоев.

```bash
# Standalone режим (все в одном)
java -jar selenium-server-4.25.0.jar standalone

# Hub (Router + Distributor + Session Map + Event Bus)
java -jar selenium-server-4.25.0.jar hub

# Node (регистрируется на Hub)
java -jar selenium-server-4.25.0.jar node --publish-events tcp://hub:4442 --subscribe-events tcp://hub:4443

# Docker Compose запуск (рекомендуемый для CI)
docker-compose -f docker-compose-v3.yml up -d
```

```toml
# config.toml — пример конфигурации Node
[server]
host = "node-chrome"
port = 5555

[node]
detect-drivers = true
max-sessions = 3

[[driver-configuration]]
display-name = "Chrome"
stereotype = '{"browserName": "chrome", "platformName": "linux"}'
max-sessions = 3
```

## Настройка и управление драйверами в Grid

### Selenium Manager в распределённой среде

- Автоматическая загрузка драйверов на каждом Node при старте
- Кэширование в `~/.cache/selenium/`, поддержка offline-режима через `SE_MANAGER_CACHE_PATH`
- Приоритет: локальный кэш → скачивание → fallback на системный PATH

```java
// RemoteWebDriver с автоматическим матчингом через Selenium Manager
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless=new", "--no-sandbox", "--disable-dev-shm-usage");

WebDriver driver = new RemoteWebDriver(
    URI.create("http://grid-host:4444").toURL(),
    options
);
```

### Capabilities матчинг и кастомизация

- Строгий матчинг по `platformName`, `browserVersion`, `se:options`
- Кастомные capabilities для маршрутизации: `se:build`, `se:project`, `se:region`
- Использование `MutableCapabilities` для динамического слияния

```java
MutableCapabilities caps = new MutableCapabilities();
caps.setCapability(ChromeOptions.CAPABILITY, options);
caps.setCapability("se:build", "prod-v2.3");
caps.setCapability("se:project", "checkout-flow");

WebDriver driver = new RemoteWebDriver(
    URI.create("http://grid:4444").toURL(),
    caps
);
```

### Конфигурация через переменные окружения

- `SE_NODE_MAX_SESSIONS` — лимит параллельных сессий на узле
- `SE_NODE_SESSION_TIMEOUT` — таймаут неактивной сессии (по умолчанию 300 сек)
- `SE_VNC_NO_PASSWORD` — отключение аутентификации для VNC в dev-средах
- `SE_ENABLE_TRACING` — включение OpenTelemetry для трассировки запросов

```bash
# Запуск Node с кастомными параметрами
export SE_NODE_MAX_SESSIONS=5
export SE_NODE_SESSION_TIMEOUT=600
export SE_VNC_VIEW_ONLY=true

java -jar selenium-server.jar node \
  --publish-events tcp://hub:4442 \
  --subscribe-events tcp://hub:4443 \
  --detect-drivers true
```

## Параллельное выполнение тестов

### TestNG конфигурация

```xml
<!-- testng.xml: параллелизм на уровне методов -->
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="ParallelSuite" parallel="methods" thread-count="5" data-provider-thread-count="3">
    <test name="ChromeTests">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.example.tests.LoginTest"/>
            <class name="com.example.tests.CheckoutTest"/>
        </classes>
    </test>
</suite>
```

- `parallel="methods"` — каждый `@Test`-метод в отдельном потоке
- `parallel="classes"` — весь тест-класс выполняется в одном потоке
- `thread-count` — общее количество потоков, распределяется динамически
- `data-provider-thread-count` — отдельный лимит для `@DataProvider`

### JUnit 5 параллелизм

```java
@Execution(ExecutionMode.CONCURRENT)
@ResourceLock(value = "browser", mode = ResourceLockMode.READ)
class ParallelTests {

    @Test
    @DisplayName("Проверка корзины")
    void cartTest() {
        // ThreadLocal<WebDriver> обеспечивает изоляцию
    }

    @Test
    @DisplayName("Оформление заказа")
    void checkoutTest() {
        // Независимый тест, может выполняться параллельно
    }
}
```

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.config.strategy = fixed
junit.jupiter.execution.parallel.config.fixed.parallelism = 4
```

### Изоляция данных и ThreadLocal

- `ThreadLocal<WebDriver>` — гарантия, что каждый поток работает со своим драйвером
- Уникальные тестовые данные: генерация email/username через Faker с суффиксом потока
- Cleanup в `@AfterMethod(alwaysRun = true)` — удаление тестовых записей независимо от исхода

```java
public class DriverManager {
    
    private static final ThreadLocal<WebDriver> driver = new ThreadLocal<>();

    public static WebDriver get() {
        return driver.get();
    }

    public static void set(WebDriver webDriver) {
        driver.set(webDriver);
    }

    public static void quit() {
        if (driver.get() != null) {
            driver.get().quit();
            driver.remove();
        }
    }
}
```

```java
@BeforeMethod
public void setUp(Method method) {
    ChromeOptions options = new ChromeOptions();
    options.addArguments("--headless=new");
    
    WebDriver webDriver = new RemoteWebDriver(
        URI.create("http://grid:4444").toURL(),
        options
    );
    DriverManager.set(webDriver);
    
    // Уникальные данные для изоляции
    String testUser = "user_" + method.getName() + "_" + Thread.currentThread().getId();
    PageFactory.initElements(webDriver, LoginPage.class).loginAs(testUser);
}
```

## Мониторинг и диагностика

### Grid UI и API

- Встроенная панель: `http://grid-host:4444` — активные сессии, очередь, метрики узлов
- REST API: `GET /status`, `GET /session/{id}`, `POST /session` для программной диагностики
- Метрики Prometheus: `http://grid-host:4444/metrics` — `grid_session_active`, `node_sessions_used`

```bash
# Проверка статуса Grid через curl
curl -s http://localhost:4444/status | jq '.value.ready, .value.message'

# Получение списка активных сессий
curl -s http://localhost:4444/sessions | jq '.value[] | {id, capabilities}'
```

### Логирование и трассировка

- Уровень логирования: `--log-level FINE` для отладки, `INFO` для production
- Структурированные логи в JSON: `--log-format json` для парсинга в ELK/Splunk
- OpenTelemetry интеграция: `--enable-tracing --tracing-exporter otlp`

```properties
# logging.properties для детальной отладки
.level = INFO
org.openqa.selenium.grid.level = FINE
org.openqa.selenium.remote.level = FINER

# Формат вывода
java.util.logging.SimpleFormatter.format = [%1$tF %1$tT] [%4$s] %5$s %n
```

### Обработка сбоев и retry-логика

- `SessionNotCreatedException` — повторная регистрация Node или переключение на резервный браузер
- `StaleElementReferenceException` в параллели — избегать кеширования `WebElement` между шагами
- Кастомный `IRetryAnalyzer` для flaky-тестов с экспоненциальной задержкой

```java
public class GridRetryAnalyzer implements IRetryAnalyzer {
    
    private int count = 0;
    private static final int MAX_RETRY = 2;

    @Override
    public boolean retry(ITestResult result) {
        if (count < MAX_RETRY && isGridRelatedFailure(result)) {
            count++;
            Thread.sleep(1000 * count); // экспоненциальная задержка
            return true;
        }
        return false;
    }

    private boolean isGridRelatedFailure(ITestResult result) {
        Throwable t = result.getThrowable();
        return t instanceof SessionNotCreatedException ||
               t instanceof WebDriverException && t.getMessage().contains("grid");
    }
}
```

## Best Practices

- Использовать Docker Compose для локальной Grid-инфраструктуры — обеспечивает воспроизводимость окружения
- Настраивать `max-sessions` и `session-timeout` под нагрузку проекта: 3–5 сессий на ядро — оптимальный баланс
- Параллелить только изолированные тесты, избегать общих state/cookies — использовать `ThreadLocal` и уникальные данные
- В `RemoteWebDriver` передавать только `Options`/`MutableCapabilities`, не `DesiredCapabilities` (deprecated в v4)
- Мониторить Grid через встроенный UI и метрики Prometheus, настраивать алерты на `session_queue_size > threshold`
- При падениях в параллели проверять инициализацию `ThreadLocal<WebDriver>` и cleanup в `@AfterMethod`
- В CI использовать `--headless=new` с отключением GPU и sandbox для стабильности в контейнерах
- Для отладки зависших сессий включать `--log-level FINE` и анализировать `grid_session_active` метрику

---

# 11. Интеграция с тестовыми раннерами и экосистемой

> Интеграция Selenium с фреймворками выполнения, системами параметризации, CI/CD и инструментами отчетности для построения масштабируемой и поддерживаемой автоматизации.

## TestNG

- **Аннотации жизненного цикла** — `@BeforeSuite`, `@BeforeClass`, `@BeforeMethod`, `@AfterMethod`, `@AfterClass`, `@AfterSuite` для управления инициализацией и очисткой
- **Группировка и приоритеты** — `groups`, `priority`, `dependsOnMethods` для фильтрации сьютов и управления порядком выполнения
- **Soft Assertions** — `SoftAssert` позволяет продолжить выполнение после падения проверки, финализируется вызовом `assertAll()`
- **Listeners** — `ITestListener`, `ISuiteListener`, `IInvokedMethodListener` для кастомной логики до/после шагов, скриншотов и логирования

```xml
<!-- testng.xml: базовая конфигурация сьюта -->
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="E2E Suite" parallel="tests" thread-count="4" verbose="2">
    <parameter name="env" value="staging"/>
    <test name="Regression">
        <groups>
            <run>
                <include name="smoke"/>
                <exclude name="flaky"/>
            </run>
        </groups>
        <classes>
            <class name="com.qa.tests.LoginTest"/>
            <class name="com.qa.tests.CheckoutTest"/>
        </classes>
    </test>
</suite>
```

```java
public class BaseTest {
    
    protected SoftAssert softAssert;
    protected WebDriver driver;

    @BeforeMethod
    public void init() {
        driver = DriverManager.get();
        softAssert = new SoftAssert();
    }

    @Test(groups = {"smoke", "regression"}, priority = 1)
    public void verifyLogin() {
        softAssert.assertTrue(driver.getTitle().contains("Dashboard"), "Заголовок не совпадает");
        softAssert.assertEquals(driver.findElement(By.id("status")).getText(), "Active");
    }

    @AfterMethod
    public void tearDown() {
        softAssert.assertAll(); // Выбросит AssertionError при накопленных ошибках
        DriverManager.quit();
    }
}
```

## JUnit 5

- **Аннотации** — `@Test`, `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll` с поддержкой наследования и интерфейсов
- **Параметризация** — `@ParameterizedTest`, `@ValueSource`, `@CsvSource`, `@MethodSource`, `@ArgumentsSource`
- **Extension API** — замена наследованию, внедрение драйвера и ресурсов через `@ExtendWith`
- **TestWatcher** — отслеживание статуса тестов (`testSuccessful`, `testFailed`, `testDisabled`) для кастомных реакций

```java
@ExtendWith(WebDriverExtension.class)
class CheckoutTest {

    @ParameterizedTest(name = "Покупка товара {0} с ценой {1}")
    @CsvSource({"laptop, 1200", "mouse, 25", "keyboard, 75"})
    void verifyCheckout(String item, int expectedPrice) {
        WebDriver driver = WebDriverContext.get();
        CartPage page = new CartPage(driver);
        page.addItem(item);
        assertEquals(page.getTotal(), expectedPrice);
    }

    @Test
    @DisplayName("Проверка применения промокода")
    void applyPromo() {
        WebDriver driver = WebDriverContext.get();
        CartPage page = new CartPage(driver);
        assertAll("Проверка корзины",
            () -> assertTrue(page.hasItems()),
            () -> assertEquals(page.getDiscount("SAVE10"), 10)
        );
    }
}
```

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.config.strategy = dynamic
junit.jupiter.extensions.autodetection.enabled = true
```

## Data-Driven и генерация данных

- **Внешние источники** — загрузка из `JSON`, `YAML`, `CSV`, `Excel` через парсеры (Jackson, OpenCSV, Apache POI)
- **Генерация данных** — `java-faker`, `instafaker` для создания уникальных email, имен, адресов без коллизий
- **Testcontainers** — запуск изолированных БД (PostgreSQL, MySQL) или сервисов (Redis, MockServer) в Docker на время теста
- **Управление состоянием** — фабричные методы, seed-скрипты, автоматический cleanup через `@AfterAll` или `DisposableBean`

```java
public class TestDataFactory {

    private static final Faker faker = new Faker();

  public static User generateUser() {
    return new User(
            // Генерирует случайное имя (John, Mary, Robert и т.д.)
            faker.name().firstName(),
            // Генерирует фамилию (Smith, Johnson, Williams)
            faker.name().lastName(),
            // Генерирует email вида "john.doe@example.com"
            faker.internet().emailAddress(),
            // Генерирует число от 18 до 65 (включительно)
            faker.number().numberBetween(18, 65)
    );
  }

    @DataProvider(name = "users")
    public Object[][] provideUsers() {
        return new Object[][]{
            {generateUser()},
            {generateUser()}
        };
    }
}
```

```yaml
# test-data/config.yml
environments:
  staging:
    url: "https://staging.example.com"
    db_host: "localhost"
    db_port: 5432
    timeout: 30
features:
  new_checkout: true
  legacy_api: false
```

## CI/CD и автоматизация

- **Пайплайны** — GitHub Actions, GitLab CI, Jenkins с матричным запуском (browser × OS × version)
- **Оптимизация** — кэширование `~/.m2`, `~/.gradle`, использование `fail-fast: false` для полного прохождения сьюта
- **Gate-механизмы** — блокировка merge при падении `critical` тестов, обязательный проход `smoke`
- **Статический анализ** — интеграция с SonarQube, Checkstyle, SpotBugs, JaCoCo для контроля покрытия кода

```yaml
# .github/workflows/selenium-ci.yml
# Имя workflow - отображается в интерфейсе GitHub Actions
name: UI Automation

# Триггеры (когда запускать пайплайн)
on:
  # Запуск при пуше (коммите) в указанные ветки
  push:
    branches: [ main, develop ]

  # Запуск при создании Pull Request в main
  pull_request:
    branches: [ main ]

# Определяем задачи (jobs) для выполнения
jobs:
  # Название задачи (может быть любым)
  test:
    # На каком виртуальном сервере запускать (ubuntu, windows, macos)
    runs-on: ubuntu-latest

    # Матричный запуск - тесты выполняются с разными комбинациями параметров
    strategy:
      matrix:
        # Тестируем в двух браузерах
        browser: [chrome, firefox]
        # Делим тесты на 4 параллельных шарда (части) для ускорения
        shard: [1, 2, 3, 4]

      # Если один из запусков в матрице упал, остальные продолжают выполняться
      # false - при падении одного браузера, другой все равно запустится
      fail-fast: false

    # Шаги, которые выполняются внутри задачи
    steps:
      # 1. Клонирование репозитория с кодом тестов
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Установка Java Development Kit
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'  # OpenJDK от Eclipse Temurin
          java-version: '17'       # Используем Java 17

      # 3. Кэширование Maven зависимостей (ускоряет последующие запуски)
      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      # 4. Установка и запуск браузера для тестов (Chrome)
      - name: Setup Chrome
        if: matrix.browser == 'chrome'
        uses: browser-actions/setup-chrome@v1

      # 5. Установка и запуск браузера для тестов (Firefox)
      - name: Setup Firefox
        if: matrix.browser == 'firefox'
        uses: browser-actions/setup-firefox@v1

      # 6. Запуск тестов с параметрами из матрицы
      - name: Run Tests
        run: |
          mvn test \
            -Dsurefire.parallel.methods=4 \        # Параллельный запуск 4 методов
            -Dbrowser=${{ matrix.browser }} \      # Передаем браузер из матрицы
            -Dshard=${{ matrix.shard }}            # Передаем номер шарда

      # 7. Загрузка Allure отчета (даже если тесты упали)
      # if: always() - выполняется всегда, независимо от результата тестов
      - name: Upload Allure Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          # Уникальное имя артифакта для каждого запуска
          name: allure-report-${{ matrix.browser }}-shard-${{ matrix.shard }}
          path: target/allure-results

      # 8. Загрузка скриншотов и логов (если тесты упали)
      - name: Upload Screenshots on Failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: failure-screenshots-${{ matrix.browser }}-shard-${{ matrix.shard }}
          path: target/screenshots/
```

## Отчетность и аналитика

- **Allure Framework** — шаги `@Step`, вложения `@Attachment`, категоризация дефектов, history-тренды, `environment.properties`
- **ExtentReports** — кастомные темы, логирование в реальном времени, скриншоты, дашборды
- **Метрики** — `flaky-rate`, `avg execution time`, `pass/fail ratio`, интеграция с Grafana/Prometheus
- **Интеграции** — отправка уведомлений в Slack/Teams, создание тикетов в Jira, email-рассылки по расписанию

```java
@AllureEpic("Авторизация")
@AllureFeature("Вход в систему")
public class LoginTest {

    @Step("Открытие страницы логина")
    public void openLoginPage() {
        driver.get("https://app.example.com/login");
    }

    @Step("Ввод креденшиалов: {username}")
    @Attachment(value = "Скриншот ошибки", type = "image/png")
    public byte[] attachScreenshotOnError(Throwable error) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Test
    @Severity(SeverityLevel.CRITICAL)
    @Description("Проверка успешной авторизации валидным пользователем")
    public void validLogin() {
        openLoginPage();
        LoginPage login = new LoginPage(driver);
        login.fillCredentials("admin", "password123");
        login.clickSubmit();
        Assert.assertTrue(driver.getCurrentUrl().contains("/dashboard"));
    }
}
```

## Best Practices

- Выносить раннер-специфичные хуки в отдельные классы, не смешивать с Page Object Model для соблюдения Single Responsibility
- Использовать `soft assertions` только для non-critical проверок, всегда финализировать через `assertAll()` в конце теста
- Параметризовать тесты через внешние источники (YAML/JSON/CSV), избегать хардкода данных в коде для гибкости поддержки
- В CI всегда запускать с `fail-fast: false` и собирать артефакты отчетов независимо от статуса сьюта для полной видимости
- Настраивать `Testcontainers` для полной изоляции, никогда не использовать продакшен БД или внешние API без моков
- Документировать требования к окружению, переменные и секреты в `README.md` и `.env.example` для онбординга команды
- Использовать `@Flaky` или кастомные ретраи для нестабильных тестов, но обязательно заводить баг-тикеты на их устранение
- Логировать только `WARN`/`ERROR` в CI-пайплайнах, `DEBUG` оставлять для локальной отладки и локальных запусков