## Registry

---

![](/assets/屏幕快照 2018-10-21 下午5.21.32.png)**AliasRegistry**

对于每个Bean，除了beanName，还可以为它起一个别名，即alias。AliasRegistry定义了针对alias的简单增删改等操作，即对一个name的alise。

**SimpleAliasRegistry**

对AliasRegistry的实现，使用map实现：`Map<String, String> aliasMap`，key为beanName,，value为beanName的别名。

**BeanDefinitionRegistry**

定义对BeanDefinition的增删改查操作，因为继承自AliasRegistry，自然也包含了对Bean的别名的操作。

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    // 注册一个新的BeanDefinition实例 
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)throws BeanDefinitionStoreException;

    // 移除注册表中已注册的 BeanDefinition 实例
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 从注册中取得指定的 BeanDefinition 实例
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 判断 BeanDefinition 实例是否在注册表中（是否注册）
    boolean containsBeanDefinition(String beanName);

    // 取得注册表中所有 BeanDefinition 实例的 beanName（标识）
    String[] getBeanDefinitionNames();

    // 返回注册表中 BeanDefinition 实例的数量
    int getBeanDefinitionCount();

    // beanName（标识）是否被占用
    boolean isBeanNameInUse(String beanName);
}
```

**SimpleBeanDefinitionRegistry**

对BeanDefinitionRegistry 的简单实现，主要用来对各种Reader做测试用。

使用map实现：`Map<String, BeanDefinition> beanDefinitionMap`，key为beanName,，value为BeanDefinition。

DefaultListableBeanFactory中直接实现了BeanDefinitionRegistry，而没有继承SimpleBeanDefinitionRegistry。

**SingletonBeanRegistry**

对于单例Bean，Spring单独提供了接口来定义其操作，来分担BeanFactory的一部分职责，即SingletonBeanRegistry

```java
public interface SingletonBeanRegistry {

    //在容器内注册一个单例类  
    void registerSingleton(String beanName, Object singletonObject); 

    //返回给定名称对应的单例类
    Object getSingleton(String beanName);

    //给定名称是否对应单例类
    boolean containsSingleton(String beanName);

    //返回容器内所有单例类的名字
    String[] getSingletonNames();

    //返回容器内注册的单例类数量
    int getSingletonCount();
}
```

**DefaultSingletonBeanRegistry**

对SingletonBeanRegistry的默认实现。

使用Map实现：`Map<String, Object> singletonObjects`，key为beanName,，value为Bean。

**FactoryBeanRegistrySupport**

在DefaultSingletonBeanRegistry的基础上，增加了对BeanFactory的特殊处理功能。

Register，英文解释为an official list or record，一般翻译为注册中心或者登记处等，它来负责对一些东西进行记录，并且可以查看该记录。而想要登记或者记录一些东西，我们都会有一个index的东西，比如一般公司前台会对来访人员做一个来访记录，记下来访的人的姓名，来访的时间，来访的目的等等，在这个环境中，这个来访的人就是index，或者叫做key，其他信息则是value。因此，无论是哪种Register，都会有一个index或者key，一个value，就是Map&lt;key, value&gt;这种数据结构。

上述的几个Register，其中AliasRegistry是用来记录一个beanName有哪些别名，BeanDefinitionRegistry是用来记录一个beanName 的BeanDefinition，SingletonBeanRegistry则是用来记录一个beanName的Bean。这几个Register的有一个共同点：它们的key都是beanName，或者说beanName是这个几个Register的联系枢纽。

## 参考

---

[SingletonBeanRegistry](https://blog.csdn.net/weixin_39165515/article/details/77655567)

[Spring Bean 大体架构介绍](https://gavinzhang1.gitbooks.io/spring/content/spring_bean_da_ti_jia_gou_jie_shao.html)

