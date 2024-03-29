<!-- TOC -->

- [1. Dubbo运行流程](#1-dubbo运行流程)
    - [1.1. 初始化过程细节](#11-初始化过程细节)
        - [1.1.1. 解析服务](#111-解析服务)
        - [1.1.2. 暴露服务](#112-暴露服务)
        - [1.1.3. 引用服务](#113-引用服务)
        - [1.1.4. 拦截服务](#114-拦截服务)
    - [1.2. 远程调用细节](#12-远程调用细节)
        - [1.2.1. 服务提供者暴露一个服务的详细过程](#121-服务提供者暴露一个服务的详细过程)
        - [1.2.2. 服务消费者消费一个服务的详细过程](#122-服务消费者消费一个服务的详细过程)
        - [1.2.3. Invoker](#123-invoker)

<!-- /TOC -->


# 1. Dubbo运行流程  
## 1.1. 初始化过程细节
### 1.1.1. 解析服务  
&emsp; **<font color = "lime">基于dubbo.jar内的META-INF/spring.handlers配置，Spring在遇到dubbo名称空间时，会回调DubboNamespaceHandler。所有dubbo的标签，都统一用DubboBeanDefinitionParser进行解析，基于一对一属性映射，将XML标签解析为Bean对象。</font>**  
&emsp; 在ServiceConfig.export()或ReferenceConfig.get()初始化时，将Bean对象转换 URL格式，所有Bean属性转成URL的参数。然后将URL传给协议扩展点，基于扩展点的扩展点自适应机制，根据URL的协议头，进行不同协议的服务暴露或引用。  

### 1.1.2. 暴露服务  
1. 只暴露服务端口：  
&emsp; 在没有注册中心，直接暴露提供者的情况下，ServiceConfig解析出的URL的格式为：dubbo://service-host/com.foo.FooService?version=1.0.0。  
&emsp; 基于扩展点自适应机制，通过URL的dubbo://协议头识别，直接调用DubboProtocol的export()方法，打开服务端口。  
2. 向注册中心暴露服务：  
&emsp; 在有注册中心，需要注册提供者地址的情况下，ServiceConfig 解析出的URL的格式为: registry://registry-host/org.apache.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.foo.FooService?version=1.0.0")。  
&emsp; 基于扩展点自适应机制，通过URL的 registry:// 协议头识别，就会调用 RegistryProtocol 的 export() 方法，将 export 参数中的提供者 URL，先注册到注册中心。  
&emsp; 再重新传给 Protocol 扩展点进行暴露： dubbo://service-host/com.foo.FooService?version=1.0.0，然后基于扩展点自适应机制，通过提供者 URL 的 dubbo:// 协议头识别，就会调用 DubboProtocol 的 export() 方法，打开服务端口。  

------

1. 在容器启动的时候，通过ServiceConfig解析标签，创建dubbo标签解析器来解析dubbo的标签，容器创建完成之后，触发ContextRefreshEvent事件回调开始暴露服务
2. 通过ProxyFactory获取到invoker，invoker包含了需要执行的方法的对象信息和具体的URL地址
3. 再通过DubboProtocol的实现把包装后的invoker转换成exporter，然后启动服务器server，监听端口
4. 最后RegistryProtocol保存URL地址和invoker的映射关系，同时注册到服务中心  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Dubbo/dubbo-53.png)   

### 1.1.3. 引用服务
1. 直连引用服务：  
&emsp; 在没有注册中心，直连提供者的情况下，ReferenceConfig 解析出的URL的格式为：dubbo://service-host/com.foo.FooService?version=1.0.0。  
&emsp; 基于扩展点自适应机制，通过URL的dubbo://协议头识别，直接调用DubboProtocol的refer()方法，返回提供者引用。  
2. 从注册中心发现引用服务：  
&emsp; 在有注册中心，通过注册中心发现提供者地址的情况下，ReferenceConfig 解析出的 URL 的格式为： registry://registry-host/org.apache.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.foo.FooService?version=1.0.0")。  
&emsp; 基于扩展点自适应机制，通过 URL 的 registry:// 协议头识别，就会调用 RegistryProtocol 的 refer() 方法，基于 refer 参数中的条件，查询提供者 URL，如： dubbo://service-host/com.foo.FooService?version=1.0.0。  
&emsp; 基于扩展点自适应机制，通过提供者 URL 的 dubbo:// 协议头识别，就会调用 DubboProtocol 的 refer() 方法，得到提供者引用。  
&emsp; 然后 RegistryProtocol 将多个提供者引用，通过 Cluster 扩展点，伪装成单个提供者引用返回。  

----
&emsp; 服务暴露之后，客户端就要引用服务，然后才是调用的过程。  

1. 首先客户端根据配置文件信息从注册中心订阅服务
2. 之后DubboProtocol根据订阅的得到provider地址和接口信息连接到服务端server，开启客户端client，然后创建invoker
3. invoker创建完成之后，通过invoker为服务接口生成代理对象，这个代理对象用于远程调用provider，服务的引用就完成了
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Dubbo/dubbo-54.png)   

### 1.1.4. 拦截服务
&emsp; 基于扩展点自适应机制，所有的 Protocol 扩展点都会自动套上 Wrapper 类。  
&emsp; 基于 ProtocolFilterWrapper 类，将所有 Filter 组装成链，在链的最后一节调用真实的引用。  
&emsp; 基于 ProtocolListenerWrapper 类，将所有 InvokerListener 和 ExporterListener 组装集合，在暴露和引用前后，进行回调。  
&emsp; 包括监控在内，所有附加功能，全部通过 Filter 拦截实现。  

## 1.2. 远程调用细节
### 1.2.1. 服务提供者暴露一个服务的详细过程
&emsp; 下图是服务提供者暴露服务的主过程：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Dubbo/dubbo-29.png)   
&emsp; 首先ServiceConfig类拿到对外提供服务的实际类 ref(如：HelloWorldImpl)，然后通过ProxyFactory类的getInvoker方法使用ref生成一个AbstractProxyInvoker实例，到这一步就完成具体服务到Invoker的转化。接下来就是Invoker转换到Exporter的过程。  
&emsp; Dubbo处理服务暴露的关键就在Invoker转换到Exporter的过程，上图中的红色部分。下面以Dubbo和RMI这两种典型协议的实现来进行说明：  

* **Dubbo 的实现**  
&emsp; Dubbo 协议的 Invoker 转为 Exporter 发生在 DubboProtocol 类的 export 方法，它主要是打开 socket 侦听服务，并接收客户端发来的各种请求，通讯细节由 Dubbo 自己实现。  
* **RMI 的实现**  
&emsp; RMI 协议的 Invoker 转为 Exporter 发生在 RmiProtocol类的 export 方法，它通过 Spring 或 Dubbo 或 JDK 来实现 RMI 服务，通讯细节这一块由 JDK 底层来实现，这就省了不少工作量。  

### 1.2.2. 服务消费者消费一个服务的详细过程  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Dubbo/dubbo-30.png)   
&emsp; 上图是服务消费的主过程：  
&emsp; 首先 ReferenceConfig 类的 init 方法调用 Protocol 的 refer 方法生成 Invoker 实例(如上图中的红色部分)，这是服务消费的关键。接下来把 Invoker 转换为客户端需要的接口(如：HelloWorld)。  
&emsp; 关于每种协议如 RMI/Dubbo/Web service 等它们在调用 refer 方法生成 Invoker 实例的细节和上一章节所描述的类似。  

### 1.2.3. Invoker  
&emsp; 由于 Invoker 是 Dubbo 领域模型中非常重要的一个概念，很多设计思路都是向它靠拢。这就使得 Invoker 渗透在整个实现代码里，对于刚开始接触 Dubbo 的人，确实容易给搞混了。 下面用一个精简的图来说明<font color = "lime">最重要的两种 Invoker：服务提供 Invoker 和服务消费 Invoker：</font>  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/microService/Dubbo/dubbo-31.png)   
&emsp; 为了更好的解释上面这张图，结合服务消费和提供者的代码示例来进行说明：  
&emsp; 服务消费者代码：  

```java
public class DemoClientAction {
 
    private DemoService demoService;
 
    public void setDemoService(DemoService demoService) {
        this.demoService = demoService;
    }
 
    public void start() {
        String hello = demoService.sayHello("world");
    }
}
```
&emsp; 上面代码中的 DemoService 就是上图中服务消费端的 proxy，用户代码通过这个 proxy 调用其对应的 Invoker ，而该 Invoker 实现了真正的远程服务调用。  
&emsp; 服务提供者代码：  

```java
public class DemoServiceImpl implements DemoService {
 
    public String sayHello(String name) throws RemoteException {
        return "Hello " + name;
    }
}
```
&emsp; 上面这个类会被封装成为一个 AbstractProxyInvoker 实例，并新生成一个 Exporter 实例。<font color = "red">这样当网络通讯层收到一个请求后，会找到对应的 Exporter 实例，并调用它所对应的 AbstractProxyInvoker 实例，从而真正调用了服务提供者的代码。Dubbo 里还有一些其他的 Invoker 类，但上面两种是最重要的。</font>  

