

<!-- TOC -->

- [1. MySql日志文件](#1-mysql日志文件)
    - [1.1. MySql日志文件简介](#11-mysql日志文件简介)
    - [1.2. undo log，回滚日志](#12-undo-log回滚日志)
        - [1.2.1. 简介](#121-简介)
        - [1.2.2. 写入流程及刷盘时机](#122-写入流程及刷盘时机)
        - [1.2.3. 对应的物理文件](#123-对应的物理文件)
    - [1.3. redo log，重做日志](#13-redo-log重做日志)
        - [1.3.1. 为什么需要redo log](#131-为什么需要redo-log)
        - [1.3.2. redo log简介](#132-redo-log简介)
        - [1.3.3. 写入流程及刷盘时机](#133-写入流程及刷盘时机)
        - [1.3.4. 记录形式](#134-记录形式)
        - [1.3.5. 对应的物理文件](#135-对应的物理文件)
    - [1.4. binlog，二进制日志（归档日志）](#14-binlog二进制日志归档日志)
        - [1.4.1. 简介](#141-简介)
        - [1.4.2. 写入流程及刷盘时机](#142-写入流程及刷盘时机)
        - [1.4.3. binlog日志格式](#143-binlog日志格式)
        - [1.4.4. 物理文件](#144-物理文件)
    - [1.5. redo log与bin log的区别](#15-redo-log与bin-log的区别)
    - [1.6. 两阶段提交](#16-两阶段提交)
    - [1.7. 恢复](#17-恢复)

<!-- /TOC -->
<!-- 
重做日志缓冲(redo log buffer)
https://mp.weixin.qq.com/s/Cdq5aVYXUGQqxUdsnLlA8w

知识点：了解InnoDB的Checkpoint技术 
https://mp.weixin.qq.com/s/rQX3AFivFDNIYXE7-r9U_w
-->
**<font color = "red">参考《MySQL技术内幕：InnoDB存储引擎》</font>** 

# 1. MySql日志文件  

## 1.1. MySql日志文件简介  
&emsp; MySql日志文件有：  

* 错误日志（errorlog）：记录出错信息，也记录一些警告信息或者正确的信息。
* 一般查询日志（general log）：记录所有对数据库请求的信息，不论这些请求是否得到了正确的执行。
* 慢查询日志（slow query log）：设置一个阈值，将运行时间超过该值的所有SQL语句都记录到慢查询的日志文件中。
* 二进制日志（binlog）：记录对数据库执行更改的所有操作。
* 中继日志（relay log）：中继日志也是二进制日志，用来给slave库恢复。
* 重做日志（redo log）。
* 回滚日志（undo log）。

&emsp; 重点需要关注的是二进制日志(binlog)和事务日志(包括redo log和undo log)。  

&emsp; 此外日志可以分为<font color = "red">逻辑日志和物理日志</font>。  
    
* 逻辑日志：可以简单理解为记录的就是sql语句。
* 物理日志：因为mysql数据最终是保存在数据页中的，物理日志记录的就是数据页变更。

<!-- 
WAL
-->

## 1.2. undo log，回滚日志
<!-- 
* undo log（回滚日志）  实现原子性  
&emsp; undo log 主要为事务的回滚服务。在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。<font color = "red">undo log记录了数据在每个操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。</font>单个事务的回滚，只会回滚当前事务做的操作，并不会影响到其他的事务做的操作。  
&emsp; Undo记录的是已部分完成并且写入硬盘的未完成的事务，默认情况下回滚日志是记录下表空间中的（共享表空间或者独享表空间）  

&emsp; 二种日志均可以视为一种恢复操作，redo_log是恢复提交事务修改的页操作，而undo_log是回滚行记录到特定版本。二者记录的内容也不同，redo_log是物理日志，记录页的物理修改操作，而undo_log是逻辑日志，根据每行记录进行记录。  
-->
### 1.2.1. 简介
&emsp; 数据库事务四大特性中有一个是原子性，原子性底层就是通过undo log实现的。  
&emsp; Undo log是逻辑日记。undo log主要记录了数据的逻辑变化，比如一条INSERT语句，对应一条DELETE的undo log，对于每个UPDATE语句，对应一条相反的UPDATE的undo log，这样在发生错误时，就能回滚到事务之前的数据状态。  

&emsp; Undo log作用：

* 回滚数据：当程序发生异常错误时等，根据执行 Undo log 就可以回滚到事务之前的数据状态，保证原子性，要么成功要么失败。  
* MVCC 一致性视图：通过 Undo log 找到对应的数据版本号，是保证 MVCC 视图的一致性的必要条件。  

### 1.2.2. 写入流程及刷盘时机   
<!-- 
https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html
https://guobinhit.blog.csdn.net/article/details/79345359
-->
&emsp; **<font color = "red">事务开始之前，将当前的版本生成undo log。</font>** 产生undo日志的时候，同样会伴随类似于保护事务持久化机制的redolog的产生。  

### 1.2.3. 对应的物理文件    
&emsp; MySQL5.6之前，undo表空间位于共享表空间的回滚段中，共享表空间的默认的名称是ibdata，位于数据文件目录中。  

&emsp; MySQL5.6之后，undo表空间可以配置成独立的文件，但是提前需要在配置文件中配置，完成数据库初始化后生效且不可改变undo log文件的个数，如果初始化数据库之前没有进行相关配置，那么就无法配置成独立的表空间了。  

&emsp; 关于MySQL5.7之后的独立undo 表空间配置参数如下

    innodb_undo_directory = /data/undospace/ --undo独立表空间的存放目录
    innodb_undo_logs = 128 --回滚段为128KB
    innodb_undo_tablespaces = 4 --指定有4个undo log文件

&emsp; 如果undo使用的共享表空间，这个共享表空间中又不仅仅是存储了undo的信息，共享表空间的默认为与MySQL的数据目录下面，其属性由参数innodb_data_file_path配置。  

![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-81.png)  

<!-- 
&emsp; 默认情况下undo文件是保持在共享表空间的，也即ibdatafile文件中，当数据库中发生一些大的事务性操作的时候，要生成大量的undo信息，全部保存在共享表空间中的。  
&emsp; 因此共享表空间可能会变的很大，默认情况下，也就是undo 日志使用共享表空间的时候，被“撑大”的共享表空间是不会也不能自动收缩的。  
&emsp; 因此，mysql5.7之后的“独立undo 表空间”的配置就显得很有必要了。  
-->

## 1.3. redo log，重做日志
<!-- 
Log Buffer 
https://mp.weixin.qq.com/s/-Hx2KKYMEQCcTC-ADEuwVA
https://mp.weixin.qq.com/s/mNfjT99qIbjKGraZLV8EIQ
https://mp.weixin.qq.com/s/Cdq5aVYXUGQqxUdsnLlA8w
-->
<!-- 
redo 与检查点

InnoDB 使用日志先行策略，将数据修改先在内存中完成，并且将事务记录成重做日志(Redo Log)，转换为顺序IO高效的提交事务。
这里日志先行，说的是日志记录到数据库以后，对应的事务就可以返回给用户，表示事务完成。但是实际上，这个数据可能还只在内存中修改完，并没有刷到磁盘上去。内存是易失的，如果在数据落地前，机器挂了，那么这部分数据就丢失了。
InnoDB 通过 redo 日志来保证数据的一致性。如果保存所有的重做日志，显然可以在系统崩溃时根据日志重建数据。
当然记录所有的重做日志不太现实，所以 InnoDB 引入了检查点机制。即定期检查，保证检查点之前的日志都已经写到磁盘，则下次恢复只需要从检查点开始。
-->

### 1.3.1. 为什么需要redo log  
&emsp; 事务的四大特性里面有一个是一致性，具体来说就是只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态。那么mysql是如何保证一致性的呢？最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有严重的性能问题，主要体现在两个方面：  

* 因为Innodb是以页为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，太浪费资源了！  
* 一个事务可能涉及修改多个数据页，并且这些数据页在物理上并不连续，使用随机IO写入性能太差！  

&emsp; 因此mysql设计了redo log，具体来说就是只记录事务对数据页做了哪些修改，这样就能完美地解决性能问题了(相对而言文件更小并且是顺序IO)。  

### 1.3.2. redo log简介  
<!--
* redo log（重做日志） 实现持久化  
&emsp; 在innoDB的存储引擎中，事务日志通过重做(redo)日志和innoDB存储引擎的日志缓冲(InnoDB Log Buffer)实现。<font color = "red">事务开启时，事务中的操作，都会先写入存储引擎的日志缓冲中，在事务提交之前，这些缓冲的日志都需要提前刷新到磁盘上持久化，</font>这就是DBA们口中常说的“日志先行”(Write-Ahead Logging)。<font color = "red">当事务提交之后，在Buffer Pool中映射的数据文件才会慢慢刷新到磁盘。</font>此时如果数据库崩溃或者宕机，那么当系统重启进行恢复时，就可以根据redo log中记录的日志，把数据库恢复到崩溃前的一个状态。未完成的事务，可以继续提交，也可以选择回滚，这基于恢复的策略而定。  
&emsp; 在系统启动的时候，就已经为redo log分配了一块连续的存储空间，以顺序追加的方式记录Redo Log，通过顺序IO来改善性能。所有的事务共享redo log的存储空间，它们的Redo Log按语句的执行顺序，依次交替的记录在一起。  
-->
&emsp; 物理格式的日志，记录的是物理数据页面的修改的信息，这个页 “做了什么改动”。如：add xx记录 to Page1，向数据页Page1增加一个记录。    
&emsp; **作用：**  

* <font color = "red">确保事务的持久性。</font>防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性。  
* 提高性能：先写Redo log记录更新。当等到有空闲线程、内存不足、Redo log满了时 刷脏。写 Redo log 是顺序写入，刷脏是随机写，节省的是随机写磁盘的 IO 消耗（转成顺序写），所以性能得到提升。此技术称为WAL技术：Write-Ahead Logging，它的关键点就是先写日记磁盘，再写数据磁盘。  


### 1.3.3. 写入流程及刷盘时机
**什么时候产生：**  
&emsp; <font color = "lime">事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo log文件中。</font>  
&emsp; mysql支持三种将redo log buffer写入redo log file的时机，可以通过innodb_flush_log_at_trx_commit参数配置，各参数值含义如下：  

|参数值	|含义|
|---|---|
|0（延迟写）	|事务提交时不会将redo log buffer中日志写入到os buffer，而是每秒写入os buffer并调用fsync()写入到redo log file中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。|
|1（实时写，实时刷）	|事务每次提交都会将redo log buffer中的日志写入os buffer并调用fsync()刷到redo log file中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。|
|2（实时写，延迟刷）	|每次提交都仅写入到os buffer，然后是每秒调用fsync()将os buffer中的日志写入到redo log file。|

![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-142.png)  

### 1.3.4. 记录形式
&emsp; redo log实际上记录数据页的变更，而这种变更记录是没必要全部保存，因此redo log实现上采用了大小固定，循环写入的方式，当写到结尾时，会回到开头循环写日志。如下图：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-143.png)  
&emsp; 在innodb中，既有redo log需要刷盘，还有数据页也需要刷盘，redo log存在的意义主要就是降低对数据页刷盘的要求。在上图中，write pos表示redo log当前记录的LSN(逻辑序列号)位置，check point表示数据页更改记录刷盘后对应redo log所处的LSN(逻辑序列号)位置。write pos到check point之间的部分是redo log空着的部分，用于记录新的记录；check point到write pos之间是redo log待落盘的数据页更改记录。当write pos追上check point时，会先推动check point向前移动，空出位置再记录新的日志。  
&emsp; 启动innodb的时候，不管上次是正常关闭还是异常关闭，总是会进行恢复操作。因为redo log记录的是数据页的物理变化，因此恢复的时候速度比逻辑日志(如binlog)要快很多。重启innodb时，首先会检查磁盘中数据页的LSN，如果数据页的LSN小于日志中的LSN，则会从checkpoint开始恢复。还有一种情况，在宕机前正处于checkpoint的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度，此时会出现数据页中记录的LSN大于日志中的LSN，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。  

----
&emsp; <font color = "lime">redo log是循环写的，redo log不是记录数据页更新之后的状态，而是记录这个页做了什么改动。</font>  
&emsp; redo log是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，那么日志总共就可以记录4GB的操作。从头开始写，写到末尾就又回到开头循环写，如下图所示。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-94.png)  
&emsp; 图中展示了一组4个文件的redo log日志，checkpoint是当前要擦除的位置，擦除记录前需要先把对应的数据落盘（更新内存页，等待刷脏页）。write pos 到 checkpoint之间的部分可以用来记录新的操作，如果 write pos和checkpoint 相遇，说明 redolog 已满，这个时候数据库停止进行数据库更新语句的执行，转而进行 redo log 日志同步到磁盘中。checkpoint 到 write pos 之间的部分等待落盘（先更新内存页，然后等待刷脏页）。  
&emsp; 有了 redo log 日志，那么在数据库进行异常重启的时候，可以根据 redo log 日志进行恢复，也就达到了 crash-safe。  
&emsp; redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数建议设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。  

### 1.3.5. 对应的物理文件  

* 默认情况下，对应的物理文件位于数据库的data目录下的ib_logfile1&ib_logfile2  
* innodb_log_group_home_dir 指定日志文件组所在的路径，默认./ ，表示在数据库的数据目录下。  
* innodb_log_files_in_group 指定重做日志文件组中文件的数量，默认2  

&emsp; 关于文件的大小和数量，由以下两个参数配置  

* innodb_log_file_size 重做日志文件的大小。  
* innodb_mirrored_log_groups 指定了日志镜像文件组的数量，默认1  

**其他：**  
&emsp; **<font color = "red">很重要一点，redo log是什么时候写盘的？前面说了是在事物开始之后逐步写盘的。</font>**  
&emsp; <font color = "lime">之所以说重做日志是在事务开始之后逐步写入重做日志文件，而不一定是事务提交才写入重做日志缓存，原因就是，重做日志有一个缓存区Innodb_log_buffer，Innodb存储引擎先将重做日志写入innodb_log_buffer中。</font>Innodb_log_buffer的默认大小为8M(这里设置的16M)。  
    ![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-80.png)  

&emsp; <font color = "lime">然后会通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘。</font>  

* Master Thread 每秒一次执行刷新Innodb_log_buffer到重做日志文件。  
* 每个事务提交时会将重做日志刷新到重做日志文件。  
* 当重做日志缓存可用空间 少于一半时，重做日志缓存被刷新到重做日志文件。  

&emsp; 由此可以看出，重做日志通过不止一种方式写入到磁盘，尤其是对于第一种方式，Innodb_log_buffer到重做日志文件是Master Thread线程的定时任务。  
&emsp; 因此重做日志的写盘，并不一定是随着事务的提交才写入重做日志文件的，而是随着事务的开始，逐步开始的。  

&emsp; 在 MySQL 中，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。为了解决这个问题，MySQL 的设计者就采用了日志（redo log）来提升更新效率。  
&emsp; 而日志和磁盘配合的整个过程，其实就是 MySQL 里的 WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。  
&emsp; 具体来说，<font color = "lime">当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log（redolog buffer）里面，并更新内存（buffer pool），这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候（如系统空闲时），将这个操作记录更新到磁盘里面（刷脏页）。</font>  

## 1.4. binlog，二进制日志（归档日志）  
### 1.4.1. 简介  
&emsp; binlog用于记录数据库执行的写入性操作(不包括查询)信息，以二进制的形式保存在磁盘中。binlog是mysql的逻辑日志，并且由Server层进行记录，使用任何存储引擎的mysql数据库都会记录binlog日志。  
&emsp; binlog是通过追加的方式进行写入的，可以通过max_binlog_size参数设置每个binlog文件的大小，当文件大小达到给定值之后，会生成新的文件来保存日志。  

**<font color = "red">作用：</font>**  
&emsp; 在实际应用中，主要用在两个场景：主从复制和数据恢复  

* 主从复制：在Master端开启binlog，然后将binlog发送到各个Slave端，Slave端重放binlog从而达到主从数据一致。  
* 数据恢复：通过使用mysqlbinlog工具来恢复数据。  

### 1.4.2. 写入流程及刷盘时机   
&emsp; **写入流程：**  
&emsp; SQL修改语句先写 Binlog Buffer，事务提交时，按照一定的格式刷到磁盘中。  
&emsp; **binlog刷盘时机**  
&emsp; 对于InnoDB存储引擎而言，mysql通过sync_binlog参数控制biglog的刷盘时机，取值范围是0-N：  

    0：不去强制要求，由系统自行判断何时写入磁盘；
    1：每次commit的时候都要将binlog写入磁盘；
    N：每N个事务，才会将binlog写入磁盘。

&emsp; 从上面可以看出，sync_binlog最安全的是设置是1，这也是MySQL 5.7.7之后版本的默认值。但是设置一个大一些的值可以提升数据库性能，因此实际情况下也可以将值适当调大，牺牲一定的一致性来获取更好的性能。  

### 1.4.3. binlog日志格式  
&emsp; binlog日志有三种格式，分别为STATMENT、ROW和MIXED。  

    在 MySQL 5.7.7之前，默认的格式是STATEMENT，MySQL 5.7.7之后，默认值是ROW。日志格式通过binlog-format指定。

* STATMENT 基于SQL语句的复制(statement-based replication, SBR)，每一条会修改数据的sql语句会记录到binlog中。优点：不需要记录每一行的变化，减少了binlog日志量，节约了IO, 从而提高了性能；缺点：在某些情况下会导致主从数据不一致，比如执行sysdate()、slepp()等。  
* ROW 基于行的复制(row-based replication, RBR)，不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了。优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题；缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨  
* MIXED 基于STATMENT和ROW两种模式的混合复制(mixed-based replication, MBR)，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog  

### 1.4.4. 物理文件  
&emsp; 配置文件的路径为log_bin_basename，binlog日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新，生成新的日志文件。  
&emsp; 对于每个binlog日志文件，通过一个统一的index文件来组织。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-83.png)  

## 1.5. redo log与bin log的区别
&emsp; redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。  
&emsp; redo log 是物理日志，记录的是在某个数据页上做了什么修改；binlog 是逻辑日志，记录的是DDL和DML操作语句。  
&emsp; redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。追加写是指binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。  
&emsp; **redo log+bin log保证crash-safe，bin log日志用于数据恢复和主从复制。**    
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-144.png)  


---
&emsp; <font color = "red">二进制日志的作用之一是还原数据库的，这与redo log很类似，</font>很多人混淆过，但是两者有本质的不同  

* 作用不同：redo log是保证事务的持久性的，是事务层面的，binlog作为还原的功能，是数据库层面的（当然也可以精确到事务层面的），虽然都有还原的意思，但是其保护数据的层次是不一样的。
* 内容不同：<font color = "lime">redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录的就是sql语句。</font>
* 另外，两者日志产生的时间，可以释放的时间，在可释放的情况下清理机制，都是完全不同的。
* 恢复数据时候的效率，基于物理日志的redo log恢复数据的效率要高于语句逻辑日志的binlog。

&emsp; 关于事务提交时，redo log和binlog的写入顺序，为了保证主从复制时候的主从一致(当然也包括使用binlog进行基于时间点还原的情况)，是要严格一致的，MySQL通过两阶段提交过程来完成事务的一致性的，也即redo log和binlog的一致性的，理论上是先写redo log，再写binlog，两个日志都提交成功(刷入磁盘)，事务才算真正的完成。

<!-- 
1. redo log是在InnoDB存储引擎层产生，而binlog是MySQL数据库的上层产生的，并且二进制日志不仅仅针对INNODB存储引擎，MySQL数据库中的任何存储引擎对于数据库的更改都会产生二进制日志。  
2. 两种日志记录的内容形式不同。MySQL的binlog是逻辑日志，其记录是对应的SQL语句。而innodb存储引擎层面的重做日志是物理日志。  
3. 两种日志与记录写入磁盘的时间点不同，二进制日志只在事务提交完成后进行一次写入。而innodb存储引擎的重做日志在事务进行中不断地被写入，并日志不是随事务提交的顺序进行写入的。  
&emsp; 二进制日志仅在事务提交时记录，并且对于每一个事务，仅在事务提交时记录，并且对于每一个事务，仅包含对应事务的一个日志。而对于innodb存储引擎的重做日志，由于其记录是物理操作日志，因此每个事务对应多个日志条目，并且事务的重做日志写入是并发的，并非在事务提交时写入，其在文件中记录的顺序并非是事务开始的顺序。  
4. binlog不是循环使用，在写满或者重启之后，会生成新的binlog文件，redo log是循环使用。  
5. binlog可以作为恢复数据使用，主从复制搭建，redo log作为异常宕机或者介质故障后的数据恢复使用。  
-->

&emsp; redo log 和 binlog 是怎么关联起来的?  
&emsp; redo log 和 binlog 有一个共同的数据字段，叫XID。崩溃恢复的时候，会按顺序扫描 redo log：  

* 如果碰到既有 prepare、又有commit的redo log，就直接提交；  
* 如果碰到只有 parepare、而没有commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。  

-----

| |	redo log	|binlog|
|---|---|---|
|文件大小	|redo log的大小是固定的。	|binlog可通过配置参数max_binlog_size设置每个binlog文件的大小。|
|实现方式	|redo log是InnoDB引擎层实现的，并不是所有引擎都有。	binlog是Server层实现的，所有引擎都可以使用 binlog日志|
|记录方式	|redo log 采用循环写的方式记录，当写到结尾时，会回到开头循环写日志。	binlog 通过追加的方式记录，当文件大小大于给定值后，后续的日志会记录到新的文件上|
|适用场景	|redo log适用于崩溃恢复(crash-safe)	|binlog适用于主从复制和数据恢复|

&emsp; 由binlog和redo log的区别可知：binlog日志只用于归档，只依靠binlog是没有crash-safe能力的。但只有redo log也不行，因为redo log是InnoDB特有的，且日志上的记录落盘后会被覆盖掉。因此需要binlog和redo log二者同时记录，才能保证当数据库发生宕机重启时，数据不会丢失。  

## 1.6. 两阶段提交  
<!-- 
Redo log 两阶段提交
更新内存后引擎层写 Redo log 将状态改成 prepare 为预提交第一阶段，Server 层写 Binlog，将状态改成 commit为提交第二阶段。两阶段提交可以确保 Binlog 和 Redo log 数据一致性。  
-->
&emsp; MySQL 使用两阶段提交主要解决 binlog 和 redo log 的数据一致性的问题。  
&emsp; redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。下图为 MySQL 二阶段提交简图：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-96.png)  
&emsp; <font color = "lime">两阶段提交原理描述: </font> 

1. InnoDB redo log写盘，InnoDB事务进入prepare状态。  
2. 如果前面prepare成功，binlog写盘，那么再继续将事务日志持久化到binlog，如果持久化成功，那么InnoDB事务则进入commit状态(在redo log里面写一个commit记录)  

&emsp; 备注: 每个事务 binlog 的末尾，会记录一个 XID event，标志着事务是否提交成功，也就是说，recovery 过程中，binlog 最后一个 XID event 之后的内容都应该被 purge。

## 1.7. 恢复
<!-- 
https://www.jianshu.com/p/d0e16db410e4
Redo log 容灾恢复过程

MySQL的处理过程如下

    判断 redo log 是否完整，如果判断是完整（commit）的，直接用 Redo log 恢复
    如果 redo log 只是预提交 prepare 但不是 commit 状态，这个时候就会去判断 binlog 是否完整，如果完整就提交 Redo log，用 Redo log 恢复，不完整就回滚事务，丢弃数据。

只有在 redo log 状态为 prepare 时，才会去检查 binlog 是否存在，否则只校验 redo log 是否是 commit 就可以啦。怎么检查 binlog：一个完整事务 binlog 结尾有固定的格式。
-->
&emsp; 数据库关闭只有2种情况，正常关闭，非正常关闭（包括数据库实例crash及服务器crash）。正常关闭情况，所有buffer pool里边的脏页都会都会刷新一遍到磁盘，同时记录最新LSN到ibdata文件的第一个page中。而非正常关闭来不及做这些操作，也就是没及时把脏数据flush到磁盘，也没有记录最新LSN到ibdata file。  
&emsp; 当重启数据库实例的时候，数据库做2个阶段性操作：redo log处理，undo log及binlog 处理。(在崩溃恢复中还需要回滚没有提交的事务，提交没有提交成功的事务。<font color = "red">由于回滚操作需要undo日志的支持，undo日志的完整性和可靠性需要redo日志来保证，所以崩溃恢复先做redo前滚，然后做undo回滚。</font>)

<!-- 
怎么进行数据恢复？
binlog 会记录所有的逻辑操作，并且是采用追加写的形式。当需要恢复到指定的某一秒时，比如今天下午二点发现中午十二点有一次误删表，需要找回数据，那你可以这么做：
•首先，找到最近的一次全量备份，从这个备份恢复到临时库•然后，从备份的时间点开始，将备份的 binlog 依次取出来，重放到中午误删表之前的那个时刻。
这样你的临时库就跟误删之前的线上库一样了，然后你可以把表数据从临时库取出来，按需要恢复到线上库去。
-->
&emsp; 启动innodb的时候，不管上次是正常关闭还是异常关闭，总是会进行恢复操作。因为redo log记录的是数据页的物理变化，因此恢复的时候速度比逻辑日志(如binlog)要快很多。重启innodb时，首先会检查磁盘中数据页的LSN，如果数据页的LSN小于日志中的LSN，则会从checkpoint开始恢复。还有一种情况，在宕机前正处于checkpoint的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度，此时会出现数据页中记录的LSN大于日志中的LSN，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。  

---


