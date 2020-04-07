# 第21章 并发

## 21.1 并发的多面性

### 1.并发所解决的问题？

- 速度
- 设计可管理性

### 2.Java的线程机制？

Java的线程机制是抢占式的，这表示调度机制会周期性地中断线程，将上下文切换到另一个线程，从而为每个线程都提供时间片，使得每个线程都会分配到数量合理的时间去驱动它的任务。

## 21.2 基本的线程控制

### 1.线程和进程的区别和联系？

一个线程就是在进程中的一个单一的顺序控制流，因此，单个进程可以拥有多个并发执行的任务。

### 2.创建线程有哪些方式？

1. 继承Thread类
2. 实现Runnable接口
3. 使用线程池创建
4. 实现Callable接口

### 3.创建线程：继承Thread类

```java
public class MyThread extends Thread {
    private static int task = 0;
    private final int id = task++;

    @Override
    public void run() {
        int i = 1;
        while (i <= 4) {
            System.out.println("#" + id + " " + i);
            i++;
            Thread.yield();
        }
        System.out.println();
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.start();
        //不能再次调用
        //myThread.start();
        MyThread myThread1 = new MyThread();
        myThread1.start();
        //非顺序执行
        //可以把start方法写到构造器里面，实现new Thread()便执行
    }
}
```



### 4.创建线程：实现Runnable接口

```java
public class MyRunnable implements Runnable {
    private static int task = 0;
    private final int id = task++;
    @Override
    public void run() {
        int i = 3;
        while(i >= 1) {
            i--;
            System.out.println("#(" + id + ")" + System.currentTimeMillis());
            //让步：向线程调度器提出此时可以切换任务（具有相同优先级的其他线程）的建议
            Thread.yield();
        }
    }

    public static void main(String[] args) {
        //顺序执行：#0，#1，#2
        new MyRunnable().run();
        new MyRunnable().run();
        new MyRunnable().run();
        //非顺序执行
        Thread thread = new Thread(new MyRunnable());
        thread.start();
        Thread thread1 = new Thread(new MyRunnable());
        thread1.start();
        Thread thread2 = new Thread(new MyRunnable());
        thread2.start();
        new Thread(new MyRunnable()).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Inner");
            }
        }).start();
    }
}
```

### 5.创建线程：使用线程池创建

```java
public class CachedThreadPool {
    public static void main(String[] args) {
        //执行器允许管理异步任务的执行，而无须显式地管理线程的生命周期
        ExecutorService service = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i++) {
            //CachedThreadPool将为每一个任务创建一个线程
            service.execute(new MyRunnable());
        }
        //调用shutdown可以防止新任务被提交给这个Executor
        //当前线程将继续运行在shutdown被调用之前提交的任务
        service.shutdown();
    }
}
```

```java
public class FixedThreadPool {
    public static void main(String[] args) {
        //FixedThreadPool使用有限的线程集来执行所提交的任务
        ExecutorService service = Executors.newFixedThreadPool(2);
        for (int i = 0; i < 5; i++) {
            service.execute(new MyRunnable());
        }
        service.shutdown();
        //Out: #0 #1 #2 #3 #4 #5 when nThreads = 1
    }
}
```

```java
public class SingleThreadExecutor {
    public static void main(String[] args) {
        //SingleThreadExecutor相当于线程数量为1的FixedThreadPool
        //适合长期存活的任务
        ExecutorService service = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 5; i++) {
            service.execute(new MyRunnable());
        }
        service.shutdown();
        //Out: #0 #1 #2 #3 #4 #5
    }
}
```

### 6.创建线程：实现Callable接口

```java
public class TaskWithResult implements Callable<String> {
    @Override
    public String call() throws Exception {
        return System.currentTimeMillis() % 2 == 0 ? "偶数" : "奇数";
    }
}
```

```java
public class CallableDemo {
    public static void main(String[] args) {
        ExecutorService service = Executors.newCachedThreadPool();
        Future<String> future = service.submit(new TaskWithResult());
        try {
            System.out.println(future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
            return;
        } catch (ExecutionException e) {
            e.printStackTrace();
        } finally {
            service.shutdown();
        }
        //Out: 偶数或奇数
    }
}
```

### 7.利用休眠使任务终止执行给定的时间

调用Thread.sleep或TimeUnit.MILLISECONDS.sleep方法以休眠（均会抛出Interrupted Exception异常）

```java
public class MyRunnable implements Runnable {
    private static int task = 0;
    private final int id = task++;
    @Override
    public void run() {
        int i = 3;
        while(i >= 1) {
            i--;
            System.out.println("#(" + id + ")" + System.currentTimeMillis());
            try {
                //或者用：Thread.sleep(1000);
                TimeUnit.MILLISECONDS.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //向线程调度器提出此时可以切换任务的建议
            Thread.yield();
        }
    }
}

public class FixedThreadPool {
    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(1);
        for (int i = 0; i < 5; i++) {
            service.execute(new MyRunnable());
        }
        service.shutdown();
        //Out: #0 #1 #2 #3 #4 #5 每隔一秒输出
    }
}
```

### 8.线程的优先级

调度器倾向于让优先级最高的线程执行，优先级较低的线程仅仅是执行的频率较低；

所有线程都应该以默认的优先级运行，试图操纵优先级通常是一种错误。

```java
//在run中设置优先级
Thread.currentThread().setPriority(Thread.MAX_PRIORITY);
Thread.currentThread().setPriority(Thread.NORM_PRIORITY);
Thread.currentThread().setPriority(Thread.MIN_PRIORITY);
```

### 9.创建后台线程

```java
public class Daemons {
    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                //如果是一个后台线程，那么它创建的任何线程将被自动设置成后台线程
                Thread thread1 = new Thread(new Runnable() {
                    @Override
                    public void run() {
                    }
                });
                thread1.start();
                System.out.println("Inner：" + thread1.isDaemon());
                //Out: true
            }
        });
        //必须在线程启动之前调用setDaemon()方法
        thread.setDaemon(true);
        thread.start();
        System.out.println("Outer: " + thread.isDaemon());
        //Out: true
        //当最后一个非后台线程终止时，后台线程会“突然”终止，finally子句也不会执行。
    }
}
```

### 10.加入一个线程

```java
public class Joining {
    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.MILLISECONDS.sleep(1500);
                    //0 Out: thread over; false; thread1 join over
                    //1500 Out: thread interrupt; false; thread1 join over;
                } catch (InterruptedException e) {
                    System.out.println("thread interrupt");
                    return;
                }
                System.out.println("thread over");
            }
        });
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    thread.join();
                    System.out.println(thread.isAlive());
                    //thread1被挂起，直到thread线程结束才恢复
                    System.out.println("thread1 join over");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.start();
        thread.interrupt();
        thread1.start();
    }
}
```

### 11.捕获异常

```java
public class ExceptionThread implements Runnable {
    @Override
    public void run() {
        throw new RuntimeException();
    }
}
public class MyUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("捕获到异常");
    }
}
public class CaptureUncaughtException {
    public static void main(String[] args) {
        try {
            //第一种：设置为默认的未捕获异常处理器
            //Thread.setDefaultUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
            //第二种：修改Executor产生线程的方式
            ExecutorService service = Executors.newCachedThreadPool(new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r);
                    t.setUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
                    return t;
                }
            });
            service.execute(new ExceptionThread());
            //显示创建线程时
            Thread thread = new Thread(new ExceptionThread());
            thread.setUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
            thread.start();
            //Out: 捕获到异常; 捕获到异常
        } catch (Exception e) {
            System.out.println("不会被打印：不能捕获从线程中逃逸的异常");
        }
    }
}
```



## 21.3 共享受限资源

### 1.不正确的访问资源

```java
public class Test {
    static int i = 0;
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                int n = 0;
                while (n < 100000) {
                    n++;
                    //资源竞争
                    i++;
                }
                System.out.println(i);
            }
        };
        new Thread(runnable).start();
        new Thread(runnable).start();
        //Out: ...; != 200000
    }
}

```

```java
public class Test {
    static int i = 0;
    static Object lock = new Object();
    public static void main(String[] args) {
        Runnable runnableLock = () -> {
            int n = 0;
            while (n < 100000) {
                n++;
                synchronized (lock) {
                    i++;
                }
            }
            System.out.println(i);
        };
        new Thread(runnableLock).start();
        new Thread(runnableLock).start();
        //Out: ...; == 200000
    }
}
```

### 2.解决资源竞争的方法有哪些？

* 使用synchronized关键字（检查锁是否可用，获取锁，执行代码，释放锁）
* 使用显式的Lock对象（显式创建，锁定和释放）

### 3.加锁：使用synchronized关键字

* 对象锁作用在非静态方法上，锁是当前实例对象 ，进入同步代码前要获得当前实例的锁
* 类锁作用在静态方法上，锁是当前类的class对象 ，进入同步代码前要获得当前类Class对象的锁
* 作用在代码块上，锁是括号里面的对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁

```java
public class SynchronizedTest {
    static int i = 0;
    static Object lock = new Object();
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                synchronized (lock) {
                    //给定对象锁
                    i++;
                }
            }
            public synchronized void addOne() {
                //对象锁
                i++;
            }
        };
    }
    public static synchronized void addOne() {
        //类锁
        i++;
    }
}
```

### 4.加锁：使用显式的Lock对象

```java
public class LockTest {
    static int i = 0;
    static Lock lock = new ReentrantLock();
    public static void main(String[] args) {
        Runnable runnableLock = () -> {
            int n = 0;
            while (n < 100000) {
                n++;
                lock.lock();
                try {
                    i++;
                    //如果有return语句，必须在try子句中出现，以确保unlock()不会过早发生
                } finally {
                    //使用synchronized时，某些事物失败了，那么就会抛出一个异常
                    //而有显式的Lock对象可以使用finally子句将系统维护在正确的状态
                    //通常只有在解决特殊问题时，才使用显式的Lock对象
                    lock.unlock();
                }
            }
            System.out.println(i);
        };
        new Thread(runnableLock).start();
        new Thread(runnableLock).start();
        //Out: ...; 200000
    }
}
```

### 5.原子性与易变性

如果将一个域声明为volatile的，那么只要对这个域产生了写操作，那么所有的读操作就都可以看到这个修改。

对基本类型的读取和赋值操作被认为是安全的原子操作，递增不是原子操作。

```java
public class VolatileTest {
    static volatile int i = 0;
    public static void main(String[] args) {
        Runnable runnableLock = () -> {
            int n = 0;
            while (n < 100000) {
                n++;
                //递增不是原子操作
                i++;
            }
            System.out.println(i);
        };
        new Thread(runnableLock).start();
        new Thread(runnableLock).start();
        //Out: ...; != 200000
    }
}
```

### 6.原子类

```java
public class AtomicIntegerTest {
    private static AtomicInteger atomicInteger = new AtomicInteger(0);

    public static void main(String[] args) {
        Runnable runnableLock = new Runnable() {
            @Override
            public void run() {
                int n = 0;
                while (n < 100000) {
                    n++;
                    atomicInteger.incrementAndGet();
                }
                System.out.println(atomicInteger.get());
            }
        };
        new Thread(runnableLock).start();
        new Thread(runnableLock).start();
        //Out: ...; == 200000
    }
}
```

### 7.临界区

多个线程同时访问方法内部的部分代码而不是访问整个方法，分离出来的代码段被称为临界区（同步控制块）。

可以使多个任务访问对象的时间性能得到显著提高。

### 8.加锁方式总结

```java
public class Main {
    public static void main(String[] args) {
        //以下均用MyRunnable.lock加锁，static
        Thread thread = new Thread(new MyRunnable());
        Thread thread1 = new Thread(new MyRunnable());
        Thread thread2 = new Thread(new MyRunnable1());
        Thread thread3 = new Thread(new MyRunnable1());
        thread.start();
        thread1.start();
        thread2.start();
        thread3.start();
        //Out: ...; ...; ...; 400000
        //以下均用MyRunnable的类对象加锁，class
        Thread thread4 = new Thread(new MyRunnable2());
        Thread thread5 = new Thread(new MyRunnable2());
        thread4.start();
        thread5.start();
        //Out: ...; 200000;
        //以下均用runnableLock加锁，this
        Runnable runnableLock = new MyRunnable3();
        Thread thread6 = new Thread(runnableLock);
        Thread thread7 = new Thread(runnableLock);
        thread6.start();
        thread7.start();
        //Out: ...; 200000;
    }
}

public class MyRunnable implements Runnable {
    public static int i = 0;
    //必须用static才能加锁成功
    final static Object lock = new Object();

    @Override
    public void run() {
        int n = 0;
        while (n < 100000) {
            n++;
            synchronized (lock) {
                i++;
            }
        }
        System.out.println("MyRunnable: " + i);
    }
}


public class MyRunnable1 implements Runnable {
    @Override
    public void run() {
        int n = 0;
        while (n < 100000) {
            n++;
            synchronized (MyRunnable.lock) {
                MyRunnable.i++;
            }
        }
        System.out.println("MyRunnable1: " + MyRunnable.i);
    }
}

public class MyRunnable2 implements Runnable {
    private static int i = 0;
    @Override
    public void run() {
        int n = 0;
        while (n < 100000) {
            n++;
            increment();
        }
        System.out.println("MyRunnable2: " + i);
    }

    public synchronized static void increment() {
        i++;
    }
}

public class MyRunnable3 implements Runnable {
    private static int i = 0;
    @Override
    public void run() {
        int n = 0;
        while (n < 100000) {
            n++;
            increment();
        }
        System.out.println("MyRunnable3: " + i);
    }

    public synchronized void increment() {
        i++;
    }
}
```

### 9.线程本地存储

```java
public class ThreadLocalVariableHolder {
    private static ThreadLocal<Integer> integerThreadLocal = new ThreadLocal<>() {
        private Random random = new Random(47);
        @Override
        protected Integer initialValue() {
            return random.nextInt(10000);
        }
    };

    public static void increment() {
        integerThreadLocal.set(integerThreadLocal.get() + 1);
    }

    public static int get() {
        return integerThreadLocal.get();
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 4; i++) {
            executorService.execute(new Accessor(i));
        }
        executorService.shutdown();
        //Out: 每个不同的线程都创建不同的存储
        //Accessor1: 556
        //Accessor3: 6694
        //Accessor2: 1862
        //Accessor0: 9259
    }
}

public class Accessor implements Runnable {
    private final int id;

    public Accessor(int id) {
        this.id = id;
    }
    @Override
    public void run() {
        ThreadLocalVariableHolder.increment();
        System.out.println("Accessor" + id + ": " + ThreadLocalVariableHolder.get());
    }
}
```



## 21.4 终结任务

### 1.线程有哪些状态？

* 新建
* 就绪
* 阻塞
* 死亡

### 2.线程进入阻塞状态的原因有哪些？

* 调用sleep()使任务进入休眠状态
* 调用wait()使线程挂起
* 任务在等待某个输入/输出完成
* 任务试图在某个对象上调用其同步控制方法，当另一个任务已经获取了这个锁

### 3.如何中断

如果一个线程已经被阻塞，或者试图执行一个阻塞操作，那么这个线程的中断状态将抛出Interrupted Exception

```java
public class InnerRunnable implements Runnable {
    @Override
    public void run() {
        try {
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName());
        } catch (InterruptedException e) {
            System.out.println("Interrupted");
        }
    }
}

public class Interrupted {
    public static void main(String[] args) {
        Thread t = new Thread(new InnerRunnable());
        t.interrupt();
        ExecutorService service = Executors.newCachedThreadPool();
        service.execute(new InnerRunnable());
        service.shutdownNow();
        //中断单个任务
        ExecutorService service1 = Executors.newCachedThreadPool();
        Future<?> future = service1.submit(new InnerRunnable());
        future.cancel(true);
        //用来关闭
        service1.shutdown();
    }
}
```



## 21.5 线程之间的协作

### 1.示例题：交替打印one和two？

```java
public class PrintOneAndTwo {
    private boolean isOne = true;
    private boolean isTwo = false;

    public synchronized void waitForPrintingOne() throws InterruptedException {
        while (isTwo && !isOne) {
            wait();
        }
    }

    public synchronized void printOne() {
        System.out.println("one");
        isOne = false;
        isTwo = true;
        notifyAll();
    }

    public synchronized void waitForPrintingTwo() throws InterruptedException {
        while (isOne && !isTwo) {
            wait();
        }
    }

    public synchronized void printTwo() {
        System.out.println("two");
        isOne = true;
        isTwo = false;
        notifyAll();
    }
}

public class OneAndTwo {
    public static void main(String[] args) throws InterruptedException {
        PrintOneAndTwo printOneAndTwo = new PrintOneAndTwo();
        Runnable runnableOne = () -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    printOneAndTwo.waitForPrintingOne();
                    printOneAndTwo.printOne();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
        Runnable runnableTwo = () -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    printOneAndTwo.waitForPrintingTwo();
                    printOneAndTwo.printTwo();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(runnableOne);
        executorService.execute(runnableTwo);
        TimeUnit.MILLISECONDS.sleep(2000);
        //终止所有线程
        executorService.shutdownNow();
        //关闭
        executorService.shutdown();
    }
}
```

### 3.错失的信号

```java
//有缺陷的写法
while (someCondition) {
	synchronized (shareMonitor) {
		shareMonitor.wait();
	}
}
//正确的写法
synchronized (shareMonitor) {
	while (someCondition) {
		shareMonitor.wait();
	}
}
```

### 4.notify()和notifyAll()

1. 使用notify()而不是notifyAll()是一种优化；使用notify()时，在众多等待同一个锁的任务中只有一个会被唤醒
2. 如果希望使用notify()，就必须保证被唤醒的是恰当的任务，所有任务必须等待相同的条件
3. 当notifyAll()因某个特定锁而被调用时，只有等待这个锁的任务才会被唤醒



## 21.6 死锁

### 1.死锁的定义

多个进程由于竞争资源而造成的相互等待的现象叫做死锁。

### 2.产生死锁的必要条件？

1. 互斥条件
2. 请求和保持条件
3. 不剥夺条件
4. 环路等待条件

### 3.多线程的主要缺陷有哪些？

1. 等待共享资源的时候性能降低
2. 需要处理线程的额外CPU花费
3. 糟糕的程序设计导致不必要的复杂度
4. 有可能产生一些病态行为，如饿死，竞争，死锁和活锁（多个运行各自任务的线程使得整体无法完成）
5. 不同平台导致的不一致性