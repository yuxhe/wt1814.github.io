
<!-- TOC -->

- [1. 主从复制模式](#1-主从复制模式)
    - [1.1. 主从复制的作用](#11-主从复制的作用)
    - [1.2. 主从复制启用](#12-主从复制启用)
    - [1.3. 复制流程](#13-复制流程)
        - [1.3.1. 复制流程概述](#131-复制流程概述)
        - [1.3.2. 数据同步阶段两种复制方式](#132-数据同步阶段两种复制方式)
            - [1.3.2.1. 全量复制](#1321-全量复制)
            - [1.3.2.2. 部分复制](#1322-部分复制)
                - [1.3.2.2.1. 基本概念](#13221-基本概念)
                    - [1.3.2.2.1.1. 复制偏移量](#132211-复制偏移量)
                    - [1.3.2.2.1.2. 复制积压缓冲区](#132212-复制积压缓冲区)
                    - [1.3.2.2.1.3. 运行 ID（runid）](#132213-运行-idrunid)
                - [1.3.2.2.2. 部分复制流程](#13222-部分复制流程)
            - [1.3.2.3. psync 命令的执行过程详解](#1323-psync-命令的执行过程详解)
    - [1.4. 主从复制应用与问题](#14-主从复制应用与问题)
        - [1.4.1. 读写分离](#141-读写分离)
            - [1.4.1.1. 数据延迟](#1411-数据延迟)
            - [1.4.1.2. 读到过期数据](#1412-读到过期数据)
            - [1.4.1.3. 从节点故障](#1413-从节点故障)
        - [1.4.2. 规避全量复制](#142-规避全量复制)
            - [1.4.2.1. 第一次复制](#1421-第一次复制)
            - [1.4.2.2. 节点运行 ID 不匹配](#1422-节点运行-id-不匹配)
            - [1.4.2.3. 复制偏移量 offset 不在复制积压缓冲区中](#1423-复制偏移量-offset-不在复制积压缓冲区中)
        - [1.4.3. 规避复制风暴](#143-规避复制风暴)
            - [1.4.3.1. 单一主节点](#1431-单一主节点)
            - [1.4.3.2. 单一机器](#1432-单一机器)

<!-- /TOC -->

![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-76.png)  

# 1. 主从复制模式  
<!-- 
Redis的主从复制是如何做的？复制过程中也会产生各种问题 
 https://mp.weixin.qq.com/s/ee7Xdj4d1O-D-Yaa9G6U3w
Redis复制原理你应该理解
https://mp.weixin.qq.com/s/XUmmwykpiO8r5-FDt4XyYw
-->

## 1.1. 主从复制的作用  

* 数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。  
* 故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。  
* 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。  
* 读写分离：可以用于实现读写分离，主库写、从库读，读写分离不仅可以提高服务器的负载能力，同时可根据需求的变化，改变从库的数量；  
* 高可用基石：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。  

## 1.2. 主从复制启用  
&emsp; 临时配置：redis-cli进入redis从节点后，使用 --slaveof [masterIP] [masterPort]  
&emsp; 永久配置：进入从节点的配置文件redis.conf，增加slaveof [masterIP] [masterPort]  

## 1.3. 复制流程  
### 1.3.1. 复制流程概述  
&emsp; <font color = "red">主从复制过程大体可以分为3个阶段：连接建立阶段（即准备阶段）、数据同步阶段、命令传播阶段。</font>  
&emsp; 在从节点执行slaveof命令后，复制过程便开始运作，从下图中可以看出复制过程大致分为6个过程。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-20.png)  

    连接建立阶段  

&emsp; <font color = "red">1. 保存主节点（master）信息。</font>  
&emsp; 执行 slaveof 后 Redis 会打印如下日志：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-21.png)  

&emsp; **<font color = "lime">2. 从节点（slave）内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接。</font>**  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-22.png)  
&emsp; 从节点与主节点建立网络连接：从节点会建立一个 socket 套接字，从节点建立了一个端口为51234的套接字，专门用于接受主节点发送的复制命令。从节点连接成功后打印如下日志。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-23.png)  

&emsp; 如果从节点无法建立连接，定时任务会无限重试直到连接成功或者执行slaveof no one取消复制。  

&emsp; 关于连接失败，可以在从节点执行 info replication 查看 master_link_down_since_seconds 指标，它会记录与主节点连接失败的系统时间。从节点连接主节点失败时也会每秒打印如下日志，方便发现问题：  

```
# Error condition on socket for SYNC: {socket_error_reason}
```

&emsp; <font color = "red">3. 发送 ping 命令。</font>  
&emsp; 连接建立成功后从节点发送 ping 请求进行首次通信，ping 请求主要目的如下：  

* 检测主从之间网络套接字是否可用。  
* 检测主节点当前是否可接受处理命令。  

&emsp; 如果发送 ping 命令后，从节点没有收到主节点的 pong 回复或者超时，比如网络超时或者主节点正在阻塞无法响应命令，从节点会断开复制连接，下次定时任务会发起重连。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-24.png)  
&emsp; 从节点发送的 ping 命令成功返回，Redis 打印如下日志，并继续后续复制流程：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-25.png)  
&emsp; <font color = "red">4. 权限验证。</font>  
&emsp; 如果主节点设置了 requirepass 参数，则需要密码验证，从节点必须配置 masterauth 参数保证与主节点相同的密码才能通过验证；如果验证失败复制将终止，从节点重新发起复制流程。  

    数据同步阶段  

&emsp; <font color = "red">5. 同步数据集。</font>主从复制连接正常通信后，对于首次建立复制的场景，主节点会把持有的数据全部发送给从节点，这部分操作是耗时最长的步骤。  

    命令传播阶段  
    
&emsp; <font color = "red">6. 命令持续复制。</font>当主节点把当前的数据同步给从节点后，便完成了复制的建立流程。接下来主节点会持续地把写命令发送给从节点，保证主从数据一致性。  

### 1.3.2. 数据同步阶段两种复制方式
&emsp; 主从节点在数据同步阶段，主节点会根据当前状态的不同执行不同复制操作，包括：全量复制和部分复制。  
&emsp; redis 2.8之前使用sync [runId] [offset]同步命令，redis2.8之后使用psync [runId] [offset]命令。两者不同在于，sync命令仅支持全量复制过程，psync支持全量和部分复制。  

* <font color = "red">全量复制：用于首次复制或者其他不能进行部分复制的情况。</font>全量复制是一个非常重的操作，一般都要规避它。  
* <font color = "red">部分复制：用于从节点短暂中断的情况（网络中断、短暂的服务宕机）。</font>部分复制是一个非常轻量级的操作，因为它只需要将中断期间的命令同步给从节点即可，相比于全量复制，它显得更加高效。  

#### 1.3.2.1. 全量复制  
&emsp; 全量复制的流程图如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-40.png)  

&emsp; 1. 由于是第一次进行数据同步，从节点并不知道主节点的 runid，所以发送 psync ? -1  
&emsp; 2. 主节点接收从节点的命令后，判定是进行全量复制，所以回复 +FULLRESYNC ，同时也会将自身的 runid 和 偏移量发送给从节点，响应为 +FULLRESYNC{runid}{offset}  
&emsp; 3. 从节点接受主节点的响应后，会保存主节点的 runid 和 偏移量 offset。打印日志如下：  

    62760:S 16 May 2019 21:24:36.818 * Trying a partial resynchronization (request cf2836e6d8f3628c81b3ebb36fea4410f21f05b0:1).
    62760:S 16 May 2019 21:24:36.820 * Full resync from master: a7113788690a86b166cf978b874b3c6056167b54:0

&emsp; 从节点尝试部分复制，请求节点的runid 为 cf2836e6d8f3628c81b3ebb36fea4410f21f05b0，offset：1，但是主节点告知从节点是全量复制，runid：a7113788690a86b166cf978b874b3c6056167b54，offset：1。  
&emsp; 4. 主节点响应从节点命令后，会执行 bgsave，将生成的 RDB 文件保存在本地。打印日志如下：  

    62743:M 16 May 2019 21:24:36.819 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'cf2836e6d8f3628c81b3ebb36fea4410f21f05b0', my replication IDs are '0156844c881cf978fa35de7deeb9f85ef7cd1b0e' and '0000000000000000000000000000000000000000')
    62743:M 16 May 2019 21:24:36.819 * Starting BGSAVE for SYNC with target: disk
    62743:M 16 May 2019 21:24:36.820 * Background saving started by pid 62770
    62770:C 16 May 2019 21:24:36.821 * DB saved on disk
    62743:M 16 May 2019 21:24:36.884 * Background saving terminated with success

&emsp; 主节点接受从节点的部分请求，但是 runid 不一致，进行全量复制，返回 +FULLRESYNC，并将自身的 runid 和 offset 返回给从节点。  
&emsp; 5. 主节点将生成的 RDB 文件发送给从节点，从节点接收后保存在本地直接将其作为数据文件，如果从节点本地有 RDB 文件，则从节点会先清空 RDB 文件。从节点打印日志如下：

    62760:S 16 May 2019 21:24:36.885 * MASTER <-> REPLICA sync: receiving 234 bytes from master
    62760:S 16 May 2019 21:24:36.886 * MASTER <-> REPLICA sync: Flushing old data
    62760:S 16 May 2019 21:24:36.886 * MASTER <-> REPLICA sync: Loading DB in memory
    62760:S 16 May 2019 21:24:36.886 * MASTER <-> REPLICA sync: Finished with success

&emsp; 6. 如果从节点还开启了 AOF，则还会进行 AOF 重写，日志如下：  

    62760:S 16 May 2019 21:24:36.887 * Background append only file rewriting started by pid 62771
    62760:S 16 May 2019 21:24:36.911 * AOF rewrite child asks to stop sending diffs.
    62771:C 16 May 2019 21:24:36.912 * Parent agreed to stop sending diffs. Finalizing AOF...
    62771:C 16 May 2019 21:24:36.913 * Concatenating 0.00 MB of AOF diff received from parent.
    62771:C 16 May 2019 21:24:36.913 * SYNC append only file rewrite performed
    62760:S 16 May 2019 21:24:36.918 * Background AOF rewrite terminated with success
    62760:S 16 May 2019 21:24:36.919 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
    62760:S 16 May 2019 21:24:36.919 * Background AOF rewrite finished successfully

&emsp; 从上面过程可以看出，全量复制是一个非常重的操作过程，它的开销主要有：  

* 主节点执行 bgsave 过程，在持久化那篇博客中我们知道该过程是非常消耗 CPU、内存(页表复制)、硬盘 IO 的  
* 主节点发送 RDB 给从节点的网络开销  
* 从节点清空 RBD 文件（如果有）和加载 RDB 文件，同时该过程是阻塞的，无法响应客户端命令  
* 如果从节点开启了 AOF，则还有 bgrewriteaof 的开销  

&emsp; 所以，需要尽可能避免全量复制，当然第一次建立连接数据同步是必不可免的，但是其他的情况是可以避免的。  


&emsp; <font color="red">第一次建立连接进行数据同步是全量复制，还有以下几种情况也是全量复制：</font>  

* 从节点发送 psync {runid} {offset} 时，runid 与当前主节点的 runid 不匹配则进行全量复制
* 从节点所需要同步数据的偏移量 offset 不在复制积压缓冲区中，也会进行全量复制。 

#### 1.3.2.2. 部分复制
<!--
* 部分重同步是用于处理断线后重复制情况：  
&emsp; 当从服务器在断线后重新连接主服务器时，主服务可以将主从服务器连接断开期间执行的写命令发送给从服务器，从服务器只要接收并执行这些写命令，就可以将数据库更新至主服务器当前所处的状态。  
 -->
&emsp; 在 Redis 2.8 开始提供了部分复制，用于处理网络中断的数据同步。  

##### 1.3.2.2.1. 基本概念  
&emsp; 部分复制中设计的几个基本概念：复制偏移量、复制积压缓冲区、运行 ID（runid）。  

###### 1.3.2.2.1.1. 复制偏移量  
&emsp; 主从节点都维护这一个复制偏移量（offset），它代表着当前节点接受数据的字节数，主节点表示接收客户端的字节数，从节点表示接收主节点的字节数，比如从节点接收主节点传来的 N 个字节数据时，从节点的 offset 会增加 N。  
&emsp; 偏移量的作用非常大，它是用来衡量主从节点数据是否一直的唯一标准，如果主从节点的 offset 相等，表明数据一直，否则表明数据不一致。在不一致的情况下，可以根据两个节点的 offset 找出从节点的缺少的那部分数据。比如，主节点的 offset 是 500，从节点的 offset 是 400，那么主节点在进行数据传输时只需要将 401 ~ 500 传递给从节点即可，这就是部分复制。  
&emsp; 从节点通过心跳每秒都会将自身的偏移量告知主节点，所以主节点会保存从节点的偏移量。同时，主节点处理完命令后，会将命令的字节长度累加到自身的偏移量中，如下图：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-41.png)  
&emsp; 从节点每次接受到主节点发送的命令后，也会累加到自身的偏移量中，主节点，如下图  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-42.png)  

###### 1.3.2.2.1.2. 复制积压缓冲区
&emsp; 复制积压缓冲区是一个由主节点维护的缓存队列，它具有如下几个特点：  

* 由主节点维护  
* 固定大小，默认为 1MB，配置参数为：repl-backlog-size  
* 是一个先进先出的队列  

&emsp; 在命令传播节点，主节点除了将写命令传递给从节点，也会将写命令写入到复制积压缓冲区中，当做一个备份，用于在部分复制流程中。由于它是先进先出的队列，且大小固定，所以他只能保存主节点最近执行的写命令，当主从节点的 offset 相差较大时，超出了复制积压缓冲区的范围，则无法进行部分复制，只能进行全量复制了，所以为了能够提高网络中断引起的全量复制，需要认真评估复制积压缓冲区的大小，将其适当调大，比如网络中断时间是 60s，主节点每秒接收的写命令为 100KB，则复制积压缓冲区的平均大小应该为 6MB，所以可以将其大小设置为 6MB，甚至是 10MB，来保证绝大多数中断情况下都可以使用部分复制。  

###### 1.3.2.2.1.3. 运行 ID（runid）  
&emsp; 每个 Redis 节点在启动时都会生成一个运行 ID，即 runid，该 ID 用于唯一标识 Redis 节点，它是一个由 40 位随机的十六进制的字符组成的字符串，通过 info server 命令可以查看节点的 runid，如下：  

    127.0.0.1:6379> info server
    ...
    run_id:e88221d68ce96fc28c2f2b3afbf3495ea6de512a
    ...

&emsp; 主从节点在初次建立连接进行全量复制时（从节点发送 psync?-1），主节点会将自己的 runid 告知给从节点，从节点将其保存起来。当主从节点断开重连时，从节点会将这个 runid 发送给主节点，主节点会根据从节点发送的 runid 来判断选择何种复制：  

* 如果从节点发送的 runid 与当前主节点的 runid 一致时，主节点则尝试进行部分复制，当然能不能进行部分复制还要看偏移量是否在复制积压缓冲区  
* 如果从节点发送的 runid 与当前主节点的 runid 不一致时，则进行全量复制  

##### 1.3.2.2.2. 部分复制流程  
&emsp; 当主从节点在命令传播节点发生了网络中断，出现数据丢失情况，则从节点会向主节点请求发送丢失的数据，如果请求的偏移量在复制积压缓冲区中，则主节点就将剩余的数据补发给从节点，保持主从节点数据一致，由于补发的数据一般都会比较小，所以开销相当于全量复制而言也会很小，流程如下：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-43.png)  
&emsp; 1. 当主从节点出现网络闪退时，如果超过了 repl-timeout 时间，主节点会认为从节点出现故障不可达，打印日志如下：  

    -- master
    2496:M 19 May 2019 10:06:02.970 # Connection with replica 127.0.0.1:6391 lost.
    --slave
    2655:S 19 May 2019 10:21:59.515 * Connecting to MASTER no:6390
    2655:S 19 May 2019 10:21:59.516 # Unable to connect to MASTER: Undefined error: 0

&emsp; 由于主节点没有宕机，所以它依然会响应客户端命令，当然这些命令也不会丢失，都会存储在复制积压缓冲区中，默认 1MB。  

&emsp; 2. 当主从直接恢复连接，从节点再次连接主节点，打印日志如下：  

    2655:S 19 May 2019 10:22:00.082 * REPLICAOF 127.0.0.1:6390 enabled (user request from 'id=6 addr=127.0.0.1:55985 fd=7 name= age=11 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=42 qbuf-free=32726 obl=0 oll=0 omem=0 events=r cmd=slaveof')
    2655:S 19 May 2019 10:22:00.527 * Connecting to MASTER 127.0.0.1:6390
    2655:S 19 May 2019 10:22:00.527 * MASTER <-> REPLICA sync started
    2655:S 19 May 2019 10:22:00.528 * Non blocking connect for SYNC fired the event.
    2655:S 19 May 2019 10:22:00.528 * Master replied to PING, replication can continue...

&emsp; 这里一定要注意：不要关闭从节点然后启动，这样是模拟不出来的，一定是要执行 slaveof no one 命令，因为重启从节点，它的 master_replid 会丢失，在请求的时候因为 runid 不一致而导致全量复制，当然也选择将 slaveof 写入到配置文件中再重启，这样也可以进行部分复制。

&emsp; 3. 当主从建立连接后，由于从节点保存了主节点的 runid 和 offset ，所以只需要发送命令 psync{runid}{offset}即可，从节点打印日志如下：  

    2655:S 19 May 2019 10:22:00.529 * Trying a partial resynchronization (request dfa92dd668e0c6c3447af0d502b6ee4b0b07d75d:2268).

&emsp; 可以看到请求的 runid：dfa92dd668e0c6c3447af0d502b6ee4b0b07d75d，offset：2268  

&emsp; 4. 主节点接受从节点的 psync 命令，会先核对请求的 runid 是否和自身的的 runid 一致，如果一致，说明该从节点复制的当前主节点。然后查看请求的 offset 是否在复制积压缓冲区，如果在则进行部分复制，否则进行全量复制，部分复制回复 +CONTINUE 响应，从节点接受回复后，打印日志如下：  

    2655:S 19 May 2019 10:22:00.530 * Successful partial resynchronization with master.
    2655:S 19 May 2019 10:22:00.530 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.

&emsp; 5. 在进行部分复制时，主节点只需要根据 offset 将复制积压缓冲区的数据补发给从节点即可，主节点打印日志如下：

    2496:M 19 May 2019 10:22:00.529 * Replica 127.0.0.1:6391 asks for synchronization
    2496:M 19 May 2019 10:22:00.529 * Partial resynchronization request from 127.0.0.1:6391 accepted. Sending 183 bytes of backlog starting from offset 2268.

&emsp; 从日志中，可以看出主节点发送了 183 个字节数据给从节点。  

#### 1.3.2.3. psync 命令的执行过程详解  
&emsp; 在Redis 2.8以前一直都是通过命令sync进行全量复制，但是Redis 2.8以后都是通过命令psync进行全量复制和部分复制了，所以有必要了解下psync命令的执行过程，如下图：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Redis/redis-44.png)  
1. 首先从节点根据当前状态来决定如何调用psync命令
    * 如果主从节点从未建立过连接或者之间执行过slave of none，则从节点发送命令 psync?-1，向主节点请求全量复制。
    * 如果主从节点建立过连接，则发送命令 psync{runid}{offset} 尝试部分复制，具体是全量还是部分复制，则需要根据主节点的情况来确定
2. 主节点则根据自身情况来做出不同的响应：  
    * 如果主节点的版本低于2.8，则响应 -ERR，从节点接受该回复后发送 sync 进行全量复制  
    * 如果主节点发现请求的命令为 psync?-1, 则判定该从节点是第一次进行连接，则响应 +FULLRESYNC<runid\><offset\>，进行全量复制  
    * 如果主节点比对命令请求的 runid 和自身的 runid 不一致或者一致，但是请求的 offset 不在复制积压缓冲区中，则响应 +FULLRESYNC<runid><offset> 进行全量复制  
    * 如果主节点比对命令请求的 runid 和自身的 runid 一致，且 offset 也在复制积压缓冲区，则响应 +CONTINUE 进行部分复制  

## 1.4. 主从复制应用与问题  
### 1.4.1. 读写分离  
&emsp; 对于Redis的主从模式，可以利用主节点提供写服务，一个或者多个从节点提供读服务，这样就最大化 Redis 的读负载能力。 **<font color = "red">主从架构下的读写分离中的问题：数据延迟、读到过期数据、从节点故障。</font>**  

#### 1.4.1.1. 数据延迟  
&emsp; 在命令传送阶段，Redis主从节点同步命令的过程是异步的，所以势必会导致主从节点的数据不一致性，如果应用对数据的不一致性接受程度不是很高，则可以从以下几个方面优化：  

* 优化主从节点的网络环境（比如主从节点部署在同机房）  
* 监控从节点的延迟性，如果延迟过大，则通知应用不从该节点读取数据，这种方案需要改造客户端，工作量不小，一般不推荐  

&emsp; 从节点的 slave-serve-stale-data参数便与其有关，它控制着从节点对读请求的响应情况，如果为yes（默认值），则从节点可以响应客户端的命令，如果为 NO，则只能响应info、slaveof等少数命令。所以如果应用对数据一致性要求较高，则应该将该参数设置为NO。  

#### 1.4.1.2. 读到过期数据  
&emsp; Redis 内部对过期数据的删除有两种方案：惰性删除、定时删除。  

* 惰性删除：服务器不会主动删除数据，只有当客户端查询某个数据时，服务器判断该数据是否过期，过期则删除  
* 定时删除：服务器内部维护着一个定时任务，会定时删除过期的数据  

&emsp; 在主从复制的架构模式下，所有的写操作都发生在主节点，删除数据时也是在主节点删除，然后将删除命令同步给从节点，由于主从节点的延迟性，主节点删除了某些数据，从节点不一定也删除了，所以在从节点是比较容易读到过期的数据。但是在 Redis 3.2 版本中，增加了对数据是否过期的判断，如果读到的数据过期了，则不会返回给客户端了，所以将 Redis 升级到 3.2 版本后即可解决该问题。  

#### 1.4.1.3. 从节点故障  
&emsp; 如果从节点出现故障了，在没有哨兵的情况下，客户端是无法进行切换的，所以需要对客户端进行改造，它内部需要维护一个可用的 Redis 节点列表，当某个节点出现故障不可用后，立刻切换到其他可用节点，当然这种工作量会比较大，得不偿失。  

### 1.4.2. 规避全量复制  
&emsp; 全量复制要经过如下几个过程：建立连接 ---> 生成 RDB 文件 ---> 发送 RDB 文件 ---> 清空旧数据 ---> 加载新数据，五个过程，工作量巨大，是一个非常消耗资源的操作，所以我们需要尽可能的规避。有以下几种情况会发生全量复制：  

* 第一次复制
* 节点运行 ID 不匹配
* 复制偏移量 offset 不在复制积压缓冲区中

#### 1.4.2.1. 第一次复制  
&emsp; 这种情况是无法避免的，所以如果要对数据量较大且流量较高的主节点添加从节点的话，最好是选择在低峰值进行操作。

#### 1.4.2.2. 节点运行 ID 不匹配  
&emsp; 主节点重启时，runid 会发生改变，在进行复制的时候，发现主从节点的 runid 不一致，则进行全量复制，对于这种情况一般都应该在架构上面规避，比如提供故障转移的功能。还有一种情况就是修改了主节点的配置，需要重启才能够生效，这个时候如果重启主节点发生全量复制就得不偿失了，可以选择安全重启的方式（debug reload），在这种情况，主节点重启的 runid 是不会发生改变的。  

#### 1.4.2.3. 复制偏移量 offset 不在复制积压缓冲区中  
&emsp; 当主从节点因网络中断而重新连接，从节点会发送 psync 命令进行部分复制请求，主节点除了校验 runid 是否一致外还会判断请求的 offset 是否在缓冲区中，如果不在这进行全量复制。复制积压缓冲区默认大小为 1MB，这对于高流量的主节点而言势必显得有点儿小了，所以为了避免全量复制，需要根据中断时长来调整复制积压缓冲区的大小，调整为复制积压缓冲区大小>平均中断时长*平均写命令字节数，这样就可以避免因为复制积压缓冲区不足而导致的全量复制。  

### 1.4.3. 规避复制风暴  
&emsp; 复制风暴指的是大量从节点对同一主节点或者同一台服务器的多个主节点短时间内发起全量复制的过程。复制风暴会导致主节点消耗大量的 CPU、内存、宽带，所以需要尽可能的规避复制风暴。  

#### 1.4.3.1. 单一主节点  
&emsp; 这种情况一般都发生在一个主节点挂载了多个从节点，所以规避的方案也比较简单：  

* 减少挂载的从节点  
* 将架构调整为树状结构，增加中间层，但是这样导致的后果是中间层越多，后面节点的数据延迟就越高，同时也增加了运维的难度  

#### 1.4.3.2. 单一机器  
&emsp; Redis 的单线程运行的，所以会存在一台机器上面部署多个 Redis 服务，同时也有多个是主节点，所以规避方案：  

* 将主节点分散在多台不同的机器上  
* 当主节点所在机器故障后提供故障转移机制，避免机器回复后进行密集的全量复制  

<!-- 
主从复制的相关配置
https://mp.weixin.qq.com/s/OdvVn5pBG3Qlz9x6jttMXQ
-->

