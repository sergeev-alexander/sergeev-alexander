# Redis

## Содержание

1. [Теория и Архитектура](#1-теория-и-архитектура)
2. [Основные Концепции](#2-основные-концепции)
3. [Структуры Данных](#3-структуры-данных)
4. [CRUD Операции](#4-crud-операции)
5. [Расширенные Возможности](#5-расширенные-возможности)
6. [Персистентность и Репликация](#6-персистентность-и-репликация)
7. [Производительность и Best Practices](#7-производительность-и-best-practices)
8. [Java Интеграция](#8-java-интеграция)
9. [Тестирование](#9-тестирование)

---

## 1. Теория и Архитектура

### Что такое Redis?

Redis (Remote Dictionary Server) — in-memory хранилище данных типа «ключ-значение», которое может использоваться как база данных, 
кэш или брокер сообщений.

### Ключевые особенности:

- **In-Memory** — все данные хранятся в оперативной памяти (скорость до 100,000+ операций/сек)
- **Single-threaded** — однопоточная архитектура для операций с данными (нет race conditions)
- **Множество структур данных** — Strings, Hashes, Lists, Sets, Sorted Sets, Streams, и др.
- **Персистентность** — опциональное сохранение на диск (RDB/AOF)
- **Pub/Sub** — встроенная система публикации/подписки
- **Redis Cluster** — горизонтальное масштабирование через шардирование

### Архитектурные уровни:

| Уровень  | Описание                                       |
|:---------|------------------------------------------------|
| Key      | Уникальный идентификатор (строка)              |
| Value    | Данные любой поддерживаемой структуры          |
| Database | Логическое разделение (0-15 по умолчанию)      |
| Cluster  | Группа нод для горизонтального масштабирования |

### Архитектура Redis (упрощённо):

```text
    ┌──────────────────────────────────────────────────────────┐
    │                    Redis Server                          │
    │  ┌────────────────────────────────────────────────────┐  │
    │  │              Event Loop (Single Thread)            │  │
    │  │  ┌──────────┐ ┌────────────┐  ┌─────────────────┐  │  │
    │  │  │ Command  │ │    I/O     │  │   Timer Events  │  │  │
    │  │  │ Processor│ │ Multiplexer│  │  (expire, etc)  │  │  │
    │  │  └──────────┘ └────────────┘  └─────────────────┘  │  │
    │  └────────────────────────────────────────────────────┘  │
    │   ┌─────────────────┐  ┌─────────────────────────────┐   │
    │   │   In-Memory     │  │   Persistence Layer         │   │
    │   │   Data Store    │  │   (RDB / AOF)               │   │
    │   └─────────────────┘  └─────────────────────────────┘   │
    └──────────────────────────────────────────────────────────┘
```

### Почему однопоточная архитектура?

1. **Нет накладных расходов на синхронизацию** — не нужны locks, mutexes
2. **Предсказуемая производительность** — нет context switching между потоками
3. **Проще отладка** — последовательное выполнение команд
4. **I/O multiplexing** — epoll/kqueue обрабатывают множество подключений

Важно: Redis 6.0+ добавил многопоточность ТОЛЬКО для сетевого I/O (чтение/запись сокета), но обработка команд остаётся однопоточной.

---

### Redis Java Client (современный):

#### Lettuce (рекомендуется, используется в Spring Data Redis):

```xml
    <dependency>
        <groupId>io.lettuce</groupId>
        <artifactId>lettuce-core</artifactId>
        <version>6.5.0.RELEASE</version>
    </dependency>
```

#### Jedis (классический):

```xml
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>5.2.0</version>
    </dependency>
```

#### Spring Data Redis:

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
```

---

## 2. Основные Концепции

### Ключи (Keys)

- Строковые идентификаторы (до 512 MB)
- Рекомендация: используйте префиксы для namespaces
- Пример: `user:1001:profile`, `cache:product:5567`

### Значения (Values)

Redis поддерживает несколько типов данных:

| Тип         | Описание                              | Пример использования                |
|:------------|---------------------------------------|-------------------------------------|
| String      | Бинарно-безопасная строка (до 512 MB) | Кэш, счётчики, сессии               |
| Hash        | Поле-значение внутри ключа            | Объекты (пользователи, товары)      |
| List        | Упорядоченный список строк            | Очереди, ленты активности           |
| Set         | Неупорядоченное множество уникальных  | Теги, друзья, уникальные посетители |
| Sorted Set  | Set с числовым score для сортировки   | Рейтинги, leaderboards              |
| Stream      | Поток событий с consumer groups       | Event sourcing, логи                |
| Bitmap      | Битовые массивы                       | Флаги, онлайн-статус                |
| HyperLogLog | probabilistic структура для подсчёта  | Уникальные посетители (UV)          |
| Geospatial  | Геоданные с координатами              | Поиск поблизости                    |

### Время жизни (TTL — Time To Live)

- Любой ключ может иметь время жизни
- По истечении TTL ключ автоматически удаляется
- Используется для кэширования, сессий, временных данных

### Базы данных (Logical Databases)

> "Логические базы данных" в Redis означают, что внутри одного экземпляра Redis (одного процесса, занимающего один порт, 
> например 6379) данные разделяются по именованным пространствам (namespaces) с числовыми индексами, 
> но при этом физически они хранятся в одной и той же оперативной памяти и используют одни и те же RDB/AOF файлы для персистентности.

- Redis по умолчанию имеет 16 логических БД (0-15)
  
  **Изоляция, но не полная**: Ключи в `БД 0` не видны в `БД 1` (нельзя получить ключ из другой БД без `SELECT`), но операции, 
  затрагивающие весь сервер (например, `FLUSHALL`), очищают все логические БД.

- Переключение: `SELECT <db_number>`

  Производительность: Переключение между БД через `SELECT` — это операция на стороне клиента (переключение контекста), 
  которая не создает накладных расходов на память или процессор, сравнимых с переключением между отдельными физическими серверами.

- Важно: в Redis Cluster поддерживается только БД 0

  В Redis Cluster данные шардируются (распределяются) по хеш-слотам между узлами. Если бы поддерживались разные логические БД, 
  это сильно усложнило бы логику маршрутизации запросов между нодами кластера и механизмы репликации. 
  
  Поэтому в кластерном режиме разработчики Redis отказались от этой возможности в пользу многопоточности и изоляции через отдельные инстансы.

### События истечения (Keyspace Notifications)

- Redis может публиковать события при изменении/удалении ключей
- Включается в redis.conf: notify-keyspace-events Ex

---

### Основные клиенты для Redis 

1. Jedis — простой, синхронный, блокирующий
   
- Стиль: прямое отображение Redis-команд в методы
- Потокобезопасность: не потокобезопасен (требуется пул соединений)
- Когда использовать: простые приложения, монолитная архитектура, где важна простота и понятность кода
- Статус: активно поддерживается, но считается более "старым" подходом

2. Lettuce — асинхронный, неблокирующий, Netty-based
   
- Стиль: реактивный (async/ reactive streams), поддерживает синхронный API как обертку
- Потокобезопасность: потокобезопасен, одно соединение на весь приложение
- Когда использовать: микросервисная архитектура, high-load системы, Spring Boot 2+ (используется по умолчанию)
- Особенность: встроенная поддержка Redis Cluster, Sentinel, Master/Replica

3. Redisson — высокоуровневый, объектно-ориентированный
   
- Стиль: Java-коллекции и примитивы поверх Redis (RMap, RAtomicLong, RLock)
- Особенность: не просто клиент, а распределенная Java-платформа
- Когда использовать: нужны распределенные структуры данных (Lock, Semaphore, Queue), сложные сценарии координации
- Ключевая фича: реализация Redlock для распределенных блокировок

4. RedisTemplate — это НЕ отдельный клиент, а абстракция Spring Data Redis

- Суть: унифицированный шаблон (как JdbcTemplate), который может использовать Jedis или Lettuce как драйвер
- Композиция: `Spring Data Redis` → `RedisTemplate` → `LettuceConnectionFactory`/`JedisConnectionFactory` → `фактический клиент`
- Когда использовать: когда вы работаете в экосистеме Spring (Spring Boot, Spring Cache)

> RedisTemplate — это универсальный Spring-API, который скрывает разницу между Jedis и Lettuce; 
> код на RedisTemplate будет идентичным при смене клиента, меняется только конфигурация ConnectionFactory. 
> Redisson — отдельный клиент со своим API, не использующий RedisTemplate.

## 3. Структуры Данных

### String (Строки)

Базовый тип, бинарно-безопасный (может хранить любые данные, включая изображения).

#### Основные операции:

| Redis CLI                     | Описание операции                    | RedisTemplate                                                                                                   | Redisson                                                                                                                                       |
|:------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| SET key value                 | Установить значение                  | redisTemplate.opsForValue().set("key", "value")                                                                 | redisson.getBucket("key").set("value")                                                                                                         |
| GET key                       | Получить значение                    | redisTemplate.opsForValue().get("key")                                                                          | redisson.getBucket("key").get()                                                                                                                |
| GETSET key value              | Установить и вернуть старое значение | redisTemplate.opsForValue().getAndSet("key", "value")                                                           | redisson.getBucket("key").getAndSet("value")                                                                                                   |
| MSET key1 value1 key2 value2  | Установить несколько                 | Map<String, String> map = new HashMap<>(); map.put("key1", "value1"); redisTemplate.opsForValue().multiSet(map) | RBatch batch = redisson.createBatch(); batch.getBucket("key1").setAsync("value1"); batch.getBucket("key2").setAsync("value2"); batch.execute() |
| MGET key1 key2                | Получить несколько                   | redisTemplate.opsForValue().multiGet(Arrays.asList("key1", "key2"))                                             | RBatch batch = redisson.createBatch(); batch.getBucket("key1").getAsync(); batch.getBucket("key2").getAsync(); batch.execute()                 |
| INCR key                      | Инкремент на 1                       | redisTemplate.opsForValue().increment("key")                                                                    | redisson.getAtomicLong("key").incrementAndGet()                                                                                                |
| INCRBY key increment          | Инкремент на N                       | redisTemplate.opsForValue().increment("key", delta)                                                             | redisson.getAtomicLong("key").addAndGet(delta)                                                                                                 |
| DECR key                      | Декремент на 1                       | redisTemplate.opsForValue().decrement("key")                                                                    | redisson.getAtomicLong("key").decrementAndGet()                                                                                                |
| APPEND key value              | Добавить к строке                    | redisTemplate.opsForValue().append("key", "value")                                                              | redisson.getBucket("key").append("value")                                                                                                      |
| STRLEN key                    | Длина строки                         | redisTemplate.opsForValue().size("key")                                                                         | redisson.getBucket("key").size()                                                                                                               |
| SETEX key seconds value       | Установить с TTL (сек)               | redisTemplate.opsForValue().set("key", "value", seconds, TimeUnit.SECONDS)                                      | redisson.getBucket("key").set("value", seconds, TimeUnit.SECONDS)                                                                              |
| SETNX key value               | Установить если не существует        | redisTemplate.opsForValue().setIfAbsent("key", "value")                                                         | redisson.getBucket("key").trySet("value")                                                                                                      |
| GETRANGE key start end        | Получить подстроку                   | redisTemplate.opsForValue().get("key", start, end)                                                              | redisson.getBucket("key").get().substring(start, end)                                                                                          |
| SETRANGE key offset value     | Перезаписать часть строки            | redisTemplate.opsForValue().set("key", "value", offset)                                                         | Не поддерживается напрямую, использовать Lua script                                                                                            |
| GETDEL key                    | Получить и удалить                   | redisTemplate.opsForValue().getAndDelete("key")                                                                 | redisson.getBucket("key").getAndDelete()                                                                                                       |
| GETEX key EX seconds          | Получить и установить TTL            | redisTemplate.opsForValue().getAndExpire("key", seconds, TimeUnit.SECONDS)                                      | redisson.getBucket("key").getAndExpire(seconds, TimeUnit.SECONDS)                                                                              |
| INCRBYFLOAT key increment     | Инкремент на float                   | redisTemplate.opsForValue().increment("key", delta)                                                             | redisson.getAtomicDouble("key").addAndGet(delta)                                                                                               |
| PSETEX key milliseconds value | Установить с TTL (мс)                | redisTemplate.opsForValue().set("key", "value", milliseconds, TimeUnit.MILLISECONDS)                            | redisson.getBucket("key").set("value", milliseconds, TimeUnit.MILLISECONDS)                                                                    |

---

## Дополнительные операции с TTL

| Redis CLI                | Описание операции              | RedisTemplate                                                    | Redisson                                                              |
|:-------------------------|--------------------------------|------------------------------------------------------------------|-----------------------------------------------------------------------|
| EXPIRE key seconds       | Установить TTL (сек)           | redisTemplate.expire("key", seconds, TimeUnit.SECONDS)           | redisson.getBucket("key").expire(seconds, TimeUnit.SECONDS)           |
| PEXPIRE key milliseconds | Установить TTL (мс)            | redisTemplate.expire("key", milliseconds, TimeUnit.MILLISECONDS) | redisson.getBucket("key").expire(milliseconds, TimeUnit.MILLISECONDS) |
| TTL key                  | Оставшееся время (сек)         | redisTemplate.getExpire("key", TimeUnit.SECONDS)                 | redisson.getBucket("key").remainTimeToLive()                          |
| PTTL key                 | Оставшееся время (мс)          | redisTemplate.getExpire("key", TimeUnit.MILLISECONDS)            | redisson.getBucket("key").remainTimeToLive()                          |
| PERSIST key              | Убрать TTL                     | redisTemplate.persist("key")                                     | redisson.getBucket("key").clearExpire()                               |
| EXPIREAT key timestamp   | Установить expire по timestamp | redisTemplate.expireAt("key", date)                              | redisson.getBucket("key").expireAt(date)                              |

---

## Операции с несколькими ключами

| Redis CLI                 | Описание операции       | RedisTemplate                                                          | Redisson                                                                 |
|:--------------------------|-------------------------|------------------------------------------------------------------------|--------------------------------------------------------------------------|
| DEL key1 key2             | Удалить ключи           | redisTemplate.delete(Arrays.asList("key1", "key2"))                    | redisson.getBucket("key1").delete(); redisson.getBucket("key2").delete() |
| EXISTS key1 key2          | Проверить существование | redisTemplate.hasKey("key")                                            | redisson.getBucket("key").isExists()                                     |
| KEYS pattern              | Найти по паттерну       | redisTemplate.keys("pattern")                                          | Не рекомендуется, использовать scan                                      |
| SCAN cursor MATCH pattern | Итеративный поиск       | redisTemplate.scan(ScanOptions.scanOptions().match("pattern").build()) | redisson.getKeys().getKeysStreamByPattern("pattern")                     |
| RENAME key newKey         | Переименовать ключ      | redisTemplate.rename("key", "newKey")                                  | redisson.getKeys().rename("key", "newKey")                               |
| TYPE key                  | Получить тип ключа      | redisTemplate.type("key")                                              | redisson.getKeys().getType("key")                                        |
| MOVE key db               | Переместить в другую БД | redisTemplate.move("key", dbIndex)                                     | Не поддерживается в кластере                                             |
| COPY source destination   | Копировать ключ         | redisTemplate.copy("source", "destination", false)                     | Не поддерживается напрямую                                               |

---

## Примечания по использованию

### RedisTemplate:

- `opsForValue()` — операции со String
- `opsForHash()` — операции с Hash
- `opsForList()` — операции с List
- `opsForSet()` — операции с Set
- `opsForZSet()` — операции с Sorted Set
- `opsForGeo()` — операции с Geospatial
- `opsForHyperLogLog()` — операции с HyperLogLog

### Redisson:

- `getBucket()` — простые значения (String)
- `getMap()` — Hash
- `getList()` — List
- `getSet()` — Set
- `getSortedSet()` — Sorted Set
- `getAtomicLong()` — атомарные счётчики
- `getKeys()` — операции с ключами
- `createBatch()` — батч-операции

### Важные отличия:

1. RedisTemplate возвращает null при отсутствии ключа, Redisson может возвращать default значения
2. Redisson имеет более богатый API для распределённых структур (Lock, Semaphore, etc.)
3. RedisTemplate ближе к нативным Redis командам
4. Redisson лучше подходит для распределённых блокировок и коллекций

#### Пример использования (счётчики):

```redis
    // Инициализация счётчика
    SET page:views:12345 0

    // Инкремент при каждом просмотре
    INCR page:views:12345

    // Получить текущее значение
    GET page:views:12345
```

#### Атомарные операции:

`INCR` / `DECR` — атомарны, безопасны для конкурентного доступа
`GETSET` — атомарно, полезно для распределённых locks

### Hash (Хеши)

Идеально для хранения объектов с полями.

#### Основные операции:

| Redis CLI                        | Описание операции                      | RedisTemplate                                                                                                           | Redisson                                                                                                                                      |
|:---------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| HSET key field value             | Установить поле                        | redisTemplate.opsForHash().put("key", "field", "value")                                                                 | redisson.getMap("key").put("field", "value")                                                                                                  |
| HGET key field                   | Получить поле                          | redisTemplate.opsForHash().get("key", "field")                                                                          | redisson.getMap("key").get("field")                                                                                                           |
| HMSET key field1 value1 ...      | Установить несколько полей             | Map<String, Object> map = new HashMap<>(); <br/>map.put("f1", "v1"); <br/>redisTemplate.opsForHash().putAll("key", map) | redisson.getMap("key").putAll(map)                                                                                                            |
| HMGET key field1 field2 ...      | Получить несколько полей               | redisTemplate.opsForHash().multiGet("key", Arrays.asList("f1", "f2"))                                                   | RBatch batch = redisson.createBatch(); <br/>batch.getMap("key").getAsync("f1"); <br/>batch.getMap("key").getAsync("f2"); <br/>batch.execute() |
| HGETALL key                      | Получить все поля                      | redisTemplate.opsForHash().entries("key")                                                                               | redisson.getMap("key").readAllMap()                                                                                                           |
| HDEL key field                   | Удалить поле                           | redisTemplate.opsForHash().delete("key", "field")                                                                       | redisson.getMap("key").remove("field")                                                                                                        |
| HEXISTS key field                | Проверить существование поля           | redisTemplate.opsForHash().hasKey("key", "field")                                                                       | redisson.getMap("key").containsKey("field")                                                                                                   |
| HLEN key                         | Количество полей                       | redisTemplate.opsForHash().size("key")                                                                                  | redisson.getMap("key").size()                                                                                                                 |
| HINCRBY key field increment      | Инкремент поля                         | redisTemplate.opsForHash().increment("key", "field", delta)                                                             | redisson.getMap("key").addAndGet("field", delta)                                                                                              |
| HSETNX key field value           | Установить поле если не существует     | redisTemplate.opsForHash().putIfAbsent("key", "field", "value")                                                         | redisson.getMap("key").putIfAbsent("field", "value")                                                                                          |
| HVALS key                        | Получить все значения                  | redisTemplate.opsForHash().values("key")                                                                                | redisson.getMap("key").readAllValues()                                                                                                        |
| HKEYS key                        | Получить все ключи полей               | redisTemplate.opsForHash().keys("key")                                                                                  | redisson.getMap("key").readAllKeySet()                                                                                                        |
| HINCRBYFLOAT key field increment | Инкремент поля на float                | redisTemplate.opsForHash().increment("key", "field", delta)                                                             | redisson.getMap("key").addAndGet("field", delta)                                                                                              |
| HSTRLEN key field                | Длина значения поля                    | redisTemplate.opsForHash().size("key", "field")                                                                         | Не поддерживается напрямую, использовать Lua script                                                                                           |
| HRANDFIELD key count             | Случайные поля                         | Не поддерживается напрямую                                                                                              | Не поддерживается напрямую                                                                                                                    |
| HSCAN key cursor                 | Итеративный проход по полям            | redisTemplate.opsForHash().scan("key", ScanOptions.scanOptions().build())                                               | redisson.getMap("key").entrySet().parallelStream()                                                                                            |
| HSET key field value XX          | Установить только если поле существует | redisTemplate.opsForHash().putIfExists("key", "field", "value")                                                         | Не поддерживается напрямую                                                                                                                    |

---

## Примеры использования Hash операций

### RedisTemplate примеры:

```java
    // Создание хеша (пользователь)
    redisTemplate.opsForHash().put("user:1001", "name", "Alex");
    redisTemplate.opsForHash().put("user:1001", "email", "alex@example.com");
    redisTemplate.opsForHash().put("user:1001", "age", "30");

    // Установка нескольких полей
    Map<String, Object> userMap = new HashMap<>();
    userMap.put("name", "Alex");
    userMap.put("email", "alex@example.com");
    userMap.put("age", 30);
    redisTemplate.opsForHash().putAll("user:1001", userMap);

    // Получение одного поля
    String name = (String) redisTemplate.opsForHash().get("user:1001", "name");

    // Получение нескольких полей
    List<Object> values = redisTemplate.opsForHash().multiGet("user:1001", 
        Arrays.asList("name", "email", "age"));

    // Получение всех полей
    Map<Object, Object> allFields = redisTemplate.opsForHash().entries("user:1001");

    // Проверка существования поля
    Boolean exists = redisTemplate.opsForHash().hasKey("user:1001", "email");

    // Удаление поля
    redisTemplate.opsForHash().delete("user:1001", "age");

    // Инкремент числового поля
    redisTemplate.opsForHash().increment("user:1001", "loginCount", 1);

    // Количество полей
    Long size = redisTemplate.opsForHash().size("user:1001");
```

### Redisson примеры:

```java
    // Создание хеша (пользователь)
    RMap<String, Object> userMap = redisson.getMap("user:1001");
    userMap.put("name", "Alex");
    userMap.put("email", "alex@example.com");
    userMap.put("age", 30);

    // Получение одного поля
    String name = (String) userMap.get("name");

    // Получение всех полей
    Map<String, Object> allFields = userMap.readAllMap();

    // Проверка существования поля
    boolean exists = userMap.containsKey("email");

    // Удаление поля
    userMap.remove("age");

    // Инкремент числового поля
    userMap.addAndGet("loginCount", 1);

    // Количество полей
    int size = userMap.size();

    // Асинхронные операции
    RFuture<Object> future = userMap.getAsync("name");
    future.thenAccept(name -> System.out.println("Name: " + name));
```

---

## Паттерны использования Hash в Redis

### Хранение объектов:

```redis
    // Вместо JSON строки (String)
    SET user:1001 "{\"name\":\"Alex\",\"age\":30}"

    // Используем Hash для доступа к отдельным полям
    HSET user:1001 name "Alex"
    HSET user:1001 age 30
    HGET user:1001 name    // Не нужно десериализовать весь объект
```

### Счётчики внутри объекта:

```redis
    // Счётчик просмотров статьи
    HINCRBY article:123 views 1
    HINCRBY article:123 likes 1

    // Получить статистику
    HMGET article:123 views likes comments
```

### Корзина покупок:

```redis
    // Добавить товар
    HSET cart:user:1001 product:5567 2    // 2 штуки

    // Обновить количество
    HINCRBY cart:user:1001 product:5567 1

    // Получить все товары
    HGETALL cart:user:1001

    // Удалить товар
    HDEL cart:user:1001 product:5567
```

### Кэширование с частичным обновлением:

```redis
    // Кэш пользователя с TTL
    HSET user:cache:1001 name "Alex" email "alex@example.com"
    EXPIRE user:cache:1001 3600    // 1 час

    // Обновить только одно поле без перезаписи всего
    HSET user:cache:1001 lastLogin "2026-03-24T10:00:00Z"
```

---

## Сравнение Hash vs String (JSON) для хранения объектов

| Критерий                     | Hash                               | String (JSON)                               |
|:-----------------------------|------------------------------------|---------------------------------------------|
| Обновление отдельных полей   | Да, без чтения всего объекта       | Нет, нужно читать-модифицировать-записывать |
| Память                       | Эффективнее для небольших объектов | Больше overhead                             |
| Сериализация                 | Не требуется                       | Требуется JSON сериализация                 |
| Сложные вложенные структуры  | Не поддерживается                  | Поддерживается                              |
| Атомарность операций на поле | Да                                 | Нет                                         |
| Query по полям               | Нет                                | Нет (нужен индекс)                          |
| Размер объекта               | До 2^32 полей                      | До 512 MB                                   |

### Рекомендации:

Использовать Hash когда:

- Объект имеет плоскую структуру
- Нужно обновлять отдельные поля
- Важна атомарность операций на поле
- Объект небольшого размера (< 100 полей)

Использовать String (JSON) когда:

- Сложная вложенная структура
- Объект читается/записывается целиком
- Проще сериализация в приложении
- Нужна совместимость с другими системами

---

## Best Practices для Hash

1. Используйте префиксы для ключей: `user:1001:profile`, не `user1001profile`
2. Избегайте Hash с > 1000 полей — рассмотрите разделение на несколько ключей
3. Для счётчиков внутри Hash используйте `HINCRBY` (атомарно)
4. Используйте `HMSET` / `HMGET` для групповых операций (меньше round-trips)
5. Устанавливайте TTL на ключ Hash если это кэш
6. Мониторьте размер Hash через `MEMORY USAGE key`
7. Для больших Hash используйте `HSCAN` вместо `HGETALL`

#### Пример (хранение пользователя):

```redis
    // Создание пользователя
    HSET user:1001 name "Alex" email "alex@example.com" age 30

    // Обновление одного поля
    HSET user:1001 lastLogin "2026-03-23T10:00:00Z"

    // Получение всех данных
    HGETALL user:1001

    // Получение конкретных полей
    HMGET user:1001 name email

    // Инкремент счётчика внутри хеша
    HINCRBY user:1001 loginCount 1
```

#### Hash vs String (JSON):

Hash преимущества:

- Можно обновлять отдельные поля без чтения всего объекта
- Эффективнее по памяти для небольших объектов
- Атомарные операции на уровне полей

String (JSON) преимущества:

- Проще сериализация/десериализация
- Лучше для сложных вложенных структур
- Можно хранить как есть из приложения

### List (Списки)

Упорядоченный список строк (реализован как linked list).

#### Основные операции:

| Redis CLI                            | Описание операции                      | RedisTemplate                                                         | Redisson                                                        |
|:-------------------------------------|----------------------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------|
| LPUSH key value                      | Добавить в начало                      | redisTemplate.opsForList().leftPush("key", "value")                   | redisson.getList("key").addFirst("value")                       |
| RPUSH key value                      | Добавить в конец                       | redisTemplate.opsForList().rightPush("key", "value")                  | redisson.getList("key").add("value")                            |
| LPUSHX key value                     | Добавить в начало если ключ существует | redisTemplate.opsForList().leftPushIfPresent("key", "value")          | Не поддерживается напрямую                                      |
| RPUSHX key value                     | Добавить в конец если ключ существует  | redisTemplate.opsForList().rightPushIfPresent("key", "value")         | Не поддерживается напрямую                                      |
| LPOP key                             | Извлечь из начала                      | redisTemplate.opsForList().leftPop("key")                             | redisson.getList("key").removeFirst()                           |
| RPOP key                             | Извлечь из конца                       | redisTemplate.opsForList().rightPop("key")                            | redisson.getList("key").removeLast()                            |
| LINDEX key index                     | Получить по индексу                    | redisTemplate.opsForList().index("key", index)                        | redisson.getList("key").get(index)                              |
| LRANGE key start stop                | Получить диапазон                      | redisTemplate.opsForList().range("key", start, stop)                  | redisson.getList("key").readRange(start, stop)                  |
| LLEN key                             | Длина списка                           | redisTemplate.opsForList().size("key")                                | redisson.getList("key").size()                                  |
| LREM key count value                 | Удалить элементы                       | redisTemplate.opsForList().remove("key", count, "value")              | redisson.getList("key").remove(Object value)                    |
| LSET key index value                 | Установить значение по индексу         | redisTemplate.opsForList().set("key", index, "value")                 | redisson.getList("key").set(index, "value")                     |
| LTRIM key start stop                 | Обрезать список                        | redisTemplate.opsForList().trim("key", start, stop)                   | Не поддерживается напрямую, использовать Lua script             |
| BLPOP key timeout                    | Blocking LPOP                          | redisTemplate.opsForList().leftPop("key", timeout, TimeUnit.SECONDS)  | redisson.getList("key").pollFirst(timeout, TimeUnit.SECONDS)    |
| BRPOP key timeout                    | Blocking RPOP                          | redisTemplate.opsForList().rightPop("key", timeout, TimeUnit.SECONDS) | redisson.getList("key").pollLast(timeout, TimeUnit.SECONDS)     |
| BLMOVE source dest LEFT LEFT         | Blocking перемещение                   | Не поддерживается напрямую                                            | Не поддерживается напрямую                                      |
| LINSERT key BEFORE/AFTER pivot value | Вставить перед/после элемента          | redisTemplate.opsForList().set("key", index, "value")                 | redisson.getList("key").add(index, "value")                     |
| LMOVE source dest LEFT LEFT          | Переместить элемент                    | Не поддерживается напрямую                                            | Не поддерживается напрямую                                      |
| RPOPLPUSH source dest                | Извлечь и добавить в другой список     | redisTemplate.opsForList().rightPopAndLeftPush("source", "dest")      | Не поддерживается напрямую                                      |
| LPUSH key v1 v2 v3                   | Добавить несколько в начало            | redisTemplate.opsForList().leftPushAll("key", "v1", "v2", "v3")       | redisson.getList("key").addAll(Arrays.asList("v1", "v2", "v3")) |
| RPUSH key v1 v2 v3                   | Добавить несколько в конец             | redisTemplate.opsForList().rightPushAll("key", "v1", "v2", "v3")      | redisson.getList("key").addAll(Arrays.asList("v1", "v2", "v3")) |

---

## Примеры использования List операций

### RedisTemplate примеры:

```java
    // Очередь задач (FIFO)
    redisTemplate.opsForList().rightPush("queue:tasks", "task_1");
    redisTemplate.opsForList().rightPush("queue:tasks", "task_2");
    redisTemplate.opsForList().rightPush("queue:tasks", "task_3");

    // Извлечение задачи
    String task = (String) redisTemplate.opsForList().leftPop("queue:tasks");

    // Blocking извлечение (ждёт 5 секунд)
    String task = (String) redisTemplate.opsForList().leftPop("queue:tasks", 5, TimeUnit.SECONDS);

    // Лента активности (последнее сверху)
    redisTemplate.opsForList().leftPush("feed:user:1001", "event_1");
    redisTemplate.opsForList().leftPush("feed:user:1001", "event_2");

    // Получить последние 10 событий
    List<Object> events = redisTemplate.opsForList().range("feed:user:1001", 0, 9);

    // Ограничить размер ленты (держать только 100)
    redisTemplate.opsForList().trim("feed:user:1001", 0, 99);

    // Получить элемент по индексу
    Object event = redisTemplate.opsForList().index("feed:user:1001", 0);

    // Длина списка
    Long size = redisTemplate.opsForList().size("feed:user:1001");

    // Добавить несколько элементов
    redisTemplate.opsForList().rightPushAll("queue:tasks", "task_4", "task_5", "task_6");

    // Удалить элементы со значением
    redisTemplate.opsForList().remove("queue:tasks", 0, "task_1");
```

### Redisson примеры:

```java
    // Очередь задач (FIFO)
    RList<String> taskQueue = redisson.getList("queue:tasks");
    taskQueue.add("task_1");
    taskQueue.add("task_2");
    taskQueue.add("task_3");

    // Извлечение задачи
    String task = taskQueue.removeFirst();

    // Blocking извлечение (ждёт 5 секунд)
    /*
    Blocking извлечение — это механизм, при котором поток-потребитель "засыпает" до появления элемента в очереди или истечения таймаута, 
    что позволяет эффективно использовать ресурсы CPU без постоянных "холостых" проверок в цикле.
     */
    String task = taskQueue.pollFirst(5, TimeUnit.SECONDS);

    // Лента активности
    RList<String> feed = redisson.getList("feed:user:1001");
    feed.addFirst("event_1");
    feed.addFirst("event_2");

    // Получить последние 10 событий
    List<String> events = feed.readRange(0, 9);

    // Получить элемент по индексу
    String event = feed.get(0);

    // Длина списка
    int size = feed.size();

    // Добавить несколько элементов
    feed.addAll(Arrays.asList("task_4", "task_5", "task_6"));

    // Удалить элемент
    feed.remove("task_1");

    // Асинхронные операции
    RFuture<String> future = taskQueue.pollFirstAsync();
    future.thenAccept(task -> System.out.println("Task: " + task));

    // Blocking очередь с таймаутом
    RFuture<String> future = taskQueue.pollFirstAsync(5, TimeUnit.SECONDS);
```

---

## Паттерны использования List в Redis

### Очередь задач (FIFO):

```redis
    // Producer
    RPUSH queue:tasks "task_1"
    RPUSH queue:tasks "task_2"

    // Consumer
    BRPOP queue:tasks 0    // 0 = ждать бесконечно
```

### Стек (LIFO):

```redis
    // Push
    LPUSH stack:ops "op_1"
    LPUSH stack:ops "op_2"

    // Pop
    LPOP stack:ops
```

### Лента активности:

```redis
    // Добавить событие (последнее сверху)
    LPUSH feed:user:1001 "event_1"
    LPUSH feed:user:1001 "event_2"

    // Получить последние 10
    LRANGE feed:user:1001 0 9

    // Ограничить размер
    LTRIM feed:user:1001 0 99
```

### Очередь с приоритетами (через Sorted Set):

```redis
    // List не поддерживает приоритеты, используйте ZADD с score
    ZADD queue:priority 1 "urgent_task"
    ZADD queue:priority 10 "normal_task"
    ZPOPMIN queue:priority    // Извлечь с высшим приоритетом
```

### Reliable Queue (с подтверждением):

```redis
    // Переместить задачу в processing
    RPOPLPUSH queue:tasks queue:processing

    // После выполнения удалить из processing
    LREM queue:processing 0 "task_1"
```

---

## Сравнение List операций

| Операция    | Сложность | Использование                                    |
|:------------|-----------|--------------------------------------------------|
| LPUSH/RPUSH | O(1)      | Добавление элементов                             |
| LPOP/RPOP   | O(1)      | Извлечение элементов                             |
| LINDEX      | O(N)      | Доступ по индексу (медленно для больших списков) |
| LRANGE      | O(S+N)    | Получение диапазона (S = start, N = количество)  |
| LLEN        | O(1)      | Получение длины                                  |
| LREM        | O(N)      | Удаление элементов                               |
| LSET        | O(N)      | Установка по индексу                             |
| LTRIM       | O(N)      | Обрезка списка                                   |
| BLPOP/BRPOP | O(1)      | Blocking операции                                |

---

## Best Practices для List

1. Используйте `LPUSH` + `BRPOP` для очередей (эффективнее)
2. Избегайте `LINDEX` для больших списков — O(N) сложность
3. Используйте `LRANGE` с небольшими диапазонами для пагинации
4. Устанавливайте максимальный размер через `LTRIM` для лент активности
5. Для приоритетных очередей используйте `Sorted Set` вместо `List`
6. Используйте `BLPOP` / `BRPOP` с таймаутом для consumer (избегайте busy waiting)
7. Для надёжных очередей используйте `RPOPLPUSH` или `Streams` с ACK
8. Мониторьте размер списка через `LLEN` или `MEMORY USAGE`
9. Избегайте списков > 10,000 элементов — рассмотрите разделение
10. Для больших очередей используйте `Redis Streams` вместо `List`

---

## List vs Streams для очередей

| Критерий                | List                  | Streams                          |
|:------------------------|-----------------------|----------------------------------|
| Простота                | Проще                 | Сложнее                          |
| Подтверждение обработки | Нет (нужен RPOPLPUSH) | Да (XACK)                        |
| Consumer Groups         | Нет                   | Да                               |
| Повторная доставка      | Нет                   | Да (pending)                     |
| История сообщений       | Нет                   | Да                               |
| Производительность      | Высокая               | Высокая                          |
| Use case                | Простые очереди       | Event sourcing, надёжные очереди |

### Рекомендации:

Использовать List когда:

- Простая очередь без подтверждений
- Не нужна история сообщений
- Не нужны consumer groups
- Максимальная простота реализации

Использовать Streams когда:

- Нужно подтверждение обработки (ACK)
- Нужны consumer groups
- Нужна история сообщений
- Нужна повторная доставка
- Event sourcing паттерн

---

## Пример: Очередь задач с RedisTemplate

```java
    @Service
    public class TaskQueueService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        private static final String QUEUE_KEY = "queue:tasks";
        
        // Добавить задачу
        public void enqueue(String taskId) {
            redisTemplate.opsForList().rightPush(QUEUE_KEY, taskId);
        }
        
        // Добавить несколько задач
        public void enqueueAll(List<String> taskIds) {
            redisTemplate.opsForList().rightPushAll(QUEUE_KEY, taskIds.toArray(new String[0]));
        }
        
        // Извлечь задачу (не блокируя)
        public String dequeue() {
            return (String) redisTemplate.opsForList().leftPop(QUEUE_KEY);
        }
        
        // Извлечь задачу с ожиданием
        public String dequeueWithTimeout(long timeout, TimeUnit unit) {
            return (String) redisTemplate.opsForList().leftPop(QUEUE_KEY, timeout, unit);
        }
        
        // Получить размер очереди
        public long getQueueSize() {
            return redisTemplate.opsForList().size(QUEUE_KEY);
        }
        
        // Получить задачи без удаления (для мониторинга)
        public List<String> peek(int count) {
            return (List<String>) redisTemplate.opsForList().range(QUEUE_KEY, 0, count - 1);
        }
        
        // Очистить очередь
        public void clear() {
            redisTemplate.delete(QUEUE_KEY);
        }
    }
```

---

## Пример: Очередь задач с Redisson

```java
    @Service
    public class TaskQueueService {
        
        @Autowired
        private RedissonClient redisson;
        
        private static final String QUEUE_KEY = "queue:tasks";
        
        // Добавить задачу
        public void enqueue(String taskId) {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            queue.add(taskId);
        }
        
        // Извлечь задачу (не блокируя)
        public String dequeue() {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            return queue.removeFirst();
        }
        
        // Извлечь задачу с ожиданием
        public String dequeueWithTimeout(long timeout, TimeUnit unit) throws InterruptedException {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            return queue.pollFirst(timeout, unit);
        }
        
        // Получить размер очереди
        public long getQueueSize() {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            return queue.size();
        }
        
        // Получить задачи без удаления
        public List<String> peek(int count) {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            return queue.readRange(0, count - 1);
        }
        
        // Асинхронное извлечение
        public RFuture<String> dequeueAsync() {
            RList<String> queue = redisson.getList(QUEUE_KEY);
            return queue.removeFirstAsync();
        }
        
        // Blocking очередь (Deque)
        public String dequeueFromBlockingQueue() throws InterruptedException {
            RBlockingQueue<String> queue = redisson.getBlockingQueue(QUEUE_KEY);
            return queue.take();  // Блокирует пока не появится элемент
        }
    }
```

---

### Set (Множества)

Неупорядоченное множество уникальных элементов.

#### Основные операции:
| Redis CLI                  | Описание операции                 | RedisTemplate                                                            | Redisson                                                          |
|:---------------------------|-----------------------------------|--------------------------------------------------------------------------|-------------------------------------------------------------------|
| SADD key member            | Добавить элемент                  | redisTemplate.opsForSet().add("key", "member")                           | redisson.getSet("key").add("member")                              |
| SADD key m1 m2 m3          | Добавить несколько элементов      | redisTemplate.opsForSet().add("key", "m1", "m2", "m3")                   | redisson.getSet("key").addAll(Arrays.asList("m1", "m2", "m3"))    |
| SREM key member            | Удалить элемент                   | redisTemplate.opsForSet().remove("key", "member")                        | redisson.getSet("key").remove("member")                           |
| SREM key m1 m2 m3          | Удалить несколько элементов       | redisTemplate.opsForSet().remove("key", "m1", "m2", "m3")                | redisson.getSet("key").removeAll(Arrays.asList("m1", "m2", "m3")) |
| SMEMBERS key               | Получить все элементы             | redisTemplate.opsForSet().members("key")                                 | redisson.getSet("key").readAll()                                  |
| SISMEMBER key member       | Проверить наличие                 | redisTemplate.opsForSet().isMember("key", "member")                      | redisson.getSet("key").contains("member")                         |
| SCARD key                  | Количество элементов              | redisTemplate.opsForSet().size("key")                                    | redisson.getSet("key").size()                                     |
| SPOP key                   | Извлечь случайный элемент         | redisTemplate.opsForSet().pop("key")                                     | redisson.getSet("key").random()                                   |
| SPOP key count             | Извлечь несколько случайных       | redisTemplate.opsForSet().pop("key", count)                              | Не поддерживается напрямую                                        |
| SRANDMEMBER key count      | Случайные элементы (без удаления) | redisTemplate.opsForSet().randomMembers("key", count)                    | redisson.getSet("key").random(count)                              |
| SINTER key1 key2           | Пересечение                       | redisTemplate.opsForSet().intersect("key1", "key2")                      | redisson.getSet("key1").readIntersection(redisson.getSet("key2")) |
| SUNION key1 key2           | Объединение                       | redisTemplate.opsForSet().union("key1", "key2")                          | redisson.getSet("key1").readUnion(redisson.getSet("key2"))        |
| SDIFF key1 key2            | Разность                          | redisTemplate.opsForSet().difference("key1", "key2")                     | redisson.getSet("key1").readDiff(redisson.getSet("key2"))         |
| SINTERSTORE dest key1 key2 | Пересечение с сохранением         | redisTemplate.opsForSet().intersectAndStore("key1", "key2", "dest")      | Не поддерживается напрямую                                        |
| SUNIONSTORE dest key1 key2 | Объединение с сохранением         | redisTemplate.opsForSet().unionAndStore("key1", "key2", "dest")          | Не поддерживается напрямую                                        |
| SDIFFSTORE dest key1 key2  | Разность с сохранением            | redisTemplate.opsForSet().differenceAndStore("key1", "key2", "dest")     | Не поддерживается напрямую                                        |
| SMOVE source dest member   | Переместить элемент               | redisTemplate.opsForSet().move("source", "member", "dest")               | Не поддерживается напрямую                                        |
| SMISMEMBER key m1 m2       | Проверить несколько элементов     | redisTemplate.opsForSet().isMember("key", Arrays.asList("m1", "m2"))     | Не поддерживается напрямую                                        |
| SSCAN key cursor           | Итеративный проход                | redisTemplate.opsForSet().scan("key", ScanOptions.scanOptions().build()) | redisson.getSet("key").stream()                                   |

---

## Примеры использования Set операций

### RedisTemplate примеры:

```java
    // Добавить теги статьи
    redisTemplate.opsForSet().add("article:123:tags", "java", "redis", "cache");

    // Проверить наличие тега
    Boolean hasJava = redisTemplate.opsForSet().isMember("article:123:tags", "java");

    // Получить все теги
    Set<Object> tags = redisTemplate.opsForSet().members("article:123:tags");

    // Количество тегов
    Long tagCount = redisTemplate.opsForSet().size("article:123:tags");

    // Удалить тег
    redisTemplate.opsForSet().remove("article:123:tags", "cache");

    // Извлечь случайный тег
    Object randomTag = redisTemplate.opsForSet().pop("article:123:tags");

    // Случайные теги (без удаления)
    List<Object> randomTags = redisTemplate.opsForSet().randomMembers("article:123:tags", 3);

    // Пересечение тегов (статьи с обоими тегами)
    Set<Object> commonTags = redisTemplate.opsForSet().intersect("article:123:tags", "article:456:tags");

    // Объединение тегов
    Set<Object> allTags = redisTemplate.opsForSet().union("article:123:tags", "article:456:tags");

    // Разность тегов
    Set<Object> diffTags = redisTemplate.opsForSet().difference("article:123:tags", "article:456:tags");
```

### Redisson примеры:

```java
    // Добавить теги статьи
    RSet<String> tags = redisson.getSet("article:123:tags");
    tags.add("java");
    tags.add("redis");
    tags.add("cache");

    // Проверить наличие тега
    boolean hasJava = tags.contains("java");

    // Получить все теги
    Collection<String> allTags = tags.readAll();

    // Количество тегов
    int tagCount = tags.size();

    // Удалить тег
    tags.remove("cache");

    // Случайный элемент
    String randomTag = tags.random();

    // Несколько случайных элементов
    List<String> randomTags = tags.random(3);

    // Пересечение
    Set<String> commonTags = tags.readIntersection(redisson.getSet("article:456:tags"));

    // Объединение
    Set<String> allTags = tags.readUnion(redisson.getSet("article:456:tags"));

    // Разность
    Set<String> diffTags = tags.readDiff(redisson.getSet("article:456:tags"));

    // Асинхронные операции
    RFuture<Boolean> future = tags.addAsync("spring");
    future.thenAccept(added -> System.out.println("Added: " + added));
```

---

## Паттерны использования Set в Redis

### Теги статьи:

```redis
    // Добавить теги
    SADD article:123:tags "java" "redis" "spring"

    // Найти статьи с определённым тегом
    SMEMBERS tag:java:articles

    // Статьи с несколькими тегами (пересечение)
    SINTER tag:java:articles tag:spring:articles
```

### Онлайн пользователи:

```redis
    // Пользователь онлайн
    SADD online:users "user_1001"
    SADD online:users "user_1002"

    // Пользователь офлайн
    SREM online:users "user_1001"

    // Количество онлайн
    SCARD online:users

    // Проверить конкретного пользователя
    SISMEMBER online:users "user_1002"
```

### Друзья/Подписчики:

```redis
    // Добавить друга
    SADD user:1001:friends "user_1002"
    SADD user:1002:friends "user_1001"

    // Получить всех друзей
    SMEMBERS user:1001:friends

    // Общие друзья
    SINTER user:1001:friends user:1002:friends

    // Друзья которые не взаимны
    SDIFF user:1001:friends user:1002:friends
```

### Блокировка пользователей (blacklist):

```redis
    // Добавить в blacklist
    SADD blacklist:ips "192.168.1.100"
    SADD blacklist:ips "192.168.1.101"

    // Проверить IP
    SISMEMBER blacklist:ips "192.168.1.100"    // 1 = заблокирован
```

### Лотерея/Розыгрыш:

```redis
    // Участники
    SADD lottery:2026 "user_1" "user_2" "user_3"

    // Выбрать победителя
    SPOP lottery:2026    // Извлечь и удалить случайного

    // Выбрать несколько победителей
    SPOP lottery:2026 3
```

---

## Сравнение Set операций

| Операция    | Сложность | Использование            |
|:------------|-----------|--------------------------|
| SADD        | O(1)      | Добавление элементов     |
| SREM        | O(1)      | Удаление элементов       |
| SISMEMBER   | O(1)      | Проверка наличия         |
| SMEMBERS    | O(N)      | Получение всех элементов |
| SCARD       | O(1)      | Получение размера        |
| SPOP        | O(1)      | Извлечение случайного    |
| SRANDMEMBER | O(N)      | Случайные элементы       |
| SINTER      | O(N*M)    | Пересечение множеств     |
| SUNION      | O(N)      | Объединение множеств     |
| SDIFF       | O(N)      | Разность множеств        |

---

## Best Practices для Set

1. Используйте Set для хранения уникальных элементов (автоматическая дедупликация)
2. `SISMEMBER` имеет O(1) сложность — эффективно для проверок наличия
3. Избегайте `SMEMBERS`, для больших Set (> 10,000 элементов) — используйте `SSCAN`
4. Для подсчёта уникальных элементов используйте `SCARD` (O(1)) вместо `SIZE` в коде
5. Используйте `SINTER` / `SUNION` / `SDIFF` для операций с множествами
6. Для больших множеств рассмотрите HyperLogLog если нужна только cardinality
7. Устанавливайте TTL на Set если это временные данные (кэш, сессии)
8. Мониторьте размер Set через `SCARD` или `MEMORY USAGE`
9. Используйте `SPOP` для случайного выбора с удалением (лотереи, розыгрыши)
10. Для геоданных используйте Geospatial вместо Set

---

## Set vs Hash для хранения коллекций

| Критерий               | Set                              | Hash                        |
|:-----------------------|----------------------------------|-----------------------------|
| Уникальность элементов | Автоматическая                   | Нет (уникальны только keys) |
| Значения элементов     | Только ключ                      | Ключ + значение             |
| Проверка наличия       | O(1) `SISMEMBER`                 | O(1) `HEXISTS`              |
| Операции с множествами | Да (`SINTER`, `SUNION`, `SDIFF`) | Нет                         |
| Хранение метаданных    | Нет                              | Да (поле = значение)        |
| Use case               | Теги, друзья, онлайн             | Объекты с полями            |

### Рекомендации:

Использовать Set когда:

- Нужна только уникальность элементов
- Нужны операции с множествами (пересечение, объединение)
- Не нужно хранить метаданные элементов
- Примеры: теги, друзья, онлайн пользователи, blacklist

Использовать Hash когда:

- Нужно хранить метаданные элементов
- Нужен доступ к отдельным полям
- Примеры: объекты пользователей, настройки, кэш объектов

---

## Пример: Сервис тегов с RedisTemplate

```java
    @Service
    public class TagService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        // Добавить теги статье
        public void addTags(String articleId, Set<String> tags) {
            String key = "article:" + articleId + ":tags";
            redisTemplate.opsForSet().add(key, tags.toArray(new String[0]));
            
            // Индекс для поиска статей по тегу
            for (String tag : tags) {
                redisTemplate.opsForSet().add("tag:" + tag + ":articles", articleId);
            }
        }
        
        // Получить теги статьи
        public Set<String> getTags(String articleId) {
            String key = "article:" + articleId + ":tags";
            Set<Object> tags = redisTemplate.opsForSet().members(key);
            return tags.stream().map(Object::toString).collect(Collectors.toSet());
        }
        
        // Удалить тег из статьи
        public void removeTag(String articleId, String tag) {
            String articleKey = "article:" + articleId + ":tags";
            String indexKey = "tag:" + tag + ":articles";
            
            redisTemplate.opsForSet().remove(articleKey, tag);
            redisTemplate.opsForSet().remove(indexKey, articleId);
        }
        
        // Найти статьи с тегом
        public Set<String> getArticlesByTag(String tag) {
            String key = "tag:" + tag + ":articles";
            Set<Object> articles = redisTemplate.opsForSet().members(key);
            return articles.stream().map(Object::toString).collect(Collectors.toSet());
        }
        
        // Найти статьи с несколькими тегами (пересечение)
        public Set<String> getArticlesByTags(Set<String> tags) {
            if (tags.isEmpty()) {
                return Collections.emptySet();
            }
            
            List<String> keys = tags.stream()
                .map(tag -> "tag:" + tag + ":articles")
                .collect(Collectors.toList());
            
            String firstKey = keys.get(0);
            String[] otherKeys = keys.subList(1, keys.size()).toArray(new String[0]);
            
            Set<Object> articles = redisTemplate.opsForSet().intersect(firstKey, otherKeys);
            return articles.stream().map(Object::toString).collect(Collectors.toSet());
        }
        
        // Проверить наличие тега
        public boolean hasTag(String articleId, String tag) {
            String key = "article:" + articleId + ":tags";
            Boolean isMember = redisTemplate.opsForSet().isMember(key, tag);
            return isMember != null && isMember;
        }
    }
```

---

## Пример: Сервис тегов с Redisson

```java
    @Service
    public class TagService {
        
        @Autowired
        private RedissonClient redisson;
        
        // Добавить теги статье
        public void addTags(String articleId, Set<String> tags) {
            RSet<String> articleTags = redisson.getSet("article:" + articleId + ":tags");
            articleTags.addAll(tags);
            
            // Индекс для поиска статей по тегу
            for (String tag : tags) {
                RSet<String> tagIndex = redisson.getSet("tag:" + tag + ":articles");
                tagIndex.add(articleId);
            }
        }
        
        // Получить теги статьи
        public Set<String> getTags(String articleId) {
            RSet<String> articleTags = redisson.getSet("article:" + articleId + ":tags");
            return articleTags.readAll();
        }
        
        // Удалить тег из статьи
        public void removeTag(String articleId, String tag) {
            RSet<String> articleTags = redisson.getSet("article:" + articleId + ":tags");
            RSet<String> tagIndex = redisson.getSet("tag:" + tag + ":articles");
            
            articleTags.remove(tag);
            tagIndex.remove(articleId);
        }
        
        // Найти статьи с тегом
        public Set<String> getArticlesByTag(String tag) {
            RSet<String> tagIndex = redisson.getSet("tag:" + tag + ":articles");
            return tagIndex.readAll();
        }
        
        // Найти статьи с несколькими тегами (пересечение)
        public Set<String> getArticlesByTags(Set<String> tags) {
            if (tags.isEmpty()) {
                return Collections.emptySet();
            }
            
            List<RSet<String>> sets = tags.stream()
                .map(tag -> redisson.getSet("tag:" + tag + ":articles"))
                .collect(Collectors.toList());
            
            RSet<String> firstSet = sets.get(0);
            Set<String> result = firstSet.readAll();
            
            for (int i = 1; i < sets.size(); i++) {
                result.retainAll(sets.get(i).readAll());
            }
            
            return result;
        }
        
        // Проверить наличие тега
        public boolean hasTag(String articleId, String tag) {
            RSet<String> articleTags = redisson.getSet("article:" + articleId + ":tags");
            return articleTags.contains(tag);
        }
        
        // Асинхронное добавление тега
        public RFuture<Boolean> addTagAsync(String articleId, String tag) {
            RSet<String> articleTags = redisson.getSet("article:" + articleId + ":tags");
            return articleTags.addAsync(tag);
        }
    }
```

---

## Пример: Онлайн пользователи с RedisTemplate

```java
    @Service
    public class OnlineUserService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        private static final String ONLINE_KEY = "online:users";
        
        // Пользователь онлайн
        public void userOnline(String userId) {
            redisTemplate.opsForSet().add(ONLINE_KEY, userId);
            // Установить TTL для ключа (автоочистка если сервис упал)
            redisTemplate.expire(ONLINE_KEY, 1, TimeUnit.HOURS);
        }
        
        // Пользователь офлайн
        public void userOffline(String userId) {
            redisTemplate.opsForSet().remove(ONLINE_KEY, userId);
        }
        
        // Проверить онлайн статус
        public boolean isOnline(String userId) {
            Boolean isMember = redisTemplate.opsForSet().isMember(ONLINE_KEY, userId);
            return isMember != null && isMember;
        }
        
        // Количество онлайн пользователей
        public long getOnlineCount() {
            Long size = redisTemplate.opsForSet().size(ONLINE_KEY);
            return size != null ? size : 0;
        }
        
        // Получить всех онлайн пользователей
        public Set<String> getOnlineUsers() {
            Set<Object> users = redisTemplate.opsForSet().members(ONLINE_KEY);
            return users.stream().map(Object::toString).collect(Collectors.toSet());
        }
        
        // Получить онлайн пользователей (пагинация)
        public List<String> getOnlineUsersPage(int page, int size) {
            Set<String> allUsers = getOnlineUsers();
            return allUsers.stream()
                .skip((long) page * size)
                .limit(size)
                .collect(Collectors.toList());
        }
    }
```

### Sorted Set (Упорядоченные множества)

Множество уникальных элементов с числовым score для сортировки.

| Redis CLI                           | Описание операции                      | RedisTemplate                                                                                   | Redisson                                                                                                       |
|:------------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| ZADD key score member               | Добавить элемент с score               | redisTemplate.opsForZSet().add("key", "member", score)                                          | redisson.getScoredSortedSet("key").add(score, "member")                                                        |
| ZADD key s1 m1 s2 m2                | Добавить несколько элементов           | Set<TypedTuple<String>> tuples = new HashSet<>(); redisTemplate.opsForZSet().add("key", tuples) | redisson.getScoredSortedSet("key").addAll(Arrays.asList(new ScoredEntry<>(s1, m1), new ScoredEntry<>(s2, m2))) |
| ZREM key member                     | Удалить элемент                        | redisTemplate.opsForZSet().remove("key", "member")                                              | redisson.getScoredSortedSet("key").remove("member")                                                            |
| ZREM key m1 m2 m3                   | Удалить несколько элементов            | redisTemplate.opsForZSet().remove("key", "m1", "m2", "m3")                                      | redisson.getScoredSortedSet("key").removeAll(Arrays.asList("m1", "m2", "m3"))                                  |
| ZRANGE key start stop               | По индексу (возрастание)               | redisTemplate.opsForZSet().range("key", start, stop)                                            | redisson.getScoredSortedSet("key").readRange(start, stop)                                                      |
| ZREVRANGE key start stop            | По индексу (убывание)                  | redisTemplate.opsForZSet().reverseRange("key", start, stop)                                     | redisson.getScoredSortedSet("key").readReverseRange(start, stop)                                               |
| ZRANGE key start stop WITHSCORES    | С score (возрастание)                  | redisTemplate.opsForZSet().rangeWithScores("key", start, stop)                                  | redisson.getScoredSortedSet("key").readAllWithScores()                                                         |
| ZREVRANGE key start stop WITHSCORES | С score (убывание)                     | redisTemplate.opsForZSet().reverseRangeWithScores("key", start, stop)                           | redisson.getScoredSortedSet("key").readAllWithScores()                                                         |
| ZRANK key member                    | Ранг элемента (возрастание)            | redisTemplate.opsForZSet().rank("key", "member")                                                | redisson.getScoredSortedSet("key").rank("member")                                                              |
| ZREVRANK key member                 | Ранг элемента (убывание)               | redisTemplate.opsForZSet().reverseRank("key", "member")                                         | redisson.getScoredSortedSet("key").revRank("member")                                                           |
| ZSCORE key member                   | Получить score                         | redisTemplate.opsForZSet().score("key", "member")                                               | redisson.getScoredSortedSet("key").getScore("member")                                                          |
| ZINCRBY key increment member        | Инкремент score                        | redisTemplate.opsForZSet().incrementScore("key", "member", delta)                               | redisson.getScoredSortedSet("key").addAndGetScore("member", delta)                                             |
| ZCARD key                           | Количество элементов                   | redisTemplate.opsForZSet().size("key")                                                          | redisson.getScoredSortedSet("key").size()                                                                      |
| ZCOUNT key min max                  | Count по диапазону score               | redisTemplate.opsForZSet().count("key", min, max)                                               | redisson.getScoredSortedSet("key").countScores(min, max)                                                       |
| ZREM RANGE BY SCORE key min max     | Удалить по диапазону score             | redisTemplate.opsForZSet().removeRange("key", min, max)                                         | redisson.getScoredSortedSet("key").removeRangeByScore(min, max)                                                |
| ZREM RANGE BY RANK key start stop   | Удалить по диапазону rank              | redisTemplate.opsForZSet().removeRange("key", start, stop)                                      | redisson.getScoredSortedSet("key").removeRange(start, stop)                                                    |
| ZRANGEBYSCORE key min max           | По диапазону score                     | redisTemplate.opsForZSet().rangeByScore("key", min, max)                                        | redisson.getScoredSortedSet("key").readRangeByScore(min, max)                                                  |
| ZREVRANGEBYSCORE key min max        | По диапазону score (обратно)           | redisTemplate.opsForZSet().reverseRangeByScore("key", min, max)                                 | redisson.getScoredSortedSet("key").readReverseRangeByScore(min, max)                                           |
| ZPOPMIN key                         | Извлечь минимальный                    | redisTemplate.opsForZSet().popMin("key")                                                        | redisson.getScoredSortedSet("key").pollFirst()                                                                 |
| ZPOPMAX key                         | Извлечь максимальный                   | redisTemplate.opsForZSet().popMax("key")                                                        | redisson.getScoredSortedSet("key").pollLast()                                                                  |
| ZRANGESTORE dest key start stop     | Сохранить диапазон в новый key         | Не поддерживается напрямую                                                                      | Не поддерживается напрямую                                                                                     |
| ZINTERSTORE dest key1 key2          | Пересечение с сохранением              | redisTemplate.opsForZSet().intersectAndStore("key1", "key2", "dest")                            | Не поддерживается напрямую                                                                                     |
| ZUNIONSTORE dest key1 key2          | Объединение с сохранением              | redisTemplate.opsForZSet().unionAndStore("key1", "key2", "dest")                                | Не поддерживается напрямую                                                                                     |
| ZLEXCOUNT key min max               | Count по лексикографическому диапазону | redisTemplate.opsForZSet().count("key", range)                                                  | Не поддерживается напрямую                                                                                     |
| ZRANGEBYLEX key min max             | По лексикографическому диапазону       | redisTemplate.opsForZSet().rangeByLex("key", range)                                             | Не поддерживается напрямую                                                                                     |
| ZSCAN key cursor                    | Итеративный проход                     | redisTemplate.opsForZSet().scan("key", ScanOptions.scanOptions().build())                       | redisson.getScoredSortedSet("key").stream()                                                                    |

---

## Примеры использования Sorted Set операций

### RedisTemplate примеры:

```java
    // Leaderboard - добавить игроков с очками
    redisTemplate.opsForZSet().add("leaderboard:game1", "player_1", 1500);
    redisTemplate.opsForZSet().add("leaderboard:game1", "player_2", 2300);
    redisTemplate.opsForZSet().add("leaderboard:game1", "player_3", 1800);

    // Топ-10 игроков (убывание)
    Set<Object> top10 = redisTemplate.opsForZSet().reverseRange("leaderboard:game1", 0, 9);

    // Топ-10 с очками
    Set<TypedTuple<Object>> top10WithScores = redisTemplate.opsForZSet()
        .reverseRangeWithScores("leaderboard:game1", 0, 9);
    for (TypedTuple<Object> tuple : top10WithScores) {
        System.out.println(tuple.getValue() + ": " + tuple.getScore());
    }

    // Ранг игрока
    Long rank = redisTemplate.opsForZSet().rank("leaderboard:game1", "player_1");

    // Очки игрока
    Double score = redisTemplate.opsForZSet().score("leaderboard:game1", "player_1");

    // Инкремент очков
    redisTemplate.opsForZSet().incrementScore("leaderboard:game1", "player_1", 100);

    // Игроки в диапазоне очков
    Long count = redisTemplate.opsForZSet().count("leaderboard:game1", 1000, 2000);

    // Количество игроков
    Long size = redisTemplate.opsForZSet().size("leaderboard:game1");

    // Удалить игрока
    redisTemplate.opsForZSet().remove("leaderboard:game1", "player_1");

    // Извлечь игрока с минимальным score
    TypedTuple<Object> minScore = redisTemplate.opsForZSet().popMin("leaderboard:game1");
```

### Redisson примеры:

```java
    // Leaderboard - добавить игроков с очками
    RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:game1");
    leaderboard.add(1500, "player_1");
    leaderboard.add(2300, "player_2");
    leaderboard.add(1800, "player_3");

    // Топ-10 игроков (убывание)
    Collection<String> top10 = leaderboard.readReverseRange(0, 9);

    // Топ-10 с очками
    Collection<ScoredEntry<String>> top10WithScores = leaderboard.readAllWithScores();
    for (ScoredEntry<String> entry : top10WithScores) {
        System.out.println(entry.getValue() + ": " + entry.getScore());
    }

    // Ранг игрока
    Integer rank = leaderboard.rank("player_1");

    // Очки игрока
    Double score = leaderboard.getScore("player_1");

    // Инкремент очков
    leaderboard.addAndGetScore("player_1", 100);

    // Игроки в диапазоне очков
    int count = leaderboard.countScores(1000, 2000);

    // Количество игроков
    int size = leaderboard.size();

    // Удалить игрока
    leaderboard.remove("player_1");

    // Извлечь игрока с минимальным score
    String minScorePlayer = leaderboard.pollFirst();

    // Асинхронные операции
    RFuture<Boolean> future = leaderboard.addAsync(2000, "player_4");
    future.thenAccept(added -> System.out.println("Added: " + added));
```

---

## Паттерны использования Sorted Set в Redis

### Leaderboard (таблица лидеров):

```redis
    // Добавить/обновить очки игрока
    ZADD leaderboard:game1 1500 "player_1"
    ZADD leaderboard:game1 2300 "player_2"

    // Топ-10 игроков
    ZREVRANGE leaderboard:game1 0 9 WITHSCORES

    // Ранг игрока
    ZRANK leaderboard:game1 "player_1"

    // Инкремент очков
    ZINCRBY leaderboard:game1 100 "player_1"
```

### Очередь по приоритету:

```redis
    // Задачи с приоритетом (меньше = выше приоритет)
    ZADD queue:priority 1 "urgent_task"
    ZADD queue:priority 5 "normal_task"
    ZADD queue:priority 10 "low_task"

    // Извлечь задачу с высшим приоритетом
    ZPOPMIN queue:priority
```

### Отложенные задачи (Delayed Queue):

```redis
    // Задача выполнится через 5 минут (timestamp)
    ZADD queue:delayed 1711270800 "task_1"
    ZADD queue:delayed 1711271100 "task_2"

    // Получить готовые задачи (score <= current timestamp)
    ZRANGEBYSCORE queue:delayed 0 <current_timestamp>

    // После обработки удалить
    ZREM queue:delayed "task_1"
```

### Временная шкала (Timeline):

```redis
    // События с timestamp как score
    ZADD timeline:user:1001 1711270800 "event_1"
    ZADD timeline:user:1001 1711271100 "event_2"

    // Последние 10 событий
    ZREVRANGE timeline:user:1001 0 9

    // События за период
    ZRANGEBYSCORE timeline:user:1001 <start_ts> <end_ts>
```

### Rate Limiting с скользящим окном:

```redis
    // Добавить запрос с timestamp
    ZADD ratelimit:user:1001 <timestamp> <request_id>

    // Удалить старые запросы (> 1 минута)
    ZREMRANGEBYSCORE ratelimit:user:1001 0 <timestamp - 60000>

    // Посчитать количество запросов за минуту
    ZCARD ratelimit:user:1001
```

---

## Сравнение Sorted Set операций

| Операция         | Сложность    | Использование                                  |
|:-----------------|--------------|------------------------------------------------|
| ZADD             | O(log N)     | Добавление элементов                           |
| ZREM             | O(log N)     | Удаление элементов                             |
| ZRANGE/ZREVRANGE | O(log N + M) | Получение диапазона (M = количество элементов) |
| ZRANK/ZREVRANK   | O(log N)     | Получение ранга                                |
| ZSCORE           | O(1)         | Получение score                                |
| ZINCRBY          | O(log N)     | Инкремент score                                |
| ZCARD            | O(1)         | Получение размера                              |
| ZCOUNT           | O(log N)     | Count по диапазону                             |
| ZREMRANGEBYSCORE | O(log N + M) | Удаление по диапазону                          |
| ZPOPMIN/ZPOPMAX  | O(log N)     | Извлечение min/max                             |

---

## Best Practices для Sorted Set

1. Используйте Sorted Set для leaderboard и рейтингов
2. Score должен быть уникальным для каждого элемента (или используйте member как tiebreaker)
3. Для отложенных задач используйте timestamp как score
4. Избегайте `ZRANGE` с большими диапазонами — используйте пагинацию
5. `ZRANK` / `ZREVRANK` имеет O(log N) сложность — эффективно для больших Set
6. Используйте `ZPOPMIN` / `ZPOPMAX` для очередей по приоритету
7. Для rate limiting используйте `ZREMRANGEBYSCORE` для очистки старых записей
8. Мониторьте размер через `ZCARD` или `MEMORY USAGE`
9. Устанавливайте TTL на ключ если это временные данные
10. Для очень больших Sorted Set (> 100,000 элементов) рассмотрите шардирование

---

## Sorted Set vs List для очередей

| Критерий               | Sorted Set                        | List                      |
|:-----------------------|-----------------------------------|---------------------------|
| Приоритеты             | Да (через score)                  | Нет                       |
| Отложенные задачи      | Да (timestamp как score)          | Нет                       |
| Сложность добавления   | O(log N)                          | O(1)                      |
| Сложность извлечения   | O(log N)                          | O(1)                      |
| Уникальность элементов | Автоматическая                    | Нет                       |
| Доступ по индексу      | Да                                | Да                        |
| Доступ по score        | Да                                | Нет                       |
| Use case               | Приоритетные очереди, leaderboard | Простые FIFO/LIFO очереди |

### Рекомендации:

Использовать Sorted Set когда:

- Нужны приоритеты (score)
- Нужны отложенные задачи (timestamp)
- Нужна уникальность элементов
- Нужен доступ по диапазону score
- Примеры: leaderboard, delayed queue, rate limiting

Использовать List когда:

- Простая FIFO/LIFO очередь
- Не нужны приоритеты
- Максимальная производительность
- Примеры: очередь задач, лента активности

---

### Пример: Leaderboard сервис с RedisTemplate

```java
    @Service
    public class LeaderboardService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        // Добавить/обновить очки игрока
        public void addScore(String gameId, String playerId, double score) {
            String key = "leaderboard:" + gameId;
            redisTemplate.opsForZSet().add(key, playerId, score);
        }
        
        // Инкремент очков
        public void incrementScore(String gameId, String playerId, double delta) {
            String key = "leaderboard:" + gameId;
            redisTemplate.opsForZSet().incrementScore(key, playerId, delta);
        }
        
        // Топ-N игроков
        public Set<TypedTuple<Object>> getTopPlayers(String gameId, int count) {
            String key = "leaderboard:" + gameId;
            return redisTemplate.opsForZSet().reverseRangeWithScores(key, 0, count - 1);
        }
        
        // Ранг игрока
        public Long getPlayerRank(String gameId, String playerId) {
            String key = "leaderboard:" + gameId;
            Long rank = redisTemplate.opsForZSet().rank(key, playerId);
            return rank != null ? rank + 1 : null;  // +1 для 1-based ranking
        }
        
        // Очки игрока
        public Double getPlayerScore(String gameId, String playerId) {
            String key = "leaderboard:" + gameId;
            return redisTemplate.opsForZSet().score(key, playerId);
        }
        
        // Игроки в диапазоне очков
        public Set<Object> getPlayersByScoreRange(String gameId, double min, double max) {
            String key = "leaderboard:" + gameId;
            return redisTemplate.opsForZSet().rangeByScore(key, min, max);
        }
        
        // Количество игроков в диапазоне
        public Long countPlayersByScoreRange(String gameId, double min, double max) {
            String key = "leaderboard:" + gameId;
            return redisTemplate.opsForZSet().count(key, min, max);
        }
        
        // Удалить игрока
        public void removePlayer(String gameId, String playerId) {
            String key = "leaderboard:" + gameId;
            redisTemplate.opsForZSet().remove(key, playerId);
        }
        
        // Общее количество игроков
        public Long getTotalPlayers(String gameId) {
            String key = "leaderboard:" + gameId;
            return redisTemplate.opsForZSet().size(key);
        }
    }
```
---

### Пример: Leaderboard сервис с Redisson

```java
    @Service
    public class LeaderboardService {
        
        @Autowired
        private RedissonClient redisson;
        
        // Добавить/обновить очки игрока
        public void addScore(String gameId, String playerId, double score) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            leaderboard.add(score, playerId);
        }
        
        // Инкремент очков
        public void incrementScore(String gameId, String playerId, double delta) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            leaderboard.addAndGetScore(playerId, delta);
        }
        
        // Топ-N игроков
        public Collection<ScoredEntry<String>> getTopPlayers(String gameId, int count) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            return leaderboard.readReverseRangeWithScores(0, count - 1);
        }
        
        // Ранг игрока
        public Integer getPlayerRank(String gameId, String playerId) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            Integer rank = leaderboard.rank(playerId);
            return rank != null ? rank + 1 : null;  // +1 для 1-based ranking
        }
        
        // Очки игрока
        public Double getPlayerScore(String gameId, String playerId) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            return leaderboard.getScore(playerId);
        }
        
        // Игроки в диапазоне очков
        public Collection<String> getPlayersByScoreRange(String gameId, double min, double max) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            return leaderboard.readRangeByScore(min, max);
        }
        
        // Удалить игрока
        public void removePlayer(String gameId, String playerId) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            leaderboard.remove(playerId);
        }
        
        // Общее количество игроков
        public int getTotalPlayers(String gameId) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            return leaderboard.size();
        }
        
        // Асинхронное добавление очков
        public RFuture<Boolean> addScoreAsync(String gameId, String playerId, double score) {
            RScoredSortedSet<String> leaderboard = redisson.getScoredSortedSet("leaderboard:" + gameId);
            return leaderboard.addAsync(score, playerId);
        }
    }
```

---

### Пример: Отложенная очередь (Delayed Queue) с RedisTemplate

```java
    @Service
    public class DelayedQueueService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        private static final String QUEUE_KEY = "queue:delayed";
        
        // Добавить задачу с задержкой
        public void enqueue(String taskId, long delayMs) {
            long executeAt = System.currentTimeMillis() + delayMs;
            redisTemplate.opsForZSet().add(QUEUE_KEY, taskId, executeAt);
        }
        
        // Получить готовые задачи
        public List<String> getReadyTasks() {
            long now = System.currentTimeMillis();
            return (List<String>) redisTemplate.opsForZSet()
                .rangeByScore(QUEUE_KEY, 0, now);
        }
        
        // Обработать готовые задачи
        @Scheduled(fixedRate = 1000)  // Каждую секунду
        public void processReadyTasks() {
            long now = System.currentTimeMillis();
            
            // Получить готовые задачи
            Set<String> readyTasks = redisTemplate.opsForZSet()
                .rangeByScore(QUEUE_KEY, 0, now);
            
            if (readyTasks != null && !readyTasks.isEmpty()) {
                for (String taskId : readyTasks) {
                    try {
                        // Обработать задачу
                        processTask(taskId);
                        
                        // Удалить из очереди
                        redisTemplate.opsForZSet().remove(QUEUE_KEY, taskId);
                    } catch (Exception e) {
                        // Логировать ошибку
                        log.error("Error processing task: " + taskId, e);
                    }
                }
            }
        }
        
        // Удалить задачу
        public void cancelTask(String taskId) {
            redisTemplate.opsForZSet().remove(QUEUE_KEY, taskId);
        }
        
        // Количество задач в очереди
        public long getQueueSize() {
            Long size = redisTemplate.opsForZSet().size(QUEUE_KEY);
            return size != null ? size : 0;
        }
        
        // Количество готовых задач
        public long getReadyCount() {
            long now = System.currentTimeMillis();
            Long count = redisTemplate.opsForZSet().count(QUEUE_KEY, 0, now);
            return count != null ? count : 0;
        }
        
        private void processTask(String taskId) {
            // Логика обработки задачи
        }
    }
```

---

### Пример: Rate Limiter с скользящим окном (Redisson)

```java
    @Service
    public class RateLimiterService {
        
        @Autowired
        private RedissonClient redisson;
        
        private static final long WINDOW_MS = 60000;  // 1 минута
        private static final int MAX_REQUESTS = 100;  // 100 запросов в минуту
        
        // Проверить и записать запрос
        public boolean allowRequest(String userId) {
            RScoredSortedSet<String> rateLimit = redisson.getScoredSortedSet("ratelimit:" + userId);
            long now = System.currentTimeMillis();
            long windowStart = now - WINDOW_MS;
            
            // Удалить старые запросы
            rateLimit.removeRangeByScore(0, windowStart);
            
            // Посчитать количество запросов за окно
            int count = rateLimit.countScores(windowStart, now);
            
            if (count < MAX_REQUESTS) {
                // Добавить текущий запрос
                rateLimit.add(now, UUID.randomUUID().toString());
                // Установить TTL для автоочистки
                rateLimit.expire(WINDOW_MS, TimeUnit.MILLISECONDS);
                return true;
            }
            
            return false;
        }
        
        // Получить количество оставшихся запросов
        public int getRemainingRequests(String userId) {
            RScoredSortedSet<String> rateLimit = redisson.getScoredSortedSet("ratelimit:" + userId);
            long now = System.currentTimeMillis();
            long windowStart = now - WINDOW_MS;
            
            rateLimit.removeRangeByScore(0, windowStart);
            int count = rateLimit.countScores(windowStart, now);
            
            return Math.max(0, MAX_REQUESTS - count);
        }
        
        // Получить время до следующего доступного запроса
        public long getRetryAfter(String userId) {
            RScoredSortedSet<String> rateLimit = redisson.getScoredSortedSet("ratelimit:" + userId);
            long now = System.currentTimeMillis();
            long windowStart = now - WINDOW_MS;
            
            // Получить самый старый запрос в окне
            Collection<String> oldest = rateLimit.readRangeByScore(windowStart, now, 0, 1);
            
            if (oldest.isEmpty()) {
                return 0;
            }
            
            // Получить score самого старого запроса
            Double oldestScore = rateLimit.getScore(oldest.iterator().next());
            if (oldestScore == null) {
                return 0;
            }
            
            return Math.max(0, (long) oldestScore + WINDOW_MS - now);
        }
    }
```

---

### Пример: Временная шкала событий (Timeline) с RedisTemplate

```java
    @Service
    public class TimelineService {
        
        @Autowired
        private RedisTemplate<String, String> redisTemplate;
        
        // Добавить событие
        public void addEvent(String userId, String eventData) {
            String key = "timeline:" + userId;
            long timestamp = System.currentTimeMillis();
            redisTemplate.opsForZSet().add(key, eventData, timestamp);
        }
        
        // Получить последние N событий
        public List<String> getRecentEvents(String userId, int count) {
            String key = "timeline:" + userId;
            return (List<String>) redisTemplate.opsForZSet()
                .reverseRange(key, 0, count - 1);
        }
        
        // Получить события за период
        public List<String> getEventsByPeriod(String userId, long startTime, long endTime) {
            String key = "timeline:" + userId;
            return (List<String>) redisTemplate.opsForZSet()
                .reverseRangeByScore(key, startTime, endTime);
        }
        
        // Удалить старые события (держать только последние 1000)
        public void trimTimeline(String userId) {
            String key = "timeline:" + userId;
            // Удалить всё кроме последних 1000
            redisTemplate.opsForZSet().removeRange(key, 0, -1001);
        }
        
        // Количество событий
        public long getEventCount(String userId) {
            String key = "timeline:" + userId;
            Long size = redisTemplate.opsForZSet().size(key);
            return size != null ? size : 0;
        }
        
        // Удалить все события пользователя
        public void clearTimeline(String userId) {
            String key = "timeline:" + userId;
            redisTemplate.delete(key);
        }
    }
```

---

## Stream (Потоки)

Структура для event sourcing и обработки потоков данных с consumer groups.

### Основные операции:

| Redis CLI                                   | Описание операции              | RedisTemplate                                                                                                                                                     | Redisson                                                                                           |
|:--------------------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| XADD key * field value                      | Добавить запись (авто-ID)      | redisTemplate.opsForStream().add("key", Map.of("field", "value"))                                                                                                 | redisson.getStream("key").add(Map.of("field", "value"))                                            |
| XADD key ID field value                     | Добавить запись с указанным ID | redisTemplate.opsForStream().add("key", streamId, Map.of("field", "value"))                                                                                       | redisson.getStream("key").add(streamId, Map.of("field", "value"))                                  |
| XREAD COUNT N STREAMS key ID                | Читать N записей               | StreamReadOperations ops = redisTemplate.opsForStream(); <br/>ops.read(Consumer.from("group", "consumer"), StreamOffset.create("key", ReadOffset.from("ID")))     | RStream<String> stream = redisson.getStream("key"); stream.read(10, "ID")                          |
| XREADGROUP GROUP group consumer             | Читать как consumer group      | StreamReadOperations ops = redisTemplate.opsForStream(); <br/>ops.read(Consumer.from("group", "consumer"), StreamOffset.create("key", ReadOffset.lastConsumed())) | RStream<String> stream = redisson.getStream("key"); <br/>stream.readGroup("group", "consumer", 10) |
| XACK key group ID                           | Подтвердить обработку          | redisTemplate.opsForStream().acknowledge("key", "group", streamId)                                                                                                | redisson.getStream("key").acknowledge("group", streamId)                                           |
| XPENDING key group                          | Необработанные сообщения       | redisTemplate.opsForStream().pending("key", "group")                                                                                                              | redisson.getStream("key").pending("group")                                                         |
| XDEL key ID                                 | Удалить запись                 | redisTemplate.opsForStream().delete("key", streamId)                                                                                                              | redisson.getStream("key").remove(streamId)                                                         |
| XLEN key                                    | Длина потока                   | redisTemplate.opsForStream().size("key")                                                                                                                          | redisson.getStream("key").size()                                                                   |
| XTRIM key MAXLEN count                      | Обрезать поток                 | redisTemplate.opsForStream().trim("key", count)                                                                                                                   | redisson.getStream("key").trim(count)                                                              |
| XGROUP CREATE key group ID                  | Создать consumer group         | redisTemplate.opsForStream().createGroup("key", "group")                                                                                                          | redisson.getStream("key").createGroup("group")                                                     |
| XGROUP DESTROY key group                    | Удалить consumer group         | redisTemplate.opsForStream().destroyGroup("key", "group")                                                                                                         | redisson.getStream("key").deleteGroup("group")                                                     |
| XGROUP DELCONSUMER key group consumer       | Удалить consumer               | redisTemplate.opsForStream().deleteConsumer("key", "group", "consumer")                                                                                           | redisson.getStream("key").deleteConsumer("group", "consumer")                                      |
| XINFO GROUPS key                            | Информация о группах           | redisTemplate.opsForStream().groups("key")                                                                                                                        | redisson.getStream("key").listGroups()                                                             |
| XINFO CONSUMERS key group                   | Информация о consumers         | redisTemplate.opsForStream().consumers("key", "group")                                                                                                            | redisson.getStream("key").listConsumers("group")                                                   |
| XCLAIM key group consumer minIdle ID        | Забрать сообщения              | redisTemplate.opsForStream().claim("key", "group", "consumer", minIdle, streamId)                                                                                 | redisson.getStream("key").claim("group", "consumer", minIdle, streamId)                            |
| XPENDING key group start end count          | Детали pending сообщений       | redisTemplate.opsForStream().pending("key", "group", range, count)                                                                                                | redisson.getStream("key").pending("group", start, end, count)                                      |
| XAUTOCLAIM key group consumer minIdle start | Автозабрать сообщения          | Не поддерживается напрямую                                                                                                                                        | redisson.getStream("key").autoClaim("group", "consumer", minIdle, start)                           |
| XREAD BLOCK timeout STREAMS key ID          | Blocking read                  | StreamReadOperations ops = redisTemplate.opsForStream(); <br/>ops.read(Consumer.from("group", "consumer"), StreamOffset.create("key", ReadOffset.lastConsumed())) | redisson.getStream("key").readGroup("group", "consumer", 10, 5000)                                 |

---

## Примеры использования Stream операций

### RedisTemplate примеры:

```java
    // Добавить событие в поток
    Map<String, Object> event = new HashMap<>();
    event.put("type", "login");
    event.put("userId", "1001");
    event.put("ip", "192.168.1.1");
    event.put("timestamp", String.valueOf(System.currentTimeMillis()));

    RecordId recordId = redisTemplate.opsForStream().add("events:user:1001", event);
    System.out.println("Added with ID: " + recordId.getValue());

    // Добавить с указанным ID
    StreamOffset<String> offset = StreamOffset.create("events:user:1001", ReadOffset.from("0-0"));
    RecordId recordId = redisTemplate.opsForStream().add("events:user:1001", event);

    // Создать consumer group
    redisTemplate.opsForStream().createGroup("events:user:1001", "processing-group");

    // Читать как consumer group
    Consumer consumer = Consumer.from("processing-group", "consumer-1");
    StreamOffset<String> streamOffset = StreamOffset.create("events:user:1001", ReadOffset.lastConsumed());
    StreamReadOptions options = StreamReadOptions.empty().count(10);
    
    List<MapRecord<String, Object, Object>> records = redisTemplate.opsForStream()
        .read(consumer, options, streamOffset);

    // Подтвердить обработку
    for (MapRecord<String, Object, Object> record : records) {
        // Обработать запись
        processRecord(record);
        // Подтвердить
        redisTemplate.opsForStream().acknowledge("events:user:1001", "processing-group", record.getId());
    }

    // Проверить pending сообщения
    PendingOperations pendingOps = redisTemplate.opsForStream().pending("events:user:1001", "processing-group");
    System.out.println("Pending count: " + pendingOps.getTotalPendingCount());

    // Получить длину потока
    Long size = redisTemplate.opsForStream().size("events:user:1001");

    // Обрезать поток (держать последние 1000 записей)
    redisTemplate.opsForStream().trim("events:user:1001", 1000);
```

### Redisson примеры:

```java
    // Добавить событие в поток
    RStream<String> stream = redisson.getStream("events:user:1001");
    
    Map<String, Object> event = new HashMap<>();
    event.put("type", "login");
    event.put("userId", "1001");
    event.put("ip", "192.168.1.1");
    event.put("timestamp", String.valueOf(System.currentTimeMillis()));

    String recordId = stream.add(event);
    System.out.println("Added with ID: " + recordId);

    // Создать consumer group
    stream.createGroup("processing-group");

    // Читать как consumer group
    List<Map.Entry<String, Map<String, String>>> messages = 
        stream.readGroup("processing-group", "consumer-1", 10);

    // Подтвердить обработку
    for (Map.Entry<String, Map<String, String>> message : messages) {
        // Обработать запись
        processRecord(message);
        // Подтвердить
        stream.acknowledge("processing-group", message.getKey());
    }

    // Проверить pending сообщения
    List<PendingEntry> pending = stream.pending("processing-group");
    System.out.println("Pending count: " + pending.size());

    // Получить длину потока
    int size = stream.size();

    // Обрезать поток
    stream.trim(1000);

    // Асинхронные операции
    RFuture<String> future = stream.addAsync(event);
    future.thenAccept(id -> System.out.println("Added: " + id));
```

---

## Паттерны использования Stream в Redis

### Event Sourcing:

```redis
    // Добавить событие
    XADD events:user:1001 * type "login" ip "192.168.1.1" time "2026-03-23"
    XADD events:user:1001 * type "purchase" amount "100" item "product_1"
    XADD events:user:1001 * type "logout" time "2026-03-23T12:00:00Z"

    // Читать все события пользователя
    XREAD STREAMS events:user:1001 0

    // Восстановить состояние из событий
    // Приложение читает все события и применяет их последовательно
```

### Reliable Queue с подтверждением:

```redis
    // Producer
    XADD queue:tasks * task "process_order" orderId "12345"

    // Consumer Group
    XGROUP CREATE queue:tasks task-processors 0

    // Consumer читает задачи
    XREADGROUP GROUP task-processors consumer-1 COUNT 10 STREAMS queue:tasks >

    // После обработки подтвердить
    XACK queue:tasks task-processors 1234567890-0

    // Если consumer упал, сообщения вернутся в pending
    // Другой consumer может забрать их через XCLAIM
```

### Логирование и аудит:

```redis
    // Добавить лог запись
    XADD logs:application * level "INFO" message "User logged in" userId "1001"

    // Обрезать старые логи (держать последние 10000)
    XTRIM logs:application MAXLEN 10000

    // Читать логи за период
    XREAD COUNT 100 STREAMS logs:application 1711270800-0
```

### Pub/Sub с доставкой (вместо Pub/Sub):

```redis
    // Pub/Sub не гарантирует доставку, Stream гарантирует
    XADD notifications:user:1001 * type "message" from "user_500" text "Hello!"

    // Consumer group для каждого пользователя
    XGROUP CREATE notifications:user:1001 user-1001-group 0

    // Читать уведомления
    XREADGROUP GROUP user-1001-group consumer-1 COUNT 10 STREAMS notifications:user:1001 >
```

### Обработка в реальном времени:

```redis
    // Producer добавляет данные
    XADD sensor:temperature:room1 * value "23.5" timestamp "1711270800"
    XADD sensor:temperature:room1 * value "23.7" timestamp "1711270860"

    // Consumer обрабатывает в реальном времени
    XREADGROUP GROUP sensors-group analyzer-1 BLOCK 5000 COUNT 100 STREAMS sensor:temperature:room1 >
```

---

## Сравнение Stream операций

| Операция      | Сложность | Использование                   |
|:--------------|-----------|---------------------------------|
| XADD          | O(1)      | Добавление записей              |
| XREAD         | O(N)      | Чтение записей (N = количество) |
| XREADGROUP    | O(N)      | Чтение с consumer group         |
| XACK          | O(1)      | Подтверждение обработки         |
| XDEL          | O(1)      | Удаление записи                 |
| XLEN          | O(1)      | Получение длины                 |
| XTRIM         | O(N)      | Обрезка потока                  |
| XGROUP CREATE | O(1)      | Создание группы                 |
| XPENDING      | O(N)      | Проверка pending                |
| XCLAIM        | O(log N)  | Забрать pending сообщения       |

---

## Best Practices для Stream

1. Используйте Stream для надёжных очередей с подтверждением обработки
2. Всегда создавайте Consumer Group в production
3. Подтверждайте обработку через `XACK` после успешной обработки
4. Настройте мониторинг pending сообщений через `XPENDING`
5. Используйте `XTRIM` для ограничения размера потока (экономия памяти)
6. Для event sourcing храните все события (не обрезайте)
7. Для логов и метрик используйте `XTRIM MAXLEN` для ротации
8. Настройте несколько consumers в группе для параллельной обработки
9. Используйте `XCLAIM` для обработки "зависших" сообщений от упавших consumers
10. Мониторьте lag между producer и consumer через `XINFO GROUPS`

---

## Stream vs List vs Pub/Sub для очередей

| Критерий                | Stream                           | List            | Pub/Sub               |
|:------------------------|----------------------------------|-----------------|-----------------------|
| Подтверждение обработки | Да (XACK)                        | Нет             | Нет                   |
| Consumer Groups         | Да                               | Нет             | Нет                   |
| История сообщений       | Да                               | Да              | Нет                   |
| Повторная доставка      | Да (pending)                     | Нет             | Нет                   |
| Blocking read           | Да                               | Да              | Да                    |
| Производительность      | Высокая                          | Высокая         | Очень высокая         |
| Сложность               | Средняя                          | Низкая          | Низкая                |
| Use case                | Надёжные очереди, event sourcing | Простые очереди | Real-time уведомления |

### Рекомендации:

Использовать Stream когда:

- Нужно подтверждение обработки (ACK)
- Нужны consumer groups
- Нужна история сообщений
- Нужна повторная доставка при failure
- Event sourcing паттерн

Использовать List когда:

- Простая очередь без подтверждений
- Не нужна история сообщений
- Не нужны consumer groups
- Максимальная простота реализации

Использовать Pub/Sub когда:

- Real-time уведомления
- Не важна доставка (fire-and-forget)
- Много подписчиков

---

## Пример: Event Sourcing сервис с RedisTemplate

```java
    @Service
    public class EventSourcingService {
        
        @Autowired
        private RedisTemplate<String, Object> redisTemplate;
        
        // Добавить событие
        public String addEvent(String aggregateId, String eventType, Map<String, Object> data) {
            String streamKey = "events:" + aggregateId;
            
            Map<String, Object> event = new HashMap<>();
            event.put("type", eventType);
            event.put("timestamp", String.valueOf(System.currentTimeMillis()));
            event.putAll(data);
            
            RecordId recordId = redisTemplate.opsForStream().add(streamKey, event);
            return recordId.getValue();
        }
        
        // Получить все события агрегата
        public List<MapRecord<String, Object, Object>> getEvents(String aggregateId) {
            String streamKey = "events:" + aggregateId;
            
            StreamOffset<String> offset = StreamOffset.create(streamKey, ReadOffset.from("0-0"));
            
            return redisTemplate.opsForStream().read(offset);
        }
        
        // Получить события с пагинацией
        public List<MapRecord<String, Object, Object>> getEventsPage(String aggregateId, String startId, int count) {
            String streamKey = "events:" + aggregateId;
            
            StreamOffset<String> offset = StreamOffset.create(streamKey, ReadOffset.from(startId));
            StreamReadOptions options = StreamReadOptions.empty().count(count);
            
            return redisTemplate.opsForStream().read(options, offset);
        }
        
        // Восстановить состояние агрегата из событий
        public <T> T reconstructAggregate(String aggregateId, Function<List<MapRecord<String, Object, Object>>, T> projector) {
            List<MapRecord<String, Object, Object>> events = getEvents(aggregateId);
            return projector.apply(events);
        }
        
        // Обрезать старые события (для не-critical агрегатов)
        public void trimEvents(String aggregateId, long maxEvents) {
            String streamKey = "events:" + aggregateId;
            redisTemplate.opsForStream().trim(streamKey, maxEvents);
        }
        
        // Получить количество событий
        public long getEventCount(String aggregateId) {
            String streamKey = "events:" + aggregateId;
            Long size = redisTemplate.opsForStream().size(streamKey);
            return size != null ? size : 0;
        }
    }
```
---

## Пример: Event Sourcing сервис с Redisson

```java
    @Service
    public class EventSourcingService {
        
        @Autowired
        private RedissonClient redisson;
        
        // Добавить событие
        public String addEvent(String aggregateId, String eventType, Map<String, Object> data) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            
            Map<String, Object> event = new HashMap<>();
            event.put("type", eventType);
            event.put("timestamp", String.valueOf(System.currentTimeMillis()));
            event.putAll(data);
            
            return stream.add(event);
        }
        
        // Получить все события агрегата
        public List<Map.Entry<String, Map<String, String>>> getEvents(String aggregateId) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            return stream.read(0, Integer.MAX_VALUE);
        }
        
        // Получить события с пагинацией
        public List<Map.Entry<String, Map<String, String>>> getEventsPage(String aggregateId, String startId, int count) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            return stream.read(count, startId);
        }
        
        // Обрезать старые события
        public void trimEvents(String aggregateId, long maxEvents) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            stream.trim(maxEvents);
        }
        
        // Получить количество событий
        public int getEventCount(String aggregateId) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            return stream.size();
        }
        
        // Асинхронное добавление события
        public RFuture<String> addEventAsync(String aggregateId, String eventType, Map<String, Object> data) {
            RStream<String> stream = redisson.getStream("events:" + aggregateId);
            return stream.addAsync(data);
        }
    }
```

---

## Пример: Reliable Queue с Consumer Groups (RedisTemplate)

```java
    @Service
    public class ReliableQueueService {
        
        @Autowired
        private RedisTemplate<String, Object> redisTemplate;
        
        private static final String STREAM_KEY = "queue:tasks";
        private static final String GROUP_NAME = "task-processors";
        
        // Инициализация consumer group
        @PostConstruct
        public void init() {
            try {
                redisTemplate.opsForStream().createGroup(STREAM_KEY, GROUP_NAME);
            } catch (Exception e) {
                // Группа уже существует
            }
        }
        
        // Добавить задачу в очередь
        public String enqueue(String taskId, Map<String, Object> taskData) {
            Map<String, Object> message = new HashMap<>();
            message.put("taskId", taskId);
            message.put("data", taskData);
            message.put("createdAt", String.valueOf(System.currentTimeMillis()));
            
            RecordId recordId = redisTemplate.opsForStream().add(STREAM_KEY, message);
            return recordId.getValue();
        }
        
        // Обработать задачи (consumer)
        @Scheduled(fixedRate = 1000)
        public void processTasks() {
            Consumer consumer = Consumer.from(GROUP_NAME, "consumer-" + getHostname());
            StreamOffset<String> offset = StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed());
            StreamReadOptions options = StreamReadOptions.empty().count(10);
            
            List<MapRecord<String, Object, Object>> records = redisTemplate.opsForStream()
                .read(consumer, options, offset);
            
            if (records != null) {
                for (MapRecord<String, Object, Object> record : records) {
                    try {
                        processTask(record);
                        // Подтвердить обработку
                        redisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP_NAME, record.getId());
                    } catch (Exception e) {
                        log.error("Error processing task: " + record.getId(), e);
                        // Не подтверждаем - сообщение останется в pending
                    }
                }
            }
        }
        
        // Обработать pending сообщения (recovery)
        @Scheduled(fixedRate = 30000)  // Каждые 30 секунд
        public void processPendingTasks() {
            PendingOperations pending = redisTemplate.opsForStream().pending(STREAM_KEY, GROUP_NAME);
            
            if (pending.getTotalPendingCount() > 0) {
                PendingMessagesSummary summary = redisTemplate.opsForStream()
                    .pending(STREAM_KEY, GROUP_NAME, PendingMessagesParameters.empty());
                
                for (PendingMessage message : summary) {
                    if (message.getTotalDeliveryCount() > 3) {
                        // Сообщение не обработано после 3 попыток - отправить в dead letter
                        moveToDeadLetter(message);
                        redisTemplate.opsForStream().acknowledge(STREAM_KEY, GROUP_NAME, message.getId());
                    }
                }
            }
        }
        
        private void processTask(MapRecord<String, Object, Object> record) {
            // Логика обработки задачи
        }
        
        private void moveToDeadLetter(PendingMessage message) {
            // Переместить в dead letter queue
        }
        
        private String getHostname() {
            return InetAddress.getLocalHost().getHostName();
        }
        
        // Получить статистику очереди
        public QueueStats getQueueStats() {
            QueueStats stats = new QueueStats();
            stats.setTotalSize(redisTemplate.opsForStream().size(STREAM_KEY));
            
            PendingOperations pending = redisTemplate.opsForStream().pending(STREAM_KEY, GROUP_NAME);
            stats.setPendingCount(pending.getTotalPendingCount());
            
            List<ConsumerInfo> consumers = redisTemplate.opsForStream().consumers(STREAM_KEY, GROUP_NAME);
            stats.setConsumerCount(consumers.size());
            
            return stats;
        }
    }
```

---

## Пример: Reliable Queue с Consumer Groups (Redisson)

```java
    @Service
    public class ReliableQueueService {
        
        @Autowired
        private RedissonClient redisson;
        
        private static final String STREAM_KEY = "queue:tasks";
        private static final String GROUP_NAME = "task-processors";
        
        // Инициализация consumer group
        @PostConstruct
        public void init() {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            try {
                stream.createGroup(GROUP_NAME);
            } catch (Exception e) {
                // Группа уже существует
            }
        }
        
        // Добавить задачу в очередь
        public String enqueue(String taskId, Map<String, Object> taskData) {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            
            Map<String, Object> message = new HashMap<>();
            message.put("taskId", taskId);
            message.put("data", taskData);
            message.put("createdAt", String.valueOf(System.currentTimeMillis()));
            
            return stream.add(message);
        }
        
        // Обработать задачи (consumer)
        @Scheduled(fixedRate = 1000)
        public void processTasks() {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            String consumerId = "consumer-" + getHostname();
            
            List<Map.Entry<String, Map<String, String>>> messages = 
                stream.readGroup(GROUP_NAME, consumerId, 10);
            
            if (messages != null) {
                for (Map.Entry<String, Map<String, String>> message : messages) {
                    try {
                        processTask(message);
                        // Подтвердить обработку
                        stream.acknowledge(GROUP_NAME, message.getKey());
                    } catch (Exception e) {
                        log.error("Error processing task: " + message.getKey(), e);
                        // Не подтверждаем - сообщение останется в pending
                    }
                }
            }
        }
        
        // Обработать pending сообщения (recovery)
        @Scheduled(fixedRate = 30000)
        public void processPendingTasks() {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            List<PendingEntry> pending = stream.pending(GROUP_NAME);
            
            for (PendingEntry entry : pending) {
                if (entry.getRetryCount() > 3) {
                    // Сообщение не обработано после 3 попыток
                    moveToDeadLetter(entry);
                    stream.acknowledge(GROUP_NAME, entry.getId());
                }
            }
        }
        
        // Blocking read с таймаутом
        public void processTasksBlocking() {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            String consumerId = "consumer-" + getHostname();
            
            // Блокировать до 5 секунд пока не появится сообщение
            List<Map.Entry<String, Map<String, String>>> messages = 
                stream.readGroup(GROUP_NAME, consumerId, 10, 5000);
            
            // Обработать сообщения...
        }
        
        private void processTask(Map.Entry<String, Map<String, String>> message) {
            // Логика обработки задачи
        }
        
        private void moveToDeadLetter(PendingEntry entry) {
            // Переместить в dead letter queue
        }
        
        private String getHostname() {
            return InetAddress.getLocalHost().getHostName();
        }
        
        // Получить статистику очереди
        public QueueStats getQueueStats() {
            RStream<String> stream = redisson.getStream(STREAM_KEY);
            
            QueueStats stats = new QueueStats();
            stats.setTotalSize(stream.size());
            
            List<PendingEntry> pending = stream.pending(GROUP_NAME);
            stats.setPendingCount(pending.size());
            
            List<ConsumerInfo> consumers = stream.listConsumers(GROUP_NAME);
            stats.setConsumerCount(consumers.size());
            
            return stats;
        }
    }
```

---

## Пример: Логирование с ротацией (RedisTemplate)

```java
    @Service
    public class LogStreamService {
        
        @Autowired
        private RedisTemplate<String, Object> redisTemplate;
        
        private static final long MAX_LOG_ENTRIES = 10000;
        
        // Добавить лог запись
        public void addLog(String level, String message, Map<String, Object> context) {
            String streamKey = "logs:application";
            
            Map<String, Object> logEntry = new HashMap<>();
            logEntry.put("level", level);
            logEntry.put("message", message);
            logEntry.put("timestamp", String.valueOf(System.currentTimeMillis()));
            logEntry.put("thread", Thread.currentThread().getName());
            logEntry.putAll(context);
            
            redisTemplate.opsForStream().add(streamKey, logEntry);
            
            // Обрезать старые логи
            redisTemplate.opsForStream().trim(streamKey, MAX_LOG_ENTRIES);
        }
        
        // Получить логи по уровню
        public List<MapRecord<String, Object, Object>> getLogsByLevel(String level, int count) {
            String streamKey = "logs:application";
            
            StreamOffset<String> offset = StreamOffset.create(streamKey, ReadOffset.from("0-0"));
            StreamReadOptions options = StreamReadOptions.empty().count(1000);
            
            List<MapRecord<String, Object, Object>> allLogs = redisTemplate.opsForStream().read(options, offset);
            
            return allLogs.stream()
                .filter(record -> level.equals(record.getValue().get("level")))
                .limit(count)
                .collect(Collectors.toList());
        }
        
        // Получить логи за период
        public List<MapRecord<String, Object, Object>> getLogsByPeriod(long startTime, long endTime, int count) {
            String streamKey = "logs:application";
            
            StreamOffset<String> offset = StreamOffset.create(streamKey, ReadOffset.from("0-0"));
            StreamReadOptions options = StreamReadOptions.empty().count(1000);
            
            List<MapRecord<String, Object, Object>> allLogs = redisTemplate.opsForStream().read(options, offset);
            
            return allLogs.stream()
                .filter(record -> {
                    String ts = (String) record.getValue().get("timestamp");
                    long logTime = Long.parseLong(ts);
                    return logTime >= startTime && logTime <= endTime;
                })
                .limit(count)
                .collect(Collectors.toList());
        }
        
        // Получить количество логов
        public long getLogCount() {
            String streamKey = "logs:application";
            Long size = redisTemplate.opsForStream().size(streamKey);
            return size != null ? size : 0;
        }
        
        // Очистить все логи
        public void clearLogs() {
            String streamKey = "logs:application";
            redisTemplate.delete(streamKey);
        }
    }
```

---

## Пример: Логирование с ротацией (Redisson)

```java
    @Service
    public class LogStreamService {
        
        @Autowired
        private RedissonClient redisson;
        
        private static final long MAX_LOG_ENTRIES = 10000;
        
        // Добавить лог запись
        public void addLog(String level, String message, Map<String, Object> context) {
            RStream<String> stream = redisson.getStream("logs:application");
            
            Map<String, Object> logEntry = new HashMap<>();
            logEntry.put("level", level);
            logEntry.put("message", message);
            logEntry.put("timestamp", String.valueOf(System.currentTimeMillis()));
            logEntry.put("thread", Thread.currentThread().getName());
            logEntry.putAll(context);
            
            stream.add(logEntry);
            
            // Обрезать старые логи
            stream.trim(MAX_LOG_ENTRIES);
        }
        
        // Получить логи
        public List<Map.Entry<String, Map<String, String>>> getLogs(int count) {
            RStream<String> stream = redisson.getStream("logs:application");
            return stream.read(count, "0-0");
        }
        
        // Получить количество логов
        public int getLogCount() {
            RStream<String> stream = redisson.getStream("logs:application");
            return stream.size();
        }
        
        // Очистить все логи
        public void clearLogs() {
            RStream<String> stream = redisson.getStream("logs:application");
            stream.clear();
        }
        
        // Асинхронное добавление лога
        public RFuture<String> addLogAsync(String level, String message, Map<String, Object> context) {
            RStream<String> stream = redisson.getStream("logs:application");
            
            Map<String, Object> logEntry = new HashMap<>();
            logEntry.put("level", level);
            logEntry.put("message", message);
            logEntry.put("timestamp", String.valueOf(System.currentTimeMillis()));
            logEntry.putAll(context);
            
            return stream.addAsync(logEntry);
        }
    }
```

---

## Stream ID формат

Redis Stream ID имеет формат: timestamp-ms-sequence

```text
    1711270800000-0    // Первая запись в эту миллисекунду
    1711270800000-1    // Вторая запись в эту миллисекунду
    1711270800001-0    // Первая запись в следующую миллисекунду
```

Специальные ID:

```text
    $     // Последний ID в потоке (для новых consumers)
    0     // Первый ID в потоке (читать всё)
    0-0   // Явный первый ID
    >     // Только новые сообщения (для XREADGROUP)
```

---

## Consumer Group состояния

| Состояние | Описание                                  |
|:----------|-------------------------------------------|
| Active    | Consumer активен и обрабатывает сообщения |
| Idle      | Consumer не активен более idle time       |
| Pending   | Сообщения доставлены но не подтверждены   |
| Dead      | Consumer не отвечает более timeout        |

### Мониторинг Consumer Group:

```redis
    XINFO GROUPS stream_key      // Информация о всех группах
    XINFO CONSUMERS stream_key group  // Информация о consumers в группе
    XPENDING stream_key group    // Pending сообщения
```

---

## Dead Letter Queue паттерн

```redis
    // После N попыток обработки переместить в DLQ
    XADD queue:tasks:dlq * originalId "123" reason "max_retries_exceeded" data "{...}"

    // Мониторить DLQ отдельно
    XLEN queue:tasks:dlq

    // Периодически обрабатывать DLQ вручную или автоматически
```

### Bitmap (Битовые массивы)

Эффективное хранение битовых флагов.

#### Основные операции:

```redis
    SETBIT key offset value          // Установить бит
    GETBIT key offset                // Получить бит
    BITCOUNT key                     // Посчитать установленные биты
    BITOP AND/OR/XOR dest key1 ...   // Битовые операции
    BITPOS key bit                   // Найти позицию бита
```

#### Пример (ежедневная активность):

```redis
    // Отметить день активности (день года как offset)
    SETBIT user:1001:activity 82 1    // 82-й день года

    // Проверить активность в конкретный день
    GETBIT user:1001:activity 82

    // Посчитать количество активных дней
    BITCOUNT user:1001:activity
```

#### Пример (онлайн статус по минутам):

```redis
    // Пользователь онлайн в минуту 14:30
    SETBIT online:2026-03-23 870 1    // 14*60 + 30 = 870
```

### HyperLogLog (Probabilistic подсчёт)

Эффективная структура для подсчёта уникальных элементов (ошибка ~0.81%).

#### Основные операции:

```redis
    PFADD key element                // Добавить элемент
    PFCOUNT key                      // Получить cardinality
    PFMERGE dest key1 key2           // Объединить несколько
```

#### Пример (уникальные посетители):

```redis
    // Отметить посетителя
    PFADD uv:2026-03-23 "user_1001"
    PFADD uv:2026-03-23 "user_1002"
    PFADD uv:2026-03-23 "user_1001"  // Дубликат не учитывается

    // Получить количество уникальных
    PFCOUNT uv:2026-03-23            // Вернёт 2

    // Объединить за неделю
    PFMERGE uv:week:12 uv:2026-03-17 uv:2026-03-18 ... uv:2026-03-23
    PFCOUNT uv:week:12
```

Преимущество: HyperLogLog использует ~12 KB памяти независимо от количества элементов.

### Geospatial (Геоданные)

Хранение и поиск по координатам.

#### Основные операции:

```redis
    GEOADD key longitude latitude member    // Добавить координаты
    GEOPOS key member                       // Получить координаты
    GEODIST key member1 member2             // Расстояние между
    GEORADIUS key lon lat radius unit       // Поиск в радиусе
    GEORADIUSBYMEMBER key member radius     // Поиск вокруг элемента
    GEOHASH key member                      // GeoHash строка
```

#### Пример (поиск магазинов поблизости):

```redis
    // Добавить магазины
    GEOADD shops:moscow 37.6173 55.7558 "shop_1"
    GEOADD shops:moscow 37.6250 55.7500 "shop_2"
    GEOADD shops:moscow 37.6100 55.7600 "shop_3"

    // Найти в радиусе 5 км
    GEORADIUS shops:moscow 37.6173 55.7558 5 km WITHDIST

    // Расстояние между магазинами
    GEODIST shops:moscow "shop_1" "shop_2" km
```

---

# 4. CRUD-операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| SET key value | Create | Установить значение (String) | redisTemplate.opsForValue().set("key", "value") | redisson.getBucket("key").set("value") |
| SET key value EX seconds | Create | Установить с TTL (секунды) | redisTemplate.opsForValue().set("key", "value", seconds, TimeUnit.SECONDS) | redisson.getBucket("key").set("value", seconds, TimeUnit.SECONDS) |
| SET key value PX milliseconds | Create | Установить с TTL (миллисекунды) | redisTemplate.opsForValue().set("key", "value", milliseconds, TimeUnit.MILLISECONDS) | redisson.getBucket("key").set("value", milliseconds, TimeUnit.MILLISECONDS) |
| SETEX key seconds value | Create | Атомарно SET + EXPIRE | redisTemplate.opsForValue().set("key", "value", seconds, TimeUnit.SECONDS) | redisson.getBucket("key").set("value", seconds, TimeUnit.SECONDS) |
| SETNX key value | Create | Установить если не существует | redisTemplate.opsForValue().setIfAbsent("key", "value") | redisson.getBucket("key").trySet("value") |
| MSET key1 value1 key2 value2 | Create | Установить несколько ключей | redisTemplate.opsForValue().multiSet(map) | redisson.createBatch() |
| HSET key field value | Create | Установить поле (Hash) | redisTemplate.opsForHash().put("key", "field", "value") | redisson.getMap("key").put("field", "value") |
| HMSET key field1 value1 ... | Create | Установить несколько полей | redisTemplate.opsForHash().putAll("key", map) | redisson.getMap("key").putAll(map) |
| HSETNX key field value | Create | Установить поле если не существует | redisTemplate.opsForHash().putIfAbsent("key", "field", "value") | redisson.getMap("key").putIfAbsent("field", "value") |
| LPUSH key value | Create | Добавить в начало списка | redisTemplate.opsForList().leftPush("key", "value") | redisson.getList("key").addFirst("value") |
| RPUSH key value | Create | Добавить в конец списка | redisTemplate.opsForList().rightPush("key", "value") | redisson.getList("key").add("value") |
| LPUSHX key value | Create | Добавить в начало если ключ существует | redisTemplate.opsForList().leftPushIfPresent("key", "value") | Не поддерживается |
| RPUSHX key value | Create | Добавить в конец если ключ существует | redisTemplate.opsForList().rightPushIfPresent("key", "value") | Не поддерживается |
| SADD key member | Create | Добавить элемент (Set) | redisTemplate.opsForSet().add("key", "member") | redisson.getSet("key").add("member") |
| ZADD key score member | Create | Добавить элемент с score (Sorted Set) | redisTemplate.opsForZSet().add("key", "member", score) | redisson.getScoredSortedSet("key").add(score, "member") |
| XADD key * field value | Create | Добавить запись в Stream | redisTemplate.opsForStream().add("key", map) | redisson.getStream("key").add(map) |
| GET key | Read | Получить значение (String) | redisTemplate.opsForValue().get("key") | redisson.getBucket("key").get() |
| MGET key1 key2 | Read | Получить несколько значений | redisTemplate.opsForValue().multiGet(keys) | redisson.createBatch() |
| GETRANGE key start end | Read | Получить подстроку | redisTemplate.opsForValue().get("key", start, end) | Не поддерживается напрямую |
| HGET key field | Read | Получить поле (Hash) | redisTemplate.opsForHash().get("key", "field") | redisson.getMap("key").get("field") |
| HMGET key field1 field2 | Read | Получить несколько полей | redisTemplate.opsForHash().multiGet("key", fields) | redisson.createBatch() |
| HGETALL key | Read | Получить все поля (Hash) | redisTemplate.opsForHash().entries("key") | redisson.getMap("key").readAllMap() |
| HVALS key | Read | Получить все значения (Hash) | redisTemplate.opsForHash().values("key") | redisson.getMap("key").readAllValues() |
| HKEYS key | Read | Получить все ключи полей (Hash) | redisTemplate.opsForHash().keys("key") | redisson.getMap("key").readAllKeySet() |
| HEXISTS key field | Read | Проверить существование поля | redisTemplate.opsForHash().hasKey("key", "field") | redisson.getMap("key").containsKey("field") |
| HLEN key | Read | Количество полей (Hash) | redisTemplate.opsForHash().size("key") | redisson.getMap("key").size() |
| LINDEX key index | Read | Получить элемент по индексу (List) | redisTemplate.opsForList().index("key", index) | redisson.getList("key").get(index) |
| LRANGE key start stop | Read | Получить диапазон (List) | redisTemplate.opsForList().range("key", start, stop) | redisson.getList("key").readRange(start, stop) |
| LLEN key | Read | Длина списка | redisTemplate.opsForList().size("key") | redisson.getList("key").size() |
| SMEMBERS key | Read | Получить все элементы (Set) | redisTemplate.opsForSet().members("key") | redisson.getSet("key").readAll() |
| SISMEMBER key member | Read | Проверить наличие элемента (Set) | redisTemplate.opsForSet().isMember("key", "member") | redisson.getSet("key").contains("member") |
| SCARD key | Read | Количество элементов (Set) | redisTemplate.opsForSet().size("key") | redisson.getSet("key").size() |
| SRANDMEMBER key count | Read | Случайные элементы (Set) | redisTemplate.opsForSet().randomMembers("key", count) | redisson.getSet("key").random(count) |
| ZRANGE key start stop | Read | Диапазон по индексу (Sorted Set) | redisTemplate.opsForZSet().range("key", start, stop) | redisson.getScoredSortedSet("key").readRange(start, stop) |
| ZREVRANGE key start stop | Read | Диапазон по индексу (обратно) | redisTemplate.opsForZSet().reverseRange("key", start, stop) | redisson.getScoredSortedSet("key").readReverseRange(start, stop) |
| ZRANGE key start stop WITHSCORES | Read | Диапазон с score | redisTemplate.opsForZSet().rangeWithScores("key", start, stop) | redisson.getScoredSortedSet("key").readAllWithScores() |
| ZREVRANGE key start stop WITHSCORES | Read | Диапазон с score (обратно) | redisTemplate.opsForZSet().reverseRangeWithScores("key", start, stop) | redisson.getScoredSortedSet("key").readAllWithScores() |
| ZRANK key member | Read | Ранг элемента (Sorted Set) | redisTemplate.opsForZSet().rank("key", "member") | redisson.getScoredSortedSet("key").rank("member") |
| ZREVRANK key member | Read | Ранг элемента (обратно) | redisTemplate.opsForZSet().reverseRank("key", "member") | redisson.getScoredSortedSet("key").revRank("member") |
| ZSCORE key member | Read | Получить score (Sorted Set) | redisTemplate.opsForZSet().score("key", "member") | redisson.getScoredSortedSet("key").getScore("member") |
| ZCARD key | Read | Количество элементов (Sorted Set) | redisTemplate.opsForZSet().size("key") | redisson.getScoredSortedSet("key").size() |
| ZCOUNT key min max | Read | Count по диапазону score | redisTemplate.opsForZSet().count("key", min, max) | redisson.getScoredSortedSet("key").countScores(min, max) |
| XREAD key ID | Read | Читать записи (Stream) | redisTemplate.opsForStream().read(offset) | redisson.getStream("key").read() |
| XLEN key | Read | Длина потока (Stream) | redisTemplate.opsForStream().size("key") | redisson.getStream("key").size() |
| EXISTS key | Read | Проверить существование ключа | redisTemplate.hasKey("key") | redisson.getKeys().countExists("key") |
| TYPE key | Read | Получить тип ключа | redisTemplate.type("key") | redisson.getKeys().getType("key") |
| TTL key | Read | Оставшееся время жизни (сек) | redisTemplate.getExpire("key") | redisson.getBucket("key").remainTimeToLive() |
| PTTL key | Read | Оставшееся время жизни (мс) | redisTemplate.getExpire("key", TimeUnit.MILLISECONDS) | redisson.getBucket("key").remainTimeToLive() |
| SET key value | Update | Обновить значение (String) | redisTemplate.opsForValue().set("key", "value") | redisson.getBucket("key").set("value") |
| INCR key | Update | Инкремент на 1 (String) | redisTemplate.opsForValue().increment("key") | redisson.getAtomicLong("key").incrementAndGet() |
| INCRBY key increment | Update | Инкремент на N (String) | redisTemplate.opsForValue().increment("key", delta) | redisson.getAtomicLong("key").addAndGet(delta) |
| INCRBYFLOAT key increment | Update | Инкремент на float (String) | redisTemplate.opsForValue().increment("key", delta) | redisson.getAtomicDouble("key").addAndGet(delta) |
| DECR key | Update | Декремент на 1 (String) | redisTemplate.opsForValue().decrement("key") | redisson.getAtomicLong("key").decrementAndGet() |
| DECRBY key decrement | Update | Декремент на N (String) | redisTemplate.opsForValue().decrement("key", delta) | redisson.getAtomicLong("key").addAndGet(-delta) |
| APPEND key value | Update | Добавить к строке | redisTemplate.opsForValue().append("key", "value") | redisson.getBucket("key").append("value") |
| HSET key field value | Update | Обновить поле (Hash) | redisTemplate.opsForHash().put("key", "field", "value") | redisson.getMap("key").put("field", "value") |
| HINCRBY key field increment | Update | Инкремент поля (Hash) | redisTemplate.opsForHash().increment("key", "field", delta) | redisson.getMap("key").addAndGet("field", delta) |
| HINCRBYFLOAT key field increment | Update | Инкремент поля на float (Hash) | redisTemplate.opsForHash().increment("key", "field", delta) | redisson.getMap("key").addAndGet("field", delta) |
| LSET key index value | Update | Установить по индексу (List) | redisTemplate.opsForList().set("key", index, "value") | redisson.getList("key").set(index, "value") |
| LTRIM key start stop | Update | Обрезать список | redisTemplate.opsForList().trim("key", start, stop) | Не поддерживается напрямую |
| ZINCRBY key increment member | Update | Инкремент score (Sorted Set) | redisTemplate.opsForZSet().incrementScore("key", "member", delta) | redisson.getScoredSortedSet("key").addAndGetScore("member", delta) |
| EXPIRE key seconds | Update | Установить TTL (сек) | redisTemplate.expire("key", seconds, TimeUnit.SECONDS) | redisson.getBucket("key").expire(seconds, TimeUnit.SECONDS) |
| PEXPIRE key milliseconds | Update | Установить TTL (мс) | redisTemplate.expire("key", milliseconds, TimeUnit.MILLISECONDS) | redisson.getBucket("key").expire(milliseconds, TimeUnit.MILLISECONDS) |
| PERSIST key | Update | Убрать TTL | redisTemplate.persist("key") | redisson.getBucket("key").clearExpire() |
| RENAME key newKey | Update | Переименовать ключ | redisTemplate.rename("key", "newKey") | redisson.getKeys().rename("key", "newKey") |
| DEL key1 key2 | Delete | Удалить ключи | redisTemplate.delete(keys) | redisson.getKeys().delete("key") |
| UNLINK key1 key2 | Delete | Асинхронное удаление | redisTemplate.unlink(keys) | Не поддерживается напрямую |
| HDEL key field | Delete | Удалить поле (Hash) | redisTemplate.opsForHash().delete("key", "field") | redisson.getMap("key").remove("field") |
| LPOP key | Delete | Извлечь из начала (List) | redisTemplate.opsForList().leftPop("key") | redisson.getList("key").removeFirst() |
| RPOP key | Delete | Извлечь из конца (List) | redisTemplate.opsForList().rightPop("key") | redisson.getList("key").removeLast() |
| LREM key count value | Delete | Удалить элементы по значению (List) | redisTemplate.opsForList().remove("key", count, "value") | redisson.getList("key").remove("value") |
| SPOP key | Delete | Извлечь случайный элемент (Set) | redisTemplate.opsForSet().pop("key") | redisson.getSet("key").random() |
| SREM key member | Delete | Удалить элемент (Set) | redisTemplate.opsForSet().remove("key", "member") | redisson.getSet("key").remove("member") |
| ZPOPMIN key | Delete | Извлечь минимальный (Sorted Set) | redisTemplate.opsForZSet().popMin("key") | redisson.getScoredSortedSet("key").pollFirst() |
| ZPOPMAX key | Delete | Извлечь максимальный (Sorted Set) | redisTemplate.opsForZSet().popMax("key") | redisson.getScoredSortedSet("key").pollLast() |
| ZREM key member | Delete | Удалить элемент (Sorted Set) | redisTemplate.opsForZSet().remove("key", "member") | redisson.getScoredSortedSet("key").remove("member") |
| ZREMRANGEBYRANK key start stop | Delete | Удалить по диапазону rank | redisTemplate.opsForZSet().removeRange("key", start, stop) | redisson.getScoredSortedSet("key").removeRange(start, stop) |
| ZREMRANGEBYSCORE key min max | Delete | Удалить по диапазону score | redisTemplate.opsForZSet().removeRangeByScore("key", min, max) | redisson.getScoredSortedSet("key").removeRangeByScore(min, max) |
| XDEL key ID | Delete | Удалить запись (Stream) | redisTemplate.opsForStream().delete("key", streamId) | redisson.getStream("key").remove(streamId) |
| XTRIM key MAXLEN count | Delete | Обрезать поток | redisTemplate.opsForStream().trim("key", count) | redisson.getStream("key").trim(count) |

---

## Сводная таблица CRUD операций по типам данных

### String операции:

| CRUD | Операции |
|------|----------|
| Create | SET, SETEX, SETNX, MSET |
| Read | GET, MGET, GETRANGE, STRLEN |
| Update | INCR, INCRBY, DECR, DECRBY, APPEND, GETSET |
| Delete | DEL, UNLINK, GETDEL |

### Hash операции:

| CRUD | Операции |
|------|----------|
| Create | HSET, HMSET, HSETNX |
| Read | HGET, HMGET, HGETALL, HVALS, HKEYS, HEXISTS, HLEN, HSTRLEN |
| Update | HSET, HINCRBY, HINCRBYFLOAT |
| Delete | HDEL, DEL |

### List операции:

| CRUD | Операции |
|------|----------|
| Create | LPUSH, RPUSH, LPUSHX, RPUSHX, LINSERT |
| Read | LINDEX, LRANGE, LLEN |
| Update | LSET, LTRIM, LMOVE, BLMOVE |
| Delete | LPOP, RPOP, LREM, DEL |

### Set операции:

| CRUD | Операции |
|------|----------|
| Create | SADD, SMOVE |
| Read | SMEMBERS, SISMEMBER, SCARD, SRANDMEMBER, SSCAN |
| Update | SADD (добавление новых) |
| Delete | SPOP, SREM, DEL |

### Sorted Set операции:

| CRUD | Операции |
|------|----------|
| Create | ZADD, ZINCRBY |
| Read | ZRANGE, ZREVRANGE, ZRANK, ZREVRANK, ZSCORE, ZCARD, ZCOUNT, ZSCAN |
| Update | ZADD (обновление score), ZINCRBY |
| Delete | ZPOPMIN, ZPOPMAX, ZREM, ZREMRANGEBYRANK, ZREMRANGEBYSCORE, DEL |

### Stream операции:

| CRUD | Операции |
|------|----------|
| Create | XADD |
| Read | XREAD, XREADGROUP, XLEN, XINFO |
| Update | XGROUP CREATE, XGROUP DELCONSUMER |
| Delete | XDEL, XTRIM, XGROUP DESTROY |

---

## TTL операции (отдельная категория)

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| EXPIRE key seconds | Update | Установить TTL (сек) | redisTemplate.expire("key", seconds, TimeUnit.SECONDS) | redisson.getBucket("key").expire(seconds, TimeUnit.SECONDS) |
| PEXPIRE key milliseconds | Update | Установить TTL (мс) | redisTemplate.expire("key", milliseconds, TimeUnit.MILLISECONDS) | redisson.getBucket("key").expire(milliseconds, TimeUnit.MILLISECONDS) |
| EXPIREAT key timestamp | Update | Установить expire по timestamp | redisTemplate.expireAt("key", date) | redisson.getBucket("key").expireAt(date) |
| PEXPIREAT key timestamp | Update | Установить expire по timestamp (мс) | redisTemplate.expireAt("key", date) | redisson.getBucket("key").expireAt(date) |
| TTL key | Read | Оставшееся время (сек) | redisTemplate.getExpire("key", TimeUnit.SECONDS) | redisson.getBucket("key").remainTimeToLive() |
| PTTL key | Read | Оставшееся время (мс) | redisTemplate.getExpire("key", TimeUnit.MILLISECONDS) | redisson.getBucket("key").remainTimeToLive() |
| PERSIST key | Update | Убрать TTL | redisTemplate.persist("key") | redisson.getBucket("key").clearExpire() |
| SETEX key seconds value | Create | SET + EXPIRE атомарно | redisTemplate.opsForValue().set("key", "value", seconds, TimeUnit.SECONDS) | redisson.getBucket("key").set("value", seconds, TimeUnit.SECONDS) |
| PSETEX key milliseconds value | Create | SET + PEXPIRE атомарно | redisTemplate.opsForValue().set("key", "value", milliseconds, TimeUnit.MILLISECONDS) | redisson.getBucket("key").set("value", milliseconds, TimeUnit.MILLISECONDS) |

---

## Ключ-ориентированные операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| DEL key | Delete | Удалить ключи | redisTemplate.delete(keys) | redisson.getKeys().delete("key") |
| UNLINK key | Delete | Асинхронное удаление | redisTemplate.unlink(keys) | Не поддерживается |
| EXISTS key | Read | Проверить существование | redisTemplate.hasKey("key") | redisson.getKeys().countExists("key") |
| TYPE key | Read | Получить тип ключа | redisTemplate.type("key") | redisson.getKeys().getType("key") |
| RENAME key newKey | Update | Переименовать ключ | redisTemplate.rename("key", "newKey") | redisson.getKeys().rename("key", "newKey") |
| RENAMENX key newKey | Update | Переименовать если не существует | redisTemplate.renameIfAbsent("key", "newKey") | Не поддерживается |
| MOVE key db | Update | Переместить в другую БД | redisTemplate.move("key", dbIndex) | Не поддерживается в кластере |
| COPY source dest | Create | Копировать ключ | redisTemplate.copy("source", "dest", false) | Не поддерживается |
| KEYS pattern | Read | Найти ключи по паттерну | redisTemplate.keys("pattern") | Не рекомендуется |
| SCAN cursor MATCH pattern | Read | Итеративный поиск | redisTemplate.scan(scanOptions) | redisson.getKeys().getKeysStreamByPattern("pattern") |
| RANDOMKEY | Read | Случайный ключ | Не поддерживается | Не поддерживается |
| SORT key | Read | Сортировать List/Set | redisTemplate.sort("key") | Не поддерживается напрямую |
| TOUCH key | Read | Обновить время доступа | Не поддерживается напрямую | Не поддерживается |
| OBJECT subcommand | Read | Инспекция объекта | Не поддерживается напрямую | Не поддерживается |

---

## Атомарные операции (важно для конкурентного доступа)

| Redis CLI | Тип | Описание | RedisTemplate | Redisson |
|-----------|-----|----------|---------------|----------|
| INCR key | String | Атомарный инкремент | increment() | getAtomicLong().incrementAndGet() |
| DECR key | String | Атомарный декремент | decrement() | getAtomicLong().decrementAndGet() |
| INCRBY key N | String | Атомарный инкремент на N | increment(delta) | getAtomicLong().addAndGet(N) |
| DECRBY key N | String | Атомарный декремент на N | decrement(delta) | getAtomicLong().addAndGet(-N) |
| INCRBYFLOAT key N | String | Атомарный инкремент на float | increment(delta) | getAtomicDouble().addAndGet(N) |
| HINCRBY key field N | Hash | Атомарный инкремент поля | increment(field, delta) | getMap().addAndGet(field, N) |
| HINCRBYFLOAT key field N | Hash | Атомарный инкремент поля на float | increment(field, delta) | getMap().addAndGet(field, N) |
| ZINCRBY key N member | Sorted Set | Атомарный инкремент score | incrementScore(member, delta) | getScoredSortedSet().addAndGetScore(member, N) |
| SETNX key value | String | SET если не существует | setIfAbsent() | getBucket().trySet() |
| SET key value NX | String | SET если не существует | setIfAbsent() | getBucket().trySet() |
| APPEND key value | String | Атомарное добавление к строке | append() | getBucket().append() |
| GETSET key value | String | Атомарно GET + SET | getAndSet() | getBucket().getAndSet() |
| MSET key1 value1 ... | String | Атомарная установка нескольких | multiSet() | createBatch() |

---

## Blocking операции (для очередей)

| Redis CLI | Тип | Описание | RedisTemplate | Redisson |
|-----------|-----|----------|---------------|----------|
| BLPOP key timeout | List | Blocking LPOP | leftPop(timeout, unit) | getList().pollFirst(timeout, unit) |
| BRPOP key timeout | List | Blocking RPOP | rightPop(timeout, unit) | getList().pollLast(timeout, unit) |
| BLMOVE src dest LEFT LEFT | List | Blocking перемещение | Не поддерживается | Не поддерживается |
| BZPOPMIN key timeout | Sorted Set | Blocking ZPOPMIN | popMin(timeout, unit) | getScoredSortedSet().pollFirst(timeout, unit) |
| BZPOPMAX key timeout | Sorted Set | Blocking ZPOPMAX | popMax(timeout, unit) | getScoredSortedSet().pollLast(timeout, unit) |
| XREAD BLOCK timeout | Stream | Blocking XREAD | read(options, offset) | getStream().readGroup(group, consumer, count, timeout) |
| XREADGROUP BLOCK timeout | Stream | Blocking XREADGROUP | read(consumer, options, offset) | getStream().readGroup(group, consumer, count, timeout) |

---

## Set операции с несколькими ключами

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| SINTER key1 key2 | Read | Пересечение множеств | intersect(key1, key2) | getSet().readIntersection(otherSet) |
| SUNION key1 key2 | Read | Объединение множеств | union(key1, key2) | getSet().readUnion(otherSet) |
| SDIFF key1 key2 | Read | Разность множеств | difference(key1, key2) | getSet().readDiff(otherSet) |
| SINTERSTORE dest key1 key2 | Create | Пересечение с сохранением | intersectAndStore(key1, key2, dest) | Не поддерживается |
| SUNIONSTORE dest key1 key2 | Create | Объединение с сохранением | unionAndStore(key1, key2, dest) | Не поддерживается |
| SDIFFSTORE dest key1 key2 | Create | Разность с сохранением | differenceAndStore(key1, key2, dest) | Не поддерживается |
| SMOVE source dest member | Update | Переместить элемент | move(source, member, dest) | Не поддерживается |

---

## Sorted Set операции с несколькими ключами

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| ZINTER key1 key2 | Read | Пересечение Sorted Set | intersect(key1, key2) | Не поддерживается напрямую |
| ZUNION key1 key2 | Read | Объединение Sorted Set | union(key1, key2) | Не поддерживается напрямую |
| ZDIFF key1 key2 | Read | Разность Sorted Set | Не поддерживается | Не поддерживается |
| ZINTERSTORE dest key1 key2 | Create | Пересечение с сохранением | intersectAndStore(key1, key2, dest) | Не поддерживается |
| ZUNIONSTORE dest key1 key2 | Create | Объединение с сохранением | unionAndStore(key1, key2, dest) | Не поддерживается |
| ZDIFFSTORE dest key1 key2 | Create | Разность с сохранением | Не поддерживается | Не поддерживается |
| ZRANGESTORE dest key start stop | Create | Сохранить диапазон | Не поддерживается | Не поддерживается |

---

## Stream Consumer Group операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| XGROUP CREATE key group ID | Create | Создать consumer group | createGroup(key, group) | getStream().createGroup(group) |
| XGROUP DESTROY key group | Delete | Удалить consumer group | destroyGroup(key, group) | getStream().deleteGroup(group) |
| XGROUP DELCONSUMER key group consumer | Delete | Удалить consumer | deleteConsumer(key, group, consumer) | getStream().deleteConsumer(group, consumer) |
| XGROUP SETID key group ID | Update | Установить ID группы | Не поддерживается | Не поддерживается |
| XREADGROUP GROUP group consumer | Read | Читать как consumer group | read(consumer, options, offset) | getStream().readGroup(group, consumer, count) |
| XACK key group ID | Update | Подтвердить обработку | acknowledge(key, group, streamId) | getStream().acknowledge(group, streamId) |
| XPENDING key group | Read | Необработанные сообщения | pending(key, group) | getStream().pending(group) |
| XCLAIM key group consumer minIdle ID | Update | Забрать сообщения | claim(key, group, consumer, minIdle, streamId) | getStream().claim(group, consumer, minIdle, streamId) |
| XAUTOCLAIM key group consumer minIdle start | Update | Автозабрать сообщения | Не поддерживается | getStream().autoClaim(group, consumer, minIdle, start) |
| XINFO GROUPS key | Read | Информация о группах | groups(key) | getStream().listGroups() |
| XINFO CONSUMERS key group | Read | Информация о consumers | consumers(key, group) | getStream().listConsumers(group) |

---

## Геоданные операции (Geospatial)

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| GEOADD key lon lat member | Create | Добавить координаты | add(key, point) | getGeo().add(point, member) |
| GEOPOS key member | Read | Получить координаты | position(key, member) | getGeo().getPosition(member) |
| GEODIST key member1 member2 | Read | Расстояние между | distance(key, member1, member2) | getGeo().getDistance(member1, member2) |
| GEORADIUS key lon lat radius | Read | Поиск в радиусе | radius(key, circle) | getGeo().radius(center, distance) |
| GEORADIUSBYMEMBER key member radius | Read | Поиск вокруг элемента | radiusByMember(key, member, distance) | getGeo().radiusByMember(member, distance) |
| GEOHASH key member | Read | Получить GeoHash | hash(key, member) | getGeo().getHash(member) |
| GEOSEARCH key FROMMEMBER member | Read | Поиск с расширенными опциями | search(key, circle) | Не поддерживается |

---

## HyperLogLog операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| PFADD key element | Create | Добавить элемент | add("key", "element") | getHyperLogLog("key").add("element") |
| PFCOUNT key | Read | Получить cardinality | size("key") | getHyperLogLog("key").count() |
| PFMERGE dest key1 key2 | Create | Объединить несколько | Не поддерживается | Не поддерживается |

---

## Bitmap операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| SETBIT key offset value | Update | Установить бит | Не поддерживается напрямую | Не поддерживается напрямую |
| GETBIT key offset | Read | Получить бит | Не поддерживается напрямую | Не поддерживается напрямую |
| BITCOUNT key | Read | Посчитать установленные биты | Не поддерживается напрямую | Не поддерживается напрямую |
| BITOP AND/OR/XOR dest key1 | Update | Битовые операции | Не поддерживается напрямую | Не поддерживается напрямую |
| BITPOS key bit | Read | Найти позицию бита | Не поддерживается напрямую | Не поддерживается напрямую |
| BITFIELD key | Update | Битовые операции с полями | Не поддерживается напрямую | Не поддерживается напрямую |

---

## Lua Script операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| EVAL script numkeys key | Все | Выполнить скрипт | execute(script, keys, args) | getScript().eval(script, mode, keys, args) |
| EVALSHA sha1 numkeys key | Все | Выполнить по хешу | execute(scriptSha, keys, args) | getScript().evalSha(scriptSha, mode, keys, args) |
| SCRIPT LOAD script | Create | Загрузить скрипт | scriptLoad(script) | getScript().scriptLoad(script) |
| SCRIPT EXISTS sha1 | Read | Проверить наличие | scriptExists(scriptSha) | getScript().scriptExists(scriptSha) |
| SCRIPT FLUSH | Delete | Удалить все скрипты | scriptFlush() | getScript().scriptFlush() |
| SCRIPT KILL | Delete | Убить выполняющийся скрипт | scriptKill() | getScript().scriptKill() |

---

## Transaction операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| MULTI | Update | Начать транзакцию | multi() | beginBatch() |
| EXEC | Update | Выполнить транзакцию | exec() | execute() |
| DISCARD | Update | Отменить транзакцию | discard() | Не поддерживается |
| WATCH key | Update | Оптимистичная блокировка | watch(key) | Не поддерживается |
| UNWATCH | Update | Снять WATCH | unwatch() | Не поддерживается |

---

## Pipeline операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| PIPELINE | Все | Группировка команд | executePipelined() | createBatch() |
| MULTI/EXEC | Все | Транзакция | multi()/exec() | beginBatch()/execute() |

---

## Server/Connection операции

| Redis CLI | CRUD-операция | Описание операции | RedisTemplate | Redisson |
|-----------|---------------|-------------------|---------------|----------|
| PING | Read | Проверить подключение | execute(ReduceCommand.PING) | ping() |
| ECHO message | Read | Эхо сообщение | Не поддерживается | Не поддерживается |
| SELECT db | Update | Выбрать БД | Не поддерживается в pool | Не поддерживается в кластере |
| AUTH password | Update | Аутентификация | Настраивается в подключении | Настраивается в подключении |
| QUIT | Delete | Закрыть соединение | close() | shutdown() |
| INFO | Read | Информация о сервере | Не поддерживается напрямую | Не поддерживается напрямую |
| CLIENT LIST | Read | Список клиентов | Не поддерживается напрямую | Не поддерживается напрямую |
| CONFIG GET param | Read | Получить конфигурацию | Не поддерживается напрямую | Не поддерживается напрямую |
| CONFIG SET param value | Update | Установить конфигурацию | Не поддерживается напрямую | Не поддерживается напрямую |
| DBSIZE | Read | Количество ключей в БД | Не поддерживается напрямую | Не поддерживается напрямую |
| FLUSHDB | Delete | Очистить текущую БД | Не рекомендуется | Не рекомендуется |
| FLUSHALL | Delete | Очистить все БД | Не рекомендуется | Не рекомендуется |
| SAVE | Create | Синхронное сохранение | Не поддерживается | Не поддерживается |
| BGSAVE | Create | Асинхронное сохранение | Не поддерживается | Не поддерживается |
| BGREWRITEAOF | Create | Переписать AOF | Не поддерживается | Не поддерживается |
| LASTSAVE | Read | Время последнего сохранения | Не поддерживается | Не поддерживается |
| SLOWLOG GET | Read | Медленные запросы | Не поддерживается | Не поддерживается |
| TIME | Read | Время сервера | Не поддерживается | Не поддерживается |
| MONITOR | Read | Real-time мониторинг | Не поддерживается | Не поддерживается |
| DEBUG | Все | Отладочные команды | Отключено | Отключено |

---

## Примечания по использованию

### RedisTemplate:

- `opsForValue()` — операции со String
- `opsForHash()` — операции с Hash
- `opsForList()` — операции с List
- `opsForSet()` — операции с Set
- `opsForZSet()` — операции с Sorted Set
- `opsForGeo()` — операции с Geospatial
- `opsForHyperLogLog()` — операции с HyperLogLog
- `opsForStream()` — операции с Stream
- `execute()` — выполнение произвольных команд
- `opsForCluster()` — кластерные операции

### Redisson:

- `getBucket()` — простые значения (String)
- `getMap()` — Hash
- `getList()` — List
- `getSet()` — Set
- `getScoredSortedSet()` — Sorted Set
- `getStream()` — Stream
- `getGeo()` — Geospatial
- `getHyperLogLog()` — HyperLogLog
- `getAtomicLong()` — атомарные счётчики
- `getKeys()` — операции с ключами
- `getScript()` — Lua скрипты
- `createBatch()` — батч-операции
- `getBlockingQueue()` — blocking очереди

### Важные отличия:

1. RedisTemplate ближе к нативным Redis командам
2. Redisson имеет более богатый API для распределённых структур
3. RedisTemplate возвращает null при отсутствии ключа
4. Redisson может возвращать default значения
5. Redisson лучше подходит для распределённых блокировок
6. RedisTemplate требует явной сериализации
7. Redisson имеет встроенную сериализацию

---

## Рекомендации по выбору операций

### Для кэширования:

- String: SET/GET с TTL
- Hash: HSET/HGET для объектов
- Используйте MSET/MGET для групповых операций

### Для очередей:

- List: LPUSH/BRPOP для простых очередей
- Sorted Set: ZADD/ZPOPMIN для приоритетных очередей
- Stream: XADD/XREADGROUP для надёжных очередей

### Для счётчиков:

- String: INCR/INCRBY (атомарно)
- Hash: HINCRBY для счётчиков внутри объектов

### Для уникальных элементов:

- Set: SADD/SISMEMBER
- HyperLogLog: PFADD/PFCOUNT для приблизительного подсчёта

### Для рейтингов:

- Sorted Set: ZADD/ZREVRANGE/ZRANK

### Для геоданных:

- Geospatial: GEOADD/GEORADIUS

### Для event sourcing:

- Stream: XADD/XREADGROUP/XACK

### Для сессий:

- String: SETEX для автоматического TTL
- Hash: HSET для данных сессии + EXPIRE

### Для rate limiting:

- String: INCR + EXPIRE для простого лимита
- Sorted Set: ZADD + ZREMRANGEBYSCORE для скользящего окна

### Для distributed lock:

- String: SET key value NX EX
- Redisson: RLock для готового решения

---

## 5. Расширенные Возможности

### Pub/Sub (Публикация/Подписка)

Встроенная система сообщений.

#### Команды:

```redis
    SUBSCRIBE channel              // Подписаться на канал
    PSUBSCRIBE pattern             // Подписаться по паттерну
    PUBLISH channel message        // Опубликовать сообщение
    UNSUBSCRIBE channel            // Отписаться
```

#### Пример:

```redis
    // Подписчик (terminal 1)
    SUBSCRIBE notifications:user:1001

    // Издатель (terminal 2)
    PUBLISH notifications:user:1001 "New message!"
```

Важно: Pub/Sub не сохраняет сообщения. Если подписчик офлайн — сообщение теряется.
Для надёжной доставки используйте Streams с Consumer Groups.

### Transactions (Транзакции)

Redis поддерживает транзакции через MULTI/EXEC.

#### Команды:

```redis
    MULTI                          // Начать транзакцию
    EXEC                           // Выполнить все команды
    DISCARD                        // Отменить транзакцию
    WATCH key                      // Оптимистичная блокировка
    UNWATCH                        // Снять WATCH
```

#### Пример:

```redis
    MULTI
    SET user:1001:balance 1000
    INCR user:1001:transactionCount
    EXEC
```

Важные особенности:

1. Нет изоляции — другие клиенты видят изменения после `EXEC`
2. Нет rollback — если команда ошиблась, остальные выполняются
3. `WATCH` для optimistic locking:

```redis
   WATCH user:1001:balance
   current = GET user:1001:balance
   MULTI
   SET user:1001:balance (current - 100)
   EXEC    // Вернёт null если ключ изменился между WATCH и EXEC
```

### Lua Scripts (Скрипты)

Выполнение атомарных скриптов на Lua.

#### Команды:

```redis
    EVAL script numkeys key ...    // Выполнить скрипт
    EVALSHA sha1 numkeys key ...   // Выполнить по хешу
    SCRIPT LOAD script             // Загрузить скрипт
    SCRIPT EXISTS sha1 ...         // Проверить наличие
```

#### Пример (атомарный инкремент с проверкой):

```lua
-- KEYS[] — массив ключей (передается отдельно)
-- ARGV[] — массив аргументов (передается отдельно)
-- redis.call() — выполнение Redis-команды
-- redis.pcall() — выполнение с перехватом ошибок

-- Скрипт:
local current = redis.call('GET', KEYS[1])           -- получить текущее значение
if not current or tonumber(current) < tonumber(ARGV[1]) then  -- проверка
    return redis.call('INCR', KEYS[1])               -- инкремент и возврат
else
    return -1                                         -- превышен лимит
end
```
Выполнение:

```redis
    EVAL "<script>" 1 user:1001:limit 100
```

Выполнение из Java:

```java
// RedisTemplate
String script =
    "local current = redis.call('GET', KEYS[1]) " +
    "if not current or tonumber(current) < tonumber(ARGV[1]) then " +
    "    return redis.call('INCR', KEYS[1]) " +
    "else " +
    "    return -1 " +
    "end";

DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
redisScript.setScriptText(script);
redisScript.setResultType(Long.class);

Long result = redisTemplate.execute(
    redisScript,
    Collections.singletonList("counter:limit"),  // KEYS[1]
    100  // ARGV[1] (максимальное значение)
);

if (result == -1) {
    // лимит превышен
} else {
    // result — новое значение счетчика
```

Преимущества Lua:
- Атомарное выполнение (нет race conditions)
- Меньше сетевых round-trips
- Сложная логика на стороне сервера

### Distributed Locks (Распределённые блокировки)

#### Простой lock:

```redis
    SET lock:resource unique_value NX EX 30
```

`NX` = только если не существует

`EX` = expire 30 секунд

#### Освобождение lock (через Lua для атомарности):

```lua
    // Проверить что lock принадлежит нам и удалить
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
```

#### Redlock (для Redis Cluster):

Алгоритм для распределённых блокировок across multiple Redis instances.
Рекомендуется использовать готовые библиотеки (Redisson для Java).

### Pipeline (Конвейер)

Группировка команд для уменьшения сетевых задержек.

#### Без Pipeline (N round-trips):

```redis
    SET key1 value1    // RTT 1
    SET key2 value2    // RTT 2
    SET key3 value3    // RTT 3
```

#### С Pipeline (1 round-trip):

```redis
    PIPELINE
    SET key1 value1
    SET key2 value2
    SET key3 value3
    EXEC
```

В Java (Lettuce):

```java
    StatefulRedisConnection<String, String> connection = ...
    RedisStringCommands<String, String> sync = connection.sync();
    Pipeline<String, String> pipeline = connection.newPipeline();

    pipeline.set("key1", "value1");
    pipeline.set("key2", "value2");
    pipeline.set("key3", "value3");

    pipeline.flush();  // Отправить все команды разом
```

---

## 6. Персистентность и Репликация

### RDB (Redis Database Backup)

Снэпшоты данных на диск в определённые интервалы.

#### Конфигурация (redis.conf):

```text
    save 900 1         // Сохранить если 1 изменение за 900 сек
    save 300 10        // Сохранить если 10 изменений за 300 сек
    save 60 10000      // Сохранить если 10000 изменений за 60 сек

    dbfilename dump.rdb
    dir /var/lib/redis
```

#### Преимущества:

- Компактные файлы
- Быстрое восстановление
- Подходит для backup

#### Недостатки:

- Потеря данных между снэпшотами
- Блокировка на время save (используйте `BGSAVE`)

### AOF (Append Only File)

Лог всех операций записи.

#### Конфигурация:

```text
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec    // everysec / always / no
```

#### Режимы fsync:

| Режим    | Описание                    | Производительность | Безопасность            |
|:---------|-----------------------------|--------------------|-------------------------|
| always   | fsync после каждой операции | Низкая             | Максимальная            |
| everysec | fsync каждую секунду        | Высокая            | Хорошая (потеря ~1 сек) |
| no       | OS решает когда fsync       | Максимальная       | Низкая                  |

#### Преимущества:

- Минимальная потеря данных
- Можно восстанавливать по операциям

#### Недостатки:

- Файлы больше чем RDB
- Медленнее восстановление

### Комбинированный подход (рекомендуется):

Использовать и RDB, и AOF одновременно.
При восстановлении Redis использует AOF (более полный).

### Репликация (Master-Replica)

#### Настройка Replica:

```text
    replicaof <masterip> <masterport>
```

#### Особенности:

- Асинхронная репликация (возможна потеря данных при failover)
- Read scaling — replica могут обслуживать чтение
- Один master, много replica

### Sentinel (Высокая доступность)

Автоматический failover master.

#### Конфигурация sentinel.conf:

```text
    sentinel monitor mymaster 127.0.0.1 6379 2
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 10000
    sentinel parallel-syncs mymaster 1
```

#### Параметры:

- down-after-milliseconds — время до считать master down
- failover-timeout — таймаут failover
- parallel-syncs — сколько replica синхронизируются параллельно

### Redis Cluster (Шардирование)

Горизонтальное масштабирование через шарды.

#### Архитектура:

- 16384 hash slots распределены между нодами
- Ключи распределяются по slot: CRC16(key) % 16384
- Минимум 3 master нод для кластера

#### Создание кластера:

```shell
    redis-cli --cluster create \
      127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
      127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
      --cluster-replicas 1
```

#### Hash Tags (для группировки ключей):

Ключи с одинаковым hash tag попадают на одну ноду:

```text
    {user:1001}:profile    // Все на одном слоте
    {user:1001}:orders
    {user:1001}:sessions
```

Полезно для транзакций и multi-key операций в кластере.

---

## 7. Производительность и Best Practices

### Мониторинг

#### Ключевые команды:

```redis
    INFO                         // Общая статистика
    INFO memory                  // Использование памяти
    INFO stats                   // Статистика операций
    INFO replication             // Статус репликации
    MONITOR                      // Real-time мониторинг команд (не для production!)
    SLOWLOG GET                  // Медленные запросы
    CLIENT LIST                  // Подключённые клиенты
```

#### Проверка памяти:

```redis
    MEMORY USAGE key             // Сколько памяти занимает ключ
    MEMORY STATS                 // Детальная статистика памяти
```

#### Медленные запросы:

```redis
    CONFIG SET slowlog-log-slower-than 10000    // 10ms в микросекундах
    SLOWLOG GET 10                              // Последние 10 медленных
```

### Оптимизация памяти

#### Избегайте больших ключей:

- Ключи > 10 KB — потенциальная проблема
- Ключи > 1 MB — критично (блокирует однопоточную обработку)

#### Используйте правильные структуры:

- Hash вместо String для объектов с полями
- List/Set для коллекций
- HyperLogLog для подсчёта уникальных

#### Настройка maxmemory:

```text
    maxmemory 2gb
    maxmemory-policy allkeys-lru    // LRU eviction
```

#### Политики eviction:

| Политика       | Описание                                    |
|:---------------|---------------------------------------------|
| noeviction     | Возвращать ошибку при нехватке памяти       |
| allkeys-lru    | Удалять наименее используемые (любые ключи) |
| volatile-lru   | Удалять наименее используемые (с TTL)       |
| allkeys-random | Случайное удаление                          |
| volatile-ttl   | Удалять с наименьшим TTL                    |

### Паттерны именования ключей

#### Рекомендации:

- Используйте префиксы: user:1001:profile, cache:product:5567
- Избегайте сложных префиксов: data:user:id:1001:profile (слишком длинно)
- Используйте hash tags для кластера: {user:1001}:data

#### Примеры:

Хорошо:

```text
user:1001:profile
session:abc123
cache:api:products:list
```

Плохо:

```text
userProfile1001
1001
data
```

### Connection Pooling

#### Lettuce (по умолчанию thread-safe):

Lettuce использует Netty и один connection может использоваться несколькими потоками.

```java
    RedisClient client = RedisClient.create("redis://localhost");
    StatefulRedisConnection<String, String> connection = client.connect();
    RedisStringCommands<String, String> sync = connection.sync();

    // connection можно переиспользовать
```

#### Jedis (требует pool):

```java
    JedisPool pool = new JedisPool("localhost", 6379);
    try (Jedis jedis = pool.getResource()) {
        jedis.set("key", "value");
    }
```

### Обработка ошибок

#### Типичные исключения:

```text
JedisConnectionException    // Проблемы подключения
JedisDataException          // Ошибка команды (синтаксис, тип данных)
JedisClusterException       // Ошибка кластера
RedisCommandTimeoutException // Таймаут команды
```

#### Retry логика:

- Реализуйте retry для transient ошибок
- Используйте circuit breaker для защиты от cascade failures
- Логгируйте все ошибки для анализа

### Security

#### Аутентификация:

```text
    requirepass your_secure_password
```

#### В Java:

```java
    RedisURI uri = RedisURI.Builder.redis("host", 6379)
        .withPassword("password")
        .withDatabase(0)
        .build();
```

#### TLS/SSL:

```text
    redis://user:password@host:6379/0?ssl=true
```

#### Ограничение команд:

```redis
    rename-command CONFIG ""
    rename-command FLUSHALL ""
    rename-command DEBUG ""
```

#### Network security:

- Bind только к внутренним интерфейсам
- Используйте firewall правила
- Не экспонируйте Redis публично

---

## 8. Java Интеграция

### Lettuce (рекомендуется)

#### Базовое подключение:

```java
    RedisClient client = RedisClient.create("redis://localhost:6379");
    StatefulRedisConnection<String, String> connection = client.connect();
    RedisStringCommands<String, String> sync = connection.sync();

    sync.set("key", "value");
    String value = sync.get("key");

    connection.close();
    client.shutdown();
```

#### Асинхронный API:

```java
    RedisAsyncCommands<String, String> async = connection.async();
    async.set("key", "value");
    async.get("key").thenAccept(System.out::println);
```

#### Reactive API (Project Reactor):

```java
    RedisReactiveCommands<String, String> reactive = connection.reactive();
    reactive.set("key", "value")
        .then(reactive.get("key"))
        .subscribe(System.out::println);
```

#### Connection Pool с Lettuce:

```java
    GenericObjectPoolConfig<PoolConfig> poolConfig = new GenericObjectPoolConfig<>();
    poolConfig.setMaxTotal(50);
    poolConfig.setMaxIdle(20);
    poolConfig.setMinIdle(5);

    RedisClient client = RedisClient.create("redis://localhost");
    StatefulRedisConnection<String, String> connection = client.connect();
    RedisCommands<String, String> commands = connection.sync();
```

### Spring Data Redis

#### Конфигурация (application.yml):

```yaml
    spring:
      redis:
        host: localhost
        port: 6379
        password: your_password
        database: 0
        timeout: 5000ms
        lettuce:
          pool:
            max-active: 50
            max-idle: 20
            min-idle: 5
```

#### RedisTemplate:

```java
    @Configuration
    public class RedisConfig {
        
        @Bean
        public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
            RedisTemplate<String, Object> template = new RedisTemplate<>();
            template.setConnectionFactory(factory);
            template.setKeySerializer(new StringRedisSerializer());
            template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
            template.setHashKeySerializer(new StringRedisSerializer());
            template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
            template.afterPropertiesSet();
            return template;
        }
    }
```

#### Использование:

```java
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // String операции
    redisTemplate.opsForValue().set("key", "value", 3600, TimeUnit.SECONDS);
    Object value = redisTemplate.opsForValue().get("key");

    // Hash операции
    redisTemplate.opsForHash().put("user:1001", "name", "Alex");
    Object name = redisTemplate.opsForHash().get("user:1001", "name");

    // List операции
    redisTemplate.opsForList().rightPush("queue", "task");
    Object task = redisTemplate.opsForList().leftPop("queue");
```

#### StringRedisTemplate (для строк):

```java
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    stringRedisTemplate.opsForValue().set("key", "value");
```

### Redisson (продвинутые возможности)

#### Зависимость:

```xml
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>3.27.0</version>
    </dependency>
```

#### Подключение:

```java
    Config config = new Config();
    config.useSingleServer()
        .setAddress("redis://localhost:6379")
        .setPassword("password");

    RedissonClient redisson = Redisson.create(config);
```

#### Distributed Lock:

```java
    RLock lock = redisson.getLock("lock:resource");
    lock.lock();
    try {
        // критическая секция
    } finally {
        lock.unlock();
    }
```

#### Fair Lock:

```java
    RFairLock fairLock = redisson.getFairLock("lock:fair");
    fairLock.lock();
```

#### ReadWrite Lock:

```java
    RReadWriteLock rwLock = redisson.getReadWriteLock("lock:rw");
    rwLock.readLock().lock();
    rwLock.writeLock().lock();
```

#### Distributed Collections:

```java
    RMap<String, String> map = redisson.getMap("myMap");
    RList<String> list = redisson.getList("myList");
    RSet<String> set = redisson.getSet("mySet");
```

#### Rate Limiter:

```java
    RRateLimiter rateLimiter = redisson.getRateLimiter("rate:limit");
    rateLimiter.trySetRate(RateType.OVERALL, 10, 1, RateIntervalUnit.SECONDS);
    rateLimiter.tryAcquire();  // true если разрешено
```

---

## 9. Тестирование

### TestContainers для интеграционных тестов

#### Зависимость:

```xlm
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>
```

#### Пример теста:

```java
    class RedisIntegrationTest {
        
        static RedisContainer redis = new RedisContainer("redis:7-alpine")
            .withExposedPorts(6379);
        
        static RedisClient client;
        static StatefulRedisConnection<String, String> connection;
        
        @BeforeAll
        static void setUp() {
            redis.start();
            String url = "redis://" + redis.getHost() + ":" + redis.getMappedPort(6379);
            client = RedisClient.create(url);
            connection = client.connect();
        }
        
        @AfterAll
        static void tearDown() {
            connection.close();
            client.shutdown();
            redis.stop();
        }
        
        @BeforeEach
        void cleanup() {
            RedisCommands<String, String> sync = connection.sync();
            sync.flushdb();  // Очистить БД перед каждым тестом
        }
        
        @Test
        void testSetAndGet() {
            RedisCommands<String, String> sync = connection.sync();
            
            sync.set("test:key", "test:value");
            String value = sync.get("test:key");
            
            assertEquals("test:value", value);
        }
        
        @Test
        void testTtl() {
            RedisCommands<String, String> sync = connection.sync();
            
            sync.setex("temp:key", 60, "temp:value");
            Long ttl = sync.ttl("temp:key");
            
            assertTrue(ttl > 0 && ttl <= 60);
        }
    }
```

### Embedded Redis для unit-тестов

#### Зависимость:

```xml
    <dependency>
        <groupId>it.ozimov</groupId>
        <artifactId>embedded-redis</artifactId>
        <version>0.7.3</version>
        <scope>test</scope>
    </dependency>
```

#### Пример:

```java
    class EmbeddedRedisTest {
        
        static RedisServer redisServer;
        static RedisClient client;
        
        @BeforeAll
        static void setUp() throws IOException {
            redisServer = RedisServer.builder()
                .port(6379)
                .setting("maxmemory 128M")
                .build();
            redisServer.start();
            
            client = RedisClient.create("redis://localhost:6379");
        }
        
        @AfterAll
        static void tearDown() {
            client.shutdown();
            redisServer.stop();
        }
    }
```

### Mock для unit-тестов

#### Mock RedisTemplate:

```java
    @Test
    void testServiceWithMock() {
        RedisTemplate<String, Object> mockTemplate = mock(RedisTemplate.class);
        ValueOperations<String, Object> mockOps = mock(ValueOperations.class);
        
        when(mockTemplate.opsForValue()).thenReturn(mockOps);
        when(mockOps.get("key")).thenReturn("value");
        
        // Тестирование сервиса с mock
    }
```

### Best Practices для тестирования

1. Изоляция тестов — очищайте БД между тестами (`FLUSHDB`)
2. Используйте TestContainers для интеграционных тестов
3. Не тестируйте Redis — тестируйте ваш код с Redis
4. Mock для unit-тестов, реальный Redis для integration
5. Тестируйте edge cases:
    - TTL expiration
    - Connection failures
    - Memory limits
    - Cluster failover

---

## Приложения

### Частые паттерны использования

#### Кэширование:

```java
    // Cache-aside pattern
    String key = "cache:user:" + userId;
    String cached = redis.get(key);
    if (cached == null) {
        cached = database.getUser(userId);
        redis.setex(key, 3600, cached);  // TTL 1 час
    }
    return cached;
```

#### Сессии:

```java
    String sessionKey = "session:" + sessionId;
    redis.setex(sessionKey, 1800, userId);  // 30 минут

    // Продление сессии
    redis.expire(sessionKey, 1800);
```

#### Rate Limiting:

```java
    String key = "ratelimit:" + userId + ":" + currentTimeMinute;
    long count = redis.incr(key);
    if (count == 1) {
        redis.expire(key, 60);  // 1 минута
    }
    return count <= 100;  // 100 запросов в минуту
```

#### Distributed Lock:

```java
    String lockKey = "lock:order:" + orderId;
    String lockValue = UUID.randomUUID().toString();
    boolean acquired = redis.set(lockKey, lockValue, "NX", "EX", 30);
    if (acquired) {
        try {
            // критическая секция
        } finally {
            // Lua script для безопасного удаления
            String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end";
            redis.eval(script, 1, lockKey, lockValue);
        }
    }
```

#### Очередь задач:

```java
    // Producer
    redis.rpush("queue:tasks", taskJson);

    // Consumer
    String task = redis.brpop(0, "queue:tasks");  // blocking
```

#### Leaderboard:

```java
    // Добавить/обновить score
    redis.zadd("leaderboard:game1", score, playerId);

    // Топ-10
    Set<ZSetOperations.TypedTuple> top10 = redis.zReverseRangeWithScores("leaderboard:game1", 0, 9);

    // Ранг игрока
    Long rank = redis.zRank("leaderboard:game1", playerId);
```

### Полезные ссылки

- Официальная документация: https://redis.io/docs/
- Redis Commands: https://redis.io/commands/
- Redis University (бесплатные курсы): https://university.redis.io/
- GitHub Lettuce: https://github.com/lettuce-io/lettuce-core
- GitHub Redisson: https://github.com/redisson/redisson
- Redis Best Practices: https://redis.io/docs/manual/

---

## Сравнение Redis и MongoDB (когда что использовать)

| Критерий            | Redis                          | MongoDB                       |
|:--------------------|--------------------------------|-------------------------------|
| Основное назначение | Кэш, очередь, real-time        | Основное хранилище данных     |
| Модель данных       | Key-Value + структуры          | Document (JSON/BSON)          |
| Персистентность     | Опциональная                   | Обязательная                  |
| Скорость записи     | ~100,000+ ops/sec              | ~10,000-50,000 ops/sec        |
| Сложные запросы     | Ограниченные                   | Полноценные (aggregation)     |
| Транзакции          | Ограниченные (Lua/MULTI)       | Полноценные (ACID)            |
| Размер данных       | Ограничен RAM                  | Disk + RAM                    |
| Use cases           | Кэш, сессии, очереди, realtime | Основное хранилище, аналитика |

### Типичная архитектура:

```text
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   Client    │────▶│    Redis    │────▶│  MongoDB    │
    │             │     │   (Cache)   │     │  (Primary)  │
    └─────────────┘     └─────────────┘     └─────────────┘
```

- Redis для кэширования частых запросов
- MongoDB как основное хранилище
- Redis для сессий и real-time данных
- MongoDB для сложных запросов и аналитики