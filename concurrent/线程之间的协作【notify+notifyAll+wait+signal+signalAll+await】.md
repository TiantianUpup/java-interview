> 通过互斥【synchronized、lock】可以使得当多个线程竞争某个共享资源时，有且只有一个线程能访问到共享资源。但有时需要线程协作完成某项任务，比如A线程完成某个任务之后，B线程才能执行任务，java提供了两种方式实现线程之间的协作，一是Object类中的wait、notify、notifyAll方法，二是Condition类中的await、signal方法

### Object#wait、notify、notifyAll
- wait => 线程的执行挂起，并且释放持有的锁。进入该对象的等待池等待被唤醒
- notify => **随机**唤醒等待该锁的线程，其余线程将继续等待notify或notifyAll方法唤醒。进入该对象的锁池竞争锁
- notifyAll => 唤醒所有等待该锁的线程，并且只有一个线程能获取锁。进入该对象的锁池竞争锁

其中wait方法中是可以有参数，wait有三个重载方法，分别为wait()、wait(long timeout)、wait(long timeout, int nanos)。wait()方法相当于wait(0)，永远等待，而调用wait(long timeout)将等待timeout时间然后重新去竞争锁，wait(long timeout, int nanos)等待的时间将有两部分组成，调用wait(long timeout, int nanos)方法的线程将等待1000000*timeout+nanos的时间然后在去竞争锁

**注意：**
- 必须在同步代码块或同步方法里调用wait、notify、notifyAll方法，否则将抛出IllegalMonitorStateException异常
- 调用notify或notifyAll方法以后，被唤醒的线程不会立即获取锁，而是等待notify或notifyAll退出同步方法或同步代码块 

##### 示例代码
**wait方法任务类：WaitTask.java**
```
public class WaitTask implements Runnable {
    private List<String> strList;
    
    /**
     * 用于标识线程
     * */
    private int i;

    public WaitTask(List<String> strList, int i) {
        this.strList = strList;
        this.i = i;
    }

    @Override
    public void run() {
        synchronized (strList) {
            try {
                long start = System.currentTimeMillis();
                strList.wait();
                System.out.println(String.format("Thread i = %d arose and Cost = %d ", i, System.currentTimeMillis() - start));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

**notify方法任务类：NotifyTask.java**
```
public class NotifyTask implements Runnable {
    private List<String> strList;

    public NotifyTask(List<String> strList) {
        this.strList = strList;
    }

    @Override
    public void run() {
        synchronized (strList) {
            System.out.println("Wake up thread after 3000 milliSecond");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            strList.notify();
        }
    }
}
```

**测试类：WaitAndNotifyTest.java**
```
public class WaitAndNotifyTest {
    public static void main(String[] args) {
        List<String> strList = new ArrayList<>();
        new Thread(new WaitTask(strList)).start();
        new Thread(new NotifyTask(strList)).start();
    }
}
```
**执行结果：**
```
Wake up thread after 3000 milliSecond
Thread i = 1 arose and Cost = 3001 
```
调用wait和notify方法的同步代码块里都是List<String> strList对象，3000毫秒以后调用notify方法唤醒等待的线程。因为此时只有一个线程挂起因此将立即获取锁执行

**测试notify方法随机唤醒：WaitAndNotifyTest.java**
```
public class WaitAndNotifyTest {
    public static void main(String[] args) {
        List<String> strList = new ArrayList<>();
        new Thread(new WaitTask(strList, 1)).start();
        new Thread(new WaitTask(strList, 2)).start();
        new Thread(new WaitTask(strList, 3)).start();
        new Thread(new WaitTask(strList, 4)).start();
        new Thread(new NotifyTask(strList)).start();
    }
}
```
**执行结果：**
```
Wake up thread after 3000 milliSecond
Thread i = 3 arose and Cost = 3001 
```
第三个线程被唤醒，其余线程将继续等待被唤醒，即使锁已经不被其他线程所占有

**测试notifyAll方法唤醒全部：WaitAndNotifyAllTest**
```
public class WaitAndNotifyAllTest {
    public static void main(String[] args) {
        List<String> strList = new ArrayList<>();
        new Thread(new WaitTask(strList, 1)).start();
        new Thread(new WaitTask(strList, 2)).start();
        new Thread(new WaitTask(strList, 3)).start();
        new Thread(new WaitTask(strList, 4)).start();
        new Thread(new NotifyAllTask(strList)).start();
    }
}
```
**notifyAll方法任务类：NotifyAllTask.java**
```
public class NotifyAllTask implements Runnable {
    private List<String> strList;

    public NotifyAllTask(List<String> strList) {
        this.strList = strList;
    }

    @Override
    public void run() {
        synchronized (strList) {
            System.out.println("Wake up thread after 3000 milliSecond");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            strList.notifyAll();
        }
    }
}
```
**执行结果：**
```
Wake up thread after 3000 milliSecond
Thread i = 3 arose and Cost = 3001 
Thread i = 4 arose and Cost = 3029 
Thread i = 2 arose and Cost = 3030 
Thread i = 1 arose and Cost = 3032 
```
所有等待的线程将被唤醒，并且竞争锁执行任务

### Condition#await、signal、signalAll
- await => 对应Object中的wait方法，调用interrupt方法将会抛出异常
- signal => 对应Object中的notify方法
- signalAll => 对应Object中的notifyAll方法
- awaitUninterruptibly => 和await方法类似，但是awaitUninterruptibly调用interrupt方法将不会报错
- awaitNanos => 等待指定的时间在去唤醒挂起的线程
- awaitUntil => 通过传递一个Date，直到那个日期在唤醒挂起的线程

其中await有两个重载方法一个是await()永远等待，另外一个是await(long time, TimeUnit unit)，可以只当时间和时间单位，提供更灵活的时间范围

**注意：**
- 所有方法必须在同步代码块里使用，否则将抛出IllegalMonitorStateException异常
- 调用signal或signalAll方法以后，被挂起的线程不会立即获取锁，而是等待signal或signalAll退出同步代码块 

### 实例代码
**await方法任务类：AwaitTask.java**
```
public class AwaitTask implements Runnable {
    /**
     * 锁
     * */
    private Lock lock;

    private Condition condition;

    /**
     * 线程标识符
     * */
    private int i;

    public AwaitTask(Lock lock, Condition condition, int i) {
        this.lock = lock;
        this.i = i;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            long start = System.currentTimeMillis();
            condition.await();
            System.out.println(String.format("Thread i = %d arose and Cost = %d ", i, System.currentTimeMillis() - start));
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```
**signal任务类：SignalTask.java**
```
public class SignalTask implements Runnable {
    /**
     * 锁
     * */
    private Lock lock;

    private Condition condition;


    public SignalTask(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            System.out.println("Wake up thread after 3000 milliSecond");
            Thread.sleep(3000);
            condition.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```
**测试类： AwaitAndSignalTest.java**
```
public class AwaitAndSignalTest {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        new Thread(new AwaitTask(lock, condition, 1)).start();
        new Thread(new AwaitTask(lock, condition, 2)).start();
        new Thread(new AwaitTask(lock, condition, 3)).start();
        new Thread(new AwaitTask(lock, condition, 4)).start();
        new Thread(new SignalTask(lock, condition)).start();
    }
}
```
**执行结果：**
```
Wake up thread after 3000 milliSecond
Thread i = 1 arose and Cost = 3002 
```
相同的锁、相同的Condition对象，有且只有一个线程被唤醒

**测试signalAll唤醒所有线程：AwaitAndSignalAllTest.java**
```
public class AwaitAndSignalAllTest {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        new Thread(new AwaitTask(lock, condition, 1)).start();
        new Thread(new AwaitTask(lock, condition, 2)).start();
        new Thread(new AwaitTask(lock, condition, 3)).start();
        new Thread(new AwaitTask(lock, condition, 4)).start();
        new Thread(new SignalAllTask(lock, condition)).start();
    }
}
```

**signalAll方法任务类：**
```
public class SignalAllTask implements Runnable {
    /**
     * 锁
     * */
    private Lock lock;

    private Condition condition;


    public SignalAllTask(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            System.out.println("Wake up thread after 3000 milliSecond");
            Thread.sleep(3000);
            condition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```
**执行结果：**
```
Wake up thread after 3000 milliSecond
Thread i = 2 arose and Cost = 3003 
Thread i = 3 arose and Cost = 3028 
Thread i = 1 arose and Cost = 3028 
Thread i = 4 arose and Cost = 3028 
```
所有挂起的线程将被唤醒，通过竞争锁执行任务

最后附：[实例代码]()