

# Redisson  
&emsp; 官方网址：https://redisson.org/  
&emsp; 文档：https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95  

1. 概述
2. 配置方法  
    2.1. 程序化配置  
    2.2. 文件方式配置  
        2.2.1 通过YAML格式配置  
    2.3. 常用设置  
    2.4. 集群模式  
        2.4.1. 集群设置  
        2.4.2. 通过YAML文件配置集群模式  
    2.5. 云托管模式  
        2.5.1. 云托管模式设置  
        2.5.2. 通过YAML文件配置集群模式  
    2.6. 单Redis节点模式  
        2.6.1. 单节点设置  
        2.6.2. 通过YAML文件配置集群模式 
    2.7. 哨兵模式  
        2.7.1. 哨兵模式设置  
        2.7.2. 通过YAML文件配置集群模式  
    2.8. 主从模式  
        2.8.1. 主从模式设置  
        2.8.2. 通过YAML文件配置集群模式  
3. 程序接口调用方式
    3.1. 异步执行方式
    3.2. 异步流执行方式
4. 数据序列化
5. 单个集合数据分片（Sharding）
6. 分布式对象
    6.1. 通用对象桶（Object Bucket）
    6.2. 二进制流（Binary Stream）
    6.3. 地理空间对象桶（Geospatial Bucket）
    6.4. BitSet
        6.4.1. 数据分片（Sharding）（分布式RoaringBitMap）
    6.5. 原子整长形（AtomicLong）
    6.6. 原子双精度浮点（AtomicDouble）
    6.7. 话题（订阅分发）
        6.7.1. 模糊话题
    6.8. 布隆过滤器（Bloom Filter）
        6.8.1. 数据分片（Sharding）
    6.9. 基数估计算法（HyperLogLog）
    6.10. 整长型累加器（LongAdder）
    6.11. 双精度浮点累加器（DoubleAdder）
    6.12. 限流器（RateLimiter）
7. 分布式集合
    7.1. 映射（Map）
        7.1.1. 映射（Map）的元素淘汰（Eviction），本地缓存（LocalCache）和数据分片（Sharding）
        7.1.2. 映射持久化方式（缓存策略）
        7.1.3. 映射监听器（Map Listener）
        7.1.4. LRU有界映射
    7.2. 多值映射（Multimap）
        7.2.1. 基于集（Set）的多值映射（Multimap）
        7.2.2. 基于列表（List）的多值映射（Multimap）
        7.2.3. 多值映射（Multimap）淘汰机制（Eviction）
    7.3. 集（Set）
        7.3.1. 集（Set）淘汰机制（Eviction）
        7.3.2. 集（Set）数据分片（Sharding）
    7.4. 有序集（SortedSet）
    7.5. 计分排序集（ScoredSortedSet）
    7.6. 字典排序集（LexSortedSet）
    7.7. 列表（List）
    7.8. 队列（Queue）
    7.9. 双端队列（Deque）
    7.10. 阻塞队列（Blocking Queue）
    7.11. 有界阻塞队列（Bounded Blocking Queue）
    7.12. 阻塞双端队列（Blocking Deque）
    7.13. 阻塞公平队列（Blocking Fair Queue）
    7.14. 阻塞公平双端队列（Blocking Fair Deque）
    7.15. 延迟队列（Delayed Queue）
    7.16. 优先队列（Priority Queue）
    7.17. 优先双端队列（Priority Deque）
    7.18. 优先阻塞队列（Priority Blocking Queue）
    7.19. 优先阻塞双端队列（Priority Blocking Deque）
8. 分布式锁（Lock）和同步器（Synchronizer）
    8.1. 可重入锁（Reentrant Lock）
    8.2. 公平锁（Fair Lock）
    8.3. 联锁（MultiLock）
    8.4. 红锁（RedLock）
    8.5. 读写锁（ReadWriteLock）
    8.6. 信号量（Semaphore）
    8.7. 可过期性信号量（PermitExpirableSemaphore）
    8.8. 闭锁（CountDownLatch）
9. 分布式服务
    9.1. 分布式远程服务（Remote Service）
        9.1.1. 分布式远程服务工作流程
        9.1.2. 发送即不管（Fire-and-Forget）模式和应答回执（Ack-Response）模式
        9.1.3. 异步调用
        9.1.4. 取消异步调用
    9.2. 分布式实时对象（Live Object）服务
        9.2.1. 介绍
        9.2.2. 使用方法
        9.2.3. 高级使用方法
        9.2.4. 注解（Annotation）使用方法
        9.2.5. 使用限制
    9.3. 分布式执行服务（Executor Service）
        9.3.1. 分布式执行服务概述
        9.3.2. 任务
        9.3.3. 取消任务
    9.4. 分布式调度任务服务（Scheduler Service）
        9.4.1. 分布式调度任务服务概述
        9.4.2. 设定任务计划
        9.4.3. 通过CRON表达式设定任务计划
        9.4.4. 取消计划任务
    9.5. 分布式映射归纳服务（MapReduce）
        9.5.1. 介绍
        9.5.2. 映射（Map）类型的使用范例
        9.5.3. 集合（Collection）类型的使用范例
10. 额外功能
    10.1. 对Redis节点的操作
    10.2. 复杂多维对象结构和对象引用的支持
    10.3. 命令的批量执行
    10.4. 脚本执行
    10.5. 底层Redis客户端
11. Redis命令和Redisson对象匹配列表
12. 独立节点模式
    12.1. 概述
    12.2. 配置方法
        12.2.1. 配置参数
        12.2.2. 通过JSON和YAML配置文件配置独立节点
    12.3. 初始化监听器
    12.4. 嵌入式运行方法
    12.5. 命令行运行方法
    12.6. Docker方式运行方法
13. 工具
    13.1. 集群管理工具
        13.1.1. 创建集群
        13.1.2. 踢出节点
        13.1.3. 数据槽迁移
        13.1.4. 添加从节点
        13.1.5. 添加主节点
14. 第三方框架整合
    14.1. Spring框架整合
    14.2. Spring Cache整合
        14.2.1. 本地缓存
        14.2.2. 数据分片
        14.2.3. JSON和YAML配置
    14.3. Hibernate整合
        14.3.1. 本地缓存
        14.3.2. 数据分片
    14.4. Java缓存标准规范JCache API (JSR-107）
    14.5. Tomcat会话管理器（Tomcat Session Manager）
    14.6. Spring Session会话管理器
    14.7. JMX与Dropwizard Metrics
15. 项目依赖列表


