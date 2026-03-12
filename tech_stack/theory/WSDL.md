# WSDL (Web Services Description Language)

## Основные понятия

> **WSDL (Web Services Description Language)** — это XML-документ, описывающий **публичный контракт** SOAP-сервиса:
> какие операции он предоставляет, как выглядят входные и выходные сообщения, по какому адресу доступен и через какой 
> протокол взаимодействует.
>
> **WSDL** не определяет логику работы сервиса — только его внешний интерфейс.

### Роль в архитектуре SOAP

WSDL является центральным элементом в **контракт-ориентированной разработке** (contract-first). Он связывает три
компонента:

- **SOAP** — формат сообщений (Envelope, Header, Body),
- **XSD** — описание структуры данных внутри `<Body>`,
- **Транспорт** (обычно HTTP) — способ доставки сообщений.

Без WSDL SOAP-сообщение остаётся просто XML-документом без контекста.

С WSDL клиент может:

- автоматически сгенерировать типобезопасный код,
- точно знать, какие параметры передавать,
- понимать, где находится endpoint.

### Версии WSDL

- **WSDL 1.1** - Широко используется, де-факто стандарт. Полная поддержка в `JAX-WS`, `Spring-WS`, `Apache CXF`
- **WSDL 2.0** - `W3C Recommendation` (2007), более модульный. Ограниченная поддержка; большинство enterprise-библиотек
  фокусируются на 1.1

> 💡 На практике почти все production-системы используют **WSDL 1.1**, особенно в банковской, государственной и
> legacy-средах.
>
> WSDL 2.0 редко встречается в реальных проектах.

### Триада: WSDL + XSD → SOAP

- **XSD** определяет *что*: структуру бизнес-объектов (`GetUserRequest`, `CreateOrderResponse`).
- **WSDL** определяет *как*: какие операции доступны, как они связаны с сообщениями, где сервис.
- **SOAP** определяет *формат передачи*: обёртка вокруг XSD-данных.

Эта связка обеспечивает:

- **Строгую типизацию** на этапе компиляции (благодаря генерации кода),
- **Независимость от языка реализации** (Java, .NET, Python — все могут использовать один WSDL),
- **Предсказуемость интеграций** — клиент и сервер «говорят на одном языке».

> Важно: хотя технически можно отправлять SOAP-сообщения без WSDL, в enterprise-практике это считается нарушением
> контракта и источником ошибок. WSDL — не опционален, а обязательная часть спецификации сервиса.

---

## Структура документа WSDL 1.1

WSDL 1.1 — это XML-документ с фиксированной иерархией элементов. Он описывает сервис на **абстрактном** (что делает) и *
*конкретном** (как вызывается) уровнях.

### Основные секции

- `<types>`  Встраивает или импортирует XSD-схемы, описывающие структуру данных.
- `<message>`  Определяет абстрактные сообщения: имена и типы частей (`part`).
- `<portType>`  Описывает интерфейс сервиса: операции и их вход/выход (аналог Java-интерфейса).
- `<binding>`  Привязывает `portType` к конкретному протоколу (обычно SOAP) и транспорту (HTTP).
- `<service>`  Указывает физический URL endpoint-а.

Все эти элементы объединены в корневом `<definitions>`, который также задаёт пространства имён.

### Пример полного WSDL с аннотациями

```xml
<!-- 
  Этот WSDL описывает простой UserService с одной операцией GetUser.
  Используется стиль document/literal, что является enterprise-стандартом.
-->
<definitions
        xmlns="http://schemas.xmlsoap.org/wsdl/"
        xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
        xmlns:tns="http://example.com/user"
        xmlns:xs="http://www.w3.org/2001/XMLSchema"
        targetNamespace="http://example.com/user">

    <!-- 
      TYPES: Здесь описываются все бизнес-типы через XSD.
      Лучше выносить в отдельный .xsd файл, но для примера встроим.
    -->
    <types>
        <xs:schema targetNamespace="http://example.com/user"
                   xmlns:xs="http://www.w3.org/2001/XMLSchema"
                   elementFormDefault="qualified">

            <!-- 
              ВАЖНО: все элементы, используемые в <message>, должны быть ГЛОБАЛЬНЫМИ 
              (объявлены на уровне <schema>, а не внутри complexType).
              Это требование JAX-WS для корректной генерации кода.
            -->
            <xs:element name="GetUserRequest">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="userId" type="xs:string"/>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>

            <xs:element name="GetUserResponse">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="user" type="tns:User"/>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>

            <xs:element name="UserNotFoundFault">
                <xs:complexType>
                    <xs:sequence>
                        <xs:element name="errorCode" type="xs:string"/>
                        <xs:element name="message" type="xs:string"/>
                    </xs:sequence>
                </xs:complexType>
            </xs:element>

            <!-- Тип User — используется внутри GetUserResponse -->
            <xs:complexType name="User">
                <xs:sequence>
                    <xs:element name="id" type="xs:string"/>
                    <xs:element name="name" type="xs:string"/>
                    <xs:element name="email" type="xs:string"/>
                </xs:sequence>
            </xs:complexType>

        </xs:schema>
    </types>

    <!-- 
      MESSAGE: Абстрактные описания сообщений.
      Каждое сообщение ссылается на глобальный элемент из XSD.
    -->
    <message name="GetUserInputMessage">
        <!-- part name="parameters" — историческое соглашение; может быть любым -->
        <part name="parameters" element="tns:GetUserRequest"/>
    </message>

    <message name="GetUserOutputMessage">
        <part name="parameters" element="tns:GetUserResponse"/>
    </message>

    <message name="UserNotFoundFaultMessage">
        <part name="fault" element="tns:UserNotFoundFault"/>
    </message>

    <!-- 
      PORTTYPE: Интерфейс сервиса. Аналог Java interface.
      Определяет, какие операции есть и как они связаны с сообщениями.
    -->
    <portType name="UserServicePortType">
        <operation name="GetUser">
            <!-- Входное сообщение -->
            <input message="tns:GetUserInputMessage"/>
            <!-- Успешный ответ -->
            <output message="tns:GetUserOutputMessage"/>
            <!-- Возможная ошибка (fault) -->
            <fault name="UserNotFound" message="tns:UserNotFoundFaultMessage"/>
        </operation>
    </portType>

    <!-- 
      BINDING: Привязка к SOAP.
      Указывает, что используется SOAP поверх HTTP и стиль document/literal.
    -->
    <binding name="UserServiceSoapBinding" type="tns:UserServicePortType">
        <!-- Указываем, что это SOAP-биндинг -->
        <soap:binding style="document"
                      transport="http://schemas.xmlsoap.org/soap/http"/>

        <operation name="GetUser">
            <!-- soapAction устарел в document/literal, но часто указывается для совместимости -->
            <soap:operation soapAction="http://example.com/user/GetUser"/>
            <input>
                <!-- use="literal" — данные соответствуют XSD напрямую (не закодированы) -->
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
            <fault name="UserNotFound">
                <soap:fault name="UserNotFound" use="literal"/>
            </fault>
        </operation>
    </binding>

    <!-- 
      SERVICE: Физическая точка доступа.
      Может содержать несколько портов (например, для разных протоколов), но обычно один.
    -->
    <service name="UserService">
        <port name="UserServicePort" binding="tns:UserServiceSoapBinding">
            <!-- Адрес, по которому клиент будет отправлять POST-запросы -->
            <soap:address location="https://api.example.com/v1/UserService"/>
        </port>
    </service>

</definitions>
```

### 7. Детализация основных и дополнительных секций WSDL

* **`<definitions>`** (Корневой элемент)
* **`<documentation>`** (необязательный): Человекочитаемое описание сервиса. Может быть в любом месте.
* **`<types>`** (0 или 1): Контейнер для определения типов данных.
  *   **`<xs:schema>`** (один или несколько): Сами XSD-схемы. Могут быть встроенными или импортированными из внешних
  файлов через `xs:import`.
* **`<message>`** (0 и более): Абстрактное определение передаваемых данных.
  *   **`<part>`** (0 и более): Часть сообщения. Атрибуты:
    * `name`: Имя части.
    * `element`: Ссылка на глобальный XSD-элемент (для document/literal стиля).
    * `type`: Ссылка на XSD-тип (для rpc стиля).
* **`<portType>`** (0 и более): Абстрактный набор операций.
  *   **`<operation>`** (0 и более): Абстрактное определение операции.
    * **`<input>`** (0 или 1): Сообщение, которое сервис получает. Атрибут `message`.
    * **`<output>`** (0 или 1): Сообщение, которое сервис отправляет в ответ. Атрибут `message`.
    * **`<fault>`** (0 и более): Сообщение об ошибке. Атрибуты `name` и `message`.
* **`<binding>`** (0 и более): Конкретный протокол и формат данных для операций из `portType`.
  *   **`<soap:binding>`** (для SOAP): Указывает `style` и `transport`.
  *   **`<operation>`** (0 и более): Привязка конкретной операции.
    * **`<soap:operation>`**: Может содержать `soapAction`.
    * **`<input>` / `<output>` / `<fault>`**: Описывают, как абстрактные части сообщения преобразуются в конкретный
      протокол.
    * **`<soap:body>`**: Для `input`/`output`. Атрибуты: `use` (`literal` или `encoded`), `namespace` (пространство имен
      тела), `parts` (какие части сообщения сюда включить).
    * **`<soap:fault>`**: Аналогично `<soap:body>`, но для ошибок.
    * **`<soap:header>`**: Описание SOAP-заголовков.
* **`<service>`** (0 и более): Набор портов, предоставляющих доступ к сервису.
  *   **`<port>`** (0 и более): Одна конечная точка.
    * **`<soap:address>`**: Указывает физический URL (`location`) сервиса.

---