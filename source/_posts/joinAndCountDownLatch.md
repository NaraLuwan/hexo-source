---
title: join和countDownLatch原理及区别详解
tags: [Java,多线程,join]
categories:
  - Java
date: 2018-08-25 13:08:47
description:
---

## 源码解析
- join 原理：在当前线程中调用另一个线程线程 thread 的 join() 方法时，会调用该 thread 的 wait() 方法，直到这个 thread 执行完毕(JVM在 run() 方法执行完后调用 exit() 方法，而 exit() 方法里调用了 notifyAll() 方法)会调用 notifyAll() 方法主动唤醒当前线程。

源码如下：
```java
    public final void join() throws InterruptedException {
            join(0);
        }

        /**
         *  注意这个方法是同步的 
         */
        public final synchronized void join(long millis)
                        throws InterruptedException {
            long base = System.currentTimeMillis();
            long now = 0;

            if (millis < 0) {
                throw new IllegalArgumentException("timeout value is negative");
            }

            /**
             *  join方法默认参数为0，会直接阻塞当前线程
             */
            if (millis == 0) {
                while (isAlive()) {
                    wait(0);
                }
            } else {
                while (isAlive()) {
                    long delay = millis - now;
                    if (delay <= 0) {
                        break;
                    }
                    wait(delay);
                    now = System.currentTimeMillis() - base;
                }
            }
        }

        public final native boolean isAlive();
    }
```
    
- countDownLatch 原理：可以理解为一个计数器。在初始化 CountDownLatch 的时候会在类的内部初始化一个int的变量，每当调用 countDownt() 方法的时候这个变量的值减1，而 await() 方法就是去判断这个变量的值是否为0，是则表示所有的操作都已经完成，否则继续等待。
<!-- more -->
源码如下（源码比较少，直接全贴出来了，所有中文注释是我自己加上去的）：
```java
public static class CountDownLatch {
        private static final class Sync extends AbstractQueuedSynchronizer {
            private static final long serialVersionUID = 4982264981922014374L;

            /**
             * 初始化state
             */
            Sync(int count) {
                setState(count);
            }

            int getCount() {
                return getState();
            }

            /**
             * 尝试获取同步状态
             *     只有当同步状态为0的时候返回1
             */
            protected int tryAcquireShared(int acquires) {
                return (getState() == 0) ? 1 : -1;
            }

            /**
             * 自旋+CAS的方式释放同步状态
             */
            protected boolean tryReleaseShared(int releases) {
                // Decrement count; signal when transition to zero
                for (;;) {
                    int c = getState();
                    if (c == 0)
                        return false;
                    int nextc = c-1;
                    if (compareAndSetState(c, nextc))
                        return nextc == 0;
                }
            }
        }

        private final Sync sync;

        /**
         *  初始化一个同步器
         */
        public CountDownLatch(int count) {
            if (count < 0) throw new IllegalArgumentException("count < 0");
            this.sync = new Sync(count);
        }

        /**
         * 调用同步器的acquireSharedInterruptibly方法，并且是响应中断的
         */
        public void await() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
        }

        /**
         * 调用同步器的releaseShared方法去让state减1
         */
        public void countDown() {
            sync.releaseShared(1);
        }

        /**
         * 获取剩余的count
         */
        public long getCount() {
            return sync.getCount();
        }

        public String toString() {
            return super.toString() + "[Count = " + sync.getCount() + "]";
        }
    }
```

## 区别及注意事项
- join和countDownLatch都能实现让当前线程阻塞等待其他线程执行完毕，join使用起来更简便，不过countDownLatch粒度更细。
- 由于CountDownLatch需要开发人员很明确需要等待的条件，否则容易造成await()方法一直阻塞。

## 如何使用
- 一个简单的小例子
```
public class Test {
    private static final Logger logger = LoggerFactory.getLogger(Test.class);

    public static void main(String[] args) {
        long sleepTime = 5000;
        try {
            TestJoinThread joinThread1 = new TestJoinThread("joinThread1",sleepTime);
            TestJoinThread joinThrad2 = new TestJoinThread("joinThrad2",sleepTime);
            joinThread1.start();
            joinThrad2.start();
            joinThread1.join();
            joinThrad2.join();
            logger.info("主线程开始运行...");
        } catch (InterruptedException e) {
            logger.error("test join err!",e);
        }

        try {
            CountDownLatch count = new CountDownLatch(2);
            TestCountDownLatchThread countDownLatchThread1 = new TestCountDownLatchThread(count,"countDownLatchThread1",sleepTime);
            TestCountDownLatchThread countDownLatchThread2 = new TestCountDownLatchThread(count,"countDownLatchThread2",sleepTime);
            countDownLatchThread1.start();
            countDownLatchThread2.start();
            count.await();
            logger.info("主线程开始运行...");
        } catch (InterruptedException e) {
            logger.error("test countDownLatch err!",e);
        }
    }

    static class TestJoinThread extends Thread{

        private String threadName;
        private long sleepTime;

        public TestJoinThread(String threadName,long sleepTime){
            this.threadName = threadName;
            this.sleepTime = sleepTime;
        }

        @Override
        public void run() {
            try{
                logger.info(String.format("线程[%s]开始运行...",threadName));
                Thread.sleep(sleepTime);
                logger.info(String.format("线程[%s]运行结束 耗时[%s]s",threadName,sleepTime/1000));
            }catch (Exception e){
                logger.error("TestJoinThread run err!",e);
            }
        }
    }

    static class TestCountDownLatchThread extends Thread{

        private String threadName;
        private long sleepTime;
        private CountDownLatch countDownLatch;

        public TestCountDownLatchThread(CountDownLatch countDownLatch,String threadName,long sleepTime){
            this.countDownLatch = countDownLatch;
            this.threadName = threadName;
            this.sleepTime = sleepTime;
        }

        @Override
        public void run() {
            try{
                logger.info(String.format("线程[%s]开始运行...",threadName));
                Thread.sleep(sleepTime);
                logger.info(String.format("线程[%s]运行结束 耗时[%s]s",threadName,sleepTime/1000));
                countDownLatch.countDown();
            }catch (Exception e){
                logger.error("TestCountDownLatchThread run err!",e);
            }
        }
    }
}
```
日志输出：
```
11:18:01.985 [Thread-1] INFO com.sync.Test - 线程[joinThrad2]开始运行...
11:18:01.985 [Thread-0] INFO com.sync.Test - 线程[joinThread1]开始运行...
11:18:06.993 [Thread-1] INFO com.sync.Test - 线程[joinThrad2]运行结束...耗时[5]s
11:18:06.993 [Thread-0] INFO com.sync.Test - 线程[joinThread1]运行结束...耗时[5]s
11:18:06.993 [main] INFO com.sync.Test - 主线程开始运行...
11:18:06.995 [Thread-2] INFO com.sync.Test - 线程[countDownLatchThread1]开始运行...
11:18:06.995 [Thread-3] INFO com.sync.Test - 线程[countDownLatchThread2]开始运行...
11:18:11.996 [Thread-2] INFO com.sync.Test - 线程[countDownLatchThread1]运行结束...耗时[5]s
11:18:11.996 [Thread-3] INFO com.sync.Test - 线程[countDownLatchThread2]运行结束...耗时[5]s
11:18:11.996 [main] INFO com.sync.Test - 主线程开始运行...
```
可以看到：joinThread1 和 joinThread2 同时开始执行，5s后主线程开始执行。countDownLatchThread1 和 countDownLatchThread2 也是一样的效果。

那么我上面所说的粒度更细有怎样的应用场景呢？

我对 TestCountDownLatchThread类 的 run() 方法做一点小改动：
```
@Override 
public void run() {
    try{
        logger.info(String.format("线程[%s]第一阶段开始运行...",threadName);
        Thread.sleep(sleepTime);
        logger.info(String.format("线程[%s]第一阶段运行结束耗时[%s]s",threadName,sleepTime/1000));
        countDownLatch.countDown();
        logger.info(String.format("线程[%s]第二阶段开始运行...",threadName);
        Thread.sleep(sleepTime);
        logger.info(String.format("线程[%s]第二阶段运行结束耗时[%s]s",threadName,sleepTime/1000));
    }catch (Exception e){
        logger.error("TestCountDownLatchThread run err!",e);
    }
}
```
这个时候日志输出会变成这样：
```
12:59:35.912 [Thread-1] INFO com.sync.Test - 线程[countDownLatchThread2]第一阶段开始运行...
12:59:35.912 [Thread-0] INFO com.sync.Test - 线程[countDownLatchThread1]第一阶段开始运行...
12:59:40.916 [Thread-0] INFO com.sync.Test - 线程[countDownLatchThread1]第一阶段运行结束 耗时[5]s
12:59:40.916 [Thread-1] INFO com.sync.Test - 线程[countDownLatchThread2]第一阶段运行结束 耗时[5]s
12:59:40.916 [main] INFO com.sync.Test - 主线程开始运行...
12:59:40.916 [Thread-0] INFO com.sync.Test - 线程[countDownLatchThread1]第二阶段开始运行...
12:59:40.916 [Thread-1] INFO com.sync.Test - 线程[countDownLatchThread2]第二阶段开始运行...
12:59:45.917 [Thread-0] INFO com.sync.Test - 线程[countDownLatchThread1]第二阶段运行结束 耗时[5]s
12:59:45.917 [Thread-1] INFO com.sync.Test - 线程[countDownLatchThread2]第二阶段运行结束 耗时[5]s
```
也就是说如果当前线程只需要等待其他线程一部分任务执行完毕的情况下就可以用 countDownLatch 来实现了，而 join 则实现不了这种粒度的控制。

<hr />