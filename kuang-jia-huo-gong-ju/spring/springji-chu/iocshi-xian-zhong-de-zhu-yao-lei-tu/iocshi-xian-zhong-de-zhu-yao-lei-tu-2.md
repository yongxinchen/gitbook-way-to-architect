## Spring的BeanDefinition

---

![](/assets/屏幕快照 2018-11-10 下午3.56.25.png)

**AttributeAccessor**

AttributeAccessor接口定义了对任意对象的元数据的修改或者获取。

```java
public interface AttributeAccessor {

    void setAttribute(String name, Object value);

    Object getAttribute(String name);

    Object removeAttribute(String name);

    boolean hasAttribute(String name);

    String[] attributeNames();
}
```

> 疑问：AttributeAccessor与PropertyAccessor的区别是什么？

**AttributeAccessorSupport**

 对AttributeAccessor的默认实现，其实就是对一个Map&lt;String, Object&gt; attributes的操作。

**BeanMetadataElement**

定义Bean的源数据，只有一个方法：`Object getSource()`，这个object为配置源对象，即AttributeAccessor要操纵的对象是谁。

**BeanMetadataAttribute**

对某个源数据的&lt;属性名, 属性值&gt;的封装，三个字段：属性名\(String name\)，属性值\(Object value\)，源数据\(Object source\)，相当于冗余了这个源数据。

```java
private final String name;

private final Object value;

private Object source;
```

**BeanMetadataAttributeAccessor**

对AttributeAccessorSupport进行了扩展：增加了addMetadataAttribute和getMetadataAttribute的方法。

**PropertyValue**

 表示某个Bean的某个属性的信息。一个Bean可能会有很多属性，每一个属性的信息都会用PropertyValue来表示：

```java
//属性名
private final String name;
//属性值
private final Object value;
//是否是可选的，如果是可选的，则表示Bean的这个属性可以不存在
private boolean optional = false;
//属性值是否已经被转换为最终的值
private boolean converted = false;
//最终的已被转换的属性值
private Object convertedValue;
//属性值是否需要被转换
volatile Boolean conversionNecessary;
```

由于PropertyValue继承自BeanMetadataAttributeAccessor，所以，PropertyValue也持有源数据，所以PropertyValue实际上维护了一个这样的信息：源对象，源对象的某个属性，源对象的这个属性的值，并且可以将这个属性值进行最终的确定。

**PropertyValues**

PropertyValue的集合，它定义了如下方法：

```java
public interface PropertyValues {

    PropertyValue[] getPropertyValues();

    PropertyValue getPropertyValue(String propertyName);

    PropertyValues changesSince(PropertyValues old);

    boolean contains(String propertyName);

    boolean isEmpty();
}
```

**MutablePropertyValues**

 PropertyValues的实现，内部使用ArrayList来表示PropertyValue集合。

**BeanDefinition**

定义实例化Bean时需要的所有信息，这些Bean的信息包括：

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    //表示Bean的Scope为单例：singleton
    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
    //表示Bean的Scope为原型：prototype
    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

    //表示Bean是应用的组成部分，一般是指用户定义的Bean
    int ROLE_APPLICATION = 0;
    //表示Bean是作为一些大型配置的支持类，比如ComponentDefinition
    int ROLE_SUPPORT = 1;
    //表示Bean是Spring内部用到的Bean，与用户无关
    int ROLE_INFRASTRUCTURE = 2;

    //设置该BeanDefinition的父BeanDefinition，子BeanDefinition会继承父BeanDefinition的配置
    void setParentName(@Nullable String parentName);
    String getParentName();

    //设置Bean的全类名，一般就是我们在定义Bean时写的class="com.maxwell.example.UserServiceImpl"这个字符串
    void setBeanClassName(@Nullable String beanClassName);
    String getBeanClassName();

    //设置Bean的作用域
    void setScope(@Nullable String scope);
    String getScope();

    //设置Bean是否是懒加载的，仅仅对单例Bean有效
    void setLazyInit(boolean lazyInit);
    boolean isLazyInit();

    //设置Bean在实例化时所依赖的其他的Bean的名字，Bean工厂会保证先实例化这些依赖的Bean
    void setDependsOn(@Nullable String... dependsOn);
    String[] getDependsOn();

    //设置Bean是否可以用来在其他Bean中自动注入，仅仅对根据类型自动注入的情况有效，不会影响根据名字自动注入的情况。
    //比如，为True时，当其他Bean中通过@Autowired进行注入时，如果类型匹配（即Class一样），就会把这个Bean当做候选者
    void setAutowireCandidate(boolean autowireCandidate);
    boolean isAutowireCandidate();

    //设置Bean是否为主要的自动注入的候选者
    //当一个Bean要按类型注入另外的Bean时，可能存在多个类型符合的Bean，那么就必须从中选出一个，选哪个，就以此标记为准
    void setPrimary(boolean primary);
    boolean isPrimary();

    //设置Bean的FactoryBean
    void setFactoryBeanName(@Nullable String factoryBeanName);
    String getFactoryBeanName();
    //如果Bean有FactoryBean，设置Bean的FactoryMethod
    void setFactoryMethodName(@Nullable String factoryMethodName);
    String getFactoryMethodName();

    //获取Bean的构造器中的参数值
    ConstructorArgumentValues getConstructorArgumentValues();
    default boolean hasConstructorArgumentValues() {
        return !getConstructorArgumentValues().isEmpty();
    }

    //Bean的PropertyValue集合
    MutablePropertyValues getPropertyValues();
    default boolean hasPropertyValues() {
        return !getPropertyValues().isEmpty();
    }

    //是否为单例Scope
    boolean isSingleton();
    //是否为原型Scope
    boolean isPrototype();
    //是否是abstract，如果是，则不进行实例化
    boolean isAbstract();

    //Bean的角色
    int getRole();

    //获取该BeanDefinition来源的描述
    String getResourceDescription();

    //获取原始的BeanDefinition
    BeanDefinition getOriginatingBeanDefinition();
}
```

**AbstractBeanDefinition**

 BeanDefinition的通用实现。

**GenericBeanDefinition**

GenericBeanDefinition是一站式的标准bean definition，除了具有指定类、可选的构造参数值和属性参数这些其它bean definition一样的特性外，它还具有通过parenetName属性来灵活设置parent bean definition。 

**AnnotatedBeanDefinition**

该接口扩展了  BeanDefinition，增加了获取BeanDefinition中对应的类的注解信息的方法：

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {

    AnnotationMetadata getMetadata();

    MethodMetadata getFactoryMethodMetadata();
}
```

其中， AnnotationMetadatam描述了一个类的注解信息，比如这个类被哪些注解所注解，这个类有哪些被注解的方法等等。

AnnotatedBeanDefinition的实现类则表示那些通过注解来获取的BeanDefinition：

以@Configuration注解标记的会解析为AnnotatedGenericBeanDefinition

以@Bean注解标记的会解析为ConfigurationClassBeanDefinition

以@Component（包括@Service\@Controller\@Repository）注解标记的会解析为ScannedGenericBeanDefinition

这些BeanDefinition不是通过对Xml的解析获取的，而是通过解析类中的注解来获取，由Xml定义的Bean会解析为GenericBeanDefinition。

 

## 参考

---

[ spring笔记-BeanDefinition](https://www.jianshu.com/p/a6a03d94d6f7)

[Spring4源码解析：BeanDefinition架构及实现](https://www.cnblogs.com/ninth/p/6404317.html)

[死磕Spring系列之四 BeanDefinition接口、BeanFactory接口](http://blog.51cto.com/dba10g/1728512)

[Spring 在 IOC 容器中 Bean 之间的关系](https://juejin.im/entry/58b39fab1b69e60058b45767)

 

