# 03、Java线程间的通信

## 锁与同步

-  锁的概念都是基于对象的，所以我们又经常称它为对象锁。线程和锁的关系，一个锁同一时间只能被一个线程持有。
-  线程同步是线程之间按照**一定的顺序**执行。

**例子1**：A先打印，B在打印

```java
public class ObjectLock {
    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread A " + i);
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread B " + i);
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        //防止线程B先得到锁
        Thread.sleep(10);
        new Thread(new ThreadB()).start();
    }
}
```

如果不加 synchronized ，A、B 线程会交替打印出来；加了 synchronized ，这样 A 线程先获得锁，B 线程就会进入就绪状态，等待 A 线程释放锁，待 A 线程 执行完，跳出 synchronized 的范围，B线程就可以争抢 lock 锁，进入 synchronized  的范围执行了。

## 等待/通知机制

上面一种基于“锁”的方式，线程需要不断地去尝试获得锁，如果失败了，再继续尝试。这可能会耗费服务器资源。

而等待/通知机制是另一种方式 ：

**需要注意：一下三个方法必须在 synchronized 作用域中，否则报异常 **

- Object.wait() :
  -  释放当前持有的锁；
  - 下一次被唤醒后并且重新持有锁，会执行 wait() 之后的代码。
- Object.notify()：
  - 会随机叫醒一个正在等待的线程。
  - 调用 notify() 并不会立即释放锁；
  - 必须跳出 synchronized 的作用域才会释放锁；
  - 一般情况下会按 wait 顺序唤醒，具体的实现取决于 JVM 实现。
- Object.notifyAll()： 
  - 会叫醒所有正在等待的线程。
  - 一般情况下会按 wait 倒序唤醒，具体的实现取决于 JVM 实现。

了解了这个机制我们可以设计出新的程序

1.  假如线程A现在持有了一个锁`lock`并开始执行，它可以使用`lock.wait()`让自己进入等待状态。这个时候，`lock`这个锁是被释放了的。 
2.  这时，线程B获得了`lock`这个锁并开始执行，它可以在某一时刻，使用`lock.notify()`，通知之前持有`lock`锁并进入等待状态的线程A，说“线程A你不用等了，可以往下执行了”。 

例子2：A、B 线程交替打印

```java
/**
 * 使用 wait 和 notify 实现交替打印
 */
public class WaitAndNotify {
    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadA: " + i);
                        //唤醒其他线程
                        lock.notify();
                        //释放锁
                        lock.wait();
                        System.out.println(Thread.currentThread().getName());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //必须唤醒其他线程，负责线程B 会一直在wait
                lock.notify();
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadB: " + i);
                        lock.notify();
                        lock.wait();
                        System.out.println(Thread.currentThread().getName());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                lock.notify();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}
```

>  需要注意的是等待/通知机制使用的是使用同一个对象锁，如果你两个线程使用的是不同的对象锁，那它们之间是不能用等待/通知机制通信的。 

## 信号量

###  volatile 

volatile关键字能够保证内存的可见性，如果用volatile关键字声明了一个变量，在一个线程里面改变了这个变量的值，那其它线程是立马可见更改后的值的。 

例子3： A输出0，然后线程B输出1，再然后线程A输出2…以此类推 

```java
/**
 * A 、B 线程交替累加
 */
public class Signal {
    private static volatile int signal = 0;

    static class ThreadA implements Runnable {
        @Override
        public void run() {

            while (signal < 5) {
                //System.out.print("") 包含了 synchronized，而 synchronized 具有可见性
                //打开下面两System.out.print("")注释 可以不需要有 volatile.
                //System.out.print("");
                if (signal % 2 == 0) {
                    System.out.println("threadA: " + signal);
                    signal++;
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            while (signal < 5) {
                //System.out.print("");
                if (signal % 2 == 1) {
                    System.out.println("threadB: " + signal);
                    signal = signal + 1;
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}
```



## 管道

 管道是基于“管道流”的通信方式。JDK提供了`PipedWriter`、 `PipedReader`、 `PipedOutputStream`、 `PipedInputStream`。其中，前面两个是基于字符的，后面两个是基于字节流的。 

例子4：通过管道进行线程间的通信

```java
/**
 * 通过管道进行线程间的通信
 */
public class Pipe {
    static class ReaderThread implements Runnable {
        private PipedReader reader;

        public ReaderThread(PipedReader reader) {
            this.reader = reader;
        }

        @Override
        public void run() {
            System.out.println("this is reader");
            int receive = 0;
            try {
                while ((receive = reader.read()) != -1) {
                    System.out.print((char) receive);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class WriterThread implements Runnable {

        private PipedWriter writer;

        public WriterThread(PipedWriter writer) {
            this.writer = writer;
        }

        @Override
        public void run() {
            System.out.println("this is writer");
            int receive = 0;
            try {
                writer.write("test");
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        PipedWriter writer = new PipedWriter();
        PipedReader reader = new PipedReader();
        writer.connect(reader); // 这里注意一定要连接，才能通信

        new Thread(new ReaderThread(reader)).start();
        Thread.sleep(1000);
        new Thread(new WriterThread(writer)).start();
    }
}
```

 执行流程： 

1. 线程ReaderThread开始执行，
2. 线程ReaderThread使用管道reader.read()进入”阻塞“，
3. 线程WriterThread开始执行，
4. 线程WriterThread用writer.write("test")往管道写入字符串，
5. 线程WriterThread使用writer.close()结束管道写入，并执行完毕，
6. 线程ReaderThread接受到管道输出的字符串并打印，
7. 线程ReaderThread执行完毕。

管道通信的应用场景:

- 使用管道多半与I/O流相关。当我们一个线程需要先另一个线程发送一个信息（比如字符串）或者文件等等时，就需要使用管道通信了 



## 其它通信相关

### join

join()方法是Thread类的一个实例方法。它的作用是让当前线程陷入“等待”状态，等join的这个线程执行完成后，再继续执行当前线程。

例子5：

```java
/**
 * 等待 线程A 执行完毕，在执行当前线程后续的代码
 */
public class Join {
    static class ThreadA implements Runnable {

        @Override
        public void run() {
            try {
                System.out.println("我是子线程，我先睡一秒");
                Thread.sleep(1000);
                System.out.println("我是子线程，我睡完了一秒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new ThreadA());
        thread.start();
        //等待 thread 执行完成，在继续
        thread.join();
        System.out.println("如果不加join方法，我会先被打出来，加了就不一样了");
    }
}
```

应用场景：

-  如果主线程想等待子线程执行完毕后，获得子线程中的处理完的某个数据，就要用到join方法了 

### sleep

sleep方法是Thread类的一个静态方法。它的作用是让当前线程睡眠一段时间。它有这样两个方法：

- Thread.sleep(long)
- Thread.sleep(long, int)

**sleep 和 wait 的区别**

- wait可以指定时间，也可以不指定；而sleep必须指定时间。
- wait释放cpu资源，同时释放锁；sleep释放cpu资源，但是不释放锁，所以易死锁。
- wait必须放在同步块或同步方法中，而sleep可以再任意位置。

### ThreadLocal

ThreadLocal是一个本地线程副本变量工具类。内部是一个**弱引用**的Map来维护。 ThreadLocal类并不属于多线程间的通信，而是让每个线程有自己”独立“的变量，线程之间互不影响。它为每个线程都创建一个**副本**，每个线程可以访问自己内部的副本变量。 

```java
public class ThreadLocalDemo {
    static class ThreadA implements Runnable {
        private ThreadLocal<String> threadLocal;

        public ThreadA(ThreadLocal<String> threadLocal) {
            this.threadLocal = threadLocal;
        }

        @Override
        public void run() {
            threadLocal.set("A");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("ThreadA输出：" + threadLocal.get());
        }

        static class ThreadB implements Runnable {
            private ThreadLocal<String> threadLocal;

            public ThreadB(ThreadLocal<String> threadLocal) {
                this.threadLocal = threadLocal;
            }

            @Override
            public void run() {
                threadLocal.set("B");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("ThreadB输出：" + threadLocal.get());
            }
        }

        public static void main(String[] args) {
            ThreadLocal<String> threadLocal = new ThreadLocal<>();
            new Thread(new ThreadA(threadLocal)).start();
            new Thread(new ThreadB(threadLocal)).start();
        }
    }
}
```

**应用场景**

-  数据库连接、Session管理等 

**参考文章**

> http://concurrent.redspider.group/article/01/5.html



