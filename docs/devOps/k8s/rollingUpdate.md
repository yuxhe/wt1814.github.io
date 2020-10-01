


# kubernetes多种发布方式  
<!-- 

 k8s中蓝绿部署、金丝雀发布、滚动更新汇总 
 https://mp.weixin.qq.com/s?__biz=MzU0NjEwMTg4Mg==&mid=2247484195&idx=1&sn=b841f2ea305acfa2996a667d4ff4d99e&chksm=fb638c36cc140520e6905db5923afe163d7babb5d9eb6c5e8045a795c37b33a2a2e5541e3efd&scene=21#wechat_redirect
-->

**Kubernetes蓝绿部署，金丝雀发布，滚动更新的介绍**  

* 金丝雀发布（又称灰度发布、灰度更新）：  
&emsp; 金丝雀发布一般是先发1台机器，或者一个小比例，例如2%的服务器，主要做流量验证用，也称为金丝雀 (Canary) 测试，国内常称灰度测试。以前旷工下矿前，会先放一只金丝雀进去用于探测洞里是否有有毒气体，看金丝雀能否活下来，金丝雀发布由此得名。简单的金丝雀测试一般通过手工测试验证，复杂的金丝雀测试需要比较完善的监控基础设施配合，通过监控指标反馈，观察金丝雀的健康状况，作为后续发布或回退的依据。如果金丝测试通过，则把剩余的 V1 版本全部升级为 V2 版本。如果金丝雀测试失败，则直接回退金丝雀，发布失败。  
* 滚动更新：  
&emsp; 在金丝雀发布基础上的进一步优化改进，是一种自动化程度较高的发布方式，用户体验比较平滑，是目前成熟型技术组织所采用的主流发布方式。一次滚动式发布一般由若干个发布批次组成，每批的数量一般是可以配置的（可以通过发布模板定义）。例如，第一批1台（金丝雀），第二批10%，第三批 50%，第四批100%。每个批次之间留观察间隔，通过手工验证或监控反馈确保没有问题再发下一批次，所以总体上滚动式发布过程是比较缓慢的 (其中金丝雀的时间一般会比后续批次更长，比如金丝雀10 分钟，后续间隔 2分钟)。  
* 蓝绿部署：  
&emsp; 一些应用程序只需要部署一个新版本，并需要立即切到这个版本。因此，我们需要执行蓝/绿部署。在进行蓝/绿部署时，应用程序的一个新副本（绿）将与现有版本（蓝）一起部署。然后更新应用程序的入口/路由器以切换到新版本（绿）。然后，您需要等待旧（蓝）版本来完成所有发送给它的请求，但是大多数情况下，应用程序的流量将一次更改为新版本；Kubernetes不支持内置的蓝/绿部署。目前最好的方式是创建新的部署，然后更新应用程序的服务（如service）以指向新的部署；蓝绿部署是不停老版本，部署新版本然后进行测试，确认OK后将流量逐步切到新版本。蓝绿部署无需停机，并且风险较小。  
