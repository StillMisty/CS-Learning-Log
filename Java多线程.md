# Java 多线程

## 1. 前置知识

- 并行：程序在同一时间执行多个任务，互不影响。
- 并发：程序在同一时间段内执行多个任务，可能会互相影响。

- 串行：程序按顺序执行一个任务，完成后再执行下一个任务。
- 并行：程序同时执行多个任务。

- 线程：程序执行的最小单位，包含一个调用栈和程序计数器，同一进程的线程之间共享内存空间，线程崩溃可能会导致其所属进程崩溃。
- 进程：操作系统分配资源的基本单位，包含一个或多个线程，进程之间相互独立，开销更大，隔离性更强。

## 2. 多线程用途

- 提高程序的响应速度：在等待某些操作（如网络请求、文件读写）时，可以继续执行其他任务。
- 实现并行处理：可以将任务分解成多个子任务，利用多核 CPU 的优势，提高程序的执行效率。

## 3. 线程创建

### 3.1. 继承 `Thread` 类

    ``` java
    class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("Thread is running");
        }
    }
    ```

### 3.2. 实现 `Runnable` 接口

    ``` java
    class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("Runnable is running");
        }
    }
    ```

### 3.3. 使用 `Callable` 接口

// TODO

### 3.4. 使用线程池

// TODO

## 4. 启动线程

1. 调用 Thread 对象的 start() 方法。

        ``` java
        public class ThreadDemo {
            public static void main(String[] args) {
                MyThread thread = new Thread(() -> System.out.println("Task is running in thread pool"));
                thread.start(); // 启动线程
            }
        }
        ```

2. 把任务提交给线程池，由线程池内部来管理并启动线程。虽然我们调用的是 execute()，但其底层机制仍然是线程池为我们调用了工作线程的 start() 方法。

        ``` java
        public class ThreadPoolDemo {
            public static void main(String[] args) {
                ExecutorService executor = Executors.newFixedThreadPool(2);
                executor.execute(() -> System.out.println("Task is running in thread pool"));
                executor.shutdown(); // 关闭线程池，其会等待所有任务完成后再关闭
            }
        }
        ```

## 5. 线程状态

Java 中线程的状态有以下几种：

- 新建（New）：线程被创建但尚未启动。
- 可运行（Runnable）：线程可以运行，等待 OS 调度。
- 阻塞 (Blocked)：线程被阻塞，等待获取锁。
- 等待 (Waiting)：无限期等待，需被其他线程显式唤醒。
- 计时等待 (TIMED_WAITING)：在指定时间内等待，可被唤醒或自动超时。
- 终止 (Terminated)：线程的 run() 方法执行完毕或抛出异常。

## 6. 线程同步

在多线程环境中，如果没有同步机制，当多个线程同时读写同一个共享变量时，可能会发生以下问题，统称为线程安全问题：

- 原子性（Atomicity）：一个或多个操作，要么全部执行成功，要么全部不执行，中间不能被任何其他线程中断。像 count++ 这样的操作就不是原子的，它包含“读取-修改-写入”三个步骤，任何一步都可能被中断。
- 可见性（Visibility）：当一个线程修改了共享变量的值，其他线程能够立即看到这个修改。由于CPU缓存、指令重排等原因，一个线程的修改可能不会立即对其他线程可见。
- 有序性（Ordering）：程序执行的顺序按照代码的先后顺序执行。编译器和处理器为了优化性能，可能会对指令进行重排序，这在单线程中没问题，但在多线程中可能导致意想不到的后果。

### 6.1. synchronized 关键字

这是Java中最基本、最常用的同步机制。它是一种悲观锁，总认为会有并发问题，所以每次操作前都会加锁。

可用来修饰：

- 实例方法：锁定当前对象实例。
  
        ```java
        public synchronized void method() {
            // 同步代码块
        }
        ```

- 静态方法：锁定当前类的 Class 对象。
  
        ```java
          public static synchronized void staticMethod() {
              // 同步代码块
          }
        ```

- 同步代码块：锁定指定对象。

        ```java
        public void method() {
            // 可使用任意对象作为锁
            Object lock = new getObject();
            synchronized (lock) {
                // 同步代码块
            }
        }
        ```

特点：

- 隐式锁：synchronized 是隐式的，锁的获取和释放由 JVM 自动管理。
- 可重入：同一线程可以多次获取同一把锁，而不会导致死锁。
- 不可中断：如果一个线程获取锁失败，它会被阻塞，不会被中断，直到获取到锁为止。
- 保证可见性与原子性：synchronized 会强制刷新工作内存到主内存，确保其他线程能看到最新的值。

### 6.2. volatile

一个轻量级的同步机制，它不能保证原子性，但能保证可见性和一定程度的有序性。

- 保证可见性：当一个线程修改了 volatile 变量，这个修改会立即被刷新到主内存，其他线程读取时会直接从主内存读取，从而保证能看到最新的值。
- 禁止指令重排：在 volatile 变量的读写操作前后插入内存屏障，防止编译器和CPU的过度优化。

适用场景：非常适合“一写多读”的场景，或者作为状态标志位。

    ``` java
    private volatile boolean running = true;

    public void stop() {
        running = false; // 一个线程写
    }

    public void run() {
        while (running) { // 多个线程读
            // ...
        }
    }
    ```

注意：volatile 不适用于 count++ 这样的复合操作，，因为它不能保证原子性，可使用 `synchronized` 或是 `AtomicInteger` 来处理。

### 6.3. Lock 接口

Lock 是一个比 synchronized 更强大、更灵活的锁机制。ReentrantLock 是 Lock 接口最常见的实现。

与 synchronized 的主要区别：

- 手动获取和释放锁：必须在 finally 块中调用 unlock() 来释放锁，否则可能导致死锁。
- 可中断获取锁：lock.lockInterruptibly() 允许在等待锁的过程中响应中断。
- 可尝试获取锁：lock.tryLock() 尝试获取锁，如果锁不可用，会立即返回 false 或在指定时间内等待。
- 公平性选择：可以创建公平锁（等待时间最长的线程优先获取锁）或非公平锁（默认，性能更高）。

        ``` java
        class Counter {
            private final Lock lock = new ReentrantLock(); // 创建锁
            private volatile int count;

            public void increment() {
                lock.lock(); // 获取锁
                try {
                    count++;
                } finally {
                    lock.unlock(); // 必须在finally中释放锁
                }
            }
        }
        ```

### 6.4. 原子类

原子类提供了一种**无锁**（Lock-Free）的线程安全实现方式。它们底层利用了CAS（Compare-And-Swap，比较并交换）这种乐观锁机制。

CAS原理：操作包含三个操作数——内存位置V、预期原值A和新值B。如果内存位置V的值与预期原值A相匹配，那么处理器会自动将该位置的值更新为新值B。否则，处理器不做任何操作。这是一个原子的硬件指令。

常用的原子类有：

- AtomicInteger, AtomicLong, AtomicBoolean
- AtomicReference (用于原子性地更新对象引用)
- AtomicIntegerArray, AtomicLongArray, AtomicReferenceArray (用于原子性地更新数组元素)

        ``` java
        class AtomicCounter {
            private AtomicInteger count = new AtomicInteger(0);

            public void increment() {
                count.incrementAndGet(); // 原子操作，等同于 synchronized(this){ count++; }，但底层使       用 CAS 的无锁技术，性能更高
            }

            public int getCount() {
                return count.get();
            }
        }
        ```

## 7. 高级并发工具

### 3.1 Semaphore (信号量)

Semaphore 控制同时访问特定资源的线程数量，用于限流和资源池管理。它维护了一个许可的数量，线程在访问共享资源前必须获取许可，使用完后释放许可。

主要方法：

- Semaphore(int permits): 构造函数，初始化许可证的数量。
- void acquire(): 获取一个许可证。如果当前没有可用的许可证，线程将被阻塞，直到有许可证被释放。
- void release(): 释放一个许可证，将其还给信号量。
- boolean tryAcquire(): 尝试获取一个许可证，如果成功返回 true，失败立即返回 false，不会阻塞。
- int availablePermits(): 返回当前可用的许可证数量。

        ``` java
        public class SemaphoreDemo {
            public static void main(String[] args) {
                // 创建一个有3个许可证的信号量，代表3台机器
                Semaphore semaphore = new Semaphore(3);

                // 8个工人
                for (int i = 1; i <= 8; i++) {
                    new Thread(() -> {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 准备使用机器...");
                            semaphore.acquire(); // 获取许可证
                            System.out.println(Thread.currentThread().getName() + " 获得机器，开始工作...");
                            TimeUnit.SECONDS.sleep(2); // 模拟工作
                            System.out.println(Thread.currentThread().getName() + " 工作完毕，归还机器。");
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } finally {
                            semaphore.release(); // 释放许可证
                        }
                    }, "工人-" + i).start();
                }
            }
        }
        ```

**输出分析：**
你会看到，任何时候都只有最多3个线程会打印“获得机器，开始工作...”。其他线程则会等待，直到有线程工作完毕并调用 release()。

应用场景

- 数据库连接池：池中只有固定数量的连接，线程需要先从池中获取连接（acquire），用完后归还（release）。
- 服务限流：限制某个API或服务的并发调用量，防止系统过载。

### 3.2 CountDownLatch (倒计时器)

让一个或多个线程等待其他一组线程完成操作。它是一个一次性的工具，计数器到0后无法重置。

**主要方法：**

- CountDownLatch(int count): 构造函数，设置初始计数值。
- void await(): 使当前线程阻塞，直到计数器变为0。
- boolean await(long timeout, TimeUnit unit): 带超时的等待。
- void countDown(): 将计数器减1。

        ``` java
        public class CountDownLatchDemo {
            public static void main(String[] args) throws InterruptedException {
                // 创建一个计数值为3的CountDownLatch
                CountDownLatch latch = new CountDownLatch(3);

                // 创建3个子任务线程
                new Thread(() -> {
                    System.out.println("子任务1开始执行...");
                    try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) {}
                    System.out.println("子任务1执行完毕。");
                    latch.countDown(); // 计数器减1
                }).start();

                new Thread(() -> {
                    System.out.println("子任务2开始执行...");
                    try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) {}
                    System.out.println("子任务2执行完毕。");
                    latch.countDown(); // 计数器减1
                }).start();

                new Thread(() -> {
                    System.out.println("子任务3开始执行...");
                    try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) {}
                    System.out.println("子任务3执行完毕。");
                    latch.countDown(); // 计数器减1
                }).start();

                System.out.println("主线程等待所有子任务完成...");
                latch.await(); // 阻塞，直到计数器为0
                System.out.println("所有子任务均已完成，主线程继续执行。");
            }
        }
        ```

输出分析：主线程会打印“等待...”，并一直阻塞，直到3个子任务都调用了countDown()，然后才会打印“所有子任务均已完成...”。

**应用场景：**

- 并行计算：一个主任务分解为多个子任务，主任务等待所有子任务计算结果，然后汇总。
- 应用启动：主线程需要等待多个外部资源（数据库、缓存、配置服务）加载完毕后才能继续。

### 3.3 CyclicBarrier (循环屏障)

让一组线程互相等待，直到所有线程都到达一个公共的屏障点，然后再同时继续执行。它是可循环使用的。

**主要方法：**

- CyclicBarrier(int parties): 构造函数，指定需要等待的线程数。
- CyclicBarrier(int parties, Runnable barrierAction): 构造函数，额外指定一个当屏障被打破时，由最后一个到达的线程优先执行的任务。
- int await(): 线程调用此方法表示已到达屏障，并开始等待。

        ``` java
        public class CyclicBarrierDemo {
            public static void main(String[] args) {
                // 创建一个需要5个线程到达的屏障，并在屏障打破时执行一个任务
                CyclicBarrier barrier = new CyclicBarrier(5, () -> {
                    System.out.println("--- 所有玩家已准备就绪，游戏开始！ ---");
                });

                for (int i = 1; i <= 5; i++) {
                    new Thread(() -> {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 已准备，等待其他玩家...");
                            barrier.await(); // 等待其他线程
                            System.out.println(Thread.currentThread().getName() + " 开始游戏！");
                        } catch (InterruptedException | BrokenBarrierException e) {
                            e.printStackTrace();
                        }
                    }, "玩家-" + i).start();
                }
            }
        }
        ```

**输出分析：**
5个玩家会陆续打印“已准备...”，然后等待。当第5个玩家也准备好后，会先打印“游戏开始！”（barrierAction），然后5个玩家会几乎同时打印“开始游戏！”。

### 3.4 CountDownLatch 与 CyclicBarrier 的关键区别

| 特性     | CountDownLatch             | CyclicBarrier              |
| -------- | -------------------------- | -------------------------- |
| 作用对象 | 一个或多个线程等待其他线程 | 一组线程互相等待           |
| 可重用性 | 不可重用，计数器到0后失效  | 可重用，屏障打破后自动重置 |
| 功能     | countDown() 和 await()     | await() 兼具计数和等待功能 |

### 3.5 读写锁

将读和写操作分离开来，实现“读-读”并发，提高在读多写少场景下的性能。

**主要方法**
ReadWriteLock 是一个接口，常用实现是 ReentrantReadWriteLock。

- Lock readLock(): 获取读锁。
- Lock writeLock(): 获取写锁。

**应用场景：**

- 缓存系统：缓存的读取频率远高于写入频率。
- 配置文件读取：配置加载后很少变动，但会被多个线程频繁读取。
- 共享数据容器：任何读多写少的共享数据结构。

### 3.6 Future 和 CompletableFuture
