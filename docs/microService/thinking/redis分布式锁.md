

<!-- TOC -->

- [1. Redis分布式锁](#1-redis分布式锁)
    - [1.1. 前言：实现分布式锁需要关注哪些细节呢？](#11-前言实现分布式锁需要关注哪些细节呢)
    - [1.2. 使用Redis分布式锁中的问题](#12-使用redis分布式锁中的问题)
    - [1.3. Redis Client原生API](#13-redis-client原生api)
        - [1.3.1. 单实例redis实现分布式锁](#131-单实例redis实现分布式锁)
            - [1.3.1.1. 加锁](#1311-加锁)
            - [1.3.1.2. 解锁](#1312-解锁)
        - [1.3.2. 集群redlock算法实现分布式锁](#132-集群redlock算法实现分布式锁)
    - [1.4. Redisson实现redis分布式锁](#14-redisson实现redis分布式锁)
        - [1.4.1. ※※※Redisson解决死锁问题(watch dog自动延期机制)](#141-※※※redisson解决死锁问题watch-dog自动延期机制)
        - [1.4.2. 重入锁](#142-重入锁)
            - [1.4.2.1. 重入锁解析](#1421-重入锁解析)
                - [1.4.2.1.1. 获取锁tryLock](#14211-获取锁trylock)
                - [1.4.2.1.2. 解锁unlock](#14212-解锁unlock)
            - [1.4.2.2. 重入锁缺点](#1422-重入锁缺点)
        - [1.4.3. 公平锁（Fair Lock）](#143-公平锁fair-lock)
        - [1.4.4. 联锁（MultiLock）](#144-联锁multilock)
        - [1.4.5. 红锁（RedLock）](#145-红锁redlock)
        - [1.4.6. 读写锁（ReadWriteLock）](#146-读写锁readwritelock)
        - [1.4.7. 信号量（Semaphore）](#147-信号量semaphore)

<!-- /TOC -->

# 1. Redis分布式锁  
&emsp; **<font color = "lime">总结：</font>**  
&emsp; **<font color = "lime">RedissonLock锁互斥、自动延期机制、可重入加锁。</font>**  
* 自动延期  
&emsp; <font color = "lime">只要客户端一旦加锁成功，就会启动一个后台线程，会每隔10秒检查一下，如果客户端1还持有锁key，那么就会不断的延长锁key的生存时间。</font>  

&emsp; RedissonLock加锁流程：  
1. 执行lock.lock()代码时，<font color = "red">如果该客户端面对的是一个redis cluster集群，首先会根据hash节点选择一台机器。</font>  
2. 然后发送一段lua脚本，带有三个参数：一个是锁的名字（在代码里指定的）、一个是锁的时常（默认30秒）、一个是加锁的客户端id（每个客户端对应一个id）。<font color = "red">然后脚本会判断是否有该名字的锁，如果没有就往数据结构中加入该锁的客户端id。</font>  

    * 锁不存在（exists），则加锁（hset），并设置（pexpire）锁的过期时间；  
    * 锁存在，检测（hexists）是当前线程持有锁，锁重入（hincrby），并且重新设置（pexpire）该锁的有效时间；
    * 锁存在，但不是当前线程的，返回（pttl）锁的过期时间。 

## 1.1. 前言：实现分布式锁需要关注哪些细节呢？  

* 确保互斥：在同一时刻，必须保证锁至多只能被一个客户端持有。  
* 不能死锁：在一个客户端在持有锁的期间崩溃而没有主动解锁情况下，也能保证后续其他客户端能加锁。    
* 避免活锁：在获取锁失败的情况下，反复进行重试操作，占用Cpu资源，影响性能。    
* 实现更多锁特性：锁中断、锁重入、锁超时等。确保客户端只能解锁自己持有的锁。  

## 1.2. 使用Redis分布式锁中的问题  
1. 超时问题。  
2. 部署问题：除了要考虑客户端要怎么实现分布式锁之外，还需要考虑Redis的部署问题。Redis有多种部署方式：单机模式；Master-Slave+Sentinel选举模式；Redis Cluster模式。  
    * 如果采用单机部署模式，会存在单点问题。只要 Redis 故障了，加锁就不行了。  
    * 采用Master-Slave 模式，加锁的时候只对一个节点加锁，即使通过 Sentinel做了高可用，但是<font color="lime">如果Master节点故障了，发生主从切换，此时就会有可能出现锁丢失的问题，可能导致多个客户端同时完成加锁</font>。  

## 1.3. Redis Client原生API  
### 1.3.1. 单实例redis实现分布式锁  

#### 1.3.1.1. 加锁  
&emsp; **加锁代码：**  

```java
public class RedisTool {
    private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";
    /**
     * 尝试获取分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```
&emsp; 加锁就一行代码：jedis.set(String key, String value, String nxxx, String expx, int time)，这个set()方法一共有五个形参：  

* key，使用key来当锁，因为key是唯一的。  
* value，传参requestId，代表执行的具体线程。  
* **<font color = "red">nxxx，传参NX，意思是SET IF NOT EXIST，即当key不存在时，进行set操作；若key已经存在，则不做任何操作。</font>**  
* **<font color = "red">expx，传参PX，即给这个key加一个过期的设置，具体时间由第五个参数决定。</font>**  
* time，与第四个参数相呼应，代表key的过期时间。  

&emsp; 执行上面的set()方法就只会导致两种结果：1. 锁不存在，进行加锁操作，并对锁设置个有效期，同时value表示加锁的客户端； 2. 锁存在，不做任何操作。  

&emsp; 加锁中使用了redis的set命令。加锁涉及获取锁、加锁两步操作**最初分布式锁借助于setnx和expire命令**，但是这两个命令不是原子操作，如果执行setnx之后获取锁，但是此时客户端挂掉，这样无法执行expire设置过期时间就导致锁一直无法被释放，因此**在2.8版本中Antirez为setnx增加了参数扩展，使得setnx和expire具备原子操作性**。  

```
SET KEY value [EX seconds] [PX milliseconds] [NX|XX]
``` 
* EX second:设置键的过期时间为second秒。  
* PX millisecond:设置键的过期时间为millisecond毫秒。  
* NX：只在键不存在时，才对键进行设置操作。  
* XX：只在键已经存在时，才对键进行设置操作。  

#### 1.3.1.2. 解锁  
&emsp; 解锁也涉及获取锁、删除锁两步操作，采用redis和lua脚本实现。lua脚本执行命令具有原子性。  
&emsp; **解锁代码：**  

```java
public class RedisTool {
    private static final Long RELEASE_SUCCESS = 1L;
    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {
        //Lua脚本代码
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```

### 1.3.2. 集群redlock算法实现分布式锁  
&emsp; Redis分布式锁官网中文地址：http://redis.cn/topics/distlock.html 。 

&emsp; RedLock算法描述：假设Redis的部署模式是Redis Cluster，总共有5个Master节点。客户端通过以下步骤获取一把锁。  
1. 获取当前时间戳，单位是毫秒。  
2. 轮流尝试在每个Master节点上创建锁，过期时间设置较短，一般就几十毫秒。  
3. 尝试在大多数节点上建立一个锁，比如 5 个节点就要求是3个节点（n / 2 +1）。  
4. 客户端计算建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了。  
5. 要是锁建立失败了，那么就依次删除这个锁。  
6. 只要别的线程建立了一把分布式锁，当前线程就得不断轮询去尝试获取锁。  

&emsp; <font color="red">一句话概述：当前线程尝试给每个Master节点加锁。要在多数节点上加锁，并且加锁时间小于超时时间，则加锁成功；加锁失败时，依次删除节点上的锁。</font>  

## 1.4. Redisson实现redis分布式锁  
&emsp; 基于redis的分布式锁实现客户端[Redisson](/docs/microService/Redis/Redisson.md) ，官方网址：https://redisson.org/ 。Redisson支持redis单实例、redis master-slave、redis哨兵、redis cluster等各种部署架构，都可以完美实现。  

&emsp; 使用示例： 
 
```java
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

### 1.4.1. ※※※Redisson解决死锁问题(watch dog自动延期机制)  
<!-- 
https://www.cnblogs.com/jklixin/p/13212864.html

* 自动延期  
&emsp; <font color = "lime">只要客户端一旦加锁成功，就会启动一个守护线程，会每隔10秒检查一下，如果客户端1还持有锁key，那么就会不断的延长锁key的生存时间。</font>   
&emsp; <font color = "lime">在一个分布式环境下，假如一个线程获得锁后，突然服务器宕机了，那么这个时候在一定时间后这个锁会自动释放，也可以设置锁的有效时间(不设置默认30秒），这样的目的主要是业务机器宕机，防止死锁的发生。</font>   
-->

&emsp; 普通利用Redis实现分布式锁的时候，可能会为某个锁指定某个key，当线程获取锁并执行完业务逻辑代码的时候，将该锁对应的key删除掉来释放锁。
lock->set(key)，成功->执行业务，业务执行完毕->unlock->del(key)。  
&emsp; 根据这种操作和实践方式，可以分为下面两个场景：  

1. 业务机器宕机  
&emsp; 因为业务不知道要执行多久才能结束，所以这个key一般不会设置过期时间。这样如果在执行业务的过程中，业务机器宕机，unlock操作不会执行，所以这个锁不会被释放，其他机器拿不到锁，从而形成了死锁。  
&emsp; Redisson为了解决这种情况，设定了一个叫做lockWatchdogTimeout的参数，默认为30秒钟。这样当业务方调用加锁操作的时候，  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/problems/problem-45.png)  
&emsp; 默认的leaseTime是-1，这个时候会启动一个定时任务，在业务方释放锁之前，会一直不停的增加这个锁的生命周期时间，保证在业务执行完毕之前，这个锁一直不会因为redis的超时而被释放。  
2. Redis宕机  
&emsp; 如果Redis宕机，三种情况：
    1. Redis是单点模式
    2. Redis是集群模式，master在获取到一把锁之后（写操作成功后），在没来得及把该锁同步到slave之前就宕掉，这个时候slave没有锁，这把锁失效了。
    3. Redis是集群模式，而整个集群都宕机，那么就没救了。  

&emsp; **<font color = "lime">redisson锁自动释放，看门狗线程未生效：</font>**  
&emsp; lockWatchdogTimeout（监控锁的看门狗超时，单位：毫秒），默认值：30000
&emsp; 监控锁的看门狗超时时间单位为毫秒。该参数只适用于分布式锁的加锁请求中未明确使用leaseTimeout参数的情况。如果该看门口未使用lockWatchdogTimeout去重新调整一个分布式锁的lockWatchdogTimeout超时，那么这个锁将变为失效状态。这个参数可以用来避免由Redisson客户端节点宕机或其他原因造成死锁的情况。
&emsp; <font color = "lime">设置了失效时间，所以这个看门狗设置是无效的。</font>  
      
### 1.4.2. 重入锁  
&emsp; Redisson的分布式可重入锁RLock，实现了java.util.concurrent.locks.Lock接口，以及支持自动过期解锁。同时还提供了异步（Async）、反射式（Reactive）和RxJava2标准的接口。  

```java
// 最常见的使用方法
RLock lock = redisson.getLock("anyLock");
lock.lock();
//...
lock.unlock();
 
//另外Redisson还通过加锁的方法提供了leaseTime的参数来指定加锁的时间。超过这个时间后锁便自动解开了。
 
// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);
 
// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
```

&emsp; 如果负责储存这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改Config.lockWatchdogTimeout来另行指定。  

&emsp; Redisson同时还为分布式锁提供了异步执行的相关方法：  

```java
RLock lock = redisson.getLock("anyLock");
lock.lockAsync();
lock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = lock.tryLockAsync(100, 10, TimeUnit.SECONDS);
```
&emsp; RLock对象完全符合Java的Lock规范。也就是说只有拥有锁的进程才能解锁，其他进程解锁则会抛出IllegalMonitorStateException错误。  


#### 1.4.2.1. 重入锁解析
##### 1.4.2.1.1. 获取锁tryLock  
&emsp; **<font color = "lime">RedissonLock锁互斥、自动延期(watchdog看门狗)机制、可重入加锁。</font>**  


&emsp; RedissonLock加锁流程：  
1. 执行lock.lock()代码时，<font color = "red">如果该客户端面对的是一个redis cluster集群，首先会根据hash节点选择一台机器。</font>  
2. 然后发送一段lua脚本，带有三个参数：一个是锁的名字（在代码里指定的）、一个是锁的时常（默认30秒）、一个是加锁的客户端id（每个客户端对应一个id）。<font color = "red">然后脚本会判断是否有该名字的锁，如果没有就往数据结构中加入该锁的客户端id。</font>  

    * 锁不存在（exists），则加锁（hset），并设置（pexpire）锁的过期时间；  
    * 锁存在，检测（hexists）是当前线程持有锁，锁重入（hincrby），并且重新设置（pexpire）该锁的有效时间；
    * 锁存在，但不是当前线程的，返回（pttl）锁的过期时间。 

![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/problems/problem-41.png)  

```java
Future<Long> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_LONG,
            /**
             * KEYS[1]表示的是getName()，代表的是加锁的那个key，即上面代码中的test_lock
             * ARGV[1]表示的是internalLockLeaseTime，代表的就是锁key的生存时间，默认30秒
             * ARGV[2]表示的是getLockName(threadId)，代表的是id:threadId用加锁的客户端的id+线程id，表示当前访问线程，用于区分不同服务器上的线程。
             */
            "if (redis.call('exists', KEYS[1]) == 0) then " + //如果锁名称不存在
                    "redis.call('hset', KEYS[1], ARGV[2], 1); " + //则向redis中添加一个key为test_lock的set，并且向set中添加一个field为线程id，值=1的键值对，表示此线程的重入次数为1
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " + //设置set的过期时间，防止当前服务器出问题后导致死锁;
                    "return nil; " + //end;返回nil 结束。返回值中nil与false同一个意思。
                    "end; " +
                    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " + //如果锁是存在的，检测是否是当前线程持有锁，如果是当前线程持有锁
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + //则将该线程重入的次数++
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " + //并且重新设置该锁的有效时间
                    "return nil; " + // 返回nil，结束
                    "end; " +
                    "return redis.call('pttl', KEYS[1]);", //锁存在, 但不是当前线程加的锁，则返回锁的过期时间。
            Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```

&emsp; 缺点：  
&emsp; 其实上面那种方案最大的问题，就是如果对某个redis master实例，写入了myLock这种锁key的value，此时会异步复制给对应的master slave实例。  
&emsp; 但是这个过程中一旦发生redis master宕机，主备切换，redis slave变为了redis master。接着就会导致，客户端2来尝试加锁的时候，在新的redis master上完成了加锁，而客户端1也以为自己成功加了锁。此时就会导致多个客户端对一个分布式锁完成了加锁。  
&emsp; 这时系统在业务语义上一定会出现问题，导致各种脏数据的产生。  
&emsp; 所以这个就是redis cluster，或者是redis master-slave架构的主从异步复制导致的redis分布式锁的最大缺陷：<font color = "lime">在redis master实例宕机的时候，可能导致多个客户端同时完成加锁。</font>  

##### 1.4.2.1.2. 解锁unlock  

```java
public void unlock() {
    Boolean opStatus = commandExecutor.evalWrite(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            /**
             * KEYS[1] 表示的是getName() 代表锁名test_lock
             * KEYS[2] 表示getChanelName() 表示的是发布订阅过程中使用的Chanel
             * ARGV[1] 表示的是LockPubSub.unLockMessage 是解锁消息，实际代表的是数字 0，代表解锁消息
             * ARGV[2] 表示的是internalLockLeaseTime 默认的有效时间 30s
             * ARGV[3] 表示的是getLockName(thread.currentThread().getId())，是当前加锁的客户端的id+线程id
             */
            "if (redis.call('exists', KEYS[1]) == 0) then " + //如果锁已经不存在(可能是因为过期导致不存在，也可能是因为已经解锁)
                    "redis.call('publish', KEYS[2], ARGV[1]); " + //则发布锁解除的消息
                    "return 1; " + //返回1结束
                    "end;" +
                    "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " + //如果锁存在，但是若果当前线程不是加锁的线
                    "return nil;" + //则直接返回nil 结束
                    "end; " +
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " + //如果是锁是当前线程所添加，定义变量counter，表示当前线程的重入次数-1,即直接将重入次数-1
                    "if (counter > 0) then " + //如果重入次数大于0，表示该线程还有其他任务需要执行
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " + //则重新设置该锁的有效时间
                    "return 0; " + //返回0结束
                    "else " +
                    "redis.call('del', KEYS[1]); " + //否则表示该线程执行结束，删除该锁
                    "redis.call('publish', KEYS[2], ARGV[1]); " + //并且发布该锁解除的消息
                    "return 1; "+ //返回1结束
                    "end; " +
                    "return nil;",
            Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(Thread.currentThread().getId()));
    if (opStatus == null) { //脚本执行结束之后，如果返回值不是0或1，即当前线程去解锁其他线程的加锁时，抛出异常。
        throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                + id + " thread-id: " + Thread.currentThread().getId());
    }
    if (opStatus) {
        cancelExpirationRenewal();
    }
}
```

#### 1.4.2.2. 重入锁缺点  
&emsp; Redis分布式锁会有个缺陷，就是在Redis哨兵模式下:  
&emsp; 客户端1 对某个 master节点写入了redisson锁，此时会异步复制给对应的slave节点。但是这个过程中一旦发生master节点宕机，主备切换，slave节点从变为了 master节点。  
&emsp; 这时客户端2来尝试加锁的时候，在新的master节点上也能加锁，此时就会导致多个客户端对同一个分布式锁完成了加锁。  
&emsp; 这时系统在业务语义上一定会出现问题，导致各种脏数据的产生。  
&emsp; 缺陷在哨兵模式或者主从模式下，如果 master实例宕机的时候，可能导致多个客户端同时完成加锁。  


### 1.4.3. 公平锁（Fair Lock）  
&emsp; 它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。所有请求线程会在一个队列中排队，当某个线程出现宕机时，Redisson会等待5秒后继续下一个线程，也就是说如果前面有5个线程都处于等待状态，那么后面的线程会等待至少25秒。使用方式同上，获取的时候使用如下方法：  

```java
RLock fairLock = redisson.getFairLock("anyLock");
```

### 1.4.4. 联锁（MultiLock）  
&emsp; 基于Redis的Redisson分布式联锁RedissonMultiLock对象可以将多个RLock对象关联为一个联锁，每个RLock对象实例可以来自于不同的Redisson实例。  

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");
 
RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 所有的锁都上锁成功才算成功。
lock.lock();
//...
lock.unlock();
 
//另外Redisson还通过加锁的方法提供了leaseTime的参数来指定加锁的时间。超过这个时间后锁便自动解开了。
 
RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 给lock1，lock2，lock3加锁，如果没有手动解开的话，10秒钟后将会自动解开
lock.lock(10, TimeUnit.SECONDS);
 
// 为加锁等待100秒时间，并在加锁成功10秒钟后自动解开
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
//...
lock.unlock();
```

### 1.4.5. 红锁（RedLock）  
<!-- 
https://blog.csdn.net/qq_35688140/article/details/103461115
-->
&emsp; 基于Redis的Redisson红锁RedissonRedLock对象实现了Redlock介绍的加锁算法。该对象也可以用来将多个RLock对象关联为一个红锁，每个RLock对象实例可以来自于不同的Redisson实例。  

&emsp; 假设有5个redis节点，这些节点之间既没有主从，也没有集群关系。客户端用相同的key和随机值在5个节点上请求锁，请求锁的超时时间应小于锁自动释放时间。当在3个（超过半数）redis上请求到锁的时候，才算是真正获取到了锁。如果没有获取到锁，则把部分已锁的redis释放掉。  

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");
 
RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 红锁在大部分节点上加锁成功就算成功。
lock.lock();
...
lock.unlock();
 
//另外Redisson还通过加锁的方法提供了leaseTime的参数来指定加锁的时间。超过这个时间后锁便自动解开了。
 
RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 给lock1，lock2，lock3加锁，如果没有手动解开的话，10秒钟后将会自动解开
lock.lock(10, TimeUnit.SECONDS);
 
// 为加锁等待100秒时间，并在加锁成功10秒钟后自动解开
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

```java
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://172.0.0.1:5378").setPassword("a123456").setDatabase(0);
RedissonClient redissonClient1 = Redisson.create(config1);

Config config2 = new Config();
config2.useSingleServer().setAddress("redis://172.0.0.1:5379").setPassword("a123456").setDatabase(0);
RedissonClient redissonClient2 = Redisson.create(config2);

Config config3 = new Config();
config3.useSingleServer().setAddress("redis://172.0.0.1:5380").setPassword("a123456").setDatabase(0);
RedissonClient redissonClient3 = Redisson.create(config3);

/**
 * 获取多个 RLock 对象
 */
RLock lock1 = redissonClient1.getLock(lockKey);
RLock lock2 = redissonClient2.getLock(lockKey);
RLock lock3 = redissonClient3.getLock(lockKey);

/**
 * 根据多个 RLock 对象构建 RedissonRedLock （最核心的差别就在这里）
 */
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);

try {
    /**
     * 4.尝试获取锁
     * waitTimeout 尝试获取锁的最大等待时间，超过这个值，则认为获取锁失败
     * leaseTime   锁的持有时间,超过这个时间锁会自动失效（值应设置为大于业务处理的时间，确保在锁有效期内业务能处理完）
     */
    boolean res = redLock.tryLock((long)waitTimeout, (long)leaseTime, TimeUnit.SECONDS);
    if (res) {
        //成功获得锁，在这里处理业务
    }
} catch (Exception e) {
    throw new RuntimeException("aquire lock fail");
}finally{
    //无论如何, 最后都要解锁
    redLock.unlock();
}
```

&emsp; 加锁核心源码：  

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long newLeaseTime = -1;
    if (leaseTime != -1) {
        newLeaseTime = unit.toMillis(waitTime)*2;
    }
    
    long time = System.currentTimeMillis();
    long remainTime = -1;
    if (waitTime != -1) {
        remainTime = unit.toMillis(waitTime);
    }
    long lockWaitTime = calcLockWaitTime(remainTime);
    /**
     * 1. 允许加锁失败节点个数限制（N-(N/2+1)）
     */
    int failedLocksLimit = failedLocksLimit();
    /**
     * 2. 遍历所有节点通过EVAL命令执行lua加锁
     */
    List<RLock> acquiredLocks = new ArrayList<>(locks.size());
    for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
        RLock lock = iterator.next();
        boolean lockAcquired;
        /**
         *  3.对节点尝试加锁
         */
        try {
            if (waitTime == -1 && leaseTime == -1) {
                lockAcquired = lock.tryLock();
            } else {
                long awaitTime = Math.min(lockWaitTime, remainTime);
                lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
            }
        } catch (RedisResponseTimeoutException e) {
            // 如果抛出这类异常，为了防止加锁成功，但是响应失败，需要解锁所有节点
            unlockInner(Arrays.asList(lock));
            lockAcquired = false;
        } catch (Exception e) {
            // 抛出异常表示获取锁失败
            lockAcquired = false;
        }
        
        if (lockAcquired) {
            /**
             *4. 如果获取到锁则添加到已获取锁集合中
             */
            acquiredLocks.add(lock);
        } else {
            /**
             * 5. 计算已经申请锁失败的节点是否已经到达 允许加锁失败节点个数限制 （N-(N/2+1)）
             * 如果已经到达， 就认定最终申请锁失败，则没有必要继续从后面的节点申请了
             * 因为 Redlock 算法要求至少N/2+1 个节点都加锁成功，才算最终的锁申请成功
             */
            if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                break;
            }

            if (failedLocksLimit == 0) {
                unlockInner(acquiredLocks);
                if (waitTime == -1 && leaseTime == -1) {
                    return false;
                }
                failedLocksLimit = failedLocksLimit();
                acquiredLocks.clear();
                // reset iterator
                while (iterator.hasPrevious()) {
                    iterator.previous();
                }
            } else {
                failedLocksLimit--;
            }
        }

        /**
         * 6.计算 目前从各个节点获取锁已经消耗的总时间，如果已经等于最大等待时间，则认定最终申请锁失败，返回false
         */
        if (remainTime != -1) {
            remainTime -= System.currentTimeMillis() - time;
            time = System.currentTimeMillis();
            if (remainTime <= 0) {
                unlockInner(acquiredLocks);
                return false;
            }
        }
    }

    if (leaseTime != -1) {
        List<RFuture<Boolean>> futures = new ArrayList<>(acquiredLocks.size());
        for (RLock rLock : acquiredLocks) {
            RFuture<Boolean> future = ((RedissonLock) rLock).expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
            futures.add(future);
        }
        
        for (RFuture<Boolean> rFuture : futures) {
            rFuture.syncUninterruptibly();
        }
    }

    /**
     * 7.如果逻辑正常执行完则认为最终申请锁成功，返回true
     */
    return true;
}
```


### 1.4.6. 读写锁（ReadWriteLock）  
&emsp; 基于Redis的Redisson分布式可重入读写锁RReadWriteLock Java对象实现了java.util.concurrent.locks.ReadWriteLock接口。其中读锁和写锁都继承了RLock接口。分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。  

```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();
 
//另外Redisson还通过加锁的方法提供了leaseTime的参数来指定加锁的时间。超过这个时间后锁便自动解开了。
 
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);
 
// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
//...
lock.unlock();
```

### 1.4.7. 信号量（Semaphore）  
&emsp; 基于Redis的Redisson的分布式信号量（Semaphore）Java对象RSemaphore采用了与java.util.concurrent.Semaphore相似的接口和用法。同时还提供了异步（Async）、反射式（Reactive）和RxJava2标准的接口。  

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.acquire();
//或
semaphore.acquireAsync();
semaphore.acquire(23);
semaphore.tryAcquire();
//或
semaphore.tryAcquireAsync();
semaphore.tryAcquire(23, TimeUnit.SECONDS);
//或
semaphore.tryAcquireAsync(23, TimeUnit.SECONDS);
semaphore.release(10);
semaphore.release();
//或
semaphore.releaseAsync();
```
