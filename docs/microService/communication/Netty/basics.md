<!-- TOC -->

- [1. 分布式通信基础](#1-分布式通信基础)
    - [1.1. Socket](#11-socket)
    - [1.2. linux的五种I/O模型](#12-linux的五种io模型)
        - [1.2.1. I/O交换流程](#121-io交换流程)
        - [1.2.2. 五种I/O通信模型](#122-五种io通信模型)
            - [1.2.2.1. 阻塞IO](#1221-阻塞io)
            - [1.2.2.2. 非阻塞IO](#1222-非阻塞io)
            - [1.2.2.3. 多路复用IO](#1223-多路复用io)
            - [1.2.2.4. 信号驱动IO](#1224-信号驱动io)
            - [1.2.2.5. 异步IO](#1225-异步io)
        - [1.2.3. 同步/异步、阻塞/非阻塞](#123-同步异步阻塞非阻塞)
            - [1.2.3.1. 同步和异步](#1231-同步和异步)
            - [1.2.3.2. 阻塞和非阻塞](#1232-阻塞和非阻塞)
            - [1.2.3.3. 小结](#1233-小结)
        - [1.2.4. 各I/O模型的对比与总结](#124-各io模型的对比与总结)
    - [1.3. 多路复用详解（select poll epoll）](#13-多路复用详解select-poll-epoll)

<!-- /TOC -->


# 1. 分布式通信基础  

<!-- 
 「网络IO套路」当时就靠它追到女友 
 https://mp.weixin.qq.com/s/x-AZQO5uiuu5svIvScotzA
  图解BIO、NIO、AIO、多路复用IO的区别 
 https://mp.weixin.qq.com/s/XFJX1sUYhTb8509FikgqGg
-->

## 1.1. Socket  

<!-- 
https://www.cnblogs.com/meier1205/p/5971313.html

-->
&emsp; netty是对socket的封装。一般很少直接使用socket来编程，使用框架比较多，而netty就是其中一种框架。  


## 1.2. linux的五种I/O模型  

<!-- 
https://segmentfault.com/a/1190000003063859

四图，读懂 BIO、NIO、AIO、多路复用 IO 的区别 
https://mp.weixin.qq.com/s/CRd3-vRD7xwoexqv7xyHRw
彤哥说netty系列之IO的五种模型
https://mp.weixin.qq.com/s?__biz=Mzg2ODA0ODM0Nw==&mid=2247484080&idx=1&sn=54d451db27af1067365ed1fef94a0b2d&chksm=ceb30e04f9c48712bcc13ecb14014fd3b244385881d1aabd66e794b14429ce938b8296f54297&mpshare=1&scene=1&srcid=&sharer_sharetime=1573694075606&sharer_shareid=b256218ead787d58e0b58614a973d00d&key=0fd7b4fa2fb2f076f6b32bb04fcdff32f38e3e711297c12c2ac01f3cda80a8dbf8e95fe381e01b6d0fb0124c2b23cde0c2d17b5f5363615e42acd8ef9d1dd60a86ac6cf94adacae356330adbe943613b&ascene=1&uin=MTE1MTYxNzY2MQ%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=9PZBgG0W8u5aIQH8JwuoebfJbcWXVv%2F8Jwpab0URWoWCafXeDrv6e7zaSa2n%2B7Oa
-->
### 1.2.1. I/O交换流程  
&emsp; 对于一次IO操作，数据会先拷贝到内核空间中，然后再从内核空间拷贝到用户空间中，所以一次read操作，会经历两个阶段：  
&emsp; （1）等待数据准备  
&emsp; （2）数据从内核空间拷贝到用户空间  
&emsp; 基于以上两个阶段就产生了五种不同的IO模式。  

### 1.2.2. 五种I/O通信模型  
&emsp; 在网络环境下，通俗的讲，将I/O分为两步：第一步是等待；第二步是数据搬迁。  
&emsp; 如果想要提高I/O效率，需要将等待时间降低。因此发展出来五种I/O模型，分别是：阻塞I/O模型、非阻塞I/O模型、多路复用I/O模型、异步I/O模型。其中，前四种被称为同步I/O。  

#### 1.2.2.1. 阻塞IO  
&emsp; 从进程发起IO操作，一直等待上述两个阶段完成。  
&emsp; 两阶段一起阻塞。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-1.png)  

#### 1.2.2.2. 非阻塞IO  
&emsp; 进程一直询问IO准备好了没有，准备好了再发起读取操作，这时才把数据从内核空间拷贝到用户空间。  
&emsp; 第一阶段不阻塞但要轮询，第二阶段阻塞。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-2.png)  

#### 1.2.2.3. 多路复用IO  
&emsp; 多个连接使用同一个select去询问IO准备好了没有，如果有准备好了的，就返回有数据准备好了，然后对应的连接再发起读取操作，把数据从内核空间拷贝到用户空间。  
&emsp; 两阶段分开阻塞。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-3.png)  

#### 1.2.2.4. 信号驱动IO  
&emsp; 进程发起读取操作会立即返回，当数据准备好了会以通知的形式告诉进程，进程再发起读取操作，把数据从内核空间拷贝到用户空间。  
&emsp; 第一阶段不阻塞，第二阶段阻塞。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-4.png)  

#### 1.2.2.5. 异步IO
&emsp; 进程发起读取操作会立即返回，等到数据准备好且已经拷贝到用户空间了再通知进程拿数据。  
&emsp; 两个阶段都不阻塞。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-5.png)  

### 1.2.3. 同步/异步、阻塞/非阻塞  
#### 1.2.3.1. 同步和异步
<!-- 
同步非同步的区别在于调用操作系统的recvfrom()的时候是否阻塞，可见除了最后的异步IO其它都是同步IO。

同步、异步  
&emsp; 同步请求，A调用B，B的处理是同步的，在处理完之前不会通知A，只有处理完之后才会明确的通知A。  
&emsp; 异步请求，A调用B，B的处理是异步的，B在接到请求后先告诉A已经接到请求了，然后异步去处理，处理完之后通过回调等方式再通知A。  
&emsp; 所以说，同步和异步最大的区别就是被调用方的执行方式和返回时机。同步指的是被调用方做完事情之后再返回，异步指的是被调用方先返回，然后再做事情，做完之后再想办法通知调用方。
-->
&emsp; 同步和异步其实是指CPU时间片到利用，主要看请求发起方对消息结果的获取是主动发起的，还是被动通知的，如下图所示。如果是请求方主动发起的，一直在等待应答结果（同步阻塞），或者可以先去处理其他事情，但要不断轮询查看发起的请求是否由应答结果（同步非阻塞），因为不管如何都要发起方主动获取消息结果，所以形式上还是同步操作。如果是由服务方通知的，也就是请求方发出请求后，要么一直等待通知（异步阻塞），要么先去干自己的事（异步非阻塞）。当事情处理完成后，服务方会主动通知请求方，它的请求已经完成，这就是异步。异步通知的方式一般通过状态改变、消息通知或者回调函数来完成，大多数时候采用的都是回调函数。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-8.png)  

#### 1.2.3.2. 阻塞和非阻塞  
<!-- 
阻塞、非阻塞  
&emsp; 阻塞请求，A调用B，A一直等着B的返回，别的事情什么也不干。  
&emsp; 非阻塞请求，A调用B，A不用一直等着B的返回，先去处理其他事情。  
&emsp; 所以说，阻塞非阻塞最大的区别就是在被调用方返回结果之前的这段时间内，调用方是否一直等待。阻塞指的是调用方一直等待别的事情什么都不做。非阻塞指的是调用方先去忙别的事情。
-->
&emsp; 阻塞和非阻塞在计算机的世界里，通常指针对I/O的操作，如网络I/O和磁盘I/O等。那么什么是阻塞和非阻塞呢？简单地说，就是调用来一个函数后，在等待这个函数返回结果之前，当前的线程是处于挂起状态还是运行状态。如果是挂起状态，就意味着当前线程什么都不能干，就等着获取结果，这就是同步阻塞；如果仍然是运行状态，就意味着当前线程是可以继续处理其他任务的，但要时不时地看一下是否由结果来，这就是同步非阻塞。具体如下图所示。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-9.png)  


#### 1.2.3.3. 小结  
<!-- 
&emsp; **阻塞、非阻塞和同步、异步的区别：**  
&emsp; 阻塞、非阻塞和同步、异步其实针对的对象是不一样的。阻塞、非阻塞说的是调用者，同步、异步说的是被调用者。
-->
&emsp; 从上面的描述中，可以看到阻塞和非阻塞通常是指在客户端发出请求后，在服务端处理这个请求的过程中，客户端本身是直接挂起等待结果，还是继续做其他的任务。而异步和同步则是对于请求结果的获取是客户端主动获取结果，还是由服务端来通知结果。从这一点来看，同步和阻塞其实描述的是两个不同角度的事情，阻塞和非阻塞指的是客户端等待消息处理时本身的状态，是挂起还是继续干别的。同步和异步指的是对于消息结果是客户端主动获取的，还是由服务端间接推送的。  

### 1.2.4. 各I/O模型的对比与总结  
&emsp; 前四种I/O模型都是同步I/O操作，它们的区别在于第一阶段，而第二阶段是一样的：在数据从内核拷贝到应用缓冲期间（用户空间），进程阻塞于recvfrom调用。  
&emsp; 下图是各I/O 模型的阻塞状态对比：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-6.png)  
&emsp; s从上图可以看出，阻塞程度：阻塞I/O>非阻塞I/O>多路复用I/O>信号驱动I/O>异步I/O，效率是由低到高到。最后，再看一下下表，从多维度总结了各I/O模型之间到差异。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/netty/netty-7.png)  

## 1.3. 多路复用详解（select poll epoll）
&emsp; select 有最大文件描述符的限制，只能监听到有几个文件描述符就绪了，得遍历所有文件描述符获取就绪的IO。  
&emsp; poll 没有最大文件描述符的限制，与select一样，只能监听到有几个文件描述符就绪了，得遍历所有文件描述符获取就绪的IO。  
&emsp; epoll 没有最大文件描述符的限制，它通过回调的机制，一旦某个文件描述符就绪了，迅速激活这个文件描述符，当进程下一次调用epoll_wait()的时候便得到通知。  
&emsp; 所以，在有大量空闲连接的时候，epoll的效率要高很多。  



<!-- 

IO模型，BIO、NIO、AIO  

* 同步阻塞IO（BIO，Block-IO）：用户进程在发起一个IO操作以后，必须等待IO操作的完成。只有当真正完成了IO操作以后，用户进程才能运行。Java传统的IO模型属于此种方式！  
* 同步非阻塞IO（NIO，Non-Block IO）：在此种方式下，用户进程发起一个IO操作以后便可返回做其它事情，但是用户进程需要时不时的询问IO操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的CPU资源浪费。其中目前Java的NIO就属于同步非阻塞IO。  
* 异步阻塞IO：此种方式下是指应用发起一个IO操作以后，不等待内核IO操作的完成，等内核完成IO操作以后会通知应用程序，这其实就是同步和异步最关键的区别，同步必须等待或者主动的去询问IO是否完成，那么为什么说是阻塞的呢？因为此时是通过select系统调用来完成的，而select函数本身的实现方式是阻塞的，而采用select函数有个好处就是它可以同时监听多个文件句柄，从而提高系统的并发性！  
* 异步非阻塞IO（AIO，Asynchronous IO）：在此种模式下，用户进程只需要发起一个IO操作然后立即返回，等IO操作真正的完成以后，应用程序会得到IO操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的IO读写操作，因为真正的IO读取或者写入操作已经由内核完成了。  

&emsp; 适用场景：  

* BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解。  
* NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。  
* AIO方式适用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK1.7开始支持。  
-->
