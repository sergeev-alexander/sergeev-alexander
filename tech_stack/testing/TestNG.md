План шпаргалки: TestNG для профессиональной разработки

Формат: тот же, что и для JUnit 5 — Markdown, обёртка ```text, примеры с отступами, буллиты, 7 ключевых выводов на
раздел.

---

## Обновлённая структура

### Введение

- Что такое TestNG: философия, основные возможности
- Когда выбирать TestNG: data-driven, сложные зависимости, параллелизм, группы
- Подключение зависимостей (Maven/Gradle)
- Базовая настройка проекта

### Раздел 1: Сравнение TestNG и JUnit 5

- Ключевые отличия в философии и возможностях
- Таблица соответствия аннотаций
- Преимущества TestNG для интеграционных и e2e тестов
- Когда оставаться на JUnit 5

### Раздел 2: Конфигурация и запуск

- testng.xml: структура, suites, tests, параметры
- Запуск из IDE (IntelliJ, Eclipse)
- Запуск из CLI: Maven/Gradle, фильтры по группам/классам
- Конфигурационные файлы (pom.xml, build.gradle)

### Раздел 3: Аннотации и жизненный цикл тестов

- Все аннотации TestNG: @BeforeSuite, @BeforeTest, @BeforeClass, @BeforeMethod, @Test, @After*
- Порядок выполнения и уровни иерархии
- Параметры аннотаций: enabled, dependsOnMethods, groups, invocationCount, timeOut
- @TestInstance и управление жизненным циклом

### Раздел 4: Assertions и проверки

- Hard assertions: assertEquals, assertTrue, assertThrows
- Soft assertions: SoftAssert, collectAll(), assertAll()
- Custom assertions и интеграция с AssertJ
- Best Practices: когда использовать soft vs hard

### Раздел 5: Параметризация и Data Providers

- @DataProvider: методы, фабрики данных, параллельная подача
- @Parameters: передача из testng.xml
- @Factory: динамическое создание экземпляров тестов
- Примеры: CSV, JSON, БД как источники данных

### Раздел 6: Группы, зависимости и порядок выполнения

- @Test(groups = {...}): маркировка и фильтрация
- dependsOnMethods / dependsOnGroups: управление порядком
- alwaysRun: запуск даже при падении зависимостей
- Практические сценарии: smoke, regression, e2e

### Раздел 7: Параллельное выполнение

- Уровни параллелизма: methods, classes, tests, instances
- Настройка в testng.xml и через аннотации
- Thread-safety: @BeforeMethod(alwaysRun = true), ThreadLocal
- Ограничения и рекомендации

### Раздел 8: Listeners и отчётность

- Встроенные listeners: IInvokedMethodListener, ITestListener
- Кастомные listeners: логирование, скриншоты, алерты
- Отчёты: emailable-report.html, index.html, интеграция с Allure
- Интеграция с CI/CD: артефакты, JUnit-формат для Jenkins/GitHub

### Раздел 9: Интеграция с экосистемой

- TestNG + Mockito: mocking в тестах
- TestNG + Spring: @SpringBootTest, @MockBean, контекст
- TestNG + Selenium: Page Object, параллельные браузеры
- TestNG + Database: @BeforeSuite для миграций, in-memory БД

### Раздел 10: Best Practices

- Принципы FIRST/CLEAN в контексте TestNG
- Что тестировать: юниты, интеграции, e2e
- Избегание антипаттернов: избыточные зависимости, хрупкие группы
- Рефакторинг тестов: вынос данных, хелперы, фабрики
- Покрытие кода: интерпретация, инструменты (JaCoCo)

---

# TestNG

## Содержание

1. [Введение](#1-введение)
2. [Сравнение TestNG и JUnit 5](#2-сравнение-testng-и-junit-5)
3. [Конфигурация и запуск](#3-конфигурация-и-запуск)
4. [Аннотации и жизненный цикл тестов](#4-аннотации-и-жизненный-цикл-тестов)
5. [Assertions и проверки](#5-assertions-и-проверки)
6. [Параметризация и Data Providers](#6-параметризация-и-data-providers)
7. [Группы, зависимости и порядок выполнения](#7-группы-зависимости-и-порядок-выполнения)
8. [Параллельное выполнение](#8-параллельное-выполнение)
9. [Listeners и отчётность](#9-listeners-и-отчётность)
10. [Интеграция с экосистемой](#10-интеграция-с-экосистемой)
11. [Best Practices](#11-best-practices)

---

# 1. Введение

> TestNG — фреймворк для тестирования в Java, вдохновлённый JUnit и NUnit.
>
> Предназначен для покрытия всех уровней тестирования: unit, integration, functional, e2e.

---

## Основные возможности TestNG

- **Аннотации** — Гибкая система аннотаций с разными уровнями жизненного цикла
- **Группировка тестов** — `@Test(groups = {...})` для категоризации
- **Зависимости** — `dependsOnMethods`, `dependsOnGroups` для управления порядком
- **Параметризация** — `@DataProvider` для data-driven тестирования
- **Параллелизм** — Встроенная поддержка параллельного выполнения
- **Soft Assertions** — Продолжение теста после неудачной проверки
- **Listeners** — Расширение функциональности через слушатели событий
- **Отчётность** — Встроенные HTML и XML отчёты

---

## Когда выбирать TestNG

- **Data-driven тесты** — Много тестовых данных из внешних источников
- **Сложные зависимости** — Тесты зависят от результатов других тестов
- **Группировка** — Нужно запускать smoke/regression/e2e отдельно
- **Параллельное выполнение** — Требуется гибкая настройка параллелизма
- **Интеграционные тесты** — Несколько уровней setup/teardown
- **Наследование конфигурации** — Базовые классы с общими настройками

---

## Подключение зависимостей

### Maven (pom.xml):

```xml

<dependencies>
    <!-- TestNG ядро -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.10.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Интеграция с Maven Surefire -->
    <dependency>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
    </dependency>
</dependencies>
```

### Gradle (build.gradle):

```text
    dependencies {
        testImplementation 'org.testng:testng:7.10.0'
    }
    
    test {
        useTestNG()
        // Опции для отчётов
        testLogging {
            events 'passed', 'skipped', 'failed'
            exceptionFormat 'full'
        }
    }
```

### Конфигурация Surefire для TestNG:

```xml

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>
        <properties>
            <property>
                <name>usedefaultlisteners</name>
                <value>false</value>
            </property>
        </properties>
    </configuration>
</plugin>
```

---

## Базовая структура проекта

```text
    src/
    ├── main/
    │   └── java/
    │       └── com/example/
    │           └── service/
    │               └── UserService.java
    └── test/
        ├── java/
        │   └── com/example/
        │       └── service/
        │           └── UserServiceTest.java
        └── resources/
            └── testng.xml
```

---

## Базовый testng.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">  <!-- DOCTYPE НЕ ОБЯЗАТЕЛЕН для корректной работы TestNG -->
<suite name="MySuite" parallel="methods" thread-count="4">

    <test name="UnitTests">
        <classes>
            <class name="com.example.service.UserServiceTest"/>
            <class name="com.example.repository.UserRepositoryTest"/>
        </classes>
    </test>

    <test name="IntegrationTests">
        <groups>
            <run>
                <include name="integration"/>
            </run>
        </groups>
        <classes>
            <class name="com.example.integration.ApiTest"/>
        </classes>
    </test>

</suite>
```

---

## TestNG в экосистеме Java

```text
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   IntelliJ  │     │    Maven    │     │   Jenkins   │
    │   Eclipse   │---->│    Gradle   │---->│   GitLab    │
    │   VS Code   │     │             │     │   GitHub    │
    └─────────────┘     └─────────────┘     └─────────────┘
         IDE              Build Tools            CI/CD
```

---

## Преимущества TestNG

- **Гибкая конфигурация** — `testng.xml` для централизованного управления
- **Группировка** — Запуск тестов по категориям без изменения кода
- **Зависимости** — Явное указание порядка выполнения тестов
- **Параллелизм** — Настройка на уровне `suite`, `test`, `class`, `method`
- **Soft Assert** — Сбор всех ошибок теста вместо остановки на первой
- **Data Providers** — Мощная параметризация с внешними источниками
- **Listeners** — Хуки для логирования, скриншотов, алертов

---

## Best Practices

✅ Делайте:

- Используйте `testng.xml` для управления тестовыми сьютами
- Размечайте тесты группами (`smoke`, `regression`, `integration`)
- Применяйте `SoftAssert` для e2e тестов с множественными проверками
- Настраивайте параллелизм для ускорения прогона
- Документируйте зависимости между тестами

❌ Не делайте:

- Не создавайте циклические зависимости между тестами
- Не полагайтесь на порядок выполнения без `dependsOnMethods`
- Не используйте параллелизм без проверки thread-safety
- Не игнорируйте отчёты TestNG — они содержат ценную информацию
- Не смешивайте JUnit и TestNG аннотации в одном классе

---

## Ключевые выводы

1. **TestNG для сложных сценариев** — Идеален для integration и e2e тестов с зависимостями
2. **Группировка упрощает запуск** — Теги smoke/regression позволяют выбирать тесты без кода
3. **testng.xml — центр конфигурации** — Все настройки в одном файле, легко версионировать
4. **Параллелизм встроен** — Не нужны дополнительные плагины для многопоточного запуска
5. **Soft Assert для полных отчётов** — Все ошибки теста видны сразу, а не по одной
6. **Data Providers мощнее JUnit** — Гибкая подача данных из любых источников
7. **Listeners расширяют функционал** — Логирование, скриншоты, уведомления без дублирования кода

# 2. Сравнение TestNG и JUnit 5

> TestNG и JUnit 5 — два популярных фреймворка для тестирования в Java.
---

## Философия и подход

### JUnit 5

- Фокус на unit-тестировании
- Минималистичный дизайн
- Расширяемость через Extensions
- Тесная интеграция со Spring Boot
- Стандарт де-факто для Java-проектов

### TestNG

- Фокус на всех уровнях тестирования (unit → e2e)
- Богатая конфигурация из коробки
- Группировка и зависимости тестов
- Мощная параметризация
- Популярен в Selenium и интеграционных тестах

## Таблица соответствия аннотаций

| Задача                     | JUnit 5                                             | TestNG                                       |
|:---------------------------|-----------------------------------------------------|----------------------------------------------|
| Тест-метод                 | @Test                                               | @Test                                        |
| Перед каждым тестом        | @BeforeEach                                         | @BeforeMethod                                |
| После каждого теста        | @AfterEach                                          | @AfterMethod                                 |
| Перед всеми тестами класса | @BeforeAll                                          | @BeforeClass                                 |
| После всех тестов класса   | @AfterAll                                           | @AfterClass                                  |
| Перед набором тестов       | —                                                   | @BeforeSuite / @BeforeTest                   |
| После набора тестов        | —                                                   | @AfterSuite / @AfterTest                     |
| Отключение теста           | @Disabled                                           | @Test(enabled = false)                       |
| Параметризация             | @ParameterizedTest + @ValueSource, @CsvSource и др. | @Test(dataProvider = "name") + @DataProvider |
| Таймаут                    | @Test(timeout = ms)                                 | @Test(timeOut = ms)                          |
| Группировка                | @Tag                                                | @Test(groups = {...})                        |
| Зависимости                | @Order / сторонние расширения                       | dependsOnMethods / dependsOnGroups           |
| Расширения                 | @ExtendWith                                         | Listeners / IAnnotationTransformer           |

---

## Ключевые отличия в возможностях

### Группировка тестов

- JUnit 5: `@Tag` с фильтрацией через `includeTags` / `excludeTags`
- TestNG: `@Test(groups = {...})` с запуском через `testng.xml` или `-Dgroups`

JUnit 5: теги

```java

@Tag("smoke")
@Test
void loginTest() {
}
```

Запуск: `mvn test -Dgroups=smoke`

TestNG: группы

```java

@Test(groups = {"smoke", "auth"})
void loginTest() {
}
```

Запуск: `mvn test -Dgroups=smoke`

### Зависимости между тестами

- JUnit 5: нет встроенной поддержки (тесты должны быть независимы)
- TestNG: `dependsOnMethods`, `dependsOnGroups` для явного порядка

TestNG: зависимости

```java

@Test
public void createDatabase() {
}

@Test(dependsOnMethods = {"createDatabase"})
public void insertData() {
}

@Test(dependsOnMethods = {"insertData"}, alwaysRun = true)
public void cleanup() {
} // запустится даже при падении insertData
```

### Параметризация

- JUnit 5: `@ParameterizedTest` с источниками (`@ValueSource`, `@CsvSource`, `@MethodSource`)
- TestNG: `@DataProvider` — метод возвращает `Object[][]` или `Iterator<Object[]>`

```java
// JUnit 5: параметризация
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void testNumbers(int value) {
}

// TestNG: DataProvider
@DataProvider(name = "numbers")
Object[][] provideNumbers() {
    return new Object[][]{{1}, {2}, {3}};
}

@Test(dataProvider = "numbers")
void testNumbers(int value) {
}
```

---

## Параллельное выполнение

### JUnit 5

- Через `junit-platform.properties`
- Настройки: `parallel`, `parallel.strategy`, `parallel.config`
- Требует явной конфигурации

### TestNG

- Встроенная поддержка в `testng.xml`
- Уровни: `suite`, `test`, `class`, `method`, `instances`
- Простая настройка через атрибуты

Параллелизм в testng.xml:

```xml

<suite name="ParallelSuite" parallel="methods" thread-count="4">
    <test name="ConcurrentTests">
        <classes>
            <class name="com.example.TestClass"/>
        </classes>
    </test>
</suite>
```

JUnit 5: через конфигурацию `junit-platform.properties`

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
```

---

## Assertions

### JUnit 5

- `org.junit.jupiter.api.Assertions`
- `assertAll` для группировки проверок
- `assertThrows` возвращает исключение

### TestNG

- `org.testng.Assert` (hard) + `SoftAssert` (soft)
- `SoftAssert` продолжает тест после неудачи
- `assertAll()` в конце для сбора всех ошибок

JUnit 5: множественные проверки

```java

@Test
void testUser() {
    assertAll("user",
            () -> assertEquals("John", user.getName()),
            () -> assertEquals("john@test.com", user.getEmail())
    );
}

// TestNG: SoftAssert
@Test
void testUser() {
    SoftAssert soft = new SoftAssert();
    soft.assertEquals(user.getName(), "John");
    soft.assertEquals(user.getEmail(), "john@test.com");
    soft.assertAll(); // собирает все ошибки
}
```

---

## Отчётность

### JUnit 5

- Отчёты через Maven Surefire/Failsafe
- XML для CI, HTML через плагины
- Интеграция с Allure через адаптер

### TestNG

- Встроенные отчёты: `index.html`, `emailable-report.html`
- XML в `test-output/`
- Интеграция с Allure через `testng.xml`

---

## Интеграция со Spring

### JUnit 5

- `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`
- `@MockBean` для моков в контексте
- Нативная поддержка через `spring-boot-starter-test`

### TestNG

- Те же аннотации Spring работают
- Требуется `spring-test` dependency
- Менее распространён в Spring-сообществе

---

## Когда выбирать TestNG

✅ TestNG предпочтительнее:

- Сложные интеграционные и e2e тесты
- Нужны зависимости между тестами
- Требуется гибкая группировка (smoke/regression/nightly)
- Data-driven тесты с внешними источниками
- Параллельное выполнение с тонкой настройкой
- Selenium/WebDriver тестирование
- Наследование конфигурации тестов

✅ JUnit 5 предпочтительнее:

- Чистые unit-тесты
- Spring Boot проекты (лучшая интеграция)
- Стандарт корпоративной Java-разработки
- Минимальная конфигурация
- Меньше зависимостей между тестами

---

## Best Practices

✅ Делайте:

- Выбирайте фреймворк под задачи проекта, а не по привычке
- Используйте TestNG для e2e и Selenium, а JUnit 5 для unit
- Не смешивайте аннотации JUnit и TestNG в одном классе
- Документируйте выбор фреймворка в README
- Мигрируйте постепенно, если меняете фреймворк

❌ Не делайте:

- Не создавайте зависимости между unit-тестами (нарушает FIRST)
- Не используйте TestNG только ради одной фичи (например, групп)
- Не игнорируйте отчёты TestNG — они содержат ценную информацию
- Не полагайтесь на порядок выполнения без dependsOnMethods
- Не смешивайте JUnit и TestNG зависимости без необходимости

---

# 3. Конфигурация и запуск

> Правильная настройка TestNG критична для стабильного запуска тестов.
>
> `testng.xml` — центральный файл конфигурации TestNG. Определяет suite, тесты, классы, группы, параметры и параллелизм.

### Базовая структура `testng.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">  <!-- DOCTYPE не обязателен -->

<suite name="MySuite" parallel="methods" thread-count="4">

    <!-- Параметры для всех тестов -->
    <parameter name="environment" value="staging"/>
    <parameter name="browser" value="chrome"/>

    <!-- Группа тестов -->
    <test name="UnitTests">
        <classes>
            <class name="com.example.service.UserServiceTest"/>
            <class name="com.example.repository.OrderRepositoryTest"/>
        </classes>
    </test>

    <!-- Фильтрация по группам -->
    <test name="IntegrationTests">
        <groups>
            <run>
                <include name="integration"/>
                <exclude name="slow"/>
            </run>
        </groups>
        <classes>
            <class name="com.example.integration.ApiTest"/>
        </classes>
    </test>

</suite>
```

---

- `<suite>` - корневой элемент, определяющий набор тестов; может содержать один или несколько тестов
    - `name` - имя набора тестов (обязательный)
    - `parallel` - режим параллельного выполнения (`false`, `methods`, `tests`, `classes`, `instances`)
    - `thread-count` - количество потоков для параллельного выполнения (по умолчанию 5)
    - `configfailurepolicy` - политика при падении конфигурации (`skip`, `continue`)
    - `data-provider-thread-count` - количество потоков для data-provider (по умолчанию 10)
    - `group-by-instances` - группировать методы по экземплярам (`true`/`false`)
    - `preserve-order` - сохранять порядок выполнения тестов (`true`/`false`)
    - `allow-return-values` - разрешить возвращаемые значения для методов теста (`true`/`false`)
    - `skipfailedinvocationcounts` - пропускать подсчет неудачных вызовов (`true`/`false`)
    - `time-out` - таймаут по умолчанию для всех тестов (в миллисекундах)
    - `object-factory` - класс фабрики объектов для создания экземпляров тестов
    - `listeners` - список слушателей для набора тестов (устаревший, лучше использовать `<listeners>`)

- `<test>` - группа тестов внутри suite, определяет логический блок тестирования
    - `name` - имя теста (обязательный)
    - `enabled` - включен ли тест (`true`/`false`, по умолчанию `true`)
    - `parallel` - режим параллельного выполнения для этого теста (переопределяет suite)
    - `thread-count` - количество потоков для этого теста (переопределяет suite)
    - `preserve-order` - сохранять порядок выполнения классов в тесте (`true`/`false`)
    - `group-by-instances` - группировать методы по экземплярам для этого теста
    - `skipfailedinvocationcounts` - пропускать подсчет неудачных вызовов для этого теста
    - `time-out` - таймаут для этого теста
    - `allow-return-values` - разрешить возвращаемые значения для методов теста
    - `verbose` - уровень детализации логирования

- `<parameter>` - определение параметра для передачи в тесты
    - `name` - имя параметра (обязательный)
    - `value` - значение параметра (обязательный)

- `<groups>` - контейнер для определения групп и их состава
    - `<define>` - определение группы с включенными/исключенными методами
        - `name` - имя определяемой группы (обязательный)
        - `<include>` - метод для включения в группу
            - `name` - имя метода (обязательный)
        - `<exclude>` - метод для исключения из группы
            - `name` - имя метода (обязательный)
    - `<run>` - указание, какие группы выполнять
        - `<include>` - группа для включения
            - `name` - имя группы (обязательный)
        - `<exclude>` - группа для исключения
            - `name` - имя группы (обязательный)

- `<classes>` - контейнер для списка классов тестов
    - `<class>` - отдельный класс теста
        - `name` - полное имя класса (обязательный)
        - `class-name` - альтернативное имя класса
        - `<methods>` - контейнер для фильтрации методов внутри класса
            - `<include>` - метод для включения
                - `name` - имя метода (обязательный)
                - `description` - описание метода
            - `<exclude>` - метод для исключения
                - `name` - имя метода (обязательный)
            - `<parameter>` - параметр, специфичный для метода
                - `name` - имя параметра (обязательный)
                - `value` - значение параметра (обязательный)

- `<packages>` - контейнер для списка пакетов для сканирования тестов
    - `<package>` - пакет для сканирования
        - `name` - имя пакета (обязательный)
        - `<include>` - включить методы из пакета (внутри `<methods>`)
        - `<exclude>` - исключить методы из пакета (внутри `<methods>`)

- `<listeners>` - контейнер для списка слушателей
    - `<listener>` - слушатель событий TestNG
        - `class-name` - полное имя класса слушателя (обязательный)

- `<method-selectors>` - контейнер для селекторов методов (продвинутая фильтрация)
    - `<method-selector>` - селектор для выбора методов
        - `<script>` - скрипт для селекции (BeanShell, Groovy)
            - `language` - язык скрипта (по умолчанию `beanshell`)
        - `<selector-class>` - класс-селектор
            - `name` - имя класса селектора

- `<dependencies>` - контейнер для определения зависимостей между группами
    - `<group>` - определение зависимости группы
        - `name` - имя группы (обязательный)
        - `depends-on` - список групп, от которых зависит

- `<reporter>` - настройки репортера
    - `<output>` - путь для выходных отчетов
        - `path` - путь к директории
    - `<config>` - конфигурация репортера
        - `<parameter>` - параметр конфигурации
            - `name` - имя параметра (обязательный)
            - `value` - значение параметра (обязательный)

- `<bean-shell>` - встроенный скрипт BeanShell для динамической конфигурации
    - `script` - текст скрипта

- `<xml>` - элемент для включения внешних testng.xml файлов
    - `file` - путь к внешнему файлу

- `<test-methods>` - контейнер для переопределения порядка методов
    - `<method>` - конкретный метод
        - `<selector>` - селектор для метода
        - `<parameter>` - параметр метода
            - `name` - имя параметра (обязательный)
            - `value` - значение параметра (обязательный)

---

### Уровни иерархии

- `suite` — верхний уровень, один файл `testng.xml`
- `test` — группа тестов внутри suite (может быть несколько)
- `classes` — список классов для запуска
- `methods` — отдельные методы (через `<methods>` внутри `<class>`)

Пример: запуск конкретных методов

```xml

<test name="SmokeTests">
    <classes>
        <class name="com.example.LoginTest">
            <methods>
                <include name="validLogin"/>
                <include name="invalidLogin"/>
            </methods>
        </class>
    </classes>
</test>
```

---

## Подключение зависимостей

### Maven (pom.xml)

```xml

<dependencies>

    <!-- TestNG ядро -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.10.0</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ для дополнительных assertions -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.0</version>
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
            <suiteXmlFiles>
                <suiteXmlFile>testng.xml</suiteXmlFile>
            </suiteXmlFiles>
        </configuration>
    </plugin>
</plugins>
</build>
```

### Gradle (build.gradle)

```groovy
dependencies {
    testImplementation 'org.testng:testng:7.10.0'
    testImplementation 'org.assertj:assertj-core:3.25.0'
}

test {
    useTestNG()
    // Сьюты
    suites 'testng.xml'
    // Группы
    useDefaultGroups = false
    includeGroups 'smoke', 'integration'
    excludeGroups 'slow'
    // Отчёты
    testLogging {
        events 'passed', 'skipped', 'failed'
        exceptionFormat 'full'
    }
}
```

---

## Запуск из командной строки

### Maven

```bash
# Запустить все тесты через testng.xml

mvn test

# Запустить конкретную группу

mvn test -Dgroups=smoke

# Запустить конкретный класс

mvn test -Dtest=com.example.LoginTest

# Пропустить тесты

mvn package -DskipTests

# Несколько групп

mvn test -Dgroups="smoke,regression"
```

### Gradle

```bash
# Все тесты

./gradlew test

# Конкретная группа

./gradlew test -Dgroups=smoke

# Конкретный класс

./gradlew test --tests "com.example.LoginTest"

# Исключить группу

./gradlew test -DexcludedGroups=slow
```

---

## Интеграция со Spring

TestNG работает с Spring Boot, но требует дополнительной настройки для управления контекстом.

### Зависимости

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>

    <!-- Исключаем JUnit, оставляем только Spring -->
    <exclusions>
        <exclusion>
            <groupId>org.junit.vintage</groupId>
            <artifactId>junit-vintage-engine</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
<groupId>org.testng</groupId>
<artifactId>testng</artifactId>
<version>7.10.0</version>
<scope>test</scope>
</dependency>
```

### Кастомный listener для Spring контекста

#### SpringTestNGListener.java

```java
public class SpringTestNGListener implements IBeforeSuiteListener, IAfterSuiteListener {

    private static ConfigurableApplicationContext context;

    @Override
    public void onBeforeSuite(ISuite suite) {
        context = SpringApplication.run(Application.class);
    }

    @Override
    public void onAfterSuite(ISuite suite) {
        if (context != null) {
            context.close();
        }
    }

    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }

}
```

### Пример теста со Spring

```java

@SpringBootTest
@Listeners(SpringTestNGListener.class)
public class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test(groups = {"integration"})
    public void createUser_shouldPersistToDatabase() {
        User user = userService.create("John", "john@test.com");

        assertNotNull(user.getId());
        assertEquals(user.getName(), "John");

        // Проверка в БД
        User fromDb = userRepository.findById(user.getId()).orElse(null);
        assertNotNull(fromDb);
    }

    @Test(groups = {"integration"})
    public void getUser_shouldReturnExisting() {
        User existing = SpringTestNGListener.getBean(UserRepository.class)
                .findById(1L).orElseThrow();

        User result = userService.getById(existing.getId());
        assertEquals(result.getEmail(), existing.getEmail());
    }

}
```

### @MockBean с TestNG

> TestNG не поддерживает @MockBean нативно — создаём вручную

```java

@SpringBootTest                         // Загружает полный Spring контекст, как в реальном приложении
@Listeners(SpringTestNGListener.class)  // Подключает кастомный listener для управления Spring контекстом
public class ServiceWithMockTest {

    // СЕРВИС, КОТОРЫЙ ТЕСТИРУЕМ
    // Создаем вручную, а не через @Autowired, чтобы подставить mock-зависимость
    private UserService userService;

    // MOCK ЗАВИСИМОСТИ
    // Создаем mock-объект для репозитория, который будет использоваться вместо реального
    // @Mock от Mockito создает прокси-объект, который ничего не делает по умолчанию
    @Mock
    private UserRepository userRepository;

    // НАСТРОЙКА ПЕРЕД ВСЕМИ ТЕСТАМИ В КЛАССЕ
    // Выполняется один раз перед запуском всех тестов в этом классе
    @BeforeClass
    public void setUp() {
        // 1. Инициализируем все mock-объекты, помеченные @Mock
        //    Это создает реальные mock-объекты и связывает их с полями
        MockitoAnnotations.openMocks(this);

        // 2. Ручное создание сервиса с инъекцией mock-зависимости
        //    В отличие от @Autowired, мы сами контролируем создание
        //    Передаем mock-репозиторий в конструктор сервиса
        userService = new UserService(userRepository);

        // Теперь userService использует не реальную БД, а mock-объект
        // Все вызовы userRepository внутри userService будут идти в mock
    }

    @Test
    public void testWithMock() {
        // НАСТРОЙКА ПОВЕДЕНИЯ MOCK
        // Говорим mock-объекту: "когда у тебя вызовут findById(1L) - верни Optional с пользователем John"
        when(userRepository.findById(1L))
                .thenReturn(Optional.of(new User("John")));

        // ВЫПОЛНЕНИЕ ТЕСТИРУЕМОГО МЕТОДА
        // Вызываем метод сервиса, который внутри себя вызовет userRepository.findById(1L)
        // Благодаря нашей настройке выше, вернется подготовленный пользователь
        User result = userService.getById(1L);

        assertEquals(result.getName(), "John");

        // ВАЖНО: Никаких реальных обращений к базе данных не произошло!
        // Тест выполняется быстро и изолированно от внешних систем
    }
}
```

---

## Best Practices

✅ Делайте:

- Храните testng.xml в корне проекта или src/test/resources
- Используйте группы для разделения smoke/regression/integration
- Настраивайте parallel для ускорения прогона в CI
- Закрывайте Spring контекст в @AfterSuite
- Документируйте параметры в testng.xml

❌ Не делайте:

- Не дублируйте testng.xml для разных окружений (используйте параметры)
- Не запускайте все тесты локально при каждой правке
- Не забывайте про <exclusions> для JUnit в spring-boot-starter-test
- Не создавайте несколько Spring контекстов в одном suite
- Не игнорируйте failed tests в CI — настройте failOnFailure

---

## Ключевые выводы

1. **testng.xml — центр конфигурации** — Все настройки в одном файле, легко версионировать
2. **Группы упрощают фильтрацию** — Запускайте smoke/regression без изменения кода
3. **Maven Surefire требует настройки** — Укажите suiteXmlFiles для запуска TestNG
4. **Spring требует кастомного listener** — Для управления жизненным циклом контекста
5. **Исключайте JUnit из spring-boot-starter-test** — Чтобы избежать конфликтов зависимостей
6. **Параметры для окружений** — environment, browser, url через <parameter>

# 4. Аннотации и жизненный цикл тестов

> TestNG предоставляет гибкую систему аннотаций с разными уровнями жизненного цикла.
>
> TestNG имеет 11 основных аннотаций для управления жизненным циклом тестов.

### Аннотации уровня Suite

- `@BeforeSuite` — выполняется один раз перед всем набором тестов
- `@AfterSuite` — выполняется один раз после всего набора тестов

Пример: инициализация БД для всех тестов

```java

@BeforeSuite
public void setUpDatabase() {
    // Подключение к тестовой БД
    // Миграции схемы
}


@AfterSuite
public void tearDownDatabase() {
    // Закрытие соединений
    // Очистка данных
}
```

### Аннотации уровня Test

- `@BeforeTest` — перед каждым `<test>` в testng.xml
- `@AfterTest` — после каждого `<test>` в testng.xml

Пример: настройка для группы тестов

```java

@BeforeTest
public void setUpTestGroup() {
    // Инициализация для конкретной группы тестов
}
```

### Аннотации уровня Class

- `@BeforeClass` — один раз перед первым тестом в классе
- `@AfterClass` — один раз после всех тестов в классе

Пример: создание сервиса для всех тестов класса

```java

@BeforeClass
public void setUpClass() {
    userService = new UserService();
}

@AfterClass
public void tearDownClass() {
    userService = null;
}
```

### Аннотации уровня Method

- `@BeforeMethod` — перед каждым `@Test` методом
- `@AfterMethod` — после каждого `@Test` метода

Пример: сброс состояния между тестами

```java

@BeforeMethod
public void setUpMethod() {
    // Очистка кэша
    // Сброс моков
}

@AfterMethod
public void tearDownMethod() {
    // Закрытие ресурсов
    // Логирование результата
}
```

---

## Порядок выполнения аннотаций

Порядок вызова аннотаций при запуске тестов:

1. `@BeforeSuite` (1 раз для всего файла)
2. `@BeforeTest` (1 раз для каждого `<test>`)
3. `@BeforeClass` (1 раз для каждого класса)
4. `@BeforeMethod` (перед каждым `@Test`)
5. `@Test` (сам тест)
6. `@AfterMethod` (после каждого `@Test`)
7. `@AfterClass` (1 раз для каждого класса)
8. `@AfterTest` (1 раз для каждого `<test>`)
9. `@AfterSuite` (1 раз для всего файла)

---

## Параметры аннотации @Test

`@Test` поддерживает множество параметров для гибкой настройки поведения.

| Параметр                          | Тип                             | Описание                                                                     |
|-----------------------------------|---------------------------------|------------------------------------------------------------------------------|
| `enabled`                         | boolean                         | Включение/отключение теста                                                   |
| `priority`                        | int                             | Приоритет выполнения (меньше = раньше); можно указать на классе, и на методе |
| `description`                     | String                          | Описание теста                                                               |
| `groups`                          | String[]                        | Группы, которым принадлежит тест                                             |
| `dependsOnMethods`                | String[]                        | Зависимость от других методов                                                |
| `dependsOnGroups`                 | String[]                        | Зависимость от групп                                                         |
| `alwaysRun`                       | boolean                         | Запускать даже при падении зависимостей                                      |
| `timeOut`                         | long                            | Таймаут выполнения в миллисекундах                                           |
| `invocationCount`                 | int                             | Количество запусков теста                                                    |
| `invocationTimeOut`               | long                            | Таймаут для всех запусков                                                    |
| `threadPoolSize`                  | int                             | Размер пула потоков для параллельного выполнения                             |
| `successPercentage`               | int                             | Процент успешных запусков                                                    |
| `singleThreaded`                  | boolean                         | Однопоточное выполнение методов класса                                       |
| `sequential`                      | boolean                         | Последовательное выполнение (устаревший, используйте singleThreaded)         |
| `expectedExceptions`              | Class<? extends Throwable>[]    | Ожидаемые исключения                                                         |
| `expectedExceptionsMessageRegExp` | String                          | Регулярное выражение для сообщения исключения                                |
| `dataProvider`                    | String                          | Имя dataProvider'а                                                           |
| `dataProviderClass`               | Class<?>                        | Класс, содержащий dataProvider                                               |
| `retryAnalyzer`                   | Class<? extends IRetryAnalyzer> | Анализатор повторов                                                          |

### `enabled` — включение/отключение теста

```java

@Test(enabled = false)
public void disabledTest() {
    // Тест будет пропущен
}
```

### `timeOut` — таймаут выполнения в миллисекундах

```java

@Test(timeOut = 5000)
public void timeoutTest() {
    // Должен завершиться за 5 секунд
    // Иначе TestException
}
```

### `invocationCount` — количество запусков теста

```java

@Test(invocationCount = 10)
public void repeatedTest() {
    // Запустится 10 раз
}
```

### `invocationTimeOut` — таймаут для всех запусков

```java

@Test(invocationCount = 100, invocationTimeOut = 10000)
public void performanceTest() {
    // 100 запусков за 10 секунд
}
```

### `groups` — маркировка тестов

```java

@Test(groups = {"smoke", "auth"})
public void loginTest() {
    // Принадлежит двум группам
}
```

### `dependsOnMethods` — зависимость от других методов

```java

@Test
public void createData() {
}

@Test(dependsOnMethods = {"createData"})
public void processData() {
    // Запустится только если createData успешен
}
```

### `dependsOnGroups` — зависимость от групп

```java

@Test(dependsOnGroups = {"smoke"})
public void regressionTest() {
    // Запустится только если все smoke тесты прошли
}
```

### `alwaysRun` — запуск даже при падении зависимостей

```java

@Test(dependsOnMethods = {"createData"}, alwaysRun = true)
public void cleanup() {
    // Запустится даже если createData упал
}
```

### `description` — описание теста

```java

@Test(description = "Проверка входа с валидными данными")
public void loginTest() {
}
```

### `priority` — приоритет выполнения (меньше = раньше); можно указать как на классе, так и на методе

```java

@Test(priority = 1)
public class UserServiceTest {

    // Метод с более высоким приоритетом выполнится раньше других методов класса
    @Test(priority = 1)
    public void createUser() {
        System.out.println("1.1 Создание пользователя");
    }

    // Метод без priority выполнится после методов с явным priority
    @Test
    public void setupTestData() {
        System.out.println("1.2 Подготовка данных");
    }

    // Метод с более низким приоритетом
    @Test(priority = 2)
    public void validateUser() {
        System.out.println("1.3 Валидация пользователя");
    }
}

@Test(priority = 2)
public class OrderServiceTest {
    @Test
    public void createOrder() {
        System.out.println("2.1 Создание заказа");
    }
}
```

### `expectedExceptions` — ожидаемое исключение

`expectedExceptions` нужен, чтобы:

- Проверить, что тест выбрасывает ожидаемое исключение (иначе тест упадёт)
- Избежать try-catch блоков в тестовом коде
- Убедиться, что ошибка возникает в нужном месте, а не где-то ещё
- Проверить текст ошибки через expectedExceptionsMessageRegExp

```java

@Test(expectedExceptions = RuntimeException.class)
public void testException() {
    throw new RuntimeException("Ожидаемое исключение");
}

// Проверка сообщения исключения
@Test(expectedExceptions = IllegalArgumentException.class,
        expectedExceptionsMessageRegExp = "Email is invalid")
public void testExceptionWithMessage() {
    userService.createUser("", "invalid-email");
}

// Несколько ожидаемых исключений
@Test(expectedExceptions = {NullPointerException.class, IllegalArgumentException.class})
public void testMultipleExceptions() {
    // Может выбросить любое из указанных исключений
}
```

### `dataProvider` — поставщик данных

```java

@DataProvider(name = "userData")
public Object[][] provideUserData() {
    return new Object[][]{
            {"John", "john@test.com", true},
            {"Jane", "jane@test.com", true},
            {"", "invalid", false}
    };
}

@Test(dataProvider = "userData")
public void testUserCreation(String name, String email, boolean expectedValid) {
    if (expectedValid) {
        User user = userService.create(name, email);
        assertNotNull(user.getId());
    } else {
        assertThrows(ValidationException.class, () ->
                userService.create(name, email)
        );
    }
}

// С несколькими dataProvider'ами
@Test(dataProvider = "userData", dataProviderClass = ExternalDataProvider.class)
public void testWithExternalData(String name, String email) {
    // dataProvider из другого класса
}
```

### `dataProviderClass` — класс с dataProvider'ом

```java
// Внешний класс с dataProvider
public class ExternalDataProvider {
    @DataProvider(name = "externalData")
    public static Object[][] provideData() {
        return new Object[][]{
                {"data1", 123},
                {"data2", 456}
        };
    }
}

@Test(dataProvider = "externalData", dataProviderClass = ExternalDataProvider.class)
public void testWithExternalData(String name, int value) {
    System.out.println(name + ": " + value);
}
```

### `singleThreaded` — однопоточное выполнение

```java
@Test(singleThreaded = true)
public class SingleThreadedTest {
// Все методы этого класса выполняются в одном потоке
// Даже если параллельное выполнение включено глобально

    @Test
    public void test1() { }

    @Test
    public void test2() { }
}
```

### `threadPoolSize` — размер пула потоков
        
```java
// Запуск теста в нескольких потоках
// 10 запусков, максимум 5 параллельных потоков
@Test(invocationCount = 10, threadPoolSize = 5)
public void testConcurrent() {
    System.out.println("Thread: " + Thread.currentThread().getId());
}
```

### `successPercentage` — процент успешных запусков
        
```java
// Требуется 95% успешных запусков
@Test(invocationCount = 20, successPercentage = 95)
public void testWithFlakyBehavior() {
    // Тест может иногда падать, но должен проходить в 95% случаев
    if (Math.random() < 0.05) {
        fail("Редкий сбой");
    }
}
```

### `expectedExceptionsMessageRegExp` — регулярное выражение для сообщения

```java

@Test(expectedExceptions = IllegalArgumentException.class,
        expectedExceptionsMessageRegExp = ".*email.*invalid.*")
public void testInvalidEmail() {
    userService.createUser("John", "not-an-email");
}

// Более сложное регулярное выражение
@Test(expectedExceptions = ValidationException.class,
        expectedExceptionsMessageRegExp = "(Email|Password) is (invalid|required)")
public void testValidationMessage() {
    userService.validateCredentials("", "");
}
```

### `retryAnalyzer` — анализатор повторов

```java
// Создаем свой анализатор повторов
public class CustomRetryAnalyzer implements IRetryAnalyzer {
    
    private int retryCount = 0;
    private static final int MAX_RETRY = 3;

    @Override
    public boolean retry(ITestResult result) {
        if (retryCount < MAX_RETRY) {
            retryCount++;
            return true;  // Повторить тест
        }
        return false;  // Не повторять
    }

}

@Test(retryAnalyzer = CustomRetryAnalyzer.class)
public void testWithRetry() {
    // При падении будет выполнен повторно до 3 раз
    flakyOperation();
}
```

### `sequential` — последовательное выполнение

```java
@Test(sequential = true)
public class SequentialTest {
    // Методы выполняются последовательно (устаревший параметр)
    // Рекомендуется использовать singleThreaded
}
```

---

## Интеграция со Spring

TestNG требует кастомной настройки для работы с Spring контекстом.

### Listener для управления контекстом (SpringContextListener.java)

```java
public class SpringContextListener implements IBeforeSuiteListener, IAfterSuiteListener {

    private static ConfigurableApplicationContext context;

    @Override
    public void onBeforeSuite(ISuite suite) {
        context = SpringApplication.run(Application.class);
    }

    @Override
    public void onAfterSuite(ISuite suite) {
        if (context != null) {
            context.close();
        }
    }

    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }
}
```

### Тест с Spring контекстом

```java

@SpringBootTest
@Listeners(SpringContextListener.class)
public class SpringIntegrationTest {

    @Autowired
    private UserService userService;

    @BeforeClass
    public void setUp() {
        // Дополнительная настройка перед тестами класса
    }

    @BeforeMethod
    public void resetData() {
        // Очистка БД перед каждым тестом
        userService.deleteAll();
    }

    @Test(groups = {"integration"})
    public void createUser_shouldPersist() {
        User user = userService.create("John", "john@test.com");
        assertNotNull(user.getId());
    }

    @AfterMethod
    public void logResult(ITestResult result) {
        // Логирование результата каждого теста
        System.out.println("Test: " + result.getName() + " - " + result.getStatus());
    }
}
```

### Mockito + Spring + TestNG

```java

@SpringBootTest
@Listeners(SpringContextListener.class)
public class ServiceWithMockTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @BeforeClass
    public void initMocks() {
        MockitoAnnotations.openMocks(this);
    }

    @BeforeMethod
    public void resetMocks() {
        reset(userRepository); // Метод Mockito для очистки настроек и стабов
    }

    @Test
    public void testWithMock() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User("John")));

        User result = userService.getById(1L);
        assertEquals(result.getName(), "John");

        verify(userRepository).findById(1L);
    }
}
```

---

## Best Practices

✅ Делайте:

- Используйте `@BeforeMethod` для сброса состояния между тестами
- Применяйте `@BeforeClass` для тяжёлой инициализации (создание сервисов)
- Настраивайте `timeOut` для тестов с внешними вызовами
- Используйте `groups` для категоризации тестов
- Закрывайте ресурсы в `@AfterSuite` / `@AfterClass`

❌ Не делайте:

- Не создавайте зависимости между тестами без `dependsOnMethods`
- Не используйте `@BeforeSuite` для тестовых данных (используйте `@BeforeMethod`)
- Не забывайте про `alwaysRun` для критических cleanup-методов
- Не смешивайте аннотации JUnit и TestNG в одном классе
- Не полагайтесь на `priority` вместо явных зависимостей

---

## Ключевые выводы

1. **11 аннотаций для всех уровней** — От suite до method для гибкого управления
2. **Порядок выполнения фиксирован** — Suite → Test → Class → Method → Test → After
3. **`@BeforeMethod` для изоляции** — Каждый тест начинается с чистого состояния
4. **`timeOut` защищает от зависаний** — Обязательно для внешних вызовов
5. **`groups` + `dependsOnMethods` для порядка** — Явное управление последовательностью
6. **Spring требует кастомного listener** — Для управления жизненным циклом контекста
7. **`alwaysRun` для критических операций** — Cleanup должен выполняться всегда

# 5. Assertions и проверки

> **Assertions** — механизм проверки результатов тестов. TestNG предоставляет hard assertions (останавливают тест при
> неудаче)
> и soft assertions (продолжают выполнение).

---

## Hard Assertions (останавливают выполнение теста при первой неудачной проверке)

### Базовые assertions

`assertEquals` — проверка равенства
`assertNotEquals` — проверка неравенства
`assertTrue` — проверка истинности
`assertFalse` — проверка ложности
`assertNull` — проверка на null
`assertNotNull` — проверка на не-null
`assertSame` — проверка на одинаковый объект (проверяют ссылку на объект (reference equality))
`assertNotSame` — проверка на разные объекты (проверяют ссылку на объект (reference equality))

### Проверка исключений

```java

@Test(expectedExceptions = IllegalArgumentException.class)
public void testInvalidEmail() {
    userService.create("", "invalid");
}

@Test(expectedExceptions = {NullPointerException.class, IllegalArgumentException.class})
public void testMultipleExceptions() {
    methodThatThrows();
    // Тест пройдёт если брошено любое из указанных исключений
}

// Проверка сообщения исключения
@Test
public void testExceptionMessage() {
    try {
        userService.create("", "invalid");
        fail("Expected IllegalArgumentException"); // fail() — это встроенный метод TestNG (и JUnit тоже), который немедленно проваливает тест с сообщением об ошибке.
    } catch (IllegalArgumentException e) {
        assertEquals(e.getMessage(), "Email is required");
    }
}
```

### `assertThrows` (TestNG 7.5+)

```java
@Test
public void testExceptionWithAssertThrows() {
    assertThrows(IllegalArgumentException.class, () -> {
        userService.create("", "invalid");
    });
}

// С проверкой сообщения
@Test
public void testExceptionWithMessage() {
    IllegalArgumentException ex = assertThrows(IllegalArgumentException.class, () -> {
        userService.create("", "invalid");
    });
    assertEquals(ex.getMessage(), "Email is required");
}
```

---

## Soft Assertions (собирают все неудачи и отчитываются в конце теста через `assertAll()`)

### Базовое использование

```java
@Test
public void testWithSoftAssert() {
    SoftAssert soft = new SoftAssert();

    User user = userService.create("John", "john@test.com");
    
    soft.assertNotNull(user);
    soft.assertNotNull(user.getId());
    soft.assertEquals(user.getName(), "John");
    soft.assertEquals(user.getEmail(), "john@test.com");
    
    soft.assertAll(); // Обязательно в конце!

}
```

### Что происходит без assertAll()

❌ ОШИБКА: тест пройдёт даже при неудачных проверках

```java
@Test
public void testWithoutAssertAll() {
    SoftAssert soft = new SoftAssert();
    soft.assertEquals(1, 2); // Неудача не будет отчитана!
    // Нет soft.assertAll() — тест считается успешным
}
```

### SoftAssert с коллекциями

```java
@Test
public void testWithCollections() {
    SoftAssert soft = new SoftAssert();

    List<User> users = userService.findAll();
    
    soft.assertNotNull(users);
    soft.assertEquals(users.size(), 3);
    soft.assertTrue(users.stream().anyMatch(u -> u.getName().equals("John")));
    
    soft.assertAll();
}
```

---

## Интеграция с AssertJ

> AssertJ предоставляет fluent API для более читаемых assertions.

### Подключение зависимости

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.0</version>
    <scope>test</scope>
</dependency>
```

### Примеры AssertJ

```java
@Test
public void testWithAssertJ() {
    User user = userService.create("John", "john@test.com");

    assertThat(user).isNotNull();
    assertThat(user.getName()).isEqualTo("John");
    assertThat(user.getEmail()).contains("@");
    assertThat(user.getRoles()).hasSize(2).contains("USER");
}
```

### AssertJ с коллекциями

```java
@Test
public void testCollectionsWithAssertJ() {
    List<User> users = userService.findAll();

    assertThat(users)
        .isNotEmpty()
        .hasSize(3)
        .extracting("name") // метод AssertJ для извлечения данных из коллекции (здесь по имени поля)
        .containsExactlyInAnyOrder("John", "Jane", "Bob");
    
    assertThat(users)
        .extracting(User::getEmail) // метод AssertJ для извлечения данных из коллекции (здесь по method reference)
        .allMatch(email -> email.contains("@"));

    // extracting() может также извлекать поля вложенных объектов: .extracting("user.address.city")
}
```

### AssertJ для исключений

```java

@Test
public void testExceptionWithAssertJ() {
    assertThatThrownBy(() -> {
        userService.create("", "invalid");
    })
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Email is required")
            .hasMessageContaining("required");
}
```
---

## Сводная таблица: типы assertions

| Тип     | Класс                           | Поведение                            | Когда использовать               |
|---------|---------------------------------|--------------------------------------|----------------------------------|
| Hard    | org.testng.Assert               | Останавливает тест при первой ошибке | Unit-тесты, критические проверки |
| Soft    | org.testng.asserts.SoftAssert   | Продолжает тест, отчитывает в конце  | E2E, множественные проверки      |
| AssertJ | org.assertj.core.api.Assertions | Fluent API, богатые матчеры          | Читаемость, сложные проверки     |

---

## Интеграция со Spring

### Проверка Spring-бинов

```java
@SpringBootTest
@Listeners(SpringContextListener.class)
public class SpringAssertionsTest {

    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EntityManager entityManager;
    
    @Test(groups = {"integration"})
    public void testDatabaseState() {
        
        // Hard assert для критической проверки
        assertNotNull(userRepository);
        
        User user = userRepository.findById(1L).orElse(null);
        
        // Soft assert для множественных проверок
        SoftAssert soft = new SoftAssert();
        soft.assertNotNull(user);
        soft.assertEquals(user.getName(), "John");
        soft.assertNotNull(user.getCreatedAt());
        soft.assertAll();
    }
}
```

### AssertJ + Spring Data

```java
@Test
public void testWithSpringDataAndAssertJ() {
    Page<User> users = userRepository.findAll(PageRequest.of(0, 10));

    assertThat(users)
        .isNotNull()
        .hasSize(5)
        .extracting(User::getEmail)
        .allMatch(email -> email.endsWith("@example.com"));

}
```

### Проверка транзакций

```java
@Test
public void testTransactionRollback() {
    Long initialCount = userRepository.count();

    try {
        userService.createUserWithTransaction("John", "invalid");
        fail("Expected exception");
    } catch (Exception e) {
        // Транзакция должна откатиться
        assertEquals(userRepository.count(), initialCount);
    }
}
```

---

## Best Practices

✅ Делайте:

- Используйте `hard assertions` для критических проверок (null, тип, состояние)
- Применяйте `soft assertions` для e2e тестов с множественными проверками
- Всегда вызывайте `assertAll()` в конце для `soft assertions`
- Используйте AssertJ для сложных проверок коллекций и объектов
- Проверяйте сообщения исключений, а не только тип

❌ Не делайте:

- Не используйте `soft assertions` без `assertAll()` в конце
- Не смешивайте `hard assertions` и `soft assertions` в одном тесте без необходимости
- Не проверяйте только тип исключения без сообщения
- Не используйте `assertEquals` для null (используйте `assertNull`)
- Не полагайтесь на порядок проверок в `soft assertions`

---

## Ключевые выводы

1. **Hard assertions для unit-тестов** — Останавливаются при первой ошибке, быстрее отладка
2. **Soft assertions для e2e** — Все проверки выполняются, полный отчёт об ошибках
3. **`assertAll()` обязателен** — Без него `soft assertions` не отчитают о неудачах
4. **AssertJ для читаемости** — Fluent API делает тесты понятнее
5. **Проверяйте сообщения исключений** — Не только тип, но и содержимое
6. **Spring + assertions работают вместе** — Autowired бины проверяются как обычные объекты
7. **Не смешивайте фреймворки assertions** — TestNG или AssertJ, но не оба в одном тесте

# 6. Параметризация и Data Providers

> Параметризация позволяет запускать один тест с разными наборами данных. 
> TestNG предоставляет мощные механизмы: 
> - `@DataProvider` для гибкой подачи данных 
> - `@Parameters` для конфигурации из testng.xml
> - `@Factory` для динамического создания тестовых экземпляров.

---

## `@DataProvider` — основной механизм

`@DataProvider` — метод, возвращающий данные для параметризованных тестов.

### Базовое использование

```java
@DataProvider(name = "userData")
public Object[][] provideUserData() {
    return new Object[][]{
            {"John", "john@test.com", 25},
            {"Jane", "jane@test.com", 30},
            {"Bob", "bob@test.com", 35}
    };
}

@Test(dataProvider = "userData")
public void testUserCreation(String name, String email, int age) {
    User user = userService.create(name, email, age);
    assertNotNull(user);
    assertEquals(user.getName(), name);
}
```

### Возврат Iterator для ленивой загрузки

```java
@DataProvider(name = "largeDataSet")
public Iterator<Object[]> provideLargeData() {
    
    // ✅ Ленивая загрузка: данные генерируются по мере необходимости
    return new Iterator<Object[]>() {
        
        private int currentIndex = 0;
        private static final int TOTAL = 1000;

        @Override
        public boolean hasNext() {
            return currentIndex < TOTAL;
        }

        @Override
        public Object[] next() {
            // Генерируем данные ТОЛЬКО когда тест готов их принять
            int index = currentIndex++;
            return new Object[]{
                    "User" + index,
                    "user" + index + "@test.com"
            };
        }
    };
}

@Test(dataProvider = "largeDataSet")
public void testLargeData(String name, String email) {
    // Тест запустится 1000 раз, но данные создаются по одному
    System.out.println("Testing: " + name + " - " + email);
}
```

### Параллельная подача данных

С parallel = true в DataProvider:

- Каждый набор данных подается параллельно
- Все тесты, использующие этот DataProvider, выполняются параллельно
- Общее количество параллельных потоков = количество наборов данных × количество тестов
- Это может привести к взрывному росту потоков, если не контролировать через threadPoolSize или настройки suite

```java 
@DataProvider(name = "parallelData", parallel = true)
public Object[][] provideParallelData() {
    return new Object[][]{
            {1}, {2}, {3}, {4}, {5}
    };
}

@Test(dataProvider = "parallelData")
public void testParallel1(int value) {
    // Тесты выполнятся параллельно
}

@Test(dataProvider = "parallelData")
public void testParallel2(int value) {
    // Тесты выполнятся параллельно
}
```

---

## `@Parameters` — данные из `testng.xml`

@Parameters передаёт значения из XML-конфигурации.

### Конфигурация в testng.xml

```xml
<suite name="ParameterizedSuite">
    <parameter name="environment" value="staging"/>
    <parameter name="timeout" value="5000"/>

    <test name="ConfigTest">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.example.ConfigTest"/>
        </classes>
    </test>
</suite>
```

### Использование в тесте

```java
@Parameters({"environment", "timeout", "browser"})
@Test
public void testWithParameters(String env, String timeout, String browser) {
    System.out.println("Environment: " + env);
    System.out.println("Timeout: " + timeout);
    System.out.println("Browser: " + browser);

    assertEquals(env, "staging");
    assertEquals(browser, "chrome");
}
```

### Приоритет параметров

1. `@Parameters` из `<test>` (наивысший приоритет)
2. `@Parameters` из `<suite>`
3. Значения по умолчанию в коде

```java
@Parameters({"environment"})
@Test
public void testWithDefault(@Optional("production") String env) {
    // Если параметр не задан — используется "production"
}
```

---

## `@Factory` — динамическое создание тестов


`@Factory` создает несколько независимых экземпляров тестового класса, каждый со своим состоянием (параметрами). 

Это позволяет:

- Параметризовать целый класс тестов, а не отдельные методы
- Изолировать тестовые наборы друг от друга
- Выполнять разные конфигурации (БД, браузеры, окружения) в одном запуске
- Использовать жизненный цикл (`@BeforeClass`, `@AfterClass`) для каждого набора параметров

### Базовое использование

В примере создается 3 экземпляра класса, каждый с разным inputData, и в каждом экземпляре выполняется тест testData() со своим значением.

```java
public class ParameterizedTest {

    private String inputData;  // ← Параметр, с которым будет работать экземпляр

    // Конструктор принимает параметр
    public ParameterizedTest(String inputData) {
        this.inputData = inputData;  // ← Сохраняем параметр для тестов
    }

    @Factory
    public static Object[] createInstances() {
        // ← Фабрика создает экземпляры с разными параметрами
        return new Object[]{
                new ParameterizedTest("data1"),
                new ParameterizedTest("data2"),
                new ParameterizedTest("data3")
        };
    }

    @Test
    public void testData() {
        // ← Каждый экземпляр выполнит этот тест со своим inputData
        System.out.println("Testing with: " + inputData);
        assertNotNull(inputData);
    }
}
```

### Factory + DataProvider

Пример использования @Factory + @DataProvider:

```java
// @DataProvider — передаёт данные в метод фабрики
// @Factory — создаёт экземпляры тестового класса с разными данными из DataProvider 
// Каждый экземпляр может содержать несколько тестов, использующих одно и то же состояние
public class FactoryWithProviderExample {

    private int userId;           // Уникальный ID пользователя для этого экземпляра
    private String userName;      // Имя пользователя
    private boolean isActive;     // Статус активности

    // Конструктор инициализирует состояние тестового экземпляра.
    // Это состояние будет использоваться во всех тестах этого экземпляра.
    public FactoryWithProviderExample(int userId, String userName, boolean isActive) {
        this.userId = userId;
        this.userName = userName;
        this.isActive = isActive;
    }
    
    // @Factory создаёт экземпляры тестового класса.
    // Каждый экземпляр получает свои данные из @DataProvider.
    @Factory(dataProvider = "userDataProvider")
    public static Object[] createInstances(int userId, String userName, boolean isActive) {
        return new Object[]{new FactoryWithProviderExample(userId, userName, isActive)};
    }

    // @DataProvider предоставляет наборы данных для создания экземпляров.
    @DataProvider(name = "userDataProvider")
    public static Object[][] provideUserData() {
        return new Object[][]{
                {1, "Alice", true},
                {2, "Bob", false},
                {3, "Charlie", true}
        };
    }

    @Test
    public void testUserIdIsValid() {
        assertTrue(userId > 0, "ID должен быть положительным");
        assertNotNull(userName);
        assertFalse(userName.isEmpty(), "Имя не должно быть пустым");
        if (isActive) {
            System.out.println(userName + " активен");
        } else {
            System.out.println(userName + " неактивен");
        }
    }
}
```

---

## Внешние источники данных

### CSV файлы

```java
@DataProvider(name = "csvData")
public Object[][] readFromCsv() throws IOException {
    List<Object[]> data = new ArrayList<>();
    List<String> lines = Files.readAllLines(Paths.get("src/test/resources/test-data.csv"));

    for (String line : lines) {
        String[] values = line.split(",");
        data.add(values);
    }
    
    return data.toArray(new Object[0][]);
}

@Test(dataProvider = "csvData")
public void testFromCsv(String name, String email, String expected) {
    // Данные из CSV файла
}
```

### JSON файлы

```java
@DataProvider(name = "jsonData")
public Object[][] readFromJson() throws IOException {
    String json = Files.readString(Paths.get("src/test/resources/test-data.json"));
    ObjectMapper mapper = new ObjectMapper();
    List<UserData> users = mapper.readValue(json, new TypeReference<List<UserData>>() {});

    Object[][] data = new Object[users.size()][];
    for (int i = 0; i < users.size(); i++) {
        data[i] = new Object[]{users.get(i)};
    }
    
    return data;
}

@Test(dataProvider = "jsonData")
public void testFromJson(UserData userData) {
    // Данные из JSON файла
}
```

### База данных

```java
@DataProvider(name = "dbData")
public Object[][] readFromDatabase() {
    List<Object[]> data = new ArrayList<>();

    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT name, email FROM test_users")) {
        
        while (rs.next()) {
            data.add(new Object[]{rs.getString("name"), rs.getString("email")});
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
    
    return data.toArray(new Object[0][]);
}

@Test(dataProvider = "dbData")
public void testFromDatabase(String name, String email) {
    // Данные из БД
}
```

---

## Сводная таблица: механизмы параметризации

| Механизм     | Аннотация                    | Источник данных    | Когда использовать                         |
|:-------------|------------------------------|--------------------|--------------------------------------------|
| DataProvider | @DataProvider                | Java-метод         | Гибкая генерация данных в коде             |
| Parameters   | @Parameters                  | testng.xml         | Конфигурация окружения, браузеры           |
| Factory      | @Factory                     | Конструктор класса | Создание экземпляров с разными параметрами |
| CSV          | @DataProvider + Files        | CSV файл           | Табличные данные, Excel-экспорт            |
| JSON         | @DataProvider + ObjectMapper | JSON файл          | Сложные структуры данных                   |
| Database     | @DataProvider + JDBC         | БД                 | Реальные тестовые данные из БД             |

---

## Интеграция со Spring

### DataProvider + Spring Beans

```java
@SpringBootTest
@Listeners(SpringContextListener.class)
public class SpringDataProviderTest {

    @Autowired
    private TestDataRepository testDataRepository;
    
    @DataProvider(name = "springData")
    public Object[][] provideDataFromSpring() {
        // Доступ к Spring-бину в DataProvider
        List<TestData> data = testDataRepository.findAll();
        
        Object[][] result = new Object[data.size()][];
        for (int i = 0; i < data.size(); i++) {
            result[i] = new Object[]{data.get(i)};
        }
        return result;
    }
    
    @Test(dataProvider = "springData")
    public void testWithSpringData(TestData testData) {
        assertNotNull(testData);
        assertNotNull(testData.getId());
    }
}
```

### Static DataProvider для Factory

```java
public class SpringFactoryTest {

    private UserService userService;
    
    public SpringFactoryTest(String userType) {
        this.userService = SpringContextListener.getBean(UserService.class);
    }
    
    @Factory(dataProvider = "userTypes")
    public static Object[] createInstances(String userType) {
        return new Object[]{new SpringFactoryTest(userType)};
    }
    
    @DataProvider(name = "userTypes")
    public static Object[][] provideUserTypes() { // @Factory создаёт экземпляры класса, поэтому @DataProvider для него должен работать до создания экземпляров → должен быть static.
        return new Object[][]{
            {"admin"},
            {"user"},
            {"guest"}
        };
    }
    
    @Test
    public void testUserType() {
        // Тест для разных типов пользователей
    }
}
```

---

## Best Practices

✅ Делайте:
- Используйте `@DataProvider` для сложных наборов данных
- Применяйте `@Parameters` для конфигурации окружения
- Выносите данные во внешние файлы (CSV, JSON) для читаемости
- Используйте `parallel = true` для ускорения прогона больших наборов
- Кэшируйте данные из БД в `@BeforeSuite` для производительности

❌ Не делайте:
- Не создавайте `DataProvider` с тысячами записей без необходимости
- Не подключайтесь к БД в каждом вызове `DataProvider` (кэшируйте)
- Не используйте `@Factory` без веской причины (сложнее отладка)
- Не смешивайте `@DataProvider` и `@Parameters` в одном тесте
- Не забывайте про типизацию данных в `Object[][]`

---

## Ключевые выводы

1. **`@DataProvider` — основной механизм** — Гибкая генерация данных из любого источника
2. **`@Parameters` для конфигурации** — Передача параметров из testng.xml
3. **`@Factory` для экземпляров** — Создание тестовых объектов с разными параметрами
4. **Внешние файлы для данных** — CSV, JSON для отделения данных от кода
5. **`parallel = true` для скорости** — Параллельное выполнение параметризованных тестов
6. **Spring-бины в DataProvider** — Доступ к репозиториям и сервисам для данных
7. **Кэшируйте тяжёлые источники** — БД и API вызовы кэшируйте для производительности