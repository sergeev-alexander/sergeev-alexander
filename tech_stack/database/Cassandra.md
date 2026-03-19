# Apache Cassandra

## Содержание

1. [Теория и Архитектура](#1-теория-и-архитектура)
2. [Основные Концепции](#2-основные-концепции)
3. [Работа с Драйвером и CqlSession](#3-работа-с-драйвером-и-cqlsession)
4. [CRUD Операции в CQL](#4-crud-операции-в-cql)
5. [Особенности Моделирования Данных](#5-особенности-моделирования-данных)
6. [Производительность и Внутренние Механизмы](#6-производительность-и-внутренние-механизмы)
7. [Best Practices для Production](#7-best-practices-для-production)

---

## 1. Теория и Архитектура

### Что такое Cassandra?

> **Apache Cassandra** — распределённая, колонно-ориентированная, NoSQL база данных с мастер-лесс архитектурой, 
разработанная для обработки больших объёмов данных на множестве серверов с высокой доступностью и без единой точки отказа.

### Ключевые особенности:

- **Горизонтальное масштабирование** — линейная масштабируемость добавлением узлов
- **Мастер-лесс архитектура** — все узлы равноправны (peer-to-peer)
- **Настраиваемая согласованность** — выбор между consistency и availability
- **Высокая доступность** — автоматическая репликация и восстановление
- **Отказоустойчивость** — данные реплицируются на несколько узлов

### CAP-теорема и позиция Cassandra:

| Параметр | Значение | Пояснение |
|----------|----------|-----------|
| Consistency | Настраиваемая | Можно выбрать уровень (ONE, QUORUM, ALL и др.) |
| Availability | Высокая | Система остаётся доступной при разделении сети |
| Partition Tolerance | Гарантирована | Работает при сетевых разделениях |

Cassandra — **AP-система** с возможностью настройки согласованности под задачу.

### Колонно-ориентированное хранение: как это работает?

#### Реляционная (строковая) модель:

| id | name  | age | city     |
|----|-------|-----|----------|
| 1  | Alex  | 30  | Moscow   |
| 2  | Bob   | 25  | SPb      |

- Данные одной записи хранятся последовательно
- Эффективно для запросов по всей строке
- Неэффективно для аналитики по отдельным колонкам

#### Колонно-ориентированная модель Cassandra:

```text
Partition Key: id=1
├── name: [Alex]
├── age: [30]
├── city: [Moscow]
└── clustering: [timestamp]

Partition Key: id=2
├── name: [Bob]
├── age: [25]
├── city: [SPb]
└── clustering: [timestamp]
```

**Преимущества:**

1. **Эффективное чтение отдельных колонок** — загружаются только нужные данные
2. **Лучшая компрессия** — однотипные данные в колонке сжимаются эффективнее
3. **Оптимизация для агрегаций** — SUM/AVG по колонке быстрее
4. **Гибкая схема** — можно добавлять колонки "на лету" для разных записей

**Важное ограничение:**

- Запросы должны всегда включать **Partition Key**
- Нельзя делать `"SELECT * WHERE age > 25"` без фильтра по partition key
- Модель данных проектируется **под запросы**, а не под сущности

---

### Архитектурные компоненты:

- **Node** - Отдельный сервер в кластере
- **Cluster** - Группа узлов, работающих вместе
- **Keyspace** - Контейнер для таблиц (аналог БД в RDBMS)
- **Table** - Структура данных с первичным ключом
- **Partition** - Группа строк с одинаковым partition key
- **Gossip Protocol** - Протокол обмена состоянием между узлами
- **Hinted Handoff** - Механизм временного хранения записей при недоступности узла
- **Read/Repair** - Автоматическое восстановление несогласованных реплик

---

### Уровни согласованности (Consistency Levels):

|    Level     | Запись                       | Чтение                  | Использование                                    |
|:------------:|------------------------------|-------------------------|--------------------------------------------------|
|     ANY      | 1 узел (любой, включая hint) | —                       | Максимальная доступность, возможна потеря данных |
|     ONE      | 1 реплика                    | 1 реплика               | Низкая задержка, риск чтения устаревших данных   |
|    QUORUM    | majority реплик              | majority реплик         | Баланс между согласованностью и доступностью     |
|     ALL      | все реплики                  | все реплики             | Максимальная согласованность, низкая доступность |
| LOCAL_QUORUM | majority в локальном DC      | majority в локальном DC | Мульти-датацентр с локальной согласованностью    |

**Формула для сильной согласованности:**

**`R` + `W` > `RF`**

Где:

- `R` = replicas read
- `W` = replicas written
- `RF` = replication factor

Пример: при RF=3, если W=2 и R=2 → 2+2 > 3 → сильная согласованность гарантирована.

---

### Java Driver (DataStax):

```xml
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-core</artifactId>
    <version>4.17.0</version>
</dependency>
```

Для реактивного подхода:

```xml
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-reactive-streams</artifactId>
    <version>4.17.0</version>
</dependency>
```

---

## 2. Основные Концепции

### Keyspace

Контейнер верхнего уровня, аналог базы данных в RDBMS

- #### SimpleStrategy

  Реплики по порядку на кольце

  Применяется в разработке, тестировании. Один датацентр.

```sql
CREATE KEYSPACE IF NOT EXISTS myapp
WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
};
```

**Стратегии репликации:**

- #### NetworkTopologyStrategy

  Реплики с учётом датацентров
  
  Применяется в production. Несколько DC.

```sql
CREATE KEYSPACE myapp
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,
    'dc2': 2
};
```

---

### Table и Primary Key

**Первичный ключ в Cassandra состоит из двух частей:**

```sql
CREATE TABLE users (
    user_id UUID,
    timestamp TIMESTAMP,
    email TEXT,
    name TEXT,
    PRIMARY KEY (user_id, timestamp)
);
```


- **Partition Key** (user_id) - Определяет, на каком узле хранятся данные. Должен быть в WHERE для большинства запросов
- **Clustering Key** (timestamp) - Определяет порядок сортировки внутри партиции | Можно использовать в RANGE-запросах

**Важно понимать:**

- **Partition Key** → хэшируется → определяет токен → определяет узел хранения
- **Clustering Key** → сортирует данные внутри партиции (по возрастанию по умолчанию)
- **Composite Partition Key**: `PRIMARY KEY ((user_id, category), timestamp)`
  - Cassandra объединяет значения обоих столбцов (user_id и category) в одно составное значение
  - От этого составного значения вычисляется хэш, который определяет узел для хранения
  - Все данные с одинаковой комбинацией user_id + category попадают в одну партицию

---

### Типы данных CQL:

| Тип            | Описание                         | Пример                 |
|----------------|----------------------------------|------------------------|
| TEXT / VARCHAR | Строка                           | 'Hello World'          |
| INT            | 32-bit integer                   | 42                     |
| BIGINT         | 64-bit integer                   | 9223372036854775807    |
| UUID           | Универсальный идентификатор      | a0eebc99-9c0b...       |
| TIMEUUID       | UUID версии 1 (с временем)       | d2177dd0-eaa2...       |
| TIMESTAMP      | Дата и время                     | '2024-01-15T10:30:00Z' |
| BOOLEAN        | Логический тип                   | true/false             |
| DECIMAL        | Произвольная точность            | 123.456                |
| DOUBLE         | 64-bit float                     | 3.14159                |
| LIST           | Упорядоченный список             | ['a', 'b', 'c']        |
| SET            | Неупорядоченный уникальный набор | {'a', 'b'}             |
| MAP            | Ключ-значение                    | {'key': 'value'}       |

---

### Внутреннее устройство хранения данных

#### SSTable (Sorted String Table):

- Неизменяемые файлы на диске
- Данные отсортированы по ключу
- При обновлении создаётся **новая** запись (old + new coexist)
- Автоматический внутренний фоновый механизм **Compaction** объединяет и очищает старые версии, 
  оставляя только самую свежую версию (по timestamp)

#### MemTable:

- Данные сначала пишутся в память (MemTable)
- При заполнении → flush на диск как SSTable
- Быстрая запись, асинхронная персистентность

#### Commit Log:

- Все записи сначала в commit log (для durability)
- Затем в MemTable
- При crash → восстановление из commit log

#### Compaction:

- Процесс объединения SSTable
- Удаляет помеченные на удаление записи
- Восстанавливает место на диске
- Стратегии: SizeTiered, Leveled, TimeWindow

---

## 3. Работа с Драйвером и CqlSession

### Создание подключения

```java
import com.datastax.oss.driver.api.core.CqlSession;
import java.net.InetSocketAddress;

try (CqlSession session = CqlSession.builder()
    .addContactPoint(new InetSocketAddress("127.0.0.1", 9042))
    .withLocalDatacenter("datacenter1")
    .withKeyspace("myapp")
    .build()) {

    System.out.println("Connected to Cassandra");
    // работа с сессией
}
```

**Важные параметры подключения:**

| Параметр            | Описание                 | Рекомендация                             |
|:--------------------|--------------------------|------------------------------------------|
| addContactPoint     | Адрес узла для bootstrap | Минимум 2-3 узла для production          |
| withLocalDatacenter | Локальный датацентр      | Обязательно для правильной маршрутизации |
| withKeyspace        | Keyspace по умолчанию    | Опционально, можно указывать в запросах  |
| withAuthCredentials | Логин/пароль             | Обязательно для production               |
| withSslContext      | SSL/TLS конфигурация     | Обязательно для production               |

Важно: Contact points нужны только для начального подключения — после соединения драйвер получает полную информацию о топологии кластера от узлов (через gossip протокол) и сам переключается на нужные узлы.

### Настройка пула соединений и таймаутов

```java
import com.datastax.oss.driver.api.core.config.DefaultDriverOption;
import com.datastax.oss.driver.api.core.config.DriverConfigLoader;

DriverConfigLoader loader = DriverConfigLoader.programmaticBuilder()
        // Максимальное время ожидания ответа на запрос
        .withDuration(DefaultDriverOption.REQUEST_TIMEOUT, Duration.ofSeconds(10))

        // Таймаут на установку соединения с узлом
        .withDuration(DefaultDriverOption.CONNECTION_CONNECT_TIMEOUT, Duration.ofSeconds(5))

        // Таймаут на инициализацию соединения (рукопожатие)
        .withDuration(DefaultDriverOption.CONNECTION_INIT_QUERY_TIMEOUT, Duration.ofSeconds(5))

        // Кол-во соединений на узел в локальном ДЦ (больше → выше параллелизм)
        .withInt(DefaultDriverOption.CONNECTION_POOL_LOCAL_SIZE, 8)

        // Кол-во соединений на узел в удаленных ДЦ (экономим ресурсы)
        .withInt(DefaultDriverOption.CONNECTION_POOL_REMOTE_SIZE, 4)
        .build();

try (CqlSession session = CqlSession.builder()
        .addContactPoint(new InetSocketAddress("127.0.0.1", 9042))
        .withLocalDatacenter("datacenter1")
        .withConfigLoader(loader)
        .build()) {

        // работа с сессией
        }
```

---

### PreparedStatement — безопасные и эффективные запросы

```java
PreparedStatement prepared = session.prepare(
    "INSERT INTO users (id, name, age, email) VALUES (?, ?, ?, ?)"
);

BoundStatement bound = prepared.bind(
    UUID.randomUUID(),
    "Alex",
    30,
    "alex@example.com"
);

session.execute(bound);
```

**Преимущества PreparedStatement:**

1. Запрос компилируется один раз на стороне сервера
2. Защита от CQL-инъекций
3. Меньше трафика (передаются только параметры)
4. Кэширование на стороне драйвера

---

### Асинхронные запросы

```java
import java.util.concurrent.CompletionStage;
import com.datastax.oss.driver.api.core.cql.AsyncResultSet;

CompletionStage<AsyncResultSet> future = session.executeAsync(
    "SELECT * FROM users WHERE user_id = ?", userId
);

future.thenAccept(resultSet -> {
        for (Row row : resultSet.currentPage()) {
            System.out.println(row.getString("name"));
        }
    }).exceptionally(ex -> {
        logger.error("Query failed", ex);
        return null;
});
```

**Когда использовать асинхронность:**

- Высоконагруженные приложения
- Параллельное выполнение независимых запросов
- Неблокирующий I/O

---

## 4. CRUD Операции в CQL

### Create (Вставка)

#### Простая вставка:

```sql
INSERT INTO users (id, name, age, email)
VALUES (a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11, 'Alex', 30, 'alex@example.com');
```

#### Вставка с PreparedStatement в Java:

```java
PreparedStatement insertStmt = session.prepare(
    "INSERT INTO users (id, name, age, email) VALUES (?, ?, ?, ?)"
);

session.execute(insertStmt.bind(
    UUID.randomUUID(),
    "Alex",
    30,
    "alex@example.com"
));
```

#### Вставка с TTL (время жизни записи):

```sql
INSERT INTO sessions (session_id, user_id, data)
VALUES (uuid(), a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11, '{"token":"xyz"}')
USING TTL 3600;  // 1 час
```

#### Вставка с TIMESTAMP (время записи):

```sql
INSERT INTO events (event_id, user_id, event_type, timestamp)
VALUES (uuid(), a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11, 'login', '2024-01-15T10:30:00Z')
USING TIMESTAMP 1705315800000000;  // микросекунды с epoch
```

`uuid()` — это встроенная функция CQL (Cassandra Query Language), которая генерирует случайный UUID версии 4.

---

### Read (Чтение)

#### Чтение по Partition Key (самый эффективный):

```sql
SELECT * FROM users WHERE user_id = a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11;
```

#### Чтение с фильтрацией по Clustering Key:

```sql
SELECT * FROM user_events
WHERE user_id = ?
AND event_time > '2024-01-01'
AND event_time < '2024-01-31';
```

#### Чтение с LIMIT:

```sql
SELECT * FROM users WHERE user_id = ? LIMIT 10;
```

#### Чтение с ORDER BY (только по clustering key):

```sql
SELECT * FROM user_events
WHERE user_id = ?
ORDER BY event_time DESC
LIMIT 100;
```

#### Чтение конкретных колонок (эффективнее):

```sql
SELECT name, email FROM users WHERE user_id = ?;
```

#### Чтение в Java с обработкой результатов:

```java
ResultSet resultSet = session.execute(
    "SELECT id, name, age, email FROM users WHERE user_id = ?",
    userId
);

for (Row row : resultSet) {
    UUID id = row.getUuid("id");
    String name = row.getString("name");
    Integer age = row.getInt("age");
    String email = row.getString("email");

    System.out.printf("%s: %s (%d) - %s%n", id, name, age, email);
}
```

#### Чтение с уровнем согласованности:

```java
Statement statement = new SimpleStatement("SELECT * FROM users WHERE user_id = ?")
        .setConsistencyLevel(ConsistencyLevel.QUORUM);

session.execute(statement.bind(userId));
```

---

### Update (Обновление)

#### Обновление полей:

```sql
UPDATE users
SET age = 31, email = 'newemail@example.com'
WHERE user_id = a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11;
```

#### Обновление с условием (Lightweight Transaction):

```sql
UPDATE users
SET email = 'newemail@example.com'
WHERE user_id = a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11
IF age = 30;
```

```java
ResultSet result = session.execute(statement);
boolean wasApplied = result.wasApplied();  // true если обновление выполнено
```

**Важно:** LWT (Lightweight Transactions) используют Paxos и значительно медленнее обычных операций.

#### Обновление коллекций:

**LIST:**

- `tags + ['new_tag']` — добавляет элемент в **КОНЕЦ** списка
- `['new_tag'] + tags` — добавляет элемент в **НАЧАЛО** списка

```sql
UPDATE users
SET tags = tags + ['new_tag']
WHERE user_id = ?;

UPDATE users
SET tags = tags - ['old_tag']
WHERE user_id = ?;

UPDATE users
SET tags[0] = 'replaced_tag'
WHERE user_id = ?;
```

Важно:

- Оператор `+` работает только с LIST (не с SET или MAP)
- Все элементы в квадратных скобках — это новый список, который присоединяется
- Операция не проверяет уникальность (в LIST могут быть дубликаты)

**SET:**

```sql
UPDATE users
SET permissions = permissions + {'read', 'write'}
WHERE user_id = ?;

UPDATE users
SET permissions = permissions - {'delete'}
WHERE user_id = ?;
```

Ключевые отличия от LIST:

- Фигурные скобки `{}` вместо квадратных `[]`
- Автоматическая уникальность — дубликат не добавится (ошибки не будет — это ожидаемое поведение SET)
- Нет порядка — элементы хранятся неупорядоченно
- Нельзя работать по индексу — `permissions[0]` не работает

**MAP:**

```sql
UPDATE users
SET settings = settings + {'theme': 'dark'}
WHERE user_id = ?;

UPDATE users
SET settings['language'] = 'en'
WHERE user_id = ?;
```

#### Обновление с TTL:

```sql
UPDATE users
SET session_token = 'abc123'
WHERE user_id = ?
USING TTL 3600;
```

После истечения TTL Cassandra помечает запись как удаленную (tombstone), но не удаляет физически сразу.

Процесс:

- TTL истек → Cassandra помечает колонку session_token как удаленную (ставит tombstone)
- При чтении → автоматически фильтрует такие записи (игнорирует)
- При compaction → физически удаляет данные с диска

Что происходит с остальными колонками?

- TTL применяется только к конкретной колонке (session_token)
- Остальные колонки строки (name, email и т.д.) остаются без изменений

Важно:

- Если у колонки был TTL, после истечения она становится null при чтении
- Tomstones хранятся до следующего compaction (по умолчанию до 10 дней)
- Много tombstone'ов могут влиять на производительность

---

### Delete (Удаление)

#### Удаление строки:

```sql
DELETE FROM users WHERE user_id = a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11;
```

#### Удаление конкретных колонок:

```sql
DELETE email, age FROM users WHERE user_id = ?;
```

#### Удаление элемента из коллекции:

```sql
-- LIST: удалить по индексу
DELETE tags[0] FROM users WHERE id = 1;

-- SET: удалить конкретное значение (нужен UPDATE с '-')
UPDATE users SET permissions = permissions - {'read'} WHERE id = 1;

-- MAP: удалить по ключу
DELETE addresses['home'] FROM users WHERE id = 1;

-- Удалить всю колонку
DELETE permissions FROM users WHERE user_id = 1;
```

#### Удаление с условием (LWT):

```sql
DELETE FROM users
WHERE user_id = ?
IF EXISTS;
```

```java
ResultSet result = session.execute(statement);
boolean wasDeleted = result.wasApplied();
```

> LWT (IF EXISTS/IF NOT EXISTS) нужен для атомарной проверки состояния данных перед операцией, 
> а wasApplied() позволяет получить результат этой проверки, тогда как обычный DELETE без условия не даёт информации о том, 
> была ли **реально** удалена существовавшая запись.

#### Удаление с TTL (отложенное удаление):

```sql
DELETE FROM sessions
WHERE session_id = ?
USING TIMESTAMP 1705315800000000;
```

---

## 5. Особенности Моделирования Данных

### Принципы моделирования в Cassandra

**Золотое правило:** Модель данных проектируется под запросы, а не под сущности!

В отличие от RDBMS, где мы нормализуем данные, в Cassandra мы **денормализуем** для оптимизации чтения.

### Anti-Patterns (чего избегать):

| Anti-Pattern                                | Проблема                    | Решение                                              |
|:--------------------------------------------|-----------------------------|------------------------------------------------------|
| Запрос без Partition Key                    | Full cluster scan           | Всегда фильтруйте по partition key                   |
| Unbounded partitions                        | Partition растёт бесконечно | Добавляйте bucketing (день/месяц в key)              |
| Secondary Index на высококардинальных полях | Плохая производительность   | Используйте Materialized Views или отдельные таблицы |
| ALLOW FILTERING                             | Сканирование всех данных    | Перепроектируйте таблицу под запрос                  |
| Слишком много таблиц                        | Сложность поддержки         | Балансируйте между денормализацией и поддержкой      |

### Паттерны моделирования

#### 1. Bucketing (разделение больших партиций):

Bucketing означает:

- Использование составного partition key (user_id, event_date)
- Все данные в одной таблице, но распределены по разным партициям
- Каждый день — отдельная партиция (bucket) внутри таблицы

```sql
CREATE TABLE events_by_day (
    user_id UUID,
    event_date DATE,
    event_time TIMESTAMP,
    event_type TEXT,
    event_data TEXT,
    PRIMARY KEY ((user_id, event_date), event_time)
);

-- Запрос за конкретный день:
SELECT * FROM events_by_day
WHERE user_id = ? AND event_date = '2024-01-15';
```

Плохой подход (без bucketing):

```sql
PRIMARY KEY (user_id, event_time)  -- одна партиция на user_id
-- Все события пользователя в одной партиции (растет бесконечно!)
```

Хороший подход (с bucketing):

```sql
PRIMARY KEY ((user_id, event_date), event_time)  -- партиция на user_id + день
-- Каждый день - отдельный bucket
-- Размер партиции контролируется
```

#### 2. Denormalization (дублирование данных):

**Денормализация** в Cassandra нужна для обеспечения высокой производительности чтения, 
так как она позволяет получать все необходимые данные из одной партиции за один запрос, 
избегая дорогостоящих JOIN-операций, которые Cassandra не поддерживает.

```sql
-- Таблица для просмотра профиля
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT,
    avatar_url TEXT
);

-- Таблица для ленты новостей (дублируем имя пользователя)
CREATE TABLE news_feed (
    feed_id UUID,
    user_id UUID,
    user_name TEXT,  -- Дублируем из user_profiles
    post_content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (feed_id, created_at)
);
```

#### 3. Composite Partition Key (распределение нагрузки):

Composite Partition Key с bucket — это искусственно добавляемое поле в partition key для распределения "горячих" данных по разным узлам кластера, 
чтобы избежать перегрузки одного узла при интенсивной записи.

```sql
CREATE TABLE sensor_data (
    sensor_id UUID,
    bucket INT,  -- 0-9 для распределения
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, bucket), timestamp)
);

-- Запрос требует указания bucket:
SELECT * FROM sensor_data
WHERE sensor_id = ? AND bucket IN (0, 1, 2, 3, 4);
```

#### 4. Time-Series Data (временные ряды):

Паттерн Time-Series Data — это организация хранения событий во времени, где данные партиционируются по временным интервалам для контроля размера, 
а сортируются по убыванию времени для эффективного получения самых новых записей.

```sql
CREATE TABLE metrics (
    metric_name TEXT,
    bucket_date DATE,
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((metric_name, bucket_date), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

#### 5. Many-to-Many через отдельные таблицы:

Паттерн Many-to-Many через отдельные таблицы — это создание двух денормализованных таблиц связей, 
каждая из которых оптимизирована для эффективного запроса в одном направлении: `user_groups` для получения всех групп пользователя и `group_users` для получения всех участников группы.

```sql
-- Пользователи
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT
);

-- Группы
CREATE TABLE groups (
    group_id UUID PRIMARY KEY,
    group_name TEXT
);

-- Связь: пользователь → группы
CREATE TABLE user_groups (
    user_id UUID,
    group_id UUID,
    joined_at TIMESTAMP,
    PRIMARY KEY (user_id, group_id)
);

-- Связь: группа → пользователи
CREATE TABLE group_users (
    group_id UUID,
    user_id UUID,
    joined_at TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);
```

### Secondary Indexes — когда использовать:

```sql
CREATE INDEX ON users(user_role);
CREATE INDEX ON users(country);
```

**Подходит когда:**

- Низкая кардинальность (country, status, boolean)
- Запросы не критичны по производительности
- Поле не является частью primary key

**Не подходит когда:**

- Высокая кардинальность (email, UUID)
- High-write таблицы
- Критичные по latency запросы

**Альтернатива:** Materialized Views или отдельные lookup-таблицы

### Materialized Views:

Materialized View — это автоматически синхронизируемая денормализованная копия таблицы с альтернативным первичным ключом, 
обеспечивающая эффективные запросы к данным по разным измерениям за счёт дополнительной нагрузки на запись.

```sql
CREATE MATERIALIZED VIEW users_by_email AS
SELECT user_id, name, email, age
FROM users
WHERE email IS NOT NULL AND user_id IS NOT NULL
PRIMARY KEY (email, user_id);
```

**Ограничения:**

- Не поддерживают все типы запросов
- Дополнительная нагрузка на запись
- eventual consistency (не мгновенная синхронизация)

---

## 6. Производительность и Внутренние Механизмы

### Как Cassandra обрабатывает запись:

1. Запись приходит в координирующий узел
2. Запись в Commit Log (для durability)
3. Запись в MemTable (в памяти)
4. Подтверждение клиенту (в зависимости от CL (Consistency Level))
5. Асинхронная репликация на другие узлы
6. Periodic flush MemTable → SSTable на диск
7. Compaction объединяет SSTable

### Как Cassandra обрабатывает чтение:

1. Координирующий узел получает запрос
2. Определяет узлы, хранящие данные (по token ring)
3. Отправляет запрос на реплики (по Consistency Level)
4. Собирает ответы, выбирает актуальную версию (last-write-wins)
5. Read Repair при необходимости (синхронизация реплик)
6. Возвращает результат клиенту


### Tunable Consistency в действии:

| Сценарий                | Read CL      | Write CL     | Гарантия                                   |
|-------------------------|--------------|--------------|--------------------------------------------|
| Максимальная скорость   | ONE          | ONE          | Eventual consistency                       |
| Баланс                  | QUORUM       | QUORUM       | Strong consistency                         |
| Максимальная надёжность | ALL          | ALL          | Полная согласованность, низкая доступность |
| Мульти-DC               | LOCAL_QUORUM | LOCAL_QUORUM | Локальная согласованность                  |

### Compaction Strategies:

| Strategy             | Описание                           | Когда использовать        |
|:---------------------|------------------------------------|---------------------------|
| SizeTieredCompaction | Объединяет SSTable схожего размера | Write-heavy, общие случаи |
| LeveledCompaction    | Уровни как в LSM-деревьях          | Read-heavy, low latency   |
| TimeWindowCompaction | По временным окнам                 | Time-series данные с TTL  |

### Explain Plan для запросов:

```sql
TRACE SELECT * FROM users WHERE user_id = ?;
```
Или через драйвер:

```java
ResultSet rs = session.execute("SELECT * FROM users WHERE user_id = ?");
ExecutionInfo info = rs.getExecutionInfo();
QueryTrace trace = info.getQueryTrace();
```

---

## 7. Best Practices для Production

### Подключение и конфигурация

```java
// Production-ready конфигурация
DriverConfigLoader loader = DriverConfigLoader.programmaticBuilder()
        // Таймауты
        .withDuration(DefaultDriverOption.REQUEST_TIMEOUT, Duration.ofSeconds(10))
        .withDuration(DefaultDriverOption.CONNECTION_CONNECT_TIMEOUT, Duration.ofSeconds(5))

        // Пул соединений
        .withInt(DefaultDriverOption.CONNECTION_POOL_LOCAL_SIZE, 8)
        .withInt(DefaultDriverOption.CONNECTION_POOL_REMOTE_SIZE, 4)
        .withInt(DefaultDriverOption.MAX_REQUESTS_PER_CONNECTION, 32768)
    
        // Retry policy
        .withString(DefaultDriverOption.RETRY_POLICY_CLASS, "DefaultRetryPolicy")
        .withInt(DefaultDriverOption.RETRY_POLICY_MAX_RETRIES, 3)
    
        // Load balancing
        .withString(DefaultDriverOption.LOAD_BALANCING_POLICY_CLASS, "DcAwareRoundRobinLoadBalancingPolicy")
        .withString(DefaultDriverOption.LOAD_BALANCING_LOCAL_DC, "dc1")
    
        // Metrics
        .withBoolean(DefaultDriverOption.METRICS_SESSION_ENABLED, true)
        .withBoolean(DefaultDriverOption.METRICS_NODE_ENABLED, true)
    
        .build();

PlainTextAuthProvider authProvider = new PlainTextAuthProvider(
        "cassandra_user",
        "secure_password"
);

try (CqlSession session = CqlSession.builder()
    .addContactPoints(Arrays.asList(
            new InetSocketAddress("node1.example.com", 9042), 
            new InetSocketAddress("node2.example.com", 9042),
            new InetSocketAddress("node3.example.com", 9042)
    ))
        .withLocalDatacenter("dc1")
        .withAuthCredentials("cassandra_user", "secure_password")
        .withConfigLoader(loader)
        .withSslContext(sslContext)  // TLS для production
        .build()) {

    // Работа с сессией
}
```

### Обработка ошибок

```java
import com.datastax.oss.driver.api.core.servererrors.*;
import com.datastax.oss.driver.api.core.AllNodesFailedException;

try {
session.execute(statement);
} catch (NoHostAvailableException e) {
    // Ни один узел не доступен
    logger.error("No Cassandra nodes available", e);
} catch (QueryConsistencyException e) {
    // Не достигнут требуемый уровень согласованности
    if (e instanceof WriteTimeoutException) {
        logger.warn("Write timeout", e);
        // Можно реализовать retry logic
    } else if (e instanceof ReadTimeoutException) {
        logger.warn("Read timeout", e);
    } else if (e instanceof UnavailableException) {
        logger.warn("Not enough replicas available", e);
    }
} catch (QueryValidationException e) {
    // Ошибка в запросе (синтаксис, схема)
    logger.error("Query validation failed", e);
} catch (AllNodesFailedException e) {
    // Все узлы вернули ошибки
    logger.error("All nodes failed", e);
} catch (DriverException e) {
    // Ошибка драйвера
    logger.error("Driver error", e);
}
```

### Retry Policy — когда и как retry'ить:

| Ошибка                           | Retry? | Стратегия               |
|----------------------------------|--------|-------------------------|
| ReadTimeout (достаточно ответов) | Да     | Retry на том же узле    |
| WriteTimeout (unknown)           | Нет    | Требует ручной проверки |
| Unavailable                      | Да     | Retry на другом узле    |
| NoHostAvailable                  | Нет    | Требуется вмешательство |
| QueryValidationException         | Нет    | Исправить запрос        |

### Repository Pattern для Cassandra:

```java
public class UserRepository {

    private final CqlSession session;
    private final PreparedStatement findById;
    private final PreparedStatement insert;
    private final PreparedStatement update;
    private final PreparedStatement delete;
    
    public UserRepository(CqlSession session) {
        this.session = session;
        
        // Prepared statements инициализируются один раз
        this.findById = session.prepare(
            "SELECT * FROM users WHERE user_id = ?"
        );
        
        this.insert = session.prepare(
            "INSERT INTO users (user_id, name, email, age) " +
            "VALUES (?, ?, ?, ?)"
        );
        
        this.update = session.prepare(
            "UPDATE users SET name = ?, email = ?, age = ? " +
            "WHERE user_id = ?"
        );
        
        this.delete = session.prepare(
            "DELETE FROM users WHERE user_id = ?"
        );
    }
    
    public Optional<User> findById(UUID userId) {
        Row row = session.execute(findById.bind(userId)).one();
        return Optional.ofNullable(row).map(this::mapToUser);
    }
    
    public void save(User user) {
        session.execute(insert.bind(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getAge()
        ));
    }
    
    public boolean update(User user) {
        ResultSet rs = session.execute(update.bind(
            user.getName(),
            user.getEmail(),
            user.getAge(),
            user.getId()
        ));
        return rs.wasApplied();
    }
    
    public boolean delete(UUID userId) {
        ResultSet rs = session.execute(delete.bind(userId));
        return rs.wasApplied();
    }
    
    private User mapToUser(Row row) {
        return new User(
            row.getUuid("user_id"),
            row.getString("name"),
            row.getString("email"),
            row.getInt("age")
        );
    }
}
```

### Object Mapping (Mapper Annotations):

```java
import org.apache.cassandra.datastax.driver.mapping.annotations.*;

@Table(name = "users", keyspace = "myapp")
public class User {

    @PartitionKey
    private UUID userId;
    
    @Column
    private String name;
    
    @Column
    private String email;
    
    @Column
    private Integer age;
    
    // getters/setters
}

// Использование через Mapper
User user = new User();
user.setUserId(UUID.randomUUID());
user.setName("Alex");

// 1. Инициализируем маппер через MappingManager
MappingManager mappingManager = new MappingManager(session);
Mapper<User> userMapper = mappingManager.mapper(User.class);

// 2. Используем готовые методы
userMapper.save(user);                    // INSERT/UPDATE

User found = userMapper.get(userId);      // SELECT

userMapper.delete(userId);                // DELETE

// 3. Если нужен Statement (для транзакций или TTL):
Statement stmt = userMapper.saveQuery(user).setIdempotent(true);
session.execute(stmt);
```

### Тестирование с TestContainers:

```java
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;

public class CassandraTestContainer {

    private static GenericContainer<?> cassandra;
    private static CqlSession session;
    
    @BeforeAll
    static void startCassandra() {
        cassandra = new GenericContainer<>(
            DockerImageName.parse("cassandra:5.0")
        )
        .withExposedPorts(9042)
        .withEnv("CASSANDRA_CLUSTER_NAME", "test_cluster")
        .withEnv("CASSANDRA_DC", "datacenter1");
        
        cassandra.start();
        
        session = CqlSession.builder()
            .addContactPoint(new InetSocketAddress(
                cassandra.getHost(),
                cassandra.getMappedPort(9042)
            ))
            .withLocalDatacenter("datacenter1")
            .build();
    }
    
    @AfterAll
    static void stopCassandra() {
        if (session != null) session.close();
        if (cassandra != null) cassandra.stop();
    }
    
    @BeforeEach
    void cleanup() {
        session.execute("TRUNCATE TABLE myapp.users");
    }
}
```

### Логирование и мониторинг

#### Логирование драйвера (logback.xml):

<logger name="com.datastax.oss.driver" level="INFO"/>
<logger name="com.datastax.oss.driver.core.pool" level="DEBUG"/>
<logger name="com.datastax.oss.driver.core.query" level="DEBUG"/>

#### Метрики для мониторинга:

- Session metrics (requests, errors, latency)
- Node metrics (pool size, in-flight requests)
- CQL metrics (prepared statements, query traces)

#### Интеграция с Prometheus:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.12.0</version>
</dependency>
```

### Безопасность

#### TLS/SSL подключение:

```java
import javax.net.ssl.SSLContext;

SSLContext sslContext = SSLContextBuilder.create()
        .loadTrustMaterial(new File("/path/to/truststore.jks"), 
                "truststore_password".toCharArray())
        .loadKeyMaterial(new File("/path/to/keystore.jks"), 
                "keystore_password".toCharArray(), 
                "key_password".toCharArray())
        .build();

CqlSession session = CqlSession.builder()
        .addContactPoint(new InetSocketAddress("host", 9042))
        .withSslContext(sslContext)
        .build();
```

#### RBAC (Role-Based Access Control):

Гранты (GRANT) — это разрешения (права доступа), которые выдаются ролям на выполнение определенных операций с объектами базы данных.

Основные типы грантов в Cassandra:

- `SELECT` - Чтение данных (SELECT)
- `MODIFY` - Изменение данных (INSERT, UPDATE, DELETE)
- `CREATE` - Создание объектов (таблиц, индексов)
- `ALTER` - Изменение структуры объектов
- `DROP` - Удаление объектов
- `AUTHORIZE` - Управление разрешениями (GRANT/REVOKE)
- `ALL` - Все разрешения

Создание роли:

```sql
CREATE ROLE app_user WITH PASSWORD = 'secret' AND LOGIN = true;
```

Уровни объектов:

```sql
-- На уровне ключейпейса
GRANT SELECT ON KEYSPACE myapp TO app_user;

-- На уровне конкретной таблицы
GRANT MODIFY ON TABLE myapp.users TO app_user;

-- На уровне всех функций
GRANT EXECUTE ON ALL FUNCTIONS TO app_user;
```

Проверка грантов:

```sql
-- Посмотреть права роли
LIST ALL PERMISSIONS OF app_user;
```

Ограничение по IP (через cassandra.yaml):

```yaml
start_rpc: true
rpc_address: 0.0.0.0
client_encryption_options: enabled
```

### Чек-лист перед Production:

- [ ] Репликация factor >= 3
- [ ] NetworkTopologyStrategy для мульти-DC
- [ ] TLS/SSL включён
- [ ] Аутентификация настроена
- [ ] RBAC настроен с минимальными правами
- [ ] Monitoring и alerting настроены
- [ ] Backup стратегия определена
- [ ] Compaction стратегия выбрана под workload
- [ ] GC grace seconds настроен
- [ ] Hinted handoff включён
- [ ] Load balancing policy настроен
- [ ] Retry policy настроен
- [ ] Connection pool оптимизирован
- [ ] Prepared statements используются
- [ ] No ALLOW FILTERING в production запросах
- [ ] Partition sizes проверены (< 100MB)

---

## Приложения

### Частые CQL команды

```sql
-- Список keyspace
DESCRIBE KEYSPACES;

-- Информация о keyspace
DESCRIBE KEYSPACE myapp;

-- Список таблиц
DESCRIBE TABLES;

-- Структура таблицы
DESCRIBE TABLE users;

-- Индексы
DESCRIBE INDEXES;

-- Статус кластера
nodetool status

-- Информация об узле
nodetool info
```

### Полезные ссылки

- Официальная документация: https://cassandra.apache.org/doc/latest/
- DataStax Driver Docs: https://docs.datastax.com/en/developer/java-driver/
- CQL Reference: https://cassandra.apache.org/doc/latest/cql/index.html
- GitHub драйвера: https://github.com/datastax/java-driver
- Cassandra Best Practices: https://cassandra.apache.org/doc/latest/cassandra/getting_started/best_practices.html

---