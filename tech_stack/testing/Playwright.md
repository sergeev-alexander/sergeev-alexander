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