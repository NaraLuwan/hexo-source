---
title: spring启动加载过程源码分析
date: 2017-07-31 12:11:00
tags: Spring
categories: 
- Java
- Web
---
我们知道启动spring容器两常见的两种方式（其实都是加载spring容器的xml配置文件时启动的）：
- 在应用程序下加载
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("spring-context.xml");
```
- web模式下加载
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring-context.xml</param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
可以发现，执行new ClassPathXmlApplicationContext()的时候会打印以下日志：<!-- more -->

2017-5-15 12:48:48 org.springframework.context.support.AbstractApplicationContext prepareRefresh
即调用AbstractApplicationContext类的prepareRefresh方法，我们去看AbstractApplicationContext类：

通过类图可以发现AbstractApplicationContext是一个抽象类，也属于BeanFactory体系，实现了ApplicationContext，再往下找到他的子类ClassPathXmlApplicationContext，来看它的具体实现

类里有很多重载的构造函数，到最后都是调用这个：
```java
public ClassPathXmlApplicationContext(String[] paths, Class clazz, ApplicationContext parent) throws BeansException {
　　super(parent);
    Assert.notNull(paths, "Path array must not be null");
    Assert.notNull(clazz, "Class argument must not be null");
    this.configResources = new Resource[paths.length];
    for (int i = 0; i < paths.length; i++) {
            this.configResources[i] = new ClassPathResource(paths[i], clazz);
    }
　　 //IOC初始化过程
    refresh();
}
```
这里就到重点了，refresh方法定义了ioc容器启动的整个过程，来看源码
```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1.Prepare this context for refreshing.
        prepareRefresh();

        // 2.Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3.Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // 4.Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // 5.Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

            // 6.Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // 7.Initialize message source for this context.
            initMessageSource();

            // 8.Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // 9.Initialize other special beans in specific context subclasses.
            onRefresh();

            // 10.Check for listener beans and register them.
            registerListeners();

            // 11.Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // 12.Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            // 13.Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // 14.Reset 'active' flag.
            cancelRefresh(ex);

            // 15.Propagate exception to caller.
            throw ex;
        }
    }
}
```

接下来，一步一步分析spring干了哪些事

1．初始化BeanFactory：根据配置文件实例化BeanFactory，getBeanFactory()方法由具体子类实现。在这一步里，Spring将配置文件的信息解析成为一个个的BeanDefinition对象并装入到容器的Bean定义注册表（BeanDefinitionRegistry）中，但此时Bean还未初始化；obtainFreshBeanFactory()会调用自身的refreshBeanFactory(),而refreshBeanFactory()方法由子类AbstractRefreshableApplicationContext实现，该方法返回了一个创建的DefaultListableBeanFactory对象，这个对象就是由ApplicationContext管理的BeanFactory容器对象。这一步的操作相当于，如果我们在自己的应用代码中不用ApplicationContext而直接用BeanFactory时创建BeanFactory对象的操作，核心代码如下：

reader.loadBeanDefinitions(configLocations);
2．调用工厂后处理器，根据反射机制从BeanDefinitionRegistry中找出所有BeanFactoryPostProcessor类型的Bean，并调用其postProcessBeanFactory()接口方法。经过第一步加载配置文件，已经把配置文件中定义的所有bean装载到BeanDefinitionRegistry这个Beanfactory中，对于ApplicationContext应用来说这个BeanDefinitionRegistry类型的BeanFactory就是Spring默认的DefaultListableBeanFactory

public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements ConfigurableListableBeanFactory, BeanDefinitionRegistry{}
在这些被装载的bean中，若有类型为BeanFactoryPostProcessor的bean（配置文件中配置的），则将对应的BeanDefinition生成BeanFactoryPostProcessor对象容器扫描BeanDefinitionRegistry中的BeanDefinition，使用java反射自动识别出Bean工厂后处理器（实现BeanFactoryPostProcessor接口）的bean，然后调用这些bean工厂后处理器对BeanDefinitionRegistry中的BeanDefinition进行加工处理，可以完成以下两项工作(当然也可以有其他的操作，用户自己定义)：

1).对使用到占位符的<bean>元素标签进行解析，得到最终的配置值，这意味着对一些半成品式的BeanDefinition对象进行加工处理并取得成品的BeanDefinition对象。

2).对BeanDefinitionRegistry中的BeanDefinition进行扫描，通过Java反射机制找出所有属性编辑器的Bean（实现java.beans.PropertyEditor接口的Bean），并自动将它们注册到Spring容器的属性编辑器注册表中（PropertyEditorRegistry），这个Spring提供了实现：CustomEditorConfigurer，它实现了BeanFactoryPostProcessor，用它来在此注册自定义属性编辑器，核心代码如下：

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {

        // Invoke factory processors registered with the context instance.
        for (Iterator it = getBeanFactoryPostProcessors().iterator(); it.hasNext();) {
            BeanFactoryPostProcessor factoryProcessor = (BeanFactoryPostProcessor) it.next();
            factoryProcessor.postProcessBeanFactory(beanFactory);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // 通过ApplicatinContext管理的beanfactory获取已经注册的BeanFactoryPostProcessor类型的bean的名字

        String[] factoryProcessorNames =
                beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

        // Separate between BeanFactoryPostProcessors that implement the Ordered
        // interface and those that do not.
        List orderedFactoryProcessors = new ArrayList();
        List nonOrderedFactoryProcessorNames = new ArrayList();
        for (int i = 0; i < factoryProcessorNames.length; i++) {
            if (isTypeMatch(factoryProcessorNames[i], Ordered.class)) {

                // 调用beanfactory的getBean取得所有的BeanFactoryPostProcessor对象
                orderedFactoryProcessors.add(beanFactory.getBean(factoryProcessorNames[i]));
            }
            else {
                nonOrderedFactoryProcessorNames.add(factoryProcessorNames[i]);
            }
        }

        // First, invoke the BeanFactoryPostProcessors that implement Ordered.
        Collections.sort(orderedFactoryProcessors, new OrderComparator());
        for (Iterator it = orderedFactoryProcessors.iterator(); it.hasNext();) {
            BeanFactoryPostProcessor factoryProcessor = (BeanFactoryPostProcessor) it.next();

        // 执行BeanFactoryPostProcessor的方法，传入当前持有的beanfactory对象，以获取要操作的

        // BeanDefinition
            factoryProcessor.postProcessBeanFactory(beanFactory);
        }
        // Second, invoke all other BeanFactoryPostProcessors, one by one.
        for (Iterator it = nonOrderedFactoryProcessorNames.iterator(); it.hasNext();) {
            String factoryProcessorName = (String) it.next();
            ((BeanFactoryPostProcessor) getBean(factoryProcessorName)).

                    postProcessBeanFactory(beanFactory);
        }
    }
``` 
BeanFactoryPostProcessor接口代码如下，实际的操作由用户扩展并配置：

```java
public interface BeanFactoryPostProcessor {

        /**
         * Modify the application context's internal bean factory after its standard
         */
        void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

    }
```
3．注册Bean后处理器，根据反射机制从BeanDefinitionRegistry中找出所有BeanPostProcessor类型的Bean，并将它们注册到容器Bean后处理器的注册表中，AbstractApplicatinContext中对应代码如下：

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        String[] processorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

        // Register BeanPostProcessorChecker that logs an info message when
        int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 +

                processorNames.length;
        beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory,

                beanProcessorTargetCount));
        List orderedProcessors = new ArrayList();
        List nonOrderedProcessorNames = new ArrayList();

        for (int i = 0; i < processorNames.length; i++) {
            if (isTypeMatch(processorNames[i], Ordered.class)) {
                orderedProcessors.add(getBean(processorNames[i]));
            }
            else {
                nonOrderedProcessorNames.add(processorNames[i]);
            }
        }

        // First, register the BeanPostProcessors that implement Ordered.
        Collections.sort(orderedProcessors, new OrderComparator());
        for (Iterator it = orderedProcessors.iterator(); it.hasNext();) {

        // 注册bean后处理器，该方法定义于ConfigurableBeanFactory接口
            beanFactory.addBeanPostProcessor((BeanPostProcessor) it.next());
        }
        // Second, register all other BeanPostProcessors, one by one.
        for (Iterator it = nonOrderedProcessorNames.iterator(); it.hasNext();) {
            String processorName = (String) it.next();
            beanFactory.addBeanPostProcessor((BeanPostProcessor) getBean(processorName));
        }
    }
```
整段代码类似于第三步的调用工厂后处理器，区别之处在于，工厂后处理器在获取后立即调用，而Bean后处理器在获取后注册到上下文持有的beanfactory中，供以后操作调用（在用户获取bean的过程中，对已经完成属性设置工作的Bean进行后续加工，他加工的是bean，而工厂后处理器加工的是BeanDefinition）BeanPostProcessor 接口代码如下，实际的操作由用户扩展并配置：

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
4．初始化消息源，初始化容器的国际化信息资源，源代码如下：

```java
protected void initMessageSource() {
    // 具体实现
}
```
5．初始化应用上下文事件广播器；（观察者模式中的具体主题角色，持有观察者角色的集合，称为注册表）AbstractApplciationContext拥有一个applicationEventMulticaster 成员变量，applicationEventMulticaster 提供了容器监听器的注册表，成其为事件广播器。在第七步中将会将事件监听器装入其中，AbstractApplicationContext中的代码如下：

```java
protected void initApplicationEventMulticaster() {

        // "applicationEventMulticaster"，先看配置文件中有无配置该类型类（用户扩展 扩展点，如何扩展）
        if (containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            this.applicationEventMulticaster = (ApplicationEventMulticaster)
                    getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        }
        else {

            // 若没有，则应用Spring框架提供的事件广播器实例
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster();
        }
    }
    public boolean containsLocalBean(String name) {
        return getBeanFactory().containsLocalBean(name);
    }

    public Object getBean(String name, Class requiredType) throws BeansException {
        return getBeanFactory().getBean(name, requiredType);
    }
```
Spring初始化事件广播器，用户可以在配置文件中为容器定义一个自定义的事件广播器（bean的名称要为"applicationEventMulticaster"），只要实现ApplicationEventMulticaster就可以了，Spring在此会根据beanfactory自动获取。如果没有找到外部配置的事件广播器，Spring使用SimpleApplicationEventMulticaster作为事件广播器。

6．初始化其他特殊的Bean：这是一个钩子方法，子类可以借助这个钩子方法执行一些特殊的操作，如AbstractRefreshableWebApplicationContext就使用该钩子方法执行初始化ThemeSource的操作；

```java
protected void onRefresh() throws BeansException {
        // For subclasses: do nothing by default.
}
```
7．注册事件监听器；（观察者模式中的观察者角色）

Spring根据上下文持有的beanfactory对象，从它的BeanDefinitionRegistry中找出所有实现org.springfamework.context.ApplicationListener的bean，将BeanDefinition对象生成bean，注册为容器的事件监听器，实际的操作就是将其添加到事件广播器所提供的监听器注册表中

AbstractApplicationContext中的代码如下：


```java
    private List applicationListeners = new ArrayList();

    public List getApplicationListeners() {
        return this.applicationListeners;
    }

    protected void registerListeners() {
        // Register statically specified listeners first.
        for (Iterator it = getApplicationListeners().iterator(); it.hasNext();) {
            addListener((ApplicationListener) it.next());
        }
        // 获取ApplicationListener类型的所有bean，即事件监听器
        // uninitialized to let post-processors apply to them!
        Collection listenerBeans = getBeansOfType(ApplicationListener.class, true, false).values();
        for (Iterator it = listenerBeans.iterator(); it.hasNext();) {

            // 将事件监听器装入第五步初始化的事件广播器
            addListener((ApplicationListener) it.next());
        }
    }

    public Map getBeansOfType(Class type, boolean includePrototypes, boolean allowEagerInit)
            throws BeansException {

        return getBeanFactory().getBeansOfType(type, includePrototypes, allowEagerInit);
    }

    protected void addListener(ApplicationListener listener) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }
```
ApplicationListener 的源代码如下：
```java
/**
     * Interface to be implemented by application event listeners.
     * @see org.springframework.context.event.ApplicationEventMulticaster
     */
    public interface ApplicationListener extends EventListener {
        void onApplicationEvent(ApplicationEvent event);

    }
```
8．初始化singleton的Bean：实例化所有singleton的Bean，并将它们放入Spring容器的缓存中；这就是和直接在应用中使用BeanFactory的区别之处，在创建ApplicationContext对象时，不仅创建了一个BeanFactory对象，并且还应用它实例化所有单实例的bean。（在spring的配置文件中，bean默认为单例，除非在bean的配置中显式指定scope="prototype"）

AbstractApplicationContext中的代码如下：

```java
beanFactory.preInstantiateSingletons();
```
9．发布上下文刷新事件：在此处时容器已经启动完成，发布容器refresh事件（ContextRefreshedEvent）

创建上下文刷新事件，事件广播器负责将些事件广播到每个注册的事件监听器中。

```java
publishEvent(new ContextRefreshedEvent(this));

    public void publishEvent(ApplicationEvent event) {
        Assert.notNull(event, "Event must not be null");

        // 在此获取事件广播器，并调用其方法发布事件：调用所有注册的监听器的方法
        getApplicationEventMulticaster().multicastEvent(event);
        if (this.parent != null) {
            this.parent.publishEvent(event);
        }
    }
```
至此，ApplicationContext对象就完成了初始化工作：创建BeanFactory来装配BeanDefiniton，加工处理BeanDefiniton，注册了bean后处理器，初始化了消息资源，初始化了应用上下文事件广播器，注册了事件监听器，初始化了所有singleton的bean，最后发布上下文刷新事件。

<hr />