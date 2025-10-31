**1.可重入锁源码之lua脚本加锁逻辑**

**2.可重入锁源码之WatchDog维持加锁逻辑**

**3.可重入锁源码之可重入加锁逻辑**

**4.可重入锁源码之锁的互斥阻塞逻辑**

**5.可重入锁源码之释放锁逻辑**

**6.可重入锁源码之获取锁超时与锁超时自动释放逻辑**

**7.可重入锁源码总结**

**8.****公平锁源码之加锁和排队**

**9.****公平锁****源****码之可重入加锁**

**10.****公平锁源码之队列重排**

**11.****公平锁源码之释放锁**

**12.****公平锁源码之按顺序依次加锁**

**
**

**1.可重入锁源码之lua脚本加锁逻辑**

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    //默认情况下，外部传入的leaseTime=-1时，会取lockWatchdogTimeout的默认值=30秒    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,            "if (redis.call('exists', KEYS[1]) == 0) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "return redis.call('pttl', KEYS[1]);",            Collections.singletonList(getRawName()),//锁的名字：KEYS[1]，比如"myLock"            unit.toMillis(leaseTime),//过期时间：ARGV[1]，默认时为30秒            getLockName(threadId)//ARGV[2]，值为UUID + 线程ID        );    }    ...}
```

首先执行Redis的exists命令，判断key为锁名的Hash值是否不存在，也就是判断key为锁名myLock的Hash值是否存在。



**一.如果key为锁名的Hash值不存在，那么就进行如下加锁处理**

首先通过Redis的hset命令设置一个key为锁名的Hash值。该Hash值的key为锁名，value是一个映射。也就是在value值中会有一个field为UUID + 线程ID，value为1的映射。比如：hset myLock UUID:ThreadID 1，lua脚本中的ARGV[2]就是由UUID + 线程ID组成的唯一值。



然后通过Redis的pexpire命令设置key为锁名的Hash值的过期时间，也就是设置key为锁名的Hash值的过期时间为30秒。比如：pexpire myLock 30000。所以默认情况下，myLock这个锁在30秒后就会自动过期。



**二.如果key为锁名的Hash值存在，那么就执行如下判断处理**

首先通过Redis的hexists命令判断在key为锁名的Hash值里，field为UUID + 线程ID的映射是否已经存在。



如果在key为锁名的Hash值里，field为UUID + 线程ID的映射存在，那么就通过Redis的hincrby命令，对field为UUID + 线程ID的value值进行递增1。比如：hincrby myLock UUID:ThreadID 1。也就是在key为myLock的Hash值里，把field为UUID:ThreadID的value值从1累加到2，发生这种情况的时候往往就是当前线程对锁进行了重入。接着执行：pexpire myLock 30000，再次将myLock的有效期设置为30秒。



如果在key为锁名的Hash值里，field为UUID + 线程ID的映射不存在，发生这种情况的时候往往就是其他线程获取不到这把锁而产生互斥。那么就通过Redis的pttl命令，返回key为锁名的Hash值的剩余存活时间，因为不同线程的ARGV[2]是不一样的，ARGV[2] = UUID + 线程ID。

**
**

***\*2.可重入锁源码之WatchDog维持加锁逻辑\****

如果某个客户端上锁后，过了几分钟都没释放掉这个锁，而锁对应的key一开始的过期时间其实只设置了30秒而已，那么在这种场景下这个锁就不能在上锁的30秒后被自动释放。



RedissonLock中有一个WatchDog机制：当客户端对myLock这个key成功加锁后，就会创建一个定时任务，这个定时任务会默认10秒后更新myLock这个key的过期时间为30秒。当然，前提是客户端一直持有myLock这个锁，还没有对锁进行释放。只要客户端一直持有myLock这个锁没有对其进行释放，那么就会不断创建一个定时任务，10秒后更新myLock这个key的过期时间。



那么加锁成功后的定时任务，是如何每隔10秒去更新锁的过期时间的。

**
**

**(1)异步执行完加锁的lua脚本会触发****执行thenApply()里的回调**

异步执行完RedissonLock.tryLockInnerAsync()方法里的加锁lua脚本内容后，会触发执行ttlRemainingFuture的thenApply()方法里的回调。其中加锁lua脚本会返回Long型的ttlRemaining变量，ttlRemainingFuture的thenApply()方法里的回调入参便是这个变量。



具体会先在RedissonLock的tryAcquireAsync()方法中，定义一个RFuture类型的名为ttlRemainingFuture的变量。这个变量封装了当前这个锁对应的key的剩余存活时间，单位毫秒，这个剩余存活时间是在执行完加锁lua脚本时返回的。



然后通过RFuture的thenApply()方法给ttlRemainingFuture添加一个回调。这样当lua脚本执行完成后，就会触发ttlRemainingFuture的thenApply()方法里的回调，而回调里的入参ttlRemaining就是执行加锁lua脚本的返回值。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    protected long internalLockLeaseTime;    protected final LockPubSub pubSub;    final CommandAsyncExecutor commandExecutor;        public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {        super(commandExecutor, name);        this.commandExecutor = commandExecutor;        //与WatchDog有关的internalLockLeaseTime        //通过命令执行器CommandExecutor可以获取连接管理器ConnectionManager        //通过连接管理器ConnectionManager可以获取Redis的配置信息类Config        //通过Redis的配置信息类Config可以获取lockWatchdogTimeout超时时间        this.internalLockLeaseTime = commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout();        this.pubSub = commandExecutor.getConnectionManager().getSubscribeService().getLockPubSub();    }        //加锁    @Override    public void lock() {        try {            lock(-1, null, false);        } catch (InterruptedException e) {            throw new IllegalStateException();        }    }    ...
    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {        //线程ID，用来生成设置Hash的值        long threadId = Thread.currentThread().getId();        //尝试加锁，此时执行RedissonLock.lock()方法默认传入的leaseTime=-1        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);        //ttl为null说明加锁成功        if (ttl == null) {            return;        }              //加锁失败时的处理        ...    }    ...
    private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        //默认下waitTime和leaseTime都是-1，下面调用的get方法是来自于RedissonObject的get()方法        //可以理解为异步转同步：将异步的tryAcquireAsync通过get转同步        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));    }        private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        RFuture<Long> ttlRemainingFuture;        if (leaseTime != -1) {            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);        } else {            //默认情况下，由于leaseTime=-1，所以会使用初始化RedissonLock实例时的internalLockLeaseTime            //internalLockLeaseTime的默认值就是lockWatchdogTimeout的默认值，30秒            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime, TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);        }
        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {            //异步执行完tryLockInnerAsync()里加锁lua脚本内容后            //就会触发执行ttlRemainingFuture.thenApply()里的回调            //加锁返回的ttlRemaining为null表示加锁成功            if (ttlRemaining == null) {                if (leaseTime != -1) {                    internalLockLeaseTime = unit.toMillis(leaseTime);                } else {                    //传入加锁成功的线程ID，启动一个定时任务                    scheduleExpirationRenewal(threadId);                }            }            return ttlRemaining;        });        return new CompletableFutureWrapper<>(f);    }        //默认情况下，外部传入的leaseTime=-1时，会取lockWatchdogTimeout的默认值=30秒    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,            "if (redis.call('exists', KEYS[1]) == 0) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "return redis.call('pttl', KEYS[1]);",            Collections.singletonList(getRawName()),//锁的名字：KEYS[1]            unit.toMillis(leaseTime),//过期时间：ARGV[1]，默认时为30秒            getLockName(threadId)//ARGV[2]，值为UUID + 线程ID        );    }    ...}
```

**(2)根据执行加锁lua脚本的返回值来决定是否创建定时调度任务**

当执行加锁lua脚本的返回值ttlRemaining为null时，则表明锁获取成功。



如果锁获取成功，且指定锁的过期时间，即leaseTime不是默认的-1，那么此时并不会创建定时任务。如果锁获取成功，且没指定锁的过期时间，即leaseTime是默认的-1，那么此时才会创建定时调度任务，并且是根据Netty的时间轮来实现创建。



所以调用scheduleExpirationRenewal()方法会创建一个定时调度任务，只要锁还被客户端持有，定时任务就会不断更新锁对应的key的过期时间。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    ...    private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        RFuture<Long> ttlRemainingFuture;        if (leaseTime != -1) {            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);        } else {            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime, TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);        }
        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {            //异步执行完tryLockInnerAsync()里的内容后，就会执行ttlRemainingFuture.thenApply()里的逻辑            //加锁返回的ttlRemaining为null表示加锁成功            if (ttlRemaining == null) {                if (leaseTime != -1) {                    //如果指定了锁的过期时间，并不会启动定时任务                    internalLockLeaseTime = unit.toMillis(leaseTime);                } else {                    //传入加锁成功的线程ID，启动一个定时任务                    scheduleExpirationRenewal(threadId);                }            }            return ttlRemaining;        });        return new CompletableFutureWrapper<>(f);    }    ...}
```

***\*(3)定时调度任务由Netty的时间轮机制来实现\****

Redisson的WatchDog机制底层并不是调度线程池，而是Netty时间轮。



scheduleExpirationRenewal()方法会创建一个定时调度任务TimerTask交给HashedWheelTimer，10秒后执行。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public abstract class RedissonBaseLock extends RedissonExpirable implements RLock {    ...    protected void scheduleExpirationRenewal(long threadId) {        ExpirationEntry entry = new ExpirationEntry();        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);        if (oldEntry != null) {            oldEntry.addThreadId(threadId);        } else {            entry.addThreadId(threadId);            try {                //创建一个更新过期时间的定时调度任务                renewExpiration();            } finally {                if (Thread.currentThread().isInterrupted()) {                    cancelExpirationRenewal(threadId);                }            }        }    }        //更新过期时间    private void renewExpiration() {        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());        if (ee == null) {            return;        }
        //使用了Netty的定时任务机制：HashedWheelTimer + TimerTask + Timeout        //创建一个更新过期时间的定时调度任务，下面会调用MasterSlaveConnectionManager.newTimeout()方法        //即创建一个定时调度任务TimerTask交给HashedWheelTimer，10秒后执行        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {            @Override            public void run(Timeout timeout) throws Exception {                ...            }        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);        ee.setTimeout(task);    }    ...}
public class MasterSlaveConnectionManager implements ConnectionManager {    private HashedWheelTimer timer;//Netty的时间轮    ...
    @Override    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {        try {            //给Netty的时间轮添加任务task            return timer.newTimeout(task, delay, unit);        } catch (IllegalStateException e) {            if (isShuttingDown()) {                return DUMMY_TIMEOUT;            }            throw e;        }    }    ...}
//Netty的时间轮HashedWheelTimerpublic class HashedWheelTimer implements Timer {    private final HashedWheelBucket[] wheel;    private final Queue<HashedWheelTimeout> timeouts = PlatformDependent.newMpscQueue();    private final Executor taskExecutor;    ...
    @Override    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {        ...        //启动Worker线程        start();        ...        //封装定时调度任务TimerTask实例到HashedWheelTimeout实例        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);        timeouts.add(timeout);        return timeout;    }    ...
    private final class Worker implements Runnable {        private long tick;//轮次        ...
        @Override        public void run() {            ...            do {                final long deadline = waitForNextTick();                if (deadline > 0) {                    int idx = (int) (tick & mask);                    processCancelledTasks();                    HashedWheelBucket bucket = wheel[idx];                    //将定时调度任务HashedWheelTimeout转换到时间轮分桶里                    transferTimeoutsToBuckets();                    //处理时间轮分桶HashedWheelBucket里的到期任务                    bucket.expireTimeouts(deadline);                    tick++;                }            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);            ...        }                private void transferTimeoutsToBuckets() {            for (int i = 0; i < 100000; i++) {                HashedWheelTimeout timeout = timeouts.poll();                ...                long calculated = timeout.deadline / tickDuration;                timeout.remainingRounds = (calculated - tick) / wheel.length;                final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.                int stopIndex = (int) (ticks & mask);//对mask取模                HashedWheelBucket bucket = wheel[stopIndex];                bucket.addTimeout(timeout);            }        }        ...    }        private static final class HashedWheelBucket {        ...        public void expireTimeouts(long deadline) {            HashedWheelTimeout timeout = head;            //process all timeouts            while (timeout != null) {                ...                //对定时调度任务timeout进行过期处理                timeout.expire();                ...            }        }        ...    }        private static final class HashedWheelTimeout implements Timeout, Runnable {        private final HashedWheelTimer timer;        ...
        public void expire() {            ...            //执行定时调度任务，让Executor执行HashedWheelTimeout的run方法            timer.taskExecutor.execute(this);            ...        }                @Override        public void run() {            ...            //执行定时调度任务            task.run(this);            ...        }        ...    }    ...}
```

***\*(4)10秒后会执行定时任务并判断是否要再创建一个定时任务在10秒后执行\****

该定时任务TimerTask实例会调用RedissonBaseLock的renewExpirationAsync()方法去执行lua脚本，renewExpirationAsync()方法会传入获取到锁的线程ID。



在lua脚本中，会通过Redis的hexists命令判断在key为锁名的Hash值里，field为UUID + 线程ID的映射是否存在。其中KEYS[1]就是锁的名字如myLock，ARGV[2]为UUID + 线程ID。



如果存在，说明获取锁的线程还在持有锁，并没有对锁进行释放。那么就通过Redis的pexpire命令，设置key的过期时间为30秒。



异步执行lua脚本完后，会传入布尔值触发future.whenComplete()方法的回调，回调中会根据布尔值来决定是否继续递归调用renewExpiration()方法。



也就是说，如果获取锁的线程还在持有锁，那么就重置锁的过期时间为30秒，并且lua脚本会返回1，接着在future.whenComplete()方法的回调中，继续调用RedissonBaseLock的renewExpiration()方法重新创建定时调度任务。



如果获取锁的线程已经释放了锁，那么lua脚本就会返回0，接着在future.whenComplete()方法的回调中，会调用RedissonBaseLock的cancelExpirationRenewal()方法执行清理工作。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public abstract class RedissonBaseLock extends RedissonExpirable implements RLock {    ...    protected void scheduleExpirationRenewal(long threadId) {        ExpirationEntry entry = new ExpirationEntry();        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);        if (oldEntry != null) {            oldEntry.addThreadId(threadId);        } else {            entry.addThreadId(threadId);            try {                //创建一个更新过期时间的定时调度任务                renewExpiration();            } finally {                if (Thread.currentThread().isInterrupted()) {                    cancelExpirationRenewal(threadId);                }            }        }    }        //更新过期时间    private void renewExpiration() {        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());        if (ee == null) {            return;        }
        //使用了Netty的定时任务机制：HashedWheelTimer + TimerTask + Timeout        //创建一个更新过期时间的定时调度任务，下面会调用MasterSlaveConnectionManager.newTimeout()方法        //即创建一个定时调度任务TimerTask交给HashedWheelTimer，10秒后执行        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {            @Override            public void run(Timeout timeout) throws Exception {                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());                if (ent == null) {                    return;                }
                Long threadId = ent.getFirstThreadId();                if (threadId == null) {                    return;                }
                //异步执行lua脚本去更新锁的过期时间                RFuture<Boolean> future = renewExpirationAsync(threadId);                future.whenComplete((res, e) -> {                    if (e != null) {                        log.error("Can't update lock " + getRawName() + " expiration", e);                        EXPIRATION_RENEWAL_MAP.remove(getEntryName());                        return;                    }                    //res就是执行renewExpirationAsync()里的lua脚本的返回值                    if (res) {                        //重新调度自己                        renewExpiration();                    } else {                        //执行清理工作                        cancelExpirationRenewal(null);                    }                });            }        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);        ee.setTimeout(task);    }        protected RFuture<Boolean> renewExpirationAsync(long threadId) {        //其中KEYS[1]就是锁的名字如myLock，ARGV[2]为UUID+线程ID；        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return 1; " +            "end; " +            "return 0;",            Collections.singletonList(getRawName()),            internalLockLeaseTime, getLockName(threadId));        }    }        protected void cancelExpirationRenewal(Long threadId) {        ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());        if (task == null) {            return;        }
        if (threadId != null) {            task.removeThreadId(threadId);        }        if (threadId == null || task.hasNoThreads()) {            Timeout timeout = task.getTimeout();            if (timeout != null) {                timeout.cancel();            }            EXPIRATION_RENEWAL_MAP.remove(getEntryName());        }    }    ...}
```

注意：如果持有锁的机器宕机了，就会导致该机器上的锁的WatchDog不执行了。从而锁的key会在30秒内自动过期，释放掉锁。此时其他客户端最多再等30秒就可获取到这把锁了。

**
**

**3.可****重入锁源码之可重入加锁逻辑**

实现可重入加锁的核心就是加锁的lua脚本。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class Application {    public static void main(String[] args) throws Exception {        Config config = new Config();        config.useClusterServers().addNodeAddress("redis://192.168.1.110:7001");        //创建RedissonClient实例        RedissonClient redisson = Redisson.create(config);        //获取可重入锁        RLock lock = redisson.getLock("myLock");        //首次加锁        lock.lock();        //重入加锁        lock.lock();        //重入加锁        lock.lock();        lock.unlock();        lock.unlock();        lock.unlock();        ...    }}
public class RedissonLock extends RedissonBaseLock {    ...    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,            "if (redis.call('exists', KEYS[1]) == 0) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "return redis.call('pttl', KEYS[1]);",            Collections.singletonList(getRawName()),//锁的名字：KEYS[1]            unit.toMillis(leaseTime),//过期时间：ARGV[1]，默认时为30秒            getLockName(threadId)//ARGV[2]，值为UUID + 线程ID        );    }    ...}
```

**(1)核心一是key的命名：****第一层key是锁名，第二层field是UUID+线程ID**

KEYS[1]的值就是锁的名字，ARGV[2]的值就是客户端UUID + 线程ID。由于第二层field包含了线程ID，所以可以区分申请加锁的线程是否在重入。对在key为锁名的Hash值中，field为UUID + 线程ID的value值进行递增1操作。

**
**

**(2)核心二是****Redis****的exists命令、hexists命令、hincrby命令**

判断锁是否存在，使用的是Redis的exists命令。判断锁是否正在被线程重入，则使用的是Redis的hexists命令。对field映射的value值进行递增，使用的是Redis的hincrby命令。



所以，持有锁的线程每重入锁一次，就会对field映射的value值递增1。比如：hincrby key UUID:ThreadID 1。

**
**

**4.可重入锁源码之锁的互斥阻塞逻辑**

**(1)获取锁失败时会执行Redis的pttl命令返回锁的剩余存活时间**

在执行加锁的lua脚本中：首先通过"exists myLock"，判断是否存在myLock这个锁key，发现已经存在。接着判断"hexists myLock 另一个UUID:另一个Thread的ID"，发现不存在。于是执行"pttl myLock"并返回，也就是返回myLock这个锁的剩余存活时间。



**(2)ttlRemainingFuture的回调方法发现****ttlRemaining不是null**

当执行加锁的lua脚本发现加锁不成功，返回锁的剩余存活时间时，ttlRemainingFuture的回调方法发现ttlRemaining不是null，于是便不会创建定时调度任务在10秒之后去检查锁是否还被申请的线程持有，而是会将返回的剩余存活时间封装到RFuture中并向上返回。

**
**

**(3)RedissonLock的tryAcquire()方法会返回这个剩余存活时间ttl**

主要会通过RedisObject的get()方法去获取RFuture中封装的ttl值。其中异步转同步就是将异步的tryAcquireAsync()方法通过get()方法转同步。

**
**

**(4)RedissonLock的lock()方法会对****tryAcquire()方法返回的ttl进行判断**

如果当前线程是第一次加锁，那么ttl一定是null。如果当前线程是多次加锁(可重入锁)，那么ttl也一定是null。如果当前线程加锁没成功，锁被其他机器或线程占用，那么ttl就是执行加锁lua脚本时获取到的锁对应的key的剩余存活时间。



如果ttl是null，说明当前线程加锁成功，于是直接返回。如果ttl不是null，说明当前线程加锁不成功，于是会执行阻塞逻辑。

**
**

**(5)RedissonLock的lock()方法在****ttl不为null时利用Semaphore进行阻塞**

如果ttl不为null，即加锁不成功，那么会进入while(true)死循环。在死循环内，再次执行RedissonLock的tryAcquire()方法尝试获取锁。如果再次获取锁时返回的ttl为null，即获取到锁，就退出while循环。如果再次获取锁时返回的ttl不为null，则说明其他客户端或线程还持有锁。



于是就利用同步组件Semaphore进行阻塞等待一段ttl的时间。阻塞等待一段时间后，会继续执行while循环里的逻辑，再次尝试获取锁。以此循环往复，直到获得锁。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    protected long internalLockLeaseTime;    protected final LockPubSub pubSub;    final CommandAsyncExecutor commandExecutor;        public RedissonLock(CommandAsyncExecutor commandExecutor, String name) {        super(commandExecutor, name);        this.commandExecutor = commandExecutor;        //与WatchDog有关的internalLockLeaseTime        //通过命令执行器CommandExecutor可以获取连接管理器ConnectionManager        //通过连接管理器ConnectionManager可以获取Redis的配置信息类Config        //通过Redis的配置信息类Config可以获取lockWatchdogTimeout超时时间        this.internalLockLeaseTime = commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout();        this.pubSub = commandExecutor.getConnectionManager().getSubscribeService().getLockPubSub();    }    ...
    //加锁    @Override    public void lock() {        try {            lock(-1, null, false);        } catch (InterruptedException e) {            throw new IllegalStateException();        }    }        private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {        //线程ID，用来生成设置Hash的值        long threadId = Thread.currentThread().getId();        //尝试加锁，此时执行RedissonLock.lock()方法默认传入的leaseTime=-1        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);        //ttl为null说明加锁成功        if (ttl == null) {            return;        }              //加锁失败时的处理        CompletableFuture<RedissonLockEntry> future = subscribe(threadId);        if (interruptibly) {            commandExecutor.syncSubscriptionInterrupted(future);        } else {            commandExecutor.syncSubscription(future);        }
        try {            while (true) {                //再次尝试获取锁                ttl = tryAcquire(-1, leaseTime, unit, threadId);                //返回的ttl为null，获取到锁，就退出while循环                if (ttl == null) {                    break;                }                //返回的ttl不为null，则说明其他客户端或线程还持有锁                //那么就利用同步组件Semaphore进行阻塞等待一段ttl的时间                if (ttl >= 0) {                    try {                        commandExecutor.getNow(future).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);                    } catch (InterruptedException e) {                        if (interruptibly) {                            throw e;                        }                        commandExecutor.getNow(future).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);                    }                } else {                    if (interruptibly) {                        commandExecutor.getNow(future).getLatch().acquire();                    } else {                        commandExecutor.getNow(future).getLatch().acquireUninterruptibly();                    }                }            }        } finally {            unsubscribe(commandExecutor.getNow(future), threadId);        }    }    ...
    private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        //默认下waitTime和leaseTime都是-1，下面调用的get方法是来自于RedissonObject的get()方法        //可以理解为异步转同步：将异步的tryAcquireAsync通过get转同步        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));    }        private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        RFuture<Long> ttlRemainingFuture;        if (leaseTime != -1) {            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);        } else {            //默认情况下，由于leaseTime=-1，所以会使用初始化RedissonLock实例时的internalLockLeaseTime            //internalLockLeaseTime的默认值就是lockWatchdogTimeout的默认值，30秒            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime, TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);        }
        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {            //加锁返回的ttlRemaining为null表示加锁成功            if (ttlRemaining == null) {                if (leaseTime != -1) {                    internalLockLeaseTime = unit.toMillis(leaseTime);                } else {                    scheduleExpirationRenewal(threadId);                }            }            return ttlRemaining;        });        return new CompletableFutureWrapper<>(f);    }        //默认情况下，外部传入的leaseTime=-1时，会取lockWatchdogTimeout的默认值=30秒    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,            "if (redis.call('exists', KEYS[1]) == 0) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +                "redis.call('pexpire', KEYS[1], ARGV[1]); " +                "return nil; " +            "end; " +            "return redis.call('pttl', KEYS[1]);",            Collections.singletonList(getRawName()),//锁的名字：KEYS[1]            unit.toMillis(leaseTime),//过期时间：ARGV[1]，默认时为30秒            getLockName(threadId)//ARGV[2]，值为UUID + 线程ID        );    }    ...}
public class RedissonLockEntry implements PubSubEntry<RedissonLockEntry> {    private final Semaphore latch;    ...
    public Semaphore getLatch() {        return latch;    }    ...}
```

**
**

**5.****可重入锁源码之释放锁逻辑**

**(1)宕机自动释放锁**

如果获取锁的所在机器宕机，那么10秒后检查锁是否还在被线程持有的定时调度任务就没了。于是锁对应Redis里的key，就会在最多30秒内进行过期删除。之后，其他的客户端就能成功获取锁了。



当然，前提是创建锁的时候没有设置锁的存活时间leaseTime。如果指定了leaseTime，那么就不会存在10秒后检查锁的定时调度任务。此时获取锁的所在机器宕机，那么锁对应的key会在最多leaseTime后过期。

**
**

**(2)线程主动释放锁的流程**

主动释放锁调用的是RedissonLock的unlock()方法。在RedissonLock的unlock()方法中，会调用get(unlockAsync())方法。也就是首先调用RedissonBaseLock的unlockAsync()方法，然后再调用RedissonObject的get()方法。



其中unlockAsync()方法是异步化执行的方法，释放锁的操作就是异步执行的。而RedisObject的get()方法会通过RFuture同步等待获取异步执行的结果，可以将get(unlockAsync())理解为异步转同步。



在RedissonBaseLock的unlockAsync()方法中：首先会调用RedissonLock的unlockInnerAsync()方法进行异步释放锁，然后当完成释放锁的处理后，再通过异步去取消定时调度任务。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class Application {    public static void main(String[] args) throws Exception {        Config config = new Config();        config.useClusterServers().addNodeAddress("redis://192.168.1.110:7001");        //创建RedissonClient实例        RedissonClient redisson = Redisson.create(config);        //获取可重入锁        RLock lock = redisson.getLock("myLock");        lock.lock();//获取锁        lock.unlock();//释放锁        ...    }}
public class RedissonLock extends RedissonBaseLock {    ...    @Override    public void unlock() {        ...        //异步转同步        //首先调用的是RedissonBaseLock的unlockAsync()方法        //然后调用的是RedissonObject的get()方法        get(unlockAsync(Thread.currentThread().getId()));        ...    }        //异步执行释放锁的lua脚本    protected RFuture<Boolean> unlockInnerAsync(long threadId) {        return evalWriteAsync(            getRawName(),             LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +                "return nil;" +            "end; " +            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +            "if (counter > 0) then " +                "redis.call('pexpire', KEYS[1], ARGV[2]); " +                "return 0; " +            "else " +                "redis.call('del', KEYS[1]); " +                "redis.call('publish', KEYS[2], ARGV[1]); " +                "return 1; " +            "end; " +            "return nil;",            Arrays.asList(getRawName(), getChannelName()),//KEYS[1] + KEYS[2]            LockPubSub.UNLOCK_MESSAGE,//ARGV[1]            internalLockLeaseTime,//ARGV[2]            getLockName(threadId)//ARGV[3]        );    }    ...}
public abstract class RedissonBaseLock extends RedissonExpirable implements RLock {    ...    @Override    public RFuture<Void> unlockAsync(long threadId) {        //异步执行释放锁的lua脚本        RFuture<Boolean> future = unlockInnerAsync(threadId);        CompletionStage<Void> f = future.handle((opStatus, e) -> {            //取消定时调度任务            cancelExpirationRenewal(threadId);            if (e != null) {                throw new CompletionException(e);            }            if (opStatus == null) {                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: " + id + " thread-id: " + threadId);                throw new CompletionException(cause);            }            return null;        });        return new CompletableFutureWrapper<>(f);    }    ...}
abstract class RedissonExpirable extends RedissonObject implements RExpirable {    ...    RedissonExpirable(CommandAsyncExecutor connectionManager, String name) {        super(connectionManager, name);    }    ...}
public abstract class RedissonObject implements RObject {    protected final CommandAsyncExecutor commandExecutor;    protected String name;    protected final Codec codec;        public RedissonObject(Codec codec, CommandAsyncExecutor commandExecutor, String name) {        this.codec = codec;        this.commandExecutor = commandExecutor;        if (name == null) {            throw new NullPointerException("name can't be null");        }        setName(name);    }    ...
    protected final <V> V get(RFuture<V> future) {        //下面会调用CommandAsyncService.get()方法        return commandExecutor.get(future);    }    ...}
public class CommandAsyncService implements CommandAsyncExecutor {    ...    @Override    public <V> V get(RFuture<V> future) {        if (Thread.currentThread().getName().startsWith("redisson-netty")) {            throw new IllegalStateException("Sync methods can't be invoked from async/rx/reactive listeners");        }        try {            return future.toCompletableFuture().get();        } catch (InterruptedException e) {            future.cancel(true);            Thread.currentThread().interrupt();            throw new RedisException(e);        } catch (ExecutionException e) {            throw convertException(e);        }    }    ...}
```

***\*(3)主动释放锁的lua脚本分析\****

首先判断在key为锁名的Hash值中，field为UUID + 线程ID的映射是否存在。如果不存在，则表示锁已经被释放了，直接返回。如果存在，则在key为锁名的Hash值中对field为UUID + 线程ID的value值递减1。即调用Redis的hincrby命令，进行递减1处理。



接着对递减1后的结果进行如下判断：如果递减1后的结果大于0，则表示线程还在持有锁。这对应于持有锁的线程多次重入锁，此时需要重置锁的过期时间为30秒。如果递减1后的结果小于0，则表示线程不再持有锁，于是删除锁对应的key，并且通过Redis的publish命令，发布一个事件。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    ...    //异步执行释放锁的lua脚本    protected RFuture<Boolean> unlockInnerAsync(long threadId) {        return evalWriteAsync(            getRawName(),             LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +                "return nil;" +            "end; " +            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +            "if (counter > 0) then " +                "redis.call('pexpire', KEYS[1], ARGV[2]); " +                "return 0; " +            "else " +                "redis.call('del', KEYS[1]); " +                "redis.call('publish', KEYS[2], ARGV[1]); " +                "return 1; " +            "end; " +            "return nil;",            Arrays.asList(getRawName(), getChannelName()),//KEYS[1] + KEYS[2]            LockPubSub.UNLOCK_MESSAGE,//ARGV[1]            internalLockLeaseTime,//ARGV[2]            getLockName(threadId)//ARGV[3]        );    }    ...}
```

**
**

**6.****可重入锁源码之获取锁超时与锁超时自动释放逻辑**

针对如下代码方式去获取锁，如果超过60秒获取不到锁，就自动放弃获取锁，不会进行永久性阻塞。如果获取到锁之后，锁在10秒内没有被主动释放，那么就会自动释放锁。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class Application {    public static void main(String[] args) throws Exception {        Config config = new Config();        config.useClusterServers().addNodeAddress("redis://192.168.1.110:7001");        //创建RedissonClient实例        RedissonClient redisson = Redisson.create(config);        //获取可重入锁        RLock lock = redisson.getLock("myLock");        boolean res = lock.tryLock(60, 10, TimeUnit.SECONDS);//尝试获取锁        if (res) lock.unlock();//尝试获取锁成功，则释放锁        ...    }}
```

**(1)****尝试获取锁超时**

RedissonLock的tryLock()方法，会传入waitTime和leaseTime尝试获取锁。其中waitTime就是最多可以等待多少时间去获取锁，比如60秒。leaseTime就是获取锁后锁的自动过期时间，比如10秒。



如果第一次获取锁的时间超过了等待获取锁的最大时间waitTime，那么就会返回false。



如果第一次获取锁的时间还没超过等待获取锁的最大时间waitTime，那么就进入while循环，不断地再次尝试获取锁。



如果再次尝试获取锁成功，那么就返回true。如果再次尝试获取锁失败，那么计算还剩下多少时间可以继续等待获取锁。也就是使用time去自减每次尝试获取锁的耗时以及自减每次等待的耗时。



如果发现time小于0，表示等待了waitTime都无法获取锁，则返回false。如果发现time大于0，继续下一轮while循环，尝试获取锁 + time自减耗时。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    ...    @Override    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {        long time = unit.toMillis(waitTime);//最多可以等待多少时间去获取锁        long current = System.currentTimeMillis();//current是第一次尝试获取锁之前的时间戳        long threadId = Thread.currentThread().getId();        Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);        //尝试获取锁成功        if (ttl == null) {            return true;        }
        //第一次尝试获取锁失败        //当前时间减去第一次获取锁之前的时间戳，就是这次尝试获取锁的耗费时间，使用time进行自减        time -= System.currentTimeMillis() - current;        //如果第一次获取锁的时间超过了waitTime等待获取锁的最大时间，那么就会直接返回获取锁失败        if (time <= 0) {            acquireFailed(waitTime, unit, threadId);            return false;        }
        //如果第一次获取锁的时间还没达到waitTime等待获取锁的最大时间        current = System.currentTimeMillis();        CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);        try {            subscribeFuture.toCompletableFuture().get(time, TimeUnit.MILLISECONDS);        } catch (ExecutionException | TimeoutException e) {            if (!subscribeFuture.cancel(false)) {                subscribeFuture.whenComplete((res, ex) -> {                    if (ex == null) {                        unsubscribe(res, threadId);                    }                });            }            acquireFailed(waitTime, unit, threadId);            return false;        }
        try {            time -= System.currentTimeMillis() - current;            if (time <= 0) {                acquireFailed(waitTime, unit, threadId);                return false;            }                    while (true) {                long currentTime = System.currentTimeMillis();                ttl = tryAcquire(waitTime, leaseTime, unit, threadId);                //再次尝试获取锁成功                if (ttl == null) {                    return true;                }                //剩余可用等待的时间，使用time去自减每次尝试获取锁的耗时                time -= System.currentTimeMillis() - currentTime;                if (time <= 0) {                    acquireFailed(waitTime, unit, threadId);                    return false;                }                //返回的ttl不为null，则说明其他客户端或线程还持有锁                //那么就利用同步组件Semaphore进行阻塞等待一段ttl的时间                currentTime = System.currentTimeMillis();                if (ttl >= 0 && ttl < time) {                    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);                } else {                    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);                }                //使用time去自减每次等待的耗时                time -= System.currentTimeMillis() - currentTime;                if (time <= 0) {                    acquireFailed(waitTime, unit, threadId);                    return false;                }            }        } finally {            unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);        }    }    ...}
```

**(2)锁超时自动释放**

当使用RedissonLock.tryAcquire方法尝试获取锁时，传入的leaseTime不是-1，而是一个指定的锁存活时间，那么锁超时就会自动被释放。



当leaseTime不是-1时：

一.锁的过期时间就不是lockWatchdogTimeout=30秒，而是leaseTime

二.执行加锁lua脚本成功后，不会创建一个定时调度任务在10秒后检查锁



总结：当指定了leaseTime后，那么锁最多在leaseTime后，就必须被删除。因为此时没有定时调度任务去检查锁释放还在被线程持有并更新过期时间。所以，锁要么被主动删除，要么在存活leaseTime后自动过期。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonLock extends RedissonBaseLock {    ...    private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        //默认下waitTime和leaseTime都是-1，下面调用的get方法是来自于RedissonObject的get()方法        //可以理解为异步转同步：将异步的tryAcquireAsync通过get转同步        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));    }        private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        RFuture<Long> ttlRemainingFuture;        if (leaseTime != -1) {            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);        } else {            //默认情况下，由于leaseTime=-1，所以会使用初始化RedissonLock实例时的internalLockLeaseTime            //internalLockLeaseTime的默认值就是lockWatchdogTimeout的默认值，30秒            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime, TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);        }        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {            //加锁返回的ttlRemaining为null表示加锁成功            if (ttlRemaining == null) {                if (leaseTime != -1) {                    internalLockLeaseTime = unit.toMillis(leaseTime);                } else {                    scheduleExpirationRenewal(threadId);                }            }            return ttlRemaining;        });        return new CompletableFutureWrapper<>(f);    }    ...}
```

**
**

**7.可重入锁源码总结**

**(1)加锁**

在Redis里设置两层Hash数据结构，默认的过期时间是30秒。第一层Hash值的key是锁名，第二层Hash值的key是UUID + 线程ID。涉及Redis的exists命令、hexists命令、hincrby命令。

**
**

**(2)WatchDog维持加锁**

如果获取锁的线程一直持有锁，那么Redis里的key就会一直保持存活。获取锁成功时会创建一个定时任务10秒后检查锁是否还被线程持有。如果线程还在持有锁，就会重置key的过期时间为30秒，并且创建一个新的定时任务在10秒后继续检查锁是否还被线程持有。

**
**

**(3)可重入锁**

同一个线程可以多次加锁，对第二层的key为UUID + 线程ID的Hash值，每次获取锁就进行累加1。

**
**

**(4)锁互斥**

不同线程尝试加锁时，会由于第二层Hash的key不同，而导致加锁不成功。也就是执行加锁lua脚本时会返回锁的剩余过期时间ttl。然后利用同步组件Semaphore进行阻塞等待一段ttl时间。阻塞等待一段时间后，继续在while循环里再次尝试获取锁。以此循环等待 + 尝试，直到获得锁。

**
**

**(5)手动释放锁**

使用hincrby命令对第二层key(UUID + 线程ID)的Hash值递减1。递减1后还大于0表示锁被重入，需要重置锁的过期时间。递减1后小于等于0表示锁释放完毕，需要删除锁key及取消定时调度任务。

**
**

**(6)宕机自动释放锁**

如果持有锁的客户端宕机，那么锁的WatchDog定时调度任务也没了。此时不会重置锁key的过期时间，于是锁key会自动释放。

**
**

**(7)尝试加锁超时**

在while循环中不断地尝试获取锁。使用time表示还可以等待获取锁的时间。每次循环都对time：自减尝试获取锁的耗时 + 自减每次等待的耗时。在指定时间内没有成功加锁(即time小于0)，就退出循环，表示加锁失败。

**
**

**(8)超时锁自动释放**

当指定了leaseTime时，如果获取锁没有在leaseTime内手动释放锁，那么Redis里的锁key会自动过期，自动释放锁。

**
**

**8.****公平锁源码之加锁和排队**

**(****1)加锁时的执行流程**

使用Redisson的公平锁RedissonFairLock进行加锁时：首先调用的是RedissonLock的lock()方法，然后会调用RedissonLock的tryAcquire()方法，接着会调用RedissonLock的tryAcquireAsync()方法。



在RedissonLock的tryAcquireAsync()方法中，会调用一个可以被RedissonLock子类重载的tryLockInnerAsync()方法。对于非公平锁，执行到这会调用RedissonLock的tryLockInnerAsync()方法。对于公平锁，执行到这会调用RedissonFairLock的tryLockInnerAsync()方法。



在RedissonFairLock的tryLockInnerAsync()方法中，便执行具体的lua脚本。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class RedissonDemo {    public static void main(String[] args) throws Exception {        ...        //创建RedissonClient实例        RedissonClient redisson = Redisson.create(config);                //获取公平的可重入锁        RLock fairLock = redisson.getFairLock("myLock");        fairLock.lock();//加锁        fairLock.unlock();//释放锁    }}
public class RedissonLock extends RedissonBaseLock {    ...    //不带参数的加锁    public void lock() {        try {            lock(-1, null, false);        } catch (InterruptedException e) {            throw new IllegalStateException();        }    }        //带参数的加锁    public void lock(long leaseTime, TimeUnit unit) {        try {            lock(leaseTime, unit, false);        } catch (InterruptedException e) {            throw new IllegalStateException();        }    }        private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {        long threadId = Thread.currentThread().getId();        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);        //加锁成功        if (ttl == null) {            return;        }        //加锁失败        ...    }        private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));    }        private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {        RFuture<Long> ttlRemainingFuture;        if (leaseTime != -1) {            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);        } else {            //非公平锁，接下来调用的是RedissonLock.tryLockInnerAsync()方法            //公平锁，接下来调用的是RedissonFairLock.tryLockInnerAsync()方法            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime, TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);        }
        //对RFuture<Long>类型的ttlRemainingFuture添加回调监听        CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {            //tryLockInnerAsync()里的加锁lua脚本异步执行完毕，会回调如下方法逻辑：            //加锁成功            if (ttlRemaining == null) {                if (leaseTime != -1) {                    //如果传入的leaseTime不是-1，也就是指定锁的过期时间，那么就不创建定时调度任务                    internalLockLeaseTime = unit.toMillis(leaseTime);                } else {                    //创建定时调度任务                    scheduleExpirationRenewal(threadId);                }            }            return ttlRemaining;        });        return new CompletableFutureWrapper<>(f);    }    ...}
public class RedissonFairLock extends RedissonLock implements RLock {    private final long threadWaitTime;//线程可以等待锁的时间    private final CommandAsyncExecutor commandExecutor;    private final String threadsQueueName;    private final String timeoutSetName;        public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name) {        this(commandExecutor, name, 60000*5);//传入60秒*5=5分钟    }        public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name, long threadWaitTime) {        super(commandExecutor, name);        this.commandExecutor = commandExecutor;        this.threadWaitTime = threadWaitTime;        threadsQueueName = prefixName("redisson_lock_queue", name);        timeoutSetName = prefixName("redisson_lock_timeout", name);    }    ...
    @Override    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {        long wait = threadWaitTime;        if (waitTime != -1) {            //将传入的指定的获取锁等待时间赋值给wait变量            wait = unit.toMillis(waitTime);        }          ...
        if (command == RedisCommands.EVAL_LONG) {            return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,                //步骤一：remove stale threads，移除等待超时的线程                "while true do " +                    //获取队列中的第一个元素                    //KEYS[2]是一个用来对线程排队的队列的名字                    "local firstThreadId2 = redis.call('lindex', KEYS[2], 0);" +                    "if firstThreadId2 == false then " +                        "break;" +                    "end;" +                    //获取队列中第一个元素对应的分数，也就是排第一的线程的过期时间                    //KEYS[3]是一个用来对线程排序的有序集合的名字                    "local timeout = tonumber(redis.call('zscore', KEYS[3], firstThreadId2));" +                    //如果排第一的线程的过期时间小于当前时间，说明该线程等待超时了都还没获取到锁，所以要移除                    //ARGV[4]是当前时间                    "if timeout <= tonumber(ARGV[4]) then " +                        //remove the item from the queue and timeout set NOTE we do not alter any other timeout                        //从有序集合 + 队列中移除这个线程                        "redis.call('zrem', KEYS[3], firstThreadId2);" +                        "redis.call('lpop', KEYS[2]);" +                    "else " +                        "break;" +                    "end;" +                "end;" +
                //check if the lock can be acquired now                //步骤二：判断当前线程现在能否尝试获取锁，以下两种情况可以通过判断去进行尝试获取锁                //情况一：锁不存在 + 队列也不存在；KEYS[1]是锁的名字；KEYS[2]是对线程排队的队列；                //情况二：锁不存在 + 队列存在 + 队列的第一个元素就是当前线程；ARGV[2]是当前线程的UUID + ThreadID；                "if (redis.call('exists', KEYS[1]) == 0) " +                    "and ((redis.call('exists', KEYS[2]) == 0) " +                        "or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then " +                    //步骤三：当前线程执行获取锁的操作                    //remove this thread from the queue and timeout set                    //弹出队列的第一个元素 + 从有序集合中删除UUID:ThreadID对应的元素                    "redis.call('lpop', KEYS[2]);" +                    "redis.call('zrem', KEYS[3], ARGV[2]);" +                     //decrease timeouts for all waiting in the queue                    //递减有序集合中每个线程的分数，也就是递减每个线程获取锁时的已经等待时间                    //zrange返回有序集合KEYS[3]中指定区间内(0,-1)的成员，也就是全部成员                    "local keys = redis.call('zrange', KEYS[3], 0, -1);" +                    "for i = 1, #keys, 1 do " +                        //对有序集合KEYS[3]的成员keys[i]的score减去：tonumber(ARGV[3])                        //ARGV[3]就是线程获取锁时可以等待的时间，默认是5分钟                        "redis.call('zincrby', KEYS[3], -tonumber(ARGV[3]), keys[i]);" +                    "end;" +
                    //acquire the lock and set the TTL for the lease                    //hset设置Hash值进行加锁操作 + pexpire设置锁key的过期时间 + 最后返回nil表示加锁成功                    "redis.call('hset', KEYS[1], ARGV[2], 1);" +                    "redis.call('pexpire', KEYS[1], ARGV[1]);" +                    "return nil;" +                "end;" +
                //check if the lock is already held, and this is a re-entry(可重入锁)                //步骤四：判断锁是否已经被当前线程持有，KEYS[1]是锁的名字，ARGV[2]是当前线程的UUID + ThreadID；                "if redis.call('hexists', KEYS[1], ARGV[2]) == 1 then " +                    "redis.call('hincrby', KEYS[1], ARGV[2],1);" +                    "redis.call('pexpire', KEYS[1], ARGV[1]);" +                    "return nil;" +                "end;" +                 //the lock cannot be acquired, check if the thread is already in the queue                //步骤五：判断当前获取锁失败的线程是否已经在队列中排队                //KEYS[3]是对线程排序的有序集合，ARGV[2]是当前线程的UUID + ThreadID；                "local timeout = redis.call('zscore', KEYS[3], ARGV[2]);" +                "if timeout ~= false then " +                    //the real timeout is the timeout of the prior thread in the queue,                     //but this is approximately correct, and avoids having to traverse the queue                    //如果当前获取锁失败的线程已经在队列中排队                    //那么就返回该线程等待获取锁时，还剩多少时间就超时了，外部代码拿到这个时间会阻塞等待这个时间                    //ARGV[3]是当前线程获取锁时可以等待的时间，ARGV[4]是当前时间                    "return timeout - tonumber(ARGV[3]) - tonumber(ARGV[4]);" +                "end;" +
                //add the thread to the queue at the end, and set its timeout in the timeout set to the timeout of                //the prior thread in the queue (or the timeout of the lock if the queue is empty) plus the threadWaitTime                //步骤六：对获取锁失败的线程进行排队处理                "local lastThreadId = redis.call('lindex', KEYS[2], -1);" +                "local ttl;" +                //如果在队列中排队的最后一个元素不是当前线程                "if lastThreadId ~= false and lastThreadId ~= ARGV[2] then " +                    //lastThreadId是在队列中排最后的线程，ARGV[2]是当前线程的UUID+线程ID，ARGV[4]是当前时间                    //因为拥有最大过期时间的线程在队列中是排最后的                    //所以可通过队列中的最后一个元素的过期时间，计算当前线程的过期时间                    //从而保证新加入队列和有序集合的线程的过期时间是最大的                    //下面这一行会计算出：还有多少时间，当前队列中排最后的线程就会过期，外部代码拿到这个时间会阻塞等待这个时间                    //这样后一个加入队列的线程，会阻塞等待前一个加入队列的线程的过期时间                    "ttl = tonumber(redis.call('zscore', KEYS[3], lastThreadId)) - tonumber(ARGV[4]);" +                "else " +                    //下面这一行会计算出：还有多少时间，锁就会过期，外部代码拿到这个时间会阻塞等待这个时间                    "ttl = redis.call('pttl', KEYS[1]);" +                "end;" +                //计算当前线程在排队等待锁时的过期时间                "local timeout = ttl + tonumber(ARGV[3]) + tonumber(ARGV[4]);" +                //把当前线程作为一个元素插入有序集合，并设置元素分数为该线程在排队等待锁时的过期时间                //然后再把当前线程作为一个元素插入队列尾部                "if redis.call('zadd', KEYS[3], timeout, ARGV[2]) == 1 then " +                    "redis.call('rpush', KEYS[2], ARGV[2]);" +                "end;" +                "return ttl;",                Arrays.asList(getRawName(), threadsQueueName, timeoutSetName),                unit.toMillis(leaseTime),                getLockName(threadId),                wait,                currentTime            );        }        ...    }    ...}
```

**(2)获取公平锁的lua脚本相关参数说明**

KEYS[1]是getRawName()，它是一个Hash数据结构的key，也就是锁的名字，比如"myLock"。

KEYS[2]是threadsQueueName，它是一个用来对线程排队的队列的名字，多个客户端线程申请获取锁时，会到这个队列里进行排队。比如"redisson_lock_queue:{myLock}"。

KEYS[3]是timeoutSetName，它是一个用来对线程排序的有序集合的名字，这个有序集合可以自动按照每个数据指定的分数进行排序。比如"redisson_lock_timeout:{myLock}"。



ARGV[1]是leaseTime，代表锁的过期时间。如果leaseTime没有指定，默认就是internalLockLeaseTime = 30秒。

ARGV[2]是getLockName(threadId)，代表客户端UUID + 线程ID。

ARGV[3]是threadWaitTime，代表线程可以等待的时间(默认5分钟)。

ARGV[4]是currentTime，代表当前时间。

**
**

**(3)lua脚本步骤一：****进入while循环移除队列和有序集合中等待超时的线程**

while循环中首先执行命令："lindex redisson_lock_queue:{myLock} 0"，也就是获取"redisson_lock_queue:{myLock}"这个队列中的第一个元素。一开始该队列是空的，所以什么都获取不到，firstThreadId2为false。此时就会break掉，退出while循环。



如果获取到队列中的第一个元素，那么就会执行zscore命令：从有序集合中获取该元素对应的分数，也就是该元素对应线程的过期时间。如果过期时间比当前时间小，那么就要从队列和有序集合中移除该元素。否则，也会break掉，退出while循环。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//步骤一：remove stale threads，移除等待超时的线程"while true do " +    //获取队列中的第一个元素    //KEYS[2]是一个用来对线程排队的队列的名字    "local firstThreadId2 = redis.call('lindex', KEYS[2], 0);" +    "if firstThreadId2 == false then " +        "break;" +    "end;" +    //获取队列中第一个元素对应的分数，也就是排第一的线程的过期时间    //KEYS[3]是一个用来对线程排序的有序集合的名字    "local timeout = tonumber(redis.call('zscore', KEYS[3], firstThreadId2));" +    //如果排第一的线程的过期时间小于当前时间，说明该线程等待锁超时了都还没获取到锁，所以要移除    //ARGV[4]是当前时间    "if timeout <= tonumber(ARGV[4]) then " +        //remove the item from the queue and timeout set NOTE we do not alter any other timeout        //从有序集合+队列中移除这个线程        "redis.call('zrem', KEYS[3], firstThreadId2);" +        "redis.call('lpop', KEYS[2]);" +    "else " +        "break;" +    "end;" +"end;" +
```

**(4)lua脚本步骤二：****判断当前线程能否获取锁**

**判断条件一：**

首先执行命令"exists myLock"，判断锁是否存在。一开始没有线程加过锁，所以判断条件肯定是成立的，该条件为true。



**判断条件二：**

接着执行命令"exists redisson_lock_queue:{myLock}"，看队列是否存在。一开始也没有这个队列，所以这个条件也肯定成立，该条件为true。

**
**

**判断条件三：**

如果有这个队列，则判断队列存在的条件不成立，执行"或"后面的判断。也就是执行命令"lindex redisson_lock_queue:{myLock} 0"，判断队列的第一个元素是否是当前线程的UUID + ThreadID。

- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//check if the lock can be acquired now//步骤二：判断当前线程现在能否尝试获取锁，以下两种情况可以通过判断去进行尝试获取锁//情况一：锁不存在 + 队列也不存在；KEYS[1]是锁的名字；//情况二：锁不存在 + 队列存在 + 队列的第一个元素就是当前线程；ARGV[2]是当前线程的UUID + ThreadID；"if (redis.call('exists', KEYS[1]) == 0) " +    "and ((redis.call('exists', KEYS[2]) == 0) " +        "or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then " +    ..."end;" +
```

总结当前线程现在可以尝试获取锁的情况如下：

***\*情况一：\****锁不存在 + 队列也不存在

***\*情况二：\****锁不存在 + 队列存在 + 队列的第一个元素就是当前线程

**
**

**(5)lua脚本步骤三：****执行获取锁的操作**

当判断现在能否尝试获取锁的条件通过后，便会执行如下操作：

**
**

***\*步骤一：\****执行命令"lpop redisson_lock_queue:{myLock}"，弹出队列第一个元素。一开始该队列是空的，所以该命令不会进行处理。接着执行命令"zrem redisson_lock_timeout:{myLock} UUID1:ThreadID1"，也就是从有序集合中删除UUID1:ThreadID1对应的元素。一开始该有序集合也是空的，所以该命令不会进行处理。

**
**

***\*步骤二：\****执行命令"hset myLock UUID1:ThreadID1 1"，进行加锁操作。在设置key为myLock的Hash值中，field为UUID1:ThreadID1的value值为1。接着执行命令"pexpire myLock 30000"，设置锁key的过期时间为30秒。



最后返回nil，这样在外层代码中，就会认为加锁成功。于是就会创建一个WatchDog看门狗定时调度任务，10秒后对锁进行检查。如果检查发现当前线程还持有这个锁，那么就重置锁key的过期时间为30秒，并且重新创建一个WatchDog看门狗定时调度任务在10秒后继续进行检查。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//check if the lock can be acquired now//步骤二：判断当前线程现在能否尝试获取锁，以下两种情况可以通过判断去进行尝试获取锁//情况一：锁不存在 + 队列也不存在；KEYS[1]是锁的名字；KEYS[2]是对线程排队的队列；//情况二：锁不存在 + 队列存在 + 队列的第一个元素就是当前线程；ARGV[2]是当前线程的UUID + ThreadID；"if (redis.call('exists', KEYS[1]) == 0) " +    "and ((redis.call('exists', KEYS[2]) == 0) " +        "or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then " +    //步骤三：当前线程执行获取锁的操作    //remove this thread from the queue and timeout set    //弹出队列的第一个元素 + 从有序集合中删除UUID:ThreadID对应的元素    "redis.call('lpop', KEYS[2]);" +    "redis.call('zrem', KEYS[3], ARGV[2]);" +
    //decrease timeouts for all waiting in the queue    //递减有序集合中每个线程的分数，也就是递减每个线程获取锁时的已经等待时间    //zrange返回有序集合KEYS[3]中指定区间内(0,-1)的成员，也就是全部成员    "local keys = redis.call('zrange', KEYS[3], 0, -1);" +    "for i = 1, #keys, 1 do " +        //对有序集合KEYS[3]的成员keys[i]的score减去：tonumber(ARGV[3])        //ARGV[3]就是线程获取锁时可以等待的时间，默认是5分钟        "redis.call('zincrby', KEYS[3], -tonumber(ARGV[3]), keys[i]);" +    "end;" +
    //acquire the lock and set the TTL for the lease    //hset设置Hash值进行加锁操作 + pexpire设置锁key的过期时间 + 最后返回nil表示加锁成功    "redis.call('hset', KEYS[1], ARGV[2], 1);" +    "redis.call('pexpire', KEYS[1], ARGV[1]);" +    "return nil;" +"end;" +
```

**(6)lua脚本步骤四：****判断锁是否已经被当前线程持有(可重入锁)**

此时会执行命令"hexists myLock UUID:ThreadID"。如果判断条件通过，则说明是持有锁的线程对锁进行了重入。于是会执行命令"hincrby myLock UUID:ThreadID 1"，对key为锁名的Hash值中，field为UUID + 线程ID的value值累加1。并且执行命令"pexpire myLock 300000"重置锁key的过期时间。最后返回nil，表示重入加锁成功。

- 
- 
- 
- 
- 
- 
- 

```
//check if the lock is already held, and this is a re-entry(可重入锁)//步骤四：判断锁是否已经被当前线程持有，KEYS[1]是锁的名字，ARGV[2]是当前线程的UUID + ThreadID；"if redis.call('hexists', KEYS[1], ARGV[2]) == 1 then " +    "redis.call('hincrby', KEYS[1], ARGV[2], 1);" +    "redis.call('pexpire', KEYS[1], ARGV[1]);" +    "return nil;" +"end;" +
```

**(7)lua脚本步骤五：判断当前获取锁失败的线程是否已经在队列中排队**

通过执行命令"zscore redisson_lock_timeout:{myLock} UUID:ThreadID"，获取当前线程在有序集合中的对应的分数，也就是过期时间。如果获取成功则返回：当前线程等待获取锁的超时时间还剩多少，外部代码拿到这个时间会阻塞等待这个时间。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//the lock cannot be acquired, check if the thread is already in the queue//步骤五：判断当前获取锁失败的线程是否已经在队列中排队//KEYS[3]是对线程排序的有序集合，ARGV[2]是当前线程的UUID+ThreadID；"local timeout = redis.call('zscore', KEYS[3], ARGV[2]);" +"if timeout ~= false then " +    //the real timeout is the timeout of the prior thread in the queue,     //but this is approximately correct, and avoids having to traverse the queue    //如果当前获取锁失败的线程已经在队列中排队    //那么就返回该线程等待获取锁时，还剩多少时间就超时了，外部代码拿到这个时间会阻塞等待这个时间    //ARGV[3]是当前线程获取锁时可以等待的时间，ARGV[4]是当前时间    "return timeout - tonumber(ARGV[3]) - tonumber(ARGV[4]);" +"end;" +
```

**(8)lua脚本步骤六：对获取锁失败的线程进行排队**

首先获取队列中的最后一个元素。因为拥有最大过期时间的线程在队列中是排最后的，所以可通过队列中的最后一个元素的过期时间，计算当前线程的过期时间。从而保证新加入队列和有序集合的线程的过期时间是最大的。然后获取锁或者队列中排最后的线程剩余的存活时间，接着计算当前线程在排队等待锁时的过期时间。



然后把当前线程作为一个元素插入有序集合，并设置有序集合中该元素的分数为该线程在排队等待锁时的过期时间，接着再把当前线程作为一个元素插入队列尾部。



最后返回锁或者队列中排第一的线程剩余的存活时间ttl给外层代码。如果外层代码拿到的返回值是非null，那么客户端会进入一个while循环。在while循环会每阻塞等待ttl时间再尝试去进行加锁，重新执行lua脚本。



如果队列里没有元素，那么第一个加入队列的线程，会阻塞等待锁的过期时间。如果队列里有元素，那么后一个加入队列的线程，会阻塞等待前一个加入队列的线程的过期时间。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//步骤六：对获取锁失败的线程进行排队处理"local lastThreadId = redis.call('lindex', KEYS[2], -1);" +"local ttl;" +//如果在队列中排队的最后一个元素不是当前线程"if lastThreadId ~= false and lastThreadId ~= ARGV[2] then " +    //lastThreadId是在队列中排最后的线程，ARGV[2]是当前线程的UUID + 线程ID，ARGV[4]是当前时间    //因为拥有最大过期时间的线程在队列中是排最后的    //所以可通过队列中的最后一个元素的过期时间，计算当前线程的过期时间    //从而保证新加入队列和有序集合的线程的过期时间是最大的    //下面这一行会计算出：还有多少时间，当前队列中排最后的线程就会过期，外部代码拿到这个时间会阻塞等待这个时间    //这样后一个加入队列的线程，会阻塞等待前一个加入队列的线程的过期时间    "ttl = tonumber(redis.call('zscore', KEYS[3], lastThreadId)) - tonumber(ARGV[4]);" +"else " +    //下面这一行会计算出：还有多少时间，锁就会过期，外部代码拿到这个时间会阻塞等待这个时间    "ttl = redis.call('pttl', KEYS[1]);" +"end;" +//计算当前线程在排队等待锁时的过期时间"local timeout = ttl + tonumber(ARGV[3]) + tonumber(ARGV[4]);" +//把当前线程作为一个元素插入有序集合，并设置元素分数为该线程在排队等待锁时的过期时间//然后再把当前线程作为一个元素插入队列尾部"if redis.call('zadd', KEYS[3], timeout, ARGV[2]) == 1 then " +    "redis.call('rpush', KEYS[2], ARGV[2]);" +"end;" +"return ttl;",
```

**(9)获取锁失败的第一个线程执行lua脚本的流程**

公平锁的核心在于申请加锁时，加锁失败的各个客户端会排队。之后锁被释放时，会依次获取锁，从而实现公平性。



假设此时第一个客户端线程已加锁成功，第二个客户端线程也来尝试加锁，那么会进行如下排队处理。



***\*步骤一：\****进入while循环，移除等待超时的线程。执行命令"lindex redisson_lock_queue:{myLock} 0"，获取队列排第一元素。由于此时队列还是空的，所以获取到的是false，于是退出while循环。



***\*步骤二：\****判断当前线程现在能否尝试获取锁。因为执行命令"exists myLock"，发现锁已经存在了，于是判断不通过。



***\*步骤三：\****判断锁是否已经被当前线程持有，由于第二个客户端线程的UUID + 线程ID必然不等于第一个客户端线程。所以此时执行命令"hexists myLock UUID2:ThreadID2"，发现不存在。所以此处的可重入锁的判断条件也不成立。



***\*步骤四：\****判断当前获取锁失败的线程是否已经在队列中排队。由于当前线程是第一个获取锁失败的线程，所以判断不通过。



***\*步骤五：\****接下来进行排队处理。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//对获取锁失败的线程进行排队处理"local lastThreadId = redis.call('lindex', KEYS[2], -1);" +"local ttl;" +//如果在队列中排队的最后一个元素不是当前线程"if lastThreadId ~= false and lastThreadId ~= ARGV[2] then " +    //lastThreadId是在队列中排最后的线程，ARGV[2]是当前线程的UUID+线程ID，ARGV[4]是当前时间    //因为拥有最大过期时间的线程在队列中是排最后的    //所以可通过队列中的最后一个元素的过期时间，计算当前线程的过期时间    //从而保证新加入队列和有序集合的线程的过期时间是最大的    //下面这一行会计算出：还有多少时间，当前队列中排最后的线程就会过期，外部代码拿到这个时间会阻塞等待这个时间      //这样后一个加入队列的线程，会阻塞等待前一个加入队列的线程的过期时间    "ttl = tonumber(redis.call('zscore', KEYS[3], lastThreadId)) - tonumber(ARGV[4]);" +"else " +    //下面这一行会计算出：还有多少时间，锁就会过期，外部代码拿到这个时间会阻塞等待这个时间    "ttl = redis.call('pttl', KEYS[1]);" +"end;" +//计算当前线程在排队等待锁时的过期时间"local timeout = ttl + tonumber(ARGV[3]) + tonumber(ARGV[4]);" +//把当前线程作为一个元素插入有序集合，并设置元素分数为该线程在排队等待锁时的过期时间//然后再把当前线程作为一个元素插入队列尾部"if redis.call('zadd', KEYS[3], timeout, ARGV[2]) == 1 then " +    "redis.call('rpush', KEYS[2], ARGV[2]);" +"end;" +"return ttl;"
```

首先执行命令"lindex redisson_lock_queue:{myLock} 0"。也就是从队列中获取最后一个元素，由于此时队列是空，所以获取不到元素。然后执行命令"ttl = pttl myLock"，获取锁剩余的存活时间。



接着计算当前线程在排队等待锁时的过期时间。假设myLock剩余的存活时间ttl为20秒，那么timeout = ttl + 5分钟 + 当前时间 = 20秒 + 5分钟 + 10:00:00 = 10:05:20；



然后执行命令"zadd redisson_lock_timeout:{myLock} 10:05:20 UUID2:ThreadID2"，这行命令的意思是，在有序集合中插入一个元素。元素值是UUID2:ThreadID2，元素对应的分数是10:05:20。分数会用时间的Long型时间戳来表示，时间越靠后，时间戳就越大。有序集合Sorted Set会自动根据插入的元素分数从小到大进行排序。



接着执行命令"rpush redisson_lock_queue:{myLock} UUID2:TheadID2"，这行命令的意思是，将UUID2:ThreadID2插入到队列的尾部。



最后返回ttl给外层代码，也就是返回myLock剩余的存活时间。如果外层代码拿到的ttl是非null，那么客户端会进入一个while循环。在while循环会每阻塞等待ttl时间就尝试进行加锁，重新执行lua脚本。

**
**

**(10)获取锁失败的第二个线程执行lua脚本的流程**

如果此时有第三个客户端线程也来尝试加锁，那么会进行如下排队处理。

**
**

***\*步骤一：\****进入while循环，移除等待超时的线程。执行命令"lindex redisson_lock_queue:{myLock} 0"，获取队列排第一元素。此时获取到UUID2:ThreadID2，代表着第二个客户端线程正在队列里排队。



继续执行命令"zscore redisson_lock_timeout:{myLock} UUID2:ThreadID2"，从有序集合中获取UUID2:ThreadID2对应的分数，timeout = 10:05:20。



假设当前时间是10:00:25，那么timeout <= 10:00:25的这个条件不成立，于是退出while循环。

**
**

***\*步骤二：\****判断当前线程现在能否尝试获取锁，发现不能通过。因为执行命令"exists myLock"时，发现锁已经存在。

**
**

***\*步骤三：\****判断锁是否已经被当前线程持有。由于第三个客户端线程的UUID + 线程ID必然不等于第一个客户端线程。所以此时执行命令"hexists myLock UUID3:ThreadID3"，发现不存在。所以此处的可重入锁的判断条件也不成立。

**
**

***\*步骤四：\****判断当前获取锁失败的线程是否已经在队列中排队。由于当前线程是第二个获取锁失败的线程，所以判断不通过。

**
**

***\*步骤五：\****接下来进行排队处理。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//对获取锁失败的线程进行排队处理"local lastThreadId = redis.call('lindex', KEYS[2], -1);" +"local ttl;" +//如果在队列中排队的最后一个元素不是当前线程"if lastThreadId ~= false and lastThreadId ~= ARGV[2] then " +    //lastThreadId是在队列中排最后的线程，ARGV[2]是当前线程的UUID + 线程ID，ARGV[4]是当前时间    //因为拥有最大过期时间的线程在队列中是排最后的    //所以可通过队列中的最后一个元素的过期时间，计算当前线程的过期时间    //从而保证新加入队列和有序集合的线程的过期时间是最大的    //下面这一行会计算出：还有多少时间，当前队列中排最后的线程就会过期，外部代码拿到这个时间会阻塞等待这个时间      //这样后一个加入队列的线程，会阻塞等待前一个加入队列的线程的过期时间    "ttl = tonumber(redis.call('zscore', KEYS[3], lastThreadId)) - tonumber(ARGV[4]);" +"else " +    //下面这一行会计算出：还有多少时间，锁就会过期，外部代码拿到这个时间会阻塞等待这个时间    "ttl = redis.call('pttl', KEYS[1]);" +"end;" +//计算当前线程在排队等待锁时的过期时间"local timeout = ttl + tonumber(ARGV[3]) + tonumber(ARGV[4]);" +//把当前线程作为一个元素插入有序集合，并设置元素分数为该线程在排队等待锁时的过期时间//然后再把当前线程作为一个元素插入队列尾部"if redis.call('zadd', KEYS[3], timeout, ARGV[2]) == 1 then " +    "redis.call('rpush', KEYS[2], ARGV[2]);" +"end;" +"return ttl;"
```

首先执行命令"lindex redisson_lock_queue:{myLock} 0"，获取到队列中的最后一个元素UUID2:ThreadID2。



然后判断条件是否成立：lastThreadId不为false + lastThreadId不是自己。由于此时的ARGV[2] = UUID3:ThreadID3，所以判断条件成立。即在队列里排队的最后一个元素并不是当前尝试获取锁的客户端线程。



于是执行："zscore redisson_lock_timeout:{myLock} UUID2:ThreadID2" - 当前时间，也就是获取在队列中排最后的线程还有多少时间就会过期，从而得到ttl。



接着根据ttl计算当前线程在排队等待锁时的过期时间timeout，然后执行zadd和rpush命令对当前线程进行入队和排队，最后返回ttl。

**
**

**9.****公平锁****源****码之可重入加锁**

持有公平锁的客户端重复进行lock.lock()，执行加锁lua脚本的流程如下：



***\*步骤一：\****进入while循环，移除等待超时的线程。执行命令"lindex redisson_lock_queue:{myLock} 0"，获取队列排第一元素。此时获取到UUID2:ThreadID2，代表着第二个客户端线程正在队列里排队。



继续执行命令"zscore redisson_lock_timeout:{myLock} UUID2:ThreadID2"，从有序集合中获取UUID2:ThreadID2对应的分数，timeout = 10:05:20。



假设当前时间是10:00:25，那么timeout <= 10:00:25的这个条件不成立，于是退出while循环。



***\*步骤二：\****判断当前线程现在能否尝试获取锁，发现不能通过。因为执行命令"exists myLock"时，发现锁已经存在。



***\*步\**\**骤三：\****判断锁是否已经被当前线程持有。由于当前线程的UUID + 线程ID等于持有锁的线程。即此时执行命令"hexists myLock UUID:ThreadID"发现key是存在的，所以此处的可重入锁的判断条件成立。



于是会执行命令"hincrby myLock UUID:ThreadID 1"，对key为锁名的Hash值中，key为UUID + 线程ID的Hash值累加1。并且执行命令"pexpire myLock 300000"重置锁key的过期时间。最后返回nil，表示重入加锁成功。

- 
- 
- 
- 
- 
- 
- 

```
//check if the lock is already held, and this is a re-entry(可重入锁)//步骤四：判断锁是否已经被当前线程持有，KEYS[1]是锁的名字，ARGV[2]是当前线程的UUID+ThreadID；"if redis.call('hexists', KEYS[1], ARGV[2]) == 1 then " +    "redis.call('hincrby', KEYS[1], ARGV[2], 1);" +    "redis.call('pexpire', KEYS[1], ARGV[1]);" +    "return nil;" +"end;" +
```

**
**

**10.****公平锁源码之队列重排**

**导致队列重排的是lua脚本的步骤一(****移除等待超时的线程)，也就是公平锁lua脚本中**while循环的作用。



当客户端线程使用RedissonLock的tryAcquire()方法尝试获取公平锁，并且指定了一个获取锁的超时时间时。比如指定客户端线程在队列里排队超过了20秒，就不再尝试获取锁了。如果获取锁的超时时间没有指定，新版本是默认5分钟超时，旧版本是默认5秒后超时。



此时由于这些等待获取锁已超时的线程元素还存在队列和有序集合里，所以可以通过while循环的逻辑来清除这些不再尝试获取锁的客户端线程。



在新版本，随着时间推移，这些等待获取锁超时的线程就会被移出队列。在旧版本，随着时间推移，这些等待获取锁超时的线程只要不再尝试加锁，那么其等待获取锁的超时时间就不会更新被不断延长，就会被移除队列。



如果客户端宕机了，那么客户端就不会重新尝试获取锁。在新版本中，随着时间推移，宕机的客户端线程就会被移出队列。在旧版本中，就不会刷新和延长有序集合中的超时时间分数，这样while循环的逻辑就会将这些宕机的客户端线程从队列中移出。



在新版本中，最多5分钟后，宕机的客户端线程会被移出队列。在旧版本中，最多5秒钟后，宕机的客户端线程就会被移出队列。



因为网络延迟等原因，可能会导致客户端线程等待锁时间过长，从而触发各个客户端线程的排队顺序的重排序。有的客户端如果在队列里等待时间过长，可能就会触发一次队列的重排序。新版本触发重排序的频率是每5分钟，旧版本触发重排序的频率是每5秒。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//步骤一：移除等待超时的线程"while true do " +    //获取队列中的第一个元素    //KEYS[2]是一个用来对线程排队的队列的名字    "local firstThreadId2 = redis.call('lindex', KEYS[2], 0);" +    "if firstThreadId2 == false then " +        "break;" +    "end; " +    //获取队列中第一个元素对应的分数，也就是排第一的线程的过期时间    //KEYS[3]是一个用来对线程排序的有序集合的名字    "local timeout = tonumber(redis.call('zscore', KEYS[3], firstThreadId2));" +    //如果排第一的线程的过期时间小于当前时间，说明该线程等待超时了都还没获取到锁，所以要移除    //ARGV[4]是当前时间    "if timeout <= tonumber(ARGV[4]) then " +        //从有序集合 + 队列中移除这个线程        "redis.call('zrem', KEYS[3], firstThreadId2); " +        "redis.call('lpop', KEYS[2]); " +    "else " +        "break;" +    "end; " +"end;" +
```

**
**

**11.****公平锁源码之释放锁****
**

**(1)释放公平锁的流程**

释放公平锁首先调用的还是RedissonLock的unlock()方法。



在RedissonLock的unlock()方法中，会调用get(unlockAsync())。也就是首先调用RedissonBaseLock的unlockAsync()方法，然后调用RedissonObject的get()方法。



其中个RedissonBaseLock的unlockAsync()方法是异步化执行的方法，释放锁的操作是异步执行的。而RedisObject的get()方法会通过RFuture同步等待获取异步执行的结果。所以，可以将get(unlockAsync())理解为异步转同步。



在RedissonBaseLock的unlockAsync()方法中，就会调用公平锁RedissonFairLock的unlockInnerAsync()方法进行释放锁。然后当完成释放锁的处理后，会通过异步去取消定时调度任务。

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
public class Application {    public static void main(String[] args) throws Exception {        Config config = new Config();        config.useClusterServers().addNodeAddress("redis://192.168.1.110:7001");        //创建RedissonClient实例        RedissonClient redisson = Redisson.create(config);        //获取公平的可重入锁        RLock fairLock = redisson.getFairLock("myLock");        fairLock.lock();        fairLock.unlock();        ...    }}
public class RedissonLock extends RedissonBaseLock {    ...    @Override    public void unlock() {        ...        //异步转同步        //首先调用的是RedissonBaseLock的unlockAsync()方法        //然后调用的是RedissonObject的get()方法        get(unlockAsync(Thread.currentThread().getId()));        ...    }    ...}
public abstract class RedissonBaseLock extends RedissonExpirable implements RLock {    ...    @Override    public RFuture<Void> unlockAsync(long threadId) {        //异步执行释放锁的lua脚本        RFuture<Boolean> future = unlockInnerAsync(threadId);        CompletionStage<Void> f = future.handle((opStatus, e) -> {            //取消定时调度任务            cancelExpirationRenewal(threadId);            if (e != null) {                throw new CompletionException(e);            }            if (opStatus == null) {                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: " + id + " thread-id: " + threadId);                throw new CompletionException(cause);            }            return null;        });        return new CompletableFutureWrapper<>(f);    }
    protected abstract RFuture<Boolean> unlockInnerAsync(long threadId);    ...}
public class RedissonFairLock extends RedissonLock implements RLock {    private final long threadWaitTime;    private final CommandAsyncExecutor commandExecutor;    private final String threadsQueueName;    private final String timeoutSetName;        public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name) {        this(commandExecutor, name, 60000*5);    }        public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name, long threadWaitTime) {        super(commandExecutor, name);        this.commandExecutor = commandExecutor;        this.threadWaitTime = threadWaitTime;        threadsQueueName = prefixName("redisson_lock_queue", name);        timeoutSetName = prefixName("redisson_lock_timeout", name);    }        @Override    protected RFuture<Boolean> unlockInnerAsync(long threadId) {        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,            //步骤一：移除等待超时的线程            "while true do " +                //获取队列中的第一个元素                //KEYS[2]是一个用来对线程排队的队列的名字                "local firstThreadId2 = redis.call('lindex', KEYS[2], 0);" +                "if firstThreadId2 == false then " +                    "break;" +                "end; " +                //获取队列中第一个元素对应的分数，也就是排第一的线程的过期时间                //KEYS[3]是一个用来对线程排序的有序集合的名字                "local timeout = tonumber(redis.call('zscore', KEYS[3], firstThreadId2));" +                //如果排第一的线程的过期时间小于当前时间，说明该线程等待超时了都还没获取到锁，所以要移除                //ARGV[4]是当前时间                "if timeout <= tonumber(ARGV[4]) then " +                    //从有序集合 + 队列中移除这个线程                    "redis.call('zrem', KEYS[3], firstThreadId2); " +                    "redis.call('lpop', KEYS[2]); " +                "else " +                    "break;" +                "end; " +            "end;" +
            //步骤二：判断锁是否还存在，判断key为锁名的Hash值是否存在            "if (redis.call('exists', KEYS[1]) == 0) then " +                //获取队列中排第一的线程                "local nextThreadId = redis.call('lindex', KEYS[2], 0); " +                 "if nextThreadId ~= false then " +                    //ARGV[1]为通知事件的类型                    "redis.call('publish', KEYS[4] .. ':' .. nextThreadId, ARGV[1]); " +                "end; " +                "return 1; " +            "end;" +
            //步骤二：判断锁是否还存在，判断key为UUID+线程ID的Hash值是否存在            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +                "return nil;" +            "end; " +            //对key为UUID+线程ID的Hash值还存递减1            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +            "if (counter > 0) then " +                "redis.call('pexpire', KEYS[1], ARGV[2]); " +                "return 0; " +            "end; " +                            "redis.call('del', KEYS[1]); " +            "local nextThreadId = redis.call('lindex', KEYS[2], 0); " +             "if nextThreadId ~= false then " +                //发布一个事件给在队列中排第一的线程                "redis.call('publish', KEYS[4] .. ':' .. nextThreadId, ARGV[1]); " +            "end; " +            "return 1; ",            Arrays.asList(getRawName(), threadsQueueName, timeoutSetName, getChannelName()),            LockPubSub.UNLOCK_MESSAGE,//ARGV[1]            internalLockLeaseTime,             getLockName(threadId),             System.currentTimeMillis()        );    }    ...}
```

**(2)释放公平锁的lua脚本分析****
**

**步骤一：****移除等待超时的线程**

首先也会进入while循环，移除等待超时的线程。即获取队列中排第一的线程，判断该线程的过期时间是否已小于当前时间。如果小于当前时间，那么就说明该线程在队列中的排队已经过期，于是便将该线程从有序集合 + 队列中移除。后续如果该线程再次尝试加锁，那么会重新排序 + 重新入队。

**
**

**步骤二：****判断锁是否还存在**

如果key为锁名的Hash值已不存在，那么先获取队列中排第一的线程，然后发布一个事件给该线程对应的客户端让其获取锁。



如果key为锁名的Hash值还存在，那么判断field为UUID + 线程ID的映射是否存在。如果field为UUID + 线程ID的映射不存在，那么表示锁已经被释放了，直接返回nil。如果field为UUID + 线程ID的映射存在，那么在key为锁名的Hash值中，对field为UUID + 线程ID的value值递减1。也就是调用Redis的hincrby命令，进行递减1处理。

**
**

**步骤三：****对递减1后的结果进行如下判断处理**

如果递减1后的结果大于0，表示线程还在持有锁。对应于持有锁的线程多次重入锁，此时需要重置锁的过期时间。



如果递减1后的结果小于0，表示线程不再持有锁，则删除锁对应的key，并且发布一个事件给在队列中排第一的线程所对应的客户端。

**
**

**12.****公平锁源码之按顺序依次加锁**

**(1)锁被释放后，排第二的客户端线程先来加锁**

**(2)锁被释放后，排第一的客户端线程再来加锁**

**
**

假设客户端A先持有锁，而客户端B在队列里面是排在客户端C的后面。那么如果客户端A释放了锁后，客户端B和C是如何按顺序加锁的。

**
**

**(1)锁被释放后，排第二的客户端线程先来加锁**

锁被客户端A释放掉，锁key被删除之后，客户端B先来进行尝试加锁。此时客户端B执行的lua脚本步骤二的逻辑：

- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//check if the lock can be acquired now//步骤二：判断当前线程现在能否尝试获取锁，以下两种情况可以通过判断去进行尝试获取锁//情况一：锁不存在 + 队列也不存在；KEYS[1]是锁的名字；//情况二：锁不存在 + 队列存在 + 队列的第一个元素就是当前线程；ARGV[2]是当前线程的UUID + ThreadID；"if (redis.call('exists', KEYS[1]) == 0) " +    "and ((redis.call('exists', KEYS[2]) == 0) " +    "or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then " +    ..."end;"
```

首先，执行判断"exists myLock = 0"，由于当前锁存在，所以条件不成立。



然后，执行判断"exists redisson_lock_queue:{myLock} = 0"，由于队列存在，所以条件不成立。



接着，执行判断"lindex redisson_lock_queue:{myLock} 0 == UUID2:ThreadID2"，由于队列存在，但是在队列中排第一的不是客户端B而是客户端C，所以条件不成立，客户端B无法加锁。



由此可见：即使锁释放掉后，多个客户端来尝试加锁也只认队列中排第一的客户端。从而实现按队列的顺序依次获取锁，保证了公平性。

**
**

**(2)锁被释放后，排第一的客户端线程再来加锁**

当在队列中排第一的客户端C此时过来尝试加锁时，就会执行如下步骤三的尝试加锁逻辑：

- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 

```
//check if the lock can be acquired now//步骤二：判断当前线程现在能否尝试获取锁，以下两种情况可以通过判断去进行尝试获取锁//情况一：锁不存在 + 队列也不存在；KEYS[1]是锁的名字；KEYS[2]是对线程排队的队列；//情况二：锁不存在 + 队列存在 + 队列的第一个元素就是当前线程；ARGV[2]是当前线程的UUID+ThreadID；"if (redis.call('exists', KEYS[1]) == 0) " +    "and ((redis.call('exists', KEYS[2]) == 0) " +        "or (redis.call('lindex', KEYS[2], 0) == ARGV[2])) then " +    //步骤三：当前线程执行获取锁的操作    //remove this thread from the queue and timeout set    //弹出队列的第一个元素 + 从有序集合中删除UUID:ThreadID对应的元素    "redis.call('lpop', KEYS[2]);" +    "redis.call('zrem', KEYS[3], ARGV[2]);" +    //decrease timeouts for all waiting in the queue    //递减有序集合中每个线程的分数，也就是递减每个线程获取锁时的已经等待时间    //zrange返回有序集合KEYS[3]中指定区间内(0,-1)的成员，也就是全部成员    "local keys = redis.call('zrange', KEYS[3], 0, -1);" +    "for i = 1, #keys, 1 do " +        //对有序集合KEYS[3]的成员keys[i]的score减去：tonumber(ARGV[3])        //ARGV[3]就是线程获取锁时可以等待的时间，默认是5分钟        "redis.call('zincrby', KEYS[3], -tonumber(ARGV[3]), keys[i]);" +    "end;" +    //acquire the lock and set the TTL for the lease    //hset设置Hash值进行加锁操作 + pexpire设置锁key的过期时间 + 最后返回nil表示加锁成功    "redis.call('hset', KEYS[1], ARGV[2], 1);" +    "redis.call('pexpire', KEYS[1], ARGV[1]);" +    "return nil;" +"end;"
```

首先，执行命令"lpop redisson_lock_queue:{myLock}"，将队列中的第一个元素弹出来。



然后，执行命令"zrem redisson_lock_timeout:{myLock} UUID3:ThreadID3"，将有序集合中客户端C的线程对应的元素给删除掉。



接着，执行"hset myLock UUID3:ThreadID3 1"进行加锁，设置field为UUID + 线程ID的value值为1。



最后，执行命令"pexpire myLock 30000"，设置key为锁名的Hash值的过期时间为30000毫秒。



客户端C完成加锁后，客户端C就会从队列中出队，此时排在队头的就是客户端B。
