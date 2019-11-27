# synchronized和ReentrantLock的区别

### 相同点
- 互斥 => 同时只有一个线程获取锁
- 内存可见性 => 对共享变量的修改对另一个线程立即可见
- 可重入 => 同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞

### 不同点
| 锁 | synchronize | ReentrantLock |
|:-: | :-: | :-: |
|加锁释放锁方式 | 使用者无需关心，自动加锁、释放锁 | 显式加锁、释放锁，必须调用lock方法获取锁，调用unlock方法释放锁 |
| 中断 | 不可响应中断| 可响应中断 |
| 超时获取锁 | 不允许 | 允许 |
| 是否可以实现公平锁 | 否，默认就为非公平锁 | 是，通过构造函数指定是否为公平锁，默认为非公平锁，传入true为公平锁 |
| 实现方式 | JVM级别 | API级别 |

ReentrantLock相比synchronized更灵活一些

### ReentrantLock方法测试
ReentrantLock是Lock接口的其中一个实现类，Lock接口中定义的方法有：
- void lock() => 立即获取锁
- boolean tryLock() => 立即获取锁
- boolean tryLock(long time, TimeUnit unit) throws InterruptedException => 在指定时间内获取锁，获得锁返回true，获取不到锁返回false
- void lockInterruptibly() throws InterruptedException => 获取可中断锁
- void unlock() => 释放锁
- Condition newCondition() => 返回绑定到此 Lock 实例的Condition 实例

lock和tryLock方法除了返回值不一样以外，lock获取到的锁是不可响应中断的，而tryLock获取到的锁是可响应中断的。除此以外tryLock(long time, TimeUnit unit)获取到的锁也是可响应中断，即获取锁的方法中只有lock方法获取到的锁是不可以响应中断的

**lockInterruptibly响应中断：ReentrantLockLockInterruptiblyTest.java**
```
public class ReentrantLockLockInterruptiblyTest {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);  //执行三秒后中断线程
        thread.interrupt();
    }

    public static class Task implements Runnable {
        Lock lock = new ReentrantLock();

        public Task () {
            new Thread(() -> {
                try {
                    lockMethod();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        @Override
        public void run() {
            try {
                lockMethod();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("End...");
        }

        private void lockMethod() throws InterruptedException {
            lock.lockInterruptibly();
            try {
                //模拟长时间不释放锁
                while (true) {}
            } finally {
                lock.unlock();
            }
        }
    }
}
```
执行结果：
```
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.h2t.study.concurrent.lock.ReentrantLockLockInterruptiblyTest$Task.lockMethod(ReentrantLockLockInterruptiblyTest.java:45)
	at com.h2t.study.concurrent.lock.ReentrantLockLockInterruptiblyTest$Task.run(ReentrantLockLockInterruptiblyTest.java:37)
	at java.lang.Thread.run(Thread.java:748)
End...
```
中断成功

**synchronize响应中断：SynchronizedBlock.java**
```
public class SynchronizedBlock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);  //执行三秒后中断线程
        thread.interrupt();
        System.out.println(thread.isInterrupted());
    }

    public static class Task implements Runnable {
        public Task() {
            new Thread() {
                public void run() {
                    f();
                }
            }.start();
        }

        public synchronized void f() {
            while (true) {
            }
        }

        @Override
        public void run() {
            f();
            System.out.println("End");
        }
    }
}
```
控制台永远不会抛出异常、打印出End<br>
**对于synchronized来说，如果一个线程在等待锁，调用中断线程的方法，不会生效即不响应中断。而lock可以响应中断**

**tryLock定时锁：ReentrantLockTryLockTest.java**
```
public class ReentrantLockTryLockTest {
    public static void main(String[] args) {
        ExecutorService es = Executors.newCachedThreadPool();
        for (int i = 0; i < 2; i++) {
            es.execute(new Task(i));
        }
    }

    private static class Task implements Runnable {
        private static Lock lock = new ReentrantLock();

        private int i;

        public Task(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            try {
                lockMethod();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        //每次只允许一个线程调用
        private void lockMethod() throws InterruptedException {
            long start = System.currentTimeMillis();
            //2s内获得锁
            if (lock.tryLock(2, TimeUnit.SECONDS)) {
                System.out.println(String.format("i = %d 获取到锁，耗时：%d", i, System.currentTimeMillis() - start));
                try {
                    Thread.sleep(1000 * 60 * 1);  //睡眠1分钟
                } finally {
                    lock.unlock();
                }

            } else {
                System.out.println(String.format("i = %d 获取到锁失败，耗时：%d", i, System.currentTimeMillis() - start));
            }
        }
    }
}
```
run方法中调用了加锁的方法，加锁方法中尝试在2s内获得锁
执行结果：
```
i = 0 获取到锁，耗时：0
i = 1 获取到锁失败，耗时：2001
```

** ReentrantLock公平锁与非公平锁：ReentrantLockFairTest.java**
```
public class ReentrantLockFairTest {
    //通过传入true创建一个公平锁
    private static Lock fairLock = new ReentrantLock(true);
    //非公平锁，默认为非公平锁
    private static Lock unfairLock = new ReentrantLock();

    public static void main(String[] args) {
        ExecutorService unfairEs = Executors.newCachedThreadPool();
        ExecutorService fairEs = Executors.newCachedThreadPool();

        for (int i = 0; i < 5; i++) {
            unfairEs.execute(new UnfairTask(i));
            fairEs.execute(new FairTask(i));
        }
    }

    /**
     * 非公平锁任务
     * */
    private static class UnfairTask implements Runnable {
        private int i;

        public UnfairTask(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            unfairLock.lock();
            try {
                System.out.println(String.format("unfairTask i = %d is running", i));
            } finally {
                unfairLock.unlock();
            }
        }
    }

    /**
     * 公平锁任务
     * */
    private static class FairTask implements Runnable {
        private int i;

        public FairTask(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            fairLock.lock();
            try {
                System.out.println(String.format("fairTask i = %d is running", i));
            } finally {
                fairLock.unlock();
            }
        }
    }
}
```
执行结果：
```
unfairTask i = 0 is running
fairTask i = 0 is running
unfairTask i = 1 is running
fairTask i = 1 is running
unfairTask i = 2 is running
fairTask i = 3 is running
unfairTask i = 3 is running
fairTask i = 2 is running
fairTask i = 4 is running
unfairTask i = 4 is running
```
公平锁先到先得，因此执行顺序是有序的。非公平锁如果后来提交的线程刚好获取释放掉的锁将获得锁先执行，因此结果执行顺序是无序的

**synchronized非公平锁：**
```
public class SynchronizedUnfairTest {
    public static void main(String[] args) {
        ExecutorService unfairEs = Executors.newCachedThreadPool();

        for (int i = 0; i < 5; i++) {
            unfairEs.execute(new UnfairTask(i));
        }
    }

    /**
     * 非公平锁任务
     * */
    private static class UnfairTask implements Runnable {
        private int i;

        public UnfairTask(int i) {
            this.i = i;
        }

        @Override
        public synchronized void run() {
            System.out.println(String.format("unfairTask i = %d is running", i));
        }
    }
}
```
执行结果：
```
unfairTask i = 1 is running
unfairTask i = 0 is running
unfairTask i = 3 is running
unfairTask i = 2 is running
unfairTask i = 4 is running
```
synchronized默认为非公平锁，并且只能是非公平锁，因此执行结果顺序是无序的

### 性能测试
**ReentrantLock加锁任务：ReentrantLockTask.java**
```
public class ReentrantLockTask implements Runnable {
    private int i;

    public ReentrantLockTask(int i) {
        this.i = i;
    }

    @Override
    public void run() {
        lockMethod();
    }

    ReentrantLock lock = new ReentrantLock();
    private void lockMethod() {
        int sum = 0;
        lock.lock();
        try {
            for (int j = 0; j < 10; j++) {
                sum += j;
            }

        } finally {
            lock.unlock();
        }
    }
}
```

**synchronized加锁任务：SynchronizedLockTask.java**
```
public class SynchronizedLockTask implements Runnable {
    private int i;

    public SynchronizedLockTask(int i) {
        this.i = i;
    }

    @Override
    public void run() {
        lockMethod();
    }

    private synchronized void lockMethod() {
        int sum = 0;

        for (int j = 0; j < 10; j++) {
            sum += j;
        }
    }
}
```

**测试类：PerformTest**
```
public class PerformTest {
    public static void main(String[] args) {
        for (int i = 100; i < 1000000000; i = i * 10) {
            reentrantLockTest(i);
            synchronizedLockTest(i);
        }
    }

    /**
     * 循环执行的次数
     * */
    private static void reentrantLockTest(int time) {
        ExecutorService es = Executors.newCachedThreadPool();
        long start = System.currentTimeMillis();
        for (int i = 0; i < time; i++) {
            es.execute(new ReentrantLockTask(i));
        }

        System.out.println(String.format("ReentrantLockTest time = %d Spend %d", time, System.currentTimeMillis() - start));
    }

    private static void synchronizedLockTest(int time) {
        ExecutorService es = Executors.newCachedThreadPool();
        long start = System.currentTimeMillis();
        for (int i = 0; i < time; i++) {
            es.execute(new SynchronizedLockTask(i));
        }

        System.out.println(String.format("SynchronizedLockTest time = %d Spend %d", time, System.currentTimeMillis() - start));
    }
}
```
循环执行任务，统计循环任务的耗时

**测试结果：**
```
ReentrantLockTest time = 100 Spend 6
SynchronizedLockTest time = 100 Spend 2
ReentrantLockTest time = 1000 Spend 7
SynchronizedLockTest time = 1000 Spend 14
ReentrantLockTest time = 10000 Spend 42
SynchronizedLockTest time = 10000 Spend 29
ReentrantLockTest time = 100000 Spend 186
SynchronizedLockTest time = 100000 Spend 156
ReentrantLockTest time = 1000000 Spend 1428
SynchronizedLockTest time = 1000000 Spend 1006
ReentrantLockTest time = 10000000 Spend 9716
SynchronizedLockTest time = 10000000 Spend 9791
ReentrantLockTest time = 100000000 Spend 97928
SynchronizedLockTest time = 100000000 Spend 99804
```
synchronized和ReentrantLock性能相差不大，分不出谁好谁不好

### synchronized和ReentrantLock之间如何选择
synchronized和ReentrantLock性能差不多，**当且仅当synchronized无法满足的情景下使用ReentrantLock**，因为ReentrantLock需要显式释放锁，同时synchronized是JVM级别的，JVM能对其进行优化，而Reentrant是API级别的不会有任何优化。synchronized无法满足的情景：
- 定时获取锁
- 响应中断
- 需要以公平的方式获取锁

**最后附：**[实例代码](https://github.com/TiantianUpup/java-learning/tree/master/src/main/java/com/h2t/study/concurrent/lock)