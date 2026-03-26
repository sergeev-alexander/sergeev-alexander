# JUnit 5

## Содержание

1. [Что такое JUnit](#1-что-такое-junit)
2. [Аннотации JUnit](#2-аннотации-junit)
3. [Assertions (Утверждения)](#3-assertions-утверждения)
4. [JUnit 5 Extensions](#4-junit-5-extensions)
5. [Параметризованные тесты](#5-параметризованные-тесты)
6. [Параллельные тесты](#6-параллельные-тесты)
7. [Dynamic Tests (Динамические тесты)](#7-dynamic-tests-динамические-тесты)
8. [Mocking в тестах](#8-mocking-в-тестах)
9. [Организация тестов](#9-организация-тестов)
10. [Запуск и отчётность](#10-запуск-и-отчётность)
11. [Миграция с JUnit 4](#11-миграция-с-junit-4)
12. [Best Practices](#12-best-practices)

---

# 1. Что такое JUnit

> JUnit — фреймворк для модульного тестирования в Java. 
>
> Автоматизирует проверку корректности отдельных частей кода (методов, классов, модулей).

---

## Архитектура JUnit 5

JUnit 5 состоит из трёх модулей:

```text
    ┌─────────────────────────────────────────────────┐
    │              JUnit 5 Platform                   │
    │  (запуск тестов, интеграция с IDE и build tools)│
    └─────────────────────────────────────────────────┘
               ▲                           ▲
               │                           │
     ┌─────────────────┐         ┌────────────────────┐
     │   JUnit Jupiter │         │    JUnit Vintage   │
     │  (новая модель) │         │ (запуск JUnit 3/4) │
     └─────────────────┘         └───────────────────v┘
```

- **Platform** - Базовая платформа для запуска тестов
- **Jupiter** - Новая модель тестирования (JUnit 5)
- **Vintage** - Обратная совместимость с JUnit 3/4

---

## Преимущества JUnit 5

- **Модульность** - Можно использовать только нужные компоненты
- **Лямбда-выражения** - Поддержка Java 8+ функциональности
- **Параметризованные тесты** - Один тест с разными данными
- **Расширения** - Гибкая система расширений вместо Rule
- **Динамические тесты** - Генерация тестов во время выполнения
- **Параллельное выполнение** - Встроенная поддержка параллелизма
- **Лучшие сообщения** - Более понятные сообщения об ошибках

---

## Подключение зависимостей

### Maven (pom.xml):

```xml
    <dependencies>
        <!-- JUnit 5 API -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
        
        <!-- JUnit 5 Engine для запуска -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
        
        <!-- Для запуска JUnit 4 тестов в JUnit 5 -->
        <dependency>
            <groupId>org.junit.vintage</groupId>
            <artifactId>junit-vintage-engine</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

### Gradle (build.gradle):

```groovy
    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.10.0'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.10.0'
        testRuntimeOnly 'org.junit.vintage:junit-vintage-engine:5.10.0'
    }

    test {
        useJUnitPlatform()
    }
```

### BOM для управления версиями:

```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.junit</groupId>
                <artifactId>junit-bom</artifactId>
                <version>5.10.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

---

## Для чего используют JUnit

- Тестирование бизнес-логики
- Тестирование методов
- Регрессионное тестирование
- Интеграция с CI/CD
- TDD
- Документирование

---

## Ключевые возможности

- **Автоматизация** - Тесты запускаются автоматически
- **Модульность** - Тестирование отдельных компонентов
- **Assertions** - Механизм проверки результатов
- **Жизненный цикл** - `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`
- **Параметризация** - Один тест с разными данными
- **Расширяемость** - Кастомные расширения через `@ExtendWith`
- **Документирование** - `@DisplayName` для понятных имён тестов

---

## JUnit в экосистеме Java

```text
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   IntelliJ  │     │    Maven    │     │   Jenkins   │
    │   Eclipse   │---->│    Gradle   │---->│   GitLab    │
    │   VS Code   │     │             │     │   GitHub    │
    └─────────────┘     └─────────────┘     └─────────────┘
         IDE               Build Tools           CI/CD
```

---

# 2. Аннотации JUnit

> Аннотации в JUnit 5 определяют поведение тестовых методов, их жизненный цикл и конфигурацию. 
> 
> Делают тесты читаемыми и поддерживаемыми.

---

## Основные аннотации

- `@Test` — Обозначает метод как тестовый. Метод будет выполнен как тестовый случай.
- `@DisplayName` — Задаёт человеко-понятное имя для теста вместо имени метода.
- `@ParameterizedTest` — Позволяет запускать один тест с различными наборами данных.

---

## Аннотации жизненного цикла

- `@BeforeEach` — Метод выполняется перед каждым тестом в классе. Используется для инициализации.
- `@AfterEach` — Метод выполняется после каждого теста в классе. Используется для очистки ресурсов.
- `@BeforeAll` — Метод выполняется один раз перед всеми тестами. **Должен быть `static`**.
- `@AfterAll` — Метод выполняется один раз после всех тестов. **Должен быть `static`**.

---

## Аннотации группировки

- `@Nested` — Создаёт вложенный класс для группировки связанных тестов.
- `@Tag` — Присваивает тег тесту или классу для фильтрации при запуске.

---

## Аннотации отключения

- `@Disabled` — Отключает тест или класс. Полезно для временного отключения неработающих тестов.
- `@DisabledIf` — Отключает тест при выполнении определённого условия.
- `@EnabledIf` — Включает тест только при выполнении определённого условия.

---

## Порядок выполнения тестов

- `@TestMethodOrder` — Определяет порядок выполнения тестовых методов в классе.
- `@Order` — Задаёт порядок выполнения для конкретного метода (вместе с `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)`).

---

## Дополнительные аннотации

- `@RepeatedTest` — Выполняет тест указанное количество раз.
- `@Timeout` — Ограничивает максимальное время выполнения теста.
- `@TestInstance` — Управляет жизненным циклом экземпляра тестового класса (`PER_METHOD` или `PER_CLASS`).
- `@ExtendWith` — Регистрирует расширения для кастомного поведения тестов.
- `@RegisterExtension` — Программная регистрация расширений в поле теста.

---

## Сравнение JUnit 4 и JUnit 5 аннотаций

| JUnit 4      | JUnit 5     | Описание            |
|:-------------|-------------|---------------------|
| @Test        | @Test       | Тестовый метод      |
| @Before      | @BeforeEach | Перед каждым тестом |
| @After       | @AfterEach  | После каждого теста |
| @BeforeClass | @BeforeAll  | Перед всеми тестами |
| @AfterClass  | @AfterAll   | После всех тестов   |
| @Ignore      | @Disabled   | Отключить тест      |
| @Category    | @Tag        | Тег для фильтрации  |
| @Rule        | @ExtendWith | Расширения          |

---

# 3. Assertions (Утверждения)

> Assertions — механизм проверки результатов тестов. Если утверждение не выполняется, тест считается проваленным.

---

## Основные утверждения

- `assertEquals(expected, actual)` — Проверяет равенство двух значений
- `assertEquals(expected, actual, message)` — Проверяет равенство с сообщением об ошибке
- `assertNotEquals(unexpected, actual)` — Проверяет неравенство двух значений
- `assertTrue(condition)` — Проверяет что условие истинно
- `assertFalse(condition)` — Проверяет что условие ложно
- `assertNull(object)` — Проверяет что объект равен null
- `assertNotNull(object)` — Проверяет что объект не равен null
- `assertSame(expected, actual)` — Проверяет что ссылки указывают на один объект
- `assertNotSame(unexpected, actual)` — Проверяет что ссылки указывают на разные объекты

---

## Утверждения для массивов и коллекций

- `assertArrayEquals(expected, actual)` — Проверяет равенство массивов
- `assertArrayEquals(expected, actual, message)` — Проверяет равенство массивов с сообщением
- `assertIterableEquals(expected, actual)` — Проверяет равенство итерируемых коллекций
- `assertLinesMatch(expected, actual)` — Проверяет соответствие строк (поддерживает regex)
- `assertContains(collection, element)` — Проверяет наличие элемента в коллекции (AssertJ)
- `assertAll()` — Группирует несколько утверждений для выполнения всех проверок

---

## Утверждения для исключений

- `assertThrows(exceptionType, executable)` — Проверяет что код выбрасывает ожидаемое исключение
- `assertThrows(exceptionType, executable, message)` — Проверяет исключение с сообщением
- `assertDoesNotThrow(executable)` — Проверяет что код не выбрасывает исключений
- `assertDoesNotThrow(executable, message)` — Проверяет отсутствие исключений с сообщением

### Пример использования assertThrows:

```java
    Executable executable = () -> {
        service.methodThatThrows();
    };
    Exception exception = assertThrows(IllegalArgumentException.class, executable);
    assertEquals("Invalid argument", exception.getMessage());
```

---

## Утверждения для таймаутов

- `assertTimeout(duration, executable)` — Проверяет что тест выполняется за указанное время
- `assertTimeoutPreemptively(duration, executable)` — Прерывает тест если превышено время
- `assertTimeout(duration, executable, message)` — Проверяет таймаут с сообщением

### Важно:

- `assertTimeout` - Тест выполняется до конца, потом проверка
- `assertTimeoutPreemptively` - Тест прерывается немедленно

---

## Группировка утверждений (assertAll)

- `assertAll(executables...)` — Выполняет все утверждения даже если некоторые упали
- `assertAll(groupedExecutables)` — Группирует утверждения по категориям

Сигнатуры методов:
```java
// 1. Только с заголовком
assertAll(String heading, Executable... executables){...}

// 2. Без заголовка
assertAll(Executable... executables){...}

// 3. С коллекцией
assertAll(String heading, Collection<Executable> executables){...}
```

### Преимущества assertAll:

| Обычные assertions               | assertAll                   |
|:---------------------------------|-----------------------------|
| Останавливается на первой ошибке | Выполняет все проверки      |
| Одно сообщение об ошибке         | Все ошибки в одном отчёте   |
| Нужно несколько тестов           | Один тест для всех проверок |

### Пример:

```java
    assertAll("user properties",
        () -> assertEquals("John", user.getName()),
        () -> assertEquals(30, user.getAge()),
        () -> assertTrue(user.isActive())
    );
```

---

## Assumptions (Предположения)

Assumptions — проверки, которые определяют, стоит ли запускать тест. 

Если предположение не выполняется, тест пропускается (`skipped`).

- `assumeTrue(condition)` — Выполняет тест только если условие истинно
- `assumeFalse(condition)` — Выполняет тест только если условие ложно
- `assumingThat(condition, executable)` — Выполняет блок только если условие истинно

### Когда использовать:

| Ситуация                           | Решение                                      |
|:-----------------------------------|----------------------------------------------|
| Тест только для определённой ОС    | assumeTrue(isWindows())                      |
| Тест только при наличии переменной | assumeTrue(System.getenv("API_KEY") != null) |
| Тест только в определённой среде   | assumeTrue(isStagingEnvironment())           |

### Пример:

```java
    @Test
    void testOnlyOnWindows() {
        assumeTrue(System.getProperty("os.name").contains("Windows"));
        // Тест выполнится только на Windows
    }
```

---

## Lambda-выражения в assertions

JUnit 5 поддерживает лямбда-выражения для отложенного вычисления сообщений об ошибках.

### Без лямбды (вычисляется всегда):

```java
    assertEquals(expected, actual, "Expensive message: " + computeMessage());
```

### С лямбдой (вычисляется только при ошибке):

```java
    assertEquals(expected, actual, () -> "Expensive message: " + computeMessage());
```

### Преимущества:

- Сообщение вычисляется только при ошибке
- Быстрее при успешном тесте
- Нужно для дорогих вычислений

---

## Сводная таблица Assertions

| Категория     | Метод              | Описание                       |
|:--------------|--------------------|--------------------------------|
| Равенство     | assertEquals       | Проверка равенства             |
| Равенство     | assertNotEquals    | Проверка неравенства           |
| Логические    | assertTrue         | Проверка истинности            |
| Логические    | assertFalse        | Проверка ложности              |
| Null          | assertNull         | Проверка на null               |
| Null          | assertNotNull      | Проверка на не-null            |
| Ссылки        | assertSame         | Проверка одинаковой ссылки     |
| Ссылки        | assertNotSame      | Проверка разной ссылки         |
| Массивы       | assertArrayEquals  | Проверка равенства массивов    |
| Исключения    | assertThrows       | Проверка выброса исключения    |
| Исключения    | assertDoesNotThrow | Проверка отсутствия исключений |
| Таймаут       | assertTimeout      | Проверка времени выполнения    |
| Группировка   | assertAll          | Групповое выполнение проверок  |
| Предположения | assumeTrue         | Условное выполнение теста      |
| Предположения | assumeFalse        | Условное выполнение теста      |

---

## AssertJ альтернатива

AssertJ — популярная альтернатива стандартным assertions с fluent API.

### Maven зависимость:

```xml
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>
```

### Примеры AssertJ:

```java
    // Вместо assertEquals
    assertThat(result).isEqualTo(5);
    
    // Вместо assertTrue
    assertThat(user.isActive()).isTrue();
    
    // Вместо assertNull
    assertThat(value).isNull();
    
    // Цепочка проверок
    assertThat(user)
        .isNotNull()
        .hasName("John")
        .hasAge(30)
        .isActive();
    
    // Коллекции
    assertThat(list)
        .hasSize(3)
        .contains("A", "B")
        .doesNotContain("Z");
    
    // Исключения
    assertThatThrownBy(() -> service.method())
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("Invalid argument");
```

> Цепочка проверок в AssertJ выполняется **строго последовательно**, и прерывается на первой же упавшей проверке.
> 
> - Если проверки зависимы (например, сначала проверить isNotNull(), потом использовать объект) — используйте цепочку
> - Если проверки независимы и нужно увидеть все проблемы сразу — используйте SoftAssertions или assertAll

# 4. JUnit 5 Extensions

> Extensions — механизм расширения функциональности тестов в JUnit 5. 
> 
> Позволяет добавлять кастомное поведение без наследования и правил.

Extension — класс который реализует один или несколько интерфейсов расширения JUnit 5 и подключается через `@ExtendWith`.

### Ключевые особенности:

- **Модульность** - Одно расширение = одна ответственность
- **Композиция** - Можно подключать несколько расширений
- **Переиспользование** - Одно расширение для множества тестов
- **Декларативность** - Подключение через аннотации
- **Типобезопасность** - Внедрение параметров по типу

### Отличия от JUnit 4 Rules:

| JUnit 4 Rules      | JUnit 5 Extensions    |
|:-------------------|-----------------------|
| Только для методов | Для классов и методов |
| Наследование       | Композиция            |
| Ограниченный API   | Богатый API           |
| Одно правило       | Несколько расширений  |
| @Rule / @ClassRule | @ExtendWith           |

---

## Встроенные расширения

JUnit 5 предоставляет несколько встроенных расширений:

- `TempDirectory` - Создание временных директорий
- `Timeout` - Ограничение времени выполнения
- `RepeatedTest` - Повторное выполнение тестов
- `ParameterizedTest` - Параметризованные тесты
- `MockitoExtension` - Mock объектов (Mockito)
- `SpringExtension` - Интеграция со Spring

### Пример TempDirectory:

```java
    @ExtendWith(TempDirectory.class)
    class TempDirDemo {
        @TempDir
        Path tempDir;  // Автоматически создаётся и удаляется
    }
```

---

## Интерфейсы расширений

### Callback интерфейсы (выполнение кода):

- `BeforeAllCallback` - Перед всеми тестами в классе
- `AfterAllCallback` - После всех тестов в классе
- `BeforeEachCallback` - Перед каждым тестом
- `AfterEachCallback` - После каждого теста
- `BeforeTestExecutionCallback` - Перед выполнением теста
- `AfterTestExecutionCallback` - После выполнения теста

### Condition интерфейсы (условия):

- `ExecutionCondition` - Определяет выполняется ли тест
- `TestInstancePostProcessor` - Обработка экземпляра теста
- `TestInstancePreDestroyCallback` - Перед уничтожением экземпляра

### Parameter интерфейсы (параметры):

- `ParameterResolver` - Внедрение параметров в тест
- `ParameterContext` - Контекст параметра

### Exception интерфейсы (исключения):

- `TestExecutionExceptionHandler` - Обработка исключений в тестах
- `InvocationInterceptor` - Перехват вызова теста

---

## Создание собственных расширений

### Шаг 1: Реализовать интерфейс

```java
    public class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback {
        
        @Override
        public void beforeTestExecution(ExtensionContext context) {
            // Код перед тестом
        }
        
        @Override
        public void afterTestExecution(ExtensionContext context) {
            // Код после теста
        }
    }
```    

### Шаг 2: Подключить расширение

```java
    @ExtendWith(TimingExtension.class)
    class MyTest {
   
        @Test
        void testExample() {
            // Тест
        }
    }
```    

---

## Регистрация расширений

### Способ 1: @ExtendWith (декларативный)

```java
    @ExtendWith({Extension1.class, Extension2.class})
    class MyTest { }
```

**Преимущества:**

- Чётко видно какие расширения используются
- Типобезопасно
- Легко включать/выключать

---

### Способ 2: @RegisterExtension (программный)

```java
    class MyTest {
        
        @RegisterExtension
        static TimingExtension extension = new TimingExtension();
    }
```

**Преимущества:**

- Можно настроить расширение перед использованием
- Доступ к полям теста
- Динамическая регистрация

---

### Способ 3: Автоматическая регистрация

Создать файл `META-INF/services/org.junit.jupiter.api.extension.Extension` и указать класс расширения.

**Преимущества:**

- Не нужно указывать в каждом тесте
- Работает глобально
- Удобно для библиотек

---

## ExtensionContext

> ExtensionContext — контекст, который предоставляет информацию о текущем тесте.

### Возможности:

- `getDisplayName`() - Имя теста
- `getTestClass()` - Класс теста
- `getTestMethod()` - Метод теста
- `getStore(namespace)` - Хранилище для данных
- `getParent()` - Родительский контекст
- `getConfigurationParameter()` - Параметры конфигурации

### Store для обмена данными:

> Store - это способ передать данные между разными этапами выполнения тестов и между разными расширениями JUnit, 
> не делая их статическими полями или не используя внешние контейнеры.

```java
    // Сохранить в Before
    context.getStore(Namespace.GLOBAL).put("key", value);
    
    // Получить в After
    Object value = context.getStore(Namespace.GLOBAL).get("key");
```

---

## Популярные расширения

### Mockito Extension:

```java
    @ExtendWith(MockitoExtension.class)
    class MyTest {

        @Mock
        Service service;
        
        @InjectMocks
        Controller controller;
    }
```

**Зависимость:**

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.0.0</version>
    <scope>test</scope>
</dependency>
```

---

### Spring Extension:

```java
    @ExtendWith(SpringExtension.class)
    @SpringBootTest
    class MyTest {
        @Autowired
        Repository repository;
    }
```

**Зависимость:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

---

### WireMock Extension:

```java
    @ExtendWith(WireMockExtension.class)
    class MyTest {
        
        @WireMockTest
        void testWithMockServer() { }
    }
```

---

### Database Rider Extension:

```java
    @ExtendWith(DBRiderExtension.class)
    @DataSet("data.yml")
    class MyTest {
        @Test
        void testWithDatabase() { }
    }
```

---

## Приоритет расширений

Когда подключено несколько расширений, они выполняются в определённом порядке.

### Порядок выполнения:

| Тип                | Порядок              |
|:-------------------|----------------------|
| BeforeAllCallback  | По возрастанию order |
| AfterAllCallback   | По убыванию order    |
| BeforeEachCallback | По возрастанию order |
| AfterEachCallback  | По убыванию order    |

### Настройка порядка:

```java
    @Order(1)
    public class FirstExtension implements BeforeEachCallback { }
    
    @Order(2)
    public class SecondExtension implements BeforeEachCallback { }
```

---

## Conditional Extensions

Расширения можно подключать условно:

```java
    @ExtendWith(WindowsExtension.class)
    @EnabledOnOs(OS.WINDOWS)
    class WindowsTest { }
    
    @ExtendWith(LinuxExtension.class)
    @EnabledOnOs(OS.LINUX)
    class LinuxTest { }
```

### Условия:

- `@EnabledOnOs` - Операционная система
- `@EnabledOnJre` - Версия Java
- `@EnabledIfSystemProperty` - Системное свойство
- `@EnabledIfEnvironmentVariable` - Переменная окружения
- `@DisabledIf` - Условие отключения

---

## Best Practices для Extensions

### Делайте:

- Одно расширение = одна ответственность
- Используйте Store для обмена данными между callbacks
- Регистрируйте расширения декларативно когда возможно
- Документируйте что делает расширение
- Тестируйте сами расширения

### Не делайте:

- Не создавайте расширения для простой логики (используйте @BeforeEach)
- Не храните состояние в полях расширения (stateless)
- Не используйте расширения когда достаточно встроенных функций
- Не игнорируйте исключения в расширениях

---

## Сводная таблица интерфейсов расширений

| Интерфейс                     | Метод                          | Назначение                 |
|:------------------------------|--------------------------------|----------------------------|
| BeforeAllCallback             | beforeAll()                    | Перед всеми тестами        |
| AfterAllCallback              | afterAll()                     | После всех тестов          |
| BeforeEachCallback            | beforeEach()                   | Перед каждым тестом        |
| AfterEachCallback             | afterEach()                    | После каждого теста        |
| BeforeTestExecutionCallback   | beforeTestExecution()          | Перед выполнением теста    |
| AfterTestExecutionCallback    | afterTestExecution()           | После выполнения теста     |
| ParameterResolver             | resolveParameter()             | Внедрение параметров       |
| TestExecutionExceptionHandler | handleTestExecutionException() | Обработка исключений       |
| ExecutionCondition            | evaluateExecutionCondition()   | Условие выполнения теста   |
| TestInstancePostProcessor     | postProcessTestInstance()      | Обработка экземпляра теста |

---

## Ключевые выводы

1. **Extensions — замена Rules** — более гибкий механизм чем JUnit 4 Rules
2. **Композиция вместо наследования** — подключайте несколько расширений
3. **Встроенные расширения покрывают 80% нужд** — начинайте с них
4. **Store для обмена данными** — используйте `ExtensionContext.Store`
5. **Декларативная регистрация проще** — `@ExtendWith` предпочтительнее
6. **Порядок важен** — контролируйте через `@Order`
7. **Популярные библиотеки имеют расширения** — Mockito, Spring, WireMock

# 5. Параметризованные тесты

> Параметризованные тесты позволяют запускать один тестовый метод с разными наборами входных данных. 
>
> Устраняют дублирование кода и улучшают поддерживаемость тестов.

---

## Аннотация @ParameterizedTest

- `@ParameterizedTest` — Заменяет `@Test` для тестов с параметрами
- Требует указания источника данных (`@ValueSource`, `@CsvSource`, etc.)
- Метод должен иметь параметры соответствующие источнику данных
- Каждый набор данных запускает тест отдельно

### Отличия от @Test:

| @Test            | @ParameterizedTest                |
|:-----------------|-----------------------------------|
| Один запуск      | Multiple запусков                 |
| Без параметров   | С параметрами                     |
| Простые сценарии | Проверка различных входных данных |

---

## Источники данных

### @ValueSource

- Простой источник для примитивов и строк
- Поддерживаемые типы: `int`, `long`, `double`, `boolean`, `String`
- Нельзя использовать для сложных объектов

```java
  @ParameterizedTest
  @ValueSource(ints = {1, 2, 3, 4, 5})
  void testPositiveNumbers(int number) {
    assertTrue(number > 0);
  }
```

---

### @EnumSource

- Передаёт значения enum в тест
- Можно указать все значения или выбрать конкретные
- Поддерживает EnumSource.Mode.INCLUDE(по умолчанию) / EXCLUDE / MATCH_ALL / MATCH_ANY режимы

```java
  @ParameterizedTest
  @EnumSource(DayOfWeek.class)
  void testAllDays(DayOfWeek day) { }

  @ParameterizedTest
  @EnumSource(value = DayOfWeek.class, names = {"MONDAY", "FRIDAY"})
  void testWeekdays(DayOfWeek day) { }
```

```java
  enum Priority {
    LOW, MEDIUM, HIGH, CRITICAL, URGENT
  }

  @ParameterizedTest
  @EnumSource(value = Priority.class,
        names = {"LOW", "MEDIUM"},
        mode = EnumSource.Mode.EXCLUDE)
  void testHighPriorityOnly(Priority priority) {
    // Тест выполнится для HIGH, CRITICAL, URGENT
    // LOW и MEDIUM исключены
    assertThat(priority).isNotIn(Priority.LOW, Priority.MEDIUM);
}
```

---

### @CsvSource

- Передаёт несколько параметров через CSV строки
- Разделитель по умолчанию — запятая
- Поддерживает кастомные разделители

```java
  @ParameterizedTest
  @CsvSource({
        "apple, 5",
        "banana, 6",
        "cherry, 6"
  })
  void testFruits(String fruit, int rank) {
    assertEquals(fruit.length(), rank);
  }

  @ParameterizedTest
  @CsvSource(value = {
        "apple | 1 | red",
        "banana | 2 | yellow",
        "cherry | 3 | red"
  }, delimiterString = " | ")
  void testFruitsWithPipe(String fruit, int rank, String color) {
    assertEquals(fruit.length(), rank);
  }
```
---

### @CsvFileSource

- Загружает данные из CSV файла
- Удобно для больших наборов данных
- Файл должен находиться в src/test/resources (или в classpath тестов)
- Можно читать из нескольких файлов `@CsvFileSource(resources = {"/file1.csv", "/file2.csv"})`

```java
  @ParameterizedTest
  @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
  void testFromFile(String input, int expected) { }
```

---

### @MethodSource

- Данные предоставляются методом
- Метод должен возвращать `Stream`, `Iterable` или массив
- Метод должен быть `static` (для non-per-class lifecycle: `@TestInstance(Lifecycle.PER_METHOD)` (по умолчанию))

```java
  @ParameterizedTest
  @MethodSource("stringProvider")
  void testWithMethod(String argument) {
    assertTrue(argument.length() > 0);
  }

  static Stream<String> stringProvider() {
    return Stream.of("apple", "banana", "cherry");
  }
```

---

### @ArgumentsSource

- Кастомный источник данных
- Реализует интерфейс ArgumentsProvider
- Для сложной логики генерации данных

```java
class TestClass {

    @ParameterizedTest
    @ArgumentsSource(CustomProvider.class)
    void testCustom(String fruit, int rank) { }
}    
    
class CustomProvider implements ArgumentsProvider {
  
  @Override
  public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
    return Stream.of(
        Arguments.of("apple", 1),
        Arguments.of("banana", 2)
    );
  }
}
```

---

## Именование параметризованных тестов

### @DisplayName с плейсхолдерами:

- `{0}`, `{1}`, `{2}` — значения параметров
- `{displayName}` — имя теста
- `{index}` — номер итерации

```java
  @ParameterizedTest(name = "Тест {index}: {0} = {1}")
  @CsvSource({
        "apple, 1",
        "banana, 2"
  })
  void testNamed(String fruit, int rank) { }
```

---

### Pattern для именования:

- `{index}` — Номер итерации (1, 2, 3...)
- `{0}`, `{1}`, `{2}` — Значения параметров
- `{displayName}` — Имя метода теста
- `{arguments}` — Все аргументы
- `{argumentsWithNames}` — Аргументы с именами

---

## Агрегация аргументов

### @AggregateWith

- Преобразует несколько аргументов в один объект
- Требует реализации ArgumentsAggregator
- Упрощает работу со сложными объектами

```java
class TestClass {

  @ParameterizedTest
  @CsvSource({
        "John, 30",
        "Jane, 25"
  })
  void testPerson(@AggregateWith(PersonAggregator.class) Person person) {
      assertTrue(person.age > 20);
  }
}

class PersonAggregator implements ArgumentsAggregator {
  
  @Override
  public Person aggregateArguments(ArgumentsAccessor accessor, ExtensionContext context) {
      return new Person(accessor.getString(0), accessor.getInteger(1));
  }
}
```

---

### ArgumentsAccessor

- Доступ к аргументам по индексу
- Методы для различных типов (getString, getInteger, etc.)
- Валидация типов

- `getString(index)` — Получить строку
- `getInteger(index)` — Получить Integer
- `getLong(index)` — Получить Long
- `getBoolean(index)` — Получить Boolean
- `getObject(index, Class)` — Получить объект типа

---

## Conversion и Type Matching

### Автоматическая конвертация:

- String → Integer, Long, Double
- String → Enum
- String → LocalDate, LocalDateTime

### Кастомные конвертеры:

- Реализовать ArgumentConverter
- Использовать @ConvertWith

```java
  @ParameterizedTest
  @CsvSource({"2026-01-01", "2026-03-26"})
  void testDate(@ConvertWith(DateConverter.class) LocalDate date) { }
```

---

## Сводная таблица источников данных

| Аннотация        | Тип данных        | Параметры            | Когда использовать              |
|:-----------------|-------------------|----------------------|---------------------------------|
| @ValueSource     | Примитивы, String | Один параметр        | Простые значения                |
| @EnumSource      | Enum              | Один параметр        | Тестирование всех значений enum |
| @CsvSource       | CSV строки        | Несколько параметров | Табличные данные                |
| @CsvFileSource   | CSV файл          | Несколько параметров | Большие наборы данных           |
| @MethodSource    | Метод             | Любое количество     | Сложная логика генерации        |
| @ArgumentsSource | ArgumentsProvider | Любое количество     | Кастомные источники             |

---

## Best Practices для параметризованных тестов

### Делайте:

- Используйте понятные имена с плейсхолдерами
- Выбирайте источник данных по типу данных
- Группируйте связанные тестовые данные
- Используйте `@MethodSource` для сложной логики
- Документируйте что проверяет каждый набор данных

### Не делайте:

- Не используйте для фундаментально разных тестов
  - Одна ответственность — один параметризованный тест = один сценарий/метод с разными данными
  - Связанность данных — все данные в тесте должны проверять одну и ту же логику
  - Понятность — если для понимания теста нужны комментарии/if/switch — скорее всего, нужно разделить
  - Легкость отладки — падение теста должно сразу указывать на конкретную проблему
- Не создавайте огромные наборы данных (> 100)
- Не используйте когда достаточно обычных тестов

---

## Когда использовать параметризованные тесты

| Сценарий                        | Решение                           |
|:--------------------------------|-----------------------------------|
| Проверка граничных значений     | @ValueSource с граничными числами |
| Тестирование всех enum значений | @EnumSource                       |
| Табличные данные (input/output) | @CsvSource                        |
| Сложные объекты для тестов      | @MethodSource с фабриками         |
| Данные из внешних источников    | @CsvFileSource                    |
| Специфическая логика генерации  | @ArgumentsSource                  |

---

# 6. Параллельные тесты

> Параллельное выполнение тестов ускоряет прогон тестовых наборов за счёт использования нескольких потоков. 
> 
> Особенно эффективно для больших проектов с независимыми тестами.

---

## Настройка параллельного выполнения

```text
                          Глобальное включение (обязательно)
                                        ↓
                                Параллелизм ВКЛЮЧЁН
                                        ↓
                                Аннотация @Execution
                                        ↓
                          ┌─────────────┴─────────────┐
                          ↓                           ↓
 @Execution(ExecutionMode.CONCURRENT)       @Execution(ExecutionMode.SAME_THREAD)
           (параллельно)                            (последовательно)
```

### Через `junit-platform.properties`:

Файл размещается в `src/test/resources`

```properties
    junit.jupiter.execution.parallel.enabled = true
    junit.jupiter.execution.parallel.mode.default = concurrent
    junit.jupiter.execution.parallel.mode.classes.default = concurrent
```

### Через системные свойства JVM:

```bash
    -Djunit.jupiter.execution.parallel.enabled=true \
    -Djunit.jupiter.execution.parallel.mode.default=concurrent \
    -Djunit.jupiter.execution.parallel.mode.classes.default=concurrent
```

### Через Maven (pom.xml):

```xml
    <configuration>
        <properties>
            <configurationParameters>
                junit.jupiter.execution.parallel.enabled = true
                junit.jupiter.execution.parallel.mode.default = concurrent
            </configurationParameters>
        </properties>
    </configuration>
```

### Через Gradle (build.gradle):

```groovy
    test {
        useJUnitPlatform()
        systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
        systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
    }
```

---

## Конфигурационные параметры

### Основные параметры:

- `junit.jupiter.execution.parallel.enabled` — Включает параллельное выполнение (true/false)
- `junit.jupiter.execution.parallel.mode.default` — Режим для методов (concurrent/same_thread)
- `junit.jupiter.execution.parallel.mode.classes.default` — Режим для классов (concurrent/same_thread)
- `junit.jupiter.execution.parallel.config.strategy` — Стратегия управления потоками (fixed/dynamic)
- `junit.jupiter.execution.parallel.config.fixed.parallelism` — Фиксированное количество потоков
- `junit.jupiter.execution.parallel.config.dynamic.factor` — Коэффициент для dynamic стратегии

---

## Режимы выполнения

### ExecutionMode:

- `CONCURRENT` — Тест выполняется в отдельном потоке (параллельно)
- `SAME_THREAD` — Тест выполняется в том же потоке (последовательно)

### Настройка на уровне класса:

```java
    @Execution(ExecutionMode.CONCURRENT)
    class MyParallelTest {
        
        @Test
        void test1() { }
        
        @Test
        void test2() { }
    }
```

### Настройка на уровне метода:

```java
    class MyTest {

        @Test
        @Execution(ExecutionMode.CONCURRENT)
        void parallelTest() { }
        
        @Test
        @Execution(ExecutionMode.SAME_THREAD)
        void sequentialTest() { }
    }
```

---

## Стратегии управления потоками

### Fixed стратегия:

- Фиксированное количество потоков
- Предсказуемое использование ресурсов
- Подходит для CI/CD окружений

```properties
  junit.jupiter.execution.parallel.config.strategy = fixed
  junit.jupiter.execution.parallel.config.fixed.parallelism = 4
```

### Dynamic стратегия:

- Количество потоков зависит от доступных процессоров
- Формула: `availableProcessors * dynamic.factor`
- Подходит для локальной разработки

```properties
  junit.jupiter.execution.parallel.config.strategy = dynamic
  junit.jupiter.execution.parallel.config.dynamic.factor = 2.0
```

### По умолчанию:

- Strategy: dynamic
- Factor: 1.0 (равно количеству доступных процессоров)

---

## 6.5. Управление ресурсами

### @ResourceLock аннотация:

> `@ResourceLock` решает проблему конкуренции за общие ресурсы при параллельном выполнении тестов. 
> 
> Когда несколько тестов одновременно обращаются к одному ресурсу (базе данных, файловой системе, настройкам локали), 
> могут возникать непредсказуемые ошибки из-за состояния гонки. 
> 
> Аннотация создаёт барьер синхронизации: все тесты, помеченные одинаковым идентификатором ресурса, 
> не будут выполняться одновременно — они будут выстроены в очередь, 
> что гарантирует целостность данных и стабильность результатов. Режим `READ` позволяет параллельное чтение 
> (например, несколько тестов могут одновременно проверять конфигурацию), 
> а `READ_WRITE` обеспечивает эксклюзивный доступ (только один тест может модифицировать ресурс в любой момент времени).

```java
    @Test
    @ResourceLock("database")
    void testDatabase() { }
    
    @Test
    @ResourceLock("file-system")
    void testFileSystem() { }
```

### Режимы блокировки:

- `READ` — Несколько тестов могут читать ресурс одновременно
- `READ_WRITE` — Эксклюзивный доступ (чтение + запись)

```java
  @Test
  @ResourceLock(value = "config", mode = ResourceLock.Mode.READ)
  void readConfigTest() { }

  @Test
  @ResourceLock(value = "config", mode = ResourceLock.Mode.READ_WRITE)
  void writeConfigTest() { }
```

### Встроенные ресурсы:

- `java.util.Locale` — Локаль системы
- `java.util.TimeZone` — Часовой пояс
- `java.nio.charset.Charset` — Кодировка
- `sun.stdout` / `sun.stderr` — Потоки вывода

---

## Проблемы thread-safety

### Общие проблемы:

- Статические переменные разделяются между потоками. Если один поток изменяет её значение, все остальные потоки видят это изменение.
- Общие файлы могут быть повреждены
- Подключения к БД могут конфликтовать
- Порядок выполнения не гарантируется

### Решения:

- Используйте `@ResourceLock` для общих ресурсов
- Избегайте статических изменяемых полей
- Создавайте изолированные данные для каждого теста
- Используйте `ThreadLocal` для потоко-безопасных данных

### Пример проблемного кода:

```java
    // ПЛОХО - статическое поле разделяется
    static int counter = 0;
    
    @Test
    void test1() { counter++; }
    
    @Test
    void test2() { counter++; }
```

### Пример исправленного кода:

```java
    // ХОРОШО - поле экземпляра
    int counter = 0;
    
    @Test
    void test1() { counter++; }
    
    @Test
    void test2() { counter++; }
```

---

## Best Practices для параллельных тестов

### Делайте:

- Делайте тесты независимыми друг от друга
- Используйте `@ResourceLock` для общих ресурсов
- Избегайте статических изменяемых полей
- Тестируйте каждый тест отдельно перед параллельным запуском
- Используйте разделённые тестовые данные для каждого потока

### Не делайте:

- Не полагайтесь на порядок выполнения тестов
- Не используйте общие файлы без блокировки
- Не делите состояния между тестами
- Не используйте параллелизм для маленьких наборов тестов (overhead)

---

## Когда использовать параллелизм

| Сценарий                      | Рекомендация                          |
|:------------------------------|---------------------------------------|
| Много независимых unit тестов | Включить параллелизм                  |
| Интеграционные тесты с БД     | Осторожно, использовать @ResourceLock |
| Тесты с общими файлами        | Использовать @ResourceLock            |
| Маленький набор тестов (< 50) | Не включать (overhead)                |
| CI/CD pipeline                | Включить для ускорения                |
| Flaky тесты                   | Сначала исправить, потом параллелить  |

---

## Сводная таблица параметров

| Параметр                 | Значение по умолчанию | Описание                     |
|:-------------------------|-----------------------|------------------------------|
| parallel.enabled         | false                 | Включение параллелизма       |
| mode.default             | same_thread           | Режим для методов            |
| mode.classes.default     | same_thread           | Режим для классов            |
| config.strategy          | dynamic               | Стратегия потоков            |
| config.dynamic.factor    | 1.0                   | Коэффициент для dynamic      |
| config.fixed.parallelism | availableProcessors   | Количество потоков для fixed |

---

# 7. Dynamic Tests (Динамические тесты)

> Динамические тесты — это тесты которые генерируются во время выполнения, а не на этапе компиляции. 
> 
> Они позволяют создавать тесты программно на основе данных или условий.

---

## Аннотация @TestFactory

- `@TestFactory` — Заменяет `@Test` для динамических тестов
- Метод должен возвращать `Stream`, `Collection`, `Iterable` или `Iterator`
- Возвращаемые элементы — `DynamicTest` или `DynamicContainer`
- Метод не может быть `static` (требуется экземпляр класса)

### Отличия от @Test:

| @Test                  | @TestFactory                  |
|:-----------------------|-------------------------------|
| Статические тесты      | Динамически генерируемые      |
| Компиляция             | Время выполнения              |
| Один метод = один тест | Один метод = множество тестов |

---

## Создание динамических тестов

### Базовый пример:

```java
    @TestFactory
    Stream<DynamicTest> dynamicTests() {
        return Stream.of(
            DynamicTest.dynamicTest("Test 1", () -> assertTrue(true)),
            DynamicTest.dynamicTest("Test 2", () -> assertEquals(2, 1 + 1)),
            DynamicTest.dynamicTest("Test 3", () -> assertNotNull("value"))
        );
    }
```

### Компоненты DynamicTest:

- `displayName` — Понятное имя теста
- `executable` — Executable с логикой теста (lambda)

---

## DynamicContainer

### Для группировки тестов:

```java
    @TestFactory
    Stream<DynamicContainer> dynamicContainers() {
        return Stream.of(
            DynamicContainer.dynamicContainer(
                "Group 1",
                Stream.of(
                    DynamicTest.dynamicTest("Test 1.1", () -> { }),
                    DynamicTest.dynamicTest("Test 1.2", () -> { })
                )
            ),
            DynamicContainer.dynamicContainer(
                "Group 2",
                Stream.of(
                    DynamicTest.dynamicTest("Test 2.1", () -> { }),
                    DynamicTest.dynamicTest("Test 2.2", () -> { })
                )
            )
        );
    }
```

### Иерархия тестов:

```text
     ┌─────────────────┐
     │  TestFactory    │
     └────────┬────────┘
              │
     ┌────────┴────────┐
     ▼                 ▼
   ┌─────────┐   ┌─────────┐
   │Container│   │Container│
   │  Group 1│   │  Group 2│
   └────┬────┘   └────┬────┘
        │             │
   ┌────┴────┐   ┌────┴────┐
   ▼         ▼   ▼         ▼
  Test     Test  Test    Test
  1.1      1.2    2.1     2.2
```
---

## Генерация тестов из данных

### Из коллекции:

```java
    @TestFactory
    Stream<DynamicTest> testFromCollection() {
        List<String> inputs = List.of("apple", "banana", "cherry");
        
        return inputs.stream()
            .map(input -> 
                DynamicTest.dynamicTest(
                    "Test: " + input,
                    () -> assertTrue(input.length() > 0)
                )
            );
    }
```

### Из файла:

```java
    @TestFactory
    Stream<DynamicTest> testFromFile() throws IOException {
        List<String> lines = Files.readAllLines(Paths.get("test-data.txt"));
        
        return lines.stream()
            .map(line -> 
                DynamicTest.dynamicTest(
                    "Test: " + line,
                    () -> assertNotNull(line)
                )
            );
    }
```

### Из базы данных:

```java
    @TestFactory
    Stream<DynamicTest> testFromDatabase() {
        List<TestData> testData = repository.findAll();
        
        return testData.stream()
            .map(data -> 
                DynamicTest.dynamicTest(
                    "Test: " + data.getId(),
                    () -> assertEquals(data.getExpected(), service.process(data.getInput()))
                )
            );
    }
```

---

## Условные динамические тесты

### Фильтрация тестов:

```java
    @TestFactory
    Stream<DynamicTest> conditionalTests() {
        return Stream.of(1, 2, 3, 4, 5)
            .filter(n -> n % 2 == 0)  // Только чётные
            .map(n -> 
                DynamicTest.dynamicTest(
                    "Even number: " + n,
                    () -> assertTrue(n % 2 == 0)
                )
            );
    }
```

### На основе окружения:

```java
    @TestFactory
    Stream<DynamicTest> environmentTests() {
        String os = System.getProperty("os.name");
        
        if (os.contains("Windows")) {
            return Stream.of(
                DynamicTest.dynamicTest("Windows Test", () -> { })
            );
        }
        
        return Stream.empty();  // Нет тестов для других ОС
    }
```

---

## Параметризованные динамические тесты

### Комбинация с данными:

```java
@TestFactory
Stream<DynamicTest> badExample() {
    return Stream.of(
            Arguments.of("add", 2, 3, 5),
            Arguments.of("subtract", 5, 3, 2),
            Arguments.of("multiply", 4, 3, 12)
    ).map(args ->
            DynamicTest.dynamicTest(
                    args.get(0) + " test",
                    () -> {
                        String operation = (String) args.get(0);
                        int a = (int) args.get(1);
                        int b = (int) args.get(2);
                        int expected = (int) args.get(3);

                        int result = switch (operation) {
                            case "add" -> calculator.add(a, b);
                            case "subtract" -> calculator.subtract(a, b);
                            case "multiply" -> calculator.multiply(a, b);
                            default -> throw new IllegalArgumentException();
                        };
                        assertEquals(expected, result);
                    }
            )
    );
}
```

---

## Когда использовать динамические тесты

| Сценарий                    | Решение                    |
|:----------------------------|----------------------------|
| Тесты из внешних данных     | Dynamic Tests из файла/БД  |
| Генерация на основе условий | Фильтрация в Stream        |
| Группировка по категориям   | DynamicContainer           |
| Тесты для каждой записи     | Stream.map() к DynamicTest |
| Сложная логика генерации    | Полная гибкость Java кода  |

---

## Сравнение с параметризованными тестами

| Критерий         | @ParameterizedTest             | @TestFactory                   |
|:-----------------|--------------------------------|--------------------------------|
| Когда создаются  | Компиляция                     | Выполнение                     |
| Источник данных  | Аннотации (@ValueSource, etc.) | Любой Java код                 |
| Возвращаемый тип | void                           | Stream/Collection<DynamicTest> |
| Гибкость         | Ограниченная                   | Полная                         |
| Вложенность      | Нет                            | Да (DynamicContainer)          |
| Сложность        | Проще                          | Сложнее                        |

---

## Best Practices для динамических тестов

### Делайте:

- Используйте для тестов из внешних источников
- Давайте понятные `displayName` тестам
- Фильтруйте ненужные тесты на этапе генерации
- Группируйте связанные тесты в `DynamicContainer`
- Кэшируйте данные если генерация дорогая

### Не делайте:

- Не используйте для простых статических тестов
- Не создавайте тысячи тестов (overhead)
- Не полагайтесь на порядок выполнения
- Не используйте сложную логику в `executable`
- Не смешивайте `@Test` и `@TestFactory` для одного сценария

---

## Ограничения динамических тестов

- Нельзя использовать `@BeforeEach` / `@AfterEach` внутри DynamicTest
- Нельзя использовать `@DisplayName` на DynamicTest
- Нельзя использовать `@Tag` на отдельных DynamicTest
- Отчёты могут быть менее детальными
- Отладка сложнее чем у обычных тестов

---

## Ключевые выводы

1. **@TestFactory для гибкости** — когда данные неизвестны на этапе компиляции
2. **DynamicTest = displayName + executable** — простая структура
3. **DynamicContainer для группировки** — иерархия тестов
4. **Stream API для генерации** — map, filter, collect
5. **Внешние источники данных** — файлы, БД, API
6. **Не замена параметризованным** — разные use cases
7. **Проще когда возможно** — `@ParameterizedTest` предпочтительнее для простых случаев

# 8. Mocking в тестах

> **Mocking** — создание имитаций зависимостей для изоляции тестируемого кода. 
> 
> Позволяет тестировать отдельные компоненты без реальных зависимостей (БД, внешние сервисы, etc.).

---

## Интеграция с Mockito

Mockito — популярный фреймворк для создания mock объектов в Java тестах.

### Maven зависимость:

```xml
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.0.0</version>
        <scope>test</scope>
    </dependency>
```

### Gradle зависимость:

```groovy
    testImplementation 'org.mockito:mockito-junit-jupiter:5.0.0'
```

### Подключение расширения:

```java
    @ExtendWith(MockitoExtension.class)
    class MyTest { }
```

---

## Аннотации Mockito

### @Mock

- Создаёт mock объект указанного класса
- Все методы ДО настройки поведения возвращают default значения (null, 0, false)
- Не требует реальной реализации

```java
@ExtendWith(MockitoExtension.class)
class MyTest { }

  @Mock
  UserService userService;

  @Mock
  OrderRepository orderRepository;
```

---

### Answers в Mockito

> Mockito предоставляет **9 встроенных стратегий** ответов по умолчанию через enum `Answers`.

### RETURNS_DEFAULTS
**Поведение:** значения по умолчанию (null, 0, false) — используется по умолчанию

```java
@Mock
UserService service;  // эквивалентно answer = RETURNS_DEFAULTS

service.getUser();     // → null
service.getCount();    // → 0
service.isActive();    // → false
```

### RETURNS_SMART_NULLS
   
Поведение: умные null — бросает информативное исключение при вызове метода на null

```java
@Mock(answer = Answers.RETURNS_SMART_NULLS)
UserService service;

User user = service.getUser();
user.getName();  // ❌ SmartNullPointerException:
// "You have a NullPointerException here because getUser() returned null.
//  This is how the mock was configured. See stack trace for details."
```

Плюс: вместо непонятного NPE получаете сообщение с подсказкой, какой mock и какой метод вернул null.

### RETURNS_MOCKS

Поведение: возвращает mock вместо null для объектов

```java
@Mock(answer = Answers.RETURNS_MOCKS)
UserService service;

User user = service.getUser();        // → mock User (не null)
Address address = user.getAddress();  // → mock Address (не null)
String city = address.getCity();      // → null (для примитивов всё равно null)

// Каскад моков — не упадёт
service.getUser().getAddress().getCity(); // вернёт null
```

### RETURNS_DEEP_STUBS
   
Поведение: глубокие стабы — позволяет настраивать цепочки вызовов

```java
@Mock(answer = Answers.RETURNS_DEEP_STUBS)
UserService service;

// Можно настраивать цепочки вызовов одной строкой
when(service.getUser().getAddress().getCity()).thenReturn("Moscow");

// Без DEEP_STUBS пришлось бы писать:
User user = mock(User.class);
Address address = mock(Address.class);
when(user.getAddress()).thenReturn(address);
when(address.getCity()).thenReturn("Moscow");
when(service.getUser()).thenReturn(user);

// Теперь работает:
String city = service.getUser().getAddress().getCity(); // "Moscow"
```

Важно: лучше использовать для value objects, не для сервисов!

### RETURNS_SELF
   
Поведение: возвращает сам mock (для builder-паттерна)

```java
@Mock(answer = Answers.RETURNS_SELF)
UserBuilder builder;

// Нужно настроить ТОЛЬКО конечный метод build()
when(builder.build()).thenReturn(new User("John", 30));

// Все методы withXxx() автоматически возвращают сам mock!
User user = builder
        .withName("John")            // → возвращает builder (автоматически)
        .withAge(30)                 // → возвращает builder (автоматически)
        .withEmail("john@mail.com")  // → возвращает builder (автоматически)
        .build();                    // → возвращает настроенный User
```

### CALLS_REAL_METHODS
   
Поведение: вызывает реальные методы (частичный mock)

```java
@Mock(answer = Answers.CALLS_REAL_METHODS)
Calculator calculator;

// Реальные методы вызываются, если не настроены
int sum = calculator.add(2, 3);  // реальное сложение → 5

// Можно переопределить конкретный метод
when(calculator.multiply(2, 3)).thenReturn(100);
int product = calculator.multiply(2, 3);  // → 100 (mock), а не 6
```

Используется для: спай (spy) альтернатива, но через аннотацию `@Spy` проще.

### RETURNS_DEEP_STUBS_AS_REAL
   
Поведение: комбинация глубоких стабов и реальных методов — редко используется на практике

### RETURNS_DEFAULTS_FOR_SERIALIZATION
   
Поведение: для сериализуемых моков — редко используется

### THROWS_EXCEPTION
   
Поведение: всегда бросает исключение

```java
@Mock(answer = Answers.THROWS_EXCEPTION)
UserService service;

service.getUser();  // ❌ бросает Exception
service.save(null); // ❌ бросает Exception
```

Используется: редко, для тестирования обработки ошибок.

---

### @Spy

- Создаёт spy объект (частичный mock)
- Реальные методы вызываются по умолчанию
- Можно переопределить конкретные методы

```java
  @Spy
  List<String> list = new ArrayList<>();

  // Реальный метод
  list.add("real");

  // Mock метод
  doReturn("mocked").when(list).get(0);
```

Расширенный пример:

```java
@ExtendWith(MockitoExtension.class)
class SpyDetailedTest {

    // 1. Spy на коллекции
    @Spy
    List<String> list = new ArrayList<>();
    
    // 2. Spy на реальном сервисе
    @Spy
    UserService userService = new UserService();
    
    // 3. Spy с внедрением зависимостей
    @Spy
    @InjectMocks
    OrderService orderService;
    
    @Mock
    PaymentGateway paymentGateway;  // будет внедрён в spy
    
    @Test
    void testListSpy() {
        // РЕАЛЬНЫЙ вызов — элемент добавляется
        list.add("apple");
        list.add("banana");
        
        assertEquals(2, list.size());      // реальное состояние
        assertEquals("apple", list.get(0)); // реальное значение
        
        // ПЕРЕОПРЕДЕЛЯЕМ поведение get(1)
        doReturn("mocked banana").when(list).get(1);
        
        assertEquals("apple", list.get(0));    // реальный метод
        assertEquals("mocked banana", list.get(1)); // переопределённый
        
        // Остальные методы всё ещё реальные
        assertEquals(2, list.size()); // size() не переопределяли → реальный
    }
    
    @Test
    void testServiceSpy() {
        // РЕАЛЬНЫЙ вызов
        User user = userService.createUser("John", "john@mail.com");
        assertNotNull(user);
        assertEquals("John", user.getName());
        
        // Проверяем, что реальный метод был вызван
        verify(userService).createUser("John", "john@mail.com");
        
        // ПЕРЕОПРЕДЕЛЯЕМ поведение
        doReturn(true).when(userService).isAdmin("john@mail.com");
        doReturn(false).when(userService).isAdmin("unknown@mail.com");
        
        assertTrue(userService.isAdmin("john@mail.com"));
        assertFalse(userService.isAdmin("unknown@mail.com"));
    }
    
    @Test
    void testPartialMocking() {
        // Spy позволяет переопределить ТОЛЬКО нужные методы
        // Остальные работают по-настоящему
        
        // Переопределяем метод getUserCount
        doReturn(100).when(userService).getUserCount();
        
        // Результат: mock
        assertEquals(100, userService.getUserCount());
        
        // А этот метод реальный
        User user = userService.createUser("Jane", "jane@mail.com");
        assertNotNull(user);  // реальный метод вернул настоящего пользователя
    }
}
```

### Важные особенности @Spy

#### 1. Инициализация обязательна

```java
    @Spy
    List<String> list = new ArrayList<>();  // ← нужно явно создать объект

    @Spy
    List<String> list2;  // ❌ Будет null — Mockito не создаёт объект за вас
```

#### 2. Использование doReturn() вместо when()

```java
// ❌ Так НЕ работает с spу (вызовется реальный метод!)
when(list.get(0)).thenReturn("mocked");  // выполнится реальный get(0)!

// ✅ Правильно — используйте doReturn
doReturn("mocked").when(list).get(0);  // реальный get(0) НЕ вызывается
```
Почему? `when(list.get(0)).thenReturn(...)` сначала вызывает реальный `list.get(0)`, что может бросить исключение (если список пуст). 

`doReturn` обходит эту проблему.

#### 3. Spy с @InjectMocks

```java
    @Spy
    @InjectMocks
    OrderService orderService;  // Mockito создаст spy и внедрит моки

    @Mock
    PaymentGateway paymentGateway;  // будет внедрён в orderService

    @Test
    void testWithInjection() {
        // paymentGateway — mock, orderService — spy
        doReturn(true).when(paymentGateway).charge(anyDouble());

        boolean result = orderService.processOrder(new Order(100));
        assertTrue(result);
    }
```

| Сценарий                                                       | Рекомендация               | Пример                                                                                                   |
|----------------------------------------------------------------|----------------------------|----------------------------------------------------------------------------------------------------------|
| Нужен полный контроль над поведением                           | `@Mock`                    | Все методы возвращают то, что вы настроили. Реальная реализация не вызывается.                           |
| Нужно использовать реальную реализацию большинства методов     | `@Spy`                     | Большинство методов работают по-настоящему, только отдельные переопределены.                             |
| Тестируем класс с побочными эффектами (запись в БД, вызов API) | `@Mock`                    | Чтобы реальная запись в БД или вызов API не происходили во время теста.                                  |
| Нужно проверить, что реальный метод был вызван                 | `@Spy` + `verify()`        | `verify(spy).realMethod()` — проверяем, что реальная логика действительно выполнилась.                   |
| Класс со сложной логикой инициализации                         | `@Spy` с реальным объектом | Создаёте реальный объект (со всей сложной инициализацией), а затем переопределяете только нужные методы. |
| Нужно изолировать тестируемый класс от зависимостей            | `@Mock` для зависимостей   | Все зависимости тестируемого класса заменяются моками.                                                   |
| Нужно сохранить состояние между вызовами                       | `@Spy`                     | Реальный объект хранит состояние (например, список, счётчик, кэш).                                       |
| Тестирование кэширования или логирования                       | `@Spy`                     | Нужна реальная логика, но некоторые методы (например, внешний вызов) нужно переопределить.               |
| Тестирование методов с `void` и побочными эффектами            | `@Mock` или `doNothing()`  | Легче контролировать, что именно происходит при вызове void-методов.                                     |
| Рефакторинг легаси-кода с тяжёлыми зависимостями               | `@Spy`                     | Создаёте реальный объект, переопределяете только тяжёлые зависимости, остальное работает как есть.       |

---

### @InjectMocks

- Автоматически внедряет `@Mock` / `@Spy` поля в объект
- По типу или по имени поля
- Упрощает настройку тестов

```java
  @InjectMocks
  OrderService orderService;

  @Mock
  OrderRepository orderRepository;

  @Mock
  PaymentService paymentService;
```

---

### @Captor

- Создаёт ArgumentCaptor для перехвата аргументов
- Полезно для проверки переданных значений

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository repository;

    @InjectMocks
    UserService service;

    @Captor
    ArgumentCaptor<User> userCaptor;  // 1. Создаём "ловушку" для User

    @Test
    void testSaveUser() {
        // 2. Вызываем тестируемый метод
        service.saveUser("John", "john@mail.com");
        // Внутри метода service.saveUser() вызывает repository.save(user)

        // 3. "Ловим" аргумент, переданный в mock
        verify(repository).save(userCaptor.capture());
        //                  ↑ метод, который вызывали   
        //                                     ↑ ловим аргумент

        // 4. Получаем пойманный объект и проверяем
        User capturedUser = userCaptor.getValue();
        assertEquals("John", capturedUser.getName());
        assertEquals("john@mail.com", capturedUser.getEmail());
    }
}
```

### Когда использовать @Captor

| Сценарий                                                 | Пример                                                              | Почему нужен @Captor                                                                                     |
|----------------------------------------------------------|---------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Объект создаётся внутри тестируемого метода              | `new User(name, email)` — вы не можете получить его извне           | Только captor может "поймать" объект после его создания внутри метода                                    |
| Нужно проверить поля объекта                             | проверить `user.getName()`, `user.getEmail()`                       | Обычный `verify()` проверяет только равенство объектов, а не их содержимое                               |
| Несколько вызовов с разными аргументами                  | проверить, что при первом вызове одно значение, при втором — другое | `getAllValues()` позволяет получить все переданные аргументы и проверить порядок                         |
| Проверка аргументов при `void` методах                   | `void sendEmail(Email email)` — нужно проверить, что внутри email   | `void` методы ничего не возвращают, captor — единственный способ проверить переданные данные             |
| Проверка "ленивых" инициализаций                         | объект создаётся только при определённых условиях                   | Captor может проверить, что объект создан и передан с правильными полями, только когда условие выполнено |
| Проверка аргументов, которые модифицируются после вызова | объект изменяется после передачи в mock                             | Captor сохраняет состояние на момент вызова, даже если объект потом изменится                            |
| Проверка части аргументов при сложных вызовах            | `mock.method(obj1, obj2, obj3)` — нужно проверить только один       | Можно ловить только нужные аргументы, остальные проверять через `any()` или конкретные значения          |

### Важные моменты

| Правило                                                                | Пояснение                                                                     | Пример                                                                                      |
|------------------------------------------------------------------------|-------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| `capture()` вызывается **внутри** `verify()`                           | Аргумент "ловится" в момент проверки вызова                                   | `verify(repository).save(userCaptor.capture())`                                             |
| `getValue()` — для одного вызова                                       | Возвращает последний пойманный аргумент (один объект)                         | `User user = userCaptor.getValue()`                                                         |
| `getAllValues()` — для нескольких вызовов                              | Возвращает список **всех** пойманных аргументов за все вызовы                 | `List<User> users = userCaptor.getAllValues()`                                              |
| Обязательно использовать с `verify()`                                  | Captor не работает сам по себе, только в связке с verify                      | `verify(mock, times(2)).method(captor.capture())`                                           |
| Можно комбинировать с `any()`                                          | Ловить одни аргументы, остальные игнорировать                                 | `verify(mock).method(captor.capture(), anyString(), eq(5))`                                 |
| `capture()` можно использовать в `verify` с разными matcher'ами        | Работает с `any()`, `eq()`, `argThat()` и другими                             | `verify(mock).method(captor.capture(), argThat(x -> x > 0))`                                |
| Порядок вызовов важен                                                  | `getAllValues()` возвращает аргументы в том же порядке, в котором были вызовы | первый вызов → первый в списке, второй вызов → второй в списке                              |
| Не путать `getValue()` и `getAllValues()`                              | `getValue()` вернёт последний элемент из `getAllValues()`                     | при 3 вызовах `getValue()` = `getAllValues().get(2)`                                        |
| Можно создавать несколько captor'ов для разных типов                   | Каждый captor ловит свой тип аргументов                                       | `@Captor ArgumentCaptor<User> userCaptor;`<br>`@Captor ArgumentCaptor<String> emailCaptor;` |
| При использовании с `@Captor` аннотация инициализируется автоматически | Не нужно вызывать `ArgumentCaptor.forClass()` вручную                         | `@ExtendWith(MockitoExtension.class)` делает это за вас                                     |
---

## Создание mock объектов

### Через аннотации:

```java
    @ExtendWith(MockitoExtension.class)
    class MyTest {
    
        @Mock
        Repository repository;
    }
```

### Через Mockito.mock():

```java
    Repository repository = Mockito.mock(Repository.class);
```

### Через mockStatic (для статических методов):

```java
    try (MockedStatic<Utils> utils = Mockito.mockStatic(Utils.class)) {
        utils.when(() -> Utils.getValue()).thenReturn("mocked");
        // Тесты
    }
```

### Через mockConstruction (для конструкторов):

> **mockConstruction** = инструмент для тестирования "плохо написанного" кода, 
> где объекты создаются напрямую через new внутри методов. 

Он позволяет:

- Перехватывать создание объектов через конструктор
- Настраивать поведение mock в зависимости от аргументов конструктора
- Проверять, сколько раз и с какими аргументами был вызван конструктор
- Проверять вызовы методов на созданных объектах

Когда использовать:

- Рефакторинг легаси-кода, где объекты создаются через new
- Нет возможности внедрить зависимости через конструктор или setter
- Нужно протестировать код, который сам создаёт свои зависимости

```java
// Реальный класс, который создаётся через new
class PaymentGateway {
    
    private String apiKey;
    private int timeout;

    public PaymentGateway(String apiKey, int timeout) {
        this.apiKey = apiKey;
        this.timeout = timeout;
    }

    public boolean charge(double amount) {
        // реальный вызов платёжного API
        return httpClient.post("https://api.payment.com/charge", amount);
    }
}

// Тестируемый класс
class OrderService {
    public boolean placeOrder(double amount) {
        // Прямое создание объекта через new!
        PaymentGateway gateway = new PaymentGateway("secret-key", 5000);
        return gateway.charge(amount);
    }
}

// Тест с mockConstruction
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Test
    void testPlaceOrder() {
        // 1. Создаём mock для конструктора PaymentGateway
        try (MockedConstruction<PaymentGateway> mocked = mockConstruction(PaymentGateway.class,
                (mock, context) -> {
                    // context содержит аргументы конструктора
                    String apiKey = (String) context.arguments().get(0);
                    int timeout = (int) context.arguments().get(1);

                    // Настраиваем поведение mock
                    when(mock.charge(anyDouble())).thenReturn(true);
                }
        )) {
            // 2. Создаём реальный сервис
            OrderService service = new OrderService();

            // 3. Вызываем метод — внутри него будет new PaymentGateway(...)
            boolean result = service.placeOrder(100.0);

            // 4. Проверяем результат
            assertTrue(result);

            // 5. Проверяем, что mock был вызван
            PaymentGateway mockGateway = mocked.constructed().get(0);
            verify(mockGateway).charge(100.0);

            // 6. Проверяем аргументы конструктора
            assertEquals("secret-key", mocked.constructed().get(0).apiKey);
        }
    }
}
```

---

## Настройка поведения mock

### when().thenReturn():

```java
    when(repository.findById(1L)).thenReturn(Optional.of(user));
    when(repository.findAll()).thenReturn(List.of(user1, user2));
```

### when().thenThrow():

```java
    when(repository.findById(1L)).thenThrow(new NotFoundException());
```

### when().thenAnswer():

```java
    when(repository.findById(anyLong())).thenAnswer(invocation -> {
        Long id = invocation.getArgument(0);
        return Optional.of(new User(id, "Name"));
    });
```

### doReturn().when() (для spy и void методов):

```java
    doReturn("value").when(spy).getMethod();
    doNothing().when(spy).voidMethod();
    doThrow(Exception.class).when(spy).voidMethod();
```

---

## Verification (Проверка вызовов)

### Базовая проверка:

```java
    verify(repository).save(user);
    verify(repository, times(1)).save(user);
```

### Проверка количества вызовов:

- `times(n)` — Ровно n раз
- `never()` — Ни разу
- `atLeast(n)` — Минимум n раз
- `atMost(n)` — Максимум n раз
- `only()` — Только этот метод вызывался

```java
  verify(repository, times(2)).save(any());
  verify(repository, never()).delete(any());
  verify(repository, atLeastOnce()).findAll();
```

---

### Проверка с аргументами:

- `eq(value)` — Точное совпадение
- `any()` — Любое значение
- `anyString()` — Любая строка
- `anyLong()` — Любое long
- `anyList()` — Любой список
- `contains(value)` — Содержит значение
- `startsWith(prefix)` — Начинается с prefix
- `endsWith(suffix)` — Заканчивается на suffix
- `argThat(predicate)` - это кастомный матчер (матчер = сопоставитель), который позволяет написать свою логику проверки аргумента, 
  когда стандартных матчеров (`eq()`, `any()`, `startsWith()` и т.д.) недостаточно.

```java
  verify(repository).save(eq(user));
  verify(repository).findByName(anyString());
```

Синтаксис `argThat()`:

```java
// Вариант 1: Лямбда-выражение
verify(mock).method(argThat(argument -> условие));

// Вариант 2: Ссылка на метод
verify(mock).method(argThat(this::проверка));

// Вариант 3: С описанием ошибки (лучше)
verify(mock).method(argThat(argument -> условие, "описание"));
```

---

### Проверка порядка вызовов:

> **InOrder** — это инструмент Mockito, который позволяет проверить последовательность (порядок) вызовов методов на mock-объектах. 
>
> Он гарантирует, что методы были вызваны в строго определённом порядке.

При создании inOrder Mockito начинает отслеживать и запоминать все вызовы на переданных моках. 

Каждый последующий вызов `inOrder.verify()` проверяет, что следующий по порядку вызов соответствует ожидаемому.

```java
// 1. Создаём моки
Repository repository = mock(Repository.class);
Service service = mock(Service.class);

// 2. Вызываем методы (в каком-то порядке)
repository.save(user);      // вызов #1
service.notify(user);       // вызов #2
service.log("done");        // вызов #3
repository.flush();         // вызов #4

// 3. Создаём InOrder для этих моков
InOrder inOrder = inOrder(repository, service);

// 4. Проверяем порядок
inOrder.verify(repository).save(user);   // проверяем, что вызов #1 был save()
inOrder.verify(service).notify(user);    // проверяем, что вызов #2 был notify()
inOrder.verify(service).log("done");     // проверяем, что вызов #3 был log()
inOrder.verify(repository).flush();      // проверяем, что вызов #4 был flush()
```

---

## Timeout verification

> **Timeout verification** — это проверка, что метод на mock-объекте был вызван **в течение указанного времени**. 
> 
> Если вызов не произошёл за отведённое время, тест падает.

### Проверка с таймаутом:

```java
@Test
void testAsyncTimeoutMechanism() {
    // T0: Запуск асинхронной операции
    // Метод отправляет задачу в пул потоков и СРАЗУ возвращает управление.
    // Фоновая задача начинает выполняться ПАРАЛЛЕЛЬНО.
    asyncService.saveUserAsync("John");

    // --- ПРОМЕЖУТОК (T0 -> T1) ---
    // Здесь выполняется код теста (если он есть после вызова).
    // Обычно это занимает микросекунды.
    // Важно: время, потраченное здесь, НЕ входит в таймаут verify(),
    // но фоновая задача в это время УЖЕ работает!

    // T1: ТОЧКА ОТСЧЁТА ТАЙМАУТА
    // Поток теста доходит до этой строки. Mockito фиксирует время старта.
    // Начинается активное ожидание (polling) вызова метода на моке.
    verify(repository, timeout(1000)).save(argThat(u -> u.getName().equals("John")));

    // Если save() не будет вызван в интервале [текущий момент, текущий момент + 1000мс],
    // Mockito выбросит AssertionError.
}
```

---

## Best Practices для mocking

### Делайте:

- Mock только внешние зависимости
- Используйте `@InjectMocks` для внедрения
- Проверяйте важные вызовы через `verify()`
- Используйте понятные имена для mock полей
- Тестируйте реальную логику, а не mock поведение

### Не делайте:

- Не мокируйте классы, которые создаёте сами
- Не проверяйте каждый вызов mock (over-specification)
- Не используйте mock для тестирования интеграции
- Не мокируйте value objects (DTO, Entity)
- Не создавайте сложные expectations на mock

---

## Альтернативы Mockito

### EasyMock:

```java
    Repository mock = EasyMock.createMock(Repository.class);
    EasyMock.expect(mock.findById(1L)).andReturn(user);
    EasyMock.replay(mock);
```

### JMockit:

```java
    @Mocked
    Repository repository;
    
    new Expectations() {{
        repository.findById(1L); result = user;
    }};
```

---

## Распространённые ошибки

| Ошибка                          | Решение                              |
|:--------------------------------|--------------------------------------|
| NullPointerException при verify | Проверить что mock создан (@Mock)    |
| verify не работает              | Убедиться что метод был вызван       |
| when() возвращает null          | Проверить тип возвращаемого значения |
| Static mock не работает         | Использовать mockStatic()            |
| Конструктор не mockится         | Использовать mockConstruction()      |

---

## Сводная таблица аннотаций Mockito

| Аннотация        | Назначение              | Требует Extension |
|:-----------------|-------------------------|-------------------|
| @Mock            | Создание mock объекта   | Да (@ExtendWith)  |
| @Spy             | Создание spy объекта    | Да (@ExtendWith)  |
| @InjectMocks     | Внедрение mock в объект | Да (@ExtendWith)  |
| @Captor          | Создание ArgumentCaptor | Да (@ExtendWith)  |
| @MockitoSettings | Настройка Mockito       | Да (@ExtendWith)  |

---

## Ключевые выводы

1. **Mockito — стандарт для mocking** — интеграция с JUnit 5 через extension
2. **@Mock для зависимостей** — изоляция тестируемого кода
3. **@InjectMocks упрощает настройку** — автоматическое внедрение
4. **verify проверяет вызовы** — но не переусердствуйте
5. **ArgumentCaptor для сложных проверок** — перехват аргументов
6. **Spy для частичного mocking** — когда нужна реальная логика
7. **Не mock всё подряд** — только внешние зависимости

---

## Интеграция Mockito со Spring

> При интеграции с Spring Boot важно различать unit-тесты (чистый Mockito) и integration-тесты (Spring Context + mocks). 
> 
> Основные нюансы: выбор аннотаций, жизненный цикл контекста и правильное внедрение зависимостей.

---

### @MockBean вместо @Mock в Spring-тестах

- `@MockBean` — Spring-аннотация, регистрирует mock в ApplicationContext
- Заменяет существующий бин того же типа (если есть)
- Требует загрузки контекста (`@SpringBootTest`, `@WebMvcTest` и др.)
- `@Mock` — чистый Mockito, не знает о Spring, быстрее

```java
  // ❌ Не работает: @Mock не внедряется в Spring-контекст
  @SpringBootTest
  class OrderServiceTest {
        
        @Mock
        PaymentGateway gateway;  // не будет внедрён в @Autowired сервисы
    
        @Autowired 
        OrderService service;    // получит реальный PaymentGateway!
  }

  // ✅ Правильно: @MockBean регистрирует mock в контексте
  @SpringBootTest
  class OrderServiceTest {
  
        @MockBean
        PaymentGateway gateway;  // заменит реальный бин

        @Autowired
        OrderService service;    // получит mock-версию
  }
```

---

### Выбор тестовой аннотации Spring

- `@SpringBootTest` — полная загрузка контекста (медленно, для e2e)
- `@WebMvcTest` — только MVC-слой + `@MockBean` для сервисов
- `@DataJpaTest` — только JPA-репозитории + in-memory БД
- `@JsonTest`, `@RestClientTest` — узкоспециализированные срезы

```java
  // Пример: тест контроллера с моком сервиса
  @WebMvcTest(OrderController.class)
  class OrderControllerTest {

        @Autowired
        MockMvc mockMvc;  // Spring MVC тестовый клиент
        
        @MockBean
        OrderService orderService;  // mock для внедрения в контроллер
        
        @Test
        void createOrder_shouldReturn201() throws Exception {
            when(orderService.create(any()))
                .thenReturn(new Order(1L));
            
            mockMvc.perform(post("/orders")
                    .contentType(APPLICATION_JSON)
                    .content("{\"name\":\"Test\"}"))
                .andExpect(status().isCreated());
        }
  }
```
---

### MockitoExtension + Spring: можно ли вместе?

- Технически можно: `@SpringBootTest` + `@ExtendWith(MockitoExtension.class)`
- Но избыточно: Spring сам инициализирует `@MockBean`
- MockitoExtension полезен только для `@Mock` / `@Spy` полей вне контекста

```java
  // Допустимо, но редко нужно
  @SpringBootTest
  @ExtendWith(MockitoExtension.class)
  class HybridTest {

        @MockBean
        ExternalApi externalApi;  // в Spring контексте
        
        @Mock
        Helper helper;  // чистый mock, не в контексте
        
        @InjectMocks
        LocalValidator validator;  // Mockito внедрит @Mock в локальный объект
  }
```

---

### Частые ошибки и решения

- Ошибка: `@MockBean` не заменяет бин → проверить тип и qualifier
- Ошибка: медленные тесты → использовать `@WebMvcTest` вместо `@SpringBootTest`
- Ошибка: утечка состояния между тестами → `@DirtiesContext` или перезагрузка контекста
- Ошибка: mock не сбрасывается → Spring сбрасывает `@MockBean` автоматически между тестами

```java
  // Пример: сброс mock между тестами в @SpringBootTest
  @SpringBootTest
  class StatefulTest {

        @MockBean
        CounterService counter;
        
        @BeforeEach
        void resetMock() {
            // Явный сброс, если тесты меняют состояние mock
            reset(counter);
        }
  }
```

## @DirtiesContext

> @DirtiesContext помечает Spring-контекст как "грязный" и требует его перезагрузки перед следующим тестом.

### Режимы ClassMode (уровень класса)

- `AFTER_EACH_TEST_METHOD` — контекст перезагружается ПОСЛЕ КАЖДОГО тестового метода
    - Самый строгий режим, гарантирует чистый контекст для каждого теста
    - Медленный, так как контекст пересоздаётся после каждого метода

- `AFTER_CLASS` — контекст перезагружается ПОСЛЕ ВСЕХ тестов класса
    - Используется, если класс в целом "испортил" контекст
    - Влияет только на следующие тесты в других классах

- `BEFORE_EACH_TEST_METHOD` — контекст перезагружается ПЕРЕД КАЖДЫМ тестовым методом
    - Гарантирует чистый контекст перед каждым тестом
    - Редко используется, обычно достаточно AFTER_EACH_TEST_METHOD

- `BEFORE_CLASS` — контекст перезагружается ПЕРЕД выполнением тестов класса
    - Полезно, если класс требует специфического состояния контекста

### Режимы MethodMode (уровень метода)

- `AFTER_EACH_TEST_METHOD` — перезагрузка после метода (аналогично ClassMode)
- `AFTER_METHOD` — перезагрузка после метода (устаревший синоним)

### Когда использовать

- Тесты модифицируют разделяемые ресурсы (БД, кэш, статические состояния)
- Тесты меняют конфигурацию (системные свойства, переменные окружения)
- Тесты используют разные профили или настройки
- Один тест "портит" контекст, что влияет на другие тесты

### Важные особенности

- Перезагрузка контекста — дорогая операция (может замедлить тесты)
- Приоритет: `MethodMode` выше, чем `ClassMode`
- Можно комбинировать на классе и на методе одновременно
- Автоматически сбрасывает все `@MockBean` и `@SpyBean`

### Best Practices: Spring + Mockito

Делайте:

- Используйте `@MockBean` только в integration-тестах
- Для unit-тестов сервисов — чистый Mockito без Spring
- Выбирайте минимальный срез контекста (`@WebMvcTest`, `@DataJpaTest`)
- Явно сбрасывайте mock, если тесты зависят от порядка
- Документируйте, почему тест требует Spring-контекста

Не делайте:

- Не используйте `@SpringBootTest` для простых unit-тестов
- Не смешивайте `@Mock` и `@MockBean` для одного типа в одном тесте
- Не полагайтесь на порядок выполнения тестов с состоянием
- Не мокайте слои, которые тестируете (контроллер → мок сервиса, а не контроллера)
- Не забывайте про `@DirtiesContext` при изменении контекста

---

### Сводная таблица: аннотации mocking в Spring

| Аннотация      | Фреймворк | Контекст | Скорость | Когда использовать          |
|----------------|-----------|----------|----------|-----------------------------|
| @Mock          | Mockito   | Нет      | Быстро   | Unit-тесты, чистая логика   |
| @MockBean      | Spring    | Да       | Медленно | Integration-тесты, слои     |
| @SpyBean       | Spring    | Да       | Медленно | Частичный мок в контексте   |
| @InjectMocks   | Mockito   | Нет      | Быстро   | Локальное внедрение в тесте |

---

### Ключевые выводы

1. **@MockBean для Spring-контекста** — заменяет реальные бины, требует загрузки контекста
2. **@Mock для unit-тестов** — быстрее, изолированнее, не зависит от Spring
3. **Выбирайте минимальный срез** — @WebMvcTest вместо @SpringBootTest, если возможно
4. **Spring сбрасывает @MockBean** — но явный reset() делает тесты предсказуемее
5. **Не смешивайте аннотации** — один тип зависимости → один тип mock-аннотации
6. **Документируйте интеграционные тесты** — почему требуется контекст, что мокается
7. **Тестируйте слои изолированно** — контроллер → мок сервиса, сервис → мок репозитория

## Настройки Mockito

### Глобальная конфигурация (mockito-extensions/org.mockito.plugins.MockMaker):

> Включает mocking финальных классов и методов (по умолчанию недоступно).

Файл должен находиться в `src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker`:

Файл org.mockito.plugins.MockMaker должен содержать всего одну строку:

```text
mock-maker-inline
```

Через Maven (альтернатива)

Если не хотите создавать файл, можно использовать зависимость mockito-inline:

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>5.2.0</version>
    <scope>test</scope>
</dependency>
```

Зависимость автоматически включает mock-maker-inline.

---

### Файл junit-platform.properties:

```properties
    junit.jupiter.extensions.autodetection.enabled = true
```

Автоматическое обнаружение расширений включая MockitoExtension.

---

### Настройка через @MockitoSettings:

```java
    @MockitoSettings(strictness = Strictness.LENIENT)
    class MyTest {
        
        @Mock
        Repository repository;
    }
```

### Уровни strictness:

- `STRICT_STUBS` — Строгая проверка неиспользованных stub (по умолчанию)
- `LENIENT` — Разрешает неиспользованные stub
- `WARN` — Предупреждает о неиспользованных stub

---

### Отключение MockitoExtension для отдельных тестов:

```java
// Вариант 1: Без MockitoExtension (ручная инициализация)
class ManualMockTest {

    Repository repository;  // не @Mock

    @BeforeEach
    void setUp() {
        repository = Mockito.mock(Repository.class);
    }

    @Test
    void testWithMock() {
        when(repository.findById(1)).thenReturn(new User());
        // используем mock
    }

    @Test
    void testWithoutMock() {
        // обычный тест без mock
    }
}

// Вариант 2: С MockitoExtension, но с отключением строгости для отдельных тестов
@ExtendWith(MockitoExtension.class)
class MockitoSettingsTest {

    @Mock
    Repository repository;

    @Test
    void testWithMock() {
        when(repository.findById(1)).thenReturn(new User());
        // используем mock
    }

    @Test
    @MockitoSettings(strictness = Strictness.LENIENT)
    void testWithUnusedMock() {
        // mock есть, но не используется
        // LENIENT режим позволяет это
    }
}
```

---

### Конфигурация для параллельных тестов:

```java
    @ExtendWith(MockitoExtension.class)
    @Execution(ExecutionMode.CONCURRENT)
    class ParallelMockTest {
        // Каждый тест получает свой экземпляр mock
    }
```

---

### Интеграция со Spring:

```java
    @SpringBootTest
    @ExtendWith(MockitoExtension.class)
    class SpringIntegrationTest {
        @MockBean  // Spring аннотация
        Repository repository;
        
        @Autowired
        Service service;
    }
```

### Отличия @Mock vs @MockBean:

| @Mock             | @MockBean                      |
|:------------------|--------------------------------|
| Mockito аннотация | Spring аннотация               |
| Простой mock      | Mock в Spring контексте        |
| Быстрее           | Медленнее (создание контекста) |
| Для unit тестов   | Для integration тестов         |

---

### Debug настройки:

```bash
    // Включить подробное логирование Mockito
    -Dmockito.logger=DEBUG
    
    // Показать неиспользованные stub
    -Dmockito.strictness=STRICT_STUBS
```

---

### Распространённые конфигурационные проблемы:

| Проблема                     | Решение                                     |
|:-----------------------------|---------------------------------------------|
| MockitoExtension не работает | Проверить зависимость mockito-junit-jupiter |
| Финальные классы не mockятся | Добавить mock-maker-inline                  |
| @InjectMocks не внедряет     | Проверить конструктор или setter            |
| Конфликт с Spring            | Использовать @MockBean вместо @Mock         |

# 9. Организация тестов

> Эффективная организация тестов упрощает поддержку, навигацию и выполнение. 

---

## Структура тестового проекта

- Стандартная структура (Maven/Gradle):
  
  `src/test/java/` — исходный код тестов
  
  `src/test/resources/` — конфиги, данные, SQL-скрипты
- Пакеты тестов зеркалят структуру основного кода:
  
  main: `com.example.service.UserService.java`
  
  test: `com.example.service.UserServiceTest.java`
- Группировка по функциональности или слою:
  
  `/service`, `/repository`, `/integration`, `/e2e`

---

## Именование тестовых классов и методов

- Классы: ИмяКласса + "Test" (UserServiceTest)
- Методы: описывают сценарий, а не метод
- Форматы именования методов:
    - methodName_scenario_expectedResult
    - should_returnX_when_conditionY
    - Given_Condition_When_Action_Then_Result

---

## Паттерны написания тестов (AAA / Given-When-Then)

- AAA (Arrange-Act-Assert):
  
  `Arrange` — подготовка данных, моков
  
  `Act` — вызов тестируемого метода
  
  `Assert` — проверка результата

- `Given-When-Then` — семантический вариант `AAA`
- Используйте комментарии или пустые строки для разделения фаз

```java
  // Пример: AAA паттерн
  @Test
  void calculateTotal_shouldSumPrices() {
        // Arrange
        List<Item> items = List.of(
        new Item("A", 10),
        new Item("B", 20)
        );
        Calculator calc = new Calculator();

        // Act
        BigDecimal total = calc.calculateTotal(items);
        
        // Assert
        assertEquals(new BigDecimal("30"), total);
  }

  // Пример: Given-When-Then с комментариями
  @Test
  void withdraw_shouldDecreaseBalance_whenSufficientFunds() {
        // Given
        Account acc = new Account(100);

        // When
        acc.withdraw(30);
        
        // Then
        assertEquals(70, acc.getBalance());
  }
```

## Тестовые данные и фикстуры

- Храните данные в `src/test/resources` (JSON, SQL, CSV)
- Используйте `@TempDir` для временных файлов
- Для сложных объектов — Builder-паттерн или тестовые фабрики
- Параметризованные тесты для вариаций данных

---


## Best Practices

Делайте:

- Называйте тесты как пользовательские сценарии
- Держите тесты независимыми и изолированными
- Используйте комментарии для разделения AAA-фаз
- Выносите общую логику в хелпер-методы
- Храните тестовые данные отдельно от кода

Не делайте:

- Не полагайтесь на порядок выполнения тестов
- Не используйте статическое состояние между тестами
- Не дублируйте код подготовки в каждом тесте
- Не тестируйте несколько сценариев в одном методе
- Не храните "магические" значения без пояснений

---

# 10. Запуск и отчётность

> Запуск тестов и сбор отчётов — критическая часть CI/CD-процесса.

---

## Запуск из IDE

### IntelliJ IDEA

- Правый клик по тесту/классу/пакету → `Run 'TestName'`
- Все тесты модуля: правый клик на `src/test/java` → `Run 'All Tests'`
- Горячие клавиши: `Ctrl+Shift+F10` (запуск), `Ctrl+Shift+F11` (покрыть тестами)
- Панель Run показывает прогресс, ошибки, время выполнения

### Eclipse

- `Правый клик` → `Run As` → `JUnit Test`
- Все тесты проекта: `Run` → `Run Configurations` → `JUnit`
- Горячие клавиши: `Alt+Shift+X`, `T` (запуск теста)
- `View` → `JUnit` для детального отчёта

---

## Запуск из командной строки

### Maven (surefire-plugin по умолчанию)

```text
    # Запустить все тесты
    mvn test
    
    # Запустить конкретный тест-класс
    mvn test -Dtest=UserServiceTest
    
    # Запустить конкретный метод
    mvn test -Dtest=UserServiceTest#testCreateUser
    
    # Запустить по паттерну имён
    mvn test -Dtest=*ServiceTest
    
    # Пропустить тесты (для сборки)
    mvn package -DskipTests
    
    # Запустить только интеграционные тесты (failsafe)
    mvn verify -Dit.test=OrderIntegrationTest
```

### Gradle

```text
    # Запустить все тесты
    ./gradlew test
    
    # Конкретный класс
    ./gradlew test --tests "com.example.UserServiceTest"
    
    # Конкретный метод
    ./gradlew test --tests "com.example.UserServiceTest.testCreateUser"
    
    # Паттерн имён
    ./gradlew test --tests "*ServiceTest"
    
    # Пропустить тесты
    ./gradlew build -x test
```

Пример: конфигурация test task в build.gradle:

```text
    test {
        useJUnitPlatform()
        // Фильтрация по тегам
        includeTags 'fast', 'integration'
        excludeTags 'slow', 'manual'
        // Отчёты
        testLogging {
            events 'passed', 'skipped', 'failed'
            exceptionFormat 'full'
        }
    }
```

---

## Фильтрация тестов

### По тегам (@Tag)

- Размечайте тесты тегами: `@Tag("fast")`, `@Tag("integration")`
- Запускайте выборочно через конфигурацию

```java
  // Разметка тестов тегами
  @Tag("unit")
  @Test
  void calculateTotal_shouldSumPrices() { }

  @Tag("integration")
  @Test
  void orderFlow_shouldPersistToDatabase() { }
```

- Maven: `mvn test -Dgroups=unit`
- Gradle: `includeTags 'unit'`
- junit-platform.properties:

```properties
  junit.jupiter.extensions.autodetection.enabled=true
  junit.jupiter.execution.tags.include=unit
  junit.jupiter.execution.tags.exclude=integration
```

### По пакетам и классам

- Maven: `-Dtest="com.example.service.*Test"`
- Gradle: `--tests "com.example.service.*"`

### По имени метода (паттерны)

- Поддерживаются wildcards: `*`, `?`
- Пример: `-Dtest=*Test#test*Login*`

---

## Отчёты о выполнении

### Maven Surefire (unit-тесты)

- Отчёты в `target/surefire-reports/`
- Форматы: `.txt` (человекочитаемый), `.xml` (для CI/инструментов)
- Автоматически генерируется при `mvn test`

### Maven Failsafe (integration-тесты)

- Отчёты в `target/failsafe-reports/`
- Запуск: `mvn verify` (не test!)
- Позволяет отделить быстрые и медленные тесты

### Кастомизация отчётов (pom.xml)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <reportsDirectory>${project.build.directory}/custom-reports</reportsDirectory>
        <trimStackTrace>false</trimStackTrace>
        <printSummary>true</printSummary>
    </configuration>
</plugin>
```

### HTML-отчёты через плагины

- `maven-site-plugin` + `maven-surefire-report-plugin`
- Генерация: `mvn site`
- Результат: `target/site/index.html` с графиками и статистикой

Пример: базовая конфигурация для HTML-отчётов

```xml
<reporting>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-report-plugin</artifactId>
            <version>3.2.5</version>
        </plugin>
    </plugins>
</reporting>
```
---

## Интеграция с CI/CD

### GitHub Actions

```yaml
    name: Tests
    on: [push, pull_request]
    jobs:
      test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Set up JDK
            uses: actions/setup-java@v4
            with: { java-version: '17', distribution: 'temurin' }
          - name: Run tests
            run: mvn -B test --file pom.xml
          - name: Upload reports
            uses: actions/upload-artifact@v4
            with:
              name: test-reports
              path: target/surefire-reports/
```

### GitLab CI

```yaml
    test:
      stage: test
      image: maven:3.9-eclipse-temurin-17
      script:
        - mvn test
      artifacts:
        when: always
        paths:
          - target/surefire-reports/
        reports:
          junit: target/surefire-reports/TEST-*.xml
```

### Jenkins (Declarative Pipeline)

```text
    pipeline {
        agent any
        stages {
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }
        }
        post {
            always {
                junit 'target/surefire-reports/TEST-*.xml'
            }
        }
    }
```

### Ключевые параметры для CI

- `-B, --batch-mode`: отключает интерактивный вывод
- `-Dsurefire.useFile=false`: логи в консоль (удобно для отладки)
- `-Dmaven.test.failure.ignore=true`: продолжить сборку при падении тестов
- `-Dtest=*IntegrationTest`: запускать только нужную группу

---

## Сводная таблица: команды запуска

| Задача      | Maven               | Gradle                    |
|:------------|---------------------|---------------------------|
| Все тесты   | mvn test            | ./gradlew test            |
| Класс       | -Dtest=ClassName    | --tests "ClassName"       |
| Метод       | -Dtest=Class#method | --tests "Class.method"    |
| Паттерн     | -Dtest=*Service*    | --tests "*Service*"       |
| Теги        | -Dgroups=fast       | includeTags 'fast'        |
| Пропустить  | -DskipTests         | -x test                   |
| Integration | mvn verify          | ./gradlew integrationTest |

---

## Best Practices

Делайте:

- Настройте фильтрацию по тегам для разделения fast/slow тестов
- Используйте артефакты CI для сохранения отчётов
- Включайте full stack trace в CI для быстрой отладки
- Разделяйте unit (surefire) и integration (failsafe) тесты
- Документируйте команды запуска в README

Не делайте:

- Не запускайте все тесты локально при каждой правке (используйте фильтрацию)
- Не игнорируйте падающие тесты в CI без веской причины
- Не храните большие отчёты в репозитории (используйте артефакты)
- Не полагайтесь только на IDE-запуск (проверяйте CLI)
- Не забывайте про timeout для долгих тестов в CI

---

## Ключевые выводы

1. **IDE для разработки, CLI для CI** — локально удобно через IDE, в пайплайнах только командная строка
2. **Фильтрация экономит время** — теги и паттерны позволяют запускать только нужные тесты
3. **Surefire vs Failsafe** — unit-тесты и integration-тесты требуют разных плагинов и фаз
4. **XML-отчёты для инструментов** — человекочитаемые .txt для разработчиков, .xml для CI/покрытия
5. **Артефакты в CI обязательны** — сохраняйте отчёты для анализа после завершения пайплайна
6. **Batch-mode для стабильности** — -B флаг Maven предотвращает зависания в headless-средах
7. **Документируйте запуск** — команды для IDE, Maven, Gradle и CI должны быть в README проекта

# 11. Миграция с JUnit 4

> Миграция на JUnit 5 требует обновления аннотаций, assertions и конфигурации сборки.

---

## Отличия в аннотациях

Большинство аннотаций изменились пакетом и семантикой. Ниже — основные соответствия:

- `@Test` — осталась, но теперь из `org.junit.jupiter.api.Test`
- `@Before` → `@BeforeEach` — выполняется перед каждым тестом
- `@After` → `@AfterEach` — после каждого теста
- `@BeforeClass` → `@BeforeAll` — один раз перед всеми тестами (метод должен быть static или использовать @TestInstance)
- `@AfterClass` → `@AfterAll` — один раз после всех тестов
- `@Ignore` → `@Disabled` — отключение тестов с причиной
- `@Category` → `@Tag` — маркировка для фильтрации
- `@Rule` / `@ClassRule` → `@ExtendWith` — механизм расширений вместо правил

---

## Отличия в Assertions

- Пакет: `org.junit.Assert` → `org.junit.jupiter.api.Assertions`
- Исключения: `assertThrows` возвращает исключение для дополнительных проверок
- Группировка: `assertAll` для множественных проверок без раннего выхода
- Таймауты: через `@Test(timeout)` → `assertTimeout` / `assertTimeoutPreemptively`

---

## Запуск JUnit 4 тестов в JUnit 5 (Vintage)

> Vintage Engine позволяет запускать старые JUnit 4 тесты в JUnit 5 платформе без изменений.

### Подключение зависимости

```xml
    <!-- Maven -->
    <dependency>
        <groupId>org.junit.vintage</groupId>
        <artifactId>junit-vintage-engine</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>
```

```groovy
    // Gradle
    testRuntimeOnly 'org.junit.vintage:junit-vintage-engine:5.10.0'
```

### Особенности Vintage

- Поддерживает `@Test`, `@Before`, `@Rule` и другие JUnit 4 аннотации
- Не поддерживает новые фичи JUnit 5 (`@ParameterizedTest`, `@Nested` и др.) в старых тестах
- Позволяет мигрировать постепенно: новые тесты на J5, старые работают через Vintage
- Отчёты объединяются: все тесты видны в одном запуске

```java
  // Пример: смешанные тесты в одном проекте
  // Новый тест на JUnit 5
  @ExtendWith(MockitoExtension.class)
  class OrderServiceTest {
  
    @Test
    void placeOrder_shouldSucceed() { }
  }

  // Старый тест на JUnit 4 (работает через Vintage)
  public class LegacyUserServiceTest {
  
    @Before
    public void init() { }

    @Test
    public void testCreate() { }
  }
```

---

## Пошаговая миграция

1. Обновите зависимости: `junit-jupiter 5.10.0`, удалите `junit 4.x` (оставьте для Vintage)
2. Добавьте `junit-vintage-engine` для обратной совместимости
3. Замените импорты: `org.junit.*` → `org.junit.jupiter.api.*`
4. Обновите аннотации: `@Before` → `@BeforeEach`, `@Ignore` → `@Disabled` и т.д.
5. Перепишите assertions: `assertThrows`, `assertAll`, новые матчеры
6. Обновите конфигурацию Maven/Gradle: `useJUnitPlatform()`
7. Удалите Vintage, когда все тесты мигрированы

Конфигурация Maven для миграции:

```xml   
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
        </includes>
   </configuration>
</plugin>
```

Конфигурация Gradle:

```text
   test {
   useJUnitPlatform()  // обязательно для JUnit 5
   // Vintage подключается автоматически при наличии зависимости
   }
```

---

## Сводная таблица: аннотации JUnit 4 → JUnit 5

| JUnit 4           | JUnit 5               | Примечание                                                |
|:------------------|-----------------------|-----------------------------------------------------------|
| @Test             | @Test                 | Пакет изменён, добавлены параметры (timeout, displayName) |
| @Before           | @BeforeEach           | Выполняется перед каждым @Test                            |
| @After            | @AfterEach            | Выполняется после каждого @Test                           |
| @BeforeClass      | @BeforeAll            | Метод static или @TestInstance(Lifecycle.PER_CLASS)       |
| @AfterClass       | @AfterAll             | Аналогично @BeforeAll                                     |
| @Ignore           | @Disabled             | Причина отключения рекомендуется                          |
| @Category         | @Tag                  | Фильтрация через includeTags / excludeTags                |
| @Rule             | @ExtendWith           | Механизм расширений вместо правил                         |
| @ClassRule        | @ExtendWith на классе | Расширения на уровне класса                               |
| ExpectedException | assertThrows          | Возвращает исключение для проверок                        |

---

## Best Practices

Делайте:

- Мигрируйте постепенно: начните с новых тестов на JUnit 5
- Используйте Vintage для поддержки легаси-кода
- Добавляйте причины к `@Disabled` для отслеживания пропущенных тестов
- Тестируйте миграцию в CI: убедитесь, что все тесты запускаются
- Документируйте этапы миграции в CHANGELOG

Не делайте:

- Не удаляйте Vintage, пока все тесты не переведены на JUnit 5
- Не смешивайте импорты JUnit 4 и 5 в одном файле
- Не игнорируйте предупреждения IDE об устаревших аннотациях
- Не меняйте логику тестов при замене аннотаций
- Не забывайте обновлять конфигурацию сборки (useJUnitPlatform)

---