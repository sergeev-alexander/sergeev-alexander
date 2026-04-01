# Allure

## Содержание

1. [Основы](#1-основы)
2. [Быстрое подключение](#2-быстрое-подключение)
3. [Аннотации Allure](#3-аннотации-allure)
4. [Интеграция в CI/CD](#4-интеграция-в-cicd)
5. [Полезные мелочи](#5-полезные-мелочи)
6. [Allure TestOps](#6-allure-testops)

---

## Что такое Allure

> Allure — фреймворк для генерации детализированных визуальных отчётов о результатах тестирования.
>
> Позволяет анализировать шаги тестов, вложения (скриншоты, логи), параметры и статистику в удобном интерфейсе.

- **Назначение** — Преобразование сырых результатов тестов в интерактивный HTML-отчёт
- **Архитектура** — Двухэтапная: `allure-results` (JSON-артефакты) → `allure-report` (HTML)
- **Поддерживаемые фреймворки**:
    - JUnit 4 / JUnit 5
    - TestNG
    - Cucumber JVM
    - Selenide
    - Serenity BDD
    - Spock, Mocha, pytest (для других языков)

## Как работает Allure

```text
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Тесты         │     │   allure-results│     │  allure-report  │
│ (JUnit/TestNG)  │---->│   (JSON-файлы)  │---->│ (HTML-интерфейс)│
└─────────────────┘     └─────────────────┘     └─────────────────┘
       │                       │                       │
       ▼                       ▼                       ▼
  @Step, @Attachment    target/allure-results       allure serve
  @Severity, @Feature   или build/allure-results   allure generate
```

## Зачем использовать Allure

✅ Преимущества:

- **Детализация** — Пошаговое отображение выполнения теста с параметрами
- **Артефакты** — Вложение скриншотов, логов, файлов прямо в шаги
- **Навигация** — Фильтрация по фичам, историям, приоритетам, дефектам
- **Интеграция** — Бесшовная работа с Jenkins, GitLab CI, GitHub Actions
- **Аналитика** — Графики регрессии, категоризация падений, метрики покрытия

❌ Ограничения:

- Требует дополнительной настройки зависимостей и плагинов
- Отчёт генерируется постфактум (не в реальном времени)
- Для полноценной работы нужен доступ к CLI или плагину CI

## Ключевые понятия

| Термин           | Описание                                                                                |
|:-----------------|:----------------------------------------------------------------------------------------|
| `allure-results` | Директория с сырыми JSON-результатами после запуска тестов                              |
| `allure-report`  | Сгенерированная HTML-версия отчёта для просмотра в браузере                             |
| Allure CLI       | Командная утилита для генерации и просмотра отчётов (`allure serve`, `allure generate`) |
| Adapter          | Библиотека-адаптер под конкретный фреймворк (`allure-junit5`, `allure-testng`)          |
| Attachment       | Артефакт (скриншот, лог, файл), прикреплённый к шагу или тесту                          |

---

## Ключевые выводы

1. **Allure — это слой визуализации**, а не фреймворк для написания тестов
2. **Результаты генерируются в два этапа**: сбор данных → построение отчёта
3. **Поддержка множества фреймворков** делает Allure универсальным решением для Java-проектов
4. **Интерактивный интерфейс** ускоряет анализ падений и коммуникацию в команде

---

# 2. Подключение

> Подключение Allure требует добавления адаптера под тестовый фреймворк и настройки генерации результатов.
>
> Все примеры ниже приведены для JUnit 5 и TestNG.

## Зависимости

### Maven (pom.xml)

```xml
<properties>
    <allure.version>2.24.0</allure.version>
    <aspectj.version>1.9.21</aspectj.version>
</properties>

<dependencies>
    
    <!-- Для JUnit 5 -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit5</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Для TestNG -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-testng</artifactId>
        <version>${allure.version}</version>
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
                <testFailureIgnore>false</testFailureIgnore>
                <argLine>
                    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                </argLine>
            </configuration>
            
            <dependencies>
                <dependency>
                    <groupId>org.aspectj</groupId>
                    <artifactId>aspectjweaver</artifactId>
                    <version>${aspectj.version}</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

## Gradle (build.gradle.kts)

```kotlin
val allureVersion = "2.24.0"
val aspectjVersion = "1.9.21"

dependencies {
    // для JUnit:
    testImplementation("io.qameta.allure:allure-junit5:$allureVersion")
    // для TestNG:
    testImplementation("io.qameta.allure:allure-testng:$allureVersion")
}

tasks.test {
    useJUnitPlatform()
    jvmArgs = listOf(
        "-javaagent:${configurations.testRuntimeClasspath.get().find { it.name.contains("aspectjweaver") }?.absolutePath}"
    )
}
```

## Плагин для генерации отчёта (Maven)

```xml
<plugin>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-maven</artifactId>
    <version>2.12.0</version>
    
    <configuration>
        <reportVersion>${allure.version}</reportVersion>
        <allureDirectory>${project.build.directory}/allure-results</allureDirectory>
    </configuration>
</plugin>
```

## Allure CLI: установка и команды

### Установка (все платформы)

```bash
# macOS (Homebrew)
brew install allure

# Linux / Windows (через SDKMAN)
sdk install allure

# Вручную: скачать с https://github.com/allure-framework/allure2/releases
```

## Основные команды

```bash
# Запустить тесты и сразу открыть отчёт в браузере
allure serve target/allure-results

# Сгенерировать статический HTML-отчёт
allure generate target/allure-results --clean -o allure-report

# Открыть готовый отчёт
allure open allure-report

# Показать версию и справку
allure --version
allure --help
```

## Структура результатов после запуска

```text
target/
├── allure-results/          # Сырые результаты (генерируются автоматически)
│   ├── *.json               # Данные тестов, шагов, аттачментов
│   ├── *.csv                # Метрики и категории
│   └── attachments/         # Бинарные файлы: скриншоты, логи
└── allure-report/           # Сгенерированный HTML (после allure generate)
    ├── index.html           # Точка входа
    ├── data/                # JSON-данные для интерфейса
    └── plugins/             # Плагины отчёта
```

---

## Best Practices

✅ Делайте:

- Указывайте версию Allure в `<properties>` для централизованного управления
- Используйте `${settings.localRepository}` для пути к aspectjweaver — это кросс-платформенно
- Добавляйте `--clean` в `allure generate`, чтобы избежать дублирования данных в отчёте
- Кэшируйте установку Allure CLI в CI для ускорения пайплайна

❌ Не делайте:

- Не коммитьте директорию `allure-results` или `allure-report` в Git
- Не забывайте про `-javaagent:aspectjweaver` — без него аннотации `@Step` не работают

---

## Ключевые выводы

1. **AspectJ — обязательное требование** для работы аннотаций `@Step` и `@Attachment`
2. **Результаты пишутся в `allure-results`** автоматически после каждого прогона тестов
3. **`allure serve` — лучший выбор для локальной разработки**, `allure generate` — для CI/CD
4. **Версии адаптера и CLI должны совпадать** во избежание несовместимости форматов

---

# 3. Аннотации Allure

> Аннотации Allure позволяют добавлять метаданные к тестам: описания, шаги, приоритеты, ссылки на задачи и артефакты.
>
> Все аннотации находятся в пакете `io.qameta.allure` (кроме `@Step` — `io.qameta.allure.step`).

## Основные аннотации

| Аннотация      | Назначение                   | Пример                                  |
|:---------------|------------------------------|-----------------------------------------|
| `@Description` | Текстовое описание теста     | `@Description("Проверка логина")`       |
| `@Step`        | Детализация шагов выполнения | `@Step("Ввод логина {login}")`          |
| `@Attachment`  | Вложение артефактов          | скриншоты, логи, файлы                  |
| `@Severity`    | Приоритет теста              | `SeverityLevel.CRITICAL`                |
| `@Epic`        | Высокоуровневая группировка  | `@Epic("Авторизация")`                  |
| `@Feature`     | Группировка по функционалу   | `@Feature("Логин")`                     |
| `@Story`       | Сценарий внутри функционала  | `@Story("Успешный вход")`               |
| `@Issue`       | Ссылка на баг в трекере      | `@Issue("JIRA-123")`                    |
| `@TmsLink`     | Ссылка на тест-кейс          | `@TmsLink("TC-456")`                    |
| `@Link`        | Произвольная ссылка          | `@Link(name = "Doc", url = "...")`      |
| `@Owner`       | Владелец теста               | `@Owner("ivanov")`                      |
| `@Label`       | Кастомная метка              | `@Label(name = "layer", value = "api")` |
               
---

## Уровни Severity

| Уровень                  | Описание           | Когда использовать          |
|:-------------------------|--------------------|-----------------------------|
| `SeverityLevel.BLOCKER`  | Блокирующий дефект | Критично для всего продукта |
| `SeverityLevel.CRITICAL` | Критический        | Основной функционал сломан  |
| `SeverityLevel.NORMAL`   | Обычный            | Стандартные тесты           |
| `SeverityLevel.MINOR`    | Незначительный     | Второстепенный функционал   |
| `SeverityLevel.TRIVIAL`  | Тривиальный        | Косметические проверки      |

```java
@Severity(SeverityLevel.CRITICAL)
@Test
void criticalLoginTest() {
    // критичный тест авторизации
}
```

---

## @Description — описание теста

Описание отображается в отчёте на вкладке теста и помогает понять назначение теста без чтения кода.

```java
@Description("Проверка авторизации пользователя с валидными данными")
@Test
void validLoginTest() {
    login("user", "password");
}
```

---

## @Step — шаги выполнения

Шаги отображаются иерархически в отчёте. При падении видно, на каком именно шаге произошла ошибка.

```java
@Step("Авторизация пользователя с логином {username}")
public void login(String username, String password) {
    openLoginPage();
    enterLogin(username);
    enterPassword(password);
    clickLoginButton();
}

@Step("Ввод логина {login}")
private void enterLogin(String login) {
    loginField.setValue(login);
}

@Test
void testWithSteps() {
    login("testUser", "password123");
}
```

---

## @Attachment — вложение артефактов

### Скриншот

```java
@Attachment(value = "Screenshot", type = "image/png")
public byte[] saveScreenshot(byte[] screenshot) {
    return screenshot;
}

@Test
void testWithScreenshot() {
    byte[] screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    saveScreenshot(screenshot);
}
```

### Лог

```java
@Attachment(value = "Application Log", type = "text/plain")
public String saveLog(String log) {
    return log;
}

@Test
void testWithLog() {
    String log = getApplicationLog();
    saveLog(log);
}
```

### Файл

```java
@Attachment(value = "Response JSON", type = "application/json")
public String saveJson(String json) {
    return json;
}
```

> **Типы MIME:** `image/png`, `text/plain`, `application/json`, `text/html`, `application/xml`

---

## @Epic, @Feature, @Story — BDD-группировка

В отчёте тесты группируются по `Epic` → `Feature` → `Story`, что удобно для навигации по функционалу.

```java
@Epic("Авторизация и управление пользователями")
@Feature("Авторизация")
@Story("Успешная авторизация с валидными данными")
@Test
void validLoginTest() {
    // логика теста
}

@Epic("Авторизация и управление пользователями")
@Feature("Авторизация")
@Story("Авторизация с невалидными данными")
@Test
void invalidLoginTest() {
    // логика теста
}
```

---

## @Issue и @TmsLink — ссылки на внешние системы

Ссылки становятся кликабельными в отчёте. Можно настроить префиксы URL в конфигурации.

```java
@Issue("BUG-123")
@TmsLink("TC-456")
@Test
void testWithLinks() {
    // тест связан с багом и тест-кейсом
}
```

---

## @Link — произвольные ссылки

```java
@Link(name = "Confluence Documentation", url = "http://confluence.company.com/docs")
@Link(name = "API Spec", url = "http://swagger.company.com/api")
@Test
void testWithDocs() {
    // тест с документацией
}
```

---

## @Owner и @Label — метаданные

Позволяет фильтровать тесты в отчёте по владельцу, слою, компоненту и другим кастомным меткам.

```java
@Owner("ivanov")
@Label(name = "layer", value = "api")
@Label(name = "component", value = "auth-service") // @Label(name = "layer", value = "api") — кастомная метка, которая добавляет в отчёт новый измеряемый параметр "layer" со значением "api"
@Test
void apiTest() {
    // API тест
}
```

---

## Полный пример теста

```java
@Epic("Авторизация")
@Feature("Авторизация пользователя")
@Story("Авторизация с валидными данными")
@Severity(SeverityLevel.BLOCKER)
@Owner("petrov")
@TmsLink("TC-789")
@Issue("BUG-345")
@Description("Тест проверяет успешную авторизацию пользователя с правильными данными")
@Link(name = "Требования", url = "http://confluence.company.com/req-123")
@Test
void validLoginTest() {
    login("validUser", "validPassword");
}

@Step("Авторизация пользователя с логином {username}")
public void login(String username, String password) {
    openLoginPage();
    enterLogin(username);
    enterPassword(password);
    clickLoginButton();
    verifySuccessMessage();
}

@Step("Ввод логина {login}")
@Attachment(value = "Login Input", type = "text/plain")
private void enterLogin(String login) {
    loginField.setValue(login);
}
```

---

## Best Practices

✅ Делайте:

- Используйте `@Step` для всех значимых действий в тесте
- Прикрепляйте скриншоты только при падении теста (в `@AfterMethod` или `@AfterEach`)
- Заполняйте `@Epic`, `@Feature`, `@Story` для навигации по большим проектам
- Добавляйте `@TmsLink` для связи с тест-кейсами в TestRail/Zephyr
- Используйте `@Severity` для приоритизации тестов в отчёте

❌ Не делайте:

- Не создавайте слишком глубокие вложенности шагов (максимум 3-4 уровня)
- Не прикрепляйте тяжёлые файлы (>10 МБ) — отчёт станет медленным
- Не дублируйте информацию из имени теста в `@Description`
- Не используйте `@Step` на приватных методах без необходимости

---

## Ключевые выводы

1. **`@Step` требует AspectJ** — без `-javaagent:aspectjweaver` аннотация не работает
2. **BDD-аннотации (`@Epic`, `@Feature`, `@Story`)** — основа навигации в больших отчётах
3. **`@Attachment` поддерживает любые типы файлов** — главное указать правильный MIME-type
4. **Внешние ссылки (`@Issue`, `@TmsLink`, `@Link`)** упрощают коммуникацию между командами
5. **Комбинируйте аннотации** для максимальной информативности отчёта

---

# 4. Интеграция в CI/CD

> Автоматизация генерации отчётов в CI/CD делает результаты тестов доступными для всей команды сразу после запуска пайплайна.
>
> Основные задачи: установка Allure CLI, генерация отчёта из `allure-results`, публикация артефактов.

## GitHub Actions

```yaml
name: Java CI with Allure

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'

      - name: Build and Test
        run: mvn clean test

      - name: Download Allure
        run: |
          wget https://repo.maven.apache.org/maven2/io/qameta/allure/allure-commandline/2.24.0/allure-commandline-2.24.0.zip
          unzip allure-commandline-2.24.0.zip -d /opt

      - name: Generate Allure Report
        run: /opt/allure-2.24.0/bin/allure generate target/allure-results --clean -o allure-report

      - name: Upload Allure Report
        uses: actions/upload-artifact@v3
        with:
          name: allure-report
          path: allure-report
```

## GitLab CI

```yaml
image: maven:3.8.6-openjdk-11

stages:
  - test
  - report

test:
  stage: test
  script:
    - mvn clean test
  artifacts:
    paths:
      - target/allure-results
    expire_in: 1 week

allure_report:
  stage: report
  script:
    - wget https://repo.maven.apache.org/maven2/io/qameta/allure/allure-commandline/2.24.0/allure-commandline-2.24.0.zip
    - unzip allure-commandline-2.24.0.zip -d /opt
    - /opt/allure-2.24.0/bin/allure generate target/allure-results --clean -o allure-report
  artifacts:
    paths:
      - allure-report
    expire_in: 1 week
  when: always
```

## Jenkins

### Вариант 1: Allure Plugin (рекомендуемый)

Требует установки плагина "Allure Jenkins Plugin" в Manage Plugins.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Allure Report') {
            steps {
                allure([
                    includeProperties: false,
                    jdk: '',
                    properties: [],
                    reportBuildPolicy: 'ALWAYS',
                    results: [[path: 'target/allure-results']]
                ])
            }
        }
    }
}
```
### Вариант 2: Command Line (без плагина)

```groovy
stage('Allure Report') {
    steps {
        sh '''
            wget https://repo.maven.apache.org/maven2/io/qameta/allure/allure-commandline/2.24.0/allure-commandline-2.24.0.zip
            unzip allure-commandline-2.24.0.zip -d /tmp
            /tmp/allure-2.24.0/bin/allure generate target/allure-results --clean -o allure-report
        '''
        archiveArtifacts artifacts: 'allure-report/**', fingerprint: true
    }
}
```

## Сравнение подходов

| Платформа      | Метод установки Allure | Публикация отчёта       | Особенности                                 |
|:---------------|------------------------|-------------------------|---------------------------------------------|
| GitHub Actions | `wget` + `unzip`       | `upload-artifact`       | Отчёт скачивается вручную или через preview |
| GitLab CI      | `wget` + `unzip`       | `artifacts`             | Встроенный просмотр артефактов в интерфейсе |
| Jenkins        | Плагин Allure          | Вкладка "Allure Report" | Нативная интеграция, история отчётов        |

---

## Best Practices

✅ Делайте:

- Кэшируйте установку Allure CLI (например, через Docker-образ с предустановленным Allure)
- Используйте `when: always` (GitLab) или `reportBuildPolicy: 'ALWAYS'` (Jenkins), чтобы видеть отчёт даже при падении тестов
- Сохраняйте `allure-results` как артефакт для отладки (даже если отчёт уже сгенерирован)
- Очищайте директорию отчёта флагом `--clean` перед генерацией

❌ Не делайте:

- Не генерируйте отчёт на каждом коммите в feature-ветке (только для PR и main)
- Не храните отчёты вечно — настройте `expire_in` (GitLab) или `discardOldBuilds` (Jenkins)
- Не запускайте `allure serve` в CI (он требует интерактивного доступа)

---

## Ключевые выводы

1. **Allure CLI необходим в CI** — плагин Jenkins лишь обёртка над ним
2. **`allure-results` — первичный артефакт**, отчёт вторичен и может быть сгенерирован позже
3. **Публикация отчёта должна быть гарантирована** даже при падении тестов (для анализа ошибок)
4. **Docker-образы с предустановленным Allure** ускоряют пайплайн на 30-60 секунд

---

