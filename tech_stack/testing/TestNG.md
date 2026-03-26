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
- Используйте TestNG для e2e и Selenium, JUnit 5 для unit
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

