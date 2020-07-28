针对项目中使用的分布式锁进行简单的示例配置以及源码解析，并列举源码中使用到的一些基础知识点，但是没有对redisson中使用到的netty知识进行解析。

本篇主要是对以下几个方面进行了探索

> * Maven配置
> * RedissonLock简单示例
> * 源码中使用到的Redis命令
> * 源码中使用到的lua脚本语义
> * 源码分析

## Maven配置 {#Maven配置}

```
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>2.2.12</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.6.0</version>
</dependency>
```

## RedissonLock简单示例 {#RedissonLock简单示例}

redission支持4种连接redis方式，分别为单机、主从、Sentinel、Cluster 集群，项目中使用的连接方式是Sentinel。  
redis服务器不在本地的同学请注意权限问题。

### Sentinel配置 {#Sentinel配置}

```
Config config = new Config();
config.useSentinelServers().addSentinelAddress("127.0.0.1:6479", "127.0.0.1:6489").setMasterName("master").setPassword("password").setDatabase(0);
RedissonClient redisson = Redisson.create(config);
```

### 简单使用 {#Sentinel配置}

```
RLock lock = redisson.getLock("test_lock");
try{
    boolean isLock=lock.tryLock();
    if(isLock){
        doBusiness();
    }
}catch(exception e){
}finally{
    lock.unlock();
}
```

### 源码中使用到的Redis命令 {#Sentinel配置}

分布式锁主要需要以下redis命令，这里列举一下。在源码分析部分可以继续参照命令的操作含义。

1. EXISTS key :当 key 存在，返回1；若给定的 key 不存在，返回0。
2. GETSET key value:将给定 key 的值设为 value ，并返回 key 的旧值 \(old value\)，当 key 存在但不是字符串类型时，返回一个错误，当key不存在时，返回nil。
3. GET key:返回 key 所关联的字符串值，如果 key 不存在那么返回 nil。
4. DEL key \[KEY …\]:删除给定的一个或多个 key ,不存在的 key 会被忽略,返回实际删除的key的个数（integer）。
5. HSET key field value：给一个key 设置一个{field=value}的组合值，如果key没有就直接赋值并返回1，如果field已有，那么就更新value的值，并返回0.
6. HEXISTS key field:当key中存储着field的时候返回1，如果key或者field至少有一个不存在返回0。
7. HINCRBY key field increment:将存储在key中的哈希（Hash）对象中的指定字段field的值加上增量increment。如果键key不存在，一个保存了哈希对象的新建将被创建。如果字段field不存在，在进行当前操作前，其将被创建，且对应的值被置为0，返回值是增量之后的值
8. PEXPIRE key milliseconds：设置存活时间，单位是毫秒。expire操作单位是秒。
9. PUBLISH channel message:向channel post一个message内容的消息，返回接收消息的客户端数。

## 源码中使用到的lua脚本语义 {#源码中使用到的lua脚本语义}

Redisson源码中，执行redis命令的是lua脚本，其中主要用到如下几个概念。

> * redis.call\(\) 是执行redis命令.
> * KEYS\[1\] 是指脚本中第1个参数
> * ARGV\[1\] 是指脚本中第一个参数的值
> * 返回值中nil与false同一个意思。

需要注意的是，在redis执行lua脚本时，相当于一个redis级别的锁，不能执行其他操作，类似于原子操作，也是redisson实现的一个关键点。  
另外，如果lua脚本执行过程中出现了异常或者redis服务器直接宕掉了，执行redis的根据日志回复的命令，会将脚本中已经执行的命令在日志中删除。

## 源码分析 {#源码分析}

### RLOCK结构 {#RLOCK结构}

```
public interface RLock extends Lock, RExpirable {
    void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException;
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
    void lock(long leaseTime, TimeUnit unit);
    void forceUnlock();
    boolean isLocked();
    boolean isHeldByCurrentThread();
    int getHoldCount();
    Future<Void> unlockAsync();
    Future<Boolean> tryLockAsync();
    Future<Void> lockAsync();
    Future<Void> lockAsync(long leaseTime, TimeUnit unit);
    Future<Boolean> tryLockAsync(long waitTime, TimeUnit unit);
    Future<Boolean> tryLockAsync(long waitTime, long leaseTime, TimeUnit unit);
}
```

该接口主要继承了Lock接口, 并扩展了部分方法, 比如:boolean tryLock\(long waitTime, long leaseTime, TimeUnit unit\)新加入的leaseTime主要是用来设置锁的过期时间, 如果超过leaseTime还没有解锁的话, redis就强制解锁. leaseTime的默认时间是30s

### RedissonLock获取锁 tryLock源码 {#RedissonLock获取锁-tryLock源码}

```
Future<Long> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId) {
       internalLockLeaseTime = unit.toMillis(leaseTime);
       return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_LONG,
                 "if (redis.call('exists', KEYS[1]) == 0) then " +
                     "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                     "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                     "return nil; " +
                 "end; " +
                 "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                     "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                     "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                     "return nil; " +
                 "end; " +
                 "return redis.call('pttl', KEYS[1]);",
                   Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
   }
```

其中  
KEYS\[1\] 表示的是 getName\(\) ，代表的是锁名 test\_lock  
ARGV\[1\] 表示的是 internalLockLeaseTime 默认值是30s  
ARGV\[2\] 表示的是 getLockName\(threadId\) 代表的是 id:threadId 用锁对象id+线程id， 表示当前访问线程，用于区分不同服务器上的线程.  
逐句分析：

```
if (redis.call('exists', KEYS[1]) == 0) then 
         redis.call('hset', KEYS[1], ARGV[2], 1); 
         redis.call('pexpire', KEYS[1], ARGV[1]); 
         return nil;
         end;
```

> * if \(redis.call\(‘exists’, KEYS\[1\]\) == 0\) 如果锁名称不存在
> * then redis.call\(‘hset’, KEYS\[1\], ARGV\[2\],1\) 则向redis中添加一个key为test\_lock的set，并且向set中添加一个field为线程id，值=1的键值对，表示此线程的重入次数为1
> * redis.call\(‘pexpire’, KEYS\[1\], ARGV\[1\]\) 设置set的过期时间，防止当前服务器出问题后导致死锁，return nil; end;返回nil 结束
>
> ```
> if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then 
>          redis.call('hincrby', KEYS[1], ARGV[2], 1); 
>          redis.call('pexpire', KEYS[1], ARGV[1]);
>          return nil; 
>          end;
> ```
>
> * if \(redis.call\(‘hexists’, KEYS\[1\], ARGV\[2\]\) == 1\) 如果锁是存在的，检测是否是当前线程持有锁，如果是当前线程持有锁
> * then redis.call\(‘hincrby’, KEYS\[1\], ARGV\[2\], 1\)则将该线程重入的次数++
> * redis.call\(‘pexpire’, KEYS\[1\], ARGV\[1\]\) 并且重新设置该锁的有效时间
> * return nil; end;返回nil，结束

```
return redis.call('pttl', KEYS[1]);
```

> * 锁存在, 但不是当前线程加的锁，则返回锁的过期时间

### RedissonLock解锁 unlock源码 {#RedissonLock解锁-unlock源码}

```
@Override
    public void unlock() {
        Boolean opStatus = commandExecutor.evalWrite(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                        "if (redis.call('exists', KEYS[1]) == 0) then " +
                            "redis.call('publish', KEYS[2], ARGV[1]); " +
                            "return 1; " +
                        "end;" +
                        "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                            "return nil;" +
                        "end; " +
                        "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                        "if (counter > 0) then " +
                            "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                            "return 0; " +
                        "else " +
                            "redis.call('del', KEYS[1]); " +
                            "redis.call('publish', KEYS[2], ARGV[1]); " +
                            "return 1; "+
                        "end; " +
                        "return nil;",
                        Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(Thread.currentThread().getId()));
        if (opStatus == null) {
            throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                    + id + " thread-id: " + Thread.currentThread().getId());
        }
        if (opStatus) {
            cancelExpirationRenewal();
        }
    }
```

其中  
KEYS\[1\] 表是的是getName\(\) 代表锁名test\_lock  
KEYS\[2\] 表示getChanelName\(\) 表示的是发布订阅过程中使用的Chanel  
ARGV\[1\] 表示的是LockPubSub.unLockMessage 是解锁消息，实际代表的是数字 0，代表解锁消息  
ARGV\[2\] 表示的是internalLockLeaseTime 默认的有效时间 30s  
ARGV\[3\] 表示的是getLockName\(thread.currentThread\(\).getId\(\)\)，是当前锁id+线程id  
语义分析:

```
if (redis.call('exists', KEYS[1]) == 0) then
         redis.call('publish', KEYS[2], ARGV[1]);
         return 1;
         end;
```

> * if \(redis.call\(‘exists’, KEYS\[1\]\) == 0\) 如果锁已经不存在\(可能是因为过期导致不存在，也可能是因为已经解锁\)
> * then redis.call\(‘publish’, KEYS\[2\], ARGV\[1\]\) 则发布锁解除的消息
> * return 1; end 返回1结束

```
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then 
         return nil;
         end;
```

> * if \(redis.call\(‘hexists’, KEYS\[1\], ARGV\[3\]\) == 0\) 如果锁存在，但是若果当前线程不是加锁的线
> * then return nil;end 则直接返回nil 结束

```
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
if (counter > 0) then
         redis.call('pexpire', KEYS[1], ARGV[2]); 
         return 0;
else
         redis.call('del', KEYS[1]);
         redis.call('publish', KEYS[2], ARGV[1]);
         return 1;
end;
```

> * local counter = redis.call\(‘hincrby’, KEYS\[1\], ARGV\[3\], -1\) 如果是锁是当前线程所添加，定义变量counter，表示当前线程的重入次数-1,即直接将重入次数-1
> * if \(counter &gt; 0\)如果重入次数大于0，表示该线程还有其他任务需要执行
> * then redis.call\(‘pexpire’, KEYS\[1\], ARGV\[2\]\) 则重新设置该锁的有效时间
> * return 0 返回0结束
> * else redis.call\(‘del’, KEYS\[1\]\) 否则表示该线程执行结束，删除该锁
> * redis.call\(‘publish’, KEYS\[2\], ARGV\[1\]\) 并且发布该锁解除的消息
> * return 1; end;返回1结束

```
return nil;
```

> * 其他情况返回nil并结束

```
if (opStatus == null) {
            throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                    + id + " thread-id: " + Thread.currentThread().getId());
        }
```

> * 脚本执行结束之后，如果返回值不是0或1，即当前线程去解锁其他线程的加锁时，抛出异常

## RedissonLock强制解锁源码 {#RedissonLock强制解锁源码}

```
@Override
    public void forceUnlock() {
        get(forceUnlockAsync());
    }
    Future<Boolean> forceUnlockAsync() {
        cancelExpirationRenewal();
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('del', KEYS[1]) == 1) then "
                + "redis.call('publish', KEYS[2], ARGV[1]); "
                + "return 1 "
                + "else "
                + "return 0 "
                + "end",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage);
    }
```

> * 以上是强制解锁的源码,在源码中并没有找到forceUnlock\(\)被调用的痕迹\(也有可能是我没有找对\),但是forceUnlockAsync\(\)方法被调用的地方很多，大多都是在清理资源时删除锁。此部分比较简单粗暴，删除锁成功则并发布锁被删除的消息，返回1结束，否则返回0结束。

## 总结 {#总结}

这里只是简单的一个redisson分布式锁的测试用例，并分析了执行lua脚本这部分，如果要继续分析执行结束之后的操作，需要进行netty源码分析 ，redisson使用了netty完成异步和同步的处理。

