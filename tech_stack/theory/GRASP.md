# GRASP

**GRASP** (General Responsibility Assignment Software Patterns) - это **набор фундаментальных принципов (или паттернов)
назначения ответственностей** классам и объектам при проектировании объектно-ориентированных систем.

В отличие от GoF-паттернов, GRASP **не описывает конкретные шаблоны кода**, а задаёт **направляющие принципы** для
принятия решений на этапе проектирования:

- Кому передать ответственность за выполнение действия?
- Как обеспечить слабую связанность?
- Как сделать систему устойчивой к изменениям?

> GRASP был систематизирован Крейгом Ларманом (Craig Larman) и лежит в основе **responsibility-driven design** -
> подхода, в котором объекты рассматриваются как **исполнители ролей**, а не просто контейнеры данных.

9 паттернов, которые отвечают на вопрос: "Какой класс должен нести ответственность за определенную функцию?".

---

## 1. Information Expert (Информационный эксперт)

> **«Ответственность за выполнение задачи должна быть назначена тому классу, который обладает наибольшей информацией,
необходимой для её выполнения»**

### Этот принцип предлагает **назначать поведение (методы) тому объекту, у которого уже есть нужные данные

**. Цель — сохранить **инкапсуляцию**, избежать «справочных» классов (анемичных моделей) и снизить связанность.

Если для выполнения операции нужны данные `A`, `B` и `C`, то метод должен находиться в том классе, где эти данные **уже
есть** — или в классе, который их естественным образом агрегирует.

### ❌ Антипаттерн: Анемичная модель + «менеджер»

```java
// Данные без поведения
public class Order {

    private List<OrderItem> items;

    public List<OrderItem> getItems() {
        return items;
    }
}

// Весь бизнес в отдельном "менеджере"
public class OrderManager {

    public double calculateTotal(Order order) {
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        return total;
    }
}
```

Проблемы:

- Order - просто контейнер данных (анемичная модель).
- Логика расчёта размазана по внешнему классу.
- Нарушена инкапсуляция: OrderManager лезет внутрь Order.
- Если структура Order изменится - придётся править OrderManager.

### ✅ Решение: Поведение там, где данные

```java
public class Order {
    private List<OrderItem> items;

    // Метод живёт в том же классе, где и данные
    public double calculateTotal() {
        return items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
    }
}
```

Теперь:

- Order — полноценный объект с данными и поведением.
- Клиентский код просто вызывает order.calculateTotal().
- Изменение структуры Order не требует изменения внешнего кода.
- Инкапсуляция соблюдена: данные и логика работы с ними — вместе.

### Преимущества

- Высокая связанность внутри класса (cohesion)
- Низкая связанность между классами (coupling)
- Код становится более интуитивным: объекты «умеют» делать то, что от них логично ожидать.
- Упрощается поддержка и рефакторинг.

---

# 2. Creator (Создатель)

> **«Класс B следует назначить ответственность за создание экземпляра класса A, если выполняется хотя бы одно из
> следующих условий:
> - B содержит или агрегирует объекты A,
> - B тесно использует объекты A,
> - B имеет инициирующие данные для создания A,
> - B является частью A (например, фабрика),
> - B записывает экземпляры A в долговременное хранилище»**

### Суть

Принцип **Creator** помогает ответить на вопрос: **«Кому поручить создание нового объекта?»**  
Цель - избежать случайных или произвольных мест создания объектов, которые приводят к хаотичной архитектуре и высокой
связанности.

Идея: **объект, который «естественно связан» с создаваемым экземпляром, должен быть ответственен за его создание**.

### ❌ Антипаттерн: Создание «из ниоткуда»

```java
public class OrderProcessor {
    public void processOrderData(Map<String, Object> rawData) {
        // Создаём Order внутри сервиса, хотя он не связан с его данными напрямую
        Order order = new Order();
        order.setId((Long) rawData.get("id"));
        order.setCustomer((String) rawData.get("customer"));
        // ... ручная инициализация
    }
}
```

Проблемы:

- OrderProcessor не содержит и не агрегирует Order.
- Создание происходит в случайном месте.
- Высокая связанность: процессор знает о внутренней структуре Order.

### ✅ Решение: Создание там, где естественная связь

```java
public class Order {
    private Long id;
    private String customer;
    private List<OrderItem> items = new ArrayList<>();

    // Order создаёт свои OrderItem, т.к. содержит их
    public void addItem(String product, int quantity, double price) {
        items.add(new OrderItem(product, quantity, price));
    }
}

public class OrderService {
    // Сервис создаёт Order, т.к. управляет его жизненным циклом
    public Order createOrder(String customer) {
        return new Order(customer);
    }
}
```

Теперь:

- Order создаёт OrderItem - естественная связь (агрегация).
- OrderService создаёт Order - управляет его жизненным циклом.
- Каждый класс создаёт то, с чем тесно связан.

### Преимущества

- Снижается связанность между классами.
- Логика создания находится в предсказуемом месте.
- Упрощается поддержка и тестирование.

---

# 3. Controller (Контроллер)

> **«Ответственность за обработку системных событий должна быть назначена классу, который представляет либо всю систему,
либо сценарий использования (use case), в рамках которого происходит событие»**

### Суть

Принцип **Controller** отвечает на вопрос: **«Кто должен принимать запросы от UI или внешних систем?»**  
Цель - избежать прямой связи между UI и бизнес-логикой, создать чёткую точку входа для обработки запросов.

Контроллер - это **не UI-элемент**, а объект, который координирует выполнение операции, делегируя работу другим
объектам.

### ❌ Антипаттерн: UI напрямую вызывает бизнес-логику

```java
public class OrderButton {

    private OrderRepository repository;
    private EmailService emailService;

    public void onClick() {
        // UI-компонент содержит бизнес-логику
        Order order = new Order();
        order.setStatus("CREATED");
        repository.save(order);
        emailService.sendConfirmation(order);
    }
}
```

Проблемы:

- UI-компонент знает о деталях бизнес-логики.
- Невозможно переиспользовать логику из другого UI или API.
- Сложно тестировать без UI.
- Нарушение Single Responsibility.

### ✅ Решение: Выделенный контроллер

```java
// Контроллер координирует операцию
public class OrderController {
    private OrderService orderService;

    public void createOrder(OrderRequest request) {
        orderService.createOrder(request);
    }
}

// Сервис содержит бизнес-логику
public class OrderService {
    private OrderRepository repository;
    private EmailService emailService;

    public Order createOrder(OrderRequest request) {
        Order order = new Order(request.getCustomer());
        order.setStatus("CREATED");
        repository.save(order);
        emailService.sendConfirmation(order);
        return order;
    }
}

// UI только вызывает контроллер
public class OrderButton {
    private OrderController controller;

    public void onClick() {
        controller.createOrder(new OrderRequest(customerId));
    }
}
```

Теперь:

- UI отделён от бизнес-логики.
- Контроллер - единая точка входа для операции.
- Логику можно вызвать из разных мест (UI, API, тесты).
- Каждый класс имеет одну ответственность.

### Преимущества

- Слабая связанность между UI и бизнес-логикой.
- Переиспользуемость операций.
- Упрощённое тестирование.
- Чёткая архитектура: UI → Controller → Service → Repository.

---

# 4. Low Coupling (Низкая связанность)

> **«Ответственности должны распределяться так, чтобы минимизировать зависимости между классами»**

### Суть

Принцип **Low Coupling** отвечает на вопрос: **«Как сделать так, чтобы изменения в одном классе минимально влияли на
другие?»**  
Цель - создать систему, где классы слабо зависят друг от друга, что упрощает поддержку, тестирование и
переиспользование.

**Связанность (coupling)** - это степень зависимости одного класса от другого. Чем меньше класс знает о других классах,
тем лучше.

### ❌ Антипаттерн: Высокая связанность

```java
public class OrderService {

    public void processOrder(Long orderId) {

        // Прямая зависимость от конкретных реализаций
        MySQLOrderRepository repository = new MySQLOrderRepository();
        SmtpEmailService emailService = new SmtpEmailService();

        Order order = repository.findById(orderId);
        order.setStatus("PROCESSED");
        repository.save(order);
        emailService.sendEmail(order.getCustomerEmail(), "Order processed");
    }
}
```

Проблемы:

- `OrderService` жёстко привязан к конкретным реализациям (MySQL, SMTP).
- Невозможно заменить реализацию без изменения `OrderService`.
- Сложно тестировать - нужна реальная БД и почтовый сервер.
- Нарушение `Open/Closed Principle`.

### ✅ Решение: Зависимость от абстракций

```java
public class OrderService {

    private final OrderRepository repository;
    private final EmailService emailService;

    // Зависимости через интерфейсы
    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    public void processOrder(Long orderId) {
        Order order = repository.findById(orderId);
        order.setStatus("PROCESSED");
        repository.save(order);
        emailService.sendEmail(order.getCustomerEmail(), "Order processed");
    }
}
```

Теперь:

- `OrderService` зависит от интерфейсов, а не от конкретных реализаций.
- Легко заменить реализацию (`MySQL → PostgreSQL`, `SMTP → AWS SES`).
- Простое тестирование с моками.
- Соблюдение `Dependency Inversion Principle`.

### Преимущества

- Упрощённое тестирование через моки.
- Гибкость в выборе реализаций.
- Изменения в одном классе не затрагивают другие.
- Переиспользуемость компонентов.

---

# 5. High Cohesion (Высокая связность)

> **«Ответственности должны быть сосредоточены так, чтобы класс выполнял узкий, хорошо определённый набор связанных
задач»**

### Суть

Принцип **High Cohesion** отвечает на вопрос: **«Насколько сфокусирован класс на одной задаче?»**  
Цель - создать классы, где все методы и данные работают на одну цель, что делает код понятным, поддерживаемым и
переиспользуемым.

**Связность (cohesion)** - это степень того, насколько обязанности класса связаны между собой. Высокая связность
означает, что класс делает одно дело и делает его хорошо.

### ❌ Антипаттерн: Низкая связность (God Object)

```java
public class OrderManager {

    // Работа с БД
    public void saveOrder(Order order) { /* ... */ }

    public Order findOrder(Long id) { /* ... */ }

    // Бизнес-логика
    public double calculateTotal(Order order) { /* ... */ }

    public void applyDiscount(Order order, double discount) { /* ... */ }

    // Отправка email
    public void sendConfirmation(Order order) { /* ... */ }

    public void sendInvoice(Order order) { /* ... */ }

    // Генерация отчётов
    public String generateReport(Order order) { /* ... */ }

    public byte[] exportToPdf(Order order) { /* ... */ }
}
```

Проблемы:

- Класс делает слишком много несвязанных вещей.
- Сложно понять, за что отвечает класс.
- Изменение одной функции может сломать другие.
- Невозможно переиспользовать части функциональности.

### ✅ Решение: Разделение ответственностей

```java
// Только работа с БД
public class OrderRepository {

    public void save(Order order) { /* ... */ }

    public Order findById(Long id) { /* ... */ }
}

// Только бизнес-логика
public class OrderService {

    public double calculateTotal(Order order) { /* ... */ }

    public void applyDiscount(Order order, double discount) { /* ... */ }
}

// Только отправка email
public class EmailService {

    public void sendConfirmation(Order order) { /* ... */ }

    public void sendInvoice(Order order) { /* ... */ }
}

// Только генерация отчётов
public class ReportGenerator {

    public String generateReport(Order order) { /* ... */ }

    public byte[] exportToPdf(Order order) { /* ... */ }
}
```

Теперь:

- Каждый класс имеет одну чётко определённую ответственность.
- Легко понять назначение каждого класса.
- Изменения изолированы в соответствующих классах.
- Компоненты легко переиспользовать и тестировать.

### Преимущества

- Понятность: легко понять, что делает класс.
- Поддерживаемость: изменения локализованы.
- Переиспользуемость: компоненты можно использовать независимо.
- Тестируемость: проще писать и поддерживать тесты.

---

# 6. Polymorphism (Полиморфизм)

> **«Ответственность за обработку вариантов поведения должна быть назначена через полиморфные операции классам, для
которых это поведение изменяется»**

### Суть

Принцип **Polymorphism** отвечает на вопрос: **«Как обрабатывать альтернативы, основанные на типе?»**  
Цель - избежать условной логики (if/switch по типам) и использовать полиморфизм для делегирования поведения
соответствующим классам.

**Полиморфизм** позволяет разным классам реагировать на один и тот же вызов по-своему, что делает код расширяемым и
поддерживаемым.

### ❌ Антипаттерн: Условная логика по типам

```java
public class PaymentProcessor {

    public void processPayment(Payment payment) {
        // Проверка типа и разная логика для каждого
        if (payment.getType().equals("CREDIT_CARD")) {
            // Логика для кредитной карты
            validateCard(payment);
            chargeCard(payment);
        } else if (payment.getType().equals("PAYPAL")) {
            // Логика для PayPal
            redirectToPayPal(payment);
        } else if (payment.getType().equals("CRYPTO")) {
            // Логика для криптовалюты
            validateWallet(payment);
            transferCrypto(payment);
        }
    }
}
```

Проблемы:

- Добавление нового типа оплаты требует изменения `PaymentProcessor`.
- Нарушение `Open/Closed Principle`.
- Сложная условная логика, трудная для понимания.
- Невозможно тестировать типы оплаты независимо.

### ✅ Решение: Полиморфизм через интерфейс

```java
// Общий интерфейс
public interface PaymentMethod {
    void process(Payment payment);
}

// Каждый тип - отдельный класс
public class CreditCardPayment implements PaymentMethod {

    public void process(Payment payment) {
        validateCard(payment);
        chargeCard(payment);
    }
}

public class PayPalPayment implements PaymentMethod {

    public void process(Payment payment) {
        redirectToPayPal(payment);
    }
}

public class CryptoPayment implements PaymentMethod {

    public void process(Payment payment) {
        validateWallet(payment);
        transferCrypto(payment);
    }
}

// Процессор работает с абстракцией
public class PaymentProcessor {

    public void processPayment(PaymentMethod method, Payment payment) {
        method.process(payment);
    }
}
```

Теперь:

- Каждый тип оплаты инкапсулирован в своём классе.
- Добавление нового типа не требует изменения существующего кода.
- Каждый класс можно тестировать независимо.
- Код соответствует `Open/Closed Principle`.

### Преимущества

- Расширяемость: легко добавлять новые типы.
- Упрощение кода: нет сложной условной логики.
- Независимое тестирование каждого типа.
- Соблюдение `SOLID` принципов.

---

# 7. Pure Fabrication (Чистая выдумка)

> **«Ответственность должна быть назначена искусственно созданному классу, который не представляет концепцию предметной
области, если это необходимо для достижения низкой связанности и высокой связности»**

### Суть

Принцип **Pure Fabrication** отвечает на вопрос: **«Куда поместить логику, которая не подходит ни одному из существующих
доменных объектов?»**  
Цель - создать вспомогательный класс, который не является частью бизнес-модели, но необходим для технической реализации.

**Pure Fabrication** - это класс, который не отражает реальный объект предметной области, но существует для улучшения
архитектуры.

### ❌ Антипаттерн: Техническая логика в доменных объектах

```java
public class Order {

    private Long id;
    private String customer;
    private List<OrderItem> items;

    // Бизнес-логика - нормально
    public double calculateTotal() { /* ... */ }

    // Техническая логика - не на своём месте!
    public void saveToDatabase(Connection conn) throws SQLException {
        String sql = "INSERT INTO orders (id, customer) VALUES (?, ?)";
        PreparedStatement stmt = conn.prepareStatement(sql);
        stmt.setLong(1, id);
        stmt.setString(2, customer);
        stmt.executeUpdate();
    }
}
```

Проблемы:

- `Order` знает о SQL и работе с БД.
- Смешивание бизнес-логики и технических деталей.
- Невозможно заменить способ хранения без изменения `Order`.
- Сложно тестировать бизнес-логику без БД.

### ✅ Решение: Выделение технического класса

```java
// Доменный объект - только бизнес-логика
public class Order {

    private Long id;
    private String customer;
    private List<OrderItem> items;

    public double calculateTotal() { /* ... */ }
}

// Pure Fabrication - технический класс
public class OrderRepository {

    private DataSource dataSource;

    public void save(Order order) {
        // Вся техническая логика здесь
        String sql = "INSERT INTO orders (id, customer) VALUES (?, ?)";
        // ...
    }

    public Order findById(Long id) { /* ... */ }
}
```

Теперь:

- `Order` содержит только бизнес-логику.
- `OrderRepository` - искусственный класс для технических задач.
- Легко заменить способ хранения.
- Бизнес-логику можно тестировать независимо.

### Преимущества

- Разделение бизнес-логики и технических деталей.
- Низкая связанность (coupling) между слоями - Order не зависит от деталей БД
- Высокая связность (cohesion) внутри классов - OrderRepository занимается только работой с БД
- Переиспользуемость технических компонентов.
- Упрощённое тестирование.

---

# 8. Indirection (Перенаправление)

> **«Ответственность за посредничество между компонентами должна быть назначена промежуточному объекту, чтобы избежать
прямой связи между ними»**

### Суть

Принцип **Indirection** отвечает на вопрос: **«Как избежать прямой связи между двумя компонентами?»**  
Цель - ввести посредника, который разделяет компоненты и снижает их зависимость друг от друга.

**Indirection** - это добавление промежуточного слоя для уменьшения связанности и повышения гибкости системы.

### ❌ Антипаттерн: Прямая связь между компонентами

```java
public class OrderService {

    public void processOrder(Order order) {
        // Прямое обращение к внешнему API
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://payment-api.com/charge"))
                .POST(HttpRequest.BodyPublishers.ofString(order.toJson()))
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() == 200) {
            order.setStatus("PAID");
        }
    }
}
```

Проблемы:

- `OrderService` жёстко привязан к конкретному `API` и `HTTP-клиенту`.
- Невозможно заменить способ оплаты без изменения `OrderService`.
- Сложно тестировать - нужен реальный `API`.
- Смешивание бизнес-логики и технических деталей.

### ✅ Решение: Введение посредника

```java
// Интерфейс-посредник
public interface PaymentGateway {

    PaymentResult charge(Order order);
}

// Реализация для конкретного API
public class ExternalPaymentGateway implements PaymentGateway {

    private HttpClient client;

    @Override
    public PaymentResult charge(Order order) {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://payment-api.com/charge"))
                .POST(HttpRequest.BodyPublishers.ofString(order.toJson()))
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        return new PaymentResult(response.statusCode() == 200);
    }
}

// Сервис работает через посредника
public class OrderService {

    private PaymentGateway paymentGateway;

    public void processOrder(Order order) {
        PaymentResult result = paymentGateway.charge(order);

        if (result.isSuccess()) {
            order.setStatus("PAID");
        }
    }
}
```

Теперь:

- `OrderService` не знает о деталях `HTTP` и внешнего `API`.
- Легко заменить реализацию платёжного шлюза.
- Простое тестирование с моками.
- Бизнес-логика отделена от технических деталей.

### Преимущества

- Низкая связанность между компонентами.
- Гибкость в замене реализаций.
- Упрощённое тестирование.
- Соблюдение принципа единственной ответственности.

---

# 9. Protected Variations (Защищённые вариации)

> **«Ответственность за защиту от изменений должна быть назначена через создание стабильных интерфейсов вокруг точек
предсказуемой изменчивости»**

### Суть

Принцип **Protected Variations** отвечает на вопрос: **«Как защитить систему от изменений в одной части, чтобы они не
затронули другие?»**  
Цель - идентифицировать точки вероятных изменений и создать стабильные интерфейсы, которые изолируют эти изменения.

**Protected Variations** - это принцип проектирования систем, устойчивых к изменениям.

### ❌ Антипаттерн: Прямая зависимость от изменчивых деталей

```java
public class OrderProcessor {

    public void processOrder(Order order) {
        // Прямая зависимость от конкретного формата
        String xml = "<order><id>" + order.getId() + "</id></order>";
        sendToExternalSystem(xml);

        // Прямая зависимость от конкретной структуры БД
        double tax = order.getTotal() * 0.20; // Фиксированная ставка
        order.setTax(tax);
    }
}
```

Проблемы:

- Изменение формата (`XML` → `JSON`) требует изменения `OrderProcessor`.
- Изменение логики налогообложения требует изменения кода.
- Система не защищена от изменений.
- Нарушение `Open/Closed Principle`.

### ✅ Решение: Стабильные интерфейсы

```java
// Интерфейс для сериализации
public interface OrderSerializer {
    
    String serialize(Order order);
}

// Интерфейс для расчёта налогов
public interface TaxCalculator {
    
    double calculate(Order order);
}

// Реализации можно менять
public class XmlOrderSerializer implements OrderSerializer {
    
    public String serialize(Order order) {
        return "<order><id>" + order.getId() + "</id></order>";
    }
}

public class StandardTaxCalculator implements TaxCalculator {
    
    public double calculate(Order order) {
        return order.getTotal() * 0.20;
    }
}

// Процессор защищён от изменений
public class OrderProcessor {

    private OrderSerializer serializer;
    private TaxCalculator taxCalculator;

    public void processOrder(Order order) {
        String data = serializer.serialize(order);
        sendToExternalSystem(data);

        double tax = taxCalculator.calculate(order);
        order.setTax(tax);
    }
}
```

Теперь:

- Изменение формата - просто другая реализация `OrderSerializer`.
- Изменение логики налогов - просто другая реализация `TaxCalculator`.
- `OrderProcessor` не требует изменений.
- Система защищена от изменений через стабильные интерфейсы.

### Преимущества

- Устойчивость к изменениям.
- Легкое добавление новой функциональности.
- Соблюдение `Open/Closed Principle`.
- Изоляция изменений в отдельных классах.