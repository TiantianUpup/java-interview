### 使用标志符号
通过使用一个标志，比如一个布尔类型的flag变量，当flag为true时，则取消正在运行的任务
实例：
```
public class FlagInterrupt {
    /**
     * 是否停止任务标志位
     * */
    private volatile boolean flag = false;


    public void stop() {
        flag = true;
    }

    public void start() {
        new Thread(new Task()).start();
    }

    private class Task implements Runnable {
        @Override
        public void run() {
            int i = 0;
            //如果任务不停止一直累加打印结果
            while (!flag) {
                System.out.println(i++);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        FlagInterrupt flagInterrupt = new FlagInterrupt();
        flagInterrupt.start();  //启动任务
        Thread.sleep(10 * 1000);
        flagInterrupt.stop();  //执行10s停止执行的任务
    }
}
```
说明：
- start => 启动任务，并且通过一个flag变量判断是否需要继续执行任务
- stop => 取消任务，将flag变量置为true

**该方法的优缺点：**
- 优点
  - 使用简单
- 缺点
  - 响应性低，轮询检查一定时间
  - 如果任务中调用了一个阻塞方法标志位的变化将永远检查不到

    ```
    private class MyTask implements Runnable {
        @Override
        public void run() {
            while (!flag) {
                try {
                    //队列中无值，将被阻塞
                    arrayBlockingQueue.take();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    ```
    如果run方法修改为如上所示，该任务将永远取消不了。因此使用标志来取消任务的执行只适合非阻塞任务，针对**阻塞任务**的取消可以使用**中断**

### 使用中断
一般来说，**中断**是取消任务**最合理**的方式。在java中通过使用interrupt方法来实现中断，通知线程在合适或者可能的情况下停止当前任务。中断方法只适合**终止被阻塞的任务**
run方法中调用了阻塞方法该任务就会变成一个阻塞任务，比如：
- 调用sleep方法，在指定时间内为阻塞任务
- 调用wait方法，在线程得到notify或notifyAll方法之前为阻塞任务
- 调用阻塞队列中的put、take方法
- 等待输入\输出的完成
- 调用synchronized方法但获取不到锁

##### 实战测试
**sleep方法阻塞响应中断：SleepBlock.java**
```
public class SleepBlock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);  //睡眠3s
        thread.interrupt();
    }

    public static class Task implements Runnable {
        /**
         * 自定义任务
         * */
        @Override
        public void run() {
            try {
                //睡眠足够长的时间，在测试时间内不会醒过来
                Thread.sleep(1000 * 60 * 60 * 24);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("End...");
        }
    }
}
```
run方法中睡眠足够长时间，在main线程中调用interrupt方法取消任务
执行结果：
```
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.h2t.study.concurrent.interrupt.SleepBlock$Task.run(SleepBlock.java:26)
	at java.lang.Thread.run(Thread.java:748)
End...
```
成功中断，并打印了end结果

**wait方法阻塞响应中断：WaitBlock.java**
```
public class WaitBlock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);
        thread.interrupt();
    }

    public static class Task implements Runnable {
        @Override
        public void run() {

            //睡眠足够长的时间，在测试时间内不会醒过来
            Test test = new Test();
            test.waitMethod();
            System.out.println("End...");
        }
    }

    private static class Test {
        public synchronized void waitMethod() {
            try {
                wait(1000 * 60 * 60 * 24);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
执行逻辑和SleepBlock类差不多，执行结果：
```
java.lang.InterruptedException
	at java.lang.Object.wait(Native Method)
	at com.h2t.study.concurrent.interrupt.WaitBlock$Test.waitMethod(WaitBlock.java:32)
	at com.h2t.study.concurrent.interrupt.WaitBlock$Task.run(WaitBlock.java:24)
	at java.lang.Thread.run(Thread.java:748)
End...
```
成功取消了任务的执行，因此能响应中断

**阻塞队列响应中断：BlockList.java**
```
public class BlockList {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);  //睡眠3s
        thread.interrupt();
    }

    public static class Task implements Runnable {
        ArrayBlockingQueue<Integer> arrayBlockingQueue = new ArrayBlockingQueue<>(10);

        /**
         * 自定义任务
         */
        @Override
        public void run() {
            try {
                //一直阻塞，因为阻塞队列中没有元素
                arrayBlockingQueue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("End...");
        }
    }
}
```
arrayBlockingQueue队列中没有添加任何元素，因此take方法将阻塞
执行结果：
```
End...
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.reportInterruptAfterWait(AbstractQueuedSynchronizer.java:2014)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2048)
	at java.util.concurrent.ArrayBlockingQueue.take(ArrayBlockingQueue.java:403)
	at com.h2t.study.concurrent.interrupt.BlockList$Task.run(BlockList.java:30)
	at java.lang.Thread.run(Thread.java:748)
```
成功响应了中断

**调用synchronized方法出现死锁响应中断：SynchronizedBlock.java**
```
public class SynchronizedBlock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task());
        thread.start();
        Thread.sleep(3 * 1000);  //执行三秒后中断线程
        thread.interrupt();
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
            while (true) {}
        }

        @Override
        public void run() {
            f();
            System.out.println("End");
        }
    }
}
```
f方法中需要获得锁，获得锁之后通过死循环模拟一直不释放锁的情景。run方法调用f方法，并且Task类的构造函数中使用不同的线程调用f方法。因此main方法中，thread线程获取不到锁，控制台将打印不出End...因为该中断是不可响应的

**等待输入输出响应中断：IOBlock.java**
```
public class IOBlock {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Task(System.in));  //输入流的来源来自用户输入
        thread.start();
        Thread.sleep(3 * 1000);  //睡眠3s
        thread.interrupt();
        System.out.println("是否被中断：" + thread.isInterrupted());
        System.out.println("中断已执行");
    }

    public static class Task implements Runnable {
        private InputStream inputStream;

        public Task(InputStream inputStream) {
            this.inputStream = inputStream;
        }

        /**
         * 自定义任务
         */
        @Override
        public void run() {
            try {
                inputStream.read();
            } catch (IOException e) {
                e.printStackTrace();
            }
            System.out.println("End...");
        }
    }
}
```
run方法中从一个输入流中读取内容，测试中输入流为System.in即用户从控制台中输入。如果控制台一直不输入内容将一直等待，此时调用中断方法取消任务将无效，即不会在控制台打印出End...结果

##### 总结
并不是所有阻塞方法都能响应中断，根据执行结果来看，准确的说应该是能**中断能抛出InterruptedException异常的阻塞方法**

### 不可响应中断的阻塞方法如何取消任务
**IO阻塞**
- 关闭底层资源，例如demo中使用close方法先关闭inputStream在调用interrupt方法

  ```
  public class IOBlockInterrupt {
      public static void main(String[] args) {
          MyThread thread = new MyThread(System.in);  //输入流的来源来自用户输入
          thread.start();
          thread.interrupt();
      }

      private static class MyThread extends Thread {
          private InputStream inputStream;

          public MyThread(InputStream inputStream) {
              this.inputStream = inputStream;
          }

          //重写interrupt方法
          @Override
          public void interrupt() {
              //1.关闭流
              try {
                  inputStream.close();
              } catch (IOException e) {
                  e.printStackTrace();
              } finally {
                  //2.调用中断方法
                  super.interrupt();
              }
          }

          @Override
          public void run() {
              try {
                  inputStream.read();
              } catch (IOException e) {
                  e.printStackTrace();
              }
              System.out.println("End...");
          }
      }
  }
  ```
  通过继承Thread重写interrupt方法，在interrupt方法中先关闭流，在调用中断
  执行结果：
  ```
  End...
  java.io.IOException: Stream closed
	  at java.io.BufferedInputStream.getBufIfOpen(BufferedInputStream.java:170)
	  at java.io.BufferedInputStream.fill(BufferedInputStream.java:214)
	  at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
	  at   com.h2t.study.concurrent.interrupt.IOBlockInterrupt$MyThread.run(IOBlockInterrupt.java:45)
  ```

- 使用NIO，被阻塞的NIO会自动响应中断  

**synchronize同步方法阻塞**
- 使用可响应中断的ReentrantLock锁

### 线程池取消任务
一般需要使用线程的场景会选择使用线程池，线程池调用shutdownNow方法，将会发送一个interrupt
```
public class ShutdownNowInterrupt {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService es = Executors.newCachedThreadPool();
        es.submit(new Task2());
        es.submit(new Task(555));
        Thread.sleep(3 * 1000);
        es.shutdownNow();  //3s后关闭线程
    }

    private static class Task2 implements Runnable {
        @Override
        public void run() {
            try {
                Thread.sleep(1000 * 60 * 60 * 24);  //睡眠一天
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("Task2 End");
        }
    }

    private static class Task implements Runnable {
        int i;

        public Task(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            long start = System.currentTimeMillis();
            while (true) {
                //执行一分钟
                if (System.currentTimeMillis() - start > 1000 * 60 * 1) {
                    break;
                }
            }

            System.out.println("Task End i= " + i);
        }
    }
}
```
打印结果：
```
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.h2t.study.concurrent.interrupt.ShutdownNowInterrupt$Task2.run(ShutdownNowInterrupt.java:26)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
	at java.util.concurrent.FutureTask.run(FutureTask.java)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Task2 End
Disconnected from the target VM, address: '127.0.0.1:50605', transport: 'socket'
Task End i= 555
```
**注意：**
![](https://upload-images.jianshu.io/upload_images/9358011-edf5c26be6afb8ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
查看shutdownNow的源码发现，本质调用的还是interrupt方法，因此上面的Task任务无法响应中断将会继续执行打印最后的执行结果555
### 使用Future
shutdownNow将取消线程池中的所有任务，有些时候只是想取消某一个任务，这时候可以使用Future来取消任务。当使用线程池的submit方法提交任务后会返回一个Future对象，使用该对象的cancel方法可以来取消任务  
使用姿势如下：
```
ExecutorService es = Executors.newCachedThreadPool();
Future<?> ioFuture = es.submit(new IOTask(System.in));
ioFuture.cancel(true);
```
cancel方法参数说明：
- true =>  可以取消正在执行的任务
- false => 不可以取消正在执行的任务

**注意：**
![](https://upload-images.jianshu.io/upload_images/9358011-af47acf6920d3ceb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
调用cancel方法底层仍使用的interrupt方法，因此使用cancel方法只能中断能抛出InterruptedException异常的阻塞方法

### 任务的取消有什么用
- 超时取消 => 如果一个任务没有在指定的时间内完成就取消当前任务，不会影响其他任务的执行

![](https://upload-images.jianshu.io/upload_images/9358011-5542c486cdb97212.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**最后附：**[实例代码地址](https://github.com/TiantianUpup/java-learning/tree/master/src/main/java/com/h2t/study/concurrent/interrupt)，欢迎**fork**和**star**