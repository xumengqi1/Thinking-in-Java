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
3. 实现Callable接口
4. 使用线程池创建

### 3.创建线程：继承Thread类



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
            //向线程调度器提出此时可以切换任务的建议
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

### 5.创建线程：实现Callable接口



### 6.创建线程：使用线程池创建



