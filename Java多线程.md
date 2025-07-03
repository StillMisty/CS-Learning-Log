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

## 3. 线程状态

Java 中线程的状态有以下几种：

- 新建（New）：线程被创建但尚未启动。
- 可运行（Runnable）：线程可以运行，等待 OS 调度。
- 阻塞 (Blocked)：线程被阻塞，等待获取锁。
- 等待 (Waiting)：无限期等待，需被其他线程显式唤醒。
- 计时等待 (TIMED_WAITING)：在指定时间内等待，可被唤醒或自动超时。
- 终止 (Terminated)：线程的 run() 方法执行完毕或抛出异常。

## 4. 线程创建

### 4.1. 继承 `Thread` 类

``` java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread is running");
    }
}
```

### 4.2. 实现 `Runnable` 接口

``` java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable is running");
    }
}
```

### 4.3. 实现 `Callable` 接口

Callable 是对 Runnable 的增强，解决了 Runnable 不能有返回值和不能抛出异常的痛点。

``` java
class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        System.out.println("通过实现 Callable 接口创建的线程正在执行...");
        Thread.sleep(2000);
        return "任务执行完毕，这是返回值";
    }
}
```

## 5. 获取异步函数结果

### 5.1 Future

Future 是 Java 5 引入的，位于 java.util.concurrent 包中。它代表了一个异步计算的结果。

**Future 的核心方法：**

- V get(): **阻塞式地等待**，直到计算完成，然后返回结果。
- V get(long timeout, TimeUnit unit): 带超时的阻塞等待。
- boolean isDone(): 查询计算是否已完成。
- boolean cancel(boolean mayInterruptIfRunning): 尝试取消任务的执行。

**Future 的主要问题（局限性）：**

- **阻塞式获取结果**: future.get() 会阻塞当前线程。这使得异步的优势大打折扣。你本想让主线程不被卡住，但为了获取结果，最终还是得停下来等待。
- **无法建立回调**: 你无法告诉 Future：“等你计算完成后，请自动执行这个后续任务”。你只能通过 isDone() 轮询或者直接 get() 阻塞，这非常笨拙。
- **无法链式操作**: 多个异步任务之间如果存在依赖关系（例如，任务B需要任务A的结果），Future 无法优雅地将它们串联起来。你只能在一个任务的 get() 之后，再启动下一个任务，这又退化成了同步阻塞模式。
- **无法手动完成**: 你不能自己创建一个 Future 并手动设置它的结果或异常。它必须依赖于 ExecutorService。
- **没有统一的异常处理**: 你必须在调用 get() 的地方用 try-catch 来捕获异常，无法在异步任务链中统一处理。

``` java
public class FutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newSingleThreadExecutor()
        // 提交一个任务，得到一个Future
        Future<String> future = executor.submit(() -> {
            System.out.println("子线程开始计算...");
            Thread.sleep(2000);
            return "这是计算结果";
        })
        System.out.println("主线程做其他事情...")
        // 为了获取结果，主线程不得不阻塞在这里
        System.out.println("等待获取结果...");
        String result = future.get(); // Blocking here!
        System.out.println("获取到的结果是: " + result)
        executor.shutdown();
    }
}
```

### 5.2 CompletableFuture

CompletableFuture 是 Java 8 引入的，它彻底解决了 Future 的所有痛点。它不仅是一个 Future（实现了 Future 接口），更重要的是它实现了 CompletionStage 接口，这赋予了它编排和链接异步任务的能力。

1. **非阻塞和回调机制**: 核心是 thenApply, thenAccept, thenRun 等方法。它们允许你注册一个回调函数，当异步任务完成后，这个函数会被自动调用，完全无需 get() 阻塞。
2. **强大的链式调用（编排能力）**: 你可以像Java Stream API一样，将多个异步操作流畅地串联起来，形成一个处理流水线。
    - thenApply(Function fn): 接收上一步结果，处理后返回一个新结果。
    - thenAccept(Consumer action): 接收上一步结果，进行消费，但无返回值。
    - thenRun(Runnable action): 不关心上一步结果，只要上一步完成了，就执行指定的动作。
    - 组合多个CompletableFuture:
    - thenCompose(Function fn): 用于串联两个有依赖关系的 CompletableFuture。
    - thenCombine(CompletionStage other, BiFunction fn): 合并两个独立的 CompletableFuture 的结果。
    - allOf(CompletableFuture<?>... cfs): 等待所有给定的 CompletableFuture 都完成。
    - anyOf(CompletableFuture<?>... cfs): 等待任何一个给定的 CompletableFuture 完成。
3. **优雅的异常处理**:
    - exceptionally(Function fn): 当异步链中任何一环出现异常时，可以捕获并提供一个默认值或进行处理。
    - handle(BiFunction fn): 无论成功还是失败都会执行，可以同时拿到结果和异常。
4. **手动完成**: 可以 new CompletableFuture() 并通过 complete(value) 或 completeExceptionally(ex) 手动完成它，方便与非CompletableFuture的异步API集成。

``` java
public class CompletableFutureDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("主线程开始...");

        // 1. 启动一个异步任务
        CompletableFuture.supplyAsync(() -> {
            try {
                System.out.println("阶段1：查询用户信息...");
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "用户A";
        }).thenApply(user -> { // 2. 拿到用户信息后，继续异步查询其订单
            System.out.println("阶段2：基于用户 " + user + " 查询订单...");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 模拟一个异常
            if (Math.random() > 0.5) {
                throw new RuntimeException("查询订单时网络超时！");
            }
            return user + " 的订单列表";
        }).thenAccept(orders -> { // 3. 拿到订单后，进行消费（打印）
            System.out.println("阶段3：成功获取到: " + orders);
        }).exceptionally(e -> { // 4. 异常处理
            System.err.println("!!! 出现异常: " + e.getMessage());
            return null; // 返回一个null，让链条继续
        });

        System.out.println("主线程继续做其他事情，完全不阻塞...");
        // 等待异步任务完成，否则main线程直接退出，看不到异步结果
        TimeUnit.SECONDS.sleep(5); 
    }
}
```

### 5.3. Future 与 CompletableFuture 的对比

| 特性       | Future               | CompletableFuture                               |
| ---------- | -------------------- | ----------------------------------------------- |
| 获取结果   | 阻塞式 (get())       | 非阻塞，通过回调 (thenApply, thenAccept 等)     |
| 任务编排   | 不支持，任务间孤立   | 核心优势，支持链式调用 (thenApply, thenCompose) |
| 多任务组合 | 不支持               | 支持 (allOf, anyOf, thenCombine)                |
| 异常处理   | 必须 try-catch get() | 优雅的链式处理 (exceptionally, handle)          |
| 手动完成   | 不支持               | 支持 (complete, completeExceptionally)          |
| 本质       | 只是一个结果的容器   | 一个强大的异步任务编排工具                      |

1. **Future 已经基本被视为一个“遗留”接口**。在新的代码中，除非你对接的API只返回Future，否则几乎没有理由再直接使用它。
2. **在所有新的Java异步编程场景中，都应该优先且默认使用 CompletableFuture**。它提供了声明式、函数式、非阻塞的编程模型，能让你以非常优雅和高效的方式构建复杂的异步系统。

## 6. 线程池

### 6.1. 为什么需要线程池

1. **资源消耗巨大：**
    - **创建/销毁开销**：线程的创建和销毁涉及到与操作系统的交互，需要分配和回收内存，这是非常耗费系统资源和时间的操作。
    - **内存占用**：每个线程都需要占用一定的内存（JVM栈、程序计数器等）。如果无限制地创建线程，可能很快耗尽系统内存，导致 OutOfMemoryError。
2. **管理困难，系统不稳定：**
    - **无限制创建**：大量的线程会抢占CPU资源，导致频繁的上下文切换（Context Switching），这本身就是一种巨大的性能开销。CPU大部分时间可能都花在切换线程上，而不是真正执行任务。
    - **缺乏统一管理**：无法方便地控制并发线程的数量、监控线程状态、或者统一处理异常。

线程池将线程的创建和管理与任务的执行分离开来。预先创建好一定数量的线程放在池子里，当有任务来时，就从池中取一个空闲线程去执行，任务执行完毕后，线程并不销毁，而是归还到池中，等待下一个任务。

### 6.2. 线程池的优势

1. **降低资源消耗**：通过复用已创建的线程，显著减少了线程创建和销毁带来的开销。
2. **提高响应速度**：任务到达时，无需等待线程创建即可立即执行，因为池中已经有现成的线程。
3. **提高线程的可管理性**：线程是稀缺资源，线程池可以对线程进行统一分配、调优和监控，防止无限制地消耗资源。

### 6.3. 线程池的核心工作原理

Java线程池的核心实现是 ThreadPoolExecutor。

**一个绝佳的比喻：银行办理业务**
想象你去一家银行，这家银行的运作模式就是线程池的完美写照：

- **corePoolSize** (核心柜员数)：银行始终保持开启的柜台数量。即使没人，这几个柜台也开着。
- **workQueue** (等候区)：办理业务的人坐的椅子。
- **maximumPoolSize** (最大柜员数)：银行能开启的全部柜台数量（包括临时加开的）。
- **RejectedExecutionHandler** (拒绝策略)：当等候区也满了，临时柜台也全开了，大堂经理（Handler）出来告诉你：“今天太忙了，办不了了，您改天再来吧”。

**工作流程如下：**

1. 来了一个新任务（客户）：
    - 线程池判断核心线程 (corePoolSize) 是否已满？
    - 没满：创建一个新的核心线程来处理这个任务。
    - 满了：进入下一步。
2. 核心线程已满，判断工作队列 (workQueue) 是否已满？
    - 没满：将新任务放入工作队列中等待。
    - 满了：进入下一步。
3. 工作队列也满了，判断最大线程数 (maximumPoolSize) 是否已满？
    - 没满：创建一个非核心线程（临时工）来处理这个任务。
    - 满了：进入下一步。
4. 所有线程都在忙，队列也满了：
    - 触发拒绝策略 (RejectedExecutionHandler)。

**关于线程销毁：**

- **核心线程** (corePoolSize) 默认情况下会一直存活。
- **非核心线程**（临时工）在空闲时间超过 keepAliveTime 后，会被销毁。

### 6.4. 创建线程池

#### 6.4.1 通过 Executors 工厂类 (不推荐在生产环境使用)

Java提供了 Executors 工具类来快速创建几种常见的线程池，这对于学习和简单场景很方便。

- **Executors.newFixedThreadPool(int n)**：创建固定大小的线程池。corePoolSize 和 maximumPoolSize 相等。使用无界队列 LinkedBlockingQueue。
  - 风险：队列是无界的，如果任务堆积过多，可能导致内存溢出 (OutOfMemoryError)。
- **Executors.newSingleThreadExecutor()**：创建只有一个线程的线程池。同样使用无界队列。
  - 风险：与上面相同，且能保证任务按顺序执行。
- **是Executors.newCachedThreadPool()**：创建可缓存的线程池。corePoolSize 为0，maximumPoolSize 为 Integer.MAX_VALUE。
  - 风险：maximumPoolSize 几乎是无限的，如果任务瞬间并发量很高，会创建大量线程，可能耗尽系统资源。

**《阿里巴巴Java开发手册》强制规定：不允许使用 Executors 创建线程池，而是通过 ThreadPoolExecutor 的方式。**

#### 6.4.2. 通过 ThreadPoolExecutor 构造函数 (推荐)

``` java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) { ... }
```

**参数详解：**

1. **corePoolSize**：核心线程数。线程池维持的最小线程数。
2. **maximumPoolSize**：最大线程数。线程池允许创建的最大线程数。
3. **keepAliveTime**：空闲线程存活时间。当线程数大于 corePoolSize 时，多余的空闲线程在被销毁前等待新任务的最长时间。
4. **unit**：keepAliveTime 的时间单位。
5. **workQueue**：任务队列。用于存放等待执行的任务，是 BlockingQueue 类型。
    - **ArrayBlockingQueue**：有界队列，基于数组，先进先出。创建时必须指定容量。
    - **LinkedBlockingQueue**：有界/无界队列，基于链表。不指定容量时，默认容量是 Integer.MAX_VALUE，相当于无界。
    - **SynchronousQueue**：不存储元素的队列。每个插入操作必须等待一个移除操作，反之亦然。通常配合 newCachedThreadPool 使用。
6. **threadFactory**：线程工厂。用于创建新线程，可以自定义线程名称、是否为守护线程等。给线程起一个有意义的名字对排查问题非常有帮助！
7. **handler**：拒绝策略。当线程池和队列都满了之后，如何处理新提交的任务。

**拒绝策略：**

- **AbortPolicy** (默认)：直接抛出 RejectedExecutionException 异常，阻止系统正常运行。
- **CallerRunsPolicy**："调用者运行"策略。既不抛弃任务，也不抛出异常，而是将任务回退给调用者（提交任务的那个线程）去执行。这是一种很好的反压机制，可以减慢任务提交的速度。
- **DiscardPolicy**：直接丢弃任务，不予处理，也不抛出异常。如果允许任务丢失，这是个好选择。
- **DiscardOldestPolicy**：丢弃队列最前面的任务，然后重新提交被拒绝的任务。

### 6.5. 向线程池提交任务

- **void execute(Runnable command)**:
  - 提交一个不需要返回值的任务。
  - 任务中的异常会直接抛出，导致该线程终止。
- **Future<?> submit(Runnable/Callable task)**:
  - 提交任务，并返回一个 Future 对象，代表任务的未来结果。
  - 通过 future.get()可以阻塞式地获取任务的执行结果（Callable）或等待任务完成（Runnable）。
  - 任务中的异常会被捕获并封装在 ExecutionException 中，在调用 future.get() 时抛出。这是处理任务异常的推荐方式。

### 6.6. 关闭线程池

关闭线程池非常重要，否则程序可能无法正常退出。

- **shutdown()**:
  - **平滑关闭。**
  - 不再接受新任务，但会处理完队列中已有的任务。
  - 调用后，线程池状态变为 SHUTDOWN。
- **shutdownNow()**:
  - **立即关闭。**
  - 尝试中断正在执行的线程（通过 Thread.interrupt()）。
  - 清空任务队列，并返回未执行的任务列表。
  - 调用后，线程池状态变为 STOP。

最佳实践是结合 awaitTermination() 使用，给线程池一个优雅的关闭过程：

``` java
pool.shutdown(); // 不再接受新任务
try {
    // 等待60秒，让现有任务执行完毕
    if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
        pool.shutdownNow(); // 如果超时还未关闭，则强制关闭
        if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
            System.err.println("Pool did not terminate");
        }
    }
} catch (InterruptedException ie) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}
```

### 6.7. 如何配置线程池

没有固定答案，但有基本原则：

1. **任务类型：**
    - CPU密集型任务 (如：复杂计算、加解密)：线程数应设置为 CPU核心数 + 1。过多的线程只会增加上下文切换的开销。
    - I/O密集型任务 (如：数据库查询、网络请求、文件读写)：线程大部分时间在等待I/O操作完成。可以配置更多的线程，如 CPU核心数 \* 2 或根据压测结果调整。一个经典公式是：线程数 = CPU核心数 \* (1 + 平均等待时间 / 平均计算时间)。
2. **队列选择：**
    - 优先使用有界队列 (ArrayBlockingQueue)，能有效防止资源耗尽。队列大小需要根据任务的并发量和处理能力进行估算。
3. **拒绝策略：**
    - 根据业务需求选择。关键业务不希望丢失，可使用 CallerRunsPolicy 进行限流，或者将任务持久化后由其他系统处理。非关键业务可考虑 DiscardPolicy。

## 7. 启动线程

### 7.1 调用 Thread 对象的 start() 方法

``` java
public class ThreadDemo {
    public static void main(String[] args) {
        MyThread thread = new Thread(() -> System.out.println("Task is running in thread pool"));
        thread.start(); // 启动线程
    }
}
```

### 7.2 把任务提交给线程池，由线程池内部来管理并启动线程

虽然我们调用的是 execute()，但其底层机制仍然是线程池为我们调用了工作线程的 start() 方法。

``` java
public class ThreadPoolDemo {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.execute(() -> System.out.println("Task is running in thread pool"));
        executor.shutdown(); // 关闭线程池，其会等待所有任务完成后再关闭
    }
}
```

## 8. 线程同步

在多线程环境中，如果没有同步机制，当多个线程同时读写同一个共享变量时，可能会发生以下问题，统称为线程安全问题：

- 原子性（Atomicity）：一个或多个操作，要么全部执行成功，要么全部不执行，中间不能被任何其他线程中断。像 count++ 这样的操作就不是原子的，它包含“读取-修改-写入”三个步骤，任何一步都可能被中断。
- 可见性（Visibility）：当一个线程修改了共享变量的值，其他线程能够立即看到这个修改。由于CPU缓存、指令重排等原因，一个线程的修改可能不会立即对其他线程可见。
- 有序性（Ordering）：程序执行的顺序按照代码的先后顺序执行。编译器和处理器为了优化性能，可能会对指令进行重排序，这在单线程中没问题，但在多线程中可能导致意想不到的后果。

### 8.1. synchronized 关键字

这是Java中最基本、最常用的同步机制。它是一种悲观锁，总认为会有并发问题，所以每次操作前都会加锁。

可用来修饰：

- 实例方法：锁定当前对象实例。
  
``` java
public synchronized void method() {
    // 同步代码块
}
```

- 静态方法：锁定当前类的 Class 对象。

``` java
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

### 8.2. volatile 关键字

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

### 8.3. Lock 接口

Lock 是一个比 synchronized 更强大、更灵活的锁机制。ReentrantLock 是 Lock 接口最常见的实现。

与 synchronized 的主要区别：

- 手动获取和释放锁：必须在 finally 块中调用 unlock() 来释放锁，否则可能导致死锁。
- 可中断获取锁：lock.lockInterruptibly() 允许在等待锁的过程中响应中断。
- 可尝试获取锁：lock.tryLock() 尝试获取锁，如果锁不可用，会立即返回 false 或在指定时间内等待。
- 公平性选择：可以创建公平锁（等待时间最长的线程优先获取锁）或非公平锁（默认，性能更高）。

``` java
class Counter {
    private final Lock lock = new ReentrantLock(); // 创建锁
    private volatile int count
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

### 8.4. 原子类

原子类提供了一种**无锁**（Lock-Free）的线程安全实现方式。它们底层利用了CAS（Compare-And-Swap，比较并交换）这种乐观锁机制。

CAS原理：操作包含三个操作数——内存位置V、预期原值A和新值B。如果内存位置V的值与预期原值A相匹配，那么处理器会自动将该位置的值更新为新值B。否则，处理器不做任何操作。这是一个原子的硬件指令。

常用的原子类有：

- AtomicInteger, AtomicLong, AtomicBoolean
- AtomicReference (用于原子性地更新对象引用)
- AtomicIntegerArray, AtomicLongArray, AtomicReferenceArray (用于原子性地更新数组元素)

``` java
class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0)
    public void increment() {
        count.incrementAndGet(); // 原子操作，等同于 synchronized(this){ count++; }，但底层使       用 CAS 的无锁技术，性能更高
    
    public int getCount() {
        return count.get();
    }
}
```

## 9. 高级并发工具

### 9.1 Semaphore (信号量)

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
        Semaphore semaphore = new Semaphore(3)
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

### 9.2 CountDownLatch (倒计时器)

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
        CountDownLatch latch = new CountDownLatch(3)
        // 创建3个子任务线程
        new Thread(() -> {
            System.out.println("子任务1开始执行...");
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) {}
            System.out.println("子任务1执行完毕。");
            latch.countDown(); // 计数器减1
        }).start()
        new Thread(() -> {
            System.out.println("子任务2开始执行...");
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) {}
            System.out.println("子任务2执行完毕。");
            latch.countDown(); // 计数器减1
        }).start()
        new Thread(() -> {
            System.out.println("子任务3开始执行...");
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) {}
            System.out.println("子任务3执行完毕。");
            latch.countDown(); // 计数器减1
        }).start()
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

### 9.3 CyclicBarrier (循环屏障)

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
        })
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

### 9.4 CountDownLatch 与 CyclicBarrier 的关键区别

| 特性     | CountDownLatch             | CyclicBarrier              |
| -------- | -------------------------- | -------------------------- |
| 作用对象 | 一个或多个线程等待其他线程 | 一组线程互相等待           |
| 可重用性 | 不可重用，计数器到0后失效  | 可重用，屏障打破后自动重置 |
| 功能     | countDown() 和 await()     | await() 兼具计数和等待功能 |

### 9.5 读写锁

将读和写操作分离开来，实现“读-读”并发，提高在读多写少场景下的性能。

**主要方法**
ReadWriteLock 是一个接口，常用实现是 ReentrantReadWriteLock。

- Lock readLock(): 获取读锁。
- Lock writeLock(): 获取写锁。

**应用场景：**

- 缓存系统：缓存的读取频率远高于写入频率。
- 配置文件读取：配置加载后很少变动，但会被多个线程频繁读取。
- 共享数据容器：任何读多写少的共享数据结构。
