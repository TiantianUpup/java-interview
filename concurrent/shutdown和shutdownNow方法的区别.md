### shutdown��shutdownNow����������
- shutdown => ƽ���رգ��ȴ���������ӵ��̳߳��е�����ִ�����ڹر�
- shutdownNow => ���̹رգ�ֹͣ����ִ�е����񣬲����ض�����δִ�е�����

### shutdown��shutdownNow��������ȱ��
| �رշ��� | ��ȫ�� | ��Ӧ�� |
| :----: | :----: | :----: |
| shutdown | �� | �� |
| shutdownNow | �� | �� |
ͨ�����һ�ԱȾͿ���֪��shutdown��shutdownNow��������ȱ�㣬shutdown��Ȼ��ȫ��������Ӧ�Բ��ߣ�shutdownNow������Ȼ��Ӧ�Ըߵ�����ȫ������Ŀ��ѡ��ʹ�����ַ����ر��̳߳���Ҫ����Ȩ��

### ��μ�¼shutdownNow�����ر��̳߳�δ��ɵ�����
��ΪshutdownNow����������ִֹͣ���е������������¼δ��ɵ����񣬽����������Ķ�ʧ��ʹ��shutdownNow�����ر�������Ҫ��¼����������
- ��������δִ�е�����
- �ر�ʱ����ִ�е�����

��������δִ�е��������shutdownNow�����ͻ᷵�ء���¼�ر�ʱ����ִ�е�������Ҫ��execute�������жϴ�ʱ�̳߳��Ƿ�رգ�����ر��˽���¼��ʵ�ָù�����Ҫ��дexecute����
![](https://upload-images.jianshu.io/upload_images/9358011-c9bedc6456cc55a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Executor��ExecutorServiceΪ�ӿڣ�AbstractExecutorServcieΪʵ���ࡣ��Executor������һ��execute��������дexecute��������дһ����̳�AbstractExecutorService
**AbstractExecutorService�̳��ࣺTrackingExecutor.java**
```
public class TrackingExecutor extends AbstractExecutorService {
    private static ExecutorService es;

    /**
     * ͬ��set,���δ��ɵ�����
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
˵����
- ��дexecutor�������ʾ��Ƕ��ύ��Runnable���з�װ��ʹ��try-finally�������finally���ж��̳߳��Ƿ񱻹رգ��߳��Ƿ��жϣ����������򽫵�ǰ�����¼����
- Ϊʲô�ж��̳߳عر��Ժ������жϵ�ǰ�߳��Ƿ��ж� => ��ΪshutdownNow�����ײ���õ�����interrup����������������ǲ����жϵģ���ôshutdownNow�����Ը�����Ĺر�����Ч�ģ��������һֱִ��

**TrackingExecutorʹ�ã�TrackingExecutorService**
```
public class TrackingExecutorService {
    private  TrackingExecutor trackingExecutor = new TrackingExecutor(Executors.newFixedThreadPool(3));

    private List<Runnable> runnableList;

    public void start() {
        //���10������
        for (int i = 0; i < 5; i++) {
            trackingExecutor.execute(new Task());
        }
    }

    public void stop() {
        //���̹ر��̳߳�
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
     * �Զ�������
     * */
    private class Task implements Runnable {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            while (true) {
                //ִ��һ����
                if (System.currentTimeMillis() - start > 1000 * 60) {
                    break;
                }
            }
        }
    }
}
```
˵����
- start���� => ���̳߳����ύ����
- stop���� => �ر��̳߳ز���¼δ��ɵ�����δ��ɵ���������������
- stop������awaitTermination������ʹ�� => ����shutdownNow�������̳߳ص�ֹͣ������ҪһЩʱ�䣬��������ȴ��̳߳عرգ�����shutdownNow�ر��̳߳سɹ���÷���������true
- Task�� => �Զ����࣬ǿ��run��������ִ��һ���ӣ�Ϊ��ʹ�ر��̳߳�ʱ��������δ���

**�����ࣺTest.java**
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
ִ�н����
```
com.h2t.study.concurrent.TrackingExecutor$1@13969fbe 
com.h2t.study.concurrent.TrackingExecutor$1@6aaa5eb0 
```
**ȱ�㣺**
��¼����������Ѿ�����˵��Խ����˼�¼����Ϊû���ṩAPI�ж������ִ��״̬