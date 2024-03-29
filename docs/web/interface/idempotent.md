

<!-- TOC -->

- [1. 接口幂等](#1-接口幂等)
    - [1.1. 数据问题及产生原因](#11-数据问题及产生原因)
    - [1.2. 接口幂等](#12-接口幂等)
    - [1.3. 解决方案](#13-解决方案)

<!-- /TOC -->

# 1. 接口幂等  

<!-- 
你的项目是如何处理重复请求/并发请求的？
https://mp.weixin.qq.com/s/8I8fRoYuTED6S7EcbHllhQ
-->

<!-- 
基于状态机的乐观锁 ——解决幂等性问题
https://www.jianshu.com/p/c6e9ddbea022
-->

![image](https://gitee.com/wt1814/pic-host/raw/master/images/project/idempotent/ide-1.png)  

## 1.1. 数据问题及产生原因  
&emsp; 数据库中产生重复数据或数据不一致（假定程序业务代码没问题），绝大部分就是发生了重复的请求，<font color = "red">重复请求是指同一个请求因为某些原因被多次提交。导致这个情况会有几种场景：</font>  

* 微服务场景，在传统应用架构中调用接口，要么成功，要么失败。但是在微服务架构下，会有第三个情况【未知】，也就是超时。如果超时了，微服务框架会进行重试。
* 用户交互的时候多次点击。如：快速点击按钮多次。
* <font color = "red">MQ消息中间件，消息重复消费。</font>  
* 第三方平台的接口（如：支付成功回调接口），因为异常也会导致多次异步回调。  
* 其他中间件/应用服务根据自身的特性，也有可能进行重试。  

## 1.2. 接口幂等  
&emsp; 幂等（idempotent、idempotence）是一个数学与计算机学概念，常见于抽象代数中。在编程中一个幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同。幂等函数，或幂等方法，是指可以使用相同参数重复执行，并能获得相同结果的函数。这些函数不会影响系统状态，也不用担心重复执行会对系统造成改变。  
&emsp; **<font color = "red">HTTP/1.1中对幂等性的定义是：一次和多次请求某一个资源本身应该具有同样的结果（网络超时等问题除外）。</font>**也就是说，其任意多次执行对资源本身所产生的影响均与一次执行对影响相同。    

&emsp; <font color="red">编程中常见幂等：</font>  

* select查询天然幂等  
* delete删除也是幂等，删除同一个多次效果一样  
* update直接更新某个值的，幂等  
* update更新累加操作的，非幂等  
* insert非幂等操作，每次新增一条  

## 1.3. 解决方案  
<!-- 
SpringBoot + Redis + 注解 + 拦截器来实现接口幂等性校验 
https://mp.weixin.qq.com/s/L5lOUB_cbi67eyCHmCHhbQ

 写一个通用的幂等组件，艿艿觉得很有必要 
 https://mp.weixin.qq.com/s/y6Ybk4TTlaTxL2rrjBdZlA

-->

&emsp; 以下提供方案非唯一，例如redis+token整合。  

* 前端：  

    1. 前端js提交禁止按钮可以用一些js组件  
    2. 使用Post/Redirect/Get模式   
    &emsp; 在提交后执行页面重定向，这就是所谓的Post-Redirect-Get (PRG)模式。简言之，当用户提交了表单后，去执行一个客户端的重定向，转到提交成功信息页面。这能避免用户按F5导致的重复提交，而其也不会出现浏览器表单重复提交的警告，也能消除按浏览器前进和后退按导致的同样问题。  
    3. 在session中存放一个特殊标志  
    &emsp; 在服务器端，生成一个唯一的标识符，将它存入session，同时将它写入表单的隐藏字段中，然后将表单页面发给浏览器，用户录入信息后点击提交，在服务器端，获取表单中隐藏字段的值，与session中的唯一标识符比较，相等说明是首次提交，就处理本次请求，然后将session中的唯一标识符移除；不相等说明是重复提交，就不再处理。  

* 数据库和锁：  

    4. 唯一主键机制  
    &emsp; 利用数据库的主键唯一约束的特性，解决了在insert场景时幂等问题。但主键的要求不是自增的主键，这样就需要业务生成全局唯一的主键。如果是分库分表场景下，路由规则要保证相同请求下，落地在同一个数据库和同一表中，要不然数据库主键约束就不起效果了，因为是不同的数据库和表主键不相关。因为对主键有一定的要求，这个方案就跟业务有点耦合了，无法用自增主键了。  
    5. 数据库乐观锁  
    6. 数据库悲观锁  
    7. 分布式锁  

* 接口编码  

    8. select + insert  
    9. 状态机幂等  
    10. Token机制  

&emsp; **对外提供接口的api如何保证幂等？**  
&emsp; 对外提供接口为了支持幂等调用，接口可以有两个字段必须传，一个是来源source，一个是来源方序列号seq，这个两个字段在提供方系统里面做联合唯一索引，这样当第三方调用时，先在本方系统里面查询一下，是否已经处理过，返回相应处理结果；没有处理过，进行相应处理，返回结果。注意，为了幂等友好，一定要先查询一下，是否处理过该笔业务，不查询直接插入业务系统，会报错，但实际已经处理了。    

