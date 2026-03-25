# GitLab CI

## Содержание

1. [Что такое GitLab CI](#что-такое-gitlab-ci)
2. [Основные концепции](#основные-концепции)
3. [Файл .gitlab-ci.yml](#файл-gitlab-ci-yml)
4. [Ключевые элементы](#ключевые-элементы)
5. [Примеры пайплайнов](#примеры-пайплайнов)
6. [Оптимизация](#оптимизация)
7. [Best Practices](#best-practices)

---

## Что такое GitLab CI

GitLab CI — встроенная система непрерывной интеграции в GitLab.

- Автоматически запускается при push или MR
- Выполняет сборку и тесты
- Показывает результаты прямо в интерфейсе GitLab
- Блокирует мерж если тесты упали

### Преимущества перед Jenkins:

| GitLab CI                  | Jenkins                            |
|:---------------------------|------------------------------------|
| Встроен в GitLab           | Отдельный сервер                   |
| Конфигурация в YAML        | Конфигурация в UI или Jenkinsfile  |
| Не нужно устанавливать     | Нужно устанавливать и поддерживать |
| Бесплатно в GitLab         | Бесплатно но требует ресурсов      |
| Интеграция с MR из коробки | Требует настройки                  |

---

## Основные концепции

### Pipeline (Пайплайн)

Последовательность задач которые выполняются при событии (push, MR):

```test
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │   Push   │--->│ Pipeline │--->│  Отчёт   │
    └──────────┘    └──────────┘    └──────────┘
```

### Stage (Этап)

Группа задач которые выполняются параллельно внутри пайплайна.

```yaml
    stages:
      - build
      - test
      - deploy
```

`build` → `test` → `deploy` (последовательно)

### Job (Задача)

Конкретная работа внутри stage. Несколько job в одном stage выполняются параллельно.

```yaml
    test:
      stage: test
      script:
        - mvn test
```

### Runner (Исполнитель)

Сервер, который выполняет задачи пайплайна.

- `Shared Runners` - Общие для всех проектов (предоставляет GitLab)
- `Specific Runners` - Привязаны к конкретному проекту
- `Group Runners` - Доступны всем проектам в группе

### Artifacts (Артефакты)

Файлы, которые сохраняются после выполнения `job` (отчёты, билды).

---

## Файл .gitlab-ci.yml

> `.gitlab-ci.yml` использует плоскую структуру, где:
>
>- Ключи-настройки: stages, image, cache, before_script, variables и т.д.
>- Ключи-задания: всё остальное (например, build, test, deploy)

### Расположение:

Файл должен находиться в корне репозитория.

```text
    project/
    ├── .gitlab-ci.yml
    ├── src/
    ├── pom.xml
    └── README.md
```

### Минимальная структура:

```yaml
stages:
  - build
  - test

build_job:
  stage: build
  script:
    - mvn clean compile  # Компилирует исходный код
  artifacts:
    paths:
      - target/          # Сохраняет скомпилированные файлы для следующих стадий

test_job:
  stage: test
  script:
    - mvn test           # Запускает тесты
  dependencies:
    - build_job          # Использует артефакты из build_job
```

### Как работает:

1. Разработчик делает push в репозиторий
2. GitLab находит .gitlab-ci.yml
3. Создаётся pipeline
4. Runner выполняет job
5. Результаты отображаются в UI

---

## Ключевые элементы

### image

Docker образ в котором выполняются задачи.

```yaml
    image: maven:3.8.1-jdk-11
```

**Популярные образы для Java:**

- maven:3.8-jdk-11
- maven:3.9-jdk-17
- gradle:7-jdk11
- gradle:8-jdk17

---

### stages

Определяет последовательность этапов.

```jaml
    stages:
      - build
      - test
      - deploy
```

**Правила:**

- **Job без stage попадает в test**
  
  Если в задании (job) не указан параметр stage, оно автоматически попадает в этап test:

  ```yaml
  stages:
    - build
    - deploy

  my_job:           # stage не указан
      script:
        - echo "Hello..."
  ```

  Это задание выполнится на этапе test, даже если этап test явно не объявлен в stages.


- **Этапы выполняются последовательно**
  
  Этапы идут строго один за другим в том порядке, как они перечислены в stages:
  `build` → `test` → `deploy`
  
  Следующий этап не начнётся, пока все задания предыдущего этапа не завершатся (успешно или с ошибкой — зависит от настройки).


- **Job внутри stages выполняются параллельно**
  
  Все задания внутри одного этапа запускаются одновременно (если хватает раннеров):

  ```yaml
    test_job_1:
        stage: test
        script: echo "Test 1..."

    test_job_2:
        stage: test
        script: echo "Test 2..."
  ```
  
  Оба задания выполнятся параллельно, сокращая общее время пайплайна.

---

### script

Команды, которые выполняются в job.

```yaml
    job_name:
      script:
        - mvn clean compile
        - mvn test
        - echo "Done!"
```

---

### cache

Кэширование файлов между запусками пайплайна.

**Ускоряет сборку не загружая зависимости каждый раз**

**Для Maven:**

```yaml
cache:
  paths:
    - .m2/repository
```

**Для Gradle:**

```yaml
cache:
  paths:
    - .gradle/caches
    - .gradle/wrapper
```
---

### artifacts

Сохранение файлов после выполнения job.

```yaml
    artifacts:
      paths:
        - target/*.jar
      when: always
      reports:
        junit: target/surefire-reports/*.xml
```

Параметры в artifacts:

- `paths` - Какие файлы сохранить
- `when` - Когда сохранять (`always` / `on_success` / `on_failure`)
- `expire_in` - Время хранения (1 week, 30 days)
- `reports` - Специальные отчёты (junit, coverage)

---

### rules / only / except

Условия выполнения job.

```yaml
    job_name:
      script:
        - mvn test
      rules:
        - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      only:
        - main
        - develop
      except:
        - tags
```

**rules (рекомендуется):**

- `if:` Условие на основе переменных
- `changes:` Запускать если изменились файлы
- `exists:` Запускать если файл существует

**only/except (устаревает):**
- `only:` Запускать только для указанных веток
- `except:` Не запускать для указанных веток

---

### variables

Переменные окружения.
```yaml
    variables:
      MAVEN_OPTS: "-Xmx2048m"
      JAVA_HOME: "/usr/lib/jvm/java-11"
```

**Типы переменных:**

| Тип        | Описание                | Пример                   |
|:-----------|-------------------------|--------------------------|
| Predefined | Предопределённые GitLab | CI_COMMIT_SHA, CI_JOB_ID |
| Project    | Настроенные в проекте   | API_KEY, DB_PASSWORD     |
| File       | Из файла                | secrets.env              |
| Masked     | Скрыты в логах          | Пароли, токены           |

---

## Примеры пайплайнов

### Maven проект:

```yaml
    stages:
      - build
      - test

    image: maven:3.8.1-jdk-11

    cache:
      paths:
        - .m2/repository

    build:
      stage: build
      script:
        - mvn clean compile
      artifacts:
        paths:
          - target/

    test:
      stage: test
      script:
        - mvn test
      artifacts:
        when: always
        reports:
          junit: target/surefire-reports/*.xml
```
---

### Gradle проект:

```yaml
    stages:
      - build
      - test

    image: gradle:7.6-jdk11

    cache:
      paths:
        - .gradle/caches
        - .gradle/wrapper

    build:
      stage: build
      script:
        - gradle build
      artifacts:
        paths:
          - build/libs/

    test:
      stage: test
      script:
        - gradle test
      artifacts:
        when: always
        reports:
          junit: build/test-results/test/*.xml
```

---

### С деплоем:

```yaml
    stages:
      - build
      - test
      - deploy

    image: maven:3.8.1-jdk-11

    build:
      stage: build
      script:
        - mvn clean package

    test:
      stage: test
      script:
        - mvn test

    deploy:
      stage: deploy
      script:
        - ./deploy.sh
      only:
        - main
      when: manual
```

---

### Параллельные тесты:

```yaml
    stages:
      - build
      - test

    build:
      stage: build
      script:
        - mvn clean compile

    test_unit:
      stage: test
      script:
        - mvn test -Dtest=UnitTest

    test_integration:
      stage: test
      script:
        - mvn test -Dtest=IntegrationTest

    test_e2e:
      stage: test
      script:
        - mvn test -Dtest=E2ETest
```

**Результат:** 3 набора тестов выполняются параллельно

---

## Оптимизация

### Кэширование зависимостей:

```yaml
    cache:
      key: $CI_COMMIT_REF_SLUG
      paths:
        - .m2/repository
```

**Преимущества:**

- Ускорение сборки на 30-50%
- Меньше нагрузка на сеть
- Экономия времени разработчиков

---

### Параллелизация:

```yaml
    test:
      stage: test
      parallel: 5
      script:
        - mvn test
```

**Результат:** Тесты разделяются на 5 параллельных job

---

### Только для MR:

Не запускать тесты для каждого коммита в feature ветке


```yaml
    test:
      stage: test
      script:
        - mvn test
      rules:
        - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```


Задание `test` будет выполняться ТОЛЬКО когда пайплайн запущен из-за события создания или обновления Merge Request.

Когда НЕ будет запускаться:

- При обычном пуше (git push) в любую ветку (включая feature-ветки)
- При ручном запуске
- По расписанию

Когда БУДЕТ запускаться:

- Когда вы создаёте Merge Request
- Когда пушите новые коммиты в уже открытый Merge Request

---

### Skip для определённых файлов:

```yaml
    test:
      stage: test
      script:
        - mvn test
      rules:
        - changes:
            - src/**/*
            - pom.xml
```

Задание test запустится, если изменения в коммите затрагивают:

- Любые файлы в папке src/ (и её подпапках)
- Файл pom.xml


**Зачем:** Не запускать тесты если изменились только README или документация

---

## Интеграция с тестами

### JUnit отчёты:

```yaml
    test:
      stage: test
      script:
        - mvn test
      artifacts:
        when: always
        reports:
          junit: target/surefire-reports/*.xml
```

**Что даёт:**

- Красивый UI с результатами тестов
- История прохождения тестов
- Быстрый доступ к failed тестам
- Интеграция с MR

---

### Code Coverage:

```yaml
    test:
      stage: test
      script:
        - mvn test jacoco:report
      coverage: '/Total coverage: \d+\.\d+%/'
      artifacts:
        reports:
          coverage_report:
            coverage_format: cobertura
            path: target/site/cobertura/coverage.xml
```

**Что даёт:**

- Процент покрытия кода
- График покрытия в MR
- Блокировка мержа при низком покрытии

---

### Allure отчёты:

```yaml
    test:
      stage: test
      script:
        - mvn test
        - allure generate
      artifacts:
        paths:
          - allure-report/
        expire_in: 1 week
```

**Что даёт:**

- Интерактивные отчёты
- История тестов
- Скриншоты UI тестов

---

## Триггеры запуска

| Событие       | Когда запускается          |
|:--------------|----------------------------|
| Push в ветку  | При каждом коммите         |
| Merge Request | При создании/обновлении MR |
| Tag           | При создании тега          |
| Schedule      | По расписанию (cron)       |
| Manual        | Кнопка в UI                |
| API           | Через GitLab API           |
| Webhook       | Из внешнего сервиса        |

### Настройка расписания:

`CI/CD` → `Schedules` → `New Schedule`

```text
    0 2 * * *  # Каждую ночь в 2:00
```

---

## Best Practices

### Для .gitlab-ci.yml:

- Храните файл в корне репозитория
- Используйте конкретные версии образов (не latest)
- Кэшируйте зависимости
- Делайте job атомарными
- Используйте rules вместо only/except

### Для производительности:

- Кэшируйте .m2 и .gradle
- Параллельте независимые тесты
- Используйте incremental сборку
- Очищайте старые артефакты
- Настраивайте timeout для долгих job

### Для безопасности:

- Не храните секреты в .gitlab-ci.yml
- Используйте CI/CD Variables для паролей
- Маскируйте чувствительные variables
- Ограничивайте доступ к protected веткам
- Проверяйте образы на уязвимости

### Для отладки:

- Смотрите логи в CI/CD → Jobs
- Используйте echo для дебага
- Запускайте job вручную для тестов
- Проверяйте YAML в CI/CD → Pipeline Editor
- Используйте ci/lint для валидации

---

## Глоссарий терминов

- `Pipeline` - Последовательность выполнения задач
- `Stage` - Этап внутри pipeline
- `Job` - Конкретная задача внутри stage
- `Runner` - Сервер выполняющий job
- `Artifacts` - Файлы сохраняемые после job
- `Cache` - Кэш между запусками pipeline
- `Image` - Docker образ для выполнения
- `Variables` - Переменные окружения
- `Rules` - Условия выполнения job
- `MR` - Merge Request
- `Protected Branch` - Ветка с ограниченным доступом
- `Review App` - Временное окружение для MR

---

## Предопределённые переменные

- `CI_COMMIT_SHA` - Хеш коммита
- `CI_COMMIT_BRANCH` - Имя ветки
- `CI_JOB_ID` - ID текущей job
- `CI_PIPELINE_ID` - ID pipeline
- `CI_PROJECT_DIR` - Путь к проекту
- `CI_REGISTRY_IMAGE` - Docker image в registry
- `CI_ENVIRONMENT_NAME` - Имя окружения
- `CI_MERGE_REQUEST_ID` - ID Merge Request

---

## Частые проблемы и решения

| Проблема                | Причина               | Решение                          |
|:------------------------|-----------------------|----------------------------------|
| Pipeline не запускается | Нет .gitlab-ci.yml    | Добавить файл в корень           |
| Job падает сразу        | Неправильный image    | Проверить название образа        |
| Кэш не работает         | Неправильный key      | Использовать $CI_COMMIT_REF_SLUG |
| Тесты не видны          | Неправильный path     | Проверить путь к отчётам         |
| Долгая сборка           | Нет кэша              | Добавить cache для зависимостей  |
| Permission denied       | Нет прав на файл      | chmod +x для скриптов            |
| Runner не доступен      | Нет свободных runners | Использовать shared runners      |

---

## Ключевые выводы

1. **GitLab CI встроен в GitLab** — не нужно отдельной установки
2. **Всё в YAML** — конфигурация в .gitlab-ci.yml
3. **Docker по умолчанию** — каждый job в контейнере
4. **Кэш ускоряет сборку** — обязательно кэшируйте зависимости
5. **Отчёты интегрированы** — JUnit работает из коробки
6. **MR блокируется при падении тестов** — качество кода под контролем
7. **Бесплатно для всех** — доступно даже на free тарифе