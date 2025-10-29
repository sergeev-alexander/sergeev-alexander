# 🐳 Docker Commands Cheat Sheet

### 📦 Управление контейнерами

<details>
    <summary>
        <b>Запуск и остановка</b>
    </summary>

```ignorelang
# Запуск всех сервисов
docker-compose up
docker-compose up -d          # в фоновом режиме
docker-compose up --build     # пересобрать и запустить

# Остановка
docker-compose down           # остановить и удалить контейнеры
docker-compose stop           # только остановить
docker-compose start          # запустить остановленные

# Перезапуск
docker-compose restart
```
</details>

<details>
    <summary>
        <b>Просмотр информации</b>
    </summary>

```ignorelang
# Список контейнеров
docker ps                     # работающие контейнеры
docker ps -a                  # все контейнеры (включая остановленные)

# Логи
docker-compose logs
docker-compose logs -f        # следить за логами в реальном времени
docker-compose logs app       # логи только сервиса app
docker logs bank_app          # логи конкретного контейнера

# Информация о сервисах
docker-compose ps             # статус сервисов
docker-compose images         # используемые образы
```
</details>

### 🧹 Очистка

<details>
    <summary>
        <b>Очистка контейнеров</b>
    </summary>

```ignorelang
#Запущеные контейнеры

docker-compose down            # Останавливает и удаляет контейнеры
docker-compose down -v         # Останавливает контейнеры, удаляет их И удаляет volumes

#Остановленные контейнеры

# Удалить все остановленные контейнеры
docker container prune

# Удалить конкретный контейнер
docker rm container_name
docker rm container_id

# Удалить все контейнеры (осторожно!)
docker rm -f $(docker ps -aq)
```
</details>

<details>
    <summary>
        <b>Очистка образов</b>
    </summary>

```ignorelang
# Удалить все неиспользуемые образы
docker image prune

# Удалить все образы (осторожно!)
docker rmi $(docker images -q)

# Удалить конкретный образ
docker rmi image_name:tag
docker rmi image_id
```
</details>

<details>
    <summary>
        <b>Полная очистка системы</b>
    </summary>

```ignorelang
# Удалить ВСЁ: контейнеры, образы, сети, кеш
docker system prune -a

# Удалить только определенные ресурсы
docker system prune          # контейнеры, сети, образы (dangling)
docker volume prune          # тома
docker network prune         # сети
```
</details>

### 🔍 Диагностика и отладка

<details>
    <summary>
            <b>Информация о системе</b>
    </summary>

```ignorelang
docker version               # версия Docker
docker info                  # общая информация
docker system df             # использование диска
```
</details>

<details>
    <summary>
        <b>Работа внутри контейнеров</b>
    </summary>

```ignorelang
# Зайти в контейнер
docker exec -it bank_app bash
docker exec -it bank_app sh

# Выполнить команду в контейнере
docker exec bank_app ls -la /app

# Просмотр процессов
docker top bank_app
```
</details>

<details>
    <summary>
        <b>Сети и порты</b>
    </summary>

```ignorelang
docker network ls            # список сетей
docker network inspect bank-rest_bank_network

# Проверить проброс портов
docker port bank_app         # порты контейнера
```
</details>

### 🛠 Управление образами

<details>
    <summary>
        <b>Сборка образов</b>
    </summary>

```ignorelang
# Собрать образ
docker build -t bank-rest .

# Собрать с другим контекстом
docker build -f Dockerfile.prod -t bank-rest:prod .

# Просмотр образов
docker images
docker image ls
```
</details>

<details>
    <summary>
        <b>Работа с реестрами</b>
    </summary>

```ignorelang
# Залить образ в registry
docker tag bank-rest myregistry/bank-rest:latest
docker push myregistry/bank-rest:latest

# Скачать образ
docker pull postgres:15
```
</details>

### 📊 Мониторинг

<details>
    <summary>
        <b>Статистика в реальном времени</b>
    </summary>

```ignorelang
docker stats                 # статистика всех контейнеров
docker stats bank_app        # статистика конкретного контейнера

# Просмотр ресурсов
docker system events         # события системы
```
</details>

<details>
    <summary>
        <b>Проверка здоровья</b>
    </summary>

```ignorelang
# Проверить здоровье контейнера
docker inspect --format='{{.State.Health.Status}}' bank_app

# Инспектировать контейнер
docker inspect bank_app
```
</details>

### 🚀 Полезные команды для разработки

<details>
    <summary>
        <b>Разработка с hot reload + резервное копирование БД</b>
    </summary>

```ignorelang
# Для разработки с монтированием кода
docker-compose -f docker-compose.dev.yml up

# Дамп базы данных
docker exec bank_postgres pg_dump -U bank_user bank_cards > backup.sql

# Восстановление
docker exec -i bank_postgres psql -U bank_user bank_cards < backup.sql
```
</details>

<details>
    <summary>
        <b>⚠️ Важные предупреждения и советы</b>
    </summary>

```ignorelang
# ОСТОРОЖНО с этими командами:

docker system prune -a      # УДАЛИТ ВСЁ НЕИСПОЛЬЗУЕМОЕ

docker rm -f $(docker ps -aq) # УДАЛИТ ВСЕ КОНТЕЙНЕРЫ

docker rmi $(docker images -q) # УДАЛИТ ВСЕ ОБРАЗЫ

# Перед удалением всегда проверяйте:

docker ps -a                # что удаляете

docker images               # какие образы
```

- Всегда используйте `docker-compose down` перед пересборкой

- Проверяйте логи когда что-то не работает

- Используйте `-f` флаг для следования за логами

- Регулярно чистите систему, чтобы освободить место
</details>
