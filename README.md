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

