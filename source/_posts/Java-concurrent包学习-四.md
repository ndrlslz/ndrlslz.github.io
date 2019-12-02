title: Java concurrent包学习(四)
date: 2016-06-22 00:42:19
tags:
---

上一篇通过使用锁来保证线程安全，这一篇学习使用原子变量。

---
## CAS(compare-and-swap)

**概念**
CAS是在CPU层面支持的指令，现代的CPU大部分都支持这个指令，它能对内存的共享数据进行原子的操作，因此可以保证线程安全。

**语义**
这个操作用C语言来描述，就是下面的代码:
```
int compare_and_swap (int* reg, int oldval, int newval)
{
  int old_reg_val = *reg;
  if (old_reg_val == oldval)
     *reg = newval;
  return old_reg_val;
}
```

<!-- more -->

上面代码的意思就是: 我想改变\*reg 这个地址的变量，我需要传入我认为的这个地址的值是多少，以及我想改变的新值。如果\*reg中现在的值是我认为的值，就把它改成新值。如果\*reg现在的值不是我认为的值，即已经被其他线程改了，则返回\*reg现在的正确的那个值。

一句话来描述就是:
我认为原有的值应该是什么，如果是，则将原有的值更新为新值，否则不做修改，并告诉我原来的值是多少。

---
## AtomicInteger

在内部，原子类大量使用CAS，这个原子指令是被现代的CPU支持的，这个指令要比通过锁实现的同步速度快。因此如果你只是想要并发的更改一个变量的话，优先使用原子类，而不是锁。

下面是使用`AtomicInteger`的例子:
```
AtomicInteger atomicInt = new AtomicInteger(0);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
    .forEach(i -> executor.submit(atomicInt::incrementAndGet));

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);

System.out.println(atomicInt.get());    // => 1000
```
通过使用`AtomicInteger`来替代`Integer`，我们可以在不用自己保证同步的情况下，并发的，线程安全的改变一个变量，`incrementAndGet()`是个原子操作，因此最后的结果是线程安全的。

AtomicInteger也支持多种原子操作，`updateAndGet()`方法接受一个lambda表达式来更改变量。
```
AtomicInteger atomicInt = new AtomicInteger(0);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
        .forEach(i -> {
            Runnable task = () ->
                    atomicInt.updateAndGet(n -> n + 2);
            executor.submit(task);
        });

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);

System.out.println(atomicInt);

//2000
```

和AtomicInteger类似的原子类还有`AtomicBoolean`,`AtomicLong`,`AtomicReference`。

---
## LongAdder
`LongAdder`可以用来替代`AtomicLong`来连续的增加一个变量。

```
LongAdder adder = new LongAdder();
ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
        .forEach(i -> executor.submit(adder::increment));
executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);

System.out.println(adder.sumThenReset());

//1000
```
不同于`AtomicLong`只保存单个结果的求和方式，这个原子类会保存一组变量来减少线程之间的竞争。这个类更适合于多线程情况下，写大于读的情况。但劣势是它会占用更多的内存。

---

## ConcurrentMap

`ConcurrentMap`接口实现了Map接口，几乎是最常用的并发集合。
ConcurrentMap的原理是通过锁分段的技术，加锁的粒度只需要在Segment段上，而不需要对整个Map加锁，极大的提高性能。
Java8也对这个接口添加了新的功能，下面的代码会演示一些新功能。

```
ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
map.put("foo", "bar");
map.put("han", "solo");
map.put("r2", "d2");
map.put("c3", "p0");
```

**forEach**
`forEach()`方法接受`BiConsumer`接口的lambda表达式，将key,value做为参数传入，这个方法可以用来替换for-each的遍历方式。
```
map.forEach((key, value) -> System.out.println(key + ":" + value));
```

**putIfAbsent**
`putIfAbsent()`方法，当key不存在的时候，可以给key赋值，如果key已经存在了，则不变。
```
String value = map.putIfAbsent("c3", "p1");
System.out.println(value); 

//p0
```

**getOrDefault**
`getOrDefault()`方法当key不存在的时候，会返回默认值。
```
String value = map.getOrDefault("hi", "there");
System.out.println(value);

//there
```

**replaceAll**
`replaceAll()`方法接受一个`BiFunction`的lambda表达式，这个表达式接受2个参数，返回1个值。下面的例子展示了 替换掉满足条件的key。
```
map.replaceAll((key, value) -> "r2".equals(key) ? "d3" : value);
System.out.println(map.get("r2"));

//d3
```

**compute**
`compute()`方法可以只对单个entry进行操作。
```
map.compute("foo", (key, value) -> value + value);
System.out.println(map.get("foo"));

//barbar
```

**merge**
最后，`merge()`方法可以对某个key的旧变量和新变量进行合并。 这个方法接受一个key，一个新变量的值，以及`BiFunction`表达式。
```
map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
System.out.println(map.get("foo"));

// boo was foo
```

---
## ConcurrentHashMap
上面的方法是`ConcurrentMap`接口增加的方法。对于这个接口最常用的实现类`ConcurrentHashMap`也增加了几个强大的并行计算的功能。它使用的线程池就是上一章提到的`ForkJoinPool`。

Java8对这个类增加了3种并行计算的功能，`forEach`,`search`,`reduce`。
这些方法的第一个参数都是`parallelismThreshold`,意为并行计算的阈值，假设设置为500，那么如果你的map的size小于500的话，就不会使用并行计算，如果大于500，就会使用并行计算了。
下面的例子中，为了保证使用并行计算，我们将阈值设置为1。

先创建好ConcurrentHashMap。
```
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("foo", "bar");
map.put("han", "solo");
map.put("r2", "d2");
map.put("c3", "p0");
```

**ForEach**
`forEach()`方法和上面提到的差不多，除了第一个参数为阈值外，为了表现它的确是并行计算的，我们将线程打印出来。

```
map.forEach(1, (key, value) ->
    System.out.printf("key: %s; value: %s; thread: %s\n",
        key, value, Thread.currentThread().getName()));

// key: r2; value: d2; thread: main
// key: foo; value: bar; thread: ForkJoinPool.commonPool-worker-1
// key: han; value: solo; thread: ForkJoinPool.commonPool-worker-2
// key: c3; value: p0; thread: main
```


**Search**
`search()`方法接受一个阈值，和`BiFunction`的表达式，表达式写为如果搜索到满足条件的结果，就返回结果，否则返回null。请记住map是无序的，如果有多个满足条件的key，则返回的值是不确定的。

```
String result = map.search(1, (key, value) -> {
    System.out.println(Thread.currentThread().getName());
    if ("foo".equals(key)) {
        return value;
    }
    return null;
});
System.out.println("Result: " + result);

// ForkJoinPool.commonPool-worker-2
// main
// ForkJoinPool.commonPool-worker-3
// Result: bar
```

**Reduce**
`reduce()`方法在Stream部分接触到了，这里它接受2个`BiFunction`，第一个Function对每一对key-value操作，并返回单个值。第二个Function将这些值连起来，忽略掉null的值。

```
String result = map.reduce(1,
    (key, value) -> {
        System.out.println("Transform: " + Thread.currentThread().getName());
        return key + "=" + value;
    },
    (s1, s2) -> {
        System.out.println("Reduce: " + Thread.currentThread().getName());
        return s1 + ", " + s2;
    });

System.out.println("Result: " + result);

// Transform: ForkJoinPool.commonPool-worker-2
// Transform: main
// Transform: ForkJoinPool.commonPool-worker-3
// Reduce: ForkJoinPool.commonPool-worker-3
// Transform: main
// Reduce: main
// Reduce: main
// Result: r2=d2, c3=p0, han=solo, foo=bar
```

---
## 参考文档
本文大部分翻译自[文档](http://winterbe.com/posts/2015/05/22/java8-concurrency-tutorial-atomic-concurrent-map-examples/)
[CAS](http://www.cnblogs.com/Mainz/p/3546347.html)
