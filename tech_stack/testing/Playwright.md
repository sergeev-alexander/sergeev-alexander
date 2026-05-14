**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Playwright

## Содержание

[Введение и архитектура Playwright](#1-введение-и-архитектура-playwright)
[Подключение и конфигурация окружения](#2-подключение-и-конфигурация-окружения)
[Ядро API: Браузер, Контекст, Страница](#3-ядро-api-браузер-контекст-страница)
[Селекторы и Locator API](#4-селекторы-и-locator-api)
[Взаимодействие с элементами и Auto-waiting](#5-взаимодействие-с-элементами-и-auto-waiting)
[Интеграция с JUnit 5 и TestNG](#6-интеграция-с-junit-5-и-testng)
[API-тестирование через APIRequestContext](#7-api-тестирование-через-apirequestcontext)
[Архитектурные паттерны и организация кода](#8-архитектурные-паттерны-и-организация-кода)
[Продвинутые сценарии и изоляция тестов](#9-продвинутые-сценарии-и-изоляция-тестов)
[Отчётность, трейсы и артефакты](#10-отчётность-трейсы-и-артефакты)
[Инфраструктура: Docker, CI/CD, Параллельный запуск](#11-инфраструктура-docker-cicd-параллельный-запуск)
[Эволюция версий и миграция (1.30+ vs Legacy)](#12-эволюция-версий-и-миграция-130-vs-legacy)
[Типичные ошибки и Production-Ready практики](#13-типичные-ошибки-и-production-ready-практики)

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