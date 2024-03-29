

<!-- TOC -->

- [1. 循环依赖](#1-循环依赖)
    - [1.1. 什么是循环依赖？](#11-什么是循环依赖)
    - [Spring循环依赖的场景](#spring循环依赖的场景)
    - [1.3. Spring如何解决循环依赖的问题?](#13-spring如何解决循环依赖的问题)
    - [1.4. 常见问题](#14-常见问题)

<!-- /TOC -->

<!-- 
什么是循环依赖？
循环依赖怎么解决？
Spring中为什么要使用三级缓存来解决循环依赖问题？二级缓存能不能解决循环依赖问题？

https://mp.weixin.qq.com/s/p01mrjBwstK74d3D3181og

循环依赖
https://mp.weixin.qq.com/s/-gLXHd_mylv_86sTMOgCBg
 【死磕 Spring】—– IOC 之循环依赖处理 
https://mp.weixin.qq.com/s/cxSSbbfFUNDUi9_fLfzSTw

https://mp.weixin.qq.com/s/YQRO2ZTn4T6A-iPUfs5ROg

/**
 * 一级缓存，用于存放已经初始化完成的Spring Bean（经历了完整的Spring Bean初始化生命周期 ）
 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/**
 * 二级缓存，用于存放已经被创建，但是尚未初始化完成的Bean（尚未经历了完整的Spring Bean初始化生命周期 ）
 * 这种对象提前暴露出来，就是为了解决循环引用，避免“鸡生蛋，蛋生鸡”的问题
 */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);


/**
 * 三级缓存，用于存放二级缓存中Bean的工厂
 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
-->

<!-- 
~~
https://juejin.cn/post/6895753832815394824
-->

# 1. 循环依赖  

## 1.1. 什么是循环依赖？  
&emsp; 多个bean之间相互依赖，形成了一个闭环。比如：A依赖于B、B依赖于C、C依赖于A。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SSM/Spring/spring-5.png)  

&emsp; 代码中表示：  
<!-- https://mp.weixin.qq.com/s/qXvKA0sIzo3JbledSr4NNQ -->

```java
public class A{
    B b;
}
public class B{
    C c;
}
public class C{
    A a;
}
```

## Spring循环依赖的场景  
<!-- 
https://mp.weixin.qq.com/s/wykX4CdrHT1tvSz2-24NEQ
-->
&emsp; Spring创建bean主要的几个步骤(参考SpringBean生命周期)：  

* 步骤1：实例化bean，即调用构造器创建bean实例  
* 步骤2：填充属性，注入依赖的bean，比如通过set方式、@Autowired注解的方式注入依赖的bean  
* 步骤3：bean的初始化，比如调用init方法等。    

&emsp; 从上面3个步骤中可以看出，<font color = "lime">注入依赖的对象，有2种情况：</font>  
1. **<font color = "red">通过步骤1中构造器的方式注入依赖</font>**  
2. **<font color = "red">通过步骤2填充field属性注入依赖</font>**  

&emsp; 在这两种情况，都有可能出现循环依赖。并非所有循环依赖，Spring都能自动解决。    
&emsp; <font color = "lime">Spring解决循环依赖的前置条件：</font>  
1. 出现循环依赖的Bean必须要是单例  
2. 依赖注入的方式不能全是构造器注入的方式（很多博客上说，只能解决setter方法的循环依赖，这是错误的）  

&emsp; 否则，抛出BeanCurrentlyInCreationException异常，表示循环依赖。  

&emsp; <font color = "red">检测循环依赖比较简单，使用一个列表来记录正在创建中的bean，bean创建之前，先去记录中看一下自己是否已经在列表中了，如果在，说明存在循环依赖，如果不在，则将其加入到这个列表，bean创建完毕之后，将其再从这个列表中移除。</font>  
&emsp; 源码示例：Spring创建单例bean时，会调用以下方法  

```java
protected void beforeSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
}
```
&emsp; singletonsCurrentlyInCreation就是用来记录目前正在创建中的bean名称列表，this.singletonsCurrentlyInCreation.add(beanName)返回false，说明beanName已经在当前列表中了，此时会抛循环依赖的异常BeanCurrentlyInCreationException，这个异常对应的源码：  

```java
public BeanCurrentlyInCreationException(String beanName) {
    super(beanName,
            "Requested bean is currently in creation: Is there an unresolvable circular reference?");
}
```

----
&emsp; 上面是单例bean检测循环依赖的源码，再来看看非单例bean的情况。例如prototype原型，源码位于org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean方法中，主要源码如下：  

```java
//检查正在创建的bean列表中是否存在beanName，如果存在，说明存在循环依赖，抛出循环依赖的异常
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}

//判断scope是否是prototype
if (mbd.isPrototype()) {
    Object prototypeInstance = null;
    try {
        //将beanName放入正在创建的列表中
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        //将beanName从正在创建的列表中移除
        afterPrototypeCreation(beanName);
    }
}
```
&emsp; field属性注入的原型Bean发生循环依赖，会抛出异常BeanCurrentlyInCreationException。  
&emsp; 同样，构造器注入的Bean循环依赖，也会抛出异常BeanCurrentlyInCreationException。  
&emsp; 只有含有field属性注入的单例Bean，通过3级缓存解决循环依赖。  

## 1.3. Spring如何解决循环依赖的问题?  
<!-- 
https://blog.csdn.net/lkforce/article/details/97183065
https://www.cnblogs.com/leeego-123/p/12165278.html
-->

&emsp; **<font color = "red">field属性注入的单例Bean循环依赖，Spring通过3级缓存解决。</font>**  

&emsp; 在Spring Bean的生命周期中，创建单例bean时首先会从缓存中获取这个单例的bean。  

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;

    // 方法1）从三个map中获取单例类
    Object sharedInstance = getSingleton(beanName);
    // ......
}
    else {
    // 如果是多例的循环引用，则直接报错
    if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
    // ......
    try {
        // Create bean instance.
        if (mbd.isSingleton()) {
            // 方法2) 获取单例对象
            sharedInstance = getSingleton(beanName, () -> {
                try { //方法3) 创建ObjectFactory中getObject方法的返回值
                    return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
                    // Explicitly remove instance from singleton cache: It might have been put there
                    // eagerly by the creation process, to allow for circular reference resolution.
                    // Also remove any beans that received a temporary reference to the bean.
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
    }
    // ......
    return (T) bean;
}

```

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```
&emsp; 这个方法是Spring解决循环依赖的关键方法，在这个方法中，使用了三级缓存来查询的方式：  

* 第一级缓存singletonObjects里面放置的是实例化好的单例对象。  
* 第二级earlySingletonObjects里面存放的是提前曝光的单例对象，尚未属性装配的Bean 。  
* 第三级singletonFactories里面存放的是要被实例化的对象的对象工厂。  

&emsp; 这个方法中用到的几个判断逻辑，体现了Spring解决循环依赖的思路，不过实际上对象被放入这三层的顺序是和方法查询的循序相反的，也就是说，在循环依赖出现时，对象往往会先进入singletonFactories，然后earlySingletonObjects，然后singletonObjects。  

<!-- 
&emsp; 代码解析：    
......
-->
&emsp; Spring处理循环依赖的流程（假设对象A和对象B循环依赖）：  

|步骤	|操作|	三级缓存中的内容|
|---|---|---|
|1|	开始初始化对象A	|singletonFactories：<br/>earlySingletonObjects：<br/>singletonObjects：|
|2|	调用A的构造，把A放入singletonFactories|	singletonFactories：A<br/>earlySingletonObjects：<br/>singletonObjects：|
|3|	开始注入A的依赖，发现A依赖对象B|singletonFactories：A<br/>earlySingletonObjects：<br/>singletonObjects：|
|4|	开始初始化对象B|singletonFactories：A,B<br/>earlySingletonObjects：<br/>singletonObjects：|
|5|	调用B的构造，把B放入singletonFactories|	singletonFactories：A,B<br/>earlySingletonObjects：<br/>singletonObjects：|
|6|	开始注入B的依赖，发现B依赖对象A	|singletonFactories：A,B<br/>earlySingletonObjects：<br/>singletonObjects：|
|7|开始初始化对象A，发现A在singletonFactories里有，则直接获取A，把A放入earlySingletonObjects，把A从singletonFactories删除|singletonFactories：B<br/>earlySingletonObjects：A<br/>singletonObjects：|
|8|	对象B的依赖注入完成|singletonFactories：B<br/>earlySingletonObjects：A<br/>singletonObjects：|
|9|对象B创建完成，把B放入singletonObjects，把B从earlySingletonObjects和singletonFactories中删除|singletonFactories：<br/>earlySingletonObjects：A<br/>singletonObjects：B|
|10|对象B注入给A，继续注入A的其他依赖，直到A注入完成|singletonFactories：<br/>earlySingletonObjects：A<br/>singletonObjects：B|
|11|对象A创建完成，把A放入singletonObjects，把A从earlySingletonObjects和singletonFactories中删除|singletonFactories：<br/>earlySingletonObjects：<br/>singletonObjects：A,B|
|12|循环依赖处理结束，A和B都初始化和注入完成|singletonFactories：<br/>earlySingletonObjects：<br/>singletonObjects：A,B|


## 1.4. 常见问题  
1. 为什么使用构造器注入属性的时候不能解决循环依赖问题？  
&emsp; 原因在于此方式是实例化和初始化分开（在进行实例化的时候不必给属性赋值）操作，将提前实例化好的对象前景暴露出去，供别人调用，而使用构造器的时候，必须要调用构造方法了，没有构造方法无法完成对象的实例化操作，也就无法创建对象的实例化操作，也就无法创建对象，那么永远会陷入到死循环中。   

2. 一级缓存能不能解决此问题？  
&emsp; 不能，在三个级别的缓存中，放置的对象是有区别的，（1、是完成实例化且初始化的对象，2、是实例化但是未初始化的对象），如果只有一级缓存，就有可能取到实例化但未初始化的对象，属性值都为空，肯定有问题。  

3. 二级缓存能不能解决？为什么要三级缓存？  
<!-- Spring循环依赖三级缓存是否可以减少为二级缓存？ 
https://mp.weixin.qq.com/s/3ny7oIE89c7mztV0WSAfbA-->
&emsp; 理论上来说是可以解决循环依赖的问题，但是注意：为什么要在三级缓存中放置匿名内部类？  
&emsp; 本质在于为了创建代理对象，假如现在有A类，需要生成代理对象，A需要实例化。  
&emsp; 在三级缓存中放置的是，生成具体对象的一个匿名内部类，这个匿名内部类可能是代理类，也可能是普通的实例对象，而使用三级缓存就保证了不管是否需要代理对象，都保证使用的是一个对象，而不会出现，前面使用普通bean，后面使用代理类。
