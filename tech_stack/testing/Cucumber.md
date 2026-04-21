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

## Интеграция с IntelliJ IDEA

- Установка плагина: `Settings → Plugins → Marketplace → Cucumber for Java` + `Gherkin`.
- Навигация: `Ctrl/Cmd + Click` по шагу в `.feature` файле мгновенно переводит к Java-методу. Обратная навигация работает через иконку `▶` в gutter-области.
- Запуск: контекстное меню на feature-файле или директории `Run 'Feature...'`. Поддерживает передачу VM-опций `-Dcucumber.filter.tags="@regression"`.
- Валидация синтаксиса: подсветка ошибок в Gherkin, автодополнение тегов, предупреждения о неиспользуемых step definitions.
- Конфигурации: `Edit Configurations → Add JUnit/TestNG → указать класс раннера или шаблон `*TestRunner`.

## Поддержка Java 17+ и модульная система

- `IllegalReflectiveAccess`: начиная с Java 16, доступ к внутренним полям JDK через рефлексию закрыт по умолчанию. 

  Cucumber 7 использует `--add-opens java.base/java.lang=ALL-UNNAMED` автоматически, но в кастомных средах может потребоваться явное указание в `MAVEN_OPTS` или `build.gradle`.

- `JPMS (Java Platform Module System)`: если проект модульный, добавьте `requires io.cucumber.core;` и `opens com.example.tests.steps to io.cucumber.core;` в `module-info.java`.
- `Record` и `Sealed Classes`: Cucumber Expressions поддерживает маппинг параметров в Java Records нативно. Используйте их для DTO в `DataTable.asMaps(Class)`.
- `Test Resources Isolation`: в Java 17+ рекомендуется выносить тестовые данные в `src/test/resources` и не смешивать их с production-конфигами во избежание конфликтов ClassLoader.

## Best Practices

- Не храните раннеры в пакете по умолчанию (default package): всегда размещайте их в `com.example.tests.runners`.
- Разделяйте `glue`-пакеты по доменам: `com.example.tests.steps.auth`, `com.example.tests.steps.payment` для ускорения сканирования и уменьшения ложных срабатываний.
- Используйте `cucumber.properties` для базовой конфигурации, а параметры CLI/CI — для переопределения в пайплайнах.
- В TestNG всегда расширяйте `AbstractTestNGCucumberTests`, не копируйте логику запуска вручную: это обеспечит корректную работу `DataProvider` и отчетов.
- В JUnit 5 используйте `@Suite` вместо устаревшего `@RunWith(Cucumber.class)` из JUnit 4.
- Для миграции с JUnit 4 на JUnit 5 используйте `cucumber-junit-platform-engine`, он полностью обратно совместим с синтаксисом `@CucumberOptions` через адаптеры.