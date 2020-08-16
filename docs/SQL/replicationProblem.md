

# 1.3. 复制的问题和解决方案  


## 《mysql深入浅出开发、优化与管理维护》

<!-- 
https://www.cnblogs.com/gered/p/11388986.html#_label0_7
-->

### 大对象blog ,text 传输  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-98.png)  


### 主从不一致后锁表
&emsp; flush tables with read lock;   

&emsp; 然后 show master status\G  
&emsp; 然后 show slave status\G 来查看从库同步状态 或者重新 change master to....  
&emsp; 然后 select master_pos_wait('mysql-bin.00002','389'); （即刚刚show master status找到的文件及位置），如果为1 表示超时退出 ，如果为0 则标识主从同步。  
&emsp; 最后再主库 unlock tables; 解锁  

### 如何查看主从延迟？  
.......

### 跳过错误  
&emsp; 跳过错误有两种方式：
1. 跳过指定数量的事务：（建议如果已经出现了错误，使用这种办法）

```
mysql>slave stop;
mysql>SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1        #跳过一个事务
mysql>slave start
```

2. 修改mysql的配置文件，通过slave_skip_errors参数来跳所有错误或指定类型的错误（建议配置时使用这种办法）
vi /etc/my.cnf
[mysqld]

```xml
#slave-skip-errors=1062,1053,1146 #跳过指定error no类型的错误，DDL错误类型包含 1007,1008,1050,1051,1054,1060,1061,1068,1091,1146（5.6可以用这个）
#slave-skip-errors=ddl_exist_errors #跳过DDL错误，all：跳过所有错误(mysql5.7才有ddl_exist_errors)  
```



### 如何提高复制性能？  

&emsp; 并行复制（5.6之后才有）  

```xml
#如果业务正在运行，那么直接在从库运行可能#5.7加入参数
slave-parallel-type=LOGICAL_CLOCK #5.7新增。这里设置的是基于组条件的（同一数据库内的也可以用多个SQL 重做线程）
slave-parallel-workers=4　　#默认为4#5.6加入参数#因为5.6的slave-parallel-type 参数，只能为DATABASE，是基于不同数据库之间的（也就是schema）并行，不用设置
slave-parallel-workers=2
```

### 主从切换  



## 《高性能MySql》相关章节
<!-- https://segmentfault.com/a/1190000018785903 -->

&emsp; **<font color = "red">《高性能MySql》第10章</font>**
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-61.png)  

&emsp; 中断MySQL的复制并不是件难事。因为实现简单，配置相当容易，但也意味着有很多 方式会导致复制停止，陷入混乱并中断。  

### 1.3.1. 复制延迟过大  
&emsp; 产生延迟的两种方式：  

* 突然产生延迟，然后再跟上；  
* 稳定的延迟增大  

&emsp; 前者通常是由于一条执行时间过长的 SQL 导致，而后者即使在没有慢语句也会出现。  
&emsp; 对于前者，可以通过备库上的慢查询日志来进行优化。在备库上开启 log_slow_slave_statement 选项，可以在慢查询日志中记录复制线程执行的语句。  


&emsp; ~~而对于后者，没有针对性的解决方案，只能通过各种方式提高备库的复制效率。而想去对备库做优化时，会发现，除了购买更快的磁盘和 CPU，并没有太多的调优空间。只能通过 MySQL 选项禁止某些额外的工作以减少备库的复制。可以通过下面几种方式：~~    
1. 使用 InnoDB 引擎时，设置 innodb_flush_log_at_trx_commit 值为 2，来使备库不要频繁的刷新磁盘，以提高事务提交效率。  
2. 禁止二进制日志记录。把 innodb_locks_unsafe_for_binlog 设置为 1，并把MyISAM的delay_key_write设置为ALL。要注意的是，这些设置是以安全换取速度，在将备库提升为主库时，记得把这些选项设置回安全的值。  
3. 拆分效率较低的复制 SQL，分离复杂语句中的 SELECT 和 UPDATE 语句，降低复制消耗，提高效率。

#### 1.3.1.1. 并行复制  
&emsp; 可以使用并行复制（并行是指从库多个SQL线程并行执行relay log），解决从库复制延迟的问题。  
&emsp; MySQL 5.7中引入基于组提交的并行复制，其核心思想：一个组提交的事务都是可以并行回放，因为这些事务都已进入到事务的prepare阶段，则说明事务之间没有任何冲突（否则就不可能提交）。  
&emsp; 判断事务是否处于一个组是通过last_committed变量，last_committed表示事务提交的时候，上次事务提交的编号，如果事务具有相同的last_committed，则表示这些事务都在一组内，可以进行并行的回放。  

### 1.3.2. 数据损坏或丢失
&emsp; 问题描述：服务器崩溃、断电、磁盘损坏、内存或网络错误等问题，导致数据损坏或丢失。  
&emsp; 问题原因：非正常关机导致没有把数据及时的写入硬盘。  

&emsp; 这种问题，一般可以分为几种情况导致：  

#### 1.3.2.1. 主库意外关闭
&emsp; 问题未发生，避免方案：设置主库的 sync_binlog 选项为 1。此选项表示 MySQL 是否控制 binlog 的刷新。当设置为 1 时，表示每次事务提交，MySQL 都会把 binlog 刷下去，是最安全，性能损耗也最大的设置。  
&emsp; 问题已发生，解决方案：指定备库从下一个二进制日志的开头重新读日志。但是一些日志事件将永久性丢失。可以使用Percona Toolkit中的 pt-table-checksum工具来检查主备一致性，以便于修复。  

#### 1.3.2.2. 备库意外关闭
&emsp; 备库意外关闭重启时，会去读 master.info 文件以找到上次停止复制的位置。但是在意外关闭的情况下，这个文件存储的信息可能是错误的。此外，备库也可能会尝试重新执行一些二进制文件，这可能会导致唯一索引错误。可以通过Percona Toolkit 中的 pt-slave-restart 工具，帮助备库重新执行日志文件。  
&emsp; 如果使用的是InnoDB表，可以在重启后观察 MySQL 的错误日志。InnoDB 在恢复过程中会打印出恢复点的二进制日志坐标，可以使用这个值来决定备库指向主库的偏移量。  

#### 1.3.2.3. 主库二进制日志损坏
&emsp; 如果主库上的二进制日志损坏，除了忽略损坏的位置外，别无选择。在忽略存储位置后，可以通过 FLUSH LOGS 命令在主库开始一个新的日志文件，然后将备库指向该文件的开始位置。  

#### 1.3.2.4. 备库中继日志损坏  
&emsp; 如果主库上的日志是完好的，有两种解决方案：  
1. 手工处理。找到 master binlog 日志的 pos 点，然后重新同步。  
2. 自动处理。mysql5.5 考虑到 slave 宕机中继日志损坏这一问题，只要在 slave 的的配置文件 my.cnf 里增加一个参数 relay_log_recovery=1 即可。  

#### 1.3.2.5. 二进制日志与 InnoDB 事务日志不同步  
&emsp; 由于各种各样的原因，MySQL 的复制碰到服务器崩溃、断电、磁盘损坏、内存或网络错误时，很难恢复当时丢失的数据。几乎都需要从某个点开始重启复制。  

### 1.3.3. 未定义的服务器ID  
&emsp; 如果没有在 my.cnf 里定义服务器 ID，虽然可以通过 CHANGE MASTER TO 来设置备库，但在启动复制时会遇到：

    mysql> START SLAVE;
    ERROR 1200 (HY000): The server us bit configured as slave; fix in config file or with CHANGE MASTER TO

&emsp; 这个报错可能会让人困惑。因为可能已经通过 CHANGE MASTER TO 设置了备库，并且通过 SHOW MASTER STATUS 也确认了，为什么还会有这样的报错呢？通过 SELECT @@server_id 可以获得一个值，要注意的是，这个值只是默认值，必须为备库显式地设置服务器 ID。也就是在 my.cnf 里显示的设置服务器 ID。  

### 1.3.4. 对未复制数据的依赖性  
&emsp; 如果在主库上有备库上不存在的数据库或数据表，复制就很容易中断，反之亦然。  

* 对于前者，假设在主库上有一个 single_master 表，备库没有。在主库上对此表进行操作后，备库在尝试回放这些操作时就会出现问题，导致复制中断。
* 对于后者，假设备库上有一个 single_slave 表，主库没有。在主库上执行创建 single_slave 表的语句时，备库在回放该建表语句时就会出现问题。  

&emsp; 对于此问题，能做的就是做好预防：  
1. 主备切换时，尽量在切换后对比数据，查清楚是否有不一致的表或库。
2. 一定不要在备库执行写操作。

### 1.3.5. 丢失的临时表  
&emsp; 临时表和基于语句的复制方式不相容。如果备库崩溃或者正常关闭，任何复制线程拥有的临时表都会丢失。重启备库后，所有依赖于该临时表的语句都会失败。  
&emsp; 复制时出现找不到临时表的异常时，可以做：  
1. 直接跳过错误，或者手动地创建一个名字和结构相同的表来代替消失的的临时表。  

&emsp; 临时表的特性：  
1. 只对创建临时表的连接可见。不会和其他拥有相同名字的临时表的连接起冲突；
2. 随着连接关闭而消失，无须显式的移除它们。

&emsp; **<font color = "red">更好使用临时表的方式：</font>**  
&emsp; 保留一个专用的数据库，在其中创建持久表，把它们作为伪临时表，以模拟临时表特性。只需要通过 CONNETCTION_ID() 的返回值，给临时表创建唯一的名字。  

&emsp; 伪临时表的优劣势：  

* 优势：更容易调试应用程序。可以通过别的连接来查看应用正在维护的数据；  
* 劣势：比临时表多一些开销。创建较慢伪临时表会较慢，因为表的 .frm 文件需要刷新到磁盘。  

### 1.3.6. InnoDB加锁读导致主备数据不一致  
&emsp; 使用共享锁，串行化更新，保证备库复制时数据一致。  
&emsp; 某些情况下，加锁读可以防止混乱。假设有两张表：tab1 没有数据，tab2 只有一行数据，值为 99。此时，有两个事务更新数据。事务 1 将 tab2 的数据插入到 tab1，事务 2 更新 tab2。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-62.png)  
1. 事务 1 使用获取 tab2 数据时，加入共享锁，并插入 tab1；  
2. 同时，事务 2 更新 tab2 数据时，由于写操作的排它锁机制，无法获取 tab2 的锁，等待；  
3. 事务 1 插入数据后，删除共享锁，提交事务，写入 binlog（此时 tab1 和 tab2 的记录值 都是 99）；  
4. 事务 2 获取到锁，更新数据，提交事务，写入 binlog（此时 tab1 的记录值为 99，tab2 的记录值为 100）。  

&emsp; 上述过程中，第二步非常重要。事务 2 尝试去更新 tab2 表，这需要在更新的行上加排他锁（写锁）。排他锁与其他锁不相容，包括事务 1 在行记录上加的共享锁。因此事务 2 需要等待事务 1 完成。备库在根据 binlog 进行复制时，会按同样的顺序先执行事务 1，再执行事务 2。主备数据一致。  

&emsp; 同样的过程，如果事务 1 在第一步时没有加共享锁，流程就变成：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SQL/sql-63.png)  
1. 事务 1 无锁读取 tab2 数据，并插入 tab1（此时 tab1 和 tab2 的记录值 都是 99）；  
2. 同时，事务 2 更新 tab2 数据，先与事务 1 提交事务，写入 binlog（此时 tab1 的记录值为 99，tab2 的记录值为 100）；  
3. 事务 1 提交事务，写入 binlog（此时记录值无变化）；  

&emsp; mysqldump --single-transaction --all-databases --master-data=1 --host=server1 | mysql --host=server2  
&emsp; 要注意的是，上述过程中，事务 2 先提交，先写入 binlog。在备库复制时，同样先执行事务 2，将 tab2 的记录值更新为 100。然后执行事务 1，读取 tab2 数据，插入 tab1，所以最终的结果是，tab1 的记录值和 tab2 的记录值都是 100。很明显，数据和主库有差异。  

&emsp; 建议在大多数情况下将 innodb_unsafe_for_binlog 的值设置为 0。基于行的复制由于记录了数据的变化而非语句，因此不会存在这个问题。    
 

### 1.3.7. 总结  
&emsp; <font color = "red">复制问题要分清楚是 master 的问题，还是 slave 的问题。master 问题找 binlog，slave 问题找 relaylog。</font>  





