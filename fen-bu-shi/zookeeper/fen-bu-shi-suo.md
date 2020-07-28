锁是开发过程中十分常见的工具，在处理高并发请求的时候和订单数据的时候往往需要锁来帮助我们保证数据的安全。

场景1.前端点击太快，导致后端重复调用接口。两次调用一个接口，这样就会产生同一个请求执行了两次，而从用户的角度出发，他是因为太卡而点了两次，他的目标是执行一次请求。

场景2.对于高并发场景，我们往往需要引入分布式缓存，来加快整个系统的响应速度。但是缓存是有失效机制的，如果某一时刻缓存失效，而此时有大量的请求过来，那么所有的请求会瞬间直接打到DB上，那么这么大的并发量，DB可能是扛不住的。那么这里需要引入一个保护机制。当发生“缓存击穿”的时候加锁，从而保护DB不被拖垮。

看完了上面的场景，其实分布式锁的场景一直在我们身边。说分布式锁之前，应该先说一下java提供的锁，比较能单机解决的并发问题，没必要引入分布式的解决方案。

java提供了两种内置的锁的实现，一种是由JVM实现的synchronized和JDK提供的Lock，当你的应用是单机或者说单进程应用时，可以使用synchronized或Lock来实现锁。

但是，当你的应用涉及到多机、多进程共同完成时，例如现在的互联网架构，一般都是分布式的RPC框架来支撑，那么这样你的Server有多个，由于负载均衡的路由规则随机，相同的请求可能会打到不同的Server上进行处理，那么这时候就需要一个全局锁来实现多个线程\(不同的进程\)之间的同步。

实现全局的锁需要依赖一个第三方系统，此系统需要满足高可用、一致性比较强同时能应付高并发的请求。

常见的处理办法有三种：数据库、缓存、分布式协调系统。数据库和缓存是比较常用的，但是分布式协调系统是不常用的。

#### 数据库实现分布式锁

利用DB来实现分布式锁，有两种方案。两种方案各有好坏，但是总体效果都不是很好。但是实现还是比较简单的。

1. 利用主键唯一规则：

   我们知道数据库是有唯一主键规则的，主键不能重复，对于重复的主键会抛出主键冲突异常。

   了解JDK reentrantlock的人都知道，reentrantlock是利用了OS的CAS特性实现的锁。主要是维护一个全局的状态，每次竞争锁都会CAS修改锁的状态，修改成功之后就占用了锁，失败的加入到同步队列中，等待唤醒。

   其实这和分布式锁实现方案基本是一致的，首先我们利用主键唯一规则，在争抢锁的时候向DB中写一条记录，这条记录主要包含锁的id、当前占用锁的线程名、重入的次数和创建时间等，如果插入成功表示当前线程获取到了锁，如果插入失败那么证明锁被其他人占用，等待一会儿继续争抢，直到争抢到或者超时为止。

这里我主要写了一个简单的实现：

```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

/**
 * 利用mysql实现可重入分布式锁
 */
public class MysqlprimaryLock {
    private static Connection connection;
    static {
        try {
            Class.forName("com.mysql.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        String url = "jdbc:mysql://10.0.0.212:3308/dbwww_lock?user=lock_admin&password=lock123";
        try {
            connection = DriverManager.getConnection(url);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    /**
     * 加锁
     * @param lockID
     */
    public void lock(String lockID) {
        acquire(lockID);
    }

    /**
     * 获取锁
     * @param lockID
     * @return
     */
    public boolean acquire(String lockID) {
        String sql = "insert into test_lock('id','count','thName','addtime') VALUES (?,?,?,?)";
        while (true) {
            try {
                PreparedStatement statement = connection.prepareStatement(sql);
                statement.setString(1, lockID);
                statement.setInt(2, 1);
                statement.setLong(1, System.currentTimeMillis());
                boolean ifsucess = statement.execute();//如果成功，那么就是获取到了锁
                if (ifsucess)
                    return true;
            } catch (SQLException e) {
                e.printStackTrace();
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            continue;
        }
    }

    /**
     * 超时获取锁
     * @param lockID
     * @param timeOuts
     * @return
     * @throws InterruptedException
     */
    public boolean acquire(String lockID, long timeOuts) throws InterruptedException {

        String sql = "insert into test_lock('id','count','thName','addtime') VALUES (?,?,?,?)";
        long futureTime = System.currentTimeMillis() + timeOuts;
        long ranmain = timeOuts;
        long timerange = 500;
        while (true) {
            CountDownLatch latch = new CountDownLatch(1);
            try {
                PreparedStatement statement = connection.prepareStatement(sql);
                statement.setString(1, lockID);
                statement.setInt(2, 1);
                statement.setLong(1, System.currentTimeMillis());
                boolean ifsucess = statement.execute();//如果成功，那么就是获取到了锁
                if (ifsucess)
                    return true;
            } catch (SQLException e) {
                e.printStackTrace();
            }
            latch.await(timerange, TimeUnit.MILLISECONDS);
            ranmain = futureTime - System.currentTimeMillis();
            if (ranmain <= 0)
                break;
            if (ranmain < timerange) {
                timerange = ranmain;
            }
            continue;
        }
        return false;

    }

    /**
     * 释放锁
     * @param lockID
     * @return
     * @throws SQLException
     */
    public boolean unlock(String lockID) throws SQLException {
        String sql = "DELETE  from test_lock where id = ?";
        PreparedStatement statement = connection.prepareStatement(sql);
        statement.setString(1, lockID);
        boolean ifsucess = statement.execute();
        if (ifsucess)
            return true;
        return false;

    }

}
```

这里是利用主键冲突规则，加入了id','count','thName','addtime'，count主要是为了重入计数，thName为了判断占用锁的线程，addtime是记录占用时间。上面代码没有实现重入的逻辑。

重入主要实现思路是，在每次获取锁之前去取当前锁的信息，如果锁的线程是当前线程，那么更新锁的count+1，并且执行锁之后的逻辑。如果不是当前锁，那么进行重试。释放的时候也要进行count-1，最后减到0时，删除锁标识释放锁。

优点：实现简单

缺点：没有超时保护机制，mysql存在单点，并发量大的时候请求量太大、没有线程唤醒机制，用异常去控制逻辑多少优点恶心。

对于超时保护：如果可能，可以采用定时任务去扫描超过一定阈值的锁，并删除。但是也会存在，锁住的任务执行时间很长，删除锁会导致并发问题。所以需要对超时时间有一个很好的预估。

对于单点问题：有条件可以搞一个主从，但是为了一个锁来搞一个主从是不是优点浪费？同时主从切换的时候系统不可用，这也是一个问题。

并发量大的时候请求量太大：因为这种实现方式是没有锁的唤醒机制的，不像reentrantlock在同步队列中的节点，可以通过唤醒来避免多次的循环请求。但是分布式环境数据库这种锁的实现是不能做到唤醒的。所以只能将获取锁的时间间隔调高，避免死循环给系统和DB带来的巨大压力。这样也牺牲了系统的吞吐量，因为总会有一定的间隔锁是空闲的。

用异常去控制逻辑多少优点恶心：就不说了，每次失败都抛异常.....

1. 利用Mysql行锁的特性：

## 基于数据库排他锁做分布式锁

在查询语句后面增加`for update`，数据库会在查询过程中给数据库表增加排他锁 \(**注意： InnoDB 引擎在加锁的时候，只有通过索引进行检索的时候才会使用行级锁，否则会使用表级锁。这里我们希望使用行级锁，就要给要执行的方法字段名添加索引，值得注意的是，这个索引一定要创建成唯一索引，否则会出现多个重载方法之间无法同时被访问的问题。重载方法的话建议把参数类型也加上。**\)。当某条记录被加上排他锁之后，其他线程无法再在该行记录上增加排他锁。

Mysql是有表锁、页锁和行锁的机制的，可以利用这个机制来实现锁。这里尽量使用行锁，它的吞吐量是最高的。

```
/**
     * 超时获取锁
     * @param lockID
     * @param timeOuts
     * @return
     * @throws InterruptedException
     */
    public boolean acquireByUpdate(String lockID, long timeOuts) throws InterruptedException, SQLException {

        String sql = "SELECT id from test_lock where id = ? for UPDATE ";
        long futureTime = System.currentTimeMillis() + timeOuts;
        long ranmain = timeOuts;
        long timerange = 500;
        connection.setAutoCommit(false);
        while (true) {
            CountDownLatch latch = new CountDownLatch(1);
            try {
                PreparedStatement statement = connection.prepareStatement(sql);
                statement.setString(1, lockID);
                statement.setInt(2, 1);
                statement.setLong(1, System.currentTimeMillis());
                boolean ifsucess = statement.execute();//如果成功，那么就是获取到了锁
                if (ifsucess)
                    return true;
            } catch (SQLException e) {
                e.printStackTrace();
            }
            latch.await(timerange, TimeUnit.MILLISECONDS);
            ranmain = futureTime - System.currentTimeMillis();
            if (ranmain <= 0)
                break;
            if (ranmain < timerange) {
                timerange = ranmain;
            }
            continue;
        }
        return false;

    }


    /**
     * 释放锁
     * @param lockID
     * @return
     * @throws SQLException
     */
    public void unlockforUpdtate(String lockID) throws SQLException {
        connection.commit();

    }
```

利用for update加显式的行锁，这样就能利用这个行级的排他锁来实现分布式锁了，同时unlock的时候只要释放commit这个事务，就能达到释放锁的目的。

优点：实现简单

缺点：连接池爆满和事务超时的问题单点的问题，单点问题，行锁升级为表锁的问题，并发量大的时候请求量太大、没有线程唤醒机制。

连接池爆满和事务超时的问题单点的问题：利用事务进行加锁的时候，query需要占用数据库连接，在行锁的时候连接不释放，这就会导致连接池爆满。同时由于事务是有超时时间的，过了超时时间自动回滚，会导致锁的释放，这个超时时间要把控好。

对于单点问题：同上。

并发量大的时候请求量太大：同上。

行锁升级为表锁的问题：Mysql行锁默认需要走索引，如果不走索引会导致锁表，如果可以，在sql中可以强制指定索引。

#### 缓存分布式锁

缓存实现分布式锁还是比较常见的，因为缓存比较轻量，并且缓存的响应快、吞吐高。最重要的是还有自动失效的机制来保证锁一定能释放。

缓存的分布式锁主要通过Redis实现，当然其他的缓存也是可以的。关于缓存有两种实现吧：

1. 基于SetNX实现：

   setNX是Redis提供的一个原子操作，如果指定key存在，那么setNX失败，如果不存在会进行Set操作并返回成功。我们可以利用这个来实现一个分布式的锁，主要思路就是，set成功表示获取锁，set失败表示获取失败，失败后需要重试。

具体看下代码：

```
import redis.clients.jedis.Jedis;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Redis分布式锁
 */
public class RedisLockTest {

    private Jedis jedisCli = new Jedis("localhost",6381);

    private int expireTime = 1;

    /**
     * 获取锁
     * @param lockID
     * @return
     */
    public boolean lock(String lockID){
        while(true){
            long returnFlag = jedisCli.setnx(lockID,"1");
            if (returnFlag == 1){
                System.out.println(Thread.currentThread().getName() + " get lock....");
                return true;
            }
            System.out.println(Thread.currentThread().getName() + " is trying lock....");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            }
        }
    }

    /**
     * 超时获取锁
     * @param lockID
     * @param timeOuts
     * @return
     */
    public boolean lock(String lockID,long timeOuts){
        long current = System.currentTimeMillis();
        long future = current + timeOuts;
        long timeStep = 500;
        CountDownLatch latch = new CountDownLatch(1);
        while(future > current){
            long returnFlag = jedisCli.setnx(lockID,"1");
            if (returnFlag == 1){
                System.out.println(Thread.currentThread().getName() + " get lock....");
                jedisCli.expire(lockID,expireTime);
                return true;
            }
            System.out.println(Thread.currentThread().getName() + " is trying lock....");
            try {
                latch.await(timeStep, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            current = current + timeStep;
        }
        return false;
    }

    public void unlock(String lockId){
        long flag = jedisCli.del(lockId);
        if (flag>0){
            System.out.println(Thread.currentThread().getName() + " release lock....");
        }else {
            System.out.println(Thread.currentThread().getName() + " release lock fail....");
        }
    }

    /**
     * 线程工厂,命名线程
     */
    public static class MyThreadFactory implements ThreadFactory{
        public static AtomicInteger count = new AtomicInteger();
        @Override
        public Thread newThread(Runnable r) {
            count.getAndIncrement();
            Thread thread = new Thread(r);
            thread.setName("Thread-lock-test "+count);
            return thread;
        }
    }

    public static void main(String args[]){
        final String lockID = "test1";
        Runnable task = () ->{
            RedisLockTest testCli = new RedisLockTest();
            testCli.lock(lockID);
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            testCli.unlock(lockID);
        };

        MyThreadFactory factory = new MyThreadFactory();
        ExecutorService services = Executors.newFixedThreadPool(10);
        for (int i = 0;i<3;i++)
            services.execute(factory.newThread(task));
    }

}
```

看看结果：

```
pool-1-thread-3 is trying lock....
pool-1-thread-2 get lock....
pool-1-thread-1 is trying lock....
pool-1-thread-3 is trying lock....
pool-1-thread-2 release lock....
pool-1-thread-1 get lock....
pool-1-thread-3 is trying lock....
pool-1-thread-1 release lock....
pool-1-thread-3 get lock....
pool-1-thread-3 release lock....
```

可以看到，几个线程很好的进行了同步。

这种方式也是有优点和缺点：

优点：实现简单，吞吐量十分客观，对于高并发情况应付自如，自带超时保护，对于网络抖动的情况也可以利用超时删除策略保证不会阻塞所有流程。

缺点：单点问题、没有线程唤醒机制、网络抖动可能会引起锁删除失败。

对单点问题：因为redis一般都是单实例使用，那么对于单点问题，可以做一个主从。当然主从切换的时候也是不可用的，因为主从同步是异步的，可能会并发问题。如果对于主从还是不能保证可靠性的话，可以上Redis集群，对于Redis集群，因为使用了类一致性Hash算法，虽然不能避免节点下线的并发问题\(当前的任务没有执行完，其他任务就开始执行\)，但是能保证Redis是可用的。**可用性的问题是出了问题之后的备选方案，如果我们系统天天都出问题还玩毛啊，对于突发情况牺牲一两个请求还是没问题的。**

对于线程唤醒机制：分布式锁大多都是这样轮训获取锁的，所以控制住你的重试频率，也不会导致负载特别高的。可能就是吞吐量低点而已。

对于锁删除失败：分布式锁基本都有这个问题，可以对key设置失效时间。这个超时时间需要把控好，过大那么系统吞吐量低，很容易导致超时。如果过小那么会有并发问题，部分耗时时间比较长的任务就要遭殃了。

#### 基于Zookeeper的分布式锁

Zookeeper是一个分布式一致性协调框架，主要可以实现选主、配置管理和分布式锁等常用功能，因为Zookeeper的写入都是顺序的，在一个节点创建之后，其他请求再次创建便会失败，同时可以对这个节点进行Watch，如果节点删除会通知其他节点抢占锁。

Zookeeper实现分布式锁虽然是比较重量级的，但实现的锁功能十分健全，由于Zookeeper本身需要维护自己的一致性，所以性能上较Redis还是有一定差距的。

Zookeeper实现分布式锁有几种形式，后面会单独的总结一下。

#### 对比：

Mysql实现比较简单，不需要引入第三个应用，但实现多少有些重，性能不是很好。  
 Redis的话实现比较简单，同时性能很好，引入集群可以提高可用性。同时定期失效的机制可以解决因网络抖动锁删除失败的问题，所以我比较倾向Redis实现。  
 Zookeeper实现是有些重的，同时我们还需要维护Zookeeper集群，实现起来还是比较复杂的，实现不好的话还会引起“羊群效应”。如果不是原有系统就依赖Zookeeper，同时压力不大的情况下。一般不使用Zookeeper实现分布式锁。  
[https://zhuanlan.zhihu.com/p/42056183](https://zhuanlan.zhihu.com/p/42056183)

[链接：https://www.jianshu.com/p/c2b4aa7a12f1](/链接：https://www.jianshu.com/p/c2b4aa7a12f1 "转载")

