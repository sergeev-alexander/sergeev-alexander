# `I/O`/`NIO`

## Введение в Java I/O

Java I/O (Input/Output) - это набор API для работы с вводом-выводом данных.<br>
Библиотека I/O предоставляет мощные и гибкие механизмы для чтения и записи данных из различных источников: файлов, сетевых соединений, потоков памяти и других.

<details closed> 
    <summary>
        <b>Основные концепции I/O</b>
    </summary>

Потоки (Streams) - абстракция для последовательной передачи данных:

- Байтовые потоки: работают с raw data (`InputStream` / `OutputStream`)
- Символьные потоки: работают с текстовыми данными (`Reader` / `Writer`)
- Буферизация: техника для уменьшения количества системных вызовов
- Блокирующие операции: поток **блокируется** до завершения I/O операции
- Исключения: `IOException` и его производные для обработки ошибок

Ключевые преимущества:

- Единый подход к различным источникам данных
- Автоматическое управление ресурсами (try-with-resources)
- Поддержка различных кодировок символов
- Расширяемость через декораторы

</details>

## Java I/O
<details closed> 
    <summary>
        <b>Байтовые потоки (Byte Streams)</b>
    </summary>

Основные классы для работы с бинарными данными:

- `InputStream` / `OutputStream` - абстрактные базовые классы
- `FileInputStream` / `FileOutputStream` - работа с файлами
- `BufferedInputStream` / `BufferedOutputStream` - буферизация
- `DataInputStream` / `DataOutputStream` - примитивные типы
- `ObjectInputStream` / `ObjectOutputStream` - сериализация объектов

```java
// Чтение файла побайтово
try (FileInputStream fis = new FileInputStream("file.bin")) {
    int byteData;
    while ((byteData = fis.read()) != -1) {
        // обработка байта
    }
}

// Буферизованное чтение
try (BufferedInputStream bis = new BufferedInputStream(
     new FileInputStream("file.bin"))) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = bis.read(buffer)) != -1) {
        // обработка данных из buffer
    }
}

// Запись в файл
try (FileOutputStream fos = new FileOutputStream("output.bin")) {
    byte[] data = getData();
    fos.write(data);
}
```
</details>

<details closed> 
    <summary>
        <b>Символьные потоки (Character Streams)</b>
    </summary>

Классы для работы с текстовыми данными с поддержкой кодировок:

- `Reader` / `Writer` - абстрактные базовые классы
- `FileReader` / `FileWriter` - работа с файлами (используют кодировку по умолчанию)
- `InputStreamReader` / `OutputStreamWriter` - мост между байтовыми и символьными потоками
- `BufferedReader` / `BufferedWriter` - буферизация
- `PrintWriter` - форматированный вывод

```java
// Чтение текстового файла
try (FileReader reader = new FileReader("file.txt")) {
    int charData;
    while ((charData = reader.read()) != -1) {
        char character = (char) charData;
        // обработка символа
    }
}

// Буферизованное чтение с указанием кодировки
try (BufferedReader br = new BufferedReader(
     new InputStreamReader(
         new FileInputStream("file.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = br.readLine()) != null) {
        // обработка строки
    }
}

// Запись текста
try (FileWriter writer = new FileWriter("output.txt")) {
    writer.write("Hello, World!\n");
    writer.write("Вторая строка");
}

// С буферизацией и кодировкой
try (BufferedWriter bw = new BufferedWriter(
     new OutputStreamWriter(
         new FileOutputStream("output.txt"), StandardCharsets.UTF_8))) {
    bw.write("Текст в UTF-8");
    bw.newLine();
}
```

</details>

<details closed> 
    <summary>
        <b>Декораторы (Wrapper Streams)</b>
    </summary>

Популярные декораторы:

- `BufferedInputStream` / `BufferedOutputStream` - буферизация
- `DataInputStream` / `DataOutputStream` - примитивные типы
- `PushbackInputStream` - возврат прочитанных данных
- `SequenceInputStream` - объединение потоков
- `ObjectInputStream` / `ObjectOutputStream` - сериализация
- `GZIPInputStream` / `GZIPOutputStream` - сжатие
- `CipherInputStream` / `CipherOutputStream` - шифрование

```java
// Композиция декораторов
try (DataInputStream dis = new DataInputStream(
     new BufferedInputStream(
         new FileInputStream("data.bin")))) {
    
    int intValue = dis.readInt();
    double doubleValue = dis.readDouble();
    String stringValue = dis.readUTF();
}

// Сжатие данных
try (GZIPOutputStream gzos = new GZIPOutputStream(
     new BufferedOutputStream(
         new FileOutputStream("file.gz")))) {
    gzos.write(data);
}

// Шифрование
try (CipherOutputStream cos = new CipherOutputStream(
     new FileOutputStream("encrypted.bin"), cipher)) {
    cos.write(sensitiveData);
}
```
</details>

## Java NIO

<details closed> 
    <summary>
        <b>Каналы и буферы (Channels & Buffers)</b>
    </summary>

NIO использует каналы и буферы вместо потоков.

Основные компоненты:

- `Buffer` - контейнер для данных
- `Channel` - канал для передачи данных
- `Selector` - мультиплексирование каналов
- `Charset` - работа с кодировками

Преимущества NIO:

- Неблокирующий I/O
- Мультиплексирование
- Прямые буферы (zero-copy)
- Более гибкое управление

```java
// Чтение файла через FileChannel
try (FileChannel channel = FileChannel.open(Paths.get("file.txt"))) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    
    while (channel.read(buffer) > 0) {
        buffer.flip(); // переключение в режим чтения
        while (buffer.hasRemaining()) {
            byte b = buffer.get();
            // обработка байта
        }
        buffer.clear(); // очистка для следующего чтения
    }
}

// Запись через FileChannel
try (FileChannel channel = FileChannel.open(
     Paths.get("output.txt"), 
     StandardOpenOption.CREATE, 
     StandardOpenOption.WRITE)) {
    
    ByteBuffer buffer = ByteBuffer.wrap("Hello NIO".getBytes());
    channel.write(buffer);
}

// Прямые буферы для высокопроизводительных операций
ByteBuffer directBuffer = ByteBuffer.allocateDirect(4096);
```
</details>

<details closed> 
    <summary>
        <b>Неблокирующий I/O и селекторы</b>
    </summary>

Мультиплексирование каналов через селекторы.

Ключевые операции:

- `OP_ACCEPT` - принятие соединений
- `OP_CONNECT` - установка соединений
- `OP_READ` - чтение данных
- `OP_WRITE` - запись данных

Преимущества:

- Один поток на множество соединений
- Масштабируемость
- Эффективное использование ресурсов

```java
// Создание селектора
Selector selector = Selector.open();

// Открытие серверного сокета
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));

// Регистрация в селекторе
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

// Цикл обработки событий
while (true) {
    int readyChannels = selector.select();
    if (readyChannels == 0) continue;
    
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
    
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        
        if (key.isAcceptable()) {
            // Принятие нового соединения
            acceptConnection(key);
        } else if (key.isReadable()) {
            // Чтение данных
            readData(key);
        } else if (key.isWritable()) {
            // Запись данных
            writeData(key);
        }
        
        keyIterator.remove();
    }
}
```
</details>

<details closed> 
    <summary>
        <b>Файловые операции NIO.2</b>
    </summary>

Современный API для работы с файловой системой.

Основные классы NIO.2:

- `Path` - представление пути
- `Paths` - фабрика для создания Path
- `Files` - утилиты для файловых операций
- `FileSystem` / `FileSystems` - работа с файловыми системами
- `WatchService` - мониторинг изменений

```java
// Чтение файла через FileChannel
try (FileChannel channel = FileChannel.open(Paths.get("file.txt"))) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    
    while (channel.read(buffer) > 0) {
        buffer.flip(); // переключение в режим чтения
        while (buffer.hasRemaining()) {
            byte b = buffer.get();
            // обработка байта
        }
        buffer.clear(); // очистка для следующего чтения
    }
}

// Запись через FileChannel
try (FileChannel channel = FileChannel.open(
     Paths.get("output.txt"), 
     StandardOpenOption.CREATE, 
     StandardOpenOption.WRITE)) {
    
    ByteBuffer buffer = ByteBuffer.wrap("Hello NIO".getBytes());
    channel.write(buffer);
}

// Прямые буферы для высокопроизводительных операций
ByteBuffer directBuffer = ByteBuffer.allocateDirect(4096);
```
</details>

## IO vs NIO

### Классический I/O (java.io):

- Плюсы: Простота использования, понятная модель
- Минусы: Блокирующие операции, ограниченная масштабируемость
- Использование: Простые приложения, последовательная обработка

### Новый I/O (java.nio):

- Плюсы: Неблокирующие операции, мультиплексирование, высокая производительность
- Минусы: Сложность, больше boilerplate кода
- Использование: Высоконагруженные сетевые приложения, серверы

### NIO.2 (java.nio.file):

- Плюсы: Современный API, асинхронные операции, мониторинг
- Минусы: Требует Java 7+
- Использование: Файловые операции, современные приложения

### Рекомендации по выбору:

- Для простых задач - java.io
- Для сетевых серверов - java.nio с селекторами
- Для файловых операций - java.nio.file
- Для максимальной производительности - асинхронные каналы

## Асинхронный I/O

**Асинхронный I/O** - это модель выполнения операций ввода-вывода, при которой поток не блокируется во время выполнения I/O операций.<br>
Вместо ожидания завершения операции, поток может продолжить выполнение других задач, а результат I/O операции будет обработан через callback или Future.

<details closed>
    <summary>
        <b>Основные концепции асинхронного I/O</b>
    </summary>

Ключевые преимущества асинхронного подхода:

- Масштабируемость: Один поток может обрабатывать тысячи соединений
- Эффективность: Отсутствие блокировок и ожиданий
- Реактивность: Быстрое реагирование на события
- Ресурсная экономия: Меньшее количество потоков для того же количества соединений

Сравнение моделей:

- Синхронная: Один поток на соединение - просто, но не масштабируемо
- NIO с селекторами: Один поток на множество соединений - сложно, но эффективно
- Асинхронная: Неблокирующие операции с callback'ами - современно и производительно

Основные пакеты:

- `java.nio.channels.AsynchronousChannel` - асинхронные каналы
- `java.util.concurrent.CompletableFuture` - асинхронные вычисления
- `java.nio.channels.CompletionHandler` - обработчики завершения
</details>

### AsynchronousFileChannel

Асинхронная работа с файлами

<details closed> 
    <summary>
        <b>Чтение файлов через CompletionHandler</b>
    </summary>

```java
Path path = Paths.get("largefile.dat");
AsynchronousFileChannel fileChannel = 
        AsynchronousFileChannel.open(path, StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);
long position = 0;

fileChannel.read(buffer, position, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    
    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        System.out.println("Read " + result + " bytes");
        attachment.flip();
        
        byte[] data = new byte[attachment.limit()];
        attachment.get(data);
        System.out.println("Data: " + new String(data));
        
        attachment.clear();
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        System.err.println("Read failed: " + exc.getMessage());
    }
});

// Поток продолжает выполнение, не дожидаясь завершения чтения
System.out.println("Continue with other work...");
```
</details>

<details closed>
    <summary>
        <b>Запись файлов через Future</b>
    </summary>

```java
Path outputPath = Paths.get("output.dat");
AsynchronousFileChannel writeChannel = 
        AsynchronousFileChannel.open(outputPath, StandardOpenOption.CREATE, StandardOpenOption.WRITE);

ByteBuffer writeBuffer = ByteBuffer.wrap("Hello Async World".getBytes());
Future<Integer> writeOperation = writeChannel.write(writeBuffer, 0);

// Можно выполнять другую работу
doOtherWork();

// Когда нужно получить результат
try {
    Integer bytesWritten = writeOperation.get(5, TimeUnit.SECONDS);
    System.out.println("Written " + bytesWritten + " bytes");
} catch (TimeoutException e) {
    System.err.println("Write operation timed out");
}
```
</details>

<details closed>
    <summary>
        <b>Чтение большого файла частями</b>
    </summary>

```java
public void readLargeFileAsync(Path filePath) throws IOException {
    AsynchronousFileChannel channel = AsynchronousFileChannel.open(
        filePath, StandardOpenOption.READ);
    
    ByteBuffer buffer = ByteBuffer.allocateDirect(8192);
    readChunk(channel, buffer, 0);
}

private void readChunk(AsynchronousFileChannel channel, ByteBuffer buffer, long position) {
    channel.read(buffer, position, null, new CompletionHandler<Integer, Void>() {
        @Override
        public void completed(Integer bytesRead, Void attachment) {
            if (bytesRead > 0) {
                buffer.flip();
                processBuffer(buffer);
                buffer.clear();
                
                // Читаем следующую часть
                readChunk(channel, buffer, position + bytesRead);
            } else {
                // Конец файла
                closeChannel(channel);
            }
        }

        @Override
        public void failed(Throwable exc, Void attachment) {
            System.err.println("Read failed at position " + position + ": " + exc.getMessage());
            closeChannel(channel);
        }
    });
}
```
</details>

### AsynchronousSocketChannel

Асинхронные сетевые операции

<details closed>
    <summary>
        <b>Асинхронный TCP-сервер</b>
    </summary>

```java
public class AsyncTcpServer {
    
    private final AsynchronousChannelGroup channelGroup;
    
    public AsyncTcpServer() throws IOException {
        this.channelGroup = AsynchronousChannelGroup.withFixedThreadPool(
            Runtime.getRuntime().availableProcessors(), 
            Executors.defaultThreadFactory()
        );
    }
    
    public void start(int port) throws IOException {
        AsynchronousServerSocketChannel serverChannel = AsynchronousServerSocketChannel
                .open(channelGroup)
                .bind(new InetSocketAddress(port));
        
        System.out.println("Server started on port " + port);
        
        // Начинаем принимать соединения
        acceptConnections(serverChannel);
    }
    
    private void acceptConnections(AsynchronousServerSocketChannel serverChannel) {
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            
            @Override
            public void completed(AsynchronousSocketChannel clientChannel, Void attachment) {
                // Принято новое соединение
                System.out.println("New client connected: " + clientChannel);
                
                // Обрабатываем клиента
                handleClient(clientChannel);
                
                // Продолжаем принимать новые соединения
                acceptConnections(serverChannel);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Failed to accept connection: " + exc.getMessage());
                // Продолжаем принимать соединения даже при ошибке
                acceptConnections(serverChannel);
            }
        });
    }
    
    private void handleClient(AsynchronousSocketChannel clientChannel) {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        
        // Читаем данные от клиента
        readFromClient(clientChannel, buffer);
    }
    
    private void readFromClient(AsynchronousSocketChannel clientChannel, ByteBuffer buffer) {
        clientChannel.read(buffer, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer bytesRead, Void attachment) {
                if (bytesRead == -1) {
                    // Клиент отключился
                    closeClient(clientChannel);
                    return;
                }
                
                if (bytesRead > 0) {
                    buffer.flip();
                    processClientData(clientChannel, buffer);
                    buffer.clear();
                }
                
                // Продолжаем чтение
                readFromClient(clientChannel, buffer);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Read from client failed: " + exc.getMessage());
                closeClient(clientChannel);
            }
        });
    }
    
    private void processClientData(AsynchronousSocketChannel clientChannel, ByteBuffer buffer) {
        // Обрабатываем полученные данные
        byte[] data = new byte[buffer.remaining()];
        buffer.get(data);
        String message = new String(data, StandardCharsets.UTF_8);
        
        System.out.println("Received: " + message);
        
        // Отправляем ответ
        String response = "Echo: " + message;
        ByteBuffer responseBuffer = ByteBuffer.wrap(response.getBytes(StandardCharsets.UTF_8));
        
        writeToClient(clientChannel, responseBuffer);
    }
    
    private void writeToClient(AsynchronousSocketChannel clientChannel, ByteBuffer buffer) {
        clientChannel.write(buffer, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer bytesWritten, Void attachment) {
                if (buffer.hasRemaining()) {
                    // Если не все данные записаны, продолжаем
                    writeToClient(clientChannel, buffer);
                } else {
                    System.out.println("Response sent successfully");
                }
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Write to client failed: " + exc.getMessage());
                closeClient(clientChannel);
            }
        });
    }
    
    private void closeClient(AsynchronousSocketChannel clientChannel) {
        try {
            clientChannel.close();
            System.out.println("Client disconnected");
        } catch (IOException e) {
            System.err.println("Error closing client channel: " + e.getMessage());
        }
    }
}
```
</details>

<details closed>
    <summary>
        <b>Асинхронный TCP-клиент</b>
    </summary>

```java
public class AsyncTcpClient {
    
    public void connect(String host, int port) throws IOException {
        AsynchronousSocketChannel clientChannel = AsynchronousSocketChannel.open();
        
        InetSocketAddress serverAddress = new InetSocketAddress(host, port);
        
        clientChannel.connect(serverAddress, null, new CompletionHandler<Void, Void>() {
            
            @Override
            public void completed(Void result, Void attachment) {
                System.out.println("Connected to server");
                startCommunication(clientChannel);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Connection failed: " + exc.getMessage());
            }
        });
    }
    
    private void startCommunication(AsynchronousSocketChannel clientChannel) {
        // Отправляем сообщение
        String message = "Hello from async client!";
        ByteBuffer writeBuffer = ByteBuffer.wrap(message.getBytes(StandardCharsets.UTF_8));
        
        clientChannel.write(writeBuffer, null, new CompletionHandler<Integer, Void>() {
            @Override
            public void completed(Integer bytesWritten, Void attachment) {
                System.out.println("Message sent: " + message);
                
                // Читаем ответ
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                readResponse(clientChannel, readBuffer);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Write failed: " + exc.getMessage());
            }
        });
    }
    
    private void readResponse(AsynchronousSocketChannel clientChannel, ByteBuffer buffer) {
        clientChannel.read(buffer, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer bytesRead, Void attachment) {
                if (bytesRead > 0) {
                    buffer.flip();
                    byte[] data = new byte[buffer.remaining()];
                    buffer.get(data);
                    String response = new String(data, StandardCharsets.UTF_8);
                    System.out.println("Server response: " + response);
                }
                
                // Закрываем соединение после получения ответа
                closeConnection(clientChannel);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Read failed: " + exc.getMessage());
                closeConnection(clientChannel);
            }
        });
    }
}
```
</details>

### AsynchronousChannelGroup

Управление асинхронными каналами

<details closed> 
    <summary>
        <b>Создание и настройка группы каналов</b>
    </summary>

```java
// Создание группы с фиксированным пулом потоков
AsynchronousChannelGroup fixedGroup = AsynchronousChannelGroup.withFixedThreadPool(
    10, 
    Executors.defaultThreadFactory()
);

// Создание группы с cached thread pool
AsynchronousChannelGroup cachedGroup = AsynchronousChannelGroup.withCachedThreadPool(
    Executors.newCachedThreadPool(),
    10 // initial size
);

// Создание группы с пользовательским исполнителем
ExecutorService customExecutor = Executors.newWorkStealingPool();
AsynchronousChannelGroup customGroup = AsynchronousChannelGroup.withThreadPool(customExecutor);

// Использование группы при создании каналов
AsynchronousServerSocketChannel serverChannel = AsynchronousServerSocketChannel
    .open(fixedGroup)
    .bind(new InetSocketAddress(8080));

// Мониторинг группы
System.out.println("Group is shutdown: " + fixedGroup.isShutdown());
System.out.println("Group is terminated: " + fixedGroup.isTerminated());

// Graceful shutdown
fixedGroup.shutdown();

// Принудительное завершение
fixedGroup.shutdownNow();
```
</details>

<details closed> 
    <summary>
        <b>Управление жизненным циклом</b>
    </summary>

```java
public class AsyncServerManager {
    
    private AsynchronousChannelGroup channelGroup;
    private AsynchronousServerSocketChannel serverChannel;
    
    public void startServer() throws IOException {
        // Создаем группу с обработкой необработанных исключений
        channelGroup = AsynchronousChannelGroup.withFixedThreadPool(4, new ThreadFactory() {
            
                private final AtomicInteger counter = new AtomicInteger();
                
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "async-io-" + counter.incrementAndGet());
                    thread.setUncaughtExceptionHandler((t, e) -> {
                        System.err.println("Uncaught exception in " + t.getName() + ": " + e.getMessage());
                    });
                    return thread;
                }
            }
        );
        
        serverChannel = AsynchronousServerSocketChannel.open(channelGroup);
        serverChannel.bind(new InetSocketAddress(8080));
        
        // Добавляем shutdown hook для graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdown));
    }
    
    public void shutdown() {
        try {
            if (serverChannel != null && serverChannel.isOpen()) {
                serverChannel.close();
            }
            
            if (channelGroup != null && !channelGroup.isShutdown()) {
                // Даем время на завершение текущих операций
                channelGroup.shutdown();
                if (!channelGroup.awaitTermination(10, TimeUnit.SECONDS)) {
                    channelGroup.shutdownNow();
                }
            }
        } catch (Exception e) {
            System.err.println("Error during shutdown: " + e.getMessage());
        }
    }
}
```
</details>

### Комбинирование с CompletableFuture

Асинхронные операции с Future

<details closed> 
    <summary>
        <b>Обертки для асинхронных операций</b>
    </summary>

```java
public class AsyncIOUtils {
    
    // Обертка для асинхронного чтения в CompletableFuture
    public static CompletableFuture<Integer> readAsync(
            AsynchronousFileChannel channel, 
            ByteBuffer buffer, 
            long position) {
        
        CompletableFuture<Integer> future = new CompletableFuture<>();
        
        channel.read(buffer, position, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer result, Void attachment) {
                future.complete(result);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                future.completeExceptionally(exc);
            }
        });
        
        return future;
    }
    
    // Обертка для асинхронной записи
    public static CompletableFuture<Integer> writeAsync(
            AsynchronousFileChannel channel,
            ByteBuffer buffer,
            long position) {
        
        CompletableFuture<Integer> future = new CompletableFuture<>();
        
        channel.write(buffer, position, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer result, Void attachment) {
                future.complete(result);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                future.completeExceptionally(exc);
            }
        });
        
        return future;
    }
    
    // Цепочка асинхронных операций
    public static CompletableFuture<Void> processFileAsync(Path inputPath, Path outputPath) {
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                return AsynchronousFileChannel.open(inputPath, StandardOpenOption.READ);
            } catch (IOException e) {
                throw new CompletionException(e);
            }
        }).thenCompose(inputChannel -> {
            ByteBuffer buffer = ByteBuffer.allocateDirect(8192);
            CompletableFuture<Void> processFuture = new CompletableFuture<>();
            
            processFileChunks(inputChannel, buffer, 0, outputPath, processFuture);
            return processFuture;
        });
    }
    
    private static void processFileChunks(AsynchronousFileChannel inputChannel, 
                                        ByteBuffer buffer, 
                                        long position,
                                        Path outputPath,
                                        CompletableFuture<Void> resultFuture) {
        
        readAsync(inputChannel, buffer, position).thenCompose(bytesRead -> {
            if (bytesRead == -1) {
                // Конец файла
                try {
                    inputChannel.close();
                } catch (IOException e) {
                    resultFuture.completeExceptionally(e);
                }
                resultFuture.complete(null);
                return CompletableFuture.completedFuture(null);
            }
            
            if (bytesRead > 0) {
                buffer.flip();
                // Обработка данных
                return processBufferAsync(buffer, outputPath, position)
                    .thenRun(() -> {
                        buffer.clear();
                        // Рекурсивно обрабатываем следующую часть
                        processFileChunks(inputChannel, buffer, position + bytesRead, 
                                        outputPath, resultFuture);
                    });
            }
            
            return CompletableFuture.completedFuture(null);
        }).exceptionally(throwable -> {
            resultFuture.completeExceptionally(throwable);
            return null;
        });
    }
}

// Использование
public class Main {

    public static void main(String[] args) {
        AsyncIOUtils.processFileAsync(Paths.get("input.txt"), Paths.get("output.txt"))
                        .thenRun(() -> System.out.println("File processing completed"))
                        .exceptionally(throwable -> {
                            System.err.println("Processing failed: " + throwable.getMessage());
                            return null;
                        });
    }
}
```
</details>

<details closed> 
    <summary>
        <b>Параллельная обработка нескольких файлов</b>
    </summary>

```java
public class ParallelFileProcessor {
    
    public CompletableFuture<Void> processMultipleFiles(List<Path> files) {
        List<CompletableFuture<Void>> processingFutures = files.stream()
            .map(this::processSingleFileAsync)
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(
            processingFutures.toArray(new CompletableFuture[0])
        );
    }
    
    private CompletableFuture<Void> processSingleFileAsync(Path file) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return AsynchronousFileChannel.open(file, StandardOpenOption.READ);
            } catch (IOException e) {
                throw new CompletionException(e);
            }
        }).thenCompose(channel -> {
            ByteBuffer buffer = ByteBuffer.allocate(4096);
            return readAndProcessEntireFile(channel, buffer, 0);
        });
    }
    
    private CompletableFuture<Void> readAndProcessEntireFile(AsynchronousFileChannel channel, 
                                                           ByteBuffer buffer, 
                                                           long position) {
        CompletableFuture<Void> result = new CompletableFuture<>();
        
        channel.read(buffer, position, null, new CompletionHandler<Integer, Void>() {
            
            @Override
            public void completed(Integer bytesRead, Void attachment) {
                if (bytesRead == -1) {
                    // Конец файла
                    try {
                        channel.close();
                        result.complete(null);
                    } catch (IOException e) {
                        result.completeExceptionally(e);
                    }
                    return;
                }
                
                if (bytesRead > 0) {
                    buffer.flip();
                    processChunk(buffer, position);
                    buffer.clear();
                    
                    // Рекурсивно читаем следующую часть
                    readAndProcessEntireFile(channel, buffer, position + bytesRead);
                } else {
                    // Продолжаем чтение с той же позиции
                    readAndProcessEntireFile(channel, buffer, position);
                }
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                result.completeExceptionally(exc);
            }
        });
        
        return result;
    }
}
```
</details>

### Best Practices и производительность

Типичные ошибки и их решение:

- Утечки памяти: Всегда закрывайте каналы и освобождайте буферы
- Потеря исключений: Всегда реализуйте оба метода CompletionHandler
- Неконтролируемый рост очереди: Реализуйте backpressure
- Голодание потоков: Настройте размер пула в соответствии с нагрузкой
- Race conditions: Используйте атомарные операции для управления состоянием

<details closed> 
    <summary>
        <b>Оптимизация производительности</b>
    </summary>

```java
public class AsyncIOBestPractices {
    
    // Использование прямых буферов для уменьшения копирования
    public ByteBuffer createOptimalBuffer(int size) {
        return ByteBuffer.allocateDirect(size);
    }
    
    // Пулы буферов для избежания частого выделения/освобождения памяти
    public class BufferPool {
        private final Queue<ByteBuffer> bufferPool = new ConcurrentLinkedQueue<>();
        private final int bufferSize;
        
        public BufferPool(int bufferSize, int initialCapacity) {
            this.bufferSize = bufferSize;
            
            for (int i = 0; i < initialCapacity; i++) {
                bufferPool.offer(ByteBuffer.allocateDirect(bufferSize));
            }
        }
        
        public ByteBuffer acquire() {
            ByteBuffer buffer = bufferPool.poll();
            return buffer != null ? buffer : ByteBuffer.allocateDirect(bufferSize);
        }
        
        public void release(ByteBuffer buffer) {
            buffer.clear();
            bufferPool.offer(buffer);
        }
    }
    
    // Обработка backpressure
    public class FlowControlledAsyncWriter {
        
        private final AsynchronousFileChannel channel;
        private final Queue<ByteBuffer> writeQueue = new ConcurrentLinkedQueue<>();
        private final AtomicBoolean writing = new AtomicBoolean(false);
        
        public void writeData(ByteBuffer data) {
            writeQueue.offer(data);
            tryStartWriting();
        }
        
        private void tryStartWriting() {
            if (writing.compareAndSet(false, true)) {
                writeNextChunk();
            }
        }
        
        private void writeNextChunk() {
            ByteBuffer buffer = writeQueue.poll();
            if (buffer == null) {
                writing.set(false);
                return;
            }
            
            channel.write(buffer, 0, null, new CompletionHandler<Integer, Void>() {
                
                @Override
                public void completed(Integer result, Void attachment) {
                    if (buffer.hasRemaining()) {
                        // Если не все записано, продолжаем с этого же буфера
                        channel.write(buffer, 0, null, this);
                    } else {
                        // Переходим к следующему буферу
                        writeNextChunk();
                    }
                }
                
                @Override
                public void failed(Throwable exc, Void attachment) {
                    System.err.println("Write failed: " + exc.getMessage());
                    writing.set(false);
                }
            });
        }
    }
}

// Мониторинг и метрики
public class AsyncIOMetrics {
    private final AtomicLong readOperations = new AtomicLong();
    private final AtomicLong writeOperations = new AtomicLong();
    private final AtomicLong failedOperations = new AtomicLong();
    
    public void incrementReadOps() { readOperations.incrementAndGet(); }
    public void incrementWriteOps() { writeOperations.incrementAndGet(); }
    public void incrementFailedOps() { failedOperations.incrementAndGet(); }
    
    public void printMetrics() {
        System.out.println("Async IO Metrics:");
        System.out.println("Read operations: " + readOperations.get());
        System.out.println("Write operations: " + writeOperations.get());
        System.out.println("Failed operations: " + failedOperations.get());
    }
}
```
</details>

<details closed> 
    <summary>
        <b>Настройка JVM для асинхронного I/O</b>
    </summary>

Увеличение direct memory для буферов
- `-XX:MaxDirectMemorySize=1G`

Настройка пула потоков NIO
- `-Djava.nio.channels.DefaultThreadPool.initialSize=20`
- `-Djava.nio.channels.DefaultThreadPool.maxSize=100`

Мониторинг direct memory
- `-XX:NativeMemoryTracking=detail`
</details>


