title: Java concurrent包学习(三)
date: 2016-06-08 22:40:30
tags:
---

在编写多线程程序的时候，需要考虑到对共享变量的访问。这篇学习下并发编程中锁的使用。

---
## Synchronized
在JDK5之前，通常是使用synchronized关键字来实现同步。

**synchronized简介**
考虑一种简单的场景，假设有个方法是让int变量加1。
```
int count = 0;

void increment() {
    count = count + 1;
}

```

<!-- more -->

然后利用多线程来运行这个方法10万次，最后结果会小于10万。
```
ExecutorService service = Executors.newFixedThreadPool(10);

IntStream.range(0, 100000)
        .forEach(value -> service.submit(this::increment));

service.shutdown();
service.awaitTermination(60, TimeUnit.SECONDS);

System.out.println(count);

//99978
```
因为`increment`这个方法不是原子态的，实际需要做3步。
1. 读取当前变量的值
2. 值+1
3. 将新的值写回给变量

通过synchronized可以保证同一时刻，只有一个线程能运行这个方法。
```
synchronized void increment() {
    count = count + 1;
}
```

**synchronized原理**
关于这部分的文档参考[synchronized原理](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)，我这里翻译一下。

synchronized同步的实现是基于一个叫`intrinsic lock(内部锁)`或者`monitor lock(监控器)`的锁，这个锁保证了在同一时刻，一个线程独占对象的访问权，以及建立[happen-before](http://blog.csdn.net/ns_code/article/details/17348313)关系。

每个对象都关联了一个内部锁，按照约定，如果一个线程要独占某个对象的访问权，那么在访问对象之前，需要拿到这个对象的内部锁，并且在操作完对象后，释放掉对象的内部锁。当一个线程拿到某个对象的内部锁后，其他线程如果也想拿同一个对象的内部锁的话，就会阻塞。

当一个线程访问synchronized同步方法的时候，它会自动获取这个方法所在对象的内部锁，并且在方法完成的时候，释放掉内部锁。即使在方法中出现异常，没有return，也会释放掉内部锁。

你可能会想如果这个方法是`static synchronized`的，那这个方法就属于类，而不属于某个对象。这种情况，线程会拿到`Class`的内部锁。

对于synchronized代码块，它可以更细粒度的控制同步。可以缩小同步的范围，以及控制去拿哪个对象的内部锁。

---
## Locks
synchronized关键字实际是使用隐式的锁，JDK5之后提供了`Lock`，它提供各种方法来细粒度的控制并发。

**ReentrantLock**
`ReentrantLock`和`synchronized`的隐式锁的效果差不多，都是`互斥`和`可重入`的，相比于synchronized，它还提供了一些额外的功能。

使用ReentrantLock， 通过`lock()`来获得锁，通过`unlock()`来释放锁。
```
ReentrantLock lock = new ReentrantLock();
int count = 0;

void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();
    }
}
```
很重要的一点是需要把unlock操作放到`finally`块中，避免因为发生异常，导致一直没有释放锁。

下面的例子展示了ReentrantLock除了有synchronized的功能外，还提供的部分其他功能。
```
ExecutorService executor = Executors.newFixedThreadPool(2);
ReentrantLock lock = new ReentrantLock();

executor.submit(() -> {
    lock.lock();
    try {
        sleep(1);
    } finally {
        lock.unlock();
    }
});

executor.submit(() -> {
    System.out.println("Locked: " + lock.isLocked());
    System.out.println("Held by me: " + lock.isHeldByCurrentThread());
    boolean locked = lock.tryLock();
    System.out.println("Lock acquired: " + locked);
});

service.shutdown();
service.awaitTermination(60, TimeUnit.SECONDS);

//Locked: true
//Held by me: false
//Lock acquired: false
```
除了用lock来获取锁外，可以用`tryLock()`来获取锁，但这个是非阻塞的，如果锁被其他线程占有，则返回false。所以必须通过返回值来判断是否取到了锁。

**ReadWriteLock**
ReadWriteLock提供了2种锁，`读锁`和`写锁`。

提供ReadWriteLock这种锁的原因是，当没有线程去改变共享数据的时候，并发的去读数据是线程安全的，意味着当没有线程拿到写锁的时候，是可以多个线程同时拿到读锁的。对于读数据很频繁，写数据很少的这种情况，使用ReadWriteLock可以提高性能。


```
ExecutorService executor = Executors.newFixedThreadPool(2);
Map<String, String> map = new HashMap<>();
ReadWriteLock lock = new ReentrantReadWriteLock();

executor.submit(() -> {
    lock.writeLock().lock();
    try {
        TimeUnit.SECONDS.sleep(3);
        map.put("foo", "bar");
    } finally {
        lock.writeLock().unlock();
    }
});
```
上面这段代码是起一个线程先去拿到写锁，然后sleep 3秒，改变共享数据，再释放写锁。

再通过下面的代码，在第一个线程释放写锁之前，再起2个线程去获取读锁，来读取数据。
```
Runnable readTask = () -> {
    lock.readLock().lock();
    try {
        System.out.println(map.get("foo"));
        TimeUnit.SECONDS.sleep(5);
    } finally {
        lock.readLock().unlock();
    }
};

executor.submit(readTask);
executor.submit(readTask);

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);
```
运行这两段代码，你会发现2个读的任务都被阻塞了，直到写的任务完成。当3秒钟过去，写锁被释放后，2个读的任务同时执行，打印出了结果。因为当没有线程占有写锁的时候，读锁是可以被多个线程拿到的。

**StampedLock**
`StampedLock`是Java8新增加的一种锁，除了支持上面的读锁和写锁外，它还会返回一个Long型的`stamp`，你可以用stamp来释放锁，或者检查锁是否有效。除此之外，它还支持乐观锁。

我们用StampedLock来重写上面的ReadWriteLock的代码。
```
ExecutorService executor = Executors.newFixedThreadPool(2);
Map<String, String> map = new HashMap<>();
StampedLock lock = new StampedLock();

executor.submit(() -> {
    Long stamp = lock.writeLock();
    try {
        System.out.println("lock");
        TimeUnit.SECONDS.sleep(3);
        map.put("foo", "bar");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlockWrite(stamp);
    }
});

Runnable readTask = () -> {
    Long stamp = lock.readLock();
    try {
        System.out.println(map.get("foo"));
        TimeUnit.SECONDS.sleep(5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlockRead(stamp);
    }
};

executor.submit(readTask);
executor.submit(readTask);
        
executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);
```
可以看到获取锁的时候返回了stamp，之后可以用这个stamp来释放锁。
记住一点是StampedLock是`不可重入的`。每次获取写锁都会产生新的stamp，如果写锁已被获取，便会阻塞，即使对于同一个线程也是。所以注意不要产生死锁了。


下面这个例子展示了乐观锁。
```
ExecutorService executor = Executors.newFixedThreadPool(2);
StampedLock lock = new StampedLock();

executor.submit(() -> {
    long stamp = lock.tryOptimisticRead();
    try {
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
        TimeUnit.SECONDS.sleep(1);
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
        TimeUnit.SECONDS.sleep(2);
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
    } finally {
        lock.unlock(stamp);
    }
});

executor.submit(() -> {
    long stamp = lock.writeLock();
    try {
        System.out.println("Write Lock acquired");
        TimeUnit.SECONDS.sleep(2);
    } finally {
        lock.unlock(stamp);
        System.out.println("Write done");
    }
});

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);


//Optimistic Lock Valid: true
//Write Lock acquired
//Optimistic Lock Valid: false
//Write done
//Optimistic Lock Valid: false
```
通过调用`tryOptimisticRead()`来获取乐观锁，这个方法总是会无阻塞的获取到读锁，即使有其他线程拿到写锁的时候。如果其他线程占有写锁了，再拿乐观锁的话，返回的stamp为0。可以通过`lock.validate(stamp)`来检查stamp是否有效。

从上面的结果可以看到，乐观锁不会阻止其他线程拿到写锁，但是当其他线程拿到写锁后，乐观锁之后就无效了，即使当其他线程的写锁释放后。

有时候需要将读锁变成写锁，而不用先释放读锁，再获取写锁。可以通过调用`tryConvertToWriteLock()`实现。
```
ExecutorService executor = Executors.newFixedThreadPool(2);
StampedLock lock = new StampedLock();

executor.submit(() -> {
    long stamp = lock.readLock();
    try {
        if (count == 0) {
            stamp = lock.tryConvertToWriteLock(stamp);
            if (stamp == 0L) {
                System.out.println("Could not convert to write lock");
                //无法获取读锁，因为写锁被其他线程占有
                //阻塞的等待其他线程先释放写锁
                stamp = lock.writeLock();
            }
            count = 23;
        }
        System.out.println(count);
    } finally {
        lock.unlock(stamp);
    }
});
```
上面的例子中，先调用tryConvertToWriteLock尝试把读锁换成写锁，然后判断stamp。stamp为0代表写锁已经被占有了，则阻塞的等待其他线程释放写锁。如果stamp不为0则可以直接转成写锁。

---
## Semaphores
并发包除了添加锁外，还添加了信号量semaphores。锁通常是控制单个线程独占变量或资源。而信号量是控制一组权限。当你需要控制并发线程的数量的时候，可以使用信号量。

```
ExecutorService executor = Executors.newFixedThreadPool(10);

Semaphore semaphore = new Semaphore(5);

Runnable longRunningTask = () -> {
    boolean permit = false;
    try {
        permit = semaphore.tryAcquire(1, TimeUnit.SECONDS);
        if (permit) {
            System.out.println("Semaphore acquired");
            TimeUnit.SECONDS.sleep(5);
        } else {
            System.out.println("Could not acquire semaphore");
        }
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    } finally {
        if (permit) {
            semaphore.release();
        }
    }
}

IntStream.range(0, 10)
    .forEach(i -> executor.submit(longRunningTask));

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);

//Semaphore acquired
//Semaphore acquired
//Semaphore acquired
//Semaphore acquired
//Semaphore acquired
//Could not acquire semaphore
//Could not acquire semaphore
//Could not acquire semaphore
//Could not acquire semaphore
//Could not acquire semaphore
```
上面的代码启了10个线程来运行方法，但是信号量设置为了5，因此同时只能有5个线程来运行方法。
还是需要注意将信号量的release放到finally块中，避免产生异常而没有释放掉信号量。

---
## 参考文档
本文主要翻译的这边文章[Java Lock](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)