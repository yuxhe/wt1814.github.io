
<!-- TOC -->

- [1. Kafka监控与调优](#1-kafka监控与调优)
    - [1.1. kafka监控](#11-kafka监控)
    - [1.2. 监控工具](#12-监控工具)
    - [1.3. 系统调优](#13-系统调优)
    - [1.4. Kafka调优的四个层面](#14-kafka调优的四个层面)
    - [1.5. 定位性能瓶颈](#15-定位性能瓶颈)
    - [1.6. Java Consumer的调优](#16-java-consumer的调优)

<!-- /TOC -->

# 1. Kafka监控与调优  
<!-- 
https://zhuanlan.zhihu.com/p/43702590
-->
《Kafka实战》第8、9章  

## 1.1. kafka监控  
&emsp; 从五个维度来讨论Kafka的监控。首先是要监控Kafka集群所在的主机；第二是监控Kafka broker JVM的表现；第三点，要监控Kafka Broker的性能；第四，要监控Kafka客户端的性能。这里的所指的是广义的客户端——可能是指编写的生产者、消费者，也有可能是社区提供的生产者、消费者，比如说Connect的Sink/Source或Streams等；最后需要监控服务器之间的交互行为。  

1. 主机监控  
2. 监控JVM  
&emsp; Kafka本身是一个普通的Java进程，所以任何适用于JVM监控的方法对于监控Kafka都是相通的。
3. Broker监控  
&emsp; **对于Broker的监控，主要是通过JMS指标来做的。**Kafka社区提供了特别多的JMS指标，这里列了一些比较重要的：  
&emsp; 首先是broker机器每秒出入的字节数，就是类似于可以监控网卡的流量，，并实时与网卡带宽进行比较——如果发现该值非常接近于带宽的话，就证明broker负载过高，要么增加新的broker机器，要么把该broker上的负载均衡到其他机器上。  
&emsp; 另外还有两个线程池空闲使用率要关注，最好确保它们的值都不要低于30%，否则说明Broker已经非常的繁忙。 此时需要调整线程池线程数。  
&emsp; 接下来是监控broker服务器的日志。日志中包含了非常丰富的信息。这里所说的日志不仅是broker服务器的日志，还包括Kafka controller的日志。需要经常性地查看日志中是否出现了OOM错误抑或是时刻关注日志中抛出的ERROR信息。  
&emsp; 还需要监控一些关键后台线程的运行状态。个人认为有两个比较重要的线程需要监控：一个Log Cleaner线程——该线程是执行数据压实操作的，如果该线程出问题了，用户通常无法感知到，然后会发现所有compact策略的topic会越来越大直到占满所有磁盘空间；另一个线程就是副本拉取线程，即follower broker使用该线程实时从leader处拉取数据。如果该线程“挂掉”了，用户通常也是不知道的，但会发现follower不再拉取数据了。因此一定要定期地查看这两个线程的状态，如果发现它们意味终止，则去找日志中寻找对应的报错信息。  
4. Clients监控  
&emsp; 客户端监控会分为两个，分别讨论对生产者和消费者的监控。  
&emsp; **生产者监控：**  
&emsp; 监控PRODUCE请求的处理延时。一条消息从生产者端发送到Kafka broker进行处理，之后返回给producer的总时间。整个链路中各个环节的耗时最好要做到心中有数。因为很多情况下，如果要提升生产者的TPS，了解整个链路中的瓶颈后才能做到有的放矢。  
&emsp; **消费者监控**  
&emsp; 新版本的消费者也是双线程设计，后面有一个心跳线程，如果这个线程挂掉的话，前台线程是不知情的。所以，用户最好定期监控该心跳线程的存活情况。心跳线程定期发心跳请求给Kafka服务器，告诉Kafka，这个消费者实例还活着，以避免coordinator错误地认为此实例已“死掉”从而开启rebalance。Kafka提供了很多的JMX指标可以用于监控消费者，最重要的消费进度滞后监控，也就是所谓的consumerlag。  
&emsp; 假设producer生产了100条消息，消费者读取了80条，那么lag就是20。显然落后的越少越好，这表明消费者非常及时，用户也可以用工具行命令来查lag，甚至写Java的API来查。与lag对应的还有一个lead指标，它表征的是消费者领先第一条消息的进度。比如最早的消费位移是1，如果消费者当前消费的消息是10，那么lead就是9。对于lead而言越大越好，否则表明此消费者可能处于停顿状态或者消费的非常慢，本质上lead和lag是一回事。
&emsp; 除了以上这些，还需要监控消费者组的分区分配情况，避免出现某个实例被分配了过多的分区，导致负载严重不平衡的情况出现。一般来说，如果组内所有消费者订阅的是相同的主题，那么通常不会出现明显的分配倾斜。一旦各个实例订阅的主题不相同且每个主题分区数参差不齐时就极易发生这种不平衡的情况。Kafka目前提供了3种策略来帮助用户完成分区分配，最新的策略是黏性分配策略，它能保证绝对的公平下。  
&emsp; 最后就是要监控rebalance的时间——目前来看，组内超多实例的rebalance性能很差，可能都是小时级别的。而且比较悲剧的是当前无较好的解决方案。所以，如果你的Consumer特别特别多的话，一定会有这个问题，你监控一下两个步骤所用的时间，看看是否满足需求，如果不能满足的话，看看能不能把消费者去除，尽量减少消费者数量。
5. Inter-Broker监控  
&emsp; 最后一个维度就是监控Broker之间的表现，主要是指副本拉取。Follower副本实时拉取leader处的数据，自然希望这个拉取过程越快越好。Kafka提供了一个特别重要的JMX指标，叫做备份不足的分区数，比如说规定了这条消息，应该在三个Broker上面保存，假设只有一个或者两个Broker上保存该消息，那么这条消息所在的分区就被称为“备份不足”的分区。这种情况是特别关注的，因为有可能造成数据的丢失。《Kafka权威指南》一书中是这样说的：如果你只能监控一个Kafka JMX指标，那么就监控这个好了，确保在你的Kafka集群中该值是永远是0。一旦出现大于0的情形赶紧处理。  

## 1.2. 监控工具  
&emsp; 下面这些是Kafka发展历程上比较有名气的监控系统。  

1. Kafka Manager：应该算是最有名的专属Kafka监控框架了，是独立的监控系统。  
2. Kafka Monitor：LinkedIn 开源的免费框架，支持对集群进行系统测试，并实时监控测试结果。  
3. CruiseControl：也是 LinkedIn 公司开源的监控框架，用于实时监测资源使用率，以及 提供常用运维操作等。无 UI 界面，只提供 REST API。  
4. JMX 监控：由于 Kafka 提供的监控指标都是基于 JMX 的，因此，市面上任何能够集成 JMX 的框架都可以使用，比如 Zabbix 和 Prometheus。  
5. 已有大数据平台自己的监控体系：像 Cloudera 提供的 CDH 这类大数据平台，天然就提 供 Kafka 监控方案。  
6. JMXTool：社区提供的命令行工具，能够实时监控 JMX 指标。答上这一条，属于绝对 的加分项，因为知道的人很少，而且会给人一种你对 Kafka 工具非常熟悉的感觉。如果 你暂时不了解它的用法，可以在命令行以无参数方式执行一下kafka-run-class.sh kafka.tools.JmxTool，学习下它的用法。  
7. [KafkaEagle剖](https://www.cnblogs.com/smartloli/p/9371904.html)  

## 1.3. 系统调优  
&emsp; Kafka监控的一个主要的目的就是调优Kafka集群。这里罗列了一些常见的操作系统级的调优。  
&emsp; 首先是保证页缓存的大小——至少要设置页缓存为一个日志段的大小。Kafka大量使用页缓存，只要保证页缓存足够大，那么消费者读取消息时就有大概率保证它能够直接命中页缓存中的数据而无需从底层磁盘中读取。故只要保证页缓存要满足一个日志段的大小。  
&emsp; 第二是调优文件打开数。很多人对这个资源有点畏手畏脚。实际上这是一个很廉价的资源，设置一个比较大的初始值通常都是没有什么问题的。  
&emsp; 第三是调优vm.max_map_count参数。主要适用于Kafka broker上的主题数超多的情况。Kafka日志段的索引文件是用映射文件的机制来做的，故如果有超多日志段的话，这种索引文件数必然是很多的，极易打爆这个资源限制，所以对于这种情况一般要适当调大这个参数。  
&emsp; 第四是swap的设置。很多文章说把这个值设为0，就是完全禁止swap，个人不建议这样，因为当设置成为0的时候，一旦内存耗尽了，Linux会自动开启OOM killer然后随机找一个进程杀掉。这并不是希望的处理结果。相反，建议设置该值为一个比较接近零的较小值，这样当内存快要耗尽的时候会尝试开启一小部分swap，虽然会导致broker变得非常慢，但至少给了用户发现问题并处理之的机会。  
&emsp; 第五JVM堆大小。首先鉴于目前Kafka新版本已经不支持Java7了，而Java 8本身不更新了，甚至Java9其实都不做了，直接做Java10了，所以我建议Kafka至少搭配Java8来搭建。至于堆的大小，个人认为6-10G足矣。  
&emsp; 最后，建议使用专属的多块磁盘来搭建Kafka集群。自1.1版本起Kafka正式支持JBOD，因此没必要在底层再使用一套RAID了。  

## 1.4. Kafka调优的四个层面  
&emsp; **Kafka调优通常可以从4个维度展开，分别是吞吐量、延迟、持久性和可用性。**在具体展开这些方面之前，先建议用户保证客户端与服务器端版本一致。如果版本不一致，就会出现向下转化的问题。举个例子，服务器端保存高版本的消息，当低版本消费者请求数据时，服务器端就要做转化，先把高版本消息转成低版本再发送给消费者。这件事情本身就非常非常低效。很多文章都讨论过Kafka速度快的原因，其中就谈到了零拷贝技术——即数据不需要在页缓存和堆缓存中来回拷贝。  
&emsp; 简单来说producer把生产的消息放到页缓存上，如果两边版本一致，可以直接把此消息推给Consumer，或者Consumer直接拉取，这个过程是不需要把消息再放到堆缓存。但是要做向下转化或者版本不一致的话，就要额外把数据再堆上，然后再放回到Consumer上，速度特别慢。  
1. Kafka调优 – 吞吐量  
&emsp; **调优吞吐量就是想用更短的时间做更多的事情。**这里列出了客户端需要调整的参数。前面说过了producer是把消息放在缓存区，后端Sender线程从缓存区拿出来发到broker。这里面涉及到一个打包的过程，它是批处理的操作，不是一条一条发送的。因此这个包的大小就和TPS息息相关。通常情况下调大这个值都会让TPS提升，但是也不会无限制的增加。不过调高此值的劣处在于消息延迟的增加。除了调整batch.size，设置压缩也可以提升TPS，它能够减少网络传输IO。当前Lz4的压缩效果是最好的，如果客户端机器CPU资源很充足那么建议开启压缩。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/mq/kafka/kafka-33.png)  
&emsp; 对于消费者端而言，调优TPS并没有太好的办法，能够想到的就是调整fetch.min.bytes。适当地增加该参数的值能够提升consumer端的TPS。对于Broker端而言，通常的瓶颈在于副本拉取消息时间过长，因此可以适当地增加num.replica.fetcher值，利用多个线程同时拉取数据，可以加快这一进程。  
2. Kafka调优 – 延时  
&emsp; **所谓的延时就是指消息被处理的时间。**某些情况下自然是希望越快越好。针对这方面的调优，consumer端能做的不多，简单保持fetch.min.bytes默认值即可，这样可以保证consumer能够立即返回读取到的数据。讲到这里，可能有人会有这样的疑问：TPS和延时不是一回事吗？假设发一条消息延时是2ms，TPS自然就是500了，因为一秒只能发500消息，其实这两者关系并不是简单的。因为我发一条消息2毫秒，但是如果把消息缓存起来统一发，TPS会提升很多。假设发一条消息依然是2ms，但是我先等8毫秒，在这8毫秒之内可能能收集到一万条消息，然后再发。相当于在10毫秒内发了一万条消息，大家可以算一下TPS是多少。事实上，Kafka producer在设计上就是这样的实现原理。  
3. Kafka调优 –消息持久性  
&emsp; 消息持久化本质上就是消息不丢失。Kafka对消息不丢失的承诺是有条件的。以前碰到很多人说我给Kafka发消息，发送失败，消息丢失了，怎么办？严格来说Kafka不认为这种情况属于消息丢失，因为此时消息没有放到Kafka里面。Kafka只对已经提交的消息做有条件的不丢失保障。  
&emsp; 如果要调优持久性，对于producer而言，首先要设置重试以防止因为网络出现瞬时抖动造成消息发送失败。一旦开启了重试，还需要防止乱序的问题。比如说我发送消息1与2，消息2发送成功，消息1发送失败重试，这样消息1就在消息2之后进入Kafka，也就是造成乱序了。如果用户不允许出现这样的情况，那么还需要显式地设置max.in.flight.requests.per.connection为1。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/mq/kafka/kafka-34.png)  
4. Kafka调优 –可用性  
&emsp; 最后是可用性，与刚才的持久性是相反的，我允许消息丢失，只要保证系统高可用性即可。因此我需要把consumer心跳超时设置为一个比较小的值，如果给定时间内消费者没有处理完消息，该实例可能就被踢出消费者组。我想要其他消费者更快地知道这个决定，因此调小这个参数的值。  

## 1.5. 定位性能瓶颈  
&emsp; 下面就是性能瓶颈，严格来说这不是调优，这是解决性能问题。对于生产者来说，如果要定位发送消息的瓶颈很慢，需要拆解发送过程中的各个步骤。就像这张图表示的那样，消息的发送共有6步。第一步就是生产者把消息放到Broker，第二、三步就是Broker把消息拿到之后，写到本地磁盘上，第四步是follower broker从Leader拉取消息，第五步是创建response；第六步是发送回去，告诉我已经处理完了。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/mq/kafka/kafka-35.png)  
&emsp; 这六步当中你需要确定瓶颈在哪？怎么确定？——通过不同的JMX指标。比如说步骤1是慢的，可能你经常碰到超时，你如果在日志里面经常碰到request timeout，就表示1是很慢的，此时要适当增加超时的时间。如果2、3慢的情况下，则可能体现在磁盘IO非常高，导致往磁盘上写数据非常慢。倘若是步骤4慢的话，查看名为remote-time的JMX指标，此时可以增加fetcher线程的数量。如果5慢的话，表现为response在队列导致待的时间过长，这时可以增加网络线程池的大小。6与1是一样的，如果你发现1、6经常出问题的话，查一下你的网络。所以，就这样来分解整个的耗时。这是到底哪一步的瓶颈在哪，需要看看什么样的指标，做怎样的调优。  

## 1.6. Java Consumer的调优
&emsp; 最后说一下Consumer的调优。目前消费者有两种使用方式，一种是同一个线程里面就直接处理，另一种是我采用单独的线程，consumer线程只是做获取消息，消息真正的处理逻辑放到单独的线程池中做。这两种方式有不同的使用场景：第一种方法实现较简单，因为你的消息处理逻辑直接写在一个线程里面就可以了，但是它的缺陷在于TPS可能不会很高，特别是当你的客户端的机器非常强的时候，用单线程处理的时候是很慢的，因为没有充分利用线程上的CPU资源。第二种方法的优势是能够充分利用底层服务器的硬件资源，TPS可以做的很高，但是处理提交位移将会很难。  
&emsp; 最后说一下参数，也是网上问的最多的，这几个参数到底是做什么的。第一个参数，就是控制consumer单次处理消息的最大时间。比如说设定的是600s，那么consumer给你10分钟来处理。如果10分钟内consumer无法处理完成，那么coordinator就会认为此consumer已死，从而开启rebalance。  
Coordinator是用来管理消费者组的协调者，协调者如何在有效的时间内，把消费者实例挂掉的消息传递给其他消费者，就靠心跳请求，因此可以设置heartbeat.interval.ms为一个较小的值，比如5s。  
