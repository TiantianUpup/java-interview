> ͨ�����⡾synchronized��lock������ʹ�õ�����߳̾���ĳ��������Դʱ������ֻ��һ���߳��ܷ��ʵ�������Դ������ʱ��Ҫ�߳�Э�����ĳ�����񣬱���A�߳����ĳ������֮��B�̲߳���ִ������java�ṩ�����ַ�ʽʵ���߳�֮���Э����һ��Object���е�wait��notify��notifyAll����������Condition���е�await��signal����

### Object#wait��notify��notifyAll
- wait => �̵߳�ִ�й��𣬲����ͷų��е���������ö���ĵȴ��صȴ�������
- notify => **���**���ѵȴ��������̣߳������߳̽������ȴ�notify��notifyAll�������ѡ�����ö�������ؾ�����
- notifyAll => �������еȴ��������̣߳�����ֻ��һ���߳��ܻ�ȡ��������ö�������ؾ�����

����wait�������ǿ����в�����wait���������ط������ֱ�Ϊwait()��wait(long timeout)��wait(long timeout, int nanos)��wait()�����൱��wait(0)����Զ�ȴ���������wait(long timeout)���ȴ�timeoutʱ��Ȼ������ȥ��������wait(long timeout, int nanos)�ȴ���ʱ�佫����������ɣ�����wait(long timeout, int nanos)�������߳̽��ȴ�1000000*timeout+nanos��ʱ��Ȼ����ȥ������

**ע�⣺**
- ������ͬ��������ͬ�����������wait��notify��notifyAll�����������׳�IllegalMonitorStateException�쳣
- ����notify��notifyAll�����Ժ󣬱����ѵ��̲߳���������ȡ�������ǵȴ�notify��notifyAll�˳�ͬ��������ͬ������� 

##### ʾ������
**wait���������ࣺWaitTask.java**
```
public class WaitTask implements Runnable {
    private List<String> strList;
    
    /**
     * ���ڱ�ʶ�߳�
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

**notify���������ࣺNotifyTask.java**
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

**�����ࣺWaitAndNotifyTest.java**
```
public class WaitAndNotifyTest {
    public static void main(String[] args) {
        List<String> strList = new ArrayList<>();
        new Thread(new WaitTask(strList)).start();
        new Thread(new NotifyTask(strList)).start();
    }
}
```
**ִ�н����**
```
Wake up thread after 3000 milliSecond
Thread i = 1 arose and Cost = 3001 
```
����wait��notify������ͬ��������ﶼ��List<String> strList����3000�����Ժ����notify�������ѵȴ����̡߳���Ϊ��ʱֻ��һ���̹߳�����˽�������ȡ��ִ��

**����notify����������ѣ�WaitAndNotifyTest.java**
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
**ִ�н����**
```
Wake up thread after 3000 milliSecond
Thread i = 3 arose and Cost = 3001 
```
�������̱߳����ѣ������߳̽������ȴ������ѣ���ʹ���Ѿ����������߳���ռ��

**����notifyAll��������ȫ����WaitAndNotifyAllTest**
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
**notifyAll���������ࣺNotifyAllTask.java**
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
**ִ�н����**
```
Wake up thread after 3000 milliSecond
Thread i = 3 arose and Cost = 3001 
Thread i = 4 arose and Cost = 3029 
Thread i = 2 arose and Cost = 3030 
Thread i = 1 arose and Cost = 3032 
```
���еȴ����߳̽������ѣ����Ҿ�����ִ������

### Condition#await��signal��signalAll
- await => ��ӦObject�е�wait����������interrupt���������׳��쳣
- signal => ��ӦObject�е�notify����
- signalAll => ��ӦObject�е�notifyAll����
- awaitUninterruptibly => ��await�������ƣ�����awaitUninterruptibly����interrupt���������ᱨ��
- awaitNanos => �ȴ�ָ����ʱ����ȥ���ѹ�����߳�
- awaitUntil => ͨ������һ��Date��ֱ���Ǹ������ڻ��ѹ�����߳�

����await���������ط���һ����await()��Զ�ȴ�������һ����await(long time, TimeUnit unit)������ֻ��ʱ���ʱ�䵥λ���ṩ������ʱ�䷶Χ

**ע�⣺**
- ���з���������ͬ���������ʹ�ã������׳�IllegalMonitorStateException�쳣
- ����signal��signalAll�����Ժ󣬱�������̲߳���������ȡ�������ǵȴ�signal��signalAll�˳�ͬ������� 

### ʵ������
**await���������ࣺAwaitTask.java**
```
public class AwaitTask implements Runnable {
    /**
     * ��
     * */
    private Lock lock;

    private Condition condition;

    /**
     * �̱߳�ʶ��
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
**signal�����ࣺSignalTask.java**
```
public class SignalTask implements Runnable {
    /**
     * ��
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
**�����ࣺ AwaitAndSignalTest.java**
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
**ִ�н����**
```
Wake up thread after 3000 milliSecond
Thread i = 1 arose and Cost = 3002 
```
��ͬ��������ͬ��Condition��������ֻ��һ���̱߳�����

**����signalAll���������̣߳�AwaitAndSignalAllTest.java**
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

**signalAll���������ࣺ**
```
public class SignalAllTask implements Runnable {
    /**
     * ��
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
**ִ�н����**
```
Wake up thread after 3000 milliSecond
Thread i = 2 arose and Cost = 3003 
Thread i = 3 arose and Cost = 3028 
Thread i = 1 arose and Cost = 3028 
Thread i = 4 arose and Cost = 3028 
```
���й�����߳̽������ѣ�ͨ��������ִ������

��󸽣�[ʵ������]()