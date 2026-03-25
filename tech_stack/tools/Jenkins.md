# Jenkins

## Содержание

1. [Что такое Jenkins](#что-такое-jenkins)
2. [Основные концепции](#основные-концепции)
3. [Установка](#установка)
4. [Настройка](#настройка)
5. [Pipeline](#pipeline)
6. [Интеграция с тестами](#интеграция-с-тестами)
7. [Best Practices](#best-practices)

---

## Что такое Jenkins

Jenkins - сервер автоматизации с открытым исходным кодом для CI/CD.

- Следит за изменениями в коде
- Автоматически запускает сборку и тесты
- Сообщает если что-то сломалось
- Может деплоить приложение на сервер

### Зачем нужен:

- Тесты запускаются сами
- Сборка при каждом коммите
- Красивые отчёты в UI
- Авто-уведомления в Slack/Email

---

## Основные концепции

### Job (Задача)

Единица работы в Jenkins. Бывает двух типов:

- `Freestyle` - Простые задачи с GUI настройкой
- `Pipeline` - Описанные кодом сложные CI/CD процессы (рекомендуется)

### Pipeline (Конвейер)

Последовательность этапов которые выполняются для сборки и тестирования.

```text
    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Checkout │--->│   Build  │--->│   Test   │--->│  Deploy  │
    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### Node/Agent (Агент)

Сервер, который выполняет задачи Jenkins.

- `Master` - Главный сервер Jenkins (управление)
- `Agent` - Рабочий сервер (выполняет задачи)

### Workspace

Папка на агенте где Jenkins работает с кодом проекта.

### Build (Сборка)

Один запуск пайплайна. У каждого билда есть номер и статус.

---

## Установка

### Вариант 1: Docker (рекомендуется)

Один команда для запуска:

```text
    docker run -d -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```

**Преимущества:**

- Быстрый старт
- Изолированная среда
- Легко обновлять
- Не загрязняет систему

**Доступ:** http://localhost:8080

---

### Вариант 2: Linux сервер

Установка через пакетный менеджер (Ubuntu):

```bash
    sudo apt update
    sudo apt install openjdk-11-jdk
    sudo apt install jenkins
    sudo systemctl start jenkins
```

**Когда использовать:** Production окружения, постоянные сервера

---

### Вариант 3: Локально

Скачать WAR файл и запустить:

```bash
    java -jar jenkins.war
```

**Когда использовать:** Тестирование, обучение

---

## Настройка

### Первоначальная настройка:

1. Открыть http://localhost:8080
2. Ввести начальный пароль (из логов или файла)
3. Установить рекомендуемые плагины
4. Создать администратора
5. Готово!

### Необходимые плагины:

| Плагин             | Назначение                     |
|:-------------------|--------------------------------|
| Git / GitHub       | Интеграция с Git репозиториями |
| Maven Integration  | Сборка Maven проектов          |
| Gradle Plugin      | Сборка Gradle проектов         |
| JUnit Plugin       | Отчёты по тестам               |
| Allure Plugin      | Красивые отчёты о тестах       |
| Docker Plugin      | Работа с Docker                |
| Slack Notification | Уведомления в Slack            |

### Установка плагинов:

`Manage Jenkins` → `Manage Plugins` → `Available` → `Найти` → `Install`

---

## Pipeline

### Что такое Pipeline:

Pipeline - код, который описывает весь процесс сборки и тестирования.

**Преимущества перед Freestyle:**

- Версионируется в Git
- Легко копировать между проектами
- Сложная логика (if/else, parallel)
- Лучшая визуализация

### Структура Pipeline:

```groovy
    pipeline {
        agent any
        
        stages {
            stage('Название этапа') {
                steps {
                    // Команды для выполнения
                }
            }
        }
        
        post {
            always {
                // Выполняется всегда
            }
            success {
                // Только если успешно
            }
            failure {
                // Только если упало
            }
        }
    }
```

### Основные директивы:

- `pipeline` - корневой блок, обязательный для любого Jenkins Pipeline
- `agent` - указывает, на каком агенте/ноде выполнять сборку (`any`, `none`, `label`; `any` - на любом доступном)
- `stages` - контейнер, содержащий все этапы (этапы выполняются последовательно)
- `stage`('Название') - Отдельный логический этап пайплайна (сборка, тесты, деплой и т.д.)
- `steps` - блок с фактическими командами внутри этапа (shell, batch, groovy и др.)
- `post` - блок для действий после завершения всего пайплайна (аналог try/catch/finally):
  - `always` - выполняется всегда, независимо от результата
  - `success` - только при успешном выполнении
  - `failure` - только при ошибке
- `environment` - переменные окружения
- `tools` - инструменты (Maven, Gradle, JDK)

---

### Пример для Maven проекта:

```groovy
    pipeline {
        agent any
        
        tools {
            maven 'Maven-3.8'
            jdk 'JDK-11'
        }
        
        stages {
            stage('Checkout') {
                steps {
                    git url: 'https://github.com/user/project.git', branch: 'main'
                }
            }
            
            stage('Build') {
                steps {
                    sh 'mvn clean compile'
                }
            }
            
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }
        }
        
        post {
            always {
                junit '**/target/surefire-reports/*.xml'
            }
        }
    }
```

---

### Пример для Gradle проекта:

```groovy
    pipeline {
        agent any
        
        tools {
            gradle 'Gradle-7'
            jdk 'JDK-11'
        }
        
        stages {
            stage('Checkout') {
                steps {
                    git url: 'https://github.com/user/project.git', branch: 'main'
                }
            }
            
            stage('Build & Test') {
                steps {
                    sh './gradlew build test'
                }
            }
        }
        
        post {
            always {
                junit '**/build/test-results/test/*.xml'
            }
        }
    }
```

---

### Параллельное выполнение тестов:

```groovy
    pipeline {
        agent any
        
        stages {
            stage('Parallel Tests') {
                parallel {
                    stage('Unit Tests') {
                        steps {
                            sh 'mvn test -Dtest=UnitTest'
                        }
                    }
                    
                    stage('Integration Tests') {
                        steps {
                            sh 'mvn test -Dtest=IntegrationTest'
                        }
                    }
                    
                    stage('E2E Tests') {
                        steps {
                            sh 'mvn test -Dtest=E2ETest'
                        }
                    }
                }
            }
        }
    }
```

---

## Интеграция с тестами

### Типы тестов в Jenkins:

| Тип тестов     | Когда запускать    | Плагин для отчётов |
|:---------------|--------------------|--------------------|
| Unit тесты     | При каждом коммите | JUnit Plugin       |
| Интеграционные | Nightly build      | JUnit Plugin       |
| E2E тесты      | На staging         | Allure Plugin      |
| Performance    | По расписанию      | Performance Plugin |

### Настройка JUnit отчётов:

```groovy
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
        }
    }
```

Jenkins автоматически:

- Парсит XML отчёты
- Показывает количество тестов
- Выделяет упавшие тесты красным
- Строит графики стабильности

---

### Настройка Allure отчётов:

```groovy
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
                allure includeProperties: false, 
                       jdk: '', 
                       results: [[path: 'target/allure-results']]
            }
        }
    }
```

**Преимущества Allure:**\

- Красивые интерактивные отчёты
- История тестов
- Allure может автоматически прикреплять скриншоты к отчёту, но только если вы явно добавите их в код тестов
- Классификация тестов

---

### Триггеры запуска:

| Триггер            | Описание                         | Пример                      |
|:-------------------|----------------------------------|-----------------------------|
| Poll SCM           | Проверка изменений по расписанию | H/5 * * * * (каждые 5 мин)  |
| GitHub hook        | Запуск при push в GitHub         | Автоматически               |
| Build periodically | Запуск по расписанию             | H 2 * * * (каждую ночь в 2) |
| Manual             | Ручной запуск                    | Кнопка "Build Now"          |
| Upstream           | Запуск после другого job         | После build job             |

---

### Уведомления:

#### Email уведомления:

```groovy
    post {
        failure {
            mail to: 'team@company.com',
                 subject: "Failed: ${env.JOB_NAME}",
                 body: "Check: ${env.BUILD_URL}"
        }
    }
```

#### Slack уведомления:

```groovy
    pipeline {
        agent any
        
        options {
            slackSend(color: 'GOOD', message: 'Build started')
        }
        
        post {
            failure {
                slackSend(color: 'DANGER', message: 'Build failed!')
            }
            success {
                slackSend(color: 'GOOD', message: 'Build success!')
            }
        }
    }
```

---

## Best Practices

### Для Pipeline:

- Храните Jenkinsfile в репозитории с кодом
- Используйте declarative pipeline (проще)
- Делайте этапы атомарными (один этап = одна задача)
- Кэшируйте зависимости (Maven, npm)
- Используйте parallel для ускорения тестов

### Для безопасности:

- Не храните пароли в коде
- Используйте Credentials в Jenkins
- Ограничивайте доступ к job
- Регулярно обновляйте Jenkins и плагины
- Используйте HTTPS для доступа

### Для производительности:

- Используйте агенты для тяжёлых задач
- Очищайте workspace периодически
- Настраивайте retention policy для старых билдов
- Параллельте независимые тесты
- Кэшируйте Docker образы

### Для отладки:

- Смотрите логи в UI Jenkins
- Используйте echo для дебага переменных
- Тестируйте pipeline локально (Jenkins Pipeline Linter)
- Сохраняйте артефакты для анализа

---

## Типичный workflow

### Разработка с Jenkins:

1. Разработчик делает push в Git
2. GitHub отправляет webhook в Jenkins
3. Jenkins запускает pipeline
4. Checkout кода из Git
5. Сборка проекта (Maven/Gradle)
6. Запуск unit тестов
7. Публикация отчётов
8. Уведомление в Slack
9. Если всё зелёно → готово к мержу

### Настройка нового проекта:

1. Установить Jenkins (Docker)
2. Установить необходимые плагины
3. Настроить credentials для Git
4. Создать новый Pipeline job
5. Указать URL репозитория
6. Добавить Jenkinsfile в репозиторий
7. Запустить первый билд
8. Настроить уведомления

---

## Глоссарий терминов

- `Job` - Задача в Jenkins (проект)
- `Pipeline` - Последовательность этапов сборки
- `Stage` - Этап внутри pipeline
- `Step` - Отдельная команда внутри stage
- `Agent` - Сервер для выполнения задач
- `Workspace` - Рабочая папка проекта
- `Build` - Один запуск pipeline
- `Artifact` - Файл созданный в процессе сборки
- `Trigger` - Событие запускающее билд
- `Credentials` - Учётные данные (пароли, ключи)
- `Jenkinsfile` - Файл с описанием pipeline
- `Downstream` - Job который запускается после текущего
- `Upstream` - Job который запускает текущий

---

## Частые проблемы и решения

| Проблема                | Причина                   | Решение                               |
|:------------------------|---------------------------|---------------------------------------|
| Pipeline не запускается | Нет триггера              | Проверить настройки SCM               |
| Тесты не находятся      | Неправильный путь         | Проверить pattern в junit             |
| Нет Maven/Gradle        | Не настроены tools        | Настроить в Global Tool Configuration |
| Permission denied       | Нет прав на файл          | chmod +x для скриптов                 |
| Timeout                 | Тесты слишком долгие      | Увеличить timeout или оптимизировать  |
| No space                | Кончилось место на агенте | Очистить workspace или добавить диск  |
| Plugin не работает      | Несовместимость версий    | Обновить Jenkins и плагины            |

---

## Ключевые выводы

1. **Jenkins - это автоматизация** - избавляет от ручной работы
2. **Pipeline as Code** - храните конфигурацию в Git
3. **Тесты должны быть быстрыми** - оптимизируйте для CI
4. **Уведомления важны** - команда должна знать о проблемах
5. **Безопасность прежде всего** - не храните секреты в коде
6. **Мониторьте Jenkins** - следите за здоровьем сервера
7. **Документируйте pipeline** - чтобы команда понимала процесс