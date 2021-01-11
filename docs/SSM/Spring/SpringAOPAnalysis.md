
<!-- TOC -->

- [1. SpringAOP解析](#1-springaop解析)
    - [1.1. JDK动态代理和CGLIB动态代理](#11-jdk动态代理和cglib动态代理)
    - [1.2. 开启AOP自动代理解析](#12-开启aop自动代理解析)
    - [1.3. 自动代理的触发时机](#13-自动代理的触发时机)
    - [1.4. 代理类的生成流程](#14-代理类的生成流程)
        - [1.4.1. 获取对应Bean适配的Advisors链](#141-获取对应bean适配的advisors链)
            - [1.4.1.1. 获取候选的advisors](#1411-获取候选的advisors)
            - [1.4.1.2. 筛选出适配当前类的 Advisors](#1412-筛选出适配当前类的-advisors)
            - [1.4.1.3. 小结](#1413-小结)
        - [1.4.2. 创建代理类](#142-创建代理类)
            - [1.4.2.1. 创建 AopProxy](#1421-创建-aopproxy)
            - [1.4.2.2. 创建代理类](#1422-创建代理类)
                - [1.4.2.2.1. JDK动态代理](#14221-jdk动态代理)
                - [1.4.2.2.2. Cglib代理](#14222-cglib代理)

<!-- /TOC -->

# 1. SpringAOP解析

<!-- 

6.7AOP 有哪些实现方式？

实现 AOP 的技术， 主要分为两大类：
静态代理

指使用 AOP 框架提供的命令进行编译，从而在编译阶段就可生成 AOP 代理类， 因此也称为编译时增强；

编译时编织（特殊编译器实现）
类加载时编织（特殊的类加载器实现）。

动态代理

在运行时在内存中“ 临时” 生成 AOP 动态代理类， 因此也被称为运行时增强。

JDK 动态代理
CGLIB
-->

## 1.1. JDK动态代理和CGLIB动态代理  
&emsp; 常用的代理有通过接口的[JDK动态代理](/docs/java/Design/6.proxy.md)和通过继承类的CGLIB动态代理。  
1. JDK动态代理  
&emsp; <font color = "red">利用拦截器(拦截器必须实现InvocationHanlder)加上反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。</font>  

2. CGLIB动态代理  
&emsp; <font color = "red">利用ASM开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。</font>  

3. 何时使用JDK还是CGLIB？  
    1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP。  
    2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP。  
    3. 如果目标对象没有实现了接口，必须采用CGLIB库，Spring会自动在JDK动态代理和CGLIB之间转换。  

4. 如何强制使用CGLIB实现AOP？  
    1. 添加CGLIB库(aspectjrt-xxx.jar、aspectjweaver-xxx.jar、cglib-nodep-xxx.jar)  
    2. 在Spring配置文件中加入<aop:aspectj-autoproxy proxy-target-class="true"/>  

5. JDK动态代理和CGLIB字节码生成的区别？   
    1. JDK动态代理只能对实现了接口的类生成代理，而不能针对类。  
    2. CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，并覆盖其中方法实现增强，但是因为采用的是继承，所以该类或方法最好不要声明成final，对于final类或方法，是无法继承的。  

6. CGlib比JDK快？  
    1. 使用CGLib实现动态代理，<font color = "red">CGLib底层采用ASM字节码生成框架，使用字节码技术生成代理类</font>，在jdk6之前比使用Java反射效率要高。唯一需要注意的是，CGLib不能对声明为final的方法进行代理，因为CGLib原理是动态生成被代理类的子类。  
    2. 在jdk6、jdk7、jdk8逐步对JDK动态代理优化之后，在调用次数较少的情况下，JDK代理效率高于CGLIB代理效率，只有当进行大量调用的时候，jdk6和jdk7比CGLIB代理效率低一点，但是到jdk8的时候，jdk代理效率高于CGLIB代理，总之，每一次jdk版本升级，jdk代理效率都得到提升，而CGLIB代理消息确有点跟不上步伐。  

7. Spring如何选择用JDK还是CGLIB？  
    1. 当Bean实现接口时，Spring就会用JDK的动态代理。  
    2. 当Bean没有实现接口时，Spring使用CGlib是实现。  
    3. 可以强制使用CGlib（在spring配置中加入<aop:aspectj-autoproxy proxy-target-class="true"/>）。   

----
----
----

![image](https://gitee.com/wt1814/pic-host/raw/master/images/SSM/AOP/aop-7.png)  

<!-- &emsp; Spring AOP的功能是什么？根据配置来生成代理类，拦截指定的方法，将指定的advice织入。  -->
1. <font color = "red">Spring AOP 的触发时机是什么时候？</font>  
2. <font color = "red">Spring AOP 是如何解析配置的Aspect，生成 Advisors 链的？</font>  
3. <font color = "red">Spring AOP 是如何生成代理类的，如何将 advice 织入代理类？</font>  

## 1.2. 开启AOP自动代理解析  
&emsp; 可以使用@EnableAspectJAutoProxy注解开启Spring AOP注解的使用。自动让 ioc 容器中的所有 advisor 来匹配方法，advisor 内部都是有 advice 的，让它们内部的 advice 来执行拦截处理。  

1. @EnableAspectJAutoProxy代码：  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```
&emsp; @Import(AspectJAutoProxyRegistrar.class)使用@Import 注解将 AspectJAutoProxyRegistrar注入到 IoC 容器当中。  

2. AspectJAutoProxyRegistrar.java代码  

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    AspectJAutoProxyRegistrar() {
    }

    //注册AspectJAnnotationAutoProxyCreator的BeanDefinition
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        //根据@EnableAspectJAutoProxy注解的属性值配置proxyTargetClass和exposeProxy
        //把属性值注入到AspectJAnnotationAutoProxyCreator的BeanDefinition
        if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }

        if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }

    }
}
```

&emsp; AopConfigUtils详解：  

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
    return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, (Object)null);
}

public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

```java
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
        BeanDefinition apcDefinition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }

        return null;
    } else {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
        beanDefinition.setSource(source);
        //设置高优先级
        beanDefinition.getPropertyValues().add("order", -2147483648);
        beanDefinition.setRole(2);
        //向IOC容器注册 AspectJAnnotationAutoProxyCreator的beanDefinition
        registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
        return beanDefinition;
    }
}
```
&emsp; 在AspectJAutoProxyRegistrar中，实际上就是将AspectJAnnotationAutoProxyCreator的BeanDefinition注册到IoC容器当中。  

## 1.3. 自动代理的触发时机  
&emsp; 首先看一下AspectJAnnotationAutoProxyCreator的继承体系。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SSM/AOP/AOP-5.png)  
&emsp; AspectJAnnotationAutoProxyCreator继承了BeanPostProcessor 。  
&emsp; BeanPostProcessor源码：  

```java
public interface BeanPostProcessor {

    //为在 Bean 的初始化前提供回调入口
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    //为在 Bean 的初始化之后提供回调入口
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

&emsp; 回顾Spring创建Bean的流程：  
1. Step1 创建实例对象createBeanInstance()；
2. Step2 属性装配populateBean()，得到一个真正的实现类；
3. <font color = "lime">在Step3 initializeBean()中，IoC容器会处理Bean初始化之后的各种回调事件</font>，然后返回一个“可能经过加工”的bean对象。其中就包括了BeanPostProcessor 的 postProcessBeforeInitialization 回调 和postProcessAfterInitialization 回调。  

&emsp; **而<font color = "lime">AspectJAnnotationAutoProxyCreator是一个BeanPostProcessor，</font>** 因此Spring AOP是在这一步，进行代理增强！

## 1.4. 代理类的生成流程  
&emsp; 在开启AOP自动代理解析阶段中的AnnotationAwareAspectJAutoProxyCreator是一种具体的创建创建AOP代理对象的子类。  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/SSM/AOP/AOP-6.png)  
&emsp; 可以看到实际回调的postProcessBeforeInitialization和postProcessAfterInitialization这两个方式是在AbstractAdvisorAutoProxyCreator中override的。  

&emsp; AbstractAdvisorAutoProxyCreator继承了AbstractAutoProxyCreator。  

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean != null) {
            Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                return this.wrapIfNecessary(bean, beanName, cacheKey);
            }
        }

        return bean;
    }

}
```

&emsp; JavaDoc 很清楚的注明了postProcessAfterInitialization会执行创建代理类的操作，用配置的interceptors 来创建一个代理类。

&emsp; AbstractAutoProxyCreator#wrapIfNecessary(..)  

```java
//返回代理类或者实现类本身
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    } else if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }

    //判断是不是基础的Bean(Advice、PointCut、Advisor、AopInfrastructureBean)，是就直接跳过
    //判断是不是应该跳过（AOP解析直接解析出切面信息（并且把切面信息进行缓存））
    else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) {
        //TODO 核心方法
        //返回当前bean的所有的advisor、advice、interceptor
        Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);

            //TODO 核心方法
            //创建代理
            Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        } else {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    } else {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
}
```
&emsp; 这个方法里有两个核心点：  
1. 获取当前的Spring Bean 适配的 advisors  
2. 创建代理类  

### 1.4.1. 获取对应Bean适配的Advisors链  

&emsp; 获取对应 Bean 适配的 Advisors 链，分为两步。  
1. 获取容器所有的 advisors 作为候选，即解析Spring 容器中所有 Aspect 类中的 advice 方法，包装成 advisor；  
2. 从候选的 Advisors 中筛选出适配当前 Bean的 Advisors 链； 

#### 1.4.1.1. 获取候选的advisors  
&emsp; AspectJAwareAdvisorAutoProxyCreator#shouldSkip(..)  

```java
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    //寻找候选的advisors
    List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
    Iterator var4 = candidateAdvisors.iterator();

    Advisor advisor;
    do {
        if (!var4.hasNext()) {
            return super.shouldSkip(beanClass, beanName);
        }

        advisor = (Advisor)var4.next();
    } while(!(advisor instanceof AspectJPointcutAdvisor) || !((AbstractAspectJAdvice)advisor.getAdvice()).getAspectName().equals(beanName));

    return true;
}
```

&emsp; AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean(..)-->AbstractAdvisorAutoProxyCreator#findEligibleAdvisors(..)  

```java
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
    List<Advisor> advisors = this.findEligibleAdvisors(beanClass, beanName);
    return advisors.isEmpty() ? DO_NOT_PROXY : advisors.toArray();
}
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //获取候选的advisors
    List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    this.extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = this.sortAdvisors(eligibleAdvisors);
    }

    return eligibleAdvisors;
}
```
&emsp; 看到两个方法都调用了findCandidateAdvisors()方法，也就是去获取候选的Advisors。  

#### 1.4.1.2. 筛选出适配当前类的 Advisors  
&emsp; AbstractAdvisorAutoProxyCreator#findEligibleAdvisors(..)  

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //获取候选的advisors
    List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
    //刷选出适配当前类的advisors
    List<Advisor> eligibleAdvisors = this.findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    this.extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        //排序
        eligibleAdvisors = this.sortAdvisors(eligibleAdvisors);
    }

    return eligibleAdvisors;
}
```
&emsp; AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply(..)  

```java
protected List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
    ProxyCreationContext.setCurrentProxiedBeanName(beanName);

    List var4;
    try {
        var4 = AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
    } finally {
        ProxyCreationContext.setCurrentProxiedBeanName((String)null);
    }

    return var4;
}
```
&emsp; AopUtils#findAdvisorsThatCanApply(..)  

```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
    if (candidateAdvisors.isEmpty()) {
        return candidateAdvisors;
    } else {
        List<Advisor> eligibleAdvisors = new LinkedList();
        Iterator var3 = candidateAdvisors.iterator();

        while(var3.hasNext()) {
            //获取IntroductionAdvisor类型的Advisor
            //处理IntroductionAdvisor类型的Advisor和普通的Advisor不一样，所以先单独处理
            Advisor candidate = (Advisor)var3.next();
            if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
                eligibleAdvisors.add(candidate);
            }
        }

        boolean hasIntroductions = !eligibleAdvisors.isEmpty();
        Iterator var7 = candidateAdvisors.iterator();

        while(var7.hasNext()) {
            Advisor candidate = (Advisor)var7.next();
            //TODO 筛选的关键方法
            //canApply(candidate, clazz, hasIntroductions)
            if (!(candidate instanceof IntroductionAdvisor) && canApply(candidate, clazz, hasIntroductions)) {
                eligibleAdvisors.add(candidate);
            }
        }

        return eligibleAdvisors;
    }
}
```
&emsp; AopUtils#canApply(..)  

```java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    if (advisor instanceof IntroductionAdvisor) {
        return ((IntroductionAdvisor)advisor).getClassFilter().matches(targetClass);
    } else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor)advisor;
        //判断这个advisor的pointcut是否可以应用在类中的某个方法上
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    } else {
        //它没有切入点，所以假设它适用
        return true;
    }
}

public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    //使用classfilter 来匹配class
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    } else {
        MethodMatcher methodMatcher = pc.getMethodMatcher();
        if (methodMatcher == MethodMatcher.TRUE) {
            //如果匹配任意方法的话，就不需要迭代了
            return true;
        } else {
            IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
            if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
                introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher)methodMatcher;
            }

            //查找当前类及其祖宗类实现的所有接口
            Set<Class<?>> classes = new LinkedHashSet(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
            classes.add(targetClass);
            Iterator var6 = classes.iterator();

            while(var6.hasNext()) {
                Class<?> clazz = (Class)var6.next();
                //获取当前类的方法列表，包括从父类中继承的方法
                Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
                Method[] var9 = methods;
                int var10 = methods.length;

                for(int var11 = 0; var11 < var10; ++var11) {
                    Method method = var9[var11];
                    //使用MethodMatcher匹配方法，匹配成功即可立即返回
                    if (introductionAwareMethodMatcher != null && introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) || methodMatcher.matches(method, targetClass)) {
                        return true;
                    }
                }
            }

            return false;
        }
    }
}
```
&emsp; 筛选的工作主要由 ClassFilter 和 MethodMatcher 完成，比如AspectJExpressionPointcut的实现了ClassFilter和MethodMatcher接口，最终由AspectJ表达式解析  

#### 1.4.1.3. 小结  
&emsp; Spring AOP获取对应 Bean 适配的Advisors 链的核心逻辑：
1. 获取当前 IoC 容器中所有的 Aspect 类
2. 给 每个Aspect 类的advice 方法创建一个 Spring Advisor，这一步又能细分为 
    1. 遍历所有advice 方法
    2. 解析方法的注解和pointcut
    3. 实例化 Advisor 对象
3. 获取到 候选的 Advisors，并且缓存起来，方便下一次直接获取
4. 从候选的 Advisors 中筛选出与目标类 适配的Advisor 
    1. 获取到 Advisor 的 切入点 pointcut
    2. 获取到 当前 target 类 所有的 public 方法
    3. 遍历方法，通过 切入点 的 methodMatcher 匹配当前方法，只有有一个匹配成功就相当于当前的Advisor 适配
5. 对筛选之后的 Advisor 链进行排序

### 1.4.2. 创建代理类  
&emsp; AbstractAutoProxyCreator#createProxy  

```java
protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
    //如果bean工厂为ConfigurableListableBeanFactory
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory)this.beanFactory, beanName, beanClass);
    }
    //创建ProxyFactory
    ProxyFactory proxyFactory = new ProxyFactory();
    //复制配置
    proxyFactory.copyFrom(this);
    //这个对应的是 proxyTargetClass = true 这个配置
    //ture的时候，则不管有没有接口都是用CGLB来生成代理类
    if (!proxyFactory.isProxyTargetClass()) {
        //判断一下根据接口还是类来生成代理类
        if (this.shouldProxyTargetClass(beanClass, beanName)) {
            //根据类生成代理的话，就将这个属性设置成true
            proxyFactory.setProxyTargetClass(true);
        } else {
            //根据接口生成代理的话，需要在这里做一些操作
            //把这个类对应的接口都放到ProxyFactory里面
            //proxyFactory.addInterface(ifc)
            this.evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    //这个方法决定了给出的类所有的advisors数组
    //specificInterceptors中若包含又Advice，此处将Advice转为Advisor
    Advisor[] advisors = this.buildAdvisors(beanName, specificInterceptors);
    //加入增强器
    proxyFactory.addAdvisors(advisors);
    //设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    //定制代理
    this.customizeProxyFactory(proxyFactory);
    //用来控制代理过程被配置之后，是否还允许修改通知。
    //默认值为false（即在代理被配置之后，不允许修改代理的配置）
    proxyFactory.setFrozen(this.freezeProxy);
    //是否跳过ClassFilter检查
    if (this.advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    //TODO
    //根据工厂的各种配置，生成一个代理类
    return proxyFactory.getProxy(this.getProxyClassLoader());
}
```
&emsp; 流程：  
1. 获取当前类中的属性
2. 添加代理接口
3. 封装Advisor并加入到ProxyFactory中
4. 设置要代理的类
5. Spring中为子类提供了定制的函数customizeProxyFactory，子类可以在此函数中6. 对ProxyFactory的进一步封装
7. ★★★ **获取代理操作**

```java
ProxyFactory#getProxy(..)
public Object getProxy(ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```
&emsp; 这里要分为两步：
1. 创建AopProxy
2. 获取代理类

#### 1.4.2.1. 创建 AopProxy  
&emsp; ProxyCreatorSupport#AopProxy()  

```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}
```
&emsp; DefaultAopProxyFactory#createAopProxy()  

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    //判断是使用JDK动态代理还是CGLB代理
    //optimize：用来控制通过CGLB创建的代理是否使用激进的优化策略。仅用于CGLB
    //proxyTargetClass：这个属性为true时，目标类本身代理而不是目标类的接口
    //如果这个属性值被设为true，CGLB代理被创建
    //设置方式：<aop: aspectj-autoproxy proxy-target-class = "true"/>;
    //hasNoUserSuppliedProxyInterfaces：是否存在代理接口；
    if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
        return new JdkDynamicAopProxy(config);
    } else {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
        } else {
            //isInterface目标类实现了接口
            //这个class是否已经JDK的代理类类
            //isProxyClass：当且仅当使用getProxyClass方法或newProxyInstance方法将指定的类动态生成为代理类时，返回true。
            return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
        }
    }
}
```
&emsp; 这一步之后根据ProxyConfig 获取到了对应的AopProxy的实现类，分别是JdkDynamicAopProxy和ObjenesisCglibAopProxy。  

&emsp; JdkDynamicAopProxy  

```java
public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
    Assert.notNull(config, "AdvisedSupport must not be null");
    if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
        throw new AopConfigException("No advisors and no TargetSource specified");
    } else {
        this.advised = config;
    }
}
```

&emsp; CglibAopProxy  

```java
public CglibAopProxy(AdvisedSupport config) throws AopConfigException {
    Assert.notNull(config, "AdvisedSupport must not be null");
    if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
        throw new AopConfigException("No advisors and no TargetSource specified");
    } else {
        this.advised = config;
        this.advisedDispatcher = new CglibAopProxy.AdvisedDispatcher(this.advised);
    }
}
```

#### 1.4.2.2. 创建代理类  
##### 1.4.2.2.1. JDK动态代理  
&emsp; 源码位置：JdkDynamicAopProxy#getProxy(..)  



##### 1.4.2.2.2. Cglib代理
......



