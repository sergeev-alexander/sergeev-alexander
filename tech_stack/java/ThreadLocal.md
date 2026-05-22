**Copyright (c) 2023-2026 sergeev-alexander**

All Rights Reserved.

This work is private property and is not licensed for copying, distribution, modification, or any other use without the explicit written permission of the author.

Данные материалы являются частной собственностью и не подлежат копированию, распространению, изменению или любому другому использованию без явного письменного разрешения автора.

---

[Введение в ThreadLocal](#1-введение-в-threadlocal)
[Базовый API и паттерны использования](#2-базовый-api-и-паттерны-использования)
[Типичные ошибки и подводные камни](#3-типичные-ошибки-и-подводные-камни)
[ThreadLocal и виртуальные потоки (Java 21+)](#4-threadLocal-и-виртуальные-потоки-java-21+)
[Best Practices и итоговый чек-лист](#5-best-practices-и-итоговый-чек-лист)

---

# 1. Введение в ThreadLocal

> `ThreadLocal` — это утилитарный класс Java, предоставляющий механизм хранения данных, 
> привязанных к конкретному потоку выполнения. 
> 
> Каждый поток получает собственную, независимую копию переменной, доступ к которой возможен только изнутри этого потока.

---

## Какую проблему решает ThreadLocal

> В многопоточных приложениях (веб-серверы, пулы задач, микросервисы) потоки часто обрабатывают независимые запросы параллельно. 
> 
> При использовании обычных `static` или инстансных полей возникает состояние гонки (`race condition`), 
> требующее синхронизации (`synchronized`, `ReentrantLock`), что снижает пропускную способность и усложняет логику.

- **Устранение блокировок** - Данные изолированы на уровне потока, что полностью исключает необходимость синхронизации доступа к ним.
- **Прозрачная передача контекста** - Позволяет передавать ID запроса, данные авторизации, локаль или трейсинг-идентификаторы 
  через глубокие цепочки вызовов без изменения сигнатур методов.
- **Гарантия потокобезопасности** - Обеспечивается архитектурно: потоки физически не видят значения друг друга, 
  даже если ссылаются на один и тот же экземпляр `ThreadLocal`.

---

## Механика изоляции данных на уровне JVM

> Внутри JVM реализация `ThreadLocal` завязана на внутренний класс `Thread.ThreadLocalMap`. 
> 
> Это хеш-таблица, где **ключом** выступает сама ссылка на экземпляр `ThreadLocal`, а **значением** — привязанные к нему данные. 
> 
> Карта хранится не в `ThreadLocal`, а непосредственно в объекте потока (`Thread`), что обеспечивает строгую изоляцию.

```text
┌─────────────────────────────────────────────────────────────────────┐
│        ВИЗУАЛИЗАЦИЯ СВЯЗИ Thread → ThreadLocalMap → Entry           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐      ┌─────────────────────────────────────┐   │
│  │   Thread #1     │      │  Thread.ThreadLocalMap (в Thread)   │   │
│  │  (worker-1)     │      │                                     │   │
│  │                 │      │  ┌─────────────────────────────┐    │   │
│  │  threadLocals ──┼─────►│  │  Entry [WeakRef Key]        │    │   │
│  │                 │      │  │  Key: ThreadLocal<UserCtx>  │    │   │
│  └─────────────────┘      │  │  Val: UserContext(id=42)    │    │   │
│                           │  └─────────────────────────────┘    │   │
│  ┌─────────────────┐      │                                     │   │
│  │   Thread #2     │      │  ┌─────────────────────────────┐    │   │
│  │  (worker-2)     │      │  │  Entry [WeakRef Key]        │    │   │
│  │                 │      │  │  Key: ThreadLocal<ReqId>    │    │   │
│  │  threadLocals ──┼─────►│  │  Val: "req-abc-987"         │    │   │
│  │                 │      │  └─────────────────────────────┘    │   │
│  └─────────────────┘      └─────────────────────────────────────┘   │
│                                                                     │
│  ВАЖНО: Карта принадлежит потоку. Ключи хранятся как WeakReference. │
└─────────────────────────────────────────────────────────────────────┘
```

#### Пример декларативной настройки и базовой изоляции:

```java
class Example {
    
    // 1. Создаём экземпляр ThreadLocal. Обычно это static final field класса.
    // Сам объект ThreadLocal потокобезопасен и может шериться между потоками.
    private static final ThreadLocal<UserContext> USER_CTX = ThreadLocal.withInitial(() -> new UserContext());

    public void processRequest(String userId) {
        // 2. set() привязывает объект к текущему executing thread.
        // JVM кладёт значение в Thread.currentThread().threadLocals
        USER_CTX.set(new UserContext(userId));

        // 3. get() извлекает значение ТОЛЬКО для текущего потока.
        // Другие потоки вызовут с тем же USER_CTX получат свои, изолированные экземпляры.
        UserContext ctx = USER_CTX.get();

        // 4. Выполняем бизнес-логику без передачи ctx в аргументах методов.
        // Главное преимущество ThreadLocal — скрытую передачу контекста (implicit context) через всю цепочку вызовов, 
        // без необходимости явно пробрасывать объект через все методы.
        deepNestedService.execute();

        // 5. remove() ОБЯЗАТЕЛЕН при работе с пулами потоков!
        // Без этого следующий таск, получивший этот поток, увидит старый контекст.
        USER_CTX.remove();
    }
}
```

- **`ThreadLocal` объект** - Является точкой входа и ключом для маппинга. Сам не хранит данные.
- **`Thread.ThreadLocalMap`** - Создаётся лениво при первом вызове `set()` или `get()`. Принадлежит конкретному `Thread`.
- **`Entry`** - Внутренняя ячейка карты. Ключ оборачивается в `WeakReference<ThreadLocal<?>>`, что позволяет GC удалять записи, 
  если сам `ThreadLocal` стал недостижим. 

  Значение остаётся `strong reference`, поэтому **требует явной очистки**.

---

## Архитектурное применение: когда это оправдано

- `Web Request Context` - Хранение идентификатора HTTP-запроса, tenant-ID или заголовков авторизации для логирования и аутентификации.
- `Non-Thread-Safe Utils` - Изоляция тяжёлых или не потокобезопасных объектов (`SimpleDateFormat`, `MessageDigest`, кастомные пулы буферов) для избежания `synchronized` overhead.
- `Framework State` - Реализация `SecurityContextHolder` (Spring Security), хранение транзакционных статусов, сессий `JPA EntityManager` или `Hibernate Session`.
- `Tracing & Profiling` - Накопление метрик времени выполнения или span-идентификаторов в рамках одного потока с последующим агрегированным логированием.

---

## Когда использование ThreadLocal становится антипаттерном

- `Global State Store` - Превращение `ThreadLocal` в глобальное хранилище состояния без чёткой привязки к жизненному циклу задачи или запроса.
- `Implicit Dependencies` - Скрытая передача контекста, при которой методы начинают неявно зависеть от `ThreadLocal`. 
  Это ломает тестируемость, усложняет рефакторинг и нарушает принцип явных зависимостей.
- `Thread Pool Leak` - Забытый вызов `remove()` в `ExecutorService` или веб-контейнере. 
  Приводит к утечкам памяти (Old Gen) и "перетеканию" контекста между несвязанными задачами.
- `Async/Reactive Mismatch` - Использование в `CompletableFuture`, `Project Reactor` или `Kotlin Coroutines`, 
  где логическая задача мигрирует между физическими потоками. Контекст либо теряется, либо копируется некорректно.

---

# 2. Базовый API и паттерны использования

> `ThreadLocal` предоставляет минималистичный API, состоящий всего из нескольких методов.
> 
> Несмотря на синтаксическую простоту, корректное комбинирование этих операций требует понимания их влияния 
> на внутреннюю структуру `ThreadLocalMap` и жизненный цикл потока.

---

## Полный перечень методов ThreadLocal

- `T get()` - извлекает значение, привязанное к текущему потоку. 
  
  Если значение отсутствует и метод `initialValue()` переопределён или передан `Supplier`, возвращает результат его ленивого вызова.

- `void set(T value)` - привязывает переданный объект к текущему потоку, создавая новую или перезаписывая существующую запись в `ThreadLocalMap`.
- `void remove()` - удаляет привязку текущего потока из карты. Освобождает значение и предотвращает накопление памяти в долгоживущих пулах потоков.
- `protected T initialValue()` - метод для ленивой инициализации значения по умолчанию при первом вызове `get()`. 
  В современных версиях заменён фабричным методом.
- `static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier)` - фабричный метод (Java 8+). 

  Создаёт экземпляр `ThreadLocal`, инициализирующий значение через переданную лямбду при первом обращении.
  
- `protected T childValue(T parentValue)` - метод класса `InheritableThreadLocal`. 

  Определяет логику копирования или трансформации значения из родительского потока в дочерний.
  
- `ThreadLocalMap getMap(Thread t)` - package-private метод. Возвращает прямую ссылку на карту конкретного потока для внутреннего использования.
- `void setInitialValue()` - package-private метод. Вызывает `initialValue()` и записывает результат в карту потока. 

  Вызывается автоматически при `get()`, если значение ещё не установлено.

---

## Механика работы методов (с примерами кода)

### Инициализация: initialValue() и withInitial()

> При первом вызове `get()` для потока, у которого ещё нет привязанного значения, JVM обращается к логике инициализации. 
>
> До Java 8 требовалось создавать анонимный класс и переопределять `initialValue()`. 
>
> Начиная с Java 8, используется `withInitial()`, который принимает `Supplier`. 
>
> Это значение создаётся **лениво** (только при первом чтении) и **изолированно** (по одному экземпляру на поток).

### Запись и чтение: set() и get()

> Метод `set()` помещает объект в `ThreadLocalMap` текущего потока. 
> 
> Ключом выступает сам экземпляр `ThreadLocal`, значением — переданный объект. 
> 
> Метод `get()` выполняет поиск по ключу в карте текущего потока. 
> 
> Если запись найдена, возвращается значение. Если нет, срабатывает логика инициализации. 
> 
> **Важно:** вызов `set()` не влияет на другие потоки, даже если они используют ту же переменную `ThreadLocal`.

### Очистка: remove()

> Метод `remove()` находит запись в `ThreadLocalMap` текущего потока по ключу `this` и удаляет её. 
> 
> Значение становится доступным для сборщика мусора. 
> 
> Без вызова `remove()` в средах с переиспользованием потоков (веб-серверы, `ExecutorService`) объекты остаются в памяти навсегда.

---

```text
┌───────────────────────────────────────────────────────────────────────────┐
│                 ЖИЗНЕННЫЙ ЦИКЛ ThreadLocal в потоке                       │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. ОБЪЯВЛЕНИЕ                                                            │
│     static final ThreadLocal<Context> CTX = ThreadLocal.withInitial(...); │
│                                                                           │
│  2. ПЕРВЫЙ ВЫЗОВ get()                                                    │
│     └─► Проверяет Thread.currentThread().threadLocals                     │
│     └─► Запись отсутствует                                                │
│     └─► Вызывает Supplier / initialValue()                                │
│     └─► Создаёт Entry [WeakRef Key, Strong Ref Value]                     │
│     └─► Возвращает новое значение                                         │
│                                                                           │
│  3. ВЫЗОВ set(newValue)                                                   │
│     └─► Находит Entry по ключу                                            │
│     └─► Перезаписывает Strong Ref Value на newValue                       │
│     └─► Старое значение помечается на GC (если на него нет других ссылок) │
│                                                                           │
│  4. ВЫЗОВ remove()                                                        │
│     └─► Находит Entry по ключу                                            │
│     └─► Удаляет Entry из таблицы                                          │
│     └─► WeakRef Key и Strong Ref Value теряют привязку к потоку           │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

```java
public class ThreadLocalLifecycleDemo {

    // 1. Создаём экземпляр с Supplier. 
    // Объект StringBuilder НЕ создаётся сразу. Создание произойдёт при первом get() в каждом потоке.
    private static final ThreadLocal<StringBuilder> LOG_BUFFER = 
            ThreadLocal.withInitial(() -> new StringBuilder());

    public void processTask(String taskId) {
        // 2. Проверяем, есть ли уже данные в контексте текущего потока
        // Если поток новый, get() автоматически вызовет new StringBuilder()
        StringBuilder buffer = LOG_BUFFER.get();
        
        // 3. Наполняем буфер данными текущей задачи
        buffer.append("Task ").append(taskId).append(" started at ")
              .append(System.currentTimeMillis()).append("\n");
              
        // 4. Явно перезаписываем значение (для демонстрации работы set())
        // Старый StringBuilder станет недостижимым для текущего потока
        // set() полностью заменяет объект в хранилище потока, а не модифицирует существующий
        LOG_BUFFER.set(new StringBuilder().append("OVERRIDDEN: ").append(taskId));
        
        // 5. Читаем актуальное состояние
        String currentLog = LOG_BUFFER.get().toString();
        System.out.println(currentLog);
        
        // 6. ОБЯЗАТЕЛЬНАЯ ОЧИСТКА
        // Без этой строки пул потоков вернёт этот поток следующей задаче, 
        // и она получит "загрязнённый" лог от предыдущего таска.
        LOG_BUFFER.remove();
    }
}
```

---

## Типичные сценарии применения

- `Контекст HTTP-запроса` - Хранение `requestId`, `traceId` или заголовков авторизации для сквозного логирования. 
  Позволяет не прокидывать объект контекста через 10 уровней сервисных методов.
- `Потокобезопасные форматтеры (SimpleDateFormat)` - Класс `SimpleDateFormat` не потокобезопасен. 
  Обернув его в `ThreadLocal`, каждый поток получает собственный экземпляр, что устраняет `ConcurrentModificationException` 
  без накладных расходов на `synchronized`.
- `Транзакционные данные` - В `JDBC` или `JPA` хранение `Connection` или `EntityManager` в `ThreadLocal` 
  позволяет автоматически привязывать все репозитории и DAO-слои к одной транзакции в рамках обработки запроса.
- `SecurityContext` - Реализация паттерна в Spring Security. Аутентификация проверяется на входе (фильтры), 
  объект `Authentication` кладётся в контекст, а контроллеры и сервисы читают его через статический `SecurityContextHolder.getContext()`.
- `Тестирование` (изоляция ресурсов) - При параллельном запуске тестов каждый поток получает собственный экземпляр ресурса 
  (например, браузер, подключение к БД, HTTP-клиент). 
  Это позволяет запускать тесты изолированно, без пересечения состояния между разными сьютами или классами.

#### Пример: потокобезопасный пул форматтеров дат

```java
public class DateFormatProvider {

    // Каждый поток получит свой экземпляр SimpleDateFormat при первом обращении
    private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT = 
            ThreadLocal.withInitial(() -> new SimpleDateFormat("dd.MM.yyyy HH:mm:ss"));

    public static String format(Date date) {
        // get() гарантирует, что мы работаем с изолированным экземпляром
        return DATE_FORMAT.get().format(date);
    }

    public static Date parse(String dateStr) throws ParseException {
        // Безопасно парсим строку без блокировок
        return DATE_FORMAT.get().parse(dateStr);
    }

    // Метод очистки рекомендуется вызывать в конце жизненного цикла задачи
    public static void cleanup() {
        DATE_FORMAT.remove();
    }
}
```

#### Пример: изоляция ресурсов в параллельных тестах

```java
// parallel="methods"
public class ThreadSafeTest {

    // Каждый поток видит своё значение, потому что ThreadLocal обращается к внутренней карте текущего потока
    private static ThreadLocal<UserService> userService =
            ThreadLocal.withInitial(() -> new UserService());

    @BeforeMethod
    public void setUp() {
        // Каждый поток получает свой экземпляр
        userService.get().reset();
    }

    @Test
    public void test1() {
        userService.get().create("User1");
    }

    @Test
    public void test2() {
        userService.get().create("User2");
    }

    @AfterMethod
    public void tearDown() {
        userService.remove(); // Очистка ThreadLocal
    }
}
```

---

- **Ленивая инициализация:** Значения создаются только при первом вызове `get()` для конкретного потока, 
  что экономит ресурсы при холодном старте.
- **Запись перезаписывает:** Вызов `set()` заменяет старое значение в `ThreadLocalMap`. 
  Ссылка на старый объект освобождается для GC, если на неё больше нет внешних ссылок.
- **Очистка обязательна в пулах:** В средах с переиспользованием потоков (`Tomcat`, `ExecutorService`) отсутствие `remove()` 
  приводит к утечкам памяти и перекрёстному загрязнению контекста.
- **API минимален, но строг:** Правильная последовательность `get/set -> бизнес-логика -> remove` является 
  единственным способом безопасной эксплуатации `ThreadLocal` в production.

---

# 3. Типичные ошибки и подводные камни

> Несмотря на простоту API, `ThreadLocal` часто становится источником труднодиагностируемых багов в production. 
> 
> Основные риски связаны с жизненным циклом потоков в пулах, неявным наследованием контекста 
> и архитектурными ограничениями асинхронных сред.

---

## Утечки памяти в пулах потоков

> Внутренняя структура `ThreadLocalMap` использует `WeakReference` для **ключей**, но `StrongReference` для **значений**. 
> 
> Это создаёт асимметрию: если экземпляр `ThreadLocal` становится недостижимым и собирается GC, ключ в `Entry` превращается в `null`, 
> но значение остаётся в памяти до тех пор, пока поток не вызовет `set()`, `get()` или `remove()` для этой карты. 
> 
> В долгоживущих пулах потоков (`ExecutorService`, `Tomcat`) это приводит к накоплению "осиротевших" `Entry` и утечкам в Old Gen.

```java
public class MemoryLeakDemo implements Runnable {

    // Потенциально тяжёлый объект
    private static final ThreadLocal<byte[]> CACHE = ThreadLocal.withInitial(() -> new byte[1024 * 1024]);

    @Override
    public void run() {
        // 1. Записываем данные в контекст
        CACHE.set(new byte[1024 * 1024]); // 1MB на поток
        
        // ... бизнес-логика ...
        
        // ОШИБКА: remove() забыт.
        // Поток возвращается в пул, но 1MB остаётся привязанным к нему навсегда.
        // При повторном использовании потока старый массив всё ещё висит в памяти.
    }
}

// ИСПРАВЛЕНИЕ: гарантированная очистка в блоке finally
public class SafeLeakDemo implements Runnable {
    
    private static final ThreadLocal<byte[]> CACHE = ThreadLocal.withInitial(() -> new byte[0]);

    @Override
    public void run() {
        try {
            CACHE.set(new byte[1024 * 1024]);
            process();
        } finally {
            // Всегда вызываем remove(), даже если вылетело исключение
            CACHE.remove(); 
        }
    }
}
```

- **Почему происходит:** Пул потоков переиспользует `Thread` объекты. Карта не очищается автоматически между задачами.
- **Как избежать:** Строгий `try-finally` с `remove()`, либо использование фреймворковых обёрток
  (Spring `TransactionSynchronizationManager`, Servlet `Filter`).

---

## `InheritableThreadLocal`: нюансы наследования

> `InheritableThreadLocal` позволяет дочернему потоку получить копию значения родительского потока на момент создания. 
> 
> Полезно для трейсинга или MDC-логирования. Однако в пулах потоков это создаёт иллюзию изоляции: 
> дочерний поток наследует контекст **потока-создателя пула**, а не **потока, отправившего задачу**.

```java
public class InheritablePitfallDemo {

    // Наследуемая переменная
    private static final InheritableThreadLocal<String> TRACE_ID = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        // 1. Устанавливаем контекст в главном потоке (который создаёт пул)
        TRACE_ID.set("main-trace");
        
        ExecutorService pool = Executors.newFixedThreadPool(1);
        
        for (int i = 0; i < 3; i++) {
            // 2. Меняем контекст в потоке, который отправляет задачу
            TRACE_ID.set("request-" + i);
            
            pool.submit(() -> {
                // ❌ Здесь ВСЕГДА выводится "main-trace", а не "request-N"
                // Потому что worker-thread унаследовал значение при первом старте пула
                System.out.println("Worker sees: " + TRACE_ID.get());
            });
        }
        pool.shutdown();
    }
}
```

- **Когда уместно:** Создание новых потоков "на лету" (не в пуле) для фоновых задач, требующих передачи контекста.
- **Когда опасно:** Любая работа с `ThreadPoolExecutor`, `ForkJoinPool` или веб-контейнерами.

  Для пулов используйте явную передачу контекста в аргументах `Runnable`/`Callable` (через кастомную имплементацию).

---

## Проблемы в асинхронных/реактивных моделях

> `ThreadLocal` жёстко привязан к **физическому потоку** (`Thread.currentThread()`). 
> 
> В асинхронном программировании (`CompletableFuture`, `Project Reactor`, `Kotlin Coroutines`) логическая задача 
> может стартовать в Thread-1, продолжить работу в Thread-2 после `await`/`flatMap`, и завершиться в Thread-3. 
> 
> Контекст остаётся в Thread-1 и теряется для последующих этапов цепочки.

```java
public class AsyncContextLossDemo {
    
    private static final ThreadLocal<String> USER_CTX = ThreadLocal.withInitial(() -> "anonymous");

    public CompletableFuture<String> handleRequest(String userId) {
        // Устанавливаем контекст в потоке веб-сервера
        USER_CTX.set(userId);

        return CompletableFuture.supplyAsync(() -> {
            // Этот код выполняется в потоке из ForkJoinPool.commonPool()
            // USER_CTX.get() вернёт значение по умолчанию ("anonymous"), 
            // а не "userId", потому что это другой физический поток.
            String ctx = USER_CTX.get(); 
            return fetchData(ctx);
        });
    }
}
```

- **Архитектурное ограничение:** `ThreadLocal` не поддерживает логическую привязку к задаче, только к потоку ОС/JVM.
- **Решение:** Явная передача контекста через цепочку `CompletableFuture.thenApply()`, 
  использование `Context` в Reactor/WebFlux, 
  или фреймворковые средства (`MDC.put()` + `MDCContext` для Logback в асинхронных средах).

---

### Ключевые выводы

- **Память не чистится сама:** В пулах потоков `WeakReference` по ключу не спасает от утечки значений. 
  `remove()` в `finally` — обязательный паттерн.
- **Наследование ломается в пулах:** `InheritableThreadLocal` копирует состояние только при `new Thread()`. 
  Для `ExecutorService` это антипаттерн, требующий явной передачи контекста.
- **Асинхронность убивает контекст:** Логическая задача, мигрирующая между потоками, теряет привязку к `ThreadLocal`. 
  В реактивных и `async/await` средах используйте явную передачу состояния или специализированные механизмы.
- **Отладка утечек:** При подозрении на утечку `ThreadLocal` используйте `-XX:+HeapDumpOnOutOfMemoryError` 
  и анализаторы (Eclipse MAT, VisualVM), ища цепочки `Thread → threadLocals → Entry → value`.

---

Контекст: Создаём справочную памятку по `ThreadLocal` в Java (ориентир Java 8+, с учётом Java 21+). Целевая аудитория — Junior+ (образовательный уклон, но универсальный справочный формат). Структура согласована: 1. Введение, 2. API и паттерны, 3. Ошибки, 4. Виртуальные потоки, 5. Best Practices. Форматирование строго по `formatting_rules.md`. Возвращаю один раздел за раз.

# 4. ThreadLocal и виртуальные потоки (Java 21+)

> Виртуальные потоки (Virtual Threads, Project Loom) фундаментально меняют модель многопоточности в JVM. 
> 
> Они легковесны, масштабируются до миллионов одновременно, но требуют переосмысления работы с привязанным к потоку состоянием, 
> включая `ThreadLocal`.

---

## Как изменилась модель привязки данных к `VirtualThread`

> В классической модели (Platform Threads) каждый объект `Thread` мапится 1-к-1 на поток ОС. 
> 
> `ThreadLocal` хранит данные прямо в поле `Thread.threadLocals`. 
> 
> Виртуальные потоки используют другую архитектуру: они являются задачами (`ForkJoinTask`), 
> которые планируются поверх небольшого пула потоков-носителей (Carrier Threads). 
> 
> При парковке (ожидании I/O или `Lock`) виртуальный поток отцепляется от одного носителя и может продолжить выполнение на другом.
> 
> Это означает, что `ThreadLocal` **продолжает работать** в Java 21+, но его семантика смещается: 
> привязка сохраняется за логическим виртуальным потоком, а не за физическим ядром CPU. 
> 
> Однако стоимость хранения миллионов `ThreadLocalMap` в памяти растёт нелинейно, 
> а GC сталкивается с повышенным давлением при очистке слабых ссылок.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│          СРАВНЕНИЕ МОДЕЛЕЙ: Platform Threads vs Virtual Threads             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PLATFORM THREAD (Java 20-):                                                │
│  ┌──────────────┐  1:1  ┌──────────────┐                                    │
│  │  Thread Java │ ─────►│  Thread OS   │                                    │
│  │  threadLocals│       │  Stack: 1MB  │                                    │
│  └──────────────┘       └──────────────┘                                    │
│  Статичная привязка. Высокий overhead при >10k потоков.                     │
│                                                                             │
│  VIRTUAL THREAD (Java 21+):                                                 │
│  ┌──────────────┐  M:1  ┌──────────────┐                                    │
│  │  VirtualThd  │       │ Carrier Pool │                                    │
│  │  threadLocals│ ◄────►│ (Platform)   │                                    │
│  └──────────────┘  ┌────┤ ForkJoinPool │                                    │
│                    │    └──────────────┘                                    │
│                    ▼                                                        │
│  При блокировке (sleep/io/lock) виртуальный поток отцепляется (unmount).    │
│  При разблокировке подхватывается любым свободным носителем (remount).      │
│  ThreadLocalMap мигрирует вместе с задачей, но создаёт overhead для GC.     │
└─────────────────────────────────────────────────────────────────────────────┘
```

```java
public class VirtualThreadLocalDemo {
    
    // В Java 21+ этот экземпляр будет создавать карту для каждого виртуального потока
    private static final ThreadLocal<String> CONTEXT = ThreadLocal.withInitial(() -> "default");

    public static void main(String[] args) {
        // Создаём 500_000 виртуальных потоков
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 500_000; i++) {
                final int taskId = i;
                executor.submit(() -> {
                    // Каждый поток получает изолированную копию
                    CONTEXT.set("task-" + taskId);
                    
                    // Имитация блокирующего вызова
                    // Виртуальный поток отцепится от носителя, контекст сохранится
                    sleep(Duration.ofMillis(10)); 
                    
                    // После remount контекст всё ещё доступен
                    System.out.println(Thread.currentThread() + " sees: " + CONTEXT.get());
                    
                    // Очистка обязательна даже в виртуальных потоках, 
                    // иначе карта будет жить до завершения пула/приложения
                    CONTEXT.remove();
                });
            }
        }
    }

    private static void sleep(Duration d) {
        try {
            Thread.sleep(d); 
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); 
        }
    }
}
```

- **Полная совместимость:** Существующий код с `ThreadLocal` запускается на Java 21+ без изменений компиляции.
- **Скрытая цена:** Миллионы `ThreadLocalMap` потребляют значительную кучу. 
  Каждый `Entry` содержит `WeakReference` + `Object` header + padding. 
  GC вынужден обходить миллионы слабых ссылок, что увеличивает паузы.
- **Не для реактивности:** Виртуальные потоки решают ту же задачу, что и реактивные модели, но через синхронный код. 
  Использовать `ThreadLocal` внутри них допустимо, но требует дисциплины очистки.

---

## Проблема pinning и совместимость с blocking I/O

> Pinning (закрепление) возникает, когда виртуальный поток держит `synchronized` блок или нативный вызов (JNI) в момент блокировки. 
> 
> JVM не может отцепить (`unmount`) такой поток от носителя, потому что стек и мониторы привязаны к физической памяти ОС. 
> 
> Носитель блокируется целиком, что сводит на нет преимущества виртуальных потоков.
> 
> `ThreadLocal` сам по себе не вызывает pinning, но часто используется в легаси-коде, 
> который оборачивает блокирующие операции в `synchronized` или держит `ReentrantLock` без try-with-resources. 
> 
> При миграции на виртуальные потоки такие паттерны становятся критичными.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│           PINNING: Почему виртуальный поток блокирует носитель              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  VirtualThread ──► входит в synchronized { ... }                            │
│       │                                                                     │
│       ▼                                                                     │
│  Внутри блока вызывается Thread.sleep() / socket.read()                     │
│  JVM пытается сделать unmount (отцепить от Carrier Thread)                  │
│  │                                                                          │
│  ├─► БЛОКИРОВКА: Нельзя отцепить, пока удерживается объектный монитор       │
│  │    (synchronized) или выполняется JNI-вызов                              │
│  │                                                                          │
│  └─► Carrier Thread (носитель) остаётся PARKED                              │
│       Если все носители заблокированы → пул исчерпан → новые задачи ждут    │
│       Пул виртуальных потоков деградирует до уровня platform threads        │
└─────────────────────────────────────────────────────────────────────────────┘
```

```java
// Пул носителей: 4 потока (например на 4-ядерном CPU)
// Пришло 100 запросов, создано 100 виртуальных потоков

// ❌ СЦЕНАРИЙ С PINNING:
for (int i = 0; i < 100; i++) {
    Thread.startVirtualThread(() -> {
        synchronized(lock) {  // Поток ЗАКРЕПЛЁН за носителем (Carrier Thread)
            Thread.sleep(1000);  // Блокирующая операция
            // Носитель не может быть освобождён для других задач!
        }
    });
}

// Результат:
// - Первые 4 потока захватили все 4 носителя
// - Все 4 носителя спят внутри synchronized
// - Остальные 96 виртуальных потоков ждут носитель
// - → ДЕГРАДАЦИЯ до обычных platform threads!
```

### Проблемный код из легаси-приложений:

```java
public class DatabaseTransactionManager {
    private static final ThreadLocal<Connection> CONN = new ThreadLocal<>();
    
    // АНТИПАТТЕРН: Блокирующая операция внутри synchronized
    public void doInTransaction(Runnable action) {
        synchronized(this) {  // ← ПРИЧИНА PINNING
            Connection conn = dataSource.getConnection();
            CONN.set(conn);
            try {
                action.run();  // Может содержать блокирующие вызовы!
                conn.commit();
            } catch (Exception e) {
                conn.rollback();
            } finally {
                CONN.remove();
                conn.close();
            }
        }
    }
}

// А такой код часто встречается в старых приложениях:
public class LegacyService {
    
    private final Object lock = new Object();
    private final ThreadLocal<String> context = new ThreadLocal<>();
    
    public void process() {
        synchronized(lock) {  // synchronized блок
            context.set("user-123");
            
            // Блокирующий I/O внутри synchronized = PINNING!
            try (Socket socket = new Socket("db.com", 3306)) {
                socket.getInputStream().read();  // Блокируется!
            }
            
            context.remove();
        }
    }
}
```

## Как решать проблему?

### Решение 1: Заменить `synchronized` на `ReentrantLock`

> ReentrantLock работает на уровне Java-кода и не требует привязки к конкретному OS-потоку, в отличие от synchronized, 
> который использует встроенные механизмы JVM, привязанные к объектному монитору в заголовке объекта.

```java
public class ModernService {
    
    private final ReentrantLock lock = new ReentrantLock();
    private static final ThreadLocal<String> context = new ThreadLocal<>();
    
    public void process() {
        lock.lock();  // Не вызывает pinning!
        try {
            context.set("user-123");
            blockingIO();  // Теперь безопасно
        } finally {
            context.remove();
            lock.unlock();
        }
    }
}
```

### Решение 2: Уменьшить область синхронизации

```java
public class RefactoredService {
    
    private final Object lock = new Object();
    private static final ThreadLocal<String> context = new ThreadLocal<>();
    
    public void process() {
        // Подготовка данных вне синхронизации
        String data = prepareData();
        context.set(data);
        
        try {
            // ВАЖНО: блокирующие вызовы вне synchronized
            blockingIO();  // ← безопасно, нет pinning
            
            // Синхронизируем только критическую секцию
            synchronized(lock) {
                updateSharedState();
            }
        } finally {
            context.remove();
        }
    }
}
```

### Решение 3: Использовать структурную декомпозицию

```java
public class StructuredService {
    
    // Полностью избегаем ThreadLocal в блокирующих секциях
    public void process() {
        String ctx = prepareContext();
        
        // Выносим блокирующие операции наверх
        Result result = blockingIO(ctx);  // Без ThreadLocal внутри
        
        synchronized(lock) {
            // Минимальная синхронизация без блокирующих вызовов
            updateWithResult(result);
        }
    }
}
```

## Как обнаружить pinning?

```bash
# Запустите с диагностическими опциями:
java -Djdk.tracePinnedThreads=short -jar app.jar

# Вы увидите в логах:
# WARNING: Virtual thread <id> pinned to carrier thread <id> due to synchronized block at ...
```

- **При миграции на виртуальные потоки** замените `synchronized` на `ReentrantLock`
- **Выносите блокирующие I/O** из синхронизированных блоков
- **`ThreadLocal` безопасен сам по себе**, но требует осторожности в паре с `synchronized`
- **Используйте `try-finally`** для гарантированного вызова `remove()`
- **Тестируйте под нагрузкой** - pinning проявляется только при блокировках

---

## Альтернативы: `ScopedValue` и когда мигрировать

> Java 21 вводит `ScopedValue`. 
> 
> Это современная замена `ThreadLocal`, спроектированная специально для виртуальных потоков и функционального стиля. 
> 
> Вместо изменяемой привязки `set()/get()`, `ScopedValue` использует иммутабельное связывание 
> на время выполнения блока (`where(...).run(...)`).

| Критерий           | `ThreadLocal<T>`                        | `ScopedValue<T>`                                      |
|:-------------------|-----------------------------------------|-------------------------------------------------------|
| Мутабельность      | `set(T)` изменяет значение в потоке     | Только read-only внутри блока `run()`                 |
| Область видимости  | Пока поток жив или не вызван `remove()` | Только во время выполнения лямбды `run()`             |
| Наследование       | `InheritableThreadLocal` (копирование)  | Автоматическая передача в дочерние виртуальные потоки |
| Производительность | Высокий overhead для миллионов потоков  | Оптимизирован под `VirtualThread`, нулевой GC-даун    |
| Безопасность       | Требует ручного `finally { remove() }`  | Гарантируется JVM, автоматическая очистка             |

```java
public class ScopedValueMigrationDemo {
    
    // Декларируем иммутабельное значение
    private static final ScopedValue<String> USER_CTX = ScopedValue.newInstance();

    public void handleRequest(String userId) {
        // Значение привязывается ТОЛЬКО на время выполнения лямбды
        ScopedValue.where(USER_CTX, userId).run(() -> {
            // Все вложенные вызовы и виртуальные потоки видят актуальный контекст
            processOrder();
            generateReport();
        });
        // После выхода из run() привязка автоматически уничтожается.
        // Никакого remove(), никаких утечек, никаких stale entries.
    }

    private void processOrder() {
        // Чтение контекста
        String currentId = USER_CTX.get();
        System.out.println("Processing order for: " + currentId);
        
        // Запускаем дочерний виртуальный поток
        Thread.ofVirtual().start(() -> {
            // ScopedValue автоматически прокидывается в дочерний поток!
            System.out.println("Async worker sees: " + USER_CTX.get());
        });
    }
}
```

- **Когда мигрировать на `ScopedValue`:** Новые проекты на Java 21+, высоконагруженные сервисы с миллионами виртуальных потоков, 
  требования к строгой иммутабельности и гарантированной очистке.
- **Когда оставить `ThreadLocal`:** Поддержка легаси на Java 8-17, 
  фреймворки, глубоко завязанные на `ThreadLocal` (Spring Security, старые версии Hibernate), 
  или когда требуется именно изменяемое состояние в рамках задачи.
- **Путь миграции:** Начните с рефакторинга новых модулей.
  Для легаси используйте адаптеры или оставьте `ThreadLocal` до полного перехода на Java 21+.

---

## Ключевые выводы раздела
- `ThreadLocal` работает на виртуальных потоках без изменений, но создаёт значительный overhead для GC при масштабах >100k потоков.
- `synchronized` блоки внутри виртуальных потоков вызывают pinning носителя, что ломает модель масштабируемости. 
  Используйте `ReentrantLock` или выносите блокировки.
- `ScopedValue` — архитектурно безопасная замена: иммутабельная, автоматически очищаемая, 
  оптимизированная под `VirtualThread` и дочерние задачи.
- В production на Java 21+ предпочтительно переходить на `ScopedValue` для новых контекстов, 
  сохраняя `ThreadLocal` только для интеграции со сторонними библиотеками, не поддерживающими новую модель.

---