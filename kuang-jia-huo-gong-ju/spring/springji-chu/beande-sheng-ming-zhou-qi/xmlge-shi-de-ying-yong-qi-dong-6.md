第十一步：finishBeanFactoryInitialization\(beanFactory\)，实例化非懒加载的单例Bean。

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 为当前context设置ConversionService
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 设置占位符解析器，主要是为了解析注解中出现的形如${...}的字符，将其替换为具体的值
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 实例化LoadTimeWeaverAware
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    beanFactory.setTempClassLoader(null);

    // 冻结所有的bean definitions，不允许之后再对BeanDefinition进行修改
    beanFactory.freezeConfiguration();

    // 实例化所有的非懒加载的单例Bean
    beanFactory.preInstantiateSingletons();
}
```

 preInstantiateSingletons的具体实现在DefaultListableBeanFactory中：

```java
public void preInstantiateSingletons() throws BeansException {
    //这里对beanDefinitionNames做了一次copy
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
    for (String beanName : beanNames) {
        //先拿到最终的BeanDefinition
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        //如果Bean非abstract，且为单例，并非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            //如果bean实现了FactoryBean接口
            if (isFactoryBean(beanName)) {
                //先通过前缀“&” + beanName作为beanName去获取FactoryBean
                //为了判断该Bean到底是不是非懒加载的
                //LazyInit与EagerInit可以看做一对反义词
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

接下来，最重要的就是getBean的过程了。

