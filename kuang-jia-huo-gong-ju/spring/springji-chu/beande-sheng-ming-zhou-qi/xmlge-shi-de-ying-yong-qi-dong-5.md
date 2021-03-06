第六步：registerBeanPostProcessors\(beanFactory\)，注册BeanPostProcessor。

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

第五步中，说PostProcessorRegistrationDelegate主要有两个功能，第一个功能是回调所有的BeanFactoryPostProcessor，第二个功能就是这一步中的注册BeanPostProcessor。主要逻辑如下：

①先注册实现了PriorityOrdered接口的BeanPostProcessor，再注册实现了Ordered接口的BeanPostProcessor，最后注册既没有实现PriorityOrdered也没有实现Ordered的BeanPostProcessor；

②对于MergedBeanDefinitionPostProcessor这种特殊的类型，放在最后注册。

这里说的注册，其实是添加到beanFactory中如下变量（AbstractBeanFactory）中：

```java
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();
```

第七步：initMessageSource\(\)，初始化国际化资源，TODO

第八步：initApplicationEventMulticaster\(\)，初始化应用广播器：如果自定义了ApplicationEventMulticaster，则使用这个自定义的，如果没有，则默认使用SimpleApplicationEventMulticaster。

第九步：onRefresh\(\)，留给子类实现，去实例化一些特殊的Bean，具体到ClassPathXmlApplicationContext，它没有实现该方法。

第十步：registerListeners\(\)，注册应用监听器：将context中手动设置的监听器以及从beanFactory中拿到的监听器都传给广播器，并发布早期事件。



 

