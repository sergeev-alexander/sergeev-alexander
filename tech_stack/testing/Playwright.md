**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Playwright

## Содержание

1. [Введение и архитектура Playwright](#1-введение-и-архитектура-playwright)
2. [Подключение и конфигурация окружения](#2-подключение-и-конфигурация-окружения)
3. [Ядро API: Браузер, Контекст, Страница](#3-ядро-api-браузер-контекст-страница)
4. [Селекторы и Locator API](#4-селекторы-и-locator-api)
5. [Взаимодействие с элементами и Auto-waiting](#5-взаимодействие-с-элементами-и-auto-waiting)
6. [Интеграция с JUnit 5 и TestNG](#6-интеграция-с-junit-5-и-testng)
7. [API-тестирование через APIRequestContext](#7-api-тестирование-через-apirequestcontext)
8. [Архитектурные паттерны и организация кода](#8-архитектурные-паттерны-и-организация-кода)
9. [Продвинутые сценарии и изоляция тестов](#9-продвинутые-сценарии-и-изоляция-тестов)
10. [Отчётность, трейсы и артефакты](#10-отчётность-трейсы-и-артефакты)
11. [Инфраструктура: Docker, CI/CD, Параллельный запуск](#11-инфраструктура-docker-cicd-параллельный-запуск)
12. [Эволюция версий и миграция (1.30+ vs Legacy)](#12-эволюция-версий-и-миграция-130-vs-legacy)
13. [Типичные ошибки и Production-Ready практики](#13-типичные-ошибки-и-production-ready-практики)

---

# 1. Введение и архитектура Playwright

> Playwright для Java — это современная библиотека для автоматизации тестирования веб-приложений, разработанная Microsoft. 
> Она предоставляет единый API для управления браузерами Chromium, Firefox и WebKit, обеспечивая кросс-браузерное 
> и кросс-платформенное тестирование с высокой скоростью и стабильностью выполнения сценариев.
>
> Ключевая особенность — архитектура, исключающая «прослойку» JSON-RPC, характерную для Selenium, 
> что снижает накладные расходы и повышает надёжность взаимодействия с браузером.

---

### Общее описание

`Playwright` для Java — это библиотека, позволяющая писать end-to-end тесты на языке программирования Java 
с использованием единого программного интерфейса для всех поддерживаемых браузеров.

- `Playwright` - библиотека для автоматизации браузеров с акцентом на современные веб-стандарты
- `BrowserType` - абстракция типа браузера (`chromium`, `firefox`, `webkit`) с методами запуска и подключения
- `Browser` - экземпляр запущенного браузера, управляемый через `Playwright Driver`
- `BrowserContext` - изолированная сессия браузера с независимыми `cookies`, `localStorage`, `permissions`
- `Page` - отдельная вкладка или фрейм, содержащая DOM-дерево и позволяющая выполнять действия
- `Locator` - ленивый селектор элемента, автоматически ожидающий его доступности перед действием

---

### Сравнение с альтернативами (Playwright vs Selenium WebDriver vs Selenide)

| Критерий                     | Playwright                                | Selenium WebDriver                      | Selenide                                                                               |
|:-----------------------------|:------------------------------------------|:----------------------------------------|:---------------------------------------------------------------------------------------|
| **Язык реализации драйвера** | Node.js (общий для всех языков)           | Java/другие (язык-специфичный)          | Java (обёртка над WebDriver)                                                           |
| **Протокол взаимодействия**  | WebSocket + CDP/Firefox DevTools          | JSON-RPC через WebDriver protocol       | JSON-RPC через WebDriver protocol                                                      |
| **Поддержка браузеров**      | Chromium, Firefox, WebKit (нативно)       | Все браузеры через драйверы             | Все браузеры через драйверы                                                            |
| **Работа с Shadow DOM**      | Автоматическая, без доп. кода             | Требует ручного обхода через JS         | Требует ручного обхода через JS                                                        |
| **Auto-waiting**             | Встроенный механизм `actionability`       | Требует явных `WebDriverWait`           | Встроенный «умный» механизм ожиданий                                                   |
| **Языки тестов**             | Java, C#, Python, JS/TS                   | Все популярные языки                    | Только Java                                                                            |
| **Запуск в CI**              | Официальные Docker-образы, headless-режим | Требует установки драйверов и браузеров | Требует установки драйверов и браузеров, но проще настраивается через WebDriverManager |

---

### Архитектура

Playwright использует клиент-серверную архитектуру, где ваш Java-код (клиент) взаимодействует 
с выделенным процессом `Playwright Driver` через WebSocket-соединение.

```text
┌─────────────────────────────────────────────────────────┐
│                    ВАШ ТЕСТ (Java)                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Playwright Java Client Library                 │    │
│  │  - Browser, Page, Locator API                   │    │
│  │  - Assertions, Auto-waiting logic               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
                    │
                    │ WebSocket (binary protocol)
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│             PLAYWRIGHT DRIVER (Node.js)                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │  - Управление процессами браузеров              │    │
│  │  - Трансляция команд в CDP / Firefox DevTools   │    │
│  │  - Обработка событий, сетевых запросов          │    │
│  └────────────────┬────────────────────────────────┘    │
│                   │                                     │
│     ┌─────────────┴─────────────┐                       │
│     ▼                           ▼                       │
│ ┌─────────┐           ┌─────────────────┐               │
│ │Chromium │           │     Firefox     │               │
│ │  (CDP)  │           │  (RDP/DevTools) │               │
│ └─────────┘           └─────────────────┘               │
│                                                         │
│ ┌─────────┐                                             │
│ │ WebKit  │ (только на macOS / Linux)                   │
│ └─────────┘                                             │
└─────────────────────────────────────────────────────────┘
```

- `WebSocket-канал` - бинарный протокол для передачи команд и событий между клиентом и драйвером, минимизирующий сериализацию/десериализацию
- `CDP (Chrome DevTools Protocol)` - нативный протокол отладки Chromium, используемый Playwright для прямого управления браузером без промежуточных слоёв
- `Firefox DevTools Protocol` - аналогичный протокол для Firefox, обеспечивающий эквивалентный уровень контроля
- `Отсутствие JSON-RPC overhead` - в отличие от Selenium, где каждая команда сериализуется в JSON, передаётся через HTTP и десериализуется, 
  Playwright использует компактный бинарный протокол, что ускоряет выполнение на 15–30%

---

## Основные абстракции Playwright

Ключевые классы и интерфейсы библиотеки, образующие иерархию управления браузером и взаимодействия с веб-страницами. 
Все основные типы реализуют `AutoCloseable` для безопасного управления ресурсами через `try-with-resources`.

- `Playwright` — точка входа в библиотеку, фабрика для создания экземпляров `BrowserType`. 
  Управляет жизненным циклом драйвера (запуск/остановка процесса Node.js).
  - `create()` — статический метод инициализации, запускающий драйвер и возвращающий экземпляр `Playwright`.
  - `chromium()` / `firefox()` / `webkit()` — получение `BrowserType` для конкретного движка.
  - `selectores()` — доступ к кастомным селекторам (регистрация собственных стратегий поиска).
  - `close()` — корректное завершение работы драйвера (вызывается автоматически в `try-with-resources`).
    ```java
    try (Playwright playwright = Playwright.create()) {  // Драйвер запущен
        var browser = playwright.chromium().launch();    // Используем тип браузера
        // ... тестовая логика
    } // Драйвер автоматически остановлен, ресурсы освобождены
    ```

- `BrowserType` — абстракция типа браузера, предоставляющая методы запуска (`launch`) и подключения (`connect`).
  - `launch(LaunchOptions lounchOptions)` — запуск локального экземпляра браузера с настройками (headless, slowMo, args).
  - `connect(String wsEndpoint)` — подключение к удалённому браузеру по WebSocket (Browserless, Selenium Grid).
  - `executablePath()` — возврат пути к установленному бинарному файлу браузера.
  - `name()` — строковое имя типа (`chromium`, `firefox`, `webkit`).
    ```java
    // Запуск Firefox в головном режиме с искусственной задержкой команд
    Browser firefox = playwright.firefox().launch(
        new LaunchOptions().setHeadless(false).setSlowMo(100)
    );
    ```

- `LaunchOptions` — конфигурация запуска браузера через `BrowserType.launch(LaunchOptions lounchOptions)`.
  - `setHeadless(boolean)` — режим без графического интерфейса (по умолчанию `true`).
  - `setSlowMo(int)` — задержка в миллисекундах между действиями (для отладки).
  - `setArgs(List<String>)` — передаваемые браузеру флаги командной строки (`--disable-gpu`, `--start-maximized`).
  - `setDownloadsPath(Path)` — директория для сохранения загружаемых файлов.
  - `setProxy(Proxy)` — настройка прокси-сервера для всего браузера.
  - `setChannel(String)` — выбор конкретного канала браузера (`chrome`, `msedge`, `firefox-beta`).
    ```java
    LaunchOptions options = new LaunchOptions()
        .setHeadless(true)
        .setArgs(List.of("--disable-dev-shm-usage")) // Обход проблем с /dev/shm в Docker
        .setProxy(new Proxy().setServer("http://proxy.local:8080"));
    ```

- `Browser` — запущенный экземпляр браузера, фабрика для создания изолированных контекстов (`BrowserContext`).
  - `newContext(Browser.NewContextOptions)` — создание нового контекста с настройками (viewport, locale, permissions).
  - `newPage()` — быстрое создание контекста и страницы в одном вызове (удобно для простых сценариев).
  - `contexts()` — возврат списка активных `BrowserContext`.
  - `version()` / `browserType()` — метаданные о запущенном браузере.
  - `close()` — закрытие браузера и всех связанных контекстов.
    ```java
    // Создание контекста с эмуляцией мобильного устройства
    BrowserContext context = browser.newContext(new Browser.NewContextOptions()
        .setViewportSize(375, 812)           // iPhone X resolution
        .setUserAgent("Mobile Safari...")    // Кастомный User-Agent
        .setLocale("ru-RU")                  // Язык и регион
        .setTimezoneId("Europe/Moscow"));    // Часовой пояс
    ```

- `Browser.NewContextOptions` — параметры для изолированной сессии браузера.
  - `setViewportSize(int width, int height)` — размер области просмотра (эмуляция разрешения).
  - `setUserAgent(String)` — переопределение строки пользовательского агента.
  - `setExtraHTTPHeaders(Map<String, String>)` — заголовки, добавляемые ко всем запросам контекста.
  - `setStorageState(Path/StorageState)` — загрузка сохранённых cookies и localStorage (для обхода логина).
  - `setHttpCredentials(String, String)` — базовая HTTP-аутентификация.
  - `setPermissions(List<String>)` — предоставление разрешений (геолокация, уведомления, камера).
  - `setRecordVideoDir(Path)` / `setRecordVideoSize()` — настройка записи видео сессии.
  - `setTracing(TracingOptions)` — конфигурация трассировки для последующего анализа.
    ```java
    // Предзагрузка состояния авторизации из файла (ускоряет тесты)
    BrowserContext context = browser.newContext(new Browser.NewContextOptions()
        .setStorageState(Path.of("auth-state.json")) // Cookies + localStorage
        .setExtraHTTPHeaders(Map.of("X-Test-Id", "checkout-flow")));
    ```

- `BrowserContext` — изолированная сессия браузера: независимые cookies, localStorage, кэш, разрешения. 
  Основной объект для управления тестовой средой.
  - `newPage()` — создание новой страницы (`Page`) в текущем контексте.
  - `pages()` — возврат списка всех открытых страниц в контексте.
  - `cookies()` / `addCookies(List<Cookie>)` — работа с cookies на уровне контекста.
  - `storageState()` — экспорт состояния (cookies, localStorage) в JSON для переиспользования.
  - `route(String pattern, Route.Handler)` — перехват и модификация сетевых запросов (мокинг, блокировка).
    ```java
    // Мокинг внешнего API: возврат фиктивного ответа на запросы к /api/data
    context.route("**/api/data", route -> route.fulfill(
        new Route.FulfillOptions()
            .setStatus(200)
            .setContentType("application/json")
            .setBody("{\"items\":[\"mocked\"]}")));
    ```
  - `waitForConsoleMessage()` — ожидание сообщения в консоли; ловит сообщения со всех страниц контекста, не только с одной Page.  
    ```java
    // Ожидаем конкретное сообщение в консоли (например, лог от приложения)
    ConsoleMessage msg = context.waitForConsoleMessage(() -> {
        page.evaluate("() => console.log('App initialized')"); // Триггер действия (JS)
    });
    
    // Фильтрация по типу или тексту
    ConsoleMessage errorMsg = context.waitForConsoleMessage(m ->
        m.type() == ConsoleMessage.Type.ERROR && m.text().contains("API failed")
    );
    ```
  - `waitForPage()` — ожидание событий уровня контекста (новой страницы/вкладки/попапа); возвращает уже созданную страницу, 
    но она может быть ещё не загружена — добавьте `waitForLoadState()` при необходимости.
    ```java
    // Ожидаем открытие новой вкладки (например, по клику на target="_blank")
    Page newPage = context.waitForPage(() -> {
        page.getByText("Открыть в новой вкладке").click(); // Действие, открывающее вкладку
    }); 
    ```
  - `close()` — закрытие контекста и всех связанных страниц (освобождение памяти).

- `Page` — представление одной вкладки/окна браузера. Основной интерфейс для навигации, взаимодействия с DOM и получения данных.
  - `navigate(String url)` / `reload()` / `goBack()` / `goForward()` — управление историей навигации.
  - `waitForLoadState(LoadState)` / `waitForURL(String)` / `waitForSelector(String)` / `waitForTimeout(double)` — ожидания загрузки и состояния.
  - `locator(String selector)` — получение `Locator` для поиска элементов (ленивая резолюция).
  - `getByRole()`, `getByText()`, `getByTestId()` — стратегии поиска по доступности и тестовым атрибутам.
  - `frameLocator(String selector)` — доступ к `FrameLocator` для работы с iframes.
  - `screenshot()` / `pdf()` / `title()` / `content()` — получение артефактов и метаданных страницы.
  - `onDialog()`, `onRequest()`, `onResponse()`, `onConsole()` — подписка на события браузера.
  - `close()` — закрытие страницы (не закрывает контекст).
    ```java
    // Навигация с ожиданием полной загрузки сети
    page.navigate("https://app.example.com", new Page.NavigateOptions()
        .setWaitUntil(WaitUntilState.NETWORKIDLE)); // Ждём, пока не останется активных запросов
    
    // Обработка нативных диалогов (alert/confirm/prompt)
    page.onDialog(dialog -> {
        System.out.println("Dialog: " + dialog.message());
        dialog.accept(); // или dialog.dismiss()
    });
    ```

- `Locator` — ленивый, авто-ожидающий селектор элемента. Не хранит ссылку на DOM-узел, а пересчитывает его при каждом действии.
  - `click()` / `fill(String)` / `press(String)` / `selectOption()` — действия с автоматической проверкой `actionability`.
  - `isVisible()` / `isEnabled()` / `isEditable()` — проверки состояния без выполнения действия.
  - `first()` / `last()` / `nth(int)` / `filter()` — фильтрация и выбор конкретного элемента из набора.
  - `count()` — возврат количества совпадений (синхронный вызов, требует осторожности).
  - `all()` — материализация всех совпадений в список `Locator` (редко используется из-за потери авто-ожидания).
  - `frameLocator()` — переход во вложенный iframe для дальнейшего поиска.
    ```java
    // Цепочка локаторов: поиск кнопки "Отправить" внутри формы с data-testid="checkout-form"
    var submitBtn = page.locator("[data-testid='checkout-form']")
        .locator("button")                    // Все кнопки внутри формы
        .filter(new Locator.FilterOptions().setHasText("Отправить")) // С текстом "Отправить"
        .first();                             // Берём первое совпадение (strict mode)
    
    submitBtn.click(); // Автоматически дожмётся видимости и кликабельности
    ```

- `FrameLocator` — специализированный локатор для работы с iframes. Позволяет «проникать» во вложенные фреймы без ручного переключения контекста.
  - `locator(String selector)` — получение `Locator` внутри фрейма.
  - `frameLocator(String selector)` — доступ к вложенному iframe (рекурсивно).
    ```java
    // Работа с элементом внутри двухуровневого iframe
    Locator nestedInput = page.frameLocator("#payment-frame")      // Первый уровень
        .frameLocator("#card-details")                             // Второй уровень
        .locator("input[name='cardNumber']");                      // Целевой элемент
    
    nestedInput.fill("4111 1111 1111 1111");
    ```

- `ElementHandle` — низкоуровневая ссылка на конкретный DOM-узел в момент снимка. 
  **Не рекомендуется** для обычной автоматизации (хрупкий, требует ручных ожиданий).
  - Используется только для сложных сценариев: выполнение кастомного JS относительно элемента, работа с Canvas/WebGL.
  - `$()` / `$$()` — поиск потомков относительно этого элемента (устаревший стиль, заменён на `Locator`).
    ```java
    // Пример легитимного использования: выполнение скрипта в контексте элемента
    var handle = page.querySelector("#canvas"); // Получаем handle (не Locator!)
    handle.evaluate(""" 
        (el) => {
            const ctx = el.getContext('2d');
            ctx.fillStyle = '#FF0000';
            ctx.fillRect(0, 0, 100, 100);
        }
    """);
    ```

- `APIRequestContext` — независимый HTTP-клиент для API-тестов или гибридных сценариев (UI + API). 
  Может переиспользовать cookies из `BrowserContext`.
  - `get()` / `post()` / `put()` / `patch()` / `delete()` — отправка запросов с типизированными параметрами.
  - `fetch(String url, RequestOptions)` — универсальный метод с гибкой настройкой (метод, headers, data, timeout).
  - `storageState()` — экспорт состояния авторизации для синхронизации с UI-контекстом.
    ```java
    // Создание API-контекста с теми же cookies, что и у UI-сессии
    	
    APIRequestContext api = context.newAPIRequestContext(new APIRequest.NewContextOptions()
        .setBaseURL("https://api.example.com")
        .setExtraHTTPHeaders(Map.of("X-Request-ID", UUID.randomUUID().toString())));
    
    // POST-запрос с JSON-телом и валидацией ответа
    APIResponse response = api.post("/orders", new RequestOptions()
        .setData(Map.of("productId", 123, "qty", 2))
        .setTimeout(30000));
    
    assertTrue(response.ok());
    JsonObject order = response.json().getAsJsonObject(); // (Gson)
    assertEquals(2, order.get("qty").getAsInt());
    ```

- `APIResponse` — результат выполнения запроса через `APIRequestContext`.
  - `status()` / `statusText()` / `ok()` — информация о статус-коде.
  - `headers()` / `headerValue(String)` — доступ к заголовкам ответа.
  - `text()` / `json()` / `body()` — получение тела ответа в разных форматах.
  - `dispose()` — освобождение ресурсов (закрытие потоков), если ответ не был прочитан полностью.
    ```java
    // Безопасная работа с большим ответом: чтение потока с автоматической очисткой
    try (APIResponse response = api.get("/large-export")) {
        if (response.ok()) {
            try (var stream = response.body()) {
                // Обработка InputStream без загрузки всего файла в память
                processExportStream(stream);
            }
        }
    } // response.dispose() вызван автоматически
    ```

- `PlaywrightAssertions` / `APIResponseAssertions` — типизированные утверждения с авто-ожиданием и понятными сообщениями об ошибках.
  - `expect(locator).isVisible()` / `.toHaveText()` / `.toBeChecked()` / `.toBeEnabled()` — UI-проверки.
  - `expect(apiResponse).isOk()` / `.hasStatus(int)` / `.hasHeader()` — API-проверки.
  - `setOptions().setTimeout(int)` — переопределение таймаута для конкретного утверждения.
    ```java
    // Ожидание появления текста с кастомным таймаутом (5 секунд вместо дефолтных 30)
    expect(page.getByRole(AriaRole.HEADING))
        .toHaveText("Добро пожаловать", 
            new ExpectOptions().setTimeout(5000)); // Локальное переопределение
    ```

- `Tracing` — инструмент для записи детальной трассировки выполнения теста (скриншоты, DOM-снэпшоты, исходный код шагов, сетевые логи).
  - `start(Tracing.StartOptions)` — начало записи с настройками (screenshots, snapshots, sources).
  - `stop(Tracing.StopOptions)` / `stopChunk(Tracing.StopChunkOptions)` — сохранение трейса в файл (`.zip`) через опции.
  - Интегрируется с `context.tracing()` для управления на уровне контекста.
    ```java
    // Включение трассировки только при падении теста (экономия ресурсов)
    @AfterEach
    void afterEach(TestInfo testInfo, BrowserContext context) {
        if (testInfo.getDisplayName().contains("FAILED")) {
            context.tracing().stop(new Tracing.StopOptions()
                .setPath(Path.of("traces/" + testInfo.getDisplayName() + ".zip")));
        }
    }
    ```
- `Request` / `Response` — объекты, представляющие сетевой запрос и ответ (используются в `onRequest`, `onResponse`, `route`).
  - `Request`: `url()` / `method()` / `headers()` / `postData()` / `resourceType()` / `frame()`.
  - `Response`: `request()` / `status()` / `headers()` / `body()` / `finished()` (ожидание завершения загрузки; возвращает: null — если запрос успешно завершён, 
    String с ошибкой — если соединение разорвано/сбой).
    ```java
    // Логирование всех запросов к API с фильтрацией по домену
    page.onResponse(response -> {
        if (response.url().contains("api.example.com")) {
            System.out.printf("[%d] %s %s%n", 
                response.status(), 
                response.request().method(), 
                response.url());
        }
    });
    ```

- `ConsoleMessage` — представление сообщения из консоли браузера (log, warning, error, debug).
  - `type()` / `text()` / `args()` — тип сообщения, текст и аргументы (как `JSHandle`).
  - Используется с `page.onConsoleMessage()` для отладки или валидации отсутствия ошибок в консоли.
    ```java
    // Фиксация ошибок консоли как падения теста
    List<String> errors = new ArrayList<>();
    page.onConsoleMessage(msg -> {
        if (msg.type() == ConsoleMessage.Type.ERROR) {
            errors.add(msg.text());
        }
    });
    // ... после действий:
    assertTrue(errors.isEmpty(), "Console errors: " + errors);
    ```

### Место в стеке AQA

#### Playwright интегрируется в существующую Java-экосистему без необходимости изменения инфраструктуры сборки или CI/CD.

- `Maven/Gradle` - поддержка через стандартные зависимости `com.microsoft.playwright:playwright`, транзитивные зависимости управляются автоматически
- `JUnit 5 / TestNG` - полная совместимость с аннотациями `@BeforeEach`, `@Test`, `@DataProvider`, возможность параллельного запуска
- `Allure / ReportPortal` - встроенная поддержка вложения скриншотов, видео и трейсов через `@Attachment` и кастомные листенеры
- `Docker / Kubernetes` - официальный образ `mcr.microsoft.com/playwright/java` с предустановленными браузерами и системными зависимостями

#### Поддержка современного веба:

- `Shadow DOM` - автоматическое проникновение в теневые деревья без необходимости ручного выполнения JavaScript
- `Динамические SPA` - встроенный `auto-waiting` гарантирует, что действия выполняются только после того, 
  как элемент становится видимым, стабильным и доступным для взаимодействия
- `Iframes` - метод `frameLocator()` позволяет работать с вложенными фреймами так же просто, как с основным контентом
- `Сетевые запросы` - возможность перехвата, модификации и мокирования запросов через `route()` для изоляции тестов от внешних зависимостей

#### Встроенные инструменты отладки:

```java
// Генерация кода через CLI: автоматически создаёт тест по вашим действиям в браузере
// Запуск:
// playwright codegen https://example.com

try (Playwright playwright = Playwright.create()) {
    Browser browser = playwright.chromium().launch();

    // Контекст с записью видео
    BrowserContext context = browser.newContext(
        new Browser.NewContextOptions()
                .setRecordVideoDir(Paths.get("videos"))
    );

    // 🔹 Запуск трассировки
    context.tracing().start(new Tracing.StartOptions()
            .setScreenshots(true)   // скриншоты для превью
            .setSnapshots(true)     // DOM-снэпшоты
            .setSources(true));     // исходный код

    Page page = context.newPage();
    page.navigate("https://example.com");
    // ... ваши действия ...

    // 🔹 Остановка и экспорт
    context.tracing().stop(new Tracing.StopOptions()
            .setPath(Paths.get("trace.zip")));

    browser.close();
}

// Просмотр трейса после выполнения:
// playwright show-trace trace.zip
// это CLI-команда (команда для терминала/командной строки), которая запускает GUI-приложение — Playwright Trace Viewer
```

---

#### Ограничения и требования

- `Java 11+` - минимальная версия, необходимая для работы библиотеки (используются записи, модульная система)
- `ОС-зависимости` - для headless-режима в Linux требуются системные библиотеки: `libatk-1.0`, `libnss3`, `libxkbcommon`, 
  `fonts-liberation` (автоматически устанавливаются в оф. Docker-образах)
- `Дисковое пространство` - кэш браузеров занимает ~1.5 ГБ (Chromium + Firefox + WebKit), настраивается через `PLAYWRIGHT_BROWSERS_PATH`
- `Лицензирование` - Apache 2.0, бесплатное использование в коммерческих и открытых проектах, 
  поддержка осуществляется через GitHub Issues и сообщество

#### Таблица поддержки браузеров:

| Браузер    | Поддерживаемые ОС     | Headless | Headed            | Примечания                                                |
|:-----------|-----------------------|----------|-------------------|-----------------------------------------------------------|
| `Chromium` | Windows, macOS, Linux | ✅        | ✅                 | Полная поддержка CDP, лучший перформанс                   |
| `Firefox`  | Windows, macOS, Linux | ✅        | ✅                 | Поддержка через Firefox DevTools Protocol                 |
| `WebKit`   | macOS, Linux          | ✅        | ⚠️ (только Linux) | На Windows не поддерживается, для тестов Safari-поведения |

---

## Best Practices

- **Используйте `BrowserContext` для изоляции тестов** - каждый тест должен запускаться в новом контексте, 
  чтобы избежать загрязнения состояния между сценариями
- **Предпочитайте `Locator` вместо `ElementHandle`** - ленивая резолюция и автоматическое ожидание делают тесты стабильнее и короче
- **Включайте трассировку только при падениях** - используйте условную запись `tracing.start()` в `@AfterEach` при `test.failed()`, 
  чтобы не замедлять успешные прогоны
- **Не переиспользуйте `Browser` между параллельными тестами без изоляции** - создавайте новый `BrowserContext` на каждый тест, 
  даже если `Browser` общий
- **Фиксируйте версии браузеров в CI** - используйте `playwright install chromium@1234` для детерминированного поведения, 
  избегайте автообновлений
- **Избегайте `Thread.sleep()`** - используйте встроенные ожидания `waitForLoadState()`, `locator().isVisible()` или `expect(locator).toBeVisible()`
- **Не хардкодьте пути к браузерам** - используйте переменную окружения `PLAYWRIGHT_BROWSERS_PATH` или кэш по умолчанию 
  для переносимости между окружениями
- **Не игнорируйте возвратные коды сетевых запросов** - при мокировании через `route()` всегда обрабатывайте `abort()` и `fulfill()` явно, 
  чтобы избежать «зависших» тестов

---

# 2. Подключение и конфигурация окружения

> Подключение Playwright в Java-проект требует добавления единственной артефактной зависимости, 
> после чего драйвер автоматически управляет загрузкой браузеров, проверкой системных библиотек и маршрутизацией нативных вызовов. 
> 
> Гибкая конфигурация через переменные окружения, JVM-свойства и CLI позволяет адаптировать окружение под локальную разработку 
> и изолированные CI-агенты без изменения бизнес-логики тестов.

## Зависимости

- `com.microsoft.playwright:playwright` - основная зависимость, включающая клиентскую библиотеку, драйвер-обёртку 
  и автоматический менеджер браузеров
- `net.java.dev.jna:jna` - транзитивная зависимость для нативного доступа к системным API (создание сокетов, управление процессами драйвера)
- `org.opentest4j:opentest4j` - стандартная библиотека утверждений, обеспечивающая совместимость с JUnit 5 и другими раннерами

Maven (`pom.xml`):

```xml
<dependencies>
    <!-- Основная зависимость Playwright для Java -->
    <dependency>
        <groupId>com.microsoft.playwright</groupId>
        <artifactId>playwright</artifactId>
        <version>1.44.0</version>  <!-- Фиксируем версию для стабильности CI -->
        <scope>test</scope>        <!-- Доступна только на этапе тестирования -->
    </dependency>
    
    <!-- JUnit 5 для организации тестовых сьютов -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <!-- Передача системных свойств для отладки драйвера -->
                <systemPropertyVariables>
                    <playwright.debug>false</playwright.debug>
                </systemPropertyVariables>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Gradle (`build.gradle`):

```groovy
dependencies {
    // Основная зависимость Playwright
    testImplementation 'com.microsoft.playwright:playwright:1.44.0'
    
    // JUnit 5 Engine и API
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

test {
    // Настройка использования JUnit 5
    useJUnitPlatform()
    
    // Передача JVM-аргументов для обхода валидации хост-требований в CI
    jvmArgs '-DPLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS=true'
}
```

---

## Установка браузеров

- `playwright install` - CLI-команда для загрузки бинарных файлов браузеров Chromium, Firefox и WebKit в системный кэш
- `playwright install <browser>` - установка конкретного браузера (`chromium`, `firefox`, `webkit`) для экономии трафика и места
- `PLAYWRIGHT_BROWSERS_PATH` - переменная окружения, переопределяющая стандартный путь кэша (`~/.cache/ms-playwright`)
- `--with-deps` - флаг для автоматической установки недостающих системных библиотек Linux (только для Debian/Ubuntu-based образов)

Локальная установка:

```bash
# Установка всех поддерживаемых браузеров с проверкой системных зависимостей
npx playwright install --with-deps

# Установка только Chromium для быстрой настройки локального окружения
npx playwright install chromium

# Проверка пути к кэшу (возвращает директорию, куда будут сохранены браузеры)
echo $PLAYWRIGHT_BROWSERS_PATH
```

CI-агенты и offline-режим:

```bash
# 1. Предварительная загрузка браузеров в артефакт или Docker-слой
npx playwright install --with-deps

# 2. Установка переменной кэша для изоляции пайплайна
export PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers

# 3. Запуск тестов без обращения к сети (браузеры берутся из кэша)
# При отсутствии кэша драйвер бросит RuntimeException, а не пытается скачать
npx playwright test --headed --browser chromium
```

#### Схема процесса инициализации драйвера:

```
┌─────────────────────────────────────────────────────────────┐
│                    JVM (Java Test)                          │
│                                                             │
│  com.microsoft.playwright.Playwright.create()               │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │  Driver Process │ ← Запускается автоматически при первом │
│  │    (Node.js)    │    вызове API. Проверяет кэш браузеров │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Браузеры (~/.cache/ms-playwright/)                 │    │
│  │  ├── chromium-.../chrome                            │    │
│  │  ├── firefox-.../firefox                            │    │
│  │  └── webkit-.../minibrowser                         │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Глобальная конфигурация

- `DEBUG=pw:api` - включение детального логирования всех вызовов Playwright API (полезно для анализа флаков)
- `DEBUG=pw:browser` - логирование внутренних событий браузера (загрузка страниц, сетевые запросы, события DOM)
- `PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS` - пропуск проверки системных зависимостей (ускоряет старт в Docker, 
  но может скрыть missing libs)
- `PLAYWRIGHT_BROWSERS_PATH` - переопределение директории кэша браузеров
- `PLAYWRIGHT_TRACE` - автоматическое включение трассировки (`on`, `off`, `retain-on-failure`)

### Настройка через переменные окружения и JVM-свойства:

```bash
# Linux/macOS: экспорт переменных перед запуском тестов
export DEBUG=pw:api,pw:browser
export PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS=true
export PLAYWRIGHT_BROWSERS_PATH=/custom/cache/path

# Запуск Maven с передачей переменных в JVM
mvn test -DDEBUG=pw:api -Dplaywright.trace=retain-on-failure
```

Программная конфигурация логирования драйвера:

```java
public class DebugConfigExample {
    
    public static void main(String[] args) {
        // Инициализация Playwright с включённым API-логированием
        // Логи будут выводиться в stderr в реальном времени
        try (Playwright playwright = Playwright.create()) {
            
            // Настройка параметров запуска Chromium
            BrowserType.LaunchOptions launchOptions = new BrowserType.LaunchOptions()
                .setHeadless(true)                    // Headless режим для CI
                .setSlowMo(50)                        // Искусственная задержка 50мс для отладки — это намеренное замедление выполнения каждого действия Playwright на указанное количество миллисекунд
                .setArgs(List.of("--disable-gpu"));   // Обход проблем с рендерингом в VM; заставляет Chromium не пытаться использовать аппаратное ускорение GPU, а работать полностью в программном режиме (software rendering)
            
            // Запуск браузера с применением параметров
            var browser = playwright.chromium().launch(launchOptions);
            
            // Все последующие действия будут логироваться, если DEBUG=pw:api установлен
            var page = browser.newPage();
            page.navigate("https://example.com");
            
            browser.close();
        }
    }
}
```

---

## Настройка IDE и тулинга

- `Playwright Test` - плагин для IntelliJ IDEA, предоставляющий подсветку синтаксиса, навигацию по Locator и интеграцию с кодогенератором
- `playwright codegen` - CLI-утилита для записи действий пользователя и автоматической генерации Java-кода
- `IntelliJ Run Configuration` - настройка кастомных конфигураций запуска с передачей env-переменных
- `Spotless / Checkstyle` - линтеры для поддержания единого стиля кода в Java-тестах

## Генерация кода в Playwright (Codegen)

`Playwright Codegen` — это инструмент, который записывает ваши действия в браузере и автоматически превращает их 
в готовый код теста на нужном языке. По сути, это мост между ручным тестированием и автоматизацией.

---

### Как это работает (по шагам)

Когда вы запускаете команду:
```bash
playwright codegen --target java https://target-app.com/login
```

Происходит следующее:

1. **Открывается два окна:**
  - **Окно браузера** — управляемое окно, где вы выполняете действия как пользователь.
  - **Окно `Playwright Inspector`** — панель инструментов, где в реальном времени отображается сгенерированный код 
    и дополнительная информация.

2. **Вы действуете как пользователь:** кликаете по полям, вводите текст, нажимаете кнопки, перемещаетесь между страницами.

3. **Codegen перехватывает каждое действие** и превращает его в строку кода. Он не просто записывает координаты клика, 
    а анализирует DOM и подбирает наиболее надёжные локаторы.

4. **Код появляется в реальном времени** в панели Inspector. Вы можете тут же его скопировать и вставить в свой проект.

---

### Какие действия Codegen умеет записывать

| Действие пользователя       | Сгенерированный код                           |
|:----------------------------|-----------------------------------------------|
| Клик по элементу            | `page.getByRole(...).click()`                 |
| Ввод текста                 | `page.getByPlaceholder(...).fill("текст")`    |
| Нажатие клавиши             | `page.locator(...).press("Enter")`            |
| Навигация по URL            | `page.navigate("https://...")`                |
| Выбор из выпадающего списка | `page.getByLabel(...).selectOption("value")`  |
| Работа с чекбоксами/радио   | `page.getByLabel(...).check()` / `.uncheck()` |
| Drag-and-drop               | `page.locator(...).dragTo(page.locator(...))` |

---

### Приоритет выбора локатора у Codegen

> Playwright отдаёт предпочтение использованию не XPath и не CSS-селекторам, а семантическим локаторам. Это осознанная стратегия.

1. **`data-testid`** — если у элемента есть атрибут `data-testid`, он используется в первую очередь. Это самый стабильный локатор, 
  потому что он не зависит ни от текста, ни от вёрстки.
2. **Ролевые локаторы (`getByRole`)** — Codegen анализирует accessibility-дерево (ARIA) и определяет роль элемента (button, heading, link, textbox).
3. **Текстовые локаторы (`getByText`, `getByLabel`)** — если есть видимый текст или связанный `<label>`.
4. **Placeholder** — для полей ввода с атрибутом `placeholder`.
5. **CSS/XPath** — используется только в крайнем случае, когда ничего из вышеперечисленного недоступно.

---

### Флаги и опции кодогенератора

```bash
# Целевой язык генерации (Java, Python, C#, TypeScript/JavaScript)
playwright codegen --target java https://example.com

# Сохранить сгенерированный код сразу в файл
playwright codegen --target java --output LoginTest.java https://example.com

# Эмулировать определённое устройство (мобильный вид)
playwright codegen --device "iPhone 13" https://example.com

# Подставить заранее сохранённое состояние (куки, localStorage)
playwright codegen --load-storage=auth.json https://example.com

# Показать цветовую дифференциацию действий в инспекторе
playwright codegen --color-scheme dark https://example.com
```

---

### Практические сценарии использования кодогенератора для QA/AQA

- **Быстрое прототипирование теста**

  Manual QA проходит новый функционал, а на выходе получает готовую болванку автотеста. 
  
  Не нужно вручную писать `page.navigate(...)`, `page.click(...)` — код уже сгенерирован, осталось только добавить assertion'ы (`assertThat(...)`).

- **Поиск надёжных локаторов**

  Вы открываете codegen, наводите курсор на проблемный элемент (который сложно залокать) и в панели Inspector 
  нажимаете кнопку **Explore**. 

  Codegen показывает все доступные варианты локаторов для этого элемента и их уникальность.

- **Запись аутентификации**

  ```bash
  # Записываем процесс логина и сохраняем состояние
  playwright codegen --save-storage=auth.json https://app.example.com/login
  ```
  После ручного ввода логина/пароля состояние сохраняется в файл, который потом можно переиспользовать в тестах, 
  чтобы не проходить аутентификацию каждый раз.

- **Отладка локаторов в CI**

  Если тест упал с ошибкой «element not found», вы можете запустить codegen на том же URL и вручную проверить, 
  видит ли Playwright этот элемент и какие локаторы для него доступны.

---

### Настройка IntelliJ Run Configuration для отладки:

1. `Run` → `Edit Configurations` → `+` → `Application`
2. `Main class`: `org.junit.platform.console.ConsoleLauncher` (или ваш тест-раннер)
3. `Environment variables`: `DEBUG=pw:api;PLAYWRIGHT_BROWSERS_PATH=~/.cache/ms-playwright`
4. `VM options`: `-ea` (включение assertions для JUnit)
5. `Modify options` → `Add before launch task` → `Run Maven Goal` → `test-compile`

---

## Best Practices

- **Фиксируйте версии в `pom.xml` / `build.gradle`** - избегайте `LATEST` или диапазонов версий, чтобы гарантировать воспроизводимость сборок в CI
- **Используйте `--with-deps` только при первом развертывании** - в CI кэшируйте образ Docker с уже установленными браузерами, чтобы не замедлять каждый запуск
- **Настройте `retain-on-failure` для трейсов** - автоматически сохраняет артефакты только при падениях, экономя дисковое пространство на успешных прогонах
- **Отделяйте `test` scope от `compile`** - Playwright не должен попадать в production-артефакт, используйте `<scope>test</scope>` или `testImplementation`
- **Используйте `.env` файлы локально** - выносите `PLAYWRIGHT_BROWSERS_PATH` и `DEBUG` в `.env`, добавляйте его в `.gitignore`, чтобы не коммитить специфичные настройки
- **Не пропускайте валидацию зависимостей в production-подобном окружении** - `PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS=true` маскирует отсутствие `libnss3` или шрифтов, 
  приводя к молчаливым падениям рендеринга
- **Не храните браузеры в `src/main/resources`** - кэш Playwright весит >1.5 ГБ, размещайте его в системной директории или монтируйте как volume в Docker
- **Не игнорируйте `slowMo` при отладке сетевых лагов** - искусственная задержка помогает визуализировать timing-проблемы, 
  но никогда не коммитьте её в основную ветку

---

# 3. Ядро API: Браузер, Контекст, Страница

> Ядро Playwright построено на строгой иерархии объектов, где каждый уровень отвечает за определённую грань управления браузером: 
> от запуска процесса до взаимодействия с отдельным элементом DOM. 

## Иерархия объектов и управление ресурсами

`Playwright` → `BrowserType` → `Browser` → `BrowserContext` → `Page` → `Frame`/`Locator` — цепочка владения, 
где каждый объект создаётся родительским и должен закрываться в обратном порядке для предотвращения утечек памяти.

- `AutoCloseable` — все ключевые классы (`Playwright`, `Browser`, `BrowserContext`, `Page`) реализуют этот интерфейс, 
  что позволяет использовать `try-with-resources` для гарантированной очистки
- `try-with-resources` — предпочтительный паттерн инициализации, обеспечивающий вызов `close()` даже при исключениях
- `ThreadLocal<T>` — рекомендуется для хранения `Page`/`BrowserContext` при параллельном выполнении, 
  чтобы избежать пересечения состояний между потоками

```
┌─────────────────────────────────────────────────────────┐
│                       Playwright                        │
│            (точка входа, фабрика BrowserType)           │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                      BrowserType                        │
│              (chromium / firefox / webkit)              │
│  → launch() / connect() → Browser                       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                        Browser                          │
│              (запущенный процесс браузера)              │
│  → newContext() → BrowserContext                        │
│  → newPage() → Page (быстрый путь)                      │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     BrowserContext                      │
│  (изолированная сессия: cookies, localStorage, cache)   │
│  → newPage() → Page                                     │
│  → route() → перехват запросов                          │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                          Page                           │
│      (одна вкладка/окно: навигация, DOM, события)       │
│  → locator() / getBy*() → Locator                       │
│  → frameLocator() → FrameLocator                        │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                 Locator / FrameLocator                  │
│      (ленивый селектор элемента / доступ к iframe)      │
│  → click() / fill() / isVisible() → действия            │
└─────────────────────────────────────────────────────────┘
```

---

## Browser и BrowserType: запуск и подключение

### `BrowserType` — фабрика для получения экземпляров `Browser`, абстрагирующая различия между движками (Chromium, Firefox, WebKit)

- `launch(LaunchOptions)` — запуск локального браузера с полным контролем над параметрами
- `connect(String wsEndpoint)` — подключение к удалённому браузеру по WebSocket (Browserless, Selenium Grid, CDP)
- `executablePath()` — возвращает абсолютный путь к бинарному файлу браузера, который Playwright будет использовать по умолчанию 
  при запуске через launch(). (для отладки или кастомных сборок)
- `name()` — строковое имя типа (`chromium`, `firefox`, `webkit`)

### `LaunchOptions` — конфигурация запуска, влияющая на поведение браузера:

- `setHeadless(boolean)` — режим без графического интерфейса (по умолчанию `true`, обязательно для CI)
- `setSlowMo(int)` — искусственная задержка в миллисекундах между действиями (для визуальной отладки)
- `setArgs(List<String>)` — флаги командной строки браузера (`--disable-gpu`, `--start-maximized`, `--no-sandbox`)
- `setDownloadsPath(Path)` — директория для сохранения загружаемых файлов
- `setProxy(Proxy)` — настройка прокси-сервера для всех запросов браузера
- `setChannel(String)` — выбор конкретного канала (`chrome`, `msedge`, `firefox-beta`, `chromium`)
- `setTimeout(int)` — глобальный таймаут операций для этого браузера (в миллисекундах)

Пример запуска с расширенной конфигурацией:

```java
// Настройка параметров запуска Chromium
BrowserType.LaunchOptions options = new BrowserType.LaunchOptions()
    .setHeadless(true)                                     // CI-режим
    .setSlowMo(0)                                          // Без задержек в продакшене
    .setArgs(List.of(
        "--disable-dev-shm-usage",                         // Обход проблем с /dev/shm в Docker
        "--disable-gpu",                                   // Отключение GPU-рендеринга для стабильности
        "--no-sandbox",                                    // Требуется в некоторых CI-окружениях
        "--disable-setuid-sandbox"                         // Дополнительный флаг для контейнеров
    ))
    .setDownloadsPath(Path.of("target/downloads"))         // Явный путь для артефактов
    .setProxy(new Proxy()
        .setServer("http://proxy.internal:8080")           // Прокси-сервер
        .setBypass("localhost,*.internal"));               // Исключения для локальных адресов

// Запуск браузера с применением конфигурации
try (Browser browser = playwright.chromium().launch(options)) {
    // Браузер готов к созданию контекстов
}
```

### Подключение к удалённому браузеру (Browserless / CDP):

| Сценарий                                         | Метод              | Где выполняются тесты | Где работает браузер                              | Работа с файлами                                                                         | Пример работы с файлами                                                                              | Пример кода                                                                                                            |
|--------------------------------------------------|--------------------|-----------------------|---------------------------------------------------|------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| **Локальная отладка**                            | `launch()`         | Локально (JVM)        | Локально (процесс ОС)                             | Прямой доступ: `Paths.get("screenshots")`, загрузка файлов через `<input>`               | `page.locator("input").setFiles(Paths.get("test.pdf"));`                                             | `browser = playwright.chromium().launch(new LaunchOptions().setHeadless(false));`                                      |
| **Локальные тесты + браузер в Docker**           | `connect()`        | Локально (JVM)        | В Docker-контейнере (через WebSocket)             | Файлы должны быть доступны в контейнере: либо монтирование `-v`, либо передача через API | `page.locator("input").setFiles(Paths.get("/container/uploads/test.pdf"));` (путь внутри контейнера) | `browser = playwright.chromium().connect("ws://localhost:3000/");`                                                     |
| **Тесты + браузер в Docker (CI/CD)**             | `launch()`         | В Docker-контейнере   | В том же контейнере (локально для процесса)       | Прямой доступ внутри контейнера; артефакты выносятся через volume/bind-mount             | `page.locator("input").setFiles(Paths.get("/app/test-data/file.png"));` + `-v /host:/app`            | `browser = playwright.chromium().launch(new LaunchOptions().setHeadless(true));`                                       |
| **Удалённый браузер (BrowserStack, LambdaTest)** | `connect()`        | Локально или в CI     | Удалённый сервер (SaaS)                           | Файлы нужно передавать через API (base64, URL) или предварительно загружать в облако     | `page.locator("input").setInputFiles("test.pdf");` через `FormData` или base64-кодирование           | `browser = playwright.chromium().connect("wss://cloud-provider/ws-endpoint", new ConnectOptions().setTimeout(30000));` |
| **Отладка через CDP (Chromium)**                 | `connectOverCDP()` | Локально              | Существующий Chromium с `--remote-debugging-port` | Как при локальном запуске, но с ограничениями протокола                                  | `page.locator("input").setFiles(Paths.get("local/file.png"));`                                       | `browser = playwright.chromium().connectOverCDP("http://localhost:9222");`                                             |

```java
// Подключение к Browserless.io через WebSocket
String wsEndpoint = "wss://chrome.browserless.io?token=YOUR_TOKEN";

try (Browser browser = playwright.chromium().connect(wsEndpoint)) {
    // Все действия выполняются на удалённой машине
    // Важно: локальные файлы (downloadsPath) должны быть доступны удалённо
    Page page = browser.newPage();
    page.navigate("https://app.example.com");
}
```

---

## BrowserContext: изоляция сессий и эмуляция

> `BrowserContext` — изолированная сессия браузера с независимыми `cookies`, `localStorage`, `IndexedDB`, `permissions`и `cache`. 
> 
> Каждый тест должен использовать отдельный контекст для гарантии чистоты состояния.

### `Browser.NewContextOptions` — параметры конфигурации контекста:

- `setViewportSize(int width, int height)` — эмуляция разрешения экрана (например, `375, 812` для iPhone)
- `setUserAgent(String)` — переопределение строки пользовательского агента (для тестов мобильных версий)
- `setExtraHTTPHeaders(Map<String, String>)` — заголовки, добавляемые ко всем запросам контекста
- `setStorageState(Path/StorageState)` — загрузка сохранённого состояния авторизации (cookies + localStorage)
- `setHttpCredentials(String username, String password)` — базовая HTTP-аутентификация
- `setPermissions(List<String>)` — предоставление разрешений (`geolocation`, `notifications`, `camera`)
- `setGeolocation(double latitude, double longitude)` — эмуляция местоположения
- `setLocale(String)` / `setTimezoneId(String)` — локаль и часовой пояс для форматирования дат/чисел
- `setRecordVideoDir(Path)` / `setRecordVideoSize(ViewportSize)` — настройка записи видео сессии
- `setTracing(TracingOptions)` — конфигурация трассировки для последующего анализа

#### Пример создания контекста с эмуляцией мобильного устройства:

```java
// Конфигурация для эмуляции iPhone 12 Pro
Browser.NewContextOptions mobileOptions = new Browser.NewContextOptions()
    .setViewportSize(390, 844)                                  // Разрешение экрана
    .setUserAgent("Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X)...") // Мобильный UA
    .setLocale("ru-RU")                                         // Язык и регион
    .setTimezoneId("Europe/Moscow")                             // Часовой пояс
    .setGeolocation(55.7558, 37.6173)                           // Координаты Москвы
    .setPermissions(List.of("geolocation"))                     // Разрешение на доступ к геолокации
    .setExtraHTTPHeaders(Map.of(                                // Кастомные заголовки
        "X-Test-Session", UUID.randomUUID().toString(),
        "X-Device-Type", "mobile"
    ));

// Создание контекста и страницы
try (BrowserContext context = browser.newContext(mobileOptions); 
     Page page = context.newPage()) {
         page.navigate("https://m.example.com");
         // Тест будет выполняться в эмулированной мобильной среде
}

```

#### Загрузка состояния авторизации для ускорения тестов:

```java
// 1. Сохранение состояния после логина (выполняется один раз)
// context.storageState(new BrowserContext.StorageStateOptions()
//     .setPath(Path.of("auth-state.json")));

// 2. Загрузка состояния в каждом тесте (пропуск экрана логина)
Browser.NewContextOptions preAuthOptions = new Browser.NewContextOptions()
    .setStorageState(Path.of("auth-state.json"))               // Cookies + localStorage
    .setExtraHTTPHeaders(Map.of("X-Test-Mode", "true"));       // Доп. заголовок для бэкенда

try (BrowserContext context = browser.newContext(preAuthOptions);
     Page page = context.newPage()) {
        page.navigate("https://app.example.com/dashboard");    // Прямой переход в приложение
        // Пользователь уже авторизован
}

```

---

## Page: навигация, ожидания и обработка событий

> `Page` — представление одной вкладки или окна браузера. 
> 
> Основной интерфейс для навигации, взаимодействия с DOM, получения артефактов и подписки на события.

#### Методы навигации:

- `navigate(String url)` / `navigate(String url, NavigateOptions)` — переход по URL с настройками ожидания
- `reload()` / `goBack()` / `goForward()` — управление историей браузера
- `url()` / `title()` / `content()` — получение метаданных текущей страницы

#### Ожидания загрузки и состояния:

- `waitForLoadState(LoadState)` — ожидание состояния загрузки
  - `LoadState.DOMCONTENTLOADED`
    Состояние, когда HTML-документ полностью загружен и разобран, построено DOM-дерево. 
    При этом внешние ресурсы (стили, изображения, iframe) могут всё ещё загружаться. 
    Соответствует событию браузера `DOMContentLoaded`.
  - `LoadState.LOAD`
    Состояние полной загрузки страницы, включая все зависимые ресурсы: изображения, стили, скрипты, iframe и т.д. 
    Это значение используется по умолчанию. Соответствует событию браузера `window.onload`.
  - `LoadState.NETWORKIDLE`
    Состояние, когда в течение как минимум 500 мс нет исходящих сетевых соединений. 
    Полезно для одностраничных приложений (SPA) с асинхронными запросами. 
    **Важно:** официальная документация Playwright не рекомендует использовать это состояние для тестирования, 
    советуя вместо него применять проверки на наличие конкретных элементов.
- `waitForURL(String pattern)` — ожидание перехода на целевой URL (поддерживает глобы (glob) - упрощенные шаблоны для сопоставления строк с использованием wildcard-символов)
- `waitForTimeout(int milliseconds)` — явная задержка (использовать только при отсутствии альтернатив)
- `waitForFunction(String script)` — ожидание выполнения произвольного JavaScript-условия

#### Обработка событий:

- `onDialog(Dialog.Handler)` — перехват нативных диалогов (`alert`, `confirm`, `prompt`)
- `onRequest(Request.Handler)` / `onResponse(Response.Handler)` — мониторинг сетевых запросов
- `onConsoleMessage(ConsoleMessage.Handler)` — логирование сообщений консоли браузера
- `onPageError(Consumer<PageError>)` — обработка неотловленных ошибок JavaScript

#### Пример навигации с ожиданием полной загрузки сети:

```java
// Переход на страницу с ожиданием отсутствия активных сетевых запросов
page.navigate("https://app.example.com/checkout", new Page.NavigateOptions()
    .setWaitUntil(WaitUntilState.NETWORKIDLE)    // Ждём, пока не завершатся все запросы
    .setTimeout(60000));                         // Глобальный таймаут 60 секунд

// Альтернатива: ожидание конкретного URL после редиректа
page.waitForURL("**/checkout/confirmation", new Page.WaitForURLOptions()
    .setTimeout(30000));                         // Локальный таймаут для этого ожидания
```

#### Обработка диалогов и консоли:

```java
// Обработка нативных диалогов: автоматическое подтверждение с вводом текста в prompt
page.onDialog(dialog -> {
    // dialog.type() — возвращает: ALERT, CONFIRM или PROMPT
    // dialog.message() — текст сообщения из диалога
    
    System.out.println("Dialog type: " + dialog.type() + ", message: " + dialog.message());
    
    if (dialog.type() == DialogType.PROMPT) {
        dialog.accept("Значение по умолчанию"); // Ввод текста в prompt
    } else {
        dialog.accept();                        // Просто подтвердить alert/confirm
    }
});
```

- Без обработки диалоги блокируют выполнение теста
- Обработчик должен быть зарегистрирован ДО того, как диалог появится
- Метод `accept()` подтверждает, `dismiss()` — отклоняет (как нажатие Cancel)

```java
// Фиксация ошибок консоли как потенциальных дефектов
List<String> consoleErrors = new ArrayList<>();
page.onConsoleMessage(msg -> {
    if (msg.type() == ConsoleMessage.Type.ERROR) {
        consoleErrors.add(String.format("[%s] %s", 
            msg.location().url(), 
            msg.text()));
    }
});

// ... выполнение действий ...
page.getByRole(AriaRole.BUTTON).click();

// Валидация отсутствия ошибок в консоли после сценария
assertTrue(consoleErrors.isEmpty(), 
    "Console errors detected: " + String.join("; ", consoleErrors));
```

#### Кастомные таймауты на уровне страницы:

```java
// Установка глобального таймаута для всех операций на этой странице (в миллисекундах)
page.setDefaultTimeout(45000); // 45 секунд вместо дефолтных 30

// Переопределение таймаута для конкретного действия
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Сохранить"))
    .click(new Locator.ClickOptions().setTimeout(10000)); // 10 секунд только для этого клика
```

---

## Best Practices

- **Используйте `BrowserContext` для изоляции тестов** — каждый тест должен запускаться в новом контексте, 
  чтобы избежать загрязнения `cookies`/`localStorage` между сценариями
- **Предпочитайте `newContext()` вместо `newPage()` на уровне `Browser`** — явное создание контекста 
  даёт больше контроля над конфигурацией (эмуляция, мокинг, трассировка)
- **Закрывайте ресурсы в обратном порядке** — `Page` → `Context` → `Browser` → `Playwright`, 
  используйте `try-with-resources` для гарантии
- **Настраивайте `viewport` и `userAgent` для кросс-девайсных тестов** — эмуляция мобильных устройств через `NewContextOptions` надёжнее, 
  чем изменение размера окна постфактум
- **Экспортируйте `storageState` после логина** — однократная авторизация с сохранением состояния 
  ускоряет последующие тесты на 30–50%
- **Не переиспользуйте `BrowserContext` между тестами** — даже если браузер один, 
  контекст должен быть уникальным для каждого теста (`@BeforeEach` → `newContext()`)
- **Не используйте `waitForTimeout()` как основное ожидание** — предпочитайте семантические ожидания: 
  `waitForURL()`, `locator().isVisible()`, `expect().toBeVisible()`
- **Не игнорируйте возвратные значения событий** — при подписке на `onRequest`/`onResponse` всегда обрабатывайте исключения 
  внутри хендлера, чтобы не «уронить» весь тест

---

# 4. Селекторы и Locator API

> Playwright предлагает мощную и гибкую систему поиска элементов, построенную вокруг ленивого `Locator` API. 
> 
> В отличие от Selenium, где поиск возвращает статический `WebElement`, Playwright пересчитывает селектор перед каждым
> действием, автоматически ожидая его доступности. 
> 
> Приоритет отдаётся семантическим стратегиям (`getByRole`, `getByText`), устойчивым к изменениям вёрстки и рефакторингу.

## Стратегии выбора элементов

> Playwright поддерживает множество стратегий поиска, от классических CSS/XPath до семантических и кастомных атрибутов. 
> 
> Выбор стратегии напрямую влияет на стабильность и скорость тестов.

- `CSS Selectors` — стандартные селекторы, быстрые и нативные для браузеров. 
  Поддерживают современные псевдо-классы (`:nth-child`, `:is()`, `:where()`).
  ```java
  // Playwright в первую очередь оптимизирован под CSS-селекторы (они нативные и быстрые)
  // CSS (префикс не нужен)
  page.locator("button.submit-btn");           // автоматически CSS
  page.locator("css=button.submit-btn");       // можно явно, но избыточно
  ```
- `XPath` — поддерживаются через `xpath=` префикс, но считаются менее производительными. 
  Используйте только когда CSS недостаточен или структура DOM не поддаётся CSS-логике.
  ```java
  // XPath (префикс обязателен)
  page.locator("xpath=//button[contains(text(),'Submit')]");  // нужно xpath=
  ```
- `Text Selector` — поиск по точному или частичному совпадению текста. 
  Поддерживает регистр и регулярные выражения: `text="Submit"`, `text=/Submit/i`.
  ```java
  // Text Selector
  page.locator("text=/^Submit$/").click(); // точное совпадение: <button>Submit</button>
  page.locator("text=Submit").click(); // подстрока с учётом регистра: <button>Submit form</button>, <div>Unsubmitted</div>
  page.locator("text=/Submit/").click(); // подстрока с учётом регистра черес regex: <button>Submit form</button>, <div>Unsubmitted</div>
  page.locator("text=/submit/i").click(); // подстрока без учёта регистра: <button>Submit</button>, <div>submit form</div>, <span>SUBMITTED</span>
  ```
- `Role Selector` — поиск по ARIA-ролям и доступным именам. 
  Самый устойчивый к рефакторингу подход: `role="button"`, `role="textbox"`.
  ```java
  // Role Selector
  page.locator("role=button[name='Submit']").click();
  page.locator("role=textbox").fill("test");
  ```
- `Test-ID` — поиск по кастомному атрибуту (по умолчанию `data-testid`, но вы можете переопределить, 
  какой атрибут использовать для этого сокращения). 
  Идеален для CI/CD, SPA и динамических приложений.
  ```java
  // Test-ID
  page.locator("data-testid=submit-button").click();
  ```
- `Visibility Pseudo-selectors` — `:visible` фильтрует элементы, скрытые через `display: none` или `visibility: hidden`. 
  Playwright по умолчанию ищет только видимые элементы.
  ```java
  // Visibility Pseudo-selectors
  page.locator("button:visible").click();
  ```
- `Combinators` — `:has()`, `:has-text()`, `:scope` для контекстного поиска и фильтрации по содержимому или родительским блокам.
  ```java
  // Combinators
  page.locator("div:has(button)").click();
  page.locator("article:has-text('Read more')").click();
  ```

### Таблица сравнения стратегий:

| Стратегия          | Синтаксис (Java)                       | Устойчивость | Производительность | Когда использовать                                |
|:-------------------|----------------------------------------|--------------|--------------------|---------------------------------------------------|
| `getByRole()`      | `page.getByRole(AriaRole.BUTTON)`      | ⭐⭐⭐⭐⭐        | Высокая            | Кнопки, ссылки, заголовки, поля форм              |
| `getByText()`      | `page.getByText("Welcome")`            | ⭐⭐⭐⭐         | Высокая            | Текст, надписи, динамические сообщения            |
| `getByTestId()`    | `page.getByTestId("submit-btn")`       | ⭐⭐⭐⭐⭐        | Максимальная       | Тестовые окружения, частые рефакторинги           |
| `getByLabel()`     | `page.getByLabel("Email address")`     | ⭐⭐⭐⭐         | Высокая            | Поля ввода с привязанным `<label>`                |
| `locator("css")`   | `page.locator(".btn-primary")`         | ⭐⭐           | Высокая            | Сложные CSS-паттерны, legacy-код                  |
| `locator("xpath")` | `page.locator("xpath=//div[@id='x']")` | ⭐            | Низкая             | Крайние случаи, парсинг неструктурированного HTML |

### Кастомные псевдоклассы (Playwright-specific)

#### Примеры использования псевдо-селекторов:

```java
// :has() — поиск родителя, содержащего определённый дочерний элемент
page.locator("article:has(span.status-active)")
    .getByRole(AriaRole.BUTTON)
    .click();

// :has-text() — фильтрация по тексту внутри элемента
page.locator("div.card:has-text('Premium')")
    .getByRole(AriaRole.LINK)
    .click();

// :visible — явное требование видимости (Playwright применяет его по умолчанию)
page.locator(".overlay:visible").click();

// :scope — ссылка на корневой элемент текущего контекста поиска (полезно в цепочках)
page.locator("#form :scope > input[type='submit']").click();
```

---

## Locator API: ленивая резолюция и strict mode

> `Locator` — это не ссылка на DOM-элемент, а «запрос», который выполняется заново перед каждым действием или проверкой. 
> 
> Это фундаментальное отличие от Selenium, делающее тесты устойчивыми к асинхронной загрузке.

- `Ленивая резолюция` — `page.locator()` не ищет элемент немедленно. Поиск происходит в момент вызова `.click()`, `.isVisible()`, `.count()` или `expect()`.
- `Auto-waiting` — перед действием Playwright автоматически проверяет: элемент видим, включён, стабилен, не перекрыт, получает события.
- `Strict Mode` — если селектор возвращает несколько элементов, а действие не указывает, какой именно выбрать, 
  Playwright выбросит `PlaywrightException: strict mode violation`. 
  Это защита от неочевидных кликов по первому найденному элементу.
- `first()` / `last()` / `nth(int)` — явное указание индекса для обхода strict mode.
- `filter()` — фильтрация набора по тексту, наличию дочерних элементов или другим локаторам.
- `count()` — синхронное получение количества совпадений. 
 
  **Важно:** `count()` прерывает цепочку авто-ожидания и возвращает результат мгновенно на текущий момент DOM-дерева.

#### Примеры работы с strict mode и фильтрацией:

```java
// ❌ Ошибка strict mode violation, если на странице несколько кнопок "Далее"
// page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Далее")).click();

// ✅ Явное указание: первая кнопка в наборе
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Далее"))
    .first()
    .click();

// ✅ Фильтрация по наличию иконки внутри кнопки
Locator submitBtn = page.locator("button")
    .filter(new Locator.FilterOptions()
        .setHas(page.locator("svg.icon-save"))); // Только кнопки с иконкой сохранения
submitBtn.click();

// ✅ Фильтрация по тексту (альтернатива getByText)
page.locator("div.row")
    .filter(new Locator.FilterOptions().setHasText("Ожидает оплаты"))
    .locator("button")
    .click();

// ⚠️ count() не ждёт появления элементов! Используйте только для отладки или синхронных проверок
int visibleItems = page.locator(".cart-item").count(); 
System.out.println("Items found instantly: " + visibleItems);
```

---

## Комбинирование селекторов и работа с DOM

> Playwright позволяет строить сложные поисковые цепочки, нативно обходить Shadow DOM и работать с вложенными фреймами 
> без ручного переключения контекста или выполнения JavaScript.

- `Цепочки locator().locator()` — сужение области поиска. Каждый следующий `locator()` ищет только внутри результатов предыдущего.
- `frameLocator()` — доступ к `iframe`. Возвращает `FrameLocator`, который ведёт себя как `Locator`, но контекст поиска ограничен фреймом.
- `Shadow DOM` — Playwright автоматически проникает в теневые деревья. Не нужно писать JS-обходы. 
  Поддерживаются `:host`, `::part`, и стандартные CSS-селекторы.
- `frameLocator().locator()` — комбинирование фреймов и элементов внутри них для изолированного поиска.

```java
page.locator("#app")
    .frameLocator("#payment-iframe")    // Вход в iframe
    .locator(":host > .card-wrapper")   // Проникновение в Shadow DOM
    .locator("input[name='cvv']")       // Поиск внутри Shadow DOM
```

#### Примеры комбинирования и Shadow DOM:

```java
// 1. Цепочка сужения контекста
Locator userRow = page.locator("table.users")
    .locator("tbody tr")
    .filter(new Locator.FilterOptions().setHasText("ivan.petrov"));

// Действие внутри найденной строки
userRow.locator("button.edit").click();

// 2. Работа с iframe
FrameLocator videoFrame = page.frameLocator("#video-player-frame");
videoFrame.locator("button.play").click();

// 3. Вложенные iframes и Shadow DOM
page.frameLocator("#complex-widget")
    .frameLocator("#inner-host")
    .locator(":host > .submit-btn")
    .click();

// 4. Кастомный атрибут test-id через цепочку
page.locator("[data-testid='checkout-modal']")
    .locator("[data-testid='confirm-payment']")
    .click();
```

---

## Best Practices

- **Приоритет `getBy*` над `locator()`** — используйте `getByRole()`, `getByText()`, `getByTestId()`, `getByLabel()`. 
  Они читаемы, семантически верны и устойчивы к изменению классов/структуры DOM.
- **Централизация `testIdAttribute`** — если разработчики добавляют `data-qa`, `data-test` или `data-automation-id`, 
  настройте Playwright на их использование глобально:
  ```java
  // В инициализации Playwright (вызывается один раз за прогон):
  playwright.selectors().setTestIdAttribute("data-qa");
  // Теперь getByTestId("login-btn") ищет [data-qa="login-btn"]
  ```
- **Избегайте хрупких XPath** — `//div[2]/span[3]/a` сломается при добавлении одного `div` в разметку. 
  Замените на семантические роли или `data-testid`.
- **Используйте `filter()` вместо сложных CSS-комбинаторов** — `page.locator("tr").filter(new FilterOptions().setHasText("Active"))` 
  читается лучше и работает стабильнее, чем `tr:has(td:contains('Active'))`.
- **Кэшируйте `Locator` в переменных** — не вызывайте `page.locator()` многократно в одном тесте. 
  Сохраните в `private final Locator submitButton = page.getByRole(...)`.
- **Не используйте `count()` для ожиданий** — `while(locator.count() < 3) Thread.sleep(500);` блокирует поток и ломает авто-ожидания. 
  Используйте `expect(locator).toHaveCount(3)`.
- **Не игнорируйте `strict mode violation`** — это не баг Playwright, а сигнал, что ваш селектор недостаточно точен. 
  Добавьте `.first()`, `.filter()` или уточните атрибуты.
- **Не пишите кастомные JS-обходы для Shadow DOM** — `page.evaluate("document.querySelector(...)shadowRoot...")` работает, 
  но лишает вас auto-waiting, типизации и отладки через Trace Viewer.
- **Не смешивайте `page.$()` и `Locator`** — `ElementHandle` устарел. Перейдите полностью на `Locator` API для согласованности и поддержки.

---

# 5. Взаимодействие с элементами и Auto-waiting

> Playwright исключает необходимость ручного управления ожиданиями благодаря встроенному механизму `Auto-waiting`. 
> 
> Перед каждым действием библиотека автоматически проверяет готовность элемента к взаимодействию, гарантируя, 
> что тесты выполняются только когда DOM стабилен, элемент видим и не перекрыт другими компонентами.

## Механика Auto-waiting

> `Auto-waiting` - ядро стабильности Playwright, заменяющее явные `Thread.sleep()` и сложные `WebDriverWait` из Selenium. 
>
> Перед выполнением любого действия (`click()`, `fill()`, `selectOption()`) фреймворк последовательно проверяет условия `actionability`.

- `visible` - элемент имеет `display != none`, `visibility != hidden` и `opacity > 0` (полностью прозрачны и не видны пользователю) 
  в вычисленных стилях
- `enabled` - элемент не имеет атрибута `disabled` и не заблокирован через `pointer-events: none`
- `stable` - позиция и размеры элемента не изменяются в течение 50 мс (защита от анимаций и лоадеров)
- `editable` - поле ввода активно для клавиатурного взаимодействия и принимает фокус
- `receivesEvents` - элемент не перекрыт другими узлами DOM (overlay, sticky headers, модальные окна)

ASCII-схема процесса проверки:

```text
┌─────────────────────────────────────────────────────────────┐
│              ВЫЗОВ ДЕЙСТВИЯ: locator.click()                │
└───────────────────────────────┬─────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  1. ПОИСК (DOM Query)                                       │
│     └─► Если 0 совпадений → повтор до таймаута              │
│     └─► Если >1 совпадений → Strict Mode Violation          │
└───────────────────────────────┬─────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  2. ACTIONABILITY CHECK (цикл до успеха или таймаута)       │
│     ├─► visible?   → Да / Ждём перерисовки                  │
│     ├─► enabled?   → Да / Ждём снятия disabled              │
│     ├─► stable?    → Да / Ждём остановки анимации           │
│     └─► unobscured?→ Да / Скроллим или ждём закрытия overlay│
└───────────────────────────────┬─────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  3. ВЫПОЛНЕНИЕ ДЕЙСТВИЯ (Dispatch Event)                    │
│     └─► pointerdown → pointerup → click                     │
└─────────────────────────────────────────────────────────────┘
```

Сравнение подходов к ожиданиям:

| Критерий              | Playwright Auto-waiting                | Selenium Explicit Wait                    | Cypress Auto-wait                         |
|:----------------------|----------------------------------------|-------------------------------------------|-------------------------------------------|
| Механизм проверки     | Встроен в каждое действие              | Отдельный `WebDriverWait.until()`         | Встроен в команды API                     |
| Проверка стабильности | `stable` (отсутствие сдвигов)          | Отсутствует (только presence/visibility)  | Частичная                                 |
| Обработка перекрытий  | Автоматический скролл/ожидание         | Требует ручного `Actions.moveToElement()` | Автоматический                            |
| Накладные расходы     | Минимальные (оптимизированный polling) | Высокие (сериализация JSON-RPC + polling) | Средние (выполнение в контексте браузера) |




Вот исправленная таблица, где вместо Cypress Auto-wait добавлен **Selenide** (популярная обёртка над Selenium WebDriver, которая также внедряет встроенные ожидания):

## Сравнение подходов к ожиданиям:

| Критерий                  | Playwright Auto-waiting                                       | Selenium Explicit Wait                              | Selenide Auto-wait                                                      |
|:--------------------------|---------------------------------------------------------------|-----------------------------------------------------|-------------------------------------------------------------------------|
| **Механизм проверки**     | Встроен в каждое действие                                     | Отдельный `WebDriverWait.until()`                   | Встроен в команды API (`.click()`, `.shouldBe()`)                       |
| **Проверка стабильности** | `stable` (отсутствие сдвигов позиции/размера в течение 50 мс) | Отсутствует (только `presence`/`visibility`)        | Отсутствует (только видимость и enabled)                                |
| **Обработка перекрытий**  | Автоматический скролл + ожидание ухода оверлея                | Требует ручного `Actions.moveToElement()`           | Автоматический скролл, но без проверки перекрытия другими элементами    |
| **Накладные расходы**     | Минимальные (оптимизированный polling через WebSocket)        | Высокие (сериализация JSON-RPC + удалённый polling) | Низкие/средние (работа поверх WebDriver, но с умными retry-механизмами) |
| **Таймауты по умолчанию** | 5 000 мс (глобальный), 30 000 мс (навигация)                  | 0 мс (требуется явная настройка)                    | 4 000 мс (можно переопределить)                                         |
| **Условия ожидания**      | `visible`, `enabled`, `stable`, `editable`, `receivesEvents`  | `presence`, `visibility`, `clickable` (редко)       | `visible`, `enabled`, `exist`                                           |


> Selenide — это большой шаг вперёд по сравнению с чистым Selenium, но Playwright уходит дальше за счёт 
> проверки стабильности и обработки перекрытий, что особенно важно для современных SPA с динамическими интерфейсами.

---

## Действия с элементами

> Playwright предоставляет типизированные методы для симуляции пользовательского ввода. 
> 
> Все действия возвращают `void` и поддерживают кастомизацию через `*Options` объекты.

- `void click()` / `void click(Locator.ClickOptions options)` - симуляция нажатия левой кнопки мыши
  - `button` - выбор кнопки мыши `MouseButton` (`LEFT`, `RIGHT`, `MIDDLE`)
    ```java
    // 1. ЛЕВАЯ кнопка (используется по умолчанию)
    page.locator("#submit").click();    // эквивалентно явному LEFT
    page.locator("#submit").click(new Locator.ClickOptions()
                        .setButton(MouseButton.LEFT));
                        
    // 2. ПРАВАЯ кнопка (открытие контекстного меню)
    page.locator("#file").click(new Locator.ClickOptions()
                        .setButton(MouseButton.RIGHT));
    ```
  - `clickCount` - количество кликов (для `dblclick` используйте `2`)
  - `delay` - задержка между `pointerdown` и `pointerup` в миллисекундах
  - `force` - принудительный клик без проверки `actionability`
  ```java
  // Клик с настройками: правая кнопка, двойное нажатие, задержка 100мс
  page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Options"))
            .click(new Locator.ClickOptions()
            .setButton(MouseButton.RIGHT)   // Симуляция правого клика
            .setClickCount(2)               // Двойное нажатие
            .setDelay(100));                // Задержка между down/up для эмуляции человека
  ```
- `void fill(String value)` - полная очистка поля и ввод текста (оптимизировано для `input`/`textarea`)
  ```java
  // Полная очистка и ввод текста в поле (заменяет clear() + sendKeys())
  page.getByLabel("Email").fill("tester@example.com"); // fill() автоматически очищает поле перед вводом
  ```
- `void type(String text)` / `void type(String text, Locator.TypeOptions options)` - посимвольный ввод с эмуляцией задержек клавиатуры
  - `delay` - задержка между нажатиями клавиш
    ```java
    // Посимвольный ввод с задержкой (полезно для тестирования автокомплита)
    page.getByLabel("Search").type("macbook", new Locator.TypeOptions().setDelay(50)); // 50мс между символами
    ```
- `void press(String key)` / `void press(String key, Locator.PressOptions options)` - нажатие комбинаций клавиш
  - `setDelay(double delay)` - Задержка в мс между нажатием и отпусканием клавиши (симуляция удержания)
    ```java
    page.locator("#search").fill("очень длинный текст для удаления");
    page.locator("#search").press("Backspace", new Locator.PressOptions()
              .setDelay(150));  // имитация удержания клавиши
    ```
  - `setNoWaitAfter(boolean noWaitAfter)` - Не ждать завершения навигации/загрузки после нажатия
    ```java
    // Нажатие Escape в модальном окне — закрывает его, без навигации
    page.locator(".modal").press("Escape", new Locator.PressOptions()
              .setNoWaitAfter(true));

    // Ждём, пока модалка исчезнет
    assert page.locator(".modal").isHidden();
    ```
  - `setTimeout(double timeout)` - Переопределяет глобальный таймаут ожидания только для этого конкретного действия `press()`
    ```java
    // Глобальный таймаут установлен на 5 секунд
    page.setDefaultTimeout(5000);

    // Кнопка "Экспорт" после нажатия Enter генерирует отчёт 15 секунд
    // Увеличиваем таймаут только для этого действия
    page.locator("#export").press("Enter", new Locator.PressOptions()
              .setTimeout(20000.0));  // ждём 20 секунд вместо 5

    // Другие действия продолжают использовать глобальный таймаут (5 сек)
    page.locator("#save").click();  // упадёт через 5 секунд, если элемент не появится
    ```
  - `key` - формат `Key` или `Modifier+Key` (`Enter`, `Control+Shift+A`, `Backspace`)
    ```java
    // Нажатие комбинации клавиш: Ctrl+A, затем Delete, затем ввод
    page.getByLabel("Description")
              .press("Control+a");        // Выделение всего текста
    page.getByLabel("Description")
              .press("Backspace");        // Удаление
    page.getByLabel("Description")
              .fill("Новое описание");    // Ввод нового значения
    ```
- `List<String> selectOption(String value)` / `List<String> selectOption(SelectOption option)` - выбор элемента в `<select>`
  ```java
  // 1. Базовый способ: строка (авто-поиск по value → label)
  // <option value="RU">Россия</option>
  page.getByLabel("Country").selectOption("RU");      // по value
  // <option value="msk">Москва</option>
  page.getByLabel("City").selectOption("Москва");     // по label (если value не найден)

  // 2. Явный выбор через SelectOption
  page.getByLabel("Country").selectOption(new SelectOption().setValue("RU"));    // по value
  page.getByLabel("City").selectOption(new SelectOption().setLabel("Москва"));   // по тексту
  page.getByLabel("Currency").selectOption(new SelectOption().setIndex(2));      // по индексу (c 0)

  // 3. Multiple select (выбор нескольких опций через массив строк)
  page.locator("#hobbies").selectOption(new String[] {"reading", "music"});

  // 4. Упрощённый синтаксис (без Locator API)
  page.selectOption("#country", "RU"); // Поиск по CSS-селектору
  page.selectOption("#hobbies", new String[] {"reading", "music"});

  // selectOption возвращает список выбранных значений
  List<String> selected = page.locator("#hobbies").selectOption(new String[] {"reading", "music"}); // [reading, music]
  
  // Важно: приоритет в строковом методе: value → label
  // <option value="RU">Россия</option> → selectOption("RU") найдёт по value
  // selectOption("Россия") найдёт по label (если value='Россия' не существует)
  ```
- `void check()` / `void uncheck()` - управление чекбоксами и radiobutton
  ```java
  // Чекбокс и радиокнопка
  page.getByLabel("Subscribe to newsletter").check();       // Установить
  page.getByLabel("Male").uncheck();                        // Снять
  ```
- `void dragTo(Locator target)` / `void dragTo(Locator target, Locator.DragToOptions options)` - перетаскивание элемента.
  Выполняет операцию drag-and-drop (перетаскивание) от одного элемента к другому левой кнопкой мыши.
  Перед началом перетаскивания Playwright автоматически проверяет, что оба элемента видимы, стабильны и готовы к взаимодействию.
  ```java
  // Простое перемещение элемента A в элемент B
  Locator taskCard = page.locator("#task-1");
  Locator doneColumn = page.locator("#column-done");
  taskCard.dragTo(doneColumn);
  
  // Перетаскивание с дополнительными опциями
  Locator fileIcon = page.locator(".file-icon");
  Locator folder = page.locator(".folder");
  
  fileIcon.dragTo(folder, new Locator.DragToOptions()
      .setSourcePosition(5, 5)        // захват за точку (5px, 5px) от левого верхнего угла элемента
      .setTargetPosition(10, 10)      // отпускание в точке (10px, 10px) от левого верхнего угла целевого элемента
      .setTimeout(10000.0)            // таймаут 10 секунд на выполнение
      .setNoWaitAfter(true));         // не ждать завершения навигации (если перетаскивание вызывает переход)
  ```
- `void hover()` / `void focus()` / `void blur()` - управление фокусом и наведением
  - `hover()` - Наведение курсора мыши на элемент (не работает на touch-устройствах - для мобильных тестов используйте `tap()`)
    ```java
    // Проверка появления тултипа
    Locator emailField = page.locator("#email");
    emailField.hover();
    
    assertThat(page.locator(".tooltip:has-text('корпоративную')))
             .isVisible();
    ```
  - `focus()` - Установка фокуса на элемент (клавиатурного) (только если `keyboard()`, не `fill()`)
    ```java
    // Фокус перед вводом (обычно не нужно, т.к. fill() сам ставит фокус)
    Locator passwordField = page.locator("#password");
    passwordField.focus();
    page.keyboard().type("secret123");  // ввод с клавиатуры после фокуса
    ```
  - `blur()` - Снятие фокуса с элемента (для валидации)
    ```java
    Locator phoneField = page.locator("#phone");
    phoneField.fill("123");     // начинаем вводить 
    phoneField.blur();          // снимаем фокус
            
    // Проверяем появление сообщения об ошибке
    assertThat(page.locator(".error:has-text('Должно быть 10 цифр')))
            .isVisible();
    ```

---

## Обработка ввода и специальных компонентов

> Современные веб-приложения часто используют кастомные виджеты, которые не реагируют на стандартные события. 
> 
> Playwright предоставляет обходные пути и API для работы с файлами, буфером обмена и динамическими контролами.

- `void setInputFiles(Path path)` / `void setInputFiles(FilePayload payload)` - загрузка файлов в `<input type="file">`
  - Поддерживает одиночные файлы, массивы `Path[]` и мульти-выбор
  - Обходит нативный диалог выбора файла ОС, загружая данные напрямую в DOM
- `void fill(String value)` с кастомными датапикерами - если виджет блокирует прямой ввод, 
  используйте `pressSequentially()` или JS-эвалюацию
- `void fill()` + `press("Enter")` - обход автокомплитов, которые не реагируют на `fill()` без явного подтверждения
- `Clipboard API` - работа с буфером обмена через `page.evaluate()` или кастомные события `paste`

```java
// Загрузка одного файла в поле ввода
page.locator("#upload-document").setInputFiles(Path.of("src/test/resources/report.pdf"));

// Загрузка нескольких файлов одновременно
Path[] files = {
    Path.of("src/test/resources/img1.jpg"),
    Path.of("src/test/resources/img2.png")
};
page.locator("#multi-upload").setInputFiles(files);

// Обход кастомного датапикера (если поле только для чтения)
Locator dateInput = page.getByLabel("Start Date");
dateInput.click();                        // Открыть календарь
page.getByText("15").click();             // Выбрать день
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Next Month")).click();
page.getByText("20").click();             // Подтвердить выбор

// Эмуляция вставки из буфера обмена (обход ограничений paste-событий)
page.getByLabel("Verification Code").evaluate("""
    (element, code) => {
        element.value = code;                                           // Прямая установка значения
        element.dispatchEvent(new Event('input', { bubbles: true }));   // Эмуляция события ввода
        element.dispatchEvent(new Event('change', { bubbles: true }));  // Эмуляция изменения
    }
""", "849201");
```

---

## Assertions и проверки состояния

> Playwright включает типизированный модуль утверждений `com.microsoft.playwright.assertions.PlaywrightAssertions`,
> который автоматически применяет `auto-waiting` к проверкам. 
> 
> Для удобства рекомендуется статический импорт: `import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;`

### Методы-утверждения (все возвращают `void`):

- `assertThat(Locator).toBeVisible()` - ожидание видимости элемента на экране
- `assertThat(Locator).toHaveText(String text)` - проверка точного или частичного совпадения текста
- `assertThat(Locator).toBeChecked()` / `toBeUnchecked()` - валидация состояния чекбоксов
- `assertThat(Locator).toBeEnabled()` / `toBeDisabled()` - проверка доступности элемента
- `assertThat(Locator).toHaveCSS(String property, String value)` - валидация вычисленных стилей
- `assertThat(APIResponse).isOk()` / `hasStatus(int code)` - проверка HTTP-ответов
- `assertThat(Locator).toHaveAttribute(String name, String value)` - проверка атрибутов
- `assertThat(Locator).toHaveValue(String value)` - валидация значения поля ввода
- `assertThat(Locator).toHaveCount(int count)` - проверка количества элементов в коллекции
- `assertThat(Locator).toBeAttached()` - проверяет, что элемент подключен к DOM (Document или ShadowRoot), 
  но **не обязательно видим** на странице. Добавлен в v1.33. Отличие от `toBeVisible()`: `toBeVisible()` требует 
  как подключения к DOM, **так и** видимости на экране, тогда как `toBeAttached()` проверяет только наличие в DOM, 
  что полезно для скрытых элементов или элементов до их отрисовки.

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

// Проверка видимости с кастомным таймаутом (5 секунд)
assertThat(page.getByRole(AriaRole.HEADING, new Page.GetByRoleOptions().setName("Dashboard")))
    .toBeVisible(new ExpectOptions().setTimeout(5000));

// Валидация текста с регулярным выражением
assertThat(page.locator(".status-message"))
    .toHaveText(Pattern.compile("Успешно сохранено.*"));

// Комбинированная проверка атрибута и CSS
assertThat(page.locator("#theme-toggle"))
    .toHaveAttribute("aria-pressed", "true");
assertThat(page.locator(".error-toast"))
    .toHaveCSS("background-color", "rgb(220, 53, 69)");

// Чекбоксы и радиокнопки
assertThat(page.getByLabel("Accept Terms")).toBeChecked();
assertThat(page.getByLabel("Decline")).not().toBeChecked(); // Негация

// Проверка количества элементов
assertThat(page.locator(".search-result")).toHaveCount(10);
```

#### Важные особенности:

- **`assertThat()`** - это статический алиас для `PlaywrightAssertions.expect()`, возвращает объект-ассерт
- **Все методы `toBe...()` / `toHave...()`** возвращают `void`, выполняя проверку с auto-waiting
- **`not()`** - возвращает тот же тип ассерта для негативных проверок
- **`ExpectOptions.setTimeout()`** - переопределяет таймаут только для конкретной проверки

---

## Кастомные ожидания и синхронизация

> Когда встроенных проверок недостаточно, Playwright позволяет создавать собственные условия ожидания, 
> ожидать сетевые ответы или контролировать загрузку ресурсов на уровне страницы.

- `void waitForCondition(Supplier<Boolean> condition)` - ожидание выполнения произвольного Java-условия
- `APIResponse waitForResponse(String urlPattern)` / `waitForResponse(Predicate<Response> predicate)` - блокировка потока до получения HTTP-ответа
- `void waitForLoadState(LoadState state)` - ожидание состояний загрузки страницы (`LOAD`, `DOMCONTENTLOADED`, `NETWORKIDLE`)
- `void waitForFunction(String expression)` - выполнение JS в контексте страницы до возврата `true`

```java
// Ожидание конкретного сетевого ответа (например, сохранения формы)
try (Response response = page.waitForResponse("**/api/checkout/submit")) {
    page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Place Order")).click(); // Действие, вызывающее запрос
    // Блок блокируется до получения ответа с совпадающим URL
    assertEquals(200, response.status()); // Валидация статуса после разблокировки
    assertTrue(response.json().getAsJsonObject().has("orderId")); // Проверка полезной нагрузки
}

// Ожидание загрузки ресурсов с таймаутом
page.navigate("https://app.example.com/gallery", new Page.NavigateOptions()
    .setWaitUntil(WaitUntilState.NETWORKIDLE)    // Ждём завершения всех запросов
    .setTimeout(60000));                         // Максимальное время ожидания

// Кастомное условие: ожидание появления 3 карточек товара
page.waitForCondition(() -> {
    int count = page.locator(".product-card").count();  // Мгновенный подсчёт без auto-waiting
    return count >= 3;                                  // Возврат true прервёт ожидание
});

// JS-ожидание: проверка, что React/Vue состояние обновилось
page.waitForFunction("() => window.appState.isLoading === false"); // Блокировка до изменения флага в JS
```

---

## Best Practices

- Полагайтесь на `Auto-waiting` как основной механизм синхронизации, он покрывает 95% сценариев без ручного кода
- Используйте `assertThat(locator).toHaveText()` вместо `locator.textContent().equals()`, чтобы автоматически дождаться появления текста
- Конфигурируйте глобальный таймаут через `Page.setDefaultTimeout()` один раз в `@BeforeAll` или базовом классе
- Применяйте `waitForResponse()` для синхронизации UI с бэкендом, это надёжнее, чем ожидание изменения DOM
- Загружайте файлы через `setInputFiles()` напрямую, минуя нативные диалоги ОС, для стабильности в CI
- Не используйте `Thread.sleep()` для синхронизации, это блокирует поток раннера и ломает параллельные запуски
- Не применяйте `force = true` в `ClickOptions` без крайней необходимости, обход проверки стабильности приводит к ложным успехам тестов
- Не вызывайте `count()` внутри циклов с `Thread.sleep()`, это создаёт гонки состояний и не гарантирует корректность ожидания
- Не валидируйте элементы через `page.querySelector()` и ручные `null`-проверки, используйте `expect(locator).toBeAttached()`
- Не игнорируйте возвращаемые значения `waitForResponse()`, всегда валидируйте `status()` и `body()` для полного покрытия

---

# 6. Интеграция с JUnit 5 и TestNG

> Интеграция Playwright с популярными раннерами тестов требует чёткого понимания жизненных циклов и управления ресурсами. 
> 
> Правильная настройка хуков, параллелизма и изоляции контекстов гарантирует стабильность прогонов в локальной среде и 
> CI/CD пайплайнах. 
> 
> JUnit 5 и TestNG предлагают разные подходы к параллельному выполнению и внедрению зависимостей, 
> что влияет на архитектуру тестового фреймворка.

## JUnit 5

> JUnit 5 использует модульную архитектуру `JUnit Platform` + `JUnit Jupiter`, где конфигурация параллелизма 
> и жизненные циклы управляются через свойства `surefire`/`junit-platform.properties` и аннотации.

Вот исправленный блок **«Жизненный цикл и хуки»** без буллитов с сигнатурами методов, но с сохранением сути:

### Жизненный цикл и хуки

Управление ресурсами Playwright в JUnit 5 строится на изменении стратегии создания экземпляра тестового класса и использовании стандартных хуков:

- **`@TestInstance(Lifecycle.PER_CLASS)`** — один экземпляр класса на все тесты. 

  Это позволяет переиспользовать тяжёлые объекты (`Browser`, `Playwright`) между методами, избегая их многократного пересоздания.

- **`@BeforeAll`** — хук для однократной инициализации `Playwright` и `Browser` перед выполнением всех тестов класса.

- **`@AfterAll`** — хук для закрытия `Browser` и `Playwright` после завершения всех тестов.

- **`@BeforeEach`** — создание нового изолированного `BrowserContext` перед каждым тестом. 

  Это гарантирует чистоту состояния (отдельные cookies, localStorage, сессии).

- **`@AfterEach`** — закрытие `BrowserContext` после каждого теста с освобождением временных артефактов (скриншоты, трассировки, записи).

- **`@ExtendWith(CustomPlaywrightExtension.class)`** — подключение кастомного расширения JUnit 5. 

  Через механизм `ParameterResolver` оно может автоматически внедрять в тестовые методы готовые объекты `Page` или `BrowserContext`, 
  сокращая дублирование кода инициализации.

#### Пример базовой инициализации с переиспользованием `Browser`:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS) // Один экземпляр класса на весь прогон
class LoginFlowTest {

    private Playwright playwright;
    private Browser browser;
    private BrowserContext context;
    private Page page;

    @BeforeAll
    void launchBrowser() {
        // Playwright и Browser инициализируются один раз для экономии времени запуска
        playwright = Playwright.create();
        browser = playwright.chromium()
                .launch(new BrowserType.LaunchOptions().setHeadless(true));
    }

    @AfterAll
    void closeBrowser() {
        // Закрытие ресурсов в обратном порядке инициализации
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }

    @BeforeEach
    void setupContext() {
        // Каждый тест получает свой изолированный контекст (cookies, localStorage)
        context = browser.newContext();
        page = context.newPage();
    }

    @AfterEach
    void teardownContext() {
        // Гарантированная очистка после каждого теста
        if (context != null) context.close();
    }

    @Test
    void testSuccessfulLogin() {
        page.navigate("https://app.example.com/login");
        page.getByLabel("Email").fill("user@test.com");
        page.getByLabel("Password").fill("secret");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
        
        // Валидация через встроенные assertions
        assertThat(page.getByRole(AriaRole.HEADING))
            .toHaveText("Добро пожаловать");
    }
}
```

### Параметризация и конфигурация параллелизма

- `@ParameterizedTest` - запуск одного тестового метода с разными наборами входных данных
- `@CsvSource({"data1, data2", "data3, data4"})` - передача строковых или примитивных параметров через CSV-формат
- `@MethodSource("dataProviderMethod")` - передача сложных объектов через статический поток `Stream<Arguments>`
- `junit.jupiter.execution.parallel.enabled=true` - включение параллельного выполнения тестов в `junit-platform.properties`
- `junit.jupiter.execution.parallel.config.strategy=dynamic` - автоматическое определение количества потоков на основе доступных ядер CPU
- `junit.jupiter.execution.parallel.mode.classes.default=concurrent` - параллельный запуск тестовых классов
- `junit.jupiter.execution.parallel.mode.default=concurrent` - параллельный запуск методов внутри классов

#### Конфигурация Maven Surefire для параллелизма:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <!-- Включение параллелизма через системные свойства -->
                <systemPropertyVariables>
                    <junit.jupiter.execution.parallel.enabled>true</junit.jupiter.execution.parallel.enabled>
                    <junit.jupiter.execution.parallel.mode.default>concurrent</junit.jupiter.execution.parallel.mode.default>
                    <junit.jupiter.execution.parallel.mode.classes.default>concurrent</junit.jupiter.execution.parallel.mode.classes.default>
                    <junit.jupiter.execution.parallel.config.strategy>dynamic</junit.jupiter.execution.parallel.config.strategy>
                </systemPropertyVariables>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## TestNG

> TestNG предоставляет встроенные механизмы параллелизма через XML-конфигурацию (`testng.xml`) 
> и гибкую систему листенеров для перехвата событий выполнения.

### Конфигурация сьютов и листенеры

- `@BeforeClass void setUpClass()` - выполнение кода один раз перед запуском всех методов в текущем `<test>` блоке
- `@AfterClass void tearDownClass()` - выполнение кода один раз после завершения всех методов в `<test>` блоке
- `@BeforeMethod void setUpMethod()` - аналог `@BeforeEach`, выполняется перед каждым `@Test` методом
- `@DataProvider(name = "loginData")` - метод, возвращающий `Object[][]` для параметризации тестов
- `@Test(dataProvider = "loginData") void testLogin(String user, String pass)` - привязка параметризованного теста 
  к источнику данных
- `ITestListener onTestStart(ITestResult result)` / `onTestFailure(ITestResult result)` - перехват событий выполнения 
  для создания скриншотов или логов
- `ISuiteListener onStart(ISuite suite)` / `onFinish(ISuite suite)` - глобальные хуки уровня сьюта для инициализации драйвера 
  или сбора метрик

#### Пример TestNG с `DataProvider` и `ITestListener`:

```java
public class CheckoutTest implements ITestListener {
    
    private ThreadLocal<Browser> browser = new ThreadLocal<>();
    private ThreadLocal<BrowserContext> context = new ThreadLocal<>();
    private ThreadLocal<Page> page = new ThreadLocal<>();

    @BeforeClass
    void initBrowser() {
        // Инициализация Playwright и браузера на уровень класса
        Playwright playwright = Playwright.create();
        browser.set(playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(true)));
    }

    @BeforeMethod
    void initContext() {
        // Создание изолированного контекста для каждого потока
        context.set(browser.get().newContext());
        page.set(context.get().newPage());
    }

    @DataProvider(name = "checkoutData", parallel = true)
    Object[][] provideProducts() {
        // Данные для параметризации, выполняются параллельно
        return new Object[][]{
            {"product-A", 1},
            {"product-B", 3},
            {"product-C", 2}
        };
    }

    @Test(dataProvider = "checkoutData")
    void verifyCartPrice(String productId, int quantity) {
        Page currentPage = page.get(); // Получение Page для текущего потока
        currentPage.navigate("https://shop.example.com");
        currentPage.locator("[data-testid='product-" + productId + "']").click();
        // Логика добавления и проверки цены
    }

    @AfterMethod
    void closeContext() {
        if (context.get() != null) context.get().close();
    }

    @AfterClass
    void closeBrowser() {
        if (browser.get() != null) browser.get().close();
    }

    // Реализация ITestListener для обработки падений
    @Override
    public void onTestFailure(ITestResult result) {
        System.err.println("Test failed: " + result.getName());
        // Здесь можно добавить снятие скриншота или сохранение трейса
    }
}
```

Конфигурация `testng.xml` для параллельного запуска:

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="PlaywrightSuite" parallel="methods" thread-count="4">
    <!-- parallel="methods" запускает каждый @Test метод в отдельном потоке -->
    <test name="CheckoutTests">
        <classes>
            <class name="com.example.CheckoutTest"/>
        </classes>
    </test>
</suite>
```

Сравнение жизненных циклов:

| Этап                    | JUnit 5                                    | TestNG                                    |
|:------------------------|--------------------------------------------|-------------------------------------------|
| Инициализация браузера  | `@BeforeAll` + `@TestInstance(PER_CLASS)`  | `@BeforeClass`                            |
| Инициализация контекста | `@BeforeEach`                              | `@BeforeMethod`                           |
| Параметризация          | `@ParameterizedTest` + `@CsvSource`        | `@DataProvider` + `@Test(dataProvider)`   |
| Обработка падений       | `@AfterEach` + `TestInfo.getDisplayName()` | `ITestListener.onTestFailure()`           |
| Параллелизм             | `junit-platform.properties`                | `testng.xml` (`parallel`, `thread-count`) |

---

## Управление ресурсами

> Playwright объекты (`Playwright`, `Browser`, `BrowserContext`, `Page`) реализуют интерфейс `AutoCloseable`, 
> что требует строгого контроля времени жизни для предотвращения утечек памяти и зависших процессов браузера.

- `void close()` - метод освобождения ресурсов, закрывающий сетевые соединения, процессы драйвера и временные файлы
- `try-with-resources` - конструкция Java, гарантирующая вызов `close()` даже при возникновении исключений в блоке
- `@AfterEach(alwaysRun = true)` - аннотация TestNG, обеспечивающая выполнение хука даже если тест упал с `OutOfMemoryError` или `AssertionError`
- `Runtime.getRuntime().addShutdownHook()` - глобальный хук JVM для аварийной очистки при принудительной остановке процесса

#### Пример безопасного закрытия при падениях:

```java
public class ResourceGuardExample {
    
    private BrowserContext context;

    @AfterEach
    void cleanup(TestInfo testInfo) {
        // Всегда закрываем контекст, даже если тест упал
        // В JUnit 5 @AfterEach выполняется в любом случае, 
        // но явная проверка состояния полезна для логирования
        if (context != null) {
            try {
                // Если тест упал, можно сохранить трейс перед закрытием
                context.tracing().stop(new Tracing.StopOptions()
                    .setPath(Path.of("traces/" + testInfo.getDisplayName() + ".zip")));
                context.close();
            } catch (Exception e) {
                System.err.println("Ошибка при очистке ресурсов: " + e.getMessage());
            }
        }
    }
}
```

---

## Изоляция и многопоточность

> При параллельном запуске тестов каждый поток должен работать с независимым набором объектов Playwright. 
> 
> Переиспользование `Page` или `BrowserContext` между потоками приводит к гонкам состояний, 
> непредсказуемым кликам и ложным падениям.

- `ThreadLocal<BrowserContext>` - потокобезопасный контейнер, хранящий уникальный контекст для каждого воркера раннера
- `ThreadLocal<Page>` - аналогичный контейнер для активной вкладки, привязанной к потоку
- `thread-count="N"` - ограничение количества одновременных потоков в `testng.xml` или `maven-surefire-plugin`
- `junit.jupiter.execution.parallel.config.dynamic.factor=0.5` - коэффициент загрузки CPU (0.5 = 50% ядер, 1.0 = 100%)
- `context.storageState()` - экспорт состояния авторизации в файл для последующей загрузки без повторного логина в каждом потоке

#### Пример настройки `ThreadLocal` для параллельных запусков:

```java
public class ParallelTestBase {
    
    // Статические ThreadLocal гарантируют, что каждый поток получит свой экземпляр
    protected static final ThreadLocal<Playwright> PLAYWRIGHT = ThreadLocal.withInitial(Playwright::create);
    protected static final ThreadLocal<Browser> BROWSER = new ThreadLocal<>();
    protected static final ThreadLocal<BrowserContext> CONTEXT = new ThreadLocal<>();
    protected static final ThreadLocal<Page> PAGE = new ThreadLocal<>();

    @BeforeMethod
    void initParallelEnv() {
        if (BROWSER.get() == null) {
            BROWSER.set(PLAYWRIGHT.get().chromium().launch());
        }
        // Новый контекст для каждого метода = полная изоляция cookies и кэша
        CONTEXT.set(BROWSER.get().newContext());
        PAGE.set(CONTEXT.get().newPage());
    }

    @AfterMethod
    void cleanParallelEnv() {
        if (CONTEXT.get() != null) CONTEXT.get().close();
        // Browser закрывается в @AfterClass или @AfterAll
    }

    protected Page getPage() {
        return PAGE.get(); // Безопасный доступ из любого @Test метода
    }
}
```

---

## Best Practices

- Используйте `ThreadLocal` для `BrowserContext` и `Page` при параллельных запусках, чтобы гарантировать потокобезопасность 
  без явных блокировок
- Фиксируйте `@TestInstance(PER_CLASS)` в JUnit 5 для переиспользования `Browser`, так как запуск нового процесса браузера 
  занимает 1–3 секунды
- Настраивайте `thread-count` в `testng.xml` или `dynamic.factor` в JUnit 5 на уровне 50–70% от доступных ядер CPU, 
  чтобы избежать конкуренции за сетевые сокеты драйвера
- Применяйте `alwaysRun = true` (TestNG) или `@AfterEach` (JUnit 5) для гарантированного закрытия контекстов, 
  даже при `AssertionError` или `StackOverflowError`
- Экспортируйте `storageState` один раз в `@BeforeSuite` и загружайте его в `NewContextOptions` для всех последующих тестов, 
  ускоряя прогон на 40%
- Не передавайте `Page` или `BrowserContext` через статические поля без `ThreadLocal`, это приводит к перекрёстному воздействию потоков 
  и нестабильным результатам
- Не запускайте `Playwright.create()` внутри каждого `@Test` метода, это создаёт избыточные процессы Node.js драйвера 
  и быстро исчерпывает память CI-агентов
- Не игнорируйте `close()` для `BrowserContext` при падениях, зависшие вкладки продолжают потреблять RAM 
  и могут вызвать `OutOfMemoryError` на следующих прогонах
- Не используйте `parallel="tests"` в TestNG для методов внутри одного класса без явного разделения сьютов, 
  раннер не гарантирует изоляцию переменных класса
- Не хардкодьте `thread-count="10"` без мониторинга утилизации CPU, избыточная параллелизация 
  снижает пропускную способность из-за конкуренции за WebSocket-соединения драйвера

---

# 7. API-тестирование через APIRequestContext

> Playwright предоставляет встроенный HTTP-клиент `APIRequestContext`, позволяющий отправлять запросы и валидировать ответы 
> без привлечения сторонних библиотек вроде RestAssured или Apache HttpClient. 
> 
> Клиент полностью интегрирован с экосистемой Playwright: умеет переиспользовать cookies, состояние авторизации 
> и сетевые заголовки из UI-контекста, что делает гибридные сценарии (API-подготовка данных -> UI-валидация) 
> естественными и быстрыми.

## Инициализация API-клиента

- `browser.newAPIRequestContext(APIRequest.NewContextOptions options)` - создание HTTP-клиента, 
  привязанного к конкретному экземпляру браузера и наследующего его сетевые настройки
- `playwright.request().newContext(APIRequest.NewContextOptions options)` - автономная инициализация клиента 
  без запуска браузера (оптимально для чистых API-тестов)
- `setBaseURL(String baseURL)` - установка базового домена и пути, автоматически подставляемого ко всем относительным URL запросов
- `setExtraHTTPHeaders(Map<String, String> headers)` - глобальные HTTP-заголовки, применяемые к каждому исходящему запросу
- `setStorageState(BrowserContext.StorageStateOptions options)` - загрузка cookies и localStorage из JSON-файла 
  для имитации авторизованной сессии
- `setHttpCredentials(String username, String password)` - встроенная поддержка HTTP Basic Auth без ручной генерации `Authorization` заголовка
- `setIgnoreHTTPSErrors(boolean ignore)` - принудительное игнорирование ошибок SSL/TLS сертификатов (полезно для локальных dev-окружений)
- `setTimeout(int timeout)` - глобальный таймаут ожидания ответа сервера в миллисекундах

```java
// Конфигурация параметров создания API-клиента
APIRequest.NewContextOptions apiOptions = new APIRequest.NewContextOptions()
    .setBaseURL("https://api.staging.example.com/v2")      // Базовый хост и версия API
    .setExtraHTTPHeaders(Map.of(                           // Глобальные заголовки для всех запросов
        "X-Client-Version", "2.5.1",
        "Accept-Language", "ru-RU",
        "X-Test-Run-Id", UUID.randomUUID().toString()      // Уникальный ID для трассировки на бэкенде
    ))
    .setTimeout(15000)                                     // Глобальный таймаут запроса 15 секунд
    .setIgnoreHTTPSErrors(true);                           // Игнор ошибок сертификатов в dev-окружении

// Создание клиента (экземпляра APIRequestContext)
APIRequestContext api1 = browser.newAPIRequestContext(apiOptions);
// Альтернатива для чистых API-тестов:
APIRequestContext api2 = playwright.request().newContext(apiOptions);

// GET-запрос
APIResponse getResponse = api.get("/users/123");
System.out.println("GET Status: " + getResponse.status());
System.out.println("GET Body: " + getResponse.text());
```

> Основное различие между `api1` и `api2` заключается в **привязке к браузеру** и **наследовании контекстных настроек**:

### `browser.newAPIRequestContext()`:

- **Привязан к экземпляру браузера** - наследует сетевые настройки браузера
- **Разделяет состояние с браузером** - если браузер уже прошел авторизацию (cookies, storage), 
  API-клиент автоматически получит ту же сессию
- **Использует единый storage state** - может совместно использовать `BrowserContext` cookies для API и UI действий
- **Идеален для гибридных тестов** - например: сначала UI-авторизация в браузере, потом API-запросы от имени того же пользователя

### `playwright.request().newContext()`:

- **Полностью автономный** - не зависит от браузера, не требует его запуска
- **Легковесный** - работает быстрее, потребляет меньше ресурсов
- **Изолированное состояние** - сессии нужно настраивать отдельно (через `setStorageState()`)
- **Оптимален для чистых API-тестов** - когда не нужен UI, только проверка бэкенда

## Практический пример использования:

```java
// Сценарий 1: Чистый API-тест (без браузера)
APIRequestContext apiOnly = playwright.request().newContext(apiOptions);
// Используется для тестирования /auth/login, /users, /products

// Сценарий 2: Гибридный тест (UI + API)
Browser browser = playwright.chromium().launch();
BrowserContext context = browser.newContext();
Page page = context.newPage();

// UI-авторизация
page.navigate("https://app.example.com/login");
page.fill("#username", "testuser");
page.fill("#password", "pass123");
page.click("#login");

// API-клиент автоматически получает cookies из браузера
APIRequestContext apiWithAuth = browser.newAPIRequestContext(apiOptions);
// Теперь API-запросы будут от имени авторизованного testuser
```

> **Главный критерий выбора:** используйте `playwright.request().newContext()` для изолированных API-тестов, 
> `browser.newAPIRequestContext()` - когда нужно синхронизировать состояние с UI-сессией браузера.

---

## Выполнение запросов

- `APIResponse get(String url, RequestOptions options)` - отправка GET-запроса по указанному пути или полному URL
- `APIResponse post(String url, RequestOptions options)` - отправка POST-запроса с телом, заголовками или формой
- `APIResponse put(String url, RequestOptions options)` - полная замена ресурса (идемпотентный запрос)
- `APIResponse patch(String url, RequestOptions options)` - частичное обновление ресурса
- `APIResponse delete(String url, RequestOptions options)` - удаление ресурса по идентификатору
- `setBody(Object data)` - передача тела запроса: POJO-объект (автосериализация в JSON), String (сырой JSON/XML), byte[] или InputStream
- `setForm(Map<String, String> data)` - отправка данных в формате `application/x-www-form-urlencoded`
- `setMultipart(Map<String, Object> data)` - отправка `multipart/form-data` с поддержкой файлов и текстовых полей
- `setFailOnStatusCode(boolean fail)` - автоматическое выбрасывание исключения при статус-коде `>= 400`
- `setTimeout(int timeout)` - переопределение глобального таймаута для конкретного запроса
- `setQueryParam(String name, String value)` - добавление параметра в query-строку URL

```java
// 1. GET-запрос с query-параметрами и кастомным таймаутом
APIResponse getResponse = api.get("/users/123/profile", new RequestOptions()
    .setQueryParam("include", "orders,preferences")                 // Добавляет ?include=orders,preferences
    .setTimeout(10000));                                            // Локальный таймаут 10 сек

// 2. POST-запрос с JSON-телом (Map автоматически конвертируется в JSON)
APIResponse postResponse = api.post("/orders", new RequestOptions()
    .setHeader("Idempotency-Key", UUID.randomUUID().toString())     // Уникальный ключ для повторных отправок (на бэк)
    .setData(Map.of(                                                // Карта автоматически конвертируется в JSON
        "productId", 789,
        "quantity", 2,
        "shippingAddress", Map.of("city", "Moscow", "zip", "101000")
    ))
    .setFailOnStatusCode(true));                                    // Выбросит ошибку, если статус >= 400

// 3. PATCH-запрос с сырым JSON-строкой (для сложных legacy-API)
APIResponse patchResponse = api.patch("/settings/notifications", new RequestOptions()
    .setBody("{\"email\": true, \"sms\": false, \"push\": false}")  // Прямая передача строки
    .setHeader("Content-Type", "application/json"));

// 4. DELETE-запрос без тела
APIResponse deleteResponse = api.delete("/cart/items/456");
```

---

## Обработка ответов и валидация

- `int status()` - возврат HTTP-статус-кода ответа (200, 404, 500 и т.д.)
- `String statusText()` - текстовое описание статус-кода (OK, Not Found, Internal Server Error)
- `boolean ok()` - быстрая проверка, находится ли статус в успешном диапазоне 200–299
- `Map<String, String> headers()` - получение всех заголовков ответа в виде карты
- `String headerValue(String name)` - извлечение конкретного заголовка без учёта регистра
- `byte[] body()` - получение тела ответа в виде массива байтов (для бинарных данных или изображений)
- `String text()` - декодирование тела ответа в строку с использованием UTF-8
- `JsonElement json()` - парсинг JSON-ответа в объект `com.google.gson.JsonElement` для навигации по полям
- `InputStream bodyStream()` - получение потока для чтения больших ответов без загрузки в память

```java
// Валидация статуса и заголовков
assertTrue(postResponse.ok(), "Заказ должен быть успешно создан");
assertEquals(201, postResponse.status());                       // Проверка конкретного статус-кода
assertEquals("application/json; charset=utf-8", 
    postResponse.headerValue("Content-Type"));                  // Проверка заголовка ответа

// Парсинг и проверка JSON-полей через GSON
JsonObject orderData = postResponse.json().getAsJsonObject();
assertEquals("pending", orderData.get("status").getAsString()); // Проверка статуса заказа
assertTrue(orderData.has("orderId"));                           // Проверка наличия обязательного поля

// Интеграция с Playwright Assertions для API-ответов
assertThat(postResponse).isOk();                                // Авто-ожидание успешного статуса
assertThat(postResponse).hasStatus(201);                        // Строгая проверка кода
assertThat(postResponse).hasHeader("X-RateLimit-Remaining");    // Проверка наличия заголовка

// Работа с бинарными данными (например, экспорт PDF)
APIResponse pdfResponse = api.get("/reports/invoice/99.pdf");
if (pdfResponse.ok()) {
    byte[] pdfBytes = pdfResponse.body();                       // Чтение всего файла в память
    // Логика сохранения или валидации хеша файла
}
```

---

## Синхронизация UI и API-сценариев

- `BrowserContext.storageState(Path path)` - сохранение текущего состояния cookies, localStorage и sessionStorage в JSON-файл
- `BrowserContext.storageState()` - возврат состояния в виде объекта `StorageState` для программной передачи
- `APIRequestContext.storageState()` - экспорт состояния авторизации для последующего переиспользования в UI-тестах
- `context.route(String pattern, Route.Handler handler)` - перехват UI-запросов и подмена ответов данными из API
- `route.fulfill(Route.FulfillOptions options)` - возврат мокированного ответа браузеру без обращения к реальному серверу

```java
// 1. Авторизация через API с последующим сохранением сессии
APIRequestContext authClient = playwright.request().newContext(
    new APIRequest.NewContextOptions().setBaseURL("https://auth.example.com"));

APIResponse loginResp = authClient.post("/oauth/token", new RequestOptions()
    .setForm(Map.of("grant_type", "client_credentials", "client_id", "test", "client_secret", "secret")));

String accessToken = loginResp.json().getAsJsonObject().get("access_token").getAsString();

// 2. Передача токена в UI-контекст для прямого входа
BrowserContext uiContext = browser.newContext(new Browser.NewContextOptions()
    .setExtraHTTPHeaders(Map.of("Authorization", "Bearer " + accessToken)));

Page page = uiContext.newPage();
page.navigate("https://app.example.com/dashboard");        // Пользователь сразу авторизован

// 3. Подготовка данных через API перед UI-тестом
APIRequestContext dataClient = browser.newAPIRequestContext();
APIResponse createProduct = dataClient.post("/api/products", new RequestOptions()
    .setHeader("Authorization", "Bearer " + accessToken)
    .setData(Map.of("name", "Test-Product-01", "price", 99.99)));

String productId = createProduct.json().getAsJsonObject().get("id").getAsString();

// UI-тест использует созданный продукт без ручного заполнения форм
page.navigate("https://app.example.com/products/" + productId);
assertThat(page.getByText("Test-Product-01")).toBeVisible();
```

---

## Продвинутые сценарии

`setProxy(Proxy proxy)` - настройка прокси-сервера для маршрутизации всех API-запросов контекста
`setClientCertificates(List<ClientCertificate> certs)` - добавление mTLS-сертификатов для взаимной аутентификации
`Route.abort(String errorCode)` - принудительный обрыв запроса для эмуляции сетевых ошибок (connection refused, timeout)
`Route.fulfill(Route.FulfillOptions options)` - полная подмена статуса, заголовков и тела ответа
`BrowserContext.setOffline(boolean offline)` - эмуляция полного отключения сети на уровне браузера
`Route.continue_(Route.ContinueOptions options)` - модификация запроса (URL, метод, заголовки) перед отправкой (именно `continue_()`)

```java
// 1. Настройка прокси и клиентских сертификатов для защищённых API
APIRequest.NewContextOptions secureOptions = new APIRequest.NewContextOptions()
    .setProxy(new Proxy().setServer("http://corporate-proxy:8080"))
    .setClientCertificates(List.of(                      // mTLS для доступа к внутренним сервисам
        new ClientCertificate()
            .setOrigin("https://internal-api.corp")
            .setCertPath(Path.of("/certs/client.crt"))
            .setKeyPath(Path.of("/certs/client.key"))
    ));
APIRequestContext secureApi = playwright.request().newContext(secureOptions);

// 2. Мокирование API-ответов в UI-тесте через route()
browserContext.route("**/api/feature-flags", route -> route.fulfill(
    new Route.FulfillOptions()
        .setStatus(200)
        .setContentType("application/json")
        .setBody("{\"darkMode\": true, \"newCheckout\": false}") // Подмена фича-флагов
));

// 3. Эмуляция сетевых задержек и потери пакетов
browserContext.route("**/*", route -> {
    try { 
        Thread.sleep(2000);                                 // Искусственная задержка 2 сек
    } catch (InterruptedException e) {} 
        
    route.continue_();                                      // Передача управления дальше
});
```

> Важное примечание:
>
> Для API-тестов без браузера (`playwright.request().newContext()`) методы перехвата запросов (`route`) недоступны 
> — они работают только в контексте браузера (`browser.newContext()`).

---

## Best Practices

- Используйте `playwright.request().newContext()` для чистых API-тестов, чтобы не тратить ресурсы на запуск и инициализацию браузера
- Сохраняйте `storageState` после авторизации и переиспользуйте его во всех UI-сценариях, сокращая время прогона на 30–50%
- Настраивайте `setFailOnStatusCode(true)` по умолчанию, чтобы не пропускать 4xx/5xx ответы без явной валидации в коде теста
- Применяйте `setBody(Object)` с POJO-объектами или `Map` для типизированной сериализации JSON, избегайте ручной конкатенации строк
- Изолируйте API-подготовку данных в `@BeforeClass` или `@BeforeAll`, чтобы не дублировать сетевые запросы в каждом тестовом методе
- Не смешивайте UI-клиент (`page.locator()`) и API-клиент в одном потоке без чёткого разделения ответственности, 
  это усложняет отладку и логирование
- Не используйте `Thread.sleep()` для ожидания асинхронной обработки на бэкенде, применяйте polling через повторный `api.get()` 
  с проверкой статуса ресурса
- Не игнорируйте `setIgnoreHTTPSErrors(false)` в production-подобных окружениях, скрытие SSL-ошибок маскирует проблемы конфигурации инфраструктуры
- Не передавайте чувствительные данные (passwords, tokens) в строковых литералах, выносите их в `.env` или CI-secrets 
  с динамическим резолвингом
- Не оставляйте `APIRequestContext` открытым после завершения тестов, вызывайте `close()` или используйте `try-with-resources` 
  для своевременного освобождения сокетов

---

# 8. Архитектурные паттерны и организация кода

> Выбор архитектурного паттерна определяет масштабируемость, поддерживаемость и скорость разработки тестов. 
> 
> Playwright не навязывает конкретную структуру, но предоставляет гибкие примитивы (`Locator`, `Page`, `BrowserContext`), 
> которые можно комбинировать в классические паттерны (Page Object, Component Model) или современные подходы (Screenplay). 
> 
> Правильная организация кода снижает дублирование, упрощает рефакторинг и ускоряет онбординг новых участников команды.

## Page Object Model (POM)

> Классический паттерн инкапсуляции логики взаимодействия со страницей. 
> 
> Каждый класс POM представляет одну страницу или компонент приложения, скрывая детали селекторов и действий от тестов.

- `Locator` - приватные поля класса, хранящие селекторы элементов страницы
- `void action()` - публичные методы, выполняющие действия на странице (клик, ввод, навигация)
- `boolean isElementVisible()` - методы проверки состояния элементов
- `Page getPage()` - доступ к объекту `Page` для сложных сценариев (не рекомендуется выносить наружу)

#### Пример базовой реализации POM:

```java
// Page Object для страницы логина
public class LoginPage {

    private final Page page; // Хранение ссылки на Page для выполнения действий
    
    // Приватные локаторы: селекторы инкапсулированы внутри класса
    private final Locator emailInput = page.getByLabel("Email");
    private final Locator passwordInput = page.getByLabel("Password");
    private final Locator submitButton = page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти"));
    private final Locator errorMessage = page.locator(".error-message");
    
    // Конструктор принимает Page из теста или фикстуры
    public LoginPage(Page page) {
        this.page = page;
    }
    
    // Публичный метод: выполнение действия логина
    public void login(String email, String password) {
        emailInput.fill(email);           // Ввод email
        passwordInput.fill(password);     // Ввод пароля
        submitButton.click();             // Нажатие кнопки входа
    }
    
    // Метод проверки состояния: видимость сообщения об ошибке
    public boolean isErrorDisplayed() {
        return errorMessage.isVisible();  // Возврат булево значение
    }
    
    // Метод получения текста ошибки для валидации в тесте
    public String getErrorText() {
        return errorMessage.textContent(); // Извлечение текста из DOM
    }
    
    // Навигационный метод: переход на страницу логина
    public LoginPage navigateTo() {
        page.navigate("https://app.example.com/login"); // Переход по URL
        return this; // Возврат this для fluent-цепочек
    }
}
```

#### Использование Page Object в тесте:

```java
class LoginFlowTest {
  
    @Test
    void testSuccessfulLogin() {
        // Тест не знает о селекторах, только о бизнес-логике
        LoginPage loginPage = new LoginPage(page);
        loginPage.navigateTo()                           // Fluent-цепочка навигации
                 .login("user@test.com", "secret123");   // Выполнение действия
        
        // Валидация через assertions
        assertThat(page.getByRole(AriaRole.HEADING))
            .toHaveText("Добро пожаловать");
    }
    
    @Test
    void testInvalidCredentials() {
        LoginPage loginPage = new LoginPage(page);
        loginPage.login("wrong@email.com", "invalid");
        
        // Проверка состояния через метод Page Object
        assertTrue(loginPage.isErrorDisplayed());
        assertEquals("Неверный email или пароль", loginPage.getErrorText());
    }
}
```

---

## Page Component Model

> Расширение POM для переиспользуемых виджетов: модальные окна, хедеры, пагинация, таблицы. 
> 
> Компоненты не привязаны к конкретной странице и могут использоваться в разных контекстах.

- `Component` - базовый класс или интерфейс для переиспользуемых виджетов
- `Locator root` - корневой селектор компонента, ограничивающий область поиска
- `void interact()` - методы взаимодействия с компонентом
- `Component within(Locator parent)` - фабрика для создания компонента в контексте конкретного родителя

#### Пример реализации переиспользуемого компонента:

```java
// Компонент модального окна: может появляться на любой странице
public class ModalComponent {
    
    private final Locator root; // Корневой локатор модального окна
    
    // Приватные локаторы внутри компонента
    private final Locator title = root.locator(".modal-title");
    private final Locator closeButton = root.getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Закрыть"));
    private final Locator content = root.locator(".modal-body");
    
    // Конструктор принимает корневой локатор модального окна
    public ModalComponent(Locator root) {
        this.root = root;
    }
    
    // Метод проверки видимости модального окна
    public boolean isVisible() {
        return root.isVisible(); // Проверка состояния корневого элемента
    }
    
    // Метод получения заголовка модального окна
    public String getTitle() {
        return title.textContent(); // Извлечение текста заголовка
    }
    
    // Метод закрытия модального окна
    public void close() {
        closeButton.click(); // Клик по кнопке закрытия
    }
    
    // Метод получения содержимого модального окна
    public String getContent() {
        return content.textContent(); // Извлечение текста тела модального окна
    }
}
```

#### Использование компонента в разных Page Objects:

```java
// Page Object для страницы корзины
public class CartPage {
    
    private final Page page;
    
    // Компонент модального окна подтверждения удаления
    private final ModalComponent deleteConfirmationModal = new ModalComponent(page.locator("#delete-confirmation-modal") // Корневой селектор модального окна
    );
    
    public CartPage(Page page) {
        this.page = page;
    }
    
    // Метод удаления товара из корзины
    public void removeItem(String productName) {
        page.locator(".cart-item")
            .filter(new Locator.FilterOptions().setHasText(productName))
            .locator("button.remove")
            .click();
    }
    
    // Метод подтверждения удаления через модальное окно
    public void confirmDeletion() {
        deleteConfirmationModal.close(); // Использование компонента
    }
    
    // Метод проверки видимости модального окна
    public boolean isDeleteModalVisible() {
        return deleteConfirmationModal.isVisible(); // Делегирование компоненту
    }
}
```

---

## Screenplay Pattern

> Альтернативный подход, где тесты описываются через действия `Actor` (актёра), выполняющего `Task` (задачи) 
> с использованием `Ability` (способностей). 
> 
> Подходит для сложных бизнес-сценариев с множеством ролей.

- `Actor` - субъект, выполняющий действия (пользователь, администратор, система)
- `Ability` - способность актёра взаимодействовать с системой (браузер, API, база данных)
- `Task` - последовательность действий, выполняемых актёром
- `Question` - вопрос о состоянии системы, на который актёр получает ответ

#### Пример базовой реализации Screenplay:

```java
// Способность: доступ к браузеру
public class BrowseTheWeb implements Ability {
    
    private final Page page; // Хранение ссылки на Page
    
    public BrowseTheWeb(Page page) {
        this.page = page;
    }
    
    // Метод получения Page для выполнения действий
    public Page getPage() {
        return page;
    }
    
    // Статический фабричный метод для создания способности
    public static BrowseTheWeb with(Page page) {
        return new BrowseTheWeb(page);
    }
}

// Задача: вход в систему
public class Login implements Task {
    
    private final String email;
    private final String password;
    
    public Login(String email, String password) {
        this.email = email;
        this.password = password;
    }
    
    // Статический фабричный метод для создания задачи
    public static Login withCredentials(String email, String password) {
        return new Login(email, password);
    }
    
    // Метод выполнения задачи актером
    @Override
    public <T extends Actor> void performAs(T actor) {
        Page page = actor.abilityTo(BrowseTheWeb.class).getPage(); // Получение Page из способности
        
        // Выполнение действий входа
        page.getByLabel("Email").fill(email);
        page.getByLabel("Password").fill(password);
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
    }
}

// Вопрос: видимость сообщения об ошибке
public class TheErrorMessage implements Question<Boolean> {
    
    // Метод получения ответа на вопрос
    @Override
    public <T extends Actor> Boolean answeredBy(T actor) {
        Page page = actor.abilityTo(BrowseTheWeb.class).getPage();
        return page.locator(".error-message").isVisible(); // Возврат булева значения
    }
    
    // Статический фабричный метод для создания вопроса
    public static TheErrorMessage displayed() {
        return new TheErrorMessage();
    }
}
```

#### Использование Screenplay в тесте:

```java
class ScreenplayLoginTest {

    private Playwright playwright;
    private Browser browser;
    private Page page;
    private Actor user;

    @BeforeEach
    void setUp() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(false));
        page = browser.newPage();
        page.navigate("https://your-login-page.com");

        user = new Actor("Пользователь");
        user.can(BrowseTheWeb.with(page));
    }

    @Test
    void testInvalidLogin() {
        user.attemptsTo(Login.withCredentials("wrong@email.com", "invalid"));

        assertTrue(user.asksWhether(TheErrorMessage.displayed()));
    }

    @AfterEach
    void tearDown() {
        if (page != null) page.close();
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }
}
```

---

## Page-level абстракция vs нативный Playwright

> Сравнение подходов к организации кода: использование абстракций (POM/Component) против прямой работы с `Locator` в тестах.

| Критерий               | Page Object Model                      | Нативный Playwright                      |
|:-----------------------|----------------------------------------|------------------------------------------|
| Дублирование кода      | Минимальное (селекторы в одном месте)  | Высокое (селекторы повторяются в тестах) |
| Рефакторинг селекторов | Изменение в одном классе POM           | Изменение во всех тестах                 |
| Скорость разработки    | Медленнее (требует создания классов)   | Быстрее (прямое написание тестов)        |
| Читаемость тестов      | Высокая (бизнес-логика без деталей)    | Средняя (смесь селекторов и действий)    |
| Подходит для           | Крупных проектов, командной разработки | Маленьких проектов, прототипирования     |

#### Пример нативного подхода без абстракций:

```java
class NativePlaywrightTest {

    // ... 
    
    @Test
    void testLogin() {
        // Прямая работа с локаторами в тесте
        page.navigate("https://app.example.com/login");
        page.getByLabel("Email").fill("user@test.com");
        page.getByLabel("Password").fill("secret");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
        
        // Валидация
        assertThat(page.getByRole(AriaRole.HEADING)).toHaveText("Добро пожаловать");
    }
    
    @Test
    void testInvalidLogin() {
        // Повторение селекторов в другом тесте
        page.navigate("https://app.example.com/login");
        page.getByLabel("Email").fill("wrong@email.com");
        page.getByLabel("Password").fill("invalid");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
        
        // Валидация
        assertThat(page.locator(".error-message")).toBeVisible();
    }
}
```

---

## Организация проекта

> Структура директорий и пакетов влияет на поддерживаемость и скорость онбординга. 
> 
> Рекомендуется разделять тесты, Page Objects, компоненты, утилиты и конфигурацию.

#### Cхема структуры проекта:

```text
src/
├── test/
│   ├── java/
│   │   └── com.example.tests/
│   │       ├── pages/                    # Page Objects
│   │       │   ├── LoginPage.java
│   │       │   ├── DashboardPage.java
│   │       │   └── CartPage.java
│   │       ├── components/               # Переиспользуемые компоненты
│   │       │   ├── ModalComponent.java
│   │       │   ├── HeaderComponent.java
│   │       │   └── PaginationComponent.java
│   │       ├── tests/                    # Тестовые классы
│   │       │   ├── LoginFlowTest.java
│   │       │   ├── CheckoutFlowTest.java
│   │       │   └── ApiIntegrationTest.java
│   │       ├── api/                      # API-клиенты и DTO
│   │       │   ├── UserApiClient.java
│   │       │   └── ProductApiClient.java
│   │       ├── utils/                    # Утилиты и хелперы
│   │       │   ├── TestDataGenerator.java
│   │       │   └── ScreenshotHelper.java
│   │       ├── fixtures/                 # Фикстуры и DI
│   │       │   ├── BrowserFixture.java
│   │       │   └── PageFixture.java
│   │       └── config/                   # Конфигурационные классы
│   │           └── TestConfig.java
│   └── resources/
│       ├── test.properties               # Конфигурация окружений
│       ├── test-data/                    # Тестовые данные (JSON, CSV)
│       │   ├── users.json
│       │   └── products.csv
│       └── allure.properties             # Настройки Allure
└── main/
    └── java/                             # Исходный код приложения (если тесты внутри проекта)
```

#### Конфигурационный файл `test.properties`:

```properties
# Базовые URL для разных окружений
env.base.url=https://staging.example.com
env.api.url=https://api.staging.example.com

# Настройки браузера
browser.type=chromium
browser.headless=true
browser.slowMo=0

# Таймауты
timeout.default=30000
timeout.navigation=60000

# Пути к артефактам
artifacts.screenshots=target/screenshots
artifacts.traces=target/traces
artifacts.videos=target/videos
```

#### Класс загрузки конфигурации:

```java
// Утилита для загрузки конфигурации из test.properties
public class TestConfig {

    private static final Properties properties = new Properties();
    
    // Статический блок инициализации: загрузка файла при первом обращении к классу
    static {
        try (InputStream input = TestConfig.class.getClassLoader().getResourceAsStream("test.properties")) {
            if (input == null) {
                throw new RuntimeException("Не найден файл test.properties");
            }
            properties.load(input); // Загрузка свойств из файла
        } catch (IOException e) {
            throw new RuntimeException("Ошибка загрузки конфигурации", e);
        }
    }
    
    // Метод получения строкового свойства
    public static String get(String key) {
        return properties.getProperty(key); // Возврат значения по ключу
    }
    
    // Метод получения числового свойства с дефолтным значением
    public static int getInt(String key, int defaultValue) {
        String value = properties.getProperty(key);
        return value != null ? Integer.parseInt(value) : defaultValue; // Парсинг или дефолт
    }
    
    // Метод получения булева свойства
    public static boolean getBoolean(String key) {
        return Boolean.parseBoolean(properties.getProperty(key)); // Парсинг булева значения
    }
}
```

---

## Best Practices

- Используйте Page Object Model для проектов с более чем 10 тестами, это снижает дублирование и упрощает рефакторинг селекторов
- Выносите переиспользуемые виджеты (модальные окна, хедеры, пагинацию) в отдельные классы компонентов, 
  чтобы не дублировать логику в разных Page Objects
- Храните селекторы как приватные поля `Locator` в Page Objects, не возвращайте их наружу 
  — тесты должны работать через публичные методы
- Применяйте fluent-цепочки (`return this`) в Page Objects для навигационных методов, это улучшает читаемость тестов
- Разделяйте тесты, Page Objects, компоненты и утилиты по пакетам, это упрощает навигацию и снижает когнитивную нагрузку
- Централизуйте конфигурацию в `test.properties` или `.env`, не хардкодьте URL, таймауты и пути в коде
- Не создавайте Page Objects для одной страницы с одним тестом, это избыточная абстракция — используйте нативный Playwright
- Не возвращайте `Locator` из Page Objects в тесты, это нарушает инкапсуляцию и приводит к дублированию селекторов
- Не наследуйте Page Objects друг от друга без крайней необходимости, предпочитайте композицию через компоненты
- Не хардкодьте URL и таймауты в коде, выносите их в конфигурационные файлы для гибкого переключения окружений
- Не смешивайте бизнес-логику теста с деталями реализации (selectors, `page.click()`), используйте методы Page Objects
- Не создавайте глобальные статические `Page` или `BrowserContext` без `ThreadLocal`, это приводит к гонкам состояний при параллельных запусках

---

# 9. Продвинутые сценарии и изоляция тестов

> Playwright предоставляет встроенные механизмы для работы со сложными сценариями, 
> которые в Selenium требуют ручного переключения окон, выполнения JavaScript или привлечения сторонних библиотек. 
> 
> Многооконность, вложенные фреймы, сетевое мокирование, нативные диалоги и управление сессиями 
> — все эти задачи решаются декларативно через API `BrowserContext`, `Page` и `Route`, сохраняя авто-ожидания и типобезопасность.

## Работа с несколькими вкладками и окнами

> `BrowserContext` управляет всеми вкладками, открытыми в рамках одной сессии. 
>
> Новые окна (попапы) автоматически создаются в том же контексте, что позволяет переключаться между ними без потери cookies и localStorage.

- `Page waitForPopup(Runnable action)` - ожидание открытия новой вкладки в результате выполнения действия 
  (клик по `target="_blank"`, `window.open()`)
- `List<Page> pages()` - возврат списка всех открытых страниц в контексте
- `void bringToFront()` - перемещение страницы на передний план (эмуляция фокуса пользователя)
- `void close()` - закрытие конкретной вкладки без завершения контекста
- `BrowserContext waitForPage(Runnable action)` - ожидание создания новой страницы на уровне контекста
- `EventEmitter onPage(Consumer<Page> handler)` - подписка на событие открытия новой вкладки

#### Схема переключения вкладок:

```text
┌─────────────────────────────────────────────────────────────┐
│                    BrowserContext                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │   Page #1    │  │   Page #2    │  │   Page #3    │      │
│   │  (активная)  │  │   (попап)    │  │  (фоновая)   │      │
│   │  main.html   │  │  modal.html  │  │  report.html │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│         ▲                                                   │
│         │                                                   │
│  context.pages() → [Page#1, Page#2, Page#3]                 │
│  Page page1 = context.pages().get(0);                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

- Индекс не равен "активности". Если вы переключитесь на вкладку №2 программно (`page2.bringToFront()`), она станет активной визуально, 
  но ее индекс в списке `pages()` не изменится. Он всегда равен порядку создания.

- Закрытие вкладок. Если вы закроете `Page #1` (`page1.close()`), то индексы сдвинутся: `Page #2` станет `get(0)`, а `Page #3` — `get(1)`.

---

#### Пример работы с попапами:

```java
// Получение контекста браузера
BrowserContext context = page.context(); // Из существующей страницы
// или
BrowserContext context = browser.newContext(); // Создание нового контекста

// 1. Ожидание попапа через waitForPopup (рекомендуемый способ)
// Метод принимает Runnable с действием, вызывающим открытие новой вкладки
Page popup = page.waitForPopup(() -> {
    page.getByRole(AriaRole.LINK, new Page.GetByRoleOptions().setName("Открыть в новой вкладке"))
        .click(); // Клик по ссылке с target="_blank"
});

// popup теперь ссылается на новую вкладку, можно с ней работать
popup.waitForLoadState(LoadState.NETWORKIDLE); // Ожидание полной загрузки попапа
assertThat(popup.getByRole(AriaRole.HEADING)).toHaveText("Детали заказа");

// 2. Получение всех открытых вкладок
List<Page> allPages = context.pages();
System.out.println("Открыто вкладок: " + allPages.size()); // Например: 3

// 3. Переключение фокуса на конкретную вкладку
Page targetPage = allPages.stream()
    .filter(p -> p.url().contains("/dashboard")) // Поиск по URL
    .findFirst()
    .orElseThrow(() -> new RuntimeException("Вкладка не найдена"));

targetPage.bringToFront(); // Активация вкладки (эмуляция клика по ней)

// 4. Закрытие лишних вкладок для экономии памяти
for (Page p : context.pages()) {
    if (!p.url().contains("/checkout")) { // Оставляем только страницу оформления
        p.close(); // Закрытие вкладки без завершения контекста
    }
}
```

---

## Frames и iFrames

> `FrameLocator` — специализированный локатор для работы с `<iframe>`. 
> 
> В отличие от Selenium, где требуется ручное `driver.switchTo().frame()`, Playwright автоматически ограничивает 
> область поиска содержимым фрейма и поддерживает вложенность без явного переключения контекста.

- `FrameLocator frameLocator(String selector)` - получение фрейм-локатора по CSS-селектору iframe / рекурсивный доступ к вложенным фреймам
- `Locator locator(String selector)` - поиск элемента внутри фрейма
- `FrameLocator first()` / `last()` / `nth(int)` - выбор конкретного фрейма при наличии нескольких совпадений
- `Frame frameByName(String name)` / `frameByUrl(String pattern)` - поиск фрейма по имени или URL (для сложных сценариев)

#### Пример работы с вложенными фреймами:

```java
// 1. Простой доступ к элементу внутри одного iframe
FrameLocator paymentFrame = page.frameLocator("#payment-iframe");
paymentFrame.locator("input[name='cardNumber']").fill("4111 1111 1111 1111");
paymentFrame.locator("input[name='cvv']").fill("123");

// 2. Вложенные iframes (iframe внутри iframe)
// Структура: #outer-frame → #inner-frame → button.submit
Locator submitBtn = page.frameLocator("#outer-frame")
    .frameLocator("#inner-frame") // Рекурсивный доступ к вложенному фрейму
    .locator("button.submit");
submitBtn.click();

// 3. Обработка динамически загружаемых фреймов
// Фрейм может появиться в DOM после AJAX-запроса
page.waitForSelector("#dynamic-frame", new Page.WaitForSelectorOptions()
    .setState(WaitForSelectorState.ATTACHED)); // Ожидание появления фрейма в DOM
FrameLocator dynamicFrame = page.frameLocator("#dynamic-frame");
dynamicFrame.locator(".content-loaded").waitFor(); // Ожидание рендеринга содержимого

// 4. Обход CSP-ограничений через frameLocator
// Если iframe блокирует прямой доступ, Playwright всё равно может работать с ним
// через нативный протокол браузера (без нарушения Same-Origin Policy)
FrameLocator restrictedFrame = page.frameLocator("[name='restricted-widget']");
restrictedFrame.locator("body").textContent(); // Чтение содержимого без JS-ошибок

// 5. Поиск фрейма по URL (для случаев, когда селектор недоступен)
Frame frame = page.frames().stream()
    .filter(f -> f.url().contains("stripe.com")) // Фильтрация по домену
    .findFirst()
    .orElseThrow();
frame.locator("input[name='card-expiry']").fill("12/25");
```

---

## Сетевое мокирование и стабы

> `route()` — мощный механизм перехвата сетевых запросов на уровне `BrowserContext` или `Page`. 
> 
> Позволяет мокировать API-ответы, блокировать аналитику, эмулировать задержки и модифицировать заголовки без изменения кода приложения.

- `void route(String urlPattern, Route.Handler handler)` - регистрация обработчика для URL-паттерна (глобы `**/*`)
- `void routeFromHAR(Path harFile, RouteFromHAROptions options)` - воспроизведение записанных HAR-файлов как моков
  ```java
  // На уровне Page (применяется только к этой странице)
  Page page = context.newPage();
  page.route("**/api/**", route -> route.fulfill(...));
  page.routeFromHAR(Path.of("mocks.har"));

  // На уровне BrowserContext (применяется ко всем страницам в контексте)
  BrowserContext context = browser.newContext();
  context.route("**/api/**", route -> route.fulfill(...));
  context.routeFromHAR(Path.of("mocks.har"));
  ```
  `Page` моки имеют приоритет над `BrowserContext` моками
- `Route.fulfill(Route.FulfillOptions options)` - возврат кастомного ответа без обращения к серверу
- `Route.abort(String errorCode)` - принудительный обрыв запроса с кодом ошибки (`failed`, `timedout`, `accessdenied`)
- `Route.continue_(Route.ContinueOptions options)` - модификация запроса (URL, метод, заголовки) и передача дальше
- `Route.fallback(Route.Handler handler)` - цепочка обработчиков (если первый не fulfill/abort, управление передаётся следующему)

#### Примеры сетевого мокирования:

```java
// 1. Мокирование API-ответа с кастомным JSON
context.route("**/api/user/profile", route -> route.fulfill(
    new Route.FulfillOptions()
        .setStatus(200)                                    // HTTP-статус ответа
        .setContentType("application/json")                // Content-Type заголовок
        .setBody("{\"id\": 1, \"name\": \"Test User\", \"role\": \"admin\"}") // Тело ответа
));

// 2. Блокировка внешних ресурсов (аналитика, реклама, трекинг)
context.route("**/*", route -> {
    String url = route.request().url();
    // Список доменов для блокировки
    if (url.contains("google-analytics.com") || 
        url.contains("facebook.net") || 
        url.contains("doubleclick.net")) {
        route.abort("blockedbyclient"); // Обрыв запроса с кодом блокировки
    } else {
        route.continue_(); // Передача управления дальше для остальных запросов
    }
});

// 3. Эмуляция сетевых задержек (тестирование лоадеров и таймаутов)
context.route("**/api/slow-endpoint", route -> {
    try {
        Thread.sleep(3000); // Искусственная задержка 3 секунды
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // Восстановление флага прерывания
    }
    route.fulfill(new Route.FulfillOptions()
        .setStatus(200)
        .setBody("{\"status\": \"ok\"}"));
});

// 4. Модификация запроса перед отправкой (подмена токена, добавление заголовков)
context.route("**/api/secure/*", route -> route.continue_(
    new Route.ContinueOptions()
        .setHeaders(Map.of(                                  // Добавление кастомного заголовка
            "X-Test-Mode", "true",
            "Authorization", "Bearer test-token-123"
        ))
));

// 5. Воспроизведение HAR-файла как источника моков
// HAR-файл можно записать через браузер или Postman
context.routeFromHAR(Path.of("src/test/resources/api-mocks.har"), 
    new BrowserContext.RouteFromHAROptions()
        .setUpdate(false)                                    // Не обновлять HAR автоматически
        .setUrl("**/api/**")                                 // Применять только к API-запросам
        .setNotFound(Fallback.CONTINUE)                      // Пропускать запросы без моков
);
```

---

## Диалоги и алерты

> `page.onDialog()` — механизм перехвата нативных браузерных диалогов (`alert`, `confirm`, `prompt`, `beforeunload`). 
>
> В отличие от Selenium, где требуется ручная обработка через `Alert.accept()`, Playwright позволяет 
> декларативно задать поведение для всех диалогов через один хендлер.

- `EventEmitter onDialog(Consumer<Dialog> handler)` - подписка на событие появления диалога
- `void accept()` / `void accept(String promptText)` - подтверждение диалога с опциональным вводом текста
- `void dismiss()` - отмена диалога (эквивалент нажатия "Отмена")
- `String message()` - получение текста сообщения диалога
- `DialogType type()` - тип диалога (`ALERT`, `CONFIRM`, `PROMPT`, `BEFOREUNLOAD`)

Примеры обработки диалогов:

```java
// 1. Базовая обработка alert/confirm
page.onDialog(dialog -> {
    System.out.println("Тип диалога: " + dialog.type());   // ALERT, CONFIRM, PROMPT
    System.out.println("Сообщение: " + dialog.message());  // Текст диалога
    
    if (dialog.type() == DialogType.CONFIRM) {
        dialog.accept(); // Подтверждение (OK)
    } else {
        dialog.dismiss(); // Отмена (Cancel)
    }
});

// 2. Ввод текста в prompt-диалог
page.onDialog(dialog -> {
    if (dialog.type() == DialogType.PROMPT) {
        dialog.accept("Значение по умолчанию"); // Ввод текста и подтверждение
    }
});

// 3. Обработка beforeunload (предупреждение о несохранённых данных)
page.onDialog(dialog -> {
    if (dialog.type() == DialogType.BEFOREUNLOAD) {
        dialog.accept(); // Автоматическое подтверждение перехода
    }
});

// 4. Отключение нативных диалогов через глобальный хендлер
// Все диалоги будут автоматически закрываться
page.onDialog(Dialog::accept); // Лямбда-выражение: принять любой диалог

// 5. Валидация сообщения диалога в тесте
List<String> dialogMessages = new ArrayList<>();
page.onDialog(dialog -> {
    dialogMessages.add(dialog.message()); // Сохранение сообщения для проверки
    dialog.accept();
});

// Выполнение действия, вызывающего диалог
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Удалить")).click();

// Валидация
assertEquals("Вы уверены, что хотите удалить запись?", dialogMessages.get(0));
```

---

## Аутентификация и сессии

> `storageState()` — механизм сохранения и загрузки состояния авторизации (cookies, localStorage, sessionStorage) в JSON-файл. 
> 
> Позволяет однократно выполнить логин и переиспользовать сессию во всех последующих тестах, ускоряя прогон на 30–50%.

- `void storageState(Path path)` - сохранение текущего состояния в файл
- `StorageState storageState()` - возврат состояния в виде объекта для программной обработки
- `Browser.NewContextOptions setStorageState(Path path)` - загрузка состояния при создании контекста
- `void clearCookies()` / `void clearPermissions()` - очистка credentials и разрешений
- `void addCookies(List<Cookie> cookies)` - программное добавление cookies без логина

#### Примеры управления сессиями:

```java
// 1. Сохранение состояния после логина (выполняется один раз в @BeforeAll)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class AuthSetup {
    private BrowserContext authContext;
    
    @BeforeAll
    void saveAuthState() {
        authContext = browser.newContext();
        Page loginPage = authContext.newPage();
        
        // Выполнение логина
        loginPage.navigate("https://app.example.com/login");
        loginPage.getByLabel("Email").fill("admin@test.com");
        loginPage.getByLabel("Password").fill("secret");
        loginPage.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
        
        // Ожидание успешной авторизации
        loginPage.waitForURL("**/dashboard");
        
        // Сохранение состояния в файл (cookies + localStorage)
        authContext.storageState(new BrowserContext.StorageStateOptions()
            .setPath(Path.of("src/test/resources/auth-state.json")));
        
        authContext.close();
    }
}

// 2. Загрузка состояния в каждом тесте (пропуск экрана логина)
@BeforeEach
void setupContext() {
    context = browser.newContext(new Browser.NewContextOptions()
        .setStorageState(Path.of("src/test/resources/auth-state.json"))); // Загрузка сессии
    page = context.newPage();
}

@Test
void testDashboardAccess() {
    page.navigate("https://app.example.com/dashboard"); // Прямой переход без логина
    assertThat(page.getByRole(AriaRole.HEADING)).toHaveText("Панель управления");
}

// 3. Обход MFA/2FA через программное добавление cookies
// Если MFA-токен можно получить через API или тестовый секрет
List<Cookie> mfaCookies = List.of(
    new Cookie("mfa_verified", "true")
        .setDomain("app.example.com")
        .setPath("/")
        .setExpires(System.currentTimeMillis() / 1000 + 3600), // Срок действия 1 час
    new Cookie("session_id", "test-session-abc123")
        .setDomain("app.example.com")
        .setPath("/")
);
context.addCookies(mfaCookies); // Добавление cookies без логина

// 4. Тестирование ролей (RBAC) через разные storageState
// Подготовка: auth-admin.json, auth-user.json, auth-guest.json
@Test
void testAdminAccess() {
    BrowserContext adminContext = browser.newContext(new Browser.NewContextOptions()
        .setStorageState(Path.of("src/test/resources/auth-admin.json"))); // Роль администратора
    Page adminPage = adminContext.newPage();
    adminPage.navigate("https://app.example.com/admin/users");
    assertThat(adminPage.getByText("Список пользователей")).toBeVisible();
    adminContext.close();
}

@Test
void testUserAccessDenied() {
    BrowserContext userContext = browser.newContext(new Browser.NewContextOptions()
        .setStorageState(Path.of("src/test/resources/auth-user.json"))); // Роль обычного пользователя
    Page userPage = userContext.newPage();
    userPage.navigate("https://app.example.com/admin/users");
    assertThat(userPage.getByText("Доступ запрещён")).toBeVisible(); // Проверка отказа
    userContext.close();
}

// 5. Очистка credentials между тестами
@AfterEach
void cleanupSession() {
    context.clearCookies(); // Удаление всех cookies
    context.clearPermissions(); // Сброс разрешений (геолокация, уведомления)
    // localStorage очищается автоматически при context.close()
}
```

---

## Best Practices

- Используйте `waitForPopup()` вместо ручного перебора `context.pages()`, это гарантирует корректную синхронизацию 
  с асинхронным открытием вкладок
- Применяйте `frameLocator()` для работы с iframe, избегайте `page.frames()` и ручного поиска 
  — это сохраняет авто-ожидания и типобезопасность
- Блокируйте аналитику и трекинг через `route()` с `abort()`, это ускоряет тесты на 20–30% и снижает нагрузку на внешние сервисы
- Сохраняйте `storageState` один раз в `@BeforeAll` и загружайте его в каждом тесте, это ускоряет прогон на 30–50% 
  по сравнению с повторным логином
- Обрабатывайте `beforeunload` через `onDialog(Dialog::accept)`, чтобы избежать блокировки тестов при навигации с несохранёнными данными
- Используйте `routeFromHAR()` для воспроизведения записанных API-ответов, это упрощает поддержку моков и делает их версионируемыми
- Не переключайтесь между вкладками через `context.pages().get(index)`, это хрупкий подход — используйте `waitForPopup()` 
  или фильтрацию по URL
- Не игнорируйте вложенные iframe, Playwright поддерживает рекурсивный доступ через `frameLocator().frameLocator()`, 
  не пишите кастомный JS для обхода
- Не модифицируйте запросы через `page.evaluate("fetch(...)")`, используйте `route()` — это декларативно, 
  типобезопасно и интегрировано с трейсами
- Не обрабатывайте диалоги через `Thread.sleep()` в ожидании их появления, используйте `onDialog()` 
  — хендлер вызывается автоматически при появлении диалога
- Не логиньтесь в каждом тесте заново, сохраняйте `storageState` и переиспользуйте его — это критично для производительности CI/CD
- Не храните credentials в коде, выносите их в `.env` или CI-secrets, загружайте через `System.getenv()` или конфигурационные файлы

---

# 10. Отчётность, трейсы и артефакты

> Playwright предоставляет встроенный набор инструментов для диагностики падений: скриншоты, видеозаписи сессий 
> и детальные трейсы с DOM-снэпшотами, сетевыми логами и исходным кодом шагов. 
> 
> Интеграция с Allure и ReportPortal позволяет централизованно собирать артефакты, визуализировать падения 
> и передавать их команде для быстрого воспроизведения.

## Встроенные инструменты

Playwright предоставляет три основных типа артефактов, которые можно собирать как на уровне `Page`, так и на уровне `BrowserContext`. 

Все инструменты работают в headless-режиме и не требуют дополнительных зависимостей.

- `byte[] screenshot()` / `byte[] screenshot(ScreenshotOptions)` - получение PNG-изображения страницы или отдельного элемента 
  в виде массива байтов
- `Path saveScreenshot(Path path)` - сохранение скриншота напрямую в файл
- `void saveAsVideo(Path path)` - сохранение записанного видео в указанный файл
- `void startRecording(Tracing.StartOptions options)` - начало записи трейса с настройками (скриншоты, DOM-снэпшоты, исходный код)
- `void stop(Path path)` - завершение записи и сохранение трейса в `.zip` файл
- `void startChunk()` / `void stopChunk(Path path)` - разбиение трейса на отдельные чанки (полезно для группировки по тестам)

#### Схема типов артефактов:

```text
┌─────────────────────────────────────────────────────────────┐
│                  АРТЕФАКТЫ PLAYWRIGHT                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. SCREENSHOT (статичное изображение)                      │
│     ├─► page.screenshot() → byte[] PNG                      │
│     ├─► locator.screenshot() → элемент целиком              │
│     └─► Сохранение: Path.of("target/screenshots/test.png")  │
│                                                             │
│  2. VIDEO (запись всей сессии)                              │
│     ├─► context.newContext(setRecordVideoDir) → автозапись  │
│     ├─► Формат: .webm (Chromium), .webm (Firefox)           │
│     └─► Сохранение: page.video().saveAs(path)               │
│                                                             │
│  3. TRACE (детальная трассировка)                           │
│     ├─► context.tracing().start(options)                    │
│     ├─► Содержит: скриншоты + DOM + network + source code   │
│     ├─► Формат: .zip архив                                  │
│     └─► Просмотр: playwright show-trace trace.zip           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Пример настройки и сбора артефактов:

```java
// 1. Настройка записи видео на уровне контекста
// Видео автоматически записывается для всех страниц в этом контексте
Browser.NewContextOptions videoOptions = new Browser.NewContextOptions()
    .setRecordVideoDir(Paths.get("target/videos"))           // Директория для сохранения видео
    .setRecordVideoSize(new RecordVideoSize()                // Настройка разрешения видео
        .setWidth(1280)                                      // Ширина в пикселях
        .setHeight(720));                                    // Высота в пикселях

BrowserContext videoContext = browser.newContext(videoOptions);
Page videoPage = videoContext.newPage();
videoPage.navigate("https://app.example.com");
// ... выполнение действий ...

// Сохранение видео после завершения теста
// Важно: видео доступно только после закрытия страницы
videoPage.close();                                                  // Закрытие страницы финализирует видео
videoPage.video().saveAs(Paths.get("target/videos/test-run.webm")); // Явное сохранение в нужный путь
videoContext.close();                                               // Закрытие контекста освобождает ресурсы

// 2. Настройка трассировки с максимальным уровнем детализации
Browser.NewContextOptions traceOptions = new Browser.NewContextOptions();
BrowserContext traceContext = browser.newContext(traceOptions);

// Начало записи трейса с настройками
traceContext.tracing().start(new Tracing.StartOptions()
    .setScreenshots(true)                                       // Скриншоты на каждом шаге
    .setSnapshots(true)                                         // DOM-снэпшоты для инспекции элементов
    .setSources(true)                                           // Исходный код Java-шагов
    .setTitle("Checkout Flow Test"));                           // Название для идентификации в viewer

Page tracePage = traceContext.newPage();
tracePage.navigate("https://app.example.com/checkout");
tracePage.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Оплатить")).click();

// Завершение записи и сохранение трейса в zip-архив
traceContext.tracing().stop(new Tracing.StopOptions()
    .setPath(Paths.get("target/traces/checkout-flow.zip")));    // Путь для сохранения архива

// 3. Скриншот отдельного элемента (не всей страницы)
// Полезно для валидации конкретных компонентов: кнопок, форм, виджетов
byte[] elementScreenshot = page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit"))
    .screenshot();                                              // Возврат byte[] PNG

// Сохранение скриншота элемента в файл
page.locator(".payment-form").screenshot(new Locator.ScreenshotOptions()
    .setPath(Paths.get("target/screenshots/payment-form.png"))); // Прямое сохранение в файл

// 4. Полностраничный скриншот с настройками
page.screenshot(new Page.ScreenshotOptions()
    .setPath(Paths.get("target/screenshots/full-page.png"))  // Путь сохранения
    .setFullPage(true)                                       // Скриншот всей страницы (с прокруткой)
    .setType(ScreenshotType.PNG)                             // Формат: PNG, JPEG
    .setOmitBackground(true));                               // Прозрачный фон (для PNG)
```

---

## Интеграция с Allure

> Allure — популярный фреймворк для генерации интерактивных HTML-отчётов. 
> 
> Интеграция с Playwright позволяет автоматически вкладывать скриншоты, видео и трейсы в отчёт, 
> а также группировать шаги через аннотации `@Step` и `@Attachment`.

- `@Step("Описание шага")` - аннотация для маркировки метода как шага в отчёте Allure
- `@Attachment(name = "Имя", type = "image/png")` - вложение файла (скриншот, видео, трейс) в отчёт
- `Allure.step("Описание")` - программное создание шага без аннотации (для лямбд и циклов)
- `Allure.addAttachment(name, content)` - добавление текстового или бинарного вложения
- `AllureLifecycle` - низкоуровневый API для управления жизненным циклом шагов и вложений
- `@Severity(SeverityLevel.CRITICAL)` - маркировка важности теста для фильтрации в отчёте
- `@TmsLink("TEST-123")` / `@Issue("BUG-456")` - связывание теста с задачами в TMS/баг-трекере

Пример интеграции с Allure:

```java
// Подключение Allure-расширения для JUnit 5
@ExtendWith(AllureJunit5.class)
class CheckoutFlowTest {
    
    private Page page;
    private BrowserContext context;
    
    @BeforeEach
    void setup() {
        context = browser.newContext();
        page = context.newPage();
    }
    
    @AfterEach
    void teardown(TestInfo testInfo) {
        // Сохранение артефактов только при падении теста
        if (testInfo.getDisplayName().contains("FAILED") || 
            !testInfo.getTags().isEmpty()) {
            
            // 1. Скриншот при падении
            byte[] screenshot = page.screenshot();
            Allure.addAttachment(                          // Вложение скриншота в отчёт
                "Screenshot on Failure",                   // Название вложения
                "image/png",                               // MIME-тип
                new ByteArrayInputStream(screenshot),      // Поток с данными
                "png");                                    // Расширение файла
            
            // 2. Видео сессии (если записывалось)
            if (page.video() != null) {
                Path videoPath = Paths.get("target/videos/" + testInfo.getDisplayName() + ".webm");
                page.video().saveAs(videoPath);            // Сохранение видео в файл
                Allure.addAttachment(                      // Вложение видео
                    "Session Video",                       // Название
                    "video/webm",                          // MIME-тип
                    videoPath);                            // Путь к файлу
            }
            
            // 3. Трейс для детального анализа
            Path tracePath = Paths.get("target/traces/" + testInfo.getDisplayName() + ".zip");
            context.tracing().stop(new Tracing.StopOptions().setPath(tracePath));
            Allure.addAttachment(                          // Вложение трейса
                "Playwright Trace",                        // Название
                "application/zip",                         // MIME-тип
                tracePath);                                // Путь к архиву
        }
        
        context.close();                                   // Закрытие контекста
    }
    
    @Test
    @Severity(SeverityLevel.CRITICAL)                      // Критический тест
    @TmsLink("TEST-123")                                   // Ссылка на тест-кейс
    @Issue("BUG-456")                                      // Ссылка на баг (если тест регрессионный)
    @Feature("Checkout")                                   // Группировка по функциональности
    @Story("Payment Flow")                                 // Подгруппа внутри фичи
    void testSuccessfulCheckout() {
        // Шаг 1: Навигация на страницу корзины
        Allure.step("Открытие страницы корзины", () -> {   // Программное создание шага
            page.navigate("https://app.example.com/cart");
            assertThat(page.getByRole(AriaRole.HEADING)).toHaveText("Корзина");
        });
        
        // Шаг 2: Переход к оформлению заказа
        navigateToCheckout();                              // Вызов метода с @Step
        
        // Шаг 3: Заполнение формы доставки
        fillShippingForm();
        
        // Шаг 4: Оплата
        processPayment();
        
        // Шаг 5: Валидация успешного завершения
        verifyOrderConfirmation();
    }
    
    @Step("Переход к оформлению заказа")                   // Аннотация шага
    void navigateToCheckout() {
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Оформить заказ")).click();
        page.waitForURL("**/checkout");                    // Ожидание перехода
    }
    
    @Step("Заполнение формы доставки")
    void fillShippingForm() {
        page.getByLabel("Имя").fill("Иван Петров");        // Ввод имени
        page.getByLabel("Адрес").fill("ул. Ленина, д. 1"); // Ввод адреса
        page.getByLabel("Город").fill("Москва");           // Ввод города
        page.getByLabel("Индекс").fill("101000");          // Ввод индекса
    }
    
    @Step("Обработка оплаты")
    void processPayment() {
        page.getByLabel("Номер карты").fill("4111 1111 1111 1111");
        page.getByLabel("Срок действия").fill("12/25");
        page.getByLabel("CVV").fill("123");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Оплатить")).click();
    }
    
    @Step("Проверка подтверждения заказа")
    void verifyOrderConfirmation() {
        assertThat(page.getByRole(AriaRole.HEADING))
            .toHaveText("Заказ успешно оформлен");         // Валидация успешного завершения
    }
}
```

#### Конфигурация `allure.properties`:

```properties
# Базовая конфигурация Allure
allure.results.directory=target/allure-results                  # Директория для промежуточных результатов
allure.link.tms.pattern=https://tms.example.com/browse/{}       # Шаблон ссылки на TMS
allure.link.issue.pattern=https://jira.example.com/browse/{}    # Шаблон ссылки на баг-трекер

# Настройки отображения
allure.report.name=Playwright Test Report                  # Название отчёта
allure.report.directory=target/allure-report               # Директория для финального HTML-отчёта
```

Maven-плагин для генерации отчёта:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-maven</artifactId>
            <version>2.12.0</version>
            <configuration>
                <resultsDirectory>${project.build.directory}/allure-results</resultsDirectory>
                <reportDirectory>${project.build.directory}/allure-report</reportDirectory>
            </configuration>
            <executions>
                <execution>
                    <id>allure-report</id>                 <!-- Идентификатор выполнения -->
                    <phase>post-integration-test</phase>   <!-- Фаза сборки для генерации -->
                    <goals>
                        <goal>report</goal>                <!-- Цель: генерация HTML-отчёта -->
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

## ReportPortal (опционально)

> ReportPortal — серверная платформа для агрегации результатов тестов с аналитикой, историей запусков и интеграцией с CI/CD. 
>
> В отличие от Allure (статичный HTML), ReportPortal хранит данные в базе и предоставляет REST API для запросов.

- `ReportPortalExtension` - JUnit 5 расширение для автоматической отправки результатов в ReportPortal
- `TestNGListener` - листенер TestNG для интеграции с ReportPortal
- `@TestCaseId("TC-123")` - связывание теста с идентификатором в TMS
- `attributes` - кастомные теги для фильтрации запусков (окружение, браузер, версия)
- `launch` - группировка тестовых запусков по признакам (regression, smoke, nightly)

Пример интеграции с ReportPortal:

```java
import com.epam.reportportal.junit5.ReportPortalExtension;
import com.epam.reportportal.annotations.TestCaseId;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import java.util.Map;

// Подключение ReportPortal-расширения
@ExtendWith(ReportPortalExtension.class)
class RegressionTest {
    
    @Test
    @TestCaseId("TC-456")                                  // Связь с тест-кейсом в TMS
    @DisplayName("Проверка добавления товара в корзину")
    void testAddToCart() {
        // Установка атрибутов запуска для фильтрации в ReportPortal
        ReportPortalExtension.emitAttributes(Map.of(
            "browser", "chromium",                         // Атрибут браузера
            "env", "staging",                              // Атрибут окружения
            "version", "2.5.1"                             // Атрибут версии приложения
        ));
        
        // Тестовая логика
        page.navigate("https://staging.example.com/products");
        page.getByTestId("product-123").click();
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("В корзину")).click();
        
        assertThat(page.locator(".cart-count")).toHaveText("1");
    }
}
```

Конфигурация `reportportal.properties`:

```properties
# Подключение к серверу ReportPortal
rp.endpoint=https://reportportal.example.com               # URL сервера
rp.uuid=your-api-token-here                                # API-токен для аутентификации
rp.project=my-project                                      # Название проекта в ReportPortal
rp.launch=Playwright Regression                            # Название запуска

# Настройки логирования
rp.enable=true                                             # Включение интеграции
rp.mode=DEFAULT                                            # Режим работы (DEFAULT, DEBUG)
rp.description=Automated UI tests with Playwright          # Описание запуска
```

---

Нет, этот код **не будет работать** как ожидается.

`TestInfo.getDisplayName()` возвращает только метаданные теста (имя метода, параметры), но **не его результат**. Он никогда не содержит строку `"FAILED"` — там будет что-то вроде `testSuccessfulCheckout()` или `[1] user@test.com`.

В JUnit 5 для реакции на результат теста существует интерфейс `TestWatcher` с методами `testFailed()`, `testSuccessful()`, `testAborted()`, `testDisabled()`. Это правильный механизм для сохранения артефактов только при падении.

## Настройка артефактов

> Правильная организация путей, имён файлов и ротации артефактов критична для production-запусков.
>
> Избыточное хранение скриншотов и видео быстро исчерпывает дисковое пространство CI-агентов.

- `timestamp` - добавление временной метки к имени файла для уникальности
- `testName` - использование имени теста в имени файла для быстрой идентификации
- `rotation` - автоматическое удаление старых артефактов старше N дней
- `cleanup` - очистка директории артефактов перед каждым запуском
- `TestWatcher` - интерфейс JUnit 5 для реакции на результат теста (упал/прошёл/пропущен)
- `ExtensionContext.Store` - хранилище для передачи состояния между callback-ами раннера
- `CI-artifact storage` - интеграция с GitHub Actions, Jenkins, S3 для хранения артефактов после прогона

#### ArtifactManager: утилиты для работы с файлами

```java
public class ArtifactManager {
    
    // Директории для разных типов артефактов
    private static final Path SCREENSHOTS_DIR = Paths.get("target/screenshots");
    private static final Path VIDEOS_DIR = Paths.get("target/videos");
    private static final Path TRACES_DIR = Paths.get("target/traces");
    
    // Период хранения артефактов перед автоматическим удалением
    private static final int MAX_AGE_DAYS = 7;
    
    // Формат временной метки для уникальных имён файлов
    private static final DateTimeFormatter TIMESTAMP_FORMAT = 
        DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss");
    
    // Инициализация директорий с очисткой устаревших файлов
    public static void initializeArtifactDirectories() throws IOException {
        // Создание директорий, если они не существуют
        Files.createDirectories(SCREENSHOTS_DIR);
        Files.createDirectories(VIDEOS_DIR);
        Files.createDirectories(TRACES_DIR);
        
        // Очистка старых артефактов во всех директориях
        cleanOldArtifacts(SCREENSHOTS_DIR);
        cleanOldArtifacts(VIDEOS_DIR);
        cleanOldArtifacts(TRACES_DIR);
    }
    
    // Очистка файлов старше MAX_AGE_DAYS в указанной директории
    private static void cleanOldArtifacts(Path directory) throws IOException {
        // Вычисление даты отсечения: текущее время минус MAX_AGE_DAYS
        LocalDateTime cutoff = LocalDateTime.now().minusDays(MAX_AGE_DAYS);
        
        // Рекурсивный обход всех файлов в директории
        try (Stream<Path> paths = Files.walk(directory)) {
            paths.filter(Files::isRegularFile)                                    // Только файлы, не директории
                .filter(path -> isOlderThan(path, cutoff))                        // Фильтр по дате модификации
                .forEach(path -> deleteQuietly(path));                            // Удаление без выброса исключений
        }
    }
    
    // Проверка, старше ли файл указанной даты отсечения
    private static boolean isOlderThan(Path path, LocalDateTime cutoff) {
        try {
            // Получение времени последней модификации файла
            LocalDateTime fileTime = LocalDateTime.ofInstant(
                Files.getLastModifiedTime(path).toInstant(),
                ZoneId.systemDefault());
            return fileTime.isBefore(cutoff);                                     // true, если файл старше cutoff
        } catch (IOException e) {
            // При ошибке чтения метаданных пропускаем файл
            return false;
        }
    }
    
    // Тихое удаление файла с логированием ошибок
    private static void deleteQuietly(Path path) {
        try {
            Files.delete(path);                                                   // Удаление файла
            System.out.println("Удалён старый артефакт: " + path);
        } catch (IOException e) {
            System.err.println("Ошибка удаления: " + path + " - " + e.getMessage());
        }
    }
    
    // Генерация уникального имени файла на основе имени теста и timestamp
    public static String generateArtifactName(String testName, String extension) {
        // Формирование временной метки: 20260618_143025
        String timestamp = LocalDateTime.now().format(TIMESTAMP_FORMAT);
        
        // Санитизация имени теста: замена недопустимых символов и ограничение длины
        String sanitizedTestName = testName
            .replaceAll("[^a-zA-Z0-9_-]", "_")                                    // Замена спецсимволов на _
            .replaceAll("_+", "_")                                                // Схлопывание множественных _
            .substring(0, Math.min(testName.length(), 50));                       // Ограничение длины 50 символов
        
        // Финальное имя: testName_YYYYMMDD_HHMMSS.extension
        return sanitizedTestName + "_" + timestamp + "." + extension;
    }
    
    // Сохранение полностраничного скриншота с уникальным именем
    public static Path saveScreenshot(Page page, String testName) throws IOException {
        String fileName = generateArtifactName(testName, "png");                  // Генерация имени файла
        Path filePath = SCREENSHOTS_DIR.resolve(fileName);                        // Полный путь к файлу
        
        // Создание скриншота с сохранением в файл
        page.screenshot(new Page.ScreenshotOptions()
            .setPath(filePath)                                                    // Путь сохранения
            .setFullPage(true));                                                  // Полностраничный скриншот (с прокруткой)
        
        return filePath;                                                          // Возврат пути для логирования
    }
    
    // Сохранение видео сессии с уникальным именем
    public static Path saveVideo(Page page, String testName) throws IOException {
        if (page.video() == null) {                                               // Проверка, записывалось ли видео
            return null;                                                          // Видео не записывалось
        }
        
        String fileName = generateArtifactName(testName, "webm");                 // Генерация имени файла
        Path filePath = VIDEOS_DIR.resolve(fileName);                             // Полный путь к файлу
        
        // Сохранение видео в файл (доступно только после page.close())
        page.video().saveAs(filePath);
        
        return filePath;
    }
    
    // Сохранение трейса с уникальным именем
    public static Path saveTrace(com.microsoft.playwright.Tracing tracing, 
                                  String testName) throws IOException {
        String fileName = generateArtifactName(testName, "zip");                  // Генерация имени файла
        Path filePath = TRACES_DIR.resolve(fileName);                             // Полный путь к файлу
        
        // Остановка записи трейса с сохранением в zip-архив
        tracing.stop(new com.microsoft.playwright.Tracing.StopOptions()
            .setPath(filePath));
        
        return filePath;
    }
}
```

#### ArtifactCollector: расширение JUnit 5 для сбора артефактов при падениях

```java
// Расширение JUnit 5, реагирующее на результат выполнения теста
// Реализует TestWatcher для перехвата событий testFailed/testSuccessful
public class ArtifactCollector implements TestWatcher {
    
    // Ключ для хранения Page в ExtensionContext.Store
    private static final String PAGE_KEY = "playwright.page";
    
    // Ключ для хранения BrowserContext в ExtensionContext.Store
    private static final String CONTEXT_KEY = "playwright.context";
    
    // Ключ для хранения флага "тест упал"
    private static final String FAILED_KEY = "test.failed";
    
    // Вызывается при падении теста (AssertionError, Exception и т.д.)
    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Установка флага падения в Store для последующей обработки
        getStore(context).put(FAILED_KEY, Boolean.TRUE);
        
        // Логирование причины падения
        System.err.println("Тест упал: " + context.getDisplayName() + 
                           " - " + cause.getMessage());
        
        // Сохранение артефактов (скриншот, видео, трейс)
        saveArtifactsOnFailure(context);
    }
    
    // Вызывается при успешном завершении теста
    @Override
    public void testSuccessful(ExtensionContext context) {
        // При успешном тесте артефакты можно удалить для экономии места
        // Или оставить для анализа — зависит от политики проекта
        System.out.println("Тест успешен: " + context.getDisplayName());
    }
    
    // Вызывается при прерывании теста (AssumptionFailed)
    @Override
    public void testAborted(ExtensionContext context, Throwable cause) {
        System.out.println("Тест прерван: " + context.getDisplayName());
    }
    
    // Вызывается при отключении теста (@Disabled)
    @Override
    public void testDisabled(ExtensionContext context, java.util.Optional<String> reason) {
        System.out.println("Тест отключён: " + context.getDisplayName() + 
                           " - " + reason.orElse("без причины"));
    }
    
    // Сохранение артефактов при падении теста
    private void saveArtifactsOnFailure(ExtensionContext context) {
        try {
            // Получение Page из Store (устанавливается в @BeforeEach или фикстуре)
            Page page = getStore(context).get(PAGE_KEY, Page.class);
            BrowserContext browserContext = getStore(context).get(CONTEXT_KEY, BrowserContext.class);
            
            // Имя теста для генерации уникальных имён файлов
            String testName = context.getDisplayName();
            
            if (page != null) {
                // Сохранение скриншота
                java.nio.file.Path screenshotPath = ArtifactManager.saveScreenshot(page, testName);
                System.out.println("Скриншот сохранён: " + screenshotPath);
                
                // Сохранение видео (если записывалось)
                java.nio.file.Path videoPath = ArtifactManager.saveVideo(page, testName);
                if (videoPath != null) {
                    System.out.println("Видео сохранено: " + videoPath);
                }
            }
            
            // Сохранение трейса (если записывался)
            if (browserContext != null) {
                java.nio.file.Path tracePath = ArtifactManager.saveTrace(
                    browserContext.tracing(), testName);
                System.out.println("Трейс сохранён: " + tracePath);
            }
        } catch (Exception e) {
            // Ошибки при сохранении артефактов не должны ломать тест
            System.err.println("Ошибка сохранения артефактов: " + e.getMessage());
        }
    }
    
    // Получение Store для текущего тестового контекста
    private ExtensionContext.Store getStore(ExtensionContext context) {
        // Используем namespace на основе класса теста для изоляции
        return context.getStore(
            ExtensionContext.Namespace.create(getClass(), context.getRequiredTestClass()));
    }
    
    // Статический метод для регистрации Page в Store (вызывается из @BeforeEach)
    public static void registerPage(ExtensionContext context, Page page) {
        context.getStore(ExtensionContext.Namespace.create(
                ArtifactCollector.class, context.getRequiredTestClass()))
            .put(PAGE_KEY, page);
    }
    
    // Статический метод для регистрации BrowserContext в Store
    public static void registerContext(ExtensionContext context, BrowserContext browserContext) {
        context.getStore(ExtensionContext.Namespace.create(
                ArtifactCollector.class, context.getRequiredTestClass()))
            .put(CONTEXT_KEY, browserContext);
    }
    
    // Проверка, упал ли текущий тест (для использования в @AfterEach)
    public static boolean isTestFailed(ExtensionContext context) {
        Boolean failed = context.getStore(ExtensionContext.Namespace.create(
                ArtifactCollector.class, context.getRequiredTestClass()))
            .get(FAILED_KEY, Boolean.class);
        return Boolean.TRUE.equals(failed);
    }
}
```

#### Использование ArtifactCollector в тестах

```java
// Подключение расширения ArtifactCollector к тестовому классу
@ExtendWith(ArtifactCollector.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CheckoutFlowTest {
    
    private Playwright playwright;
    private Browser browser;
    private BrowserContext context;
    private Page page;
    
    // Однократная инициализация Playwright и Browser на весь класс
    @BeforeAll
    void launchBrowser() throws IOException {
        // Инициализация директорий артефактов с очисткой старых файлов
        ArtifactManager.initializeArtifactDirectories();
        
        // Запуск Playwright и браузера
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions().setHeadless(true));
    }
    
    // Инициализация изолированного контекста перед каждым тестом
    @BeforeEach
    void setupContext(ExtensionContext extensionContext) {
        // Создание нового контекста с записью видео
        context = browser.newContext(new Browser.NewContextOptions()
            .setRecordVideoDir(Paths.get("target/videos")));
        page = context.newPage();
        
        // Начало записи трейса для последующего сохранения при падении
        context.tracing().start(new Tracing.StartOptions()
            .setScreenshots(true)                                               // Скриншоты на каждом шаге
            .setSnapshots(true)                                                 // DOM-снэпшоты
            .setSources(true));                                                 // Исходный код шагов
        
        // Регистрация Page и BrowserContext в Store расширения ArtifactCollector
        ArtifactCollector.registerPage(extensionContext, page);
        ArtifactCollector.registerContext(extensionContext, context);
    }
    
    // Очистка контекста после каждого теста
    @AfterEach
    void teardownContext(ExtensionContext extensionContext) {
        // Проверка статуса теста через ArtifactCollector
        boolean failed = ArtifactCollector.isTestFailed(extensionContext);
        
        if (context != null) {
            // Остановка записи трейса
            // Если тест упал, артефакты уже сохранены в ArtifactCollector.testFailed()
            // Если тест успешен, трейс можно удалить для экономии места
            if (!failed) {
                context.tracing().stop();                                       // Просто остановить без сохранения
            }
            
            // Закрытие контекста (обязательно для финализации видео)
            context.close();
        }
    }
    
    // Закрытие браузера после всех тестов
    @AfterAll
    void closeBrowser() {
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }
    
    // Успешный тест: артефакты не сохраняются (или удаляются)
    @Test
    void testSuccessfulCheckout() {
        page.navigate("https://app.example.com/checkout");
        page.getByRole(AriaRole.BUTTON, 
            new Page.GetByRoleOptions().setName("Оплатить")).click();
        // ... валидация успешного оформления
    }
    
    // Падающий тест: ArtifactCollector автоматически сохранит скриншот, видео и трейс
    @Test
    void testFailedPayment() {
        page.navigate("https://app.example.com/checkout");
        page.getByLabel("Номер карты").fill("invalid-card");
        page.getByRole(AriaRole.BUTTON, 
            new Page.GetByRoleOptions().setName("Оплатить")).click();
        
        // Это утверждение упадёт, и ArtifactCollector.testFailed() сохранит артефакты
        Assertions.assertTrue(page.getByText("Ошибка оплаты").isVisible());
    }
}
```

---

### Интеграция с CI-artifact storage (GitHub Actions):

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4                        # Checkout кода
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps          # Установка браузеров
      
      - name: Run tests
        run: mvn test                                    # Запуск тестов
      
      - name: Upload test artifacts                      # Загрузка артефактов в GitHub
        if: always()                                     # Загрузка даже при падении
        uses: actions/upload-artifact@v4
        with:
          name: playwright-artifacts                     # Название артефакта
          path: |                                        # Пути к артефактам
            target/screenshots/
            target/videos/
            target/traces/
            target/allure-results/
          retention-days: 7                              # Хранение 7 дней
```

---

## Debugging UI

> Playwright предоставляет мощный инструмент `Trace Viewer` для интерактивного анализа трейсов.
>
> Позволяет воспроизвести каждый шаг теста, просматривать DOM-дерево в момент времени, просмотреть сетевые запросы и консольные логи.
>
> Это полноценное веб-приложение для отладки, которое открывается в браузере и предоставляет богатый интерактивный интерфейс 
> для анализа выполнения тестов.

- `playwright show-trace trace.zip` - открытие Trace Viewer в браузере
- `Инспектор элементов` - выбор элемента на скриншоте и просмотр его селекторов
- `Network Tab` - анализ всех HTTP-запросов с таймингами и payload
- `Console Tab` - просмотр логов консоли браузера
- `Source Tab` - просмотр исходного кода Java-шагов с привязкой к таймлайну
- `Actionability Timeline` - визуализация проверок actionability (visible, enabled, stable)

#### Схема интерфейса Trace Viewer:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PLAYWRIGHT TRACE VIEWER                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TIMELINE (горизонтальная шкала времени)                                    │
│  ├─► [Step 1] ─── [Step 2] ─── [Step 3] ─── [Step 4] ─── [Step 5]           │ 
│  │    navigate    click        fill        click       assert               │
│  │    /login      "Submit"    "email"     "Login"     "Welcome"             │
│  │                                                                          │
│  └──────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  SCREENSHOT (скриншот текущего шага)                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │                                                             │    │    │
│  │  │              [Визуальное представление страницы]            │    │    │
│  │  │                                                             │    │    │
│  │  │         ┌──────────────┐                                    │    │    │
│  │  │         │  Email Input │ ← Инспектор: можно кликнуть        │    │    │
│  │  │         └──────────────┘   и увидеть селекторы              │    │    │
│  │  │                                                             │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  DOM SNAPSHOT (снимок DOM-дерева в момент шага)                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  <form id="login-form">                                             │    │
│  │    <input type="email" name="email" value="user@test.com" />        │    │
│  │    <input type="password" name="password" />                        │    │
│  │    <button type="submit">Login</button>                             │    │
│  │  </form>                                                            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  NETWORK TAB (сетевые запросы)                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  [GET] /api/user/profile → 200 OK (45ms)                            │    │
│  │  [POST] /api/login → 200 OK (120ms)                                 │    │
│  │  [GET] /api/dashboard → 200 OK (89ms)                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  CONSOLE TAB (логи консоли браузера)                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  [INFO] Application loaded successfully                             │    │
│  │  [WARN] Deprecated API usage detected                               │    │
│  │  [ERROR] Failed to load resource: 404                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Пример использования Trace Viewer для отладки:

```java
// 1. Запись трейса с максимальным уровнем детализации
context.tracing().start(new Tracing.StartOptions()
    .setScreenshots(true)                                    // Скриншоты на каждом шаге
    .setSnapshots(true)                                      // DOM-снэпшоты
    .setSources(true)                                        // Исходный код Java
    .setTitle("Debug Session"));                             // Название для идентификации

// 2. Выполнение тестовых действий
page.navigate("https://app.example.com");
page.getByLabel("Email").fill("user@test.com");
page.getByLabel("Password").fill("secret");
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Login")).click();

// 3. Сохранение трейса
Path tracePath = Paths.get("target/traces/debug-session.zip");
context.tracing().stop(new Tracing.StopOptions().setPath(tracePath));

// 4. Открытие Trace Viewer через CLI
// Команда выполняется в терминале:
// npx playwright show-trace target/traces/debug-session.zip

// 5. Анализ в Trace Viewer:
// - Перемотка таймлайна для просмотра каждого шага
// - Клик по элементу на скриншоте для инспекции селекторов
// - Проверка Network Tab для анализа API-запросов
// - Просмотр Console Tab для выявления JS-ошибок
// - Анализ Actionability Timeline для понимания, почему действие не выполнилось
```

---

## Best Practices

- Включайте трассировку только при падениях через условную логику в `@AfterEach`, это экономит дисковое пространство и ускоряет успешные прогоны
- Настраивайте `setRecordVideoSize()` для оптимизации размера видео, разрешение 1280x720 достаточно для большинства сценариев
- Используйте `generateArtifactName()` с timestamp и testName для уникальности файлов, избегайте перезаписи при параллельных запусках
- Интегрируйте артефакты с CI-artifact storage (GitHub Actions, Jenkins, S3), чтобы команда могла скачать их после прогона
- Применяйте ротацию артефактов через `cleanOldArtifacts()`, удаляя файлы старше 7 дней для предотвращения переполнения диска
- Используйте `playwright show-trace` для анализа падений, это быстрее и информативнее, чем просмотр статичных скриншотов
- Не записывайте трейсы для каждого теста без условия, это генерирует гигабайты данных и замедляет CI/CD
- Не храните артефакты в `src/test/resources`, выносите их в `target/` или временные директории для исключения из Git
- Не игнорируйте `page.video().saveAs()`, видео доступно только после закрытия страницы, вызывайте `page.close()` перед сохранением
- Не хардкодьте пути к артефактам в коде, используйте конфигурационные файлы или переменные окружения для гибкости
- Не удаляйте артефакты сразу после прогона, оставляйте их минимум на 7 дней для анализа регрессий и исторических данных
- Не смешивайте артефакты разных запусков в одной директории, используйте timestamp или build number для изоляции

---