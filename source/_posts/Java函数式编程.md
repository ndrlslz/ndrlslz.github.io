title: Java函数式编程
date: 2016-04-30 21:48:07
tags:
---



通过Python接触到函数式编程后， 希望写Java的时候也可以写的这样简洁，优雅。幸好Java8也推出了函数式编程的概念。

Java8的新特性中，最吸引人的应该就是lambda和stream了吧。

## Lambda

### Functional Interface
**functional interface**是只包含一个abstract method的接口。可以通过`@FunctionalInterface`注解显式的标记接口，表明它是个functional interface。如果往里面加入第2个abstract method，编译器就会报错。但default method和statis method不会违反functional interface的规则。
```Java
@FunctionalInterface
public interface MyFunctionalInterface {
    void function(String input);

    default void default_function() {
        System.out.println("This is default function");
    }

    static void static_function() {
        System.out.println("This is static function");
    }
}
```
<!-- more -->

### Lambda

在没有lambda表达式的时候，我们实例化functional interface接口的时候通常用匿名内部类的方式。
```java
MyFunctionalInterface myFunctionalClass = new MyFunctionalInterface() {
    @Override
    public void function(String input) {
        System.out.println(input);
    }
};
```

通过lambda表达式可以写的更简洁，因为对于functional interface，编译器知道我们只需要重载那个唯一的abstract method。
```java
MyFunctionalInterface myFunctionalClass = 
(input) -> System.out.println(input);
```

对于上面这个lambda，因为只是把参数传给了println方法，没有做其他事情，所以可以写成通过方法引用的方式，即把方法作为一个函数指针传进去，`::`表示传入方法或构造器的引用。
```java
MyFunctionalInterface myFunctionalClass = System.out::println;
```

### 内置的functional interface

JDK8之前大家熟悉的functional interface比如有Runnable，Comparator。
JDK8新加了几种常用的functional interface。

**Predicate**
Predicte接口的abstract method接受一个参数，返回一个boolean类型。
```
Predicate<Integer> predicate = integer -> integer > 0;
System.out.println(predicate.test(-1)); //false
```

**Function**
Function接口的abstract method接受一个参数，返回一个结果。和Predicate不同的是，Predicate只返回boolean类型。
```java
Function<Integer, String> function = integer -> integer + "_test";
System.out.println(function.apply(10)); //10_test
```

**Supplier**
Supplier接口不接受参数，返回一个结果。和Function不同的是，Function要接受一个参数。
```java
Supplier supplier = () -> 10;
System.out.println(supplier.get()); //10
```

**Consumer**
Consumer接口接受一个参数，但不返回结果，和Function不同的是，Function要返回一个结果。
```java
Consumer<Integer> consumer = System.out::println;
consumer.accept(10); //10
```
----

## Stream
stream表示一些元素的序列，并且这个序列支持**串行**和**并行**的操作。
对steam的操作分为**中间操作**和**末尾操作**。中间操作会返回steam，所以可以对其继续做操作；
末尾操作会返回一个结果，表示steam操作结束。

### 创建stream
```java
//通过调用集合的stream()方法创建
Arrays.asList("a", "b", "c").stream();

//通过Stream.of()创建
Stream.of("a", "b", "c");
```

### 部分stream操作例子

**Filter**
filter操作接收一个Predicate来对stream过滤，过滤掉不满足Predicate的元素。
filter操作是个中间操作，这里我们调用一个末尾操作`forEach`来显示结果，forEach接收一个Consumer来对每个元素执行指定的方法。
```
Arrays.asList("a1", "b1", "c1", "a2", "b2", "c2")
        .stream()
        .filter(s -> s.startsWith("a"))
        .forEach(System.out::println);

//a1, a2
```

**Map**
map操作接受一个Function来对stream的每个元素执行指定的功能，map也是一个中间操作。
```
Arrays.asList("a1", "b1", "c1", "a2", "b2", "c2")
        .stream()
        .map(String::toUpperCase)
        .forEach(System.out::println);

//A1,B1,C1,A2,B2,C2
```

**Reduce**
reduce操作接收一个Function来操作stream的序列，这个函数接受2个参数，得到的结果再与序列的下一个元素做累积计算。
比如对序列[1, 2, 3, 4]做函数叫fun的reduce操作，等同于：fun(fun(fun(1, 2), 3), 4)
reduce是个末尾操作。返回Function累积计算的结果。结果是个`Optional`类型，Optional即对原来的对象封装的一层，目的是方便处理null的情况和强制让程序员思考null的情况。
```java
Optional<String> result = Arrays.asList("a1", "b1", "c1", "a2", "b2", "c2")
        .stream()
        .reduce((s1, s2) -> s1 + "," + s2);

result.ifPresent(System.out::println);

//a1,b1,c1,a2,b2,c2
```

### stream执行流程
```
Arrays.asList("a1", "b1", "a2", "b2")
        .stream()
        .map(s -> {
            System.out.println("map:" + s);
            return s.toUpperCase();
        })
        .filter(s1 -> {
            System.out.println("filter:" + s1);
            return s1.startsWith("A");
        })
        .forEach(s2 -> System.out.println("forEach" + s2));

//map:a1
//filter:A1
//forEach:A1
//map:b1
//filter:B1
//map:a2
//filter:A2
//forEach:A2
//map:b2
//filter:B2
```
可以看出来stream的执行流程不是水平执行，并不是想象中的先对所有元素map，得到结果再filter。而是**垂直执行**的，先对第一个元素执行map，filter，forEach，完了之后再执行第二个元素。

当然，sort操作就是水平执行的，因为要先对所有元素排序嘛。

这种机制的好处在于可以提升速度。表现在2个方面：
1. 我们可以合理搭配stream操作，尽可能的减少对元素的操作次数。 比如上面这个例子，通过换下filter和map的顺序,先过滤掉部分元素，再map。
```java
Arrays.asList("a1", "b1", "a2", "b2")
        .stream()
        .filter(s1 -> {
            System.out.println("filter:" + s1);
            return s1.startsWith("a");
        })
        .map(s -> {
            System.out.println("map:" + s);
            return s.toUpperCase();
        })
        .forEach(s2 -> System.out.println("forEach:" + s2));

//filter:a1
//map:a1
//forEach:A1
//filter:b1
//filter:a2
//map:a2
//forEach:A2
//filter:b2
```
就减少了2次函数操作。

第2点就是下面的并行计算了。

### parallel stream

stream有2种，sequential stream和parallel stream。
之前创建的stream就是**sequential stream**，对其的操作都是单线程来运行的。
而**parallel stream**的操作可以用多线程来运行。

在上面那个例子中，序列元素之间的stream操作互不影响，没有必要等a1完了再操作b1，完全可以并发进行，所以可以通过parallel stream这种多线程的方式来执行stream操作。


```java
Arrays.asList("a1", "b1", "a2", "b2")
        .parallelStream()
        .filter(s -> {
            System.out.printf("filter: %s [%s]\n",
                    s, Thread.currentThread().getName());
            return s.startsWith("a");
        })
        .map(s1 -> {
            System.out.printf("map: %s [%s]\n", 
                    s1, Thread.currentThread().getName());
            return s1.toUpperCase();
        })
        .forEach(s2 -> {
            System.out.printf("forEach: %s [%s]\n", 
                    s2, Thread.currentThread().getName());
        });

//filter: a2 [main]
//filter: a1 [ForkJoinPool.commonPool-worker-3]
//filter: b1 [ForkJoinPool.commonPool-worker-1]
//map: a1 [ForkJoinPool.commonPool-worker-3]
//forEach: A1 [ForkJoinPool.commonPool-worker-3]
//filter: b2 [ForkJoinPool.commonPool-worker-2]
//map: a2 [main]
//forEach: A2 [main]
```
可以看到，parallel stream从ForkJoinPool线程池中利用空闲线程来并发执行stream操作。

----

## Guava
以上是通过Java8来实现函数式编程的部分内容，但目前也有很多项目没有使用jdk8，我们可以通过google的guava库来实现类似的操作。

通过guava的fluent风格也可以写出类似代码：
```java
ImmutableList<String> lists = ImmutableList.of("a1", "b1", "a2", "b2");

/**
 * 1. [a1, b1, a2, b2] filter -> [a1, a2]
 * 2. [a1, a2] transform(map) -> [A1, A2]
 * 3. [A1, A2] join with "_" -> A1_A2
 */
String result = Joiner
        .on("_")
        .skipNulls()
        .join(from(lists)
                .filter(new Predicate<String>() {
                    public boolean apply(String s) {
                        return s.startsWith("a");
                    }
                })
                .transform(new Function<String, String>() {
                    public String apply(String s) {
                        return s.toUpperCase();
                    }
                }));

System.out.println(result);

//A1_A2
```

但不要过度使用guava的函数式编程，如果用原来的方法更简洁、快速的话，就不要用guava的函数式。
在JDK8之前的话，传统方式的代码应该还是第一选择，除非guava的函数式可以有效减少代码量或者增加速度。

----

## 参考文档

[Java8 feature](http://winterbe.com/posts/2014/03/16/java-8-tutorial/)
[Java8 stream](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)
[Guava](http://ifeve.com/google-guava/)
