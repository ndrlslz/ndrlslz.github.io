title: Java concurrent包学习(一)
date: 2016-05-31 23:21:32
tags:
---


在JDK5之后，引入了java.util.concurrent包，增加了很多功能来更方便的处理并发编程。JDK8又加入异步处理，并发集合的增强等更强大的功能。在这里记录下学习过程。

---

## Executor

在JDK5之前，启动线程通常采用下面的方法。
```
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}).start();
```
Executors引入了异步处理和线程池机制，由于线程的创建和销毁会产生很大的开销，所以利用线程池机制，重用线程而不是新创建线程，避免了由于创建线程而带来的时间延迟。

<!-- more -->

**创建线程池**

> newSingleThreadExecutor: 创建只有单个线程的线程池 
> newFixedThreadPool: 创建固定个数的线程池
> newCachedThreadPool: 创建缓存线程池，可以按需，灵活的增加和删除线程
> newScheduledThreadPool: 创建一个定期处理任务的线程池


**启动线程**
通过创建的线程池来启动线程。
```
ExecutorService service = Executors.newSingleThreadExecutor();
service.submit(() -> {
    System.out.println(Thread.currentThread().getName());
});
```
`submit`可以接受`Runnable`和`Callable`两种Functional Interface。 区别是前者没有返回值，后者有返回值。

**InvokeAny**
`InvokeAny`用来接受一个Callable的集合，但它只会返回最快完成任务的那个Callable的返回值。在下面这个例子中，会返回'task3'。
```
ExecutorService service = Executors.newFixedThreadPool(3);
List<Callable<String>> tasks = Arrays.asList(
        () -> {
            TimeUnit.SECONDS.sleep(8);
            return "task1";
        },
        () -> {
            TimeUnit.SECONDS.sleep(9);
            return "task2";
        },
        () -> {
            TimeUnit.SECONDS.sleep(7);
            return "task3";
        }
);
String result = service.invokeAny(tasks);
System.out.println(result);

//task3
```

**InvokeAll**
`InvokeAll`可以用来处理一个Callable的集合，它会返回这些Callable的结果的List。
```
ExecutorService service = Executors.newFixedThreadPool(3);
List<Callable<String>> tasks = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

List<Future<String>> futures = service.invokeAll(tasks);
futures.forEach(stringFuture -> {
    try {
        System.out.println(stringFuture.get());
    } catch (Exception e) {
        e.printStackTrace();
    }
});

//task1
//task2
//task3
```

**Future**
上面例子用到了future。future用来检索Callable的计算结果，`isDone()`用来检查是否执行完毕，`get()`用来获得返回结果，但是会阻塞当前线程。
```
ExecutorService service = Executors.newFixedThreadPool(1);
String result = service.submit(() -> "task!").get();
System.out.println(result);

//task!
```

---

## CompletableFuture
在JDK8之前的future功能很弱，只能检测线程执行状态和获得结果，而且获得结果还会阻塞当前线程。JDK8推出的completable future借鉴了Google的Guava等框架的思想，增强了future，实现了真正的异步操作。

**创建completableFuture**
`CompletableFuture.runAsync`异步运行一个Runnable，
`CompletableFuture.supplyAsync`异步运行一个Supplier，和Runnable区别是这个有返回值。
它们都可以选择指定运行的线程池，否则就用默认的`ForkJoinPool.commonPool()`。
```
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "3");
```

**异步回调**
在上一个任务做完之后，返回一个CompletableFuture，我们希望用回调函数来继续利用这个值做其他事情。可以调用`thenApply`或者`thenApplyAsync`。没有以Async结尾的表示，会使用future的同一个线程，带Async的表示使用与future不同的线程。
```
CompletableFuture.supplyAsync(() -> "3")
        .thenApplyAsync(s -> s + "_")
        .thenApplyAsync(s1 -> {
            String result = s1 + "@";
            System.out.println(result);
            return result;
        });
System.out.println("main");
Thread.sleep(1000);

//main (可能main在前，可能3_@在前)
//3_@
```

**线程任务完成**
在future任务完成后，通常调用`thenAccept`,`thenAcceptAsync`,`thenRun`,`thenRunAsync`来执行最后的操作。
```
CompletableFuture.supplyAsync(() -> "3")
        .thenApply(s -> s + "_")
        .thenApply(s1 -> s1 + "@")
        .thenAccept(System.out::println);

System.out.println("main");
Thread.sleep(1000);

//main (可能main在前，可能3_@在前)
//3_@
```

CompletableFuture还有比如异常处理，组合多个future等等功能，就不再这里举例了。

---

## 参考文档
[Java Executors and Future](http://winterbe.com/posts/2015/04/07/java8-concurrency-tutorial-thread-executor-examples/)
[Java CompletableFuture](http://blog.zhouhaocheng.cn/posts/41)
