# 线程创建的几种方式



## **1.继承Thread类**

最为简单直接的方式，直接通过继承`Thread`类，并重写`run()`方法。



**步骤：**

1. 创建一个类继承`Thread`。

2. 重写 `run()` 方法，将线程要执行的的代码写在`run`方法中。

3. 创建线程对象，通过`start()`方法启动线程。

**示例：**

```java
public class MyThread extends Thread{
    @Override
    public void run() {
        System.out.println("Thread is running");
    }
}

public class Main {
    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.start();
    }
}
```

**特点：**

- 简单直接，适合快速创建简单线程。

- 由于`Java`不支持多继承，如果已经继承了其他类，就不能使用这种方式。

  

## 2.实现Runnable接口

相比于继承`Thread`，实现`Runnable`接口更为灵活，可以避免`java`单继承的限制



**步骤：**

1. 创建一个类实现`Runnable`接口。

2. 实现 `run()` 方法，将线程要执行的的代码写在`run`方法中。

3. 创建`Thread`对象，并将`Runnable`对象传递给`Thread`的构造方法。

4. 调用`start()`方法启动线程。

**示例：**

```java
public class MyRunnable implements Runnable{

    @Override
    public void run() {
        System.out.println("Runnable is running");
    }
}

public class Main {
    public static void main(String[] args) {
        MyRunnable runnable = new MyRunnable();
        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```

**特点：**

- 更灵活，可以实现 `Runnable` 接口的同时继承其他类。

- 更加适合资源共享场景，可以将同一个 `Runnable` 实例传递给多个线程。

  

## 3.使用callable和Future

`Callable` 和 `Runnable` 类似，但 `Callable` 可以有返回值，并且可以抛出异常。需要配合 `Future` 或 `FutureTask` 使用。



**步骤：**

1. 创建一个类实现`Callable` 接口。
2. 实现`call()`方法 ,并定义返回值类型
3. 使用`ExecutorService`提交任务，获得`Future`对象
4. 通过`Future`的`get()`方法获得线程结果。

**示例：**

```java
public class MyCallable implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        return 123;
    }
}

public class Main {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> future = executor.submit(new MyCallable());
        try {
            System.out.println("callable返回值：" + future.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
        executor.shutdown();
    }
}
```

**特点：**

- 可以获取线程的执行结果。
- 支持抛出异常的处理。
- 更加适合复杂的线程操作，特别是需要返回结果时。



## 4.使用线程池（ExecutorService)

线程池管理一组线程，可以避免频繁创建和销毁线程的开销，提高性能。



**步骤：**

1. 创建一个线程池，例如使用 `Executors` 类。
2. 将任务提交给线程池执行。
3. 线程池可以管理多个线程的执行，并可复用现有线程。

**示例：**

```java
public class Main {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                System.out.println("Thread " + Thread.currentThread().getName() + " is running");
            });
        }
        executor.shutdown();
    }
}
```

**特点：**

- 提高了资源利用率，特别适合大量短小任务的执行。
- 可以控制线程的数量，避免过多线程导致的资源竞争。
