# Kotlin协程

## 1. 协程的概念

协程可被认为是轻量级线程，它是一种并发设计模式，可以简化异步编程。协程是一种更高级的抽象，它可以在代码中表现为一个线程、一个任务、一个函数或者一个代码块。协程可以在一个线程中并发执行，也可以在多个线程中并发执行。

## 2. 导入协程库

在Kotlin中使用协程需要导入kotlinx-coroutines-core库。
请到 <https://github.com/Kotlin/kotlinx.coroutines> 查看版本信息。

- 在Gradle中添加依赖：

```groovy
dependencies {
    val coroutinesVersion = "1.8.0"
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")
}
```

- 在Maven中添加依赖：

```xml
<dependency>
    <groupId>org.jetbrains.kotlinx</groupId>
    <artifactId>kotlinx-coroutines-core</artifactId>
    <version>1.8.0</version>
</dependency>
```

## 3. 协程的创建

1. launch函数创建协程，launch函数会返回一个Job对象。用于无返回值的异步任务。
2. async函数创建协程，async函数会返回一个Deferred对象。用于有返回值的异步任务，使用其await方法获取返回值。
3. 可用runBlocking函数创建一个协程，runBlocking函数会阻塞当前线程，直到其和其内所有的协程执行完毕。用于桥接常规程序的非协程世界和带有协程的代码。

```kotlin
fun main() = runBlocking {
    // this: CoroutineScope
    launch {
        // 创建一个新的协程并继续
        delay(1000L)
        // 非阻塞的延迟1秒
        print("World!")
        // 延迟后打印
    }
    print("Hello ")
}

// 输出：Hello World!
```

## 4. 协程的启动与取消

1. 协程自动启动，可设置start参数为CoroutineStart.LAZY，此时协程不会自动启动，需要手动调用start函数启动。
2. 并不是创建了协程就会立即执行，协程的执行是异步的，协程的执行是由调度器控制的。
3. 当协程创建后，会被添加到调度器的队列中，调度器会根据其优先级和调度策略来决定协程的执行顺序，可以通过CoroutineContext来指定协程的调度器。
4. 调度器可以指定协程运行在哪个线程上，各个调度器有有不同的优势eg：Dispatchers.Default、Dispatchers.IO、Dispatchers.Main等。
5. Job和Deferred的join函数会阻塞当前线程，直到其对应的协程执行完毕。
6. 可以使用joinAll函数等待多个协程执行完毕。
7. Job和Deferred的cancel函数会取消对应的协程。
8. withTimeout函数会在指定时间内执行协程，超时后会抛出TimeoutCancellationException异常。

```kotlin
val deferred = async(
    start = CoroutineStart.LAZY,
    context = Dispatchers.IO
    ) {
    9 * 9 * 9
}
withTimeout(1000) {
    println(deferred.await())
}

// 输出：729
```

## 5. 协程的挂起

1. 协程的挂起可以使用suspend关键字修饰函数，suspend关键字修饰的函数只能在协程中调用。
2. 协程的挂起可以使用withContext函数，withContext函数会切换协程的调度器，使协程在不同的线程中执行。其可更改协程的上下文，但将其同时保留在同一协程中运行。

```kotlin
// 挂起函数
suspend fun doSomething() {
    delay(1000)
    println("do something")
}

// WithContext
launch(Dispatchers.Main) {
    val data = withContext(Dispatchers.IO) {
        // 在IO线程中执行耗时操作
        loadDataFromNetwork()
    }
    // 在主线程中更新UI
    updateUI(data)
}
```

## 6. 协程中的流Flow

1. Flow是一种冷流，只有在收集时才会开始执行，Flow的收集是懒加载的。
2. Flow的创建和操作符的调用并不需要挂起函数。
3. Flow的创建可以使用flowOf、asFlow、channelFlow等函数。
4. Flow的收集可以使用collect函数，collect函数是一个挂起函数，只能在协程中调用。因为收集Flow可能涉及到耗时的操作，例如网络请求或者磁盘读写，所以需要挂起以防止阻塞主线程。
5. Flow构建器中的代码必须遵守上下文保留属性，并且不允许从不同的上下文发出（WithContent）。
6. Flow的操作符有map、filter、transform、take、zip等。

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}
fun main() = runBlocking {
    simple().collect { value -> println(value) }
}

// 输出：1 2 3
```

## 7. 协程的渠道Channel

1. Channel是一种协程间通信的方式，Channel是一种热流，Channel的发送和接收是同步的。
2. Channel的创建可以使用Channel函数，Channel函数可以指定Channel的缓冲区大小。
3. Channel的发送和接收可以使用send和receive函数，send和receive函数是挂起函数，只能在协程中调用。
4. Channel的关闭可以使用close函数，close函数会关闭Channel，关闭后无法再发送数据，但可以继续接收数据。

```kotlin
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) {
            delay(1000)
            channel.send(x)
        }
    }
    repeat(5) { println(channel.receive()) }
}
// 你会看到每隔1秒输出一个数字
```

## 8. 协程的异常处理

1. 协程的异常处理可以使用try-catch语句，try-catch语句可以捕获协程中的异常。
2. 协程的异常处理可以使用CoroutineExceptionHandler，CoroutineExceptionHandler可以捕获协程中的异常，并且可以指定处理异常的策略。

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}
fun main() = runBlocking {
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    job.join()
}

// 输出：Caught java.lang.AssertionError
```

## 9. 总结

1. 协程是一种轻量级线程，可以简化异步编程。
2. 协程的创建可以使用launch、async、runBlocking函数。
3. 协程的启动与取消可以使用start、join、cancel函数。
4. 协程的挂起可以使用suspend关键字修饰函数，也可以使用withContext函数。
5. 协程中的流Flow可以使用Flow创建，也可以使用collect函数收集。
6. 协程中的渠道Channel可以使用Channel创建，也可以使用send和receive函数发送和接收数据。
7. 协程的异常处理可以使用try-catch语句，也可以使用CoroutineExceptionHandler。
8. 协程的调度器可以使用Dispatchers指定，也可以使用withContext函数切换。
