

<!-- TOC -->

- [1. Redis持久化](#1-redis持久化)
    - [1.1. RDB（Redis DataBase），快照](#11-rdbredis-database快照)
        - [1.1.1. RDB的触发](#111-rdb的触发)
        - [1.1.2. RDB的流程](#112-rdb的流程)
        - [1.1.3. RDB的优势和劣势](#113-rdb的优势和劣势)
    - [1.2. AOF（Append-only file）](#12-aofappend-only-file)
        - [1.2.1. 开启AOF](#121-开启aof)
        - [1.2.2. AOF持久化流程](#122-aof持久化流程)
            - [1.2.2.1. 重写机制](#1221-重写机制)
            - [1.2.2.2. 重启加载步骤（数据恢复流程）](#1222-重启加载步骤数据恢复流程)
        - [1.2.3. AOF文件损坏](#123-aof文件损坏)
        - [1.2.4. AOF的优势和劣势](#124-aof的优势和劣势)
    - [1.3. 混合持久化](#13-混合持久化)
    - [1.4. Redis集群持久化策略](#14-redis集群持久化策略)

<!-- /TOC -->

&emsp; **<font color = "red">部分参考《Redis开发与运维》</font>**  
<!-- 
redis系列--redis4.0深入持久化
https://www.cnblogs.com/wdliu/p/9377278.html
AOF和RDB混合使用
https://mp.weixin.qq.com/s/-mCgBp-pjJzKqhYut3yYgw
-->

# 1. Redis持久化  
**<font color = "lime">&emsp; 一句话概述：  
&emsp; RDB，快照；保存某一时刻的全部数据；缺点是间隔长（配置文件中默认最少60s）。  
&emsp; AOF，文件追加；记录所有操作命令；优点是默认间隔1s，丢失数据少；缺点是文件比较大，通过重写机制来压缩文件体积。  
&emsp; Redis4.0混合持久化，先RDB，后AOF。  
</font>**   

&emsp; Redis是一种内存数据库。一旦进程退出，Redis的数据就会丢失。Redis持久化拥有以下三种方式：  
1. 快照方式（RDB, Redis DataBase）将某一个时刻的内存数据，以二进制的方式写入磁盘，RDB方式是redis默认的持久化方式；  
2. 文件追加方式（AOF, Append Only File），记录所有的操作命令，并以文本的形式追加到文件中；  
3. 混合持久化方式，Redis 4.0之后新增的方式，<font color = "red">混合持久化是结合了RDB和 AOF的优点，在写入的时候，先把当前的数据以RDB的形式写入文件的开头，再将后续的操作命令以AOF的格式存入文件</font>，这样<font color = "lime">既能保证Redis重启时的速度，又能减少数据丢失的风险。</font>  

## 1.1. RDB（Redis DataBase），快照
&emsp; <font color = "red">RDB持久化是Redis默认的持久化方式。RDB是一种快照存储持久化方式，</font><font color = "lime">将Redis某一时刻的所有内存数据保存到硬盘的文件当中</font>，默认保存的文件名为dump.rdb，dump.rdb文件默认生成在%REDIS_HOME%etc目录下（如/usr/local/redis/etc/），可以修改redis.conf文件中的dir指定dump.rdb的保存路径。也可以将快照复制到其他服务器从而创建具有相同数据的服务器副本。  
&emsp; 在Redis服务器启动时，会重新加载dump.rdb文件的数据到内存当中恢复数据，即通过该文件可以还原生成RDB文件时的数据库状态（数据库状态是指 Redis服务器的非空数据库以及键值对的统称）。  

### 1.1.1. RDB的触发  
&emsp; RDB触发机制分为使用指令手动触发和自动触发。  

* 自动触发RDB持久化：
    1. 方式一：修改redis.conf文件，默认配置如下所示：  

            save 900 1 # 表示900秒内如果至少有 1 个 key 的值变化，则触发RDB
            save 300 10 # 表示300秒内如果至少有 10 个 key 的值变化，则触发RDB
            save 60 10000 # 表示60秒内如果至少有 10000 个 key 的值变化，则触发RDB  
    &emsp; 如果不需要Redis进行持久化，可以注释掉所有的save行来停用保存功能，也可以直接一个空字符串来停用持久化：save ""。  
    &emsp; Redis服务器周期操作函数serverCron默认每个100毫秒就会执行一次，该函数用于正在运行的服务器进行维护，它的一项工作就是检查save选项所设置的条件是否有一项被满足，如果满足的话，就执行bgsave指令。   
    
    2. 方式二：shutdown 触发，保证服务器正常关闭。 

<!--
<font color = "red">&emsp; 自动间隔性保存</font>   
&emsp; 对于RDB持久化而言，一般都会使用BGSAVE来持久化，因为不会阻塞服务器进程。  
&emsp; 在Redis的配置文件，有提供设置服务器每隔多久时间来执行BGSAVE命令。  
&emsp; Redis默认是如下配置：  

    save 900 1      // 900 秒内，对数据库至少修改 1 次。下面同理    
    save 300 10     
    save 60 10000

&emsp; 只要满足其中一种情况，服务器就会执行BGSAVE命令。  
-->
* 手动触发RDB持久化：客户端通过向Redis服务器发送Save或Bgsave命令让服务器生成RDB文件。  
    1. save命令：是一个同步操作。  
    ![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-31.png)  
    &emsp; 当客户端向服务器发送Save命令请求进行持久化时，服务器会阻塞Save命令之后的其他客户端的请求，直到数据同步完成。  
    &emsp; 如果数据量太大，同步数据会执行很久，而这期间Redis服务器也无法接收其他请求，所以，最好不要在生产环境使用Save命令。  
    2. Bgsave命令：  
    &emsp; 与Save命令不同，Bgsave命令是一个异步操作。  
    ![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-32.png)  
    &emsp; bgsave，执行该命令时，Redis会在后台异步执行快照操作，此时Redis仍然可以相应客户端请求。具体操作是当客户端发服务发出Bgsave命令时，Redis服务器主进程会Forks操作创建一个子进程来数据同步问题，在将数据保存到RDB文件之后，子进程会退出。新RDB文件就会原子地替换旧的RDB文件。所以，与Save命令相比，Redis服务器在处理Bgsave采用子线程进行IO写入。而主进程仍然可以接收其他请求，但Forks子进程是同步的，所以Forks子进程时，一样不能接收其他请求。这意味着，如果Forks一个子进程花费的时间太久（一般是很快的），而且占用内存会加倍，Bgsave命令仍然有阻塞其他客户的请求的情况发生。  

### 1.1.2. RDB的流程  
&emsp; bgsave是主流的触发RDB持久化方式。它的运行流程如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-53.png)  
1. 执行bgsave命令，Redis父进程判断当前是否存在正在执行的子进程，如RDB/AOF子进程，如果存在bgsave命令直接返回。  
2. 父进程执行fork操作创建子进程，fork操作过程中父进程会阻塞，通过info stats命令查看latest_fork_usec选项，可以获取最近一个fork操作的耗 时，单位为微秒。  
3. 父进程fork完成后，bgsave命令返回“Background saving started”信息 并不再阻塞父进程，可以继续响应其他命令。  
4. 子进程创建RDB文件，根据父进程内存生成临时快照文件，完成后 对原有文件进行原子替换。执行lastsave命令可以获取最后一次生成RDB的 时间，对应info统计的rdb_last_save_time选项。  
5. 进程发送信号给父进程表示完成，父进程更新统计信息，具体见 info Persistence下的rdb_*相关选项。  

### 1.1.3. RDB的优势和劣势  
* 优势 
    1. RDB 是一个非常紧凑(compact)的文件，它保存了 redis 在某个时间点上的数据集。这种文件非常适合用于进行备份和灾难恢复。  
    2. 生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作。  
    3. RDB 在恢复大数据集时的速度比AOF的恢复速度要快。 

* 劣势  
    1. RDB方式数据没办法做到实时持久化/秒级持久化。因为 bgsave 每次运行都要 执行 fork 操作创建子进程，频繁执行成本过高。 
    2. <font color = "red">在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照之后的所有修改（数据有丢失）。</font>如果数据相对来说比较重要，希望将损失降到最小，则可以使用AOF方式进行持久化。  

## 1.2. AOF（Append-only file）  
&emsp; AOF持久化机制是<font color = "red">以日志的形式记录Redis中的每一次的写操作</font>，不会记录查询操作，以文本的形式记录，打开记录的日志文件就可以查看操作记录。  
&emsp; AOF是默认不开启的。  

&emsp; AOF总体流程和RDB持久化一样。都是创建一个xxx文件、在服务器下次启动时就载入这个文件来还原数据。  
<!-- &emsp; AOF持久化方式会记录客户端对服务器的每一次写操作命令，并将这些写操作以Redis协议追加保存到后缀为AOF文件末尾。-->  

### 1.2.1. 开启AOF
&emsp; Redis默认不开启AOF持久化方式，可以在配置文件中开启并进行更加详细的配置，如下面的redis.conf文件：  

    # 开启aof机制
    appendonly yes
    # aof文件名
    appendfilename "appendonly.aof"
    # 写入策略,always表示每个写操作都保存到aof文件中,也可以是everysec或no
    appendfsync always
    # 默认不重写aof文件
    no-appendfsync-on-rewrite no
    # 保存目录
    dir ~/redis/

&emsp; 在上面的配置文件中，<font color = "red">通过 appendfsync选项指定写入策略</font>，有三个选项：  

    appendfsync always
    # appendfsync everysec
    # appendfsync no

* always：客户端的每一个写操作都保存到 AOF 文件当中，这种策略很安全，但是每个写操作都有 IO 操作，所以也很慢。  
* everysec：<font color = "red">appendfsync 的默认写入策略，每秒写入一次AOF文件，因此，最多可能会丢失1s的数据。</font>  
* no：Redis 服务器不负责写入 AOF，而是交由操作系统来处理什么时候写入 AOF文件。更快，但也是最不安全的选择，不推荐使用。 

### 1.2.2. AOF持久化流程  
&emsp; **<font color = "red">AOF的工作流程操作：命令写入 （append）、文件同步（sync）、文件重写（rewrite）、重启加载 （load）。</font>**  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-52.png)  
&emsp; 流程如下：  
1. 所有的写入命令会追加到aof_buf（缓冲区）中。   
2. AOF缓冲区根据对应的策略向硬盘做同步操作。   
3. <font color = "red">随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的。</font>  
4. 当Redis服务器重启时，可以加载AOF文件进行数据恢复。  

<!-- 
&emsp; AOF持久化功能的实现可以分为3个步骤：命令追加、文件写入、文件同步。  

* 命令追加：将写命令追加到AOF缓冲区的末尾。  
* 文件写入：缓冲区内容写到AOF文件。  
* 文件同步：将AOF文件保存到磁盘。  

&emsp; 文件写入、文件同步需要根据一定的条件来执行，而这些条件由Redis配置文件中的appendfsync选项来决定。  
-->
#### 1.2.2.1. 重写机制
&emsp; 随着命令不断写入AOF，文件会越来越大，为了解决这个问题，Redis 引入AOF重写机制压缩文件体积。AOF文件重写是把Redis进程内的数据转化为写命令同步到新AOF文件的过程。   
&emsp; **<font color = "lime">重写后的AOF文件为什么可以变小？有如下原因：</font>**  

1. <font color = "red">进程内已经超时的数据不再写入文件。</font>   
2. <font color = "red">旧的AOF文件含有无效命令，</font>如del key1、hdel key2、srem keys、set a111、set a222等。重写使用进程内数据直接生成，这样新的AOF文件只保留最终数据的写入命令。  
3. <font color = "red">多条写命令可以合并为一个，</font>如：lpush list a、lpush list b、lpush list c可以转化为：lpush list a b c。为了防止单条命令过大造成客户端缓冲区溢出，对于list、set、hash、zset等类型操作，以64个元素为界拆分为多条。  

&emsp; **<font color = "red">AOF重写降低了文件占用空间，除此之外，另一个目的是：更小的AOF 文件可以更快地被Redis加载。</font>**  

&emsp; AOF重写过程可以手动触发和自动触发：  
* 手动触发：直接调用bgrewriteaof命令。 
* 自动触发：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数确定自动触发时机。
    * auto-aof-rewrite-min-size：表示运行AOF重写时文件最小体积，默认 为64MB。
    * auto-aof-rewrite-percentage：代表当前AOF文件空间 （aof_current_size）和上一次重写后AOF文件空间（aof_base_size）的比值。  
    
    &emsp; 自动触发时机=aof_current_size>auto-aof-rewrite-min- size&&（aof_current_size-aof_base_size）/aof_base_size>=auto-aof-rewrite- percentage  
    &emsp; 其中aof_current_size和aof_base_size可以在info Persistence统计信息中查看。  

&emsp; 当触发AOF重写时，内部做了哪些事呢？下面结合图5-3介绍它的运行流程。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-86.png)  


#### 1.2.2.2. 重启加载步骤（数据恢复流程）  
&emsp; AOF和RDB文件都可以用于服务器重启时的数据恢复。如下图所示， 表示Redis持久化文件加载流程。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-54.png)  
&emsp; 流程说明： 
1. AOF持久化开启且存在AOF文件时，优先加载AOF文件。  
2. AOF关闭或者AOF文件不存在时，加载RDB文件。  
3. 加载AOF/RDB文件成功后，Redis启动成功。 
4. AOF/RDB文件存在错误时，Redis启动失败并打印错误信息。  

### 1.2.3. AOF文件损坏  
&emsp; 在写入AOF日志文件时，如果Redis服务器宕机，则AOF日志文件文件会出格式错误。在重启Redis服务器时，Redis服务器会拒绝载入这个AOF文件，可以通过以下步骤修复AOF 并恢复数据：  

* 备份现在 AOF 文件，以防万一。
* <font color = "red">使用redis-check-aof命令修复AOF文件</font>，该命令格式如下：  

        # 修复aof日志文件
        $ redis-check-aof -fix file.aof
* 重启 Redis 服务器，加载已经修复的AOF文件，恢复数据。  

### 1.2.4. AOF的优势和劣势  
* 优势  
    1. AOF 持久化的方法提供了多种的同步频率，即使使用默认的同步频率每秒同步 一次，Redis 最多也就丢失 1 秒的数据而已。  

* 缺点： 
    1. 对于具有相同数据的的 Redis，<font color = "red">AOF 文件通常会比 RDF 文件体积更大</font>（RDB 存的是数据快照）。  
    2. 虽然 AOF 提供了多种同步的频率，默认情况下，每秒同步一次的频率也具有较高的性能。在高并发的情况下，RDB 比 AOF 具好更好的性能保证。  

<!-- 
 1.3. RDB方式与AOF方式的优势对比  
 1.3.1. RDB优缺点：  
&emsp; RDB的几个优点：
* 与AOF方式相比，通过RDB文件恢复数据比较快。  
* RDB文件非常紧凑，适合于数据备份。  
* 通过RDB进行数据备份，由于使用子进程生成，所以对Redis服务器性能影响较小。  

&emsp; RDB的几个缺点：
* 如果服务器宕机的话，采用RDB的方式会造成某个时段内数据的丢失，比如设置10分钟同步一次或5分钟达到1000次写入就同步一次，那么如果还没达到触发条件服务器就死机了，那么这个时间段的数据会丢失。  
* 使用Save命令会造成服务器阻塞，直接数据同步完成才能接收后续请求。  
* 使用Bgsave命令在Forks子进程时，如果数据量太大，Forks 的过程也会发生阻塞，另外，Forks 子进程会耗费内存。  

 1.3.2. AOF优缺点：  
&emsp; AOF的优点：  
* AOF只是追加日志文件，因此对服务器性能影响较小，速度比RDB要快，消耗的内存较少。   

&emsp; AOF的缺点：
* AOF方式生成的日志文件太大，即使通过AOF重写，文件体积仍然很大。  
* 恢复数据的速度比RDB慢。  
-->
## 1.3. 混合持久化  
&emsp; 在redis4.0后混合持久化（RDB+AOF）对重写的优化，4.0版本的混合持久化默认是关闭的，可以通过以下的配置开启混合持久化：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-34.png)  
&emsp; 混合持久化也是通过bgrewriteaof来完成的，不同的是当开启混合持久化时，fork出的子进程先将共享内存的数据以RDB方式写入aof文件中，然后再将重写缓冲区的增量命令以AOF方式写入文件中。  
&emsp; 写入完成后通知主进程统计信息，并将新的含有RDB格式和AOF格式的AOF文件替换旧的AOF文件。简单的说：新的AOF文件前半段是以RDB格式的全量数据后半段是AOF格式的增量数据。  

&emsp; 优点：混合持久化结合RDB持久化和AOF持久化的优点，由于绝大部分的格式是RDB格式，加载速度快，增量数据以AOF方式保存，数据更少的丢失。  

## 1.4. Redis集群持久化策略  
&emsp; 官方建议：如果要想提供很高的数据保障性，那么建议同时使用两种持久化方式。如果可以接受灾难带来的几分钟的数据丢失，那么可以仅使用RDB。  

&emsp; 一般在生产上采用的持久化策略为：1).master关闭持久化；2).slave开RDB即可，必要的时候AOF和RDB都开启。  
&emsp; 其实，如果想要数据足够安全，可以两种方式都开启，但两种持久化方式同时进行IO操作，会严重影响服务器性能，因此有时候不得不做出选择。  

&emsp; 该策略能够适应绝大部分场景，绝大部分集群架构。  
1. 为什么是绝大部分场景？  
&emsp; 因为这套策略存在部分的数据丢失可能性。redis的主从复制是异步的，master执行完客户端请求的命令后会立即返回结果给客户端，然后异步的方式把命令同步给slave。因此master可能还未来得及将命令传输给slave，就宕机了，此时slave变为master，数据就丢了。  
&emsp; 幸运的是，绝大部分业务场景，都能容忍数据的部分丢失。假设，真的遇到缓存雪崩的情况，代码中也有熔断器来进行资源保护，不至于所有的请求都转发到数据库上，导致服务崩溃！  
&emsp; 注：这里的缓存雪崩是指同一时间来了一堆请求，请求的key在redis中不存在，导致请求全部转发到数据库上。  
2. 为什么是绝大部分集群架构？  
&emsp; 因为在集群中存在redis读写分离的情况，就不适合这套方案了。  
&emsp; 幸运的是，由于采用redis读写分离架构，就必须要考虑主从同步的延迟性问题，徒增系统复杂度。目前业内采用redis读写分离架构的项目，比较少。  




