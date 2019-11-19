### shutdown和shutdownNow方法的区别
- shutdown => 平缓关闭，等待所有已添加到线程池中的任务执行完在关闭
- shutdownNow => 立刻关闭，停止正在执行的任务，并返回队列中未执行的任务

### shutdown和shutdownNow方法的优缺点
| 关闭方法 | 安全性 | 响应性 |
| :----: | :----: | :----: |
| shutdown | 高 | 低 |
| shutdownNow | 低 | 高 |
通过表格一对比就可以知道shutdown和shutdownNow方法的优缺点，shutdown虽然安全，但是响应性不高，shutdownNow方法虽然响应性高但不安全，在项目中选择使用哪种方法关闭线程池需要进行权衡

### 如何记录shutdownNow方法关闭线程池未完成的任务
因为shutdownNow方法会立刻停止执行中的任务，如果不记录未完成的任务，将会造成任务的丢失。使用shutdownNow方法关闭任务需要记录两部分任务：
- 队列中尚未执行的任务
- 关闭时正在执行的任务

队列中尚未执行的任务调用shutdownNow方法就会返回。记录关闭时正在执行的任务需要在execute方法中判断此时线程池是否关闭，如果关闭了将记录，实现该功能需要重写execute方法
![](https://upload-images.jianshu.io/upload_images/9358011-c9bedc6456cc55a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Executor和ExecutorService为接口，AbstractExecutorServcie为实现类。在Executor类中有一个execute方法，重写execute方法就是写一个类继承AbstractExecutorService
**AbstractExecutorService继承类：TrackingExecutor.java**
```
public class TrackingExecutor extends AbstractExecutorService {
    private static ExecutorService es;

    /**
     * 同步set,存放未完成的任务
     * */
    private static Set<Runnable> tasksCancelledAtShutdown = Collections.synchronizedSet(new HashSet<>());

    public TrackingExecutor(ExecutorService es) {
        this.es = es;
    }

    public List<Runnable> getCancelledTasks() {
        if (!es.isTerminated()) {
            throw new IllegalStateException();
        }

        return new ArrayList<>(tasksCancelledAtShutdown);
    }

    @Override
    public void execute(Runnable command) {
        es.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    command.run();
                } finally {
                    if (isShutdown() && Thread.currentThread().isInterrupted()) {
                        tasksCancelledAtShutdown.add(command);
                    }
                }
            }
        });
    }


    @Override
    public void shutdown() {
        es.shutdownNow();
    }

    @Override
    public List<Runnable> shutdownNow() {
        return es.shutdownNow();
    }

    @Override
    public boolean isShutdown() {
        return es.isShutdown();
    }

    @Override
    public boolean isTerminated() {
        return es.isTerminated();
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return es.awaitTermination(timeout, unit);
    }
}
```
说明：
- 重写executor方法本质就是对提交的Runnable进行封装，使用try-finally代码块在finally中判断线程池是否被关闭，线程是否被中断，条件成立则将当前任务记录下来
- 为什么判断线程池关闭以后仍需判断当前线程是否中断 => 因为shutdownNow方法底层调用的仍是interrup方法，如果该任务是不可中断的，那么shutdownNow方法对该任务的关闭是无效的，该任务会一直执行

**TrackingExecutor使用：TrackingExecutorService**
```
public class TrackingExecutorService {
    private  TrackingExecutor trackingExecutor = new TrackingExecutor(Executors.newFixedThreadPool(3));

    private List<Runnable> runnableList;

    public void start() {
        //添加10个任务
        for (int i = 0; i < 5; i++) {
            trackingExecutor.execute(new Task());
        }
    }

    public void stop() {
        //立刻关闭线程池
        runnableList = trackingExecutor.shutdownNow();
        try {
            if (trackingExecutor.awaitTermination(30, TimeUnit.SECONDS)) {
                for (Runnable runnable : trackingExecutor.getCancelledTasks()) {
                    runnableList.add(runnable);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public List<Runnable> getRunnableList() {
        return runnableList;
    }

    /**
     * 自定义任务
     * */
    private class Task implements Runnable {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            while (true) {
                //执行一分钟
                if (System.currentTimeMillis() - start > 1000 * 60) {
                    break;
                }
            }
        }
    }
}
```
说明：
- start方法 => 往线程池中提交任务
- stop方法 => 关闭线程池并记录未完成的任务，未完成的任务来自两部分
- stop方法中awaitTermination方法的使用 => 调用shutdownNow方法后线程池的停止可能需要一些时间，因此阻塞等待线程池关闭，调用shutdownNow关闭线程池成功后该方法将返回true
- Task类 => 自定义类，强制run方法至少执行一分钟，为了使关闭线程池时仍有任务未完成

**测试类：Test.java**
```
public class Test {
    public static void main(String[] args) {
        TrackingExecutorService trackingExecutorService = new TrackingExecutorService();
        trackingExecutorService.start();
        trackingExecutorService.stop();
        trackingExecutorService.getRunnableList().stream().forEach(i -> System.out.println("Runnable unfinished: " + i));
    }
}
```
执行结果：
```
com.h2t.study.concurrent.TrackingExecutor$1@13969fbe 
com.h2t.study.concurrent.TrackingExecutor$1@6aaa5eb0 
```
**缺点：**
记录的任务可能已经完成了但仍进行了记录，因为没有提供API判断任务的执行状态