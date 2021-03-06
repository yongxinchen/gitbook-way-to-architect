# 如何在数据库事务提交成功后进行异步操作

需求：在事务成功后，执行另外一个异步操作

场景：在记录用户操作记录时，如果用户操作成功（提交事务后），则进行记录，如果操作失败，则不记录，且记录操作可能会涉及数据对比等耗时操作，要异步进行。

#### 使用TransactionSynchronizationManager在事务提交之后操作 {#articleHeader7}

```java
public void add(Book book){
    bookMapper.insert(book);
    // send after tx commit but is async
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            System.out.println("add operation log after transaction commit...");
        }
    }
   );
}
```

该方法就可以实现在事务提交之后进行操作。

#### 操作异步化 {#articleHeader8}

使用线程池来进行异步：

```java
private final ExecutorService executorService = Executors.newFixedThreadPool(5);
public void insert(Book book){
    bookMapper.insert(book);
    //send after tx commit but is async
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    System.out.println("send email after transaction commit...");
                }
            });
        }
    }
    );
}
```

#### 封装

```java
//声明接口，继承Executor，并定义需要在事务提交后要做的事情
public interface AfterCommitExecutor  extends Executor {
    /**
     * 操作记录
     * @param record
     */
    void record(OperationRecord record);
}
//实现
@Named
public class AfterCommitExecutorImpl extends TransactionSynchronizationAdapter implements AfterCommitExecutor {
    private static Logger logger = LoggerFactory.getLogger(AfterCommitExecutorImpl.class);
    private static ThreadLocal<List<Runnable>> RUNNABLES = new ThreadLocal<List<Runnable>>();
    private ExecutorService threadPool = Executors.newFixedThreadPool(3);

    @Inject
    private OperationRecordService recordService;

    @Override
    public void execute(Runnable runnable) {
        if (!TransactionSynchronizationManager.isSynchronizationActive()) {
            runnable.run();
            return;
        }
        List<Runnable> threadRunnables = RUNNABLES.get();
        if (threadRunnables == null) {
            threadRunnables = new ArrayList<Runnable>();
            RUNNABLES.set(threadRunnables);
            TransactionSynchronizationManager.registerSynchronization(this);
        }
        threadRunnables.add(runnable);
    }

    @Override
    public void record(OperationRecord record) {
        this.execute(() -> {
            recordService.add(record);
        });
    }

    @Override
    public void afterCommit() {
        List<Runnable> threadRunnables = RUNNABLES.get();
        logger.info("Transaction successfully committed, executing {} runnables", threadRunnables.size());
        for (int i = 0; i < threadRunnables.size(); i++) {
            Runnable runnable = threadRunnables.get(i);
            logger.info("Executing runnable {}", runnable);
            try {
                threadPool.execute(runnable);
            } catch (RuntimeException e) {
                logger.error("Failed to execute runnable " + runnable, e);
            }
        }
    }
    @Override
    public void afterCompletion(int status) {
        logger.info("Transaction completed with status {}", status == STATUS_COMMITTED ? "COMMITTED" : "ROLLED_BACK");
        RUNNABLES.remove();
    }
}
//使用
@Named
public class BookServiceImpl extends BookService{
    @Inject
    private AfterCommitExecutor afterCommitExecutor;
    @Override
    @Transactional
    public void add(Book book){
        bookMapper.insert(book);
        afterCommitExecutor.record(new OperationRecord());
    }
}
```

 

## 内容来源

  
[如何在数据库事务提交成功后进行异步操作](https://segmentfault.com/a/1190000004235193)

[Transaction synchronization callbacks in Spring Framework](http://azagorneanu.blogspot.com/2013/06/transaction-synchronization-callbacks.html)



