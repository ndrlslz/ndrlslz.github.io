title: Spring Batch实践
date: 2017-06-27 22:45:32
tags:
---

## 概念
Spring Batch是一个轻量级的批处理框架，它被设计来开发企业级的批处理应用。因为企业级的应用通常需要处理大量的业务数据，需要在数据集非常大的情况下定时的进行处理。比如从外部的系统收集数据，进行验证，标准化，导入内部的系统。

<!-- more -->

## 架构
![Spring Batch构图](/uploads/spring-batch-reference-model.png)

一个批处理任务被叫做`Job`, 而一个Job可以分成多个`Step`, 比如读取数据，处理数据，导入数据分别是3个步骤。`JobLauncher`是用来启动Job的，`JobRepository`用来持久化一些数据，比如Job的状态，Context信息等。

## 实践

### 多Job配置
Spring Batch的官网教程上是只有一个Job的例子，如果要实现多Job的情况，需要把`EnableBatchProcessing`注解的`modular`设置为true,让每个Job使用自己的ApplicationConext。
```
@Configuration
@EnableBatchProcessing(modular = true)
@EnableAutoConfiguration
public class BatchConfiguration {
    @Bean
    public ApplicationContextFactory firstJobContext() {
        return new GenericApplicationContextFactory(FirstJobConfiguraiton.class);
    }

    @Bean
    public ApplicationContextFactory SecondJobContext() {
        return new GenericApplicationContextFactory(SecondJobConfiguraiton.class);
    }
}
```
然后每个Job有自己的Configuration，比如上面的`FirstJobConfiguration`和`SecondJobConfiguration`.

------------------------
### 健壮性

#### Listener & Error Handle
一般有两种方式来实现监听，第一种是继承Spring Batch自带的一些Listner。以下是自带的不同级别的监听器。
>* JobExecutionListener
>* StepExecutionListener
>* ItemReadListener
>* ItemProcessListener
>* ItemWriteListener
>* ChunkListener
>* SkipListener

还有一种是使用注解来自定义监听器，它可以解决上面那种监听器的弊端。
比如下面的代码，我们继承ItemReadListener，需要重写3个方法，即使我们用不到。
```
public class FileReaderListener implements ItemReadListener {
    @Override
    public void beforeRead() {

    }

    @Override
    public void afterRead(Object item) {

    }

    @Override
    public void onReadError(Exception ex) {

    }
}
```

如果只需要实现Listener中的Error Handle， 推荐用下面的注解来实现，这样就只需要实现我们关心的异常处理部分。
```
public class CustomListener {

    @OnReadError
    public void onReadError(Exception ex) {

    }

    @OnWriteError
    public void onWriteError(Exception ex, List items) {

    }
}
```

#### Retry & Skip
如果在处理数据的过程中发生了异常，我们一般都希望Job能继续执行下去，而不会中断。这时可以指定当遇到哪些异常的时候，跳过它，或者重试（如果重试可以解决的话，比如I/O异常）。
```
stepBuilderFactory.get("myStep")
        .chunk(1)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .skipPolicy(new AlwaysSkipItemSkipPolicy())
        .retry(ResourceAccessException.class)
        .retryLimit(5)
        .listener(listener)
        .build();
```

------------------

### 并发
通常来说，大部分情况都可以用单线程搞定，如果单线程可以满足需求的话就不要用多线程了。

#### Multi-threaded Step
```
stepBuilderFactory.get("myStep")
        .chunk(1)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .taskExecutor(new SimpleAsyncTaskExecutor())
        .throttleLimit(4)
        .build();
```
在一个步骤里使用多线程，指定一个taskExecutor和throttleLimit。这段代码意味着数据的读取，处理，写入都在单独的线程中执行，所以要保证共享的数据是线程安全的。

#### Parallel Steps
```
jobBuilderFactory.get("myJob")
        .start(step1)
        .split(new SimpleAsyncTaskExecutor()).add(flow1, flow2)
        .next(step3)
        .build();
```
并行运行多个步骤，上面的例子中，先运行step1，然后并行运行flow1和flow2，最后运行step3。

------------------

### 参考文档
[官方文档](https://docs.spring.io/spring-batch/reference/htmlsingle/#scalability)
