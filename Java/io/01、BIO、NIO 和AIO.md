# BIO、NIO 和AIO

**我们先来理解几个概念：**

- 同步：发起一个调用，调用未处理完请求，调用不符合。（按顺序做事情一件一件）。
- 异步：发起一个调用后，被调用者返回已经收到调用请求，但是没有返回处理结果。当前线程可以继续处理其他事情，被调用者依靠事件通知，回调等机制通知调用者，返回结果。
- 阻塞：发起一个请求，需要等待请求返回结果，期间线程只能等待结果的返回，无法从事其他工作。
- 非阻塞：发起一个请求，无论数据是否准好了，直接返回，去做其他事情。

## BIO、NIO 和AIO 是什么？

### BIO（Blocking I/O) 

BIO（Blocking I/O) : 同步阻塞IO模式，数据读写必须阻塞在同一个线程内等待其完成。

#### 传统 BIO 模型

传统 BIO 采用的是(一请求一应答) 如图 01。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/io/01/bna_01.png)

<center>图01 单个 BIO</center>
#### BIO 通信模型

为了扩大效率，我们需要加入多线程，提高效率。引入了 BIO 通信模型，通常由一个独立的 Acceptor 线程负责监听客户端请求连接。我们一般的是实现方式是 `while(true)` 循环调用 `accept()` 方法等待接收客户端的连接；一旦收到一个请求连接，就可以新建一个线程来负责建立通信套接字在这个通信套接字上进行读写操作； 处理完成之后，通过输出流返回应答给客户端，线程销毁 。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/io/01/bna_02.png)

<center>图2  BIO 通信模型</center>
#### 伪异步的 I/O 通信框架 

BIO通信模式使IO的并发能力提升了，但是随着连接数的增多，线程数是线性增加的，这样是不利于系统并发和运行的，过多的线程反而会导致上下文切换频繁，性能低下的问题。我们继续改造，将新建线程的步骤用线程池来替换，实现伪异步的 I/O 通信框架 如图 03。实现步骤为：

1. Acceptor 监听请求的连接；
2. 如果有连接进来，将客户端的 Socket 封装成为一个 Task ；
3. 把封装好的 Task 交给线程池处理，Acceptor 继续监听请求的连接；
4. 在线程池中，获得执行 Task 的线程，处理对应的请求，等待处理好了，返回处理结果。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/io/01/bna_03.png)

<center>图03 伪异步的 I/O 通信框架 </center>
伪异步I/O通信框架采用了线程池实现，因此避免了为每个请求都创建一个独立线程造成的线程资源耗尽问题。不过因为它的底层任然是同步阻塞的 BIO 模型，因此无法从根本上解决问题。 

#### 实战 BIO

server 端：

```java
public class Server {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        try {
            ServerSocket serverSocket = new ServerSocket(8080);
            while (true) {
                Socket socket = serverSocket.accept();
                executorService.execute(new Exec(socket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            executorService.shutdown();
        }


    }

    static class Exec implements Runnable {
        Socket socket;

        public Exec(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try(
                    Socket socket = this.socket;
                    OutputStream outputStream = socket.getOutputStream();
                    InputStream inputStream = socket.getInputStream()
            ) {
                int len;
                byte[] data = new byte[1024];
                while ((len = inputStream.read(data)) != -1) {
                    System.out.println(new String(data, 0, len));
                }
                socket.shutdownInput();
                outputStream.write(("hello ，" + System.currentTimeMillis()).getBytes());
                outputStream.flush();
                socket.shutdownOutput();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

client:

```java
public class Client {
    public static void main(String[] args) {
        Thread t1 = new Thread(new Request("t1"));
        Thread t2 = new Thread(new Request("t2"));
        Thread t3 = new Thread(new Request("t3"));
        t1.start();
        t2.start();
        t3.start();
    }

    static class Request implements Runnable {
        String name;

        public Request(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            try (Socket socket = new Socket("127.0.0.1", 8080);
                 InputStream inputStream = socket.getInputStream();
                 OutputStream outputStream = socket.getOutputStream()) {
                outputStream.write((new Date() + ": hello world").getBytes());
                outputStream.flush();
                int len;
                byte[] data = new byte[1024];
                System.out.println("响应内容:");
                socket.shutdownOutput();
                while ((len = inputStream.read(data)) != -1) {
                    System.out.println(new String(data, 0, len));
                }
                socket.shutdownInput();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}
```

### NIO（ Non-Blocking  I/O) 

NIO（Non-Blocking I/O) : 同步非阻塞IO模式，在Java 1.4 中引入了 NIO 框架，对应 `java.nio` 包，提供了 Channel 、Selector、Buffer等抽象 。 它支持面向缓冲的，基于通道的I/O操作方法。

NIO 提供了与传统 BIO 模拟中的 Socket 和  ServerSocket 相对的 SocketChannel 和 ServerSocketChannel 两种不同的套接字管道实现，两种管道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞I/O来提升开发速率和更好的维护性；对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发。 

#### 三大组件

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/java/io/01/bna_04.png)

<center>图04 NIO 模型</center>

- Buffer( 缓冲区 )：
  - BIO 是面向流，而 NIO 是面向缓存区。而 Buffer 就是 NIO 读写数据的地方。 Buffer缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该模块内存;
  -  NIO 在读取数据时，它是直接读到缓冲区中的; 在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作；
  -  最常用的缓冲区是 ByteBuffer，一个 ByteBuffer 提供了一组功能用于操作 byte 数组 。
- Channel(管道) 
  - Channel 是 NIO 读取 Buffer 中数据的渠道，通道是双向的，可读也可写 。
- Selector(多路复用器) 
  - NIO 的多路复用器允许一个单独的线程来监视多个输入通道，你可以注册多个通道使用一个选择器，然
    后使用一个单独的线程来“选择”通道：这些通道里已经有可以处理的输入，或者选择已准备写入的通道。
  - 使得一个单独的线程很容易来管理多个通道。

#### 实战 NIO

NIOServer:

```java
public class NIOServer {
    private int port = 8080;
    private Selector selector;
    private ByteBuffer buffer = ByteBuffer.allocate(1024);

    public NIOServer(int port) {
        try {
            this.port = port;
            ServerSocketChannel server = ServerSocketChannel.open();
            server.bind(new InetSocketAddress(this.port));
            //BIO 升级版本 NIO，为了兼容BIO，NIO模型默认是采用阻塞式
            server.configureBlocking(false);
            selector = Selector.open();
            server.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    private void listen() {
        System.out.println("listen on " + this.port + ".");
        try {
            while (true) {
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iter = keys.iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    iter.remove();
                    process(key);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void process(SelectionKey key) throws IOException {
        if (key.isAcceptable()) {
            ServerSocketChannel server = (ServerSocketChannel) key.channel();
            SocketChannel channel = server.accept();
            channel.configureBlocking(false);
            channel.register(selector,SelectionKey.OP_READ);
        }
        else if (key.isReadable()){
            //从多路复用器中拿到客户端的引用
            SocketChannel channel = (SocketChannel) key.channel();
            int len = channel.read(buffer);
            if (len>0){
                buffer.flip();
                String content = new String(buffer.array(),0,len);
                key = channel.register(selector,SelectionKey.OP_WRITE);
                key.attach(content);
                System.out.println("读取内容：" + content);
            }
        }
        else if (key.isWritable()){
            SocketChannel channel = (SocketChannel) key.channel();
            String content = (String) key.attachment();
            channel.write(ByteBuffer.wrap(("输出：" + content).getBytes()));
            channel.close();
        }

    }

    public static void main(String[] args) {
        new NIOServer(8080).listen();
    }
}
```

NIOClient :

```java
public class NIOClient {
    public static void main(String[] args) {
        try (
                Socket socket = new Socket("127.0.0.1", 8080);
                OutputStream outputStream = socket.getOutputStream();
                InputStream inputStream = socket.getInputStream();
        ) {
            PrintWriter printWriter = new PrintWriter(outputStream);
            printWriter.write("你好,你在学习 NIO 吗？");
            printWriter.flush();
            byte b[] = new byte[1024] ;        // 所有的内容都读到此数组之中
            inputStream.read(b) ;
            System.out.println(new String(b));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}


```

### AIO（ Asynchronous I/O) 

AIO（ Asynchronous I/O) : 异步IO，JDK1.7 才是真正实现异步 AIO、把 IO 读写操作完全交给操作系统，学习了 linux epoll 模式。

#### 基本原理

服务端：AsynchronousServerSocketChannel

客户端：AsynchronousSocketChannel

用户处理器：CompletionHandler 接口，这个接口实现应用程序向操作系统发起 IO 请求,当完成后处理具体逻辑，否则做
自己该做的事情。

**异步IO模型中，当用户线程收到通知时，数据已经被内核读取完毕，并放在了用户线程指定的缓冲区内，内核在IO完成后通知用户线程直接使用即可。**

#### 实战 AIO

AIOClient:

```java
public class AIOClient {
    private final AsynchronousSocketChannel client;

    public AIOClient() throws Exception{
        client = AsynchronousSocketChannel.open();
    }

    public void connect(String host,int port){
        client.connect(new InetSocketAddress(host, port), null, new CompletionHandler<Void, Void>() {
            @Override
            public void completed(Void result, Void attachment) {
                try {
                    client.write(ByteBuffer.wrap("这是一条测试数据".getBytes())).get();
                    System.out.println("已发送至服务器");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });
        final ByteBuffer bb = ByteBuffer.allocate(1024);
        client.read(bb, null, new CompletionHandler<Integer, Object>() {
            @Override
            public void completed(Integer result, Object attachment) {
                System.out.println("IO 操作完成" + result);
                System.out.println("获取反馈结果" + new String(bb.array()));
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException ex) {
            System.out.println(ex);
        }
    }
    public static void main(String args[])throws Exception{
        new AIOClient().connect("localhost",8080);
    }

}
```

AIOServer:

```java
/**
 * AIO 服务端
 */
public class AIOServer {

    private final int port;

    public AIOServer(int port) {
        this.port = port;
        listen();
    }

    private void listen() {
        try {
            ExecutorService executorService = Executors.newCachedThreadPool();
            AsynchronousChannelGroup threadGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 1);
            final AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open(threadGroup);
            server.bind(new InetSocketAddress(port));
            System.out.println("服务已启动，监听端口" + port);

            server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
                final ByteBuffer buffer = ByteBuffer.allocate(1024);

                @Override
                public void completed(AsynchronousSocketChannel result, Object attachment) {
                    try {
                        System.out.println("IO 操作成功，开始获取数据");
                        buffer.clear();
                        result.read(buffer).get();
                        buffer.flip();
                        result.write(buffer);
                        buffer.flip();
                    } catch (InterruptedException | ExecutionException e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            result.close();
                            server.accept(null, this);
                        } catch (Exception e) {
                            System.out.println(e.toString());
                        }
                    }
                    System.out.println("操作完成");
                }

                @Override
                public void failed(Throwable exc, Object attachment) {
                    System.out.println("IO 操作是失败: " + exc);
                }
            });
            try {
                Thread.sleep(Integer.MAX_VALUE);
            } catch (InterruptedException ex) {
                System.out.println(ex);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        int port = 8080;
        new AIOServer(port);
    }


}
```



## 对比

### BIO 与 NIO

- BIO 面向 IO 流，NIO 面向字节流和字符流。NIO基于Channel和Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。
- BIO是一个连接一个线程阻塞等待返回结果，NIO是有个专门的线程轮询查询IO的状态。

#### **BIO 、NIO 和 AIO 本质区别**

- BIO：同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然可以通过线程池机制改善。
- NIO：同步非阻塞，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。
- AIO：异步非阻塞，服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。（和系统有关）

#### 理解

假设有这么一个场景，有一排水壶（客户）在烧水。

- AIO的做法是，每个水壶上装一个开关，当水开了以后会提醒对应的线程去处理。
- NIO的做法是，叫一个线程不停的循环观察每一个水壶，根据每个水壶当前的状态去处理。
- BIO的做法是，叫一个线程停留在一个水壶那，直到这个水壶烧开，才去处理下一个水壶。



> 参考文章
>
> 1. https://blog.csdn.net/m0_38109046/article/details/89449305 
> 2. https://blog.csdn.net/meism5/article/details/89469101