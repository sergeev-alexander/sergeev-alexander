


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
// Ручная настрой
// 
// 
// ---
// 
// ## 4. Взаимодействие с элементами (WebElement & Actions API)## 4. Взаимодействие с элементами (WebElement & Actions API)## 4. Взаимодействие с элементами (WebElement & Actions API)ка перед созданием драйвера
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

