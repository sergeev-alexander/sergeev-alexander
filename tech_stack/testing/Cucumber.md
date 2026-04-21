**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

# Cucumber

## Содержание

1. [Введение в BDD и Cucumber](#1-введение-в-bdd-и-cucumber)
2. [Настройка окружения и проекта](#2-настройка-окружения-и-проекта)
3. [Язык Gherkin: синтаксис и лучшие практики](#3-язык-gherkin-синтаксис-и-лучшие-практики)
4. [Step Definitions: связь Gherkin с Java-кодом](#4-step-definitions-связь-gherkin-с-java-кодом)
5. [Управление состоянием и зависимостями](#5-управление-состоянием-и-зависимостями)
6. [Hooks: жизненный цикл сценариев](#6-хуки-жизненный-цикл-сценариев)
7. [Теги (Tags): организация и фильтрация тестов](#7-теги-tags-организация-и-фильтрация-тестов)
8. [Параметризация: Scenario Outline и Examples](#8-параметризация-scenario-outline-и-examples)
9. [Продвинутые техники Step Definitions](#9-продвинутые-техники-step-definitions)
10. [Интеграция с внешними системами](#10-интеграция-с-внешними-системами)
11. [Плагины и отчетность](#11-плагины-и-отчетность)
12. [Параллельное выполнение тестов](#12-параллельное-выполнение-тестов)
13. [Отладка и диагностика](#13-отладка-и-диагностика)
14. [Интеграция с CI/CD](#14-интеграция-с-cicd)
15. [Поддержка и масштабирование проекта](#15-поддержка-и-масштабирование-проекта)

---

# 1. Введение в BDD и Cucumber

> Behavior-Driven Development (BDD) — это эволюция TDD, смещающая фокус с технической реализации на поведение системы с точки зрения бизнеса. Cucumber выступает стандартом де-факто для реализации BDD в Java-экосистеме, обеспечивая мост между человеческим языком требований и автоматизированными тестами.

---

## Философия BDD и отличия от TDD

- **TDD (Test-Driven Development)** — фокус на корректности реализации методов и классов. Тесты пишутся разработчиками на языке кода до написания логики.
- **BDD (Behavior-Driven Development)** — фокус на ожидаемом поведении системы. Сценарии описываются на понятном бизнесу языке (Gherkin) совместно всеми участниками процесса.
- **Бизнес-ценность** — снижение стоимости изменений за счёт раннего выявления несоответствий требованиям и создания "живой документации", которая всегда синхронизирована с кодом.

## Экосистема Cucumber: связка Gherkin + Java + Тест-раннер

- `Gherkin` — предметно-ориентированный язык (DSL) для описания поведения в формате `Given-When-Then`. Файлы хранятся с расширением `.feature`.
- `Step Definitions` — Java-классы, связывающие шаги Gherkin с исполняемым кодом через аннотации `@Given`, `@When`, `@Then`.
- `Test Runner` — класс-точка входа, управляющий выполнением сценариев через конфигурацию и интеграцию с популярными тестовыми фреймворками.

```text
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Business   │────>│   Gherkin   │────>│   Cucumber  │
│ Requirements│     │  (.feature) │     │  (Executor) │
└─────────────┘     └─────────────┘     └─────────────┘
        ↓                   ↓                   ↓
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│ Three Amigos│────>│ Step Defs   │────>│ Runner + App │
│  (PO/QA/Dev)│     │   (Java)    │     │(JUnit/TestNG)│
└─────────────┘     └─────────────┘     └──────────────┘
```

## Участники процесса: "Три Амира"

- **Product Owner (PO)** — формулирует бизнес-требования, валидирует критерии приемки (Acceptance Criteria), приоритизирует фичи.
- **QA Engineer** — проектирует тестовые сценарии, выявляет пограничные случаи, автоматизирует шаги, обеспечивает стабильность регрессии.
- **Developer** — реализует шаг-определения (Step Definitions), интегрирует тесты с бизнес-логикой, обеспечивает тестопригодность кода.

---

## Когда применять BDD, а когда ограничиться TDD или классическим тестированием

| Критерий выбора             | BDD (Cucumber)                                            | TDD (JUnit/TestNG)                               | Классическое тестирование                     |
|:----------------------------|-----------------------------------------------------------|--------------------------------------------------|-----------------------------------------------|
| **Сложность требований**    | Высокая, требует согласования с бизнесом                  | Средняя, чёткие технические спецификации         | Низкая, стабильный функционал                 |
| **Участники команды**       | PO + QA + Dev активно взаимодействуют                     | Только разработчики                              | QA изолированно или после разработки          |
| **Документация**            | Требуется "живая", исполняемая спецификация               | Требуется только техническая документация        | Документация не критична или ведётся отдельно |
| **Скорость обратной связи** | Средний цикл, но высокая точность бизнес-валидации        | Быстрый цикл, фокус на архитектуре и unit-логике | Медленный цикл, ручные проверки               |
| **Поддержка легаси**        | Сложно внедрить в унаследованную систему без рефакторинга | Легко внедрить для новых модулей                 | Часто единственный вариант для legacy         |

## Интеграция с тест-раннерами: JUnit 5 vs TestNG

| Параметр              | JUnit 5 (`cucumber-junit-platform-engine`)               | TestNG (`cucumber-testng`)                                              |
|:----------------------|----------------------------------------------------------|-------------------------------------------------------------------------|
| **Аннотация раннера** | `@Suite` + `@ConfigurationParameter` или `@Cucumber`     | `@CucumberOptions` на классе, наследующем `AbstractTestNGCucumberTests` |
| **Запуск из CLI**     | `mvn test -Dcucumber.filter.tags="@smoke"`               | `mvn test -Dcucumber.options="--tags @smoke"`                           |
| **Параллелизм**       | Через `junit-platform.properties` и `parallel=true`      | Через `testng.xml` (`parallel="tests"`) или `@CucumberOptions`          |
| **Hooks и Lifecycle** | `@BeforeAll`, `@AfterAll` поддерживаются нативно         | `@BeforeSuite`, `@AfterSuite` из TestNG доступны для интеграции         |
| **Рекомендация**      | Стандарт для современных Spring Boot / Java 17+ проектов | Предпочтителен в enterprise-стеках с legacy TestNG сьютами              |

---

## Ключевые преимущества Cucumber

- **Живая документация** — `.feature`-файлы всегда отражают актуальное поведение системы, так как они являются исполняемыми тестами.
- **Снижение недопонимания** — единый язык (Ubiquitous Language) устраняет разрыв между бизнес-требованиями и технической реализацией.
- **Ранняя валидация требований** — сценарии пишутся до кода (Specification by Example), что позволяет выявить противоречия на этапе проектирования.
- **Переиспользование шагов** — однажды описанный шаг может использоваться в сотнях сценариев, сокращая дублирование кода.

## Best Practices

- Используйте BDD для сложных бизнес-процессов, где важна прозрачность требований, а не для тривиальных CRUD-операций.
- Ограничивайте `@Then` валидацией бизнес-результата, а не технических деталей (избегайте проверок состояний DOM или внутренних флагов в Gherkin).
- Не превращайте Cucumber в обёртку над UI-автоматизацией: шаги должны отражать действия пользователя, а не клики по координатам.
- Проводите регулярные сессии "Трёх Амиров" для синхронизации перед стартом спринта, а не после написания кода.
- Сохраняйте Gherkin-сценарии независимыми от реализации: избегайте упоминания конкретных технологий, библиотек или структур БД в `.feature`-файлах.
- При выборе раннера ориентируйтесь на существующий стек проекта: JUnit 5 — для новой разработки и Spring Boot, TestNG — для поддержки legacy-сьютов и сложной параметризации.

---

## 2. Настройка окружения и проекта

> Правильная конфигурация проекта — фундамент стабильной автоматизации. 
> 
> В этом разделе разберём зависимости, структуру каталогов, настройку тестовых раннеров для JUnit 5 и TestNG, 
> а также особенности работы с современными версиями Java и CI-окружениями.

## Зависимости для Maven и Gradle

```xml
<!-- Maven: core + DI контейнер + раннер (выберите один из двух вариантов ниже) -->
<dependencies>
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <version>7.15.0</version>
        <scope>test</scope>
    </dependency>
    
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-picocontainer</artifactId>
        <version>7.15.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Вариант А: JUnit 5 Platform -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit-platform-engine</artifactId>
        <version>7.15.0</version>
        <scope>test</scope>
    </dependency>
    
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>

    <!-- Вариант Б: TestNG -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-testng</artifactId>
        <version>7.15.0</version>
        <scope>test</scope>
    </dependency>
    
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.9.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

```groovy
// Gradle (Kotlin DSL): аналогичная структура
dependencies {
    testImplementation("io.cucumber:cucumber-java:7.15.0")
    testImplementation("io.cucumber:cucumber-picocontainer:7.15.0")
    
    // JUnit 5
    testImplementation("io.cucumber:cucumber-junit-platform-engine:7.15.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    
    // TestNG
    testImplementation("io.cucumber:cucumber-testng:7.15.0")
    testImplementation("org.testng:testng:7.9.0")
}
```

---

## Структура проекта

```text
src/
├── test/
│   ├── java/
│   │   └── com.example.tests/
│   │       ├── runners/           # Классы-раннеры (JUnit Suite / TestNG)
│   │       ├── steps/             # Step Definitions (бизнес-логика шагов)
│   │       ├── hooks/             # Глобальные хуки (@Before, @After)
│   │       ├── context/           # TestContext, DI-компоненты, State
│   │       ├── pages/             # Page Objects (Selenide/WebDriver)
│   │       └── api/               # REST-клиенты, DTO, фабрики данных
│   └── resources/
│       ├── features/              # .feature файлы (сценарии)
│       │   ├── auth/
│       │   └── checkout/
│       ├── cucumber.properties    # Глобальные настройки Cucumber
│       └── junit-platform.properties / testng.xml  # Конфигурация раннера
└── main/
    └── java/                      # Исходный код приложения (если тесты внутри проекта)
```

---

## Конфигурация Test Runner: JUnit 5 vs TestNG

```java
// JUnit 5 Runner: используется @Suite и конфигурационные параметры
import org.junit.platform.suite.api.*;
import static io.cucumber.junit.platform.engine.Constants.*;

@Suite
@IncludeEngines("cucumber") // Указываем движок Cucumber для JUnit 5
@SelectClasspathResource("features") // Путь к .feature файлам в classpath
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.tests") // Пакеты с step definitions
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, value = "pretty, html:target/report/index.html") // Форматы отчетов
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@smoke") // Фильтр по тегам
@ConfigurationParameter(key = EXECUTION_PARALLEL_PROPERTY_NAME, value = "scenarios") // Параллельный запуск на уровне сценариев
@ConfigurationParameter(key = EXECUTION_PARALLEL_CONFIG_FIXED_PARALLELISM_PROPERTY_NAME, value = "4") // Количество потоков
public class CucumberJUnitRunner {
    // Пустой класс, служит только точкой входа для JUnit Platform
}
```

```java
// TestNG Runner: наследование от AbstractTestNGCucumberTests
import io.cucumber.testng.CucumberOptions;
import io.cucumber.testng.AbstractTestNGCucumberTests;

@CucumberOptions(
        features = "classpath:features", // Путь к .feature файлам
        glue = {"com.example.tests.steps", "com.example.tests.hooks"}, // Пакеты с шагами и хуками
        plugin = {"pretty", "html:target/report/index.html", "rerun:target/rerun.txt"}, // Отчеты + rerun для упавших
        tags = "@smoke", // Фильтр сценариев по тегу
        monochrome = true, // Отключает ANSI-цвета в консоли (чище CI-логи)
        snippets = CucumberOptions.SnippetType.CAMELCASE, // Генерация заглушек в camelCase
        publish = false // Отключает публикацию в Cucumber Reports
)
public class CucumberTestNGRunner extends AbstractTestNGCucumberTests {

    // Можно переопределить scenarios() для кастомной группировки или параллелизма
    @Override
    @DataProvider(parallel = true) // Включаем параллельный запуск сценариев
    public Object[][] scenarios() {
        return super.scenarios();
    }
}
```

Разбор ключевых параметров `@CucumberOptions` / `Constants`:

- `features` / `@SelectClasspathResource` — путь к директории с `.feature` файлами. Поддерживает маски `classpath:features/**/*.feature`.
- `glue` — пакеты с Java-классами, содержащими Step Definitions и Hooks. Все указанные пакеты сканируются рефлексивно.
- `plugin` — формат и путь вывода отчетов. Можно указывать несколько плагинов через запятую.
- `tags` — фильтрация сценариев по тегам. Логика: `and`, `or`, `not`, скобки.
- `monochrome = true` — убирает ANSI-цвета в консоли, полезно для CI-логов.
- `snippets = CAMELCASE` — генерирует заглушки шагов в `camelCase` вместо `underscore_case`.
- `publish = false` — отключает автоматическую загрузку отчетов в Cucumber Reports (экономит трафик и время).

---

## Глобальная конфигурация: cucumber.properties

```properties
# Путь к step definitions и хукам (приоритет выше, чем в коде раннера)
cucumber.glue=com.example.tests.steps,com.example.tests.hooks

# Плагины отчетности и консоли
cucumber.plugin=pretty,json:target/cucumber.json,html:target/cucumber-report.html

# Фильтрация по тегам (переопределяется аргументами CLI)
cucumber.filter.tags=not @wip and not @ignore

# Локализация Gherkin (по умолчанию en, поддерживается ru)
cucumber.language=ru

# Отключение автоматического создания отчета в облаке
cucumber.publish.enabled=false

# Параллельное выполнение (для JUnit Platform)
cucumber.execution.parallel.enabled=true
cucumber.execution.parallel.config.strategy=fixed
cucumber.execution.parallel.config.fixed.parallelism=4
```

---

## Интеграция с IntelliJ IDEA

- Установка плагина: `Settings → Plugins → Marketplace → Cucumber for Java` + `Gherkin`.
- Навигация: `Ctrl/Cmd + Click` по шагу в `.feature` файле мгновенно переводит к Java-методу. Обратная навигация работает через иконку `▶` в gutter-области.
- Запуск: контекстное меню на feature-файле или директории `Run 'Feature...'`. Поддерживает передачу VM-опций `-Dcucumber.filter.tags="@regression"`.
- Валидация синтаксиса: подсветка ошибок в Gherkin, автодополнение тегов, предупреждения о неиспользуемых step definitions.
- Конфигурации: `Edit Configurations → Add JUnit/TestNG → указать класс раннера или шаблон `*TestRunner`.

---

## Поддержка Java 17+ и модульная система

- `IllegalReflectiveAccess`: начиная с Java 16, доступ к внутренним полям JDK через рефлексию закрыт по умолчанию. 

  Cucumber 7 использует `--add-opens java.base/java.lang=ALL-UNNAMED` автоматически, но в кастомных средах может потребоваться явное указание в `MAVEN_OPTS` или `build.gradle`.

- `JPMS (Java Platform Module System)`: если проект модульный, добавьте `requires io.cucumber.core;` и `opens com.example.tests.steps to io.cucumber.core;` в `module-info.java`.
- `Record` и `Sealed Classes`: Cucumber Expressions поддерживает маппинг параметров в Java Records нативно. Используйте их для DTO в `DataTable.asMaps(Class)`.
- `Test Resources Isolation`: в Java 17+ рекомендуется выносить тестовые данные в `src/test/resources` и не смешивать их с production-конфигами во избежание конфликтов ClassLoader.

---

## Best Practices

- Не храните раннеры в пакете по умолчанию (default package): всегда размещайте их в `com.example.tests.runners`.
- Разделяйте `glue`-пакеты по доменам: `com.example.tests.steps.auth`, `com.example.tests.steps.payment` для ускорения сканирования и уменьшения ложных срабатываний.
- Используйте `cucumber.properties` для базовой конфигурации, а параметры CLI/CI — для переопределения в пайплайнах.
- В TestNG всегда расширяйте `AbstractTestNGCucumberTests`, не копируйте логику запуска вручную: это обеспечит корректную работу `DataProvider` и отчетов.
- В JUnit 5 используйте `@Suite` вместо устаревшего `@RunWith(Cucumber.class)` из JUnit 4.
- Для миграции с JUnit 4 на JUnit 5 используйте `cucumber-junit-platform-engine`, он полностью обратно совместим с синтаксисом `@CucumberOptions` через адаптеры.

---

# 3. Язык Gherkin: синтаксис и лучшие практики

> Gherkin — предметно-ориентированный язык (DSL), предназначенный для описания поведения системы на понятном бизнесу языке. 
> 
> Он обеспечивает единый стандарт спецификаций (Specification by Example), которые одновременно являются исполняемыми тестами и живой документацией.

---

## Базовые ключевые слова и структура

- **Feature / Функция** — корневой элемент, описывающий тестируемую функциональность или бизнес-модуль. Содержит краткое описание и опциональные теги.
- **Scenario / Сценарий** — конкретный пример использования функции. Должен быть атомарным и проверяемым.
- **Given / Дано** — предусловия и начальное состояние системы. Подготовка данных, контекста, авторизации.
- **When / Когда** — действие пользователя или внешний триггер, запускающий процесс.
- **Then / Тогда** — ожидаемый результат или проверка состояния после действия.
- **And / И** и **But / Но** — логические продолжатели предыдущего ключевого слова. 

  Не несут самостоятельной семантики для парсера, только улучшают читаемость текста.

```gherkin
# language: en
@checkout
Feature: Оформление заказа гостевым пользователем

Scenario: Успешная оплата картой
  Given пользователь перешел на страницу корзины
  And в корзине добавлен товар "Ноутбук Pro 15"
  When пользователь выбирает способ оплаты "Карта"
  And подтверждает заказ
  Then система показывает страницу успешной оплаты
  And заказ получает статус "Оплачен"
```

## Правила именования и бизнес-фокус

- Используйте активный залог и настоящее время: `пользователь добавляет товар`, а не `товар был добавлен пользователем`.
- Избегайте технических деталей в `.feature`-файлах: не пишите `POST /api/cart`, `click button #submit`, `wait 5s`.
- Формулируйте шаги через бизнес-ценность: `система рассчитывает итоговую сумму с учетом НДС` вместо `вызывается метод calculateTotal()`.
- Именуйте Feature-файлы по доменным областям: `payment.feature`, `user-auth.feature`, `report-generation.feature`.

---

## Использование Background для общих предусловий

- **Background / Фон** выполняет шаги автоматически перед каждым `Scenario` в файле.
- Идеально подходит для установки контекста, который одинаков для всех сценариев в файле (авторизация, навигация, загрузка страницы).
- Не используйте `Background`, если предусловия различаются для разных сценариев — вынесите их в отдельные `Given`.
- Для программной инициализации (открытие браузера, очистка БД) предпочтительнее `@Before`-хуки, так как они не отображаются в отчетах как шаги теста.

| Критерий                | `Background` в Gherkin                       | `@Before` хук в Java                            |
|:------------------------|----------------------------------------------|-------------------------------------------------|
| **Видимость в отчетах** | Отображается как явные шаги каждого сценария | Скрыт от бизнес-аудитории, виден только в логах |
| **Язык описания**       | Gherkin (бизнес-уровень)                     | Java (технический уровень)                      |
| **Область применения**  | Настройка бизнес-контекста, общие данные     | Инициализация драйвера, БД, контекста DI        |
| **Поддержка раннерами** | Одинаково в JUnit 5 и TestNG                 | Зависит от версии Cucumber и DI-контейнера      |

```gherkin
Feature: Управление заказами в интернет-магазине

  Background:
    Given пользователь авторизован как "standard_user"
    And пользователь находится на главной странице каталога
    And в корзине нет товаров

  Scenario: Успешное создание заказа
    When пользователь добавляет товар "Ноутбук" в корзину
    And переходит в корзину
    And оформляет заказ
    Then появляется сообщение "Заказ успешно создан"
    And заказ отображается в личном кабинете

  Scenario: Добавление нескольких товаров в заказ
    When пользователь добавляет товар "Мышь" в корзину
    And добавляет товар "Клавиатура" в корзину
    And переходит в корзину
    Then в корзине отображается 2 товара

  Scenario: Удаление товара из заказа
    When пользователь добавляет товар "Монитор" в корзину
    And переходит в корзину
    And удаляет товар "Монитор" из корзины
    Then корзина пуста
```

## Комментарии и локализация

- Символ `#` используется для комментариев. Cucumber игнорирует их при парсинге.
- Локализация включается директивой `# language: <код>` в начале файла или глобально через `cucumber.language=ru` в `cucumber.properties`.
- Поддерживаемые языки: `en` (по умолчанию), `ru`, `fr`, `de`, `es` и другие. Ключевые слова автоматически переводятся.

```properties
# Глобальная настройка локализации в cucumber.properties
cucumber.language=ru
```

```gherkin
# language: ru
# Этот сценарий проверяет валидацию email при регистрации
Функция: Регистрация нового пользователя

  Сценарий: Валидный email проходит проверку
    Дано пользователь открывает форму регистрации
    Когда вводит email "test@domain.com"
    Тогда поле помечается как валидное
```

## DocString и многострочные данные

- Тройные кавычки `"""` используются для передачи многострочных текстовых аргументов в шаги.
- Идеально для JSON-пейлоадов, XML-структур, SQL-запросов, YAML-конфигов.
- В Java-шагах DocString автоматически передается в последний параметр метода типа `String`.

```gherkin
  When пользователь отправляет следующий JSON на эндпоинт создания заказа
    """json
    {
      "userId": 1042,
      "items": [
        {"productId": "SKU-001", "qty": 2},
        {"productId": "SKU-005", "qty": 1}
      ],
      "paymentMethod": "CARD"
    }
    """
  Then система возвращает статус 201 и идентификатор заказа
```

```java
@When("пользователь отправляет следующий JSON на эндпоинт создания заказа")
public void submitCreateOrderPayload(String jsonPayload) {
    // jsonPayload содержит весь текст между """
    OrderRequest request = JsonMapper.fromJson(jsonPayload, OrderRequest.class);
    apiClient.createOrder(request);
}
```

## Best Practices

- Один сценарий — одна бизнес-проверка. Не комбинируйте позитивные и негативные проверки в одном `Scenario`.
- Избегайте цепочек `And`/`But` более 3 раз подряд. Если шагов слишком много, сценарий, вероятно, нарушает принцип единой ответственности.
- Не используйте технические селекторы (`#id`, `.class`, `xpath`) в Gherkin. Опишите действие бизнес-уровнем: `нажимает "Оплатить"`, а не `кликает по кнопке с id="pay-btn"`.
- Храните `Background` максимально коротким (2–4 шага). Длинные предусловия замедляют выполнение и усложняют отладку.
- При локализации используйте единый язык во всем проекте. Смешивание локализаций в одной кодовой базе приводит к ошибкам парсинга и усложняет поддержку.
- DocString не должен содержать динамических параметров. Если данные меняются, используйте `Scenario Outline` с `Examples`.
- Синтаксис Gherkin не зависит от выбранного раннера (JUnit 5 или TestNG), однако при генерации отчетов `Background` шаги в JUnit Platform могут отображаться иначе, чем в TestNG, из-за различий в обработке `DataProvider`. 

  Учитывайте это при настройке `cucumber.plugin`.

---