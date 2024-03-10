## Spring核心技术与源码解析

### Bean

#### BeanDefinition

BeanDefinition在Spring源码中为一个接口。为简化起间，先简单列举出其中比较重要的属性。

| 属性           | 作用                                            |
| -------------- | ----------------------------------------------- |
| name           |                                                 |
| class          |                                                 |
| scope          | singleton prototype request session application |
| factory-bean   |                                                 |
| factory-method |                                                 |
| init-method    |                                                 |
| destroy-method |                                                 |
| autowire-type  |                                                 |

> scope失效

#### Bean的创建

FactoryBean

FactoryBean会跳过一些Bean的生命周期。



#### Bean的生命周期

- 实例化前：beforeInstantiation
  - InstantiationAwareBeanPostProcessor定义的扩展
- 构造
- 实例化后：afterInstantiation
  - InstantiationAwareBeanPostProcessor定义的扩展
- 依赖注入前：postProcessProperties
- 依赖注入
- Aware接口
  - BeanNameAware
  - BeanFactoryAware
  - ApplicationContextAware
- 初始化前：beforeInitialization
  - BeanPostProcessor定义的扩展，初始化之前
- 初始化
  - @PostConstruct指定的init方法
  - InitializingBean实现的afterPropertiesSet方法
  - bean中指定的init-method
- 初始化后：afterInitialization
  - BeanPostProcessor定义的扩展，初始化之后
- 销毁前：beforeDestruction
  - DestructionAwareBeanPostProcessor定义的扩展，销毁之前
- 销毁bean
  - @PreDestroy指定的destroy方法
  - DisposableBean实现的destroy方法
  - bean中指定的init-method

#### XML定义bean

#### 注解定义bean

| 注解           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| @Component     | 标注该类需要交给Spring管理。需要通过ComponentScan指定扫描的basePackages。四个派生注解@Repository, @Service, @Controller, @Configuration |
| @Configuration | 配置类，如使用@Bean定义工厂方法创建bean，配置类上必须加@Configuration |
| @ComponentScan | 指定component注解扫描路径                                    |
| @Bean          |                                                              |
|                |                                                              |







### BeanFactory

#### 继承结构



#### BeanFactoryPostProcessor

BeanFactoryPostProcessor为一个接口，方法为postProcessBeanFactory，入参为ConfigurableListableBeanFactory。

BeanFactoryPostProcessor的作用主要是动态地增加一些BeanDefinition。

| 常见BeanFactoryPostProcessor    | 作用                                     |
| ------------------------------- | ---------------------------------------- |
| ConfigurationClassPostProcessor | 解析@ComponentScan、@Bean、@Import等注解 |
| MapperScannerConfigurer         | Mybatis中解析如@MapperScan等注解         |
|                                 |                                          |

> 模拟实现@ComponentScan注解

现模拟实现@ComponentScan注解的功能，以便更深入理解该注解的底层实现和BeanFactoryPostProcessor的作用。在实现时实现的是BeanDefinitionRegistryPostProcessor，该类为BeanFactoryPostProcessor的子类。

```java
public class ComponentScanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        try {
            // 这里写死配置类
            ComponentScan componentScan = AnnotationUtils.findAnnotation(MyConfig.class, ComponentScan.class);
            if (componentScan != null) {
                for (String p : componentScan.basePackages()) {
                    String path = "classpath*:" + p.replace(".", "/") + "/**/*.class";
                    // 用于读取类的元数据信息
                    CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
                    Resource[] resources = new PathMatchingResourcePatternResolver().getResources(path);
                    // 用于生成beanName
                    AnnotationBeanNameGenerator generator = new AnnotationBeanNameGenerator();
                    for (Resource resource : resources) {
                        MetadataReader reader = factory.getMetadataReader(resource);
                        AnnotationMetadata metadata = reader.getAnnotationMetadata();
                        String componentName = Component.class.getName();
                        // hasMetaAnnotation用来判断是否是Component派生注解
                        if (metadata.hasAnnotation(componentName) || metadata.hasMetaAnnotation(componentName)) {
                            BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(reader.getClassMetadata().getClassName());
                            AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

                            String beanName = generator.generateBeanName(beanDefinition, beanDefinitionRegistry);
                            beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

> 模拟实现@Bean接口

```java
public class AtBeanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        try {
            CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
            // 暂时写死，为class文件的路径
            MetadataReader reader = factory.getMetadataReader(new ClassPathResource("com/spring/demo/config/Config.class"));
            Set<MethodMetadata> methods = reader.getAnnotationMetadata().getAnnotatedMethods(Bean.class.getName());
            for (MethodMetadata method : methods) {
                String initMethod = method.getAnnotationAttributes(Bean.class.getName()).get("initMethod").toString();
                BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
                // 指定factoryMethod和factoryBean
                builder.setFactoryMethodOnBean(method.getMethodName(), "config");
                if (initMethod.length() > 0) {
                    builder.setInitMethodName(initMethod);
                }
                AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
                // 以方法名作为beanName
                beanDefinitionRegistry.registerBeanDefinition(method.getMethodName(), beanDefinition);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

> 模拟实现@Mapper接口

假如有一个标注了@Mapper注解的接口Mapper1，Spring该如何创建这个bean呢？

```java
@Configuration
class Config {
    @Bean 
    public MapperFactoryBean<Mapper1> mapper1(SqlSessionFactory sqlSessionFactory) {
        MapperFactoryBean<Mapper1> factory = new MapperFactoryBean<>(Mapper1.class);
        factory.setSqlSessionFactory(sqlSessionFactory);
        return factory;
    }
}
```

这种写法的缺陷在于，如果有多个Mapper，则每个Mapper都需要采用这种方式去写，效率比较低。

类比ComponentScan，实现一个类似MapperScan的功能，指定了包路径后会进行扫描，将加有Mapper注解的类配置为bean。

```java
public class MapperPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        try {
            PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
            Resource[] resources = resolver.getResources("classpath:com/spring/demo/mapper/**/*.class");
            CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
            AnnotationBeanNameGenerator generator = new AnnotationBeanNameGenerator();

            for (Resource resource : resources) {
                MetadataReader reader = factory.getMetadataReader(resource);
                ClassMetadata classMetadata = reader.getClassMetadata();
                if (classMetadata.isInterface()) {
                    AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(MapperFactoryBean.class)
                            .addConstructorArgValue(classMetadata.getClassName())
                            .setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE)
                            .getBeanDefinition();

                    // 仅用于生成beanName，如果用beanDefinition生成，name会是mapperFactoryBean
                    AbstractBeanDefinition bd = BeanDefinitionBuilder.genericBeanDefinition(classMetadata.getClassName()).getBeanDefinition();
                    String beanName = generator.generateBeanName(bd, beanDefinitionRegistry);
                    
                    beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
                    
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    } 

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

#### BeanPostProcessor

BeanPostProcessor主要是对Bean生命周期提供一些扩展。

| BeanFactoryPostProcessor接口                     | 方法                            | 作用                                                   |
| ------------------------------------------------ | ------------------------------- | ------------------------------------------------------ |
| BeanPostProcessor                                | postProcessBeforeInitialization | 初始化前                                               |
|                                                  | postProcessAfterInitialization  | 初始化后，通常用于创建代理                             |
| InstantiationAwareBeanPostProcessor（子接口）    | postProcessBeforeInstantiation  | 如果返回不为null，则直接使用该返回值，不创建新的对象。 |
|                                                  | postProcessAfterInstantiation   | 如果返回值不为true，则跳过后续的依赖注入等过程。       |
|                                                  | postProcessProperties           | 依赖注入前调用。                                       |
| SmartInstantiationAwareBeanPostProcessor(子接口) | getEarlyBeanReference           | 提前暴露bean，主要用于循环引用场景。                   |
| DestructionAwareBeanPostProcessor(子接口)        | postProcessBeforeDestruction    | 销毁前                                                 |

常见的BeanPostProcessor

| 常见BeanPostProcessor                       | 作用                                             |
| ------------------------------------------- | ------------------------------------------------ |
| AutowiredAnnotationBeanPostProcessor        | 解析@Autowired和@Value等注解                     |
| CommonAnnotationBeanPostProcessor           | 解析@Resource，@PostConstruct，@PreDestroy等注解 |
| ConfigurationPropertiesBindingPostProcessor | SpringBoot中，用于绑定@ConfigurationProperties   |
| AnnotationAwareAspectJAutoProxyCreator      | 用于实现AOP等功能                                |
|                                             |                                                  |



> @Autowired原理

现简单解析AutowiredAnnotationBeanPostProcessor如何解析@Autowired注解。

AutowiredAnnotationBeanPostProcessor实现了SmartInstantiationAwareBeanPostProcessor中的postProcessProperties方法。

第一步：查找哪些属性、方法加入了@Autowired注解，将其封装为InjectionMetadata。

核心方法：findAutowiringMetadata

第二步：findAutowiringMetadata的返回值为InjectionMetadata，调用其inject方法，按照类型完成依赖注入。

```java
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        InjectionMetadata metadata = this.findAutowiringMetadata(beanName, bean.getClass(), pvs);

        try {
            metadata.inject(bean, beanName, pvs);
            return pvs;
        } catch (BeanCreationException var6) {
            throw var6;
        } catch (Throwable var7) {
            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", var7);
        }
    }
```

第三步：封装为DependencyDescriptor，按照类型注入。

Autowired可以标注在方法上，对应源代码AuowiredMethodElement

也可以标注在字段上，对应源代码AutowiredFieldElement

 DependencyDescriptor举例：

```java
Field bean3Field = Bean1.class.getDeclaredField("bean3");
DependencyDescriptor dd1 = new DependencyDescriptor(bean3Field, false);
Object bean3 = beanFactory.doResolveDependency(dd1, null, null, null);

Method setBean2 = Bean1.class.getDeclaredMethod("setBean2", Bean2.class);
DependencyDescriptor dd2 = new DependencyDescriptor(new MethodParameter(setBean2, 0));
Object bean2 = beanFactory.doResolveDependency(dd2, null, null, null);
```

> 不同类型的Autowired注入模拟

现用代码模拟不同类型的Autowired注入。

```java
@Component
public class AutowireTest {

    static class Target {
        @Autowired private Service[] serviceArray;
        @Autowired private List<Service> serviceList;
        @Autowired private ConfigurableApplicationContext applicationContext;
        @Autowired private Dao<Teacher> dao;
        @Autowired @Qualifier("service2") private Service service;

    }

    private static void testArray(DefaultListableBeanFactory beanFactory) throws NoSuchFieldException {
        DependencyDescriptor descriptor = new DependencyDescriptor(Target.class.getDeclaredField("serviceArray"), true);
        if (descriptor.getDependencyType().isArray()) {
            Class<?> componentType = descriptor.getDependencyType().getComponentType();
            String[] names = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, componentType);
            List<Object> beans = new ArrayList<>();
            for (String name : names) {
                System.out.println(name);
                Object bean = descriptor.resolveCandidate(name, componentType, beanFactory);
                beans.add(bean);
            }
            Object array = beanFactory.getTypeConverter().convertIfNecessary(beans, descriptor.getDependencyType());
            System.out.println(array);
        }
    }

    private static void testList(DefaultListableBeanFactory beanFactory) throws NoSuchFieldException {
        DependencyDescriptor descriptor = new DependencyDescriptor(Target.class.getDeclaredField("serviceList"), true);
        if (descriptor.getDependencyType() == List.class) {
            Class<?> resolve = descriptor.getResolvableType().getGeneric().resolve();
            String[] names = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, resolve);
            List<Object> beans = new ArrayList<>();
            for (String name : names) {
                System.out.println(name);
                Object bean = descriptor.resolveCandidate(name, resolve, beanFactory);
                beans.add(bean);
            }
            System.out.println(beans);
        }
    }

    private static void testApplicationContext(DefaultListableBeanFactory beanFactory) throws NoSuchFieldException, IllegalAccessException {
        DependencyDescriptor descriptor = new DependencyDescriptor(Target.class.getDeclaredField("applicationContext"), true);
        Field resolvableDependencies = DefaultListableBeanFactory.class.getDeclaredField("resolvableDependencies");
        resolvableDependencies.setAccessible(true);
        Map<Class<?>, Object> dependencies = (Map<Class<?>, Object>) resolvableDependencies.get(beanFactory);
        for (Map.Entry<Class<?>, Object> entry : dependencies.entrySet()) {
            if (entry.getKey().isAssignableFrom(ConfigurableApplicationContext.class)) {
                System.out.println(entry.getValue());
                break;
            }
        }
    }

    private static void testGeneric(DefaultListableBeanFactory beanFactory) throws NoSuchFieldException {
        DependencyDescriptor descriptor = new DependencyDescriptor(Target.class.getDeclaredField("dao"), true);
        Class<?> type = descriptor.getDependencyType();
        ContextAnnotationAutowireCandidateResolver resolver = new ContextAnnotationAutowireCandidateResolver();
        resolver.setBeanFactory(beanFactory);
        for (String name : BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, type)) {
            BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(name);
            if (resolver.isAutowireCandidate(new BeanDefinitionHolder(beanDefinition, name), descriptor)) { // 泛型是否匹配
                System.out.println(name);
                Object candidate = descriptor.resolveCandidate(name, type, beanFactory);
                System.out.println(candidate);
            }
        }
    }

    private static void testQualifier(DefaultListableBeanFactory beanFactory) throws NoSuchFieldException {
        DependencyDescriptor descriptor = new DependencyDescriptor(Target.class.getDeclaredField("dao"), true);
        Class<?> type = descriptor.getDependencyType();
        ContextAnnotationAutowireCandidateResolver resolver = new ContextAnnotationAutowireCandidateResolver();
        resolver.setBeanFactory(beanFactory);
        for (String name : BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, type)) {
            BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(name);
            if (resolver.isAutowireCandidate(new BeanDefinitionHolder(beanDefinition, name), descriptor)) { // Qualifier
                System.out.println(name);
                Object candidate = descriptor.resolveCandidate(name, type, beanFactory);
                System.out.println(candidate);
            }
        }
    }

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AutowireTest.class);
        DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();
        testArray(beanFactory);
        testList(beanFactory);
        testApplicationContext(beanFactory);
        testGeneric(beanFactory);
        testQualifier(beanFactory);
    }


    interface Service {}

    @Component("service1")
    static class Service1 implements Service {}

    @Component("service2")
    static class Service2 implements Service {}

    interface Dao<T> {}

    static class Student {}
    static class Teacher {}

    @Component("dao1")
    static class Dao1 implements Dao<Student> {}

    @Component("dao2")
    static class Dao2 implements Dao<Teacher> {}

}

```

> @Value原理

关键类：ContextAnnotationAutowireCandidateResolver

首先封装为DependencyDescriptor，然后调用resolver的getSuggestedValue，传入DependencyDescriptor为入参。

| 主要场景                          | 原理                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| 普通值注入，可能需要类型转换      | beanFactory.getTypeConverter().convertIfNecessary，传入值和需要转换的类型。通过DependencyDescriptor的getDependencyType()方法 |
| ${}类型注入 如\${JAVA_HOME}       | context.getEnvironment.resolvePlaceholders()                 |
| #{} EL表达式类型注入，如#{@bean1} | 涉及到SpringEL表达式的解析。见示例代码。                     |

示例代码：

```java
@ComponentScan
public class ValueTest {
    public static void main(String[] args) throws NoSuchFieldException {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ValueTest.class);
        DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();

        ContextAnnotationAutowireCandidateResolver resolver = new ContextAnnotationAutowireCandidateResolver();
        resolver.setBeanFactory(beanFactory);

        Field field = Bean1.class.getDeclaredField("bean2");
        DependencyDescriptor descriptor = new DependencyDescriptor(field, false);

        String value = resolver.getSuggestedValue(descriptor).toString();
        Object bean2 = beanFactory.getBeanExpressionResolver().evaluate(value, new BeanExpressionContext(beanFactory, null));
        System.out.println(bean2.getClass());
    }
    
    class Bean1 {
        @Value("${JAVA_HOME}")
        private String home;

        @Value("18")
        private int age;

        @Value("#{@bean2}")
        private Bean2 bean2;
    }

    @Component
    class Bean2 {}
}

```



#### 循环依赖与三级缓存



### ApplicationContext

#### 继承体系

> 上级

Application间接继承BeanFactory，同时继承了如下四大接口

| 接口                      | 功能                       |
| ------------------------- | -------------------------- |
| MessageSource             | 国际化、本地化功能         |
| ResourcePatternResolver   | 资源                       |
| ApplicationEventPublisher | 消息发布                   |
| EnvironmentCapable        | 获取系统环境变量等配置信息 |

> 下级



ClassPathXMLApplicationContext

AnnotationConfigApplicationContext

#### Refresh

> Autowired失效



#### 事件机制

| 概念类                      | 作用                                                         |
| --------------------------- | ------------------------------------------------------------ |
| ApplicationEvent            | 继承java中的EventObject，事件最顶层父类。构造方法需要传入一个Object source |
| ApplicationListener         | 事件监听器，接口，定义了onApplicationEvent方法               |
| GenericApplicationListener  | SmartApplicationListener子类，定义了supportsEventType方法    |
| ApplicationEventPublisher   | 事件发布器，接口，定义publishEvent方法                       |
| ApplicationEventMulticaster |                                                              |

Spring自带的事件

| 事件                  | 触发                 |
| --------------------- | -------------------- |
| ContextRefreshedEvent | 上下文初始化或刷新时 |
| ContextStartedEvent   | 手动调用start        |
| ContextStoppedEvent   | 手动调用stop         |
| ContextClosedEvent    | 上下文关闭时         |

示例代码

定义事件

```java
class MyEvent extends ApplicationEvent {
    public MyEvent(Object source) {
        super(source);
    }
}
```

发布事件

```java
@Component
class MyService {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void doBusiness() {
        System.out.println("do business");
        
        publisher.publishEvent(new MyEvent("do business"));
    }
}
```

事件监听：两种方式

方式1：实现ApplicationListener接口

```java
@Component
class EmailListener implements ApplicationListener {
	@Override
    public void onApplicationEvent(MyEvent event) {
        System.out.println("send email");
    }
}
```

方式2：方法上增加EventListener注解

```java
@Component
class EmailListener implements ApplicationListener {
	@EventListener
    public void sendEmail(MyEvent event) {
        System.out.println("send email");
    }
}
```

设置线程池实现一步化发送事件

```java
	@Bean
    public ThreadPoolTaskExecutor executor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        return executor;
    }

    @Bean
    public SimpleApplicationEventMulticaster applicationEventMulticaster(ThreadPoolTaskExecutor executor) {
        SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
        multicaster.setTaskExecutor(executor);
        return multicaster;
    }
```

> 模拟实现EventListener注解

思路：将该注解标注的方法所在的类转化为ApplicationListener

首先自定义注解MyListener

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface MyListener {
}
```

添加一个SmartInitializingSingleton的实现类，SmartInitializingSingleton接口定义了afterSingletonsInstantiated方法。所有单例实例化后执行该方法。

```java
	@Bean
    public SmartInitializingSingleton smartInitializingSingleton(ConfigurableApplicationContext context) {
        return () -> {
            for (String name : context.getBeanDefinitionNames()) {
                Object bean = context.getBean(name);
                for (Method method : bean.getClass().getMethods()) {
                    if (method.isAnnotationPresent(MyListener.class)) {
                        ApplicationListener listener = event -> {
                            try {
                                Class<?> parameterType = method.getParameterTypes()[0];
                                if (parameterType.isAssignableFrom(event.getClass())) {
                                    method.invoke(bean, event);
                                }
                            } catch (InvocationTargetException | IllegalAccessException e) {
                                e.printStackTrace();
                            }
                        };
                        context.addApplicationListener(listener);
                    }
                }
            }
        };
    }
```

定义一个ApplicationEventMulticaster的空实现，名为AbstractApplicationEventMulticaster

```java
abstract class AbstractApplicationEventMulticaster implements ApplicationEventMulticaster {
    // ... 所有方法空实现
}
```

添加ApplicationEventMulticaster的实现，将ApplicationListener封装为GenericApplicationListener。

```java
	@Bean
    public ApplicationEventMulticaster applicationEventMulticaster(ConfigurableApplicationContext context, ThreadPoolTaskExecutor executor) {
        return new AbstractApplicationEventMulticaster() {
            private List<GenericApplicationListener> listeners = new ArrayList<>();
            @Override
            public void addApplicationListenerBean(String name) {
                ApplicationListener listener = context.getBean(name, ApplicationListener.class);
                ResolvableType type = ResolvableType.forClass(listener.getClass()).getInterfaces()[0].getGeneric();
                GenericApplicationListener genericApplicationListener = new GenericApplicationListener() {
                    @Override
                    public boolean supportsEventType(ResolvableType eventType) {
                        return type.isAssignableFrom(eventType);
                    }

                    @Override
                    public void onApplicationEvent(ApplicationEvent event) {
                        executor.submit(() -> listener.onApplicationEvent(event));
                    }
                };
                listeners.add(genericApplicationListener);
            }

            @Override
            public void multicastEvent(ApplicationEvent event, ResolvableType resolvableType) {
                for (GenericApplicationListener listener : listeners) {
                    if (listener.supportsEventType(ResolvableType.forType(event.getClass()))) {
                        listener.onApplicationEvent(event);
                    }
                }
            }
        };
    }
```

> 事件机制源码浅析

todo

Spring的Context在refresh时会

### AOP

#### 术语

| 术语             | 解释       |
| ---------------- | ---------- |
| 目标 Target      | 被代理的类 |
| 代理 Proxy       | 增强后的类 |
| 连接点 JoinPoint |            |
| 切点 Pointcut    |            |
| 通知 Advice      |            |
| 切面 Advisor     |            |
| 切面 Aspect      |            |



#### 代理

> JDK动态代理

JDK动态代理，要求目标必须实现接口，生成的代理类实现相同的接口，代理类和目标类是平级。

示例：

1. 目标类(Bar)实现某接口(Foo)
2. 调用Proxy.newProxyInstance创建代理，穿入classLoader，接口数组和InvocationHandler的实现类。
3. 将instance转换为接口类型

```java
public class ProxyTest {
    public static void main(String[] args) {
        Foo bar = new Bar();
        Foo proxyFoo = (Foo) Proxy.newProxyInstance(Foo.class.getClassLoader(), new Class[]{Foo.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("before");
                Object result = method.invoke(bar, args);
                System.out.println("after");
                return result;
            }
        });
        proxyFoo.func();
    }
}
```

InvocationHandler为一个接口，需要重写其invoke方法。

反编译其代码，可以看到被增强的类继承自Proxy，内部持有一个InvocationHandler的引用。其static代码块会初始化各个Method，作为代理方法调用时的参数。

```java
public final class $Proxy0 extends Proxy implements Foo {
    private static Method m1; //equals()方法
    private static Method m3; //实现的Foo中的func()方法
    private static Method m2; //toString()方法
    private static Method m0; //hashCode()方法
    
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }
 
    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
 
    public final void func() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m3 = Class.forName("proxy.service.Foo").getMethod("func", new Class[0]);
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}


```

> Cglib代理

CGLIB代理不要求目标实现接口，生成的代码类是目标的子类，代理类和目标类是子父关系。因此，final类无法被CGLIB增强。

示例：

1.引入依赖

```xml
<dependency>
  <groupId>cglib</groupId>
  <artifactId>cglib</artifactId>
  <version>3.3.0</version>
</dependency>
```

2. 定义MethodInterceptor的实现类，重写intercept方法。

注意：在调用增强方法时，有三种写法。

```java
public class ProxyTest {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(bar.getClass());
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
                System.out.println("before");
                // 写法1：反射调用
                Object invoke = method.invoke(bar, args);
                // 写法2：非反射调用
                methodProxy.invoke(bar, args);
                // 写法3：非反射调用，可忽略target对象
                methodProxy.invokeSuper(proxy, args);
                System.out.println("after");
                return invoke;
            }
        });
        Bar proxyBar = (Bar) enhancer.create();
        proxyBar.make();
    }
}
```

> 代理方式的选择

Spring中通过ProxyFactory创建代理，会调用getProxy方法。

代理的选择，详见DefaultAopProxyFactory中的createAopProxy方法。

结论：

proxyTargetClass为false且目标实现了接口，采用JDK动态代理(JdkDynamicAopProxy)。

proxyTargetClass为true，或目标未实现接口，采用Cglib动态代理(ObjenesisCglibAopProxy)。

如何配置proxyTargetClass？

注解方式：@EnableAspectJAutoProxy(proxyTargetClass = true)

XML方式：<aop:aspectj-autoproxy proxy-target-class="true"/>

> 代理创建时机

通常来说，代理创建有两种时机。

第一种：没有发生循环依赖，创建代理在初始化后。

第二种：发生了循环依赖，在实例化后、依赖注入前。

> 代理特点

- 依赖注入和初始化影响的是原始对象。

验证：
定义Bean1类
```java
@Component
public class Bean1 {

    protected Bean2 bean2;

    protected boolean initialized;

    @Autowired
    public void setBean2(Bean2 bean2) {
        System.out.println("set bean2");
        this.bean2 = bean2;
    }

    @PostConstruct
    public void init() {
        System.out.println("init");
        initialized = true;
    }
}
```
定义切面，为Bean1的所有方法都添加一个前置通知。
```java
@Aspect
@Component
public class Bean1Aspect {
    @Before("execution(* com.alipay.demo.aopstudy.Bean1.*(..))")
    public void before() {
        System.out.println("before");
    }
}
```
测试类
```java
@ComponentScan
@EnableAspectJAutoProxy
public class BeanProxyTest {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanProxyTest.class);

        Bean1 bean1 = (Bean1) context.getBean("bean1");

        bean1.setBean2(new Bean2());
        context.close();
    }
}
```
可以观察到，在第8行代码执行时，会进行bean1的依赖注入和初始化操作，调用setBean2和init方法，这两个方法并不会被增强，即不会打印before。
第10行手动再次调用setBean2，此时由于bean1为代理对象，会调用before方法。

- 代理与目标是两个对象，二者成员变量并不公用数据


在刚才的代码中稍作修改
```java
@ComponentScan
@EnableAspectJAutoProxy
public class BeanProxyTest {

    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanProxyTest.class);

        Bean1 bean1 = (Bean1) context.getBean("bean1");
        showProxyAndTarget(bean1);

        System.out.println(bean1.getBean2());
        System.out.println(bean1.isInitialized());
        
        context.close();
    }

    public static void showProxyAndTarget(Bean1 proxy) throws Exception {
        System.out.println(">>> 代理中的成员变量");
        System.out.println("initialized : " + proxy.initialized);
        System.out.println("bean2 : " + proxy.bean2);

        if (proxy instanceof Advised) {
            Advised advised = (Advised) proxy;
            System.out.println(">>> 目标中的成员变量");
            Bean1 target = (Bean1) advised.getTargetSource().getTarget();
            System.out.println("initialized : " + target.initialized);
            System.out.println("bean2 : " + target.bean2);
        }
    }
}
```
对于代理中的成员变量，其initialized为false，bean2为null。

对于目标中的成员变量，其initialized为true，bean2为实际的对象。

为Bean1增加getBean2和isInitialized方法。

执行11、12行代码，其initialized为true，bean2为实际的对象。因为代理对象最终还是会调用原始对象的方法。

- static方法、final方法和private方法均无法增强

#### 实现原理解析

> 切点表达式

execution([访问修饰符] 返回值类型 包名.类名.方法名(参数))

- 访问修饰符可以省略。
- 返回值类型、某一级包名、类名、方法名可以用*表示任意。
- 包名与类名之间单点表示该包下的类，双点表示该包及其子包下的类。
- 参数列表使用两个点表示任意参数。
  
> 通知

| 通知     | 配置           | 时机                                             |
| -------- | -------------- | ------------------------------------------------ |
| 前置通知 | @Before        | 目标方法执行前                                   |
| 后置通知 | @AfterRetuning | 目标方法执行后且无异常                           |
| 异常通知 | @AfterThrowing | 目标方法执行抛出异常                             |
| 最终通知 | @After         | 无论是否有异常均执行                             |
| 环绕通知 | @Around        | 目标方法执行前后执行。出现异常，环绕后方法不执行 |

可传参数类型

| 参数                | 作用                                                         |
| ------------------- | ------------------------------------------------------------ |
| JoinPoint           | 连接点对象，可以获得当前目标对象、方法参数等信息。getArgs()获取目标方法参数；getTarget()获取目标对象；getStaticPart获取精确的切点表达式信息。 |
| ProceedingJoinPoint | JoinPoint子类，主要用于环绕通知中调用proceed进而执行目标方法 |
| Throwable           | 异常对象，异常通知中使用，需要在配置文件中指出异常对象名称   |

> 切面配置

现介绍如何通过注解方式配置，XML配置方法类似。

- 基于Aspect的配置方式

1. 定义一个类，类上标注@Aspect和@Component注解。
2. 定义切点，定义一个方法，方法名任意，无需实现，方法上添加@Pointcut，写切点表达式。也可以在通知中指定pointcut，见第3步。
3. 定义通知，方法上添加@Before等注解。对于@AfterThrowing，需要在注解中指定throwing。
4. 开启@EnableAspectJAutoProxy，用于解析AspectJ相关的注解。

原理实现：参见AnnotationAwareAspectJAutoProxyCreator

> 切面顺序

对于Aspect高级切面，通过类上Order注解指定顺序。

对于Advisor低级切面，调用setOrder指定优先级。

> ProxyFactory用法

示例：

1. 定义切点，选用AspectJExpressionPointcut
2. 定义通知，定义MethodInterceptor
3. 创建切面，DefaultPointcutAdvisor，由一个切点和通知组成
4. 创建代理，调用getProxy()

```java
public class PointcutTest {
    public static void main(String[] args) {
        // 切点
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression("execution(* foo())");
        // 通知
        MethodInterceptor methodInterceptor = new MethodInterceptor() {
            @Nullable
            @Override
            public Object invoke(@Nonnull MethodInvocation invocation) throws Throwable {
                System.out.println("before");
                Object result = invocation.proceed();
                System.out.println("after");
                return result;
            }
        };
        // 切面
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, methodInterceptor);
        // 创建代理
        ProxyFactory factory = new ProxyFactory();
        Target1 target1 = new Target1();
        factory.setTarget(target1);
        factory.addAdvisor(advisor);
        factory.setInterfaces(target1.getClass().getInterfaces());
        I1 proxy = (I1) factory.getProxy();
        proxy.foo();
        proxy.bar();
    }

    interface I1 {
        void foo();

        void bar();
    }

    static class Target1 implements I1 {

        @Override
        public void foo() {
            System.out.println("target1 foo");
        }

        @Override
        public void bar() {
            System.out.println("target1 bar");
        }
    }
}
```


#### 源码解析

关键类

> AdvisedSupport类

重要属性：

| 字段                | 作用                          |
| ------------------- | ----------------------------- |
| TargetSource        | 目标源，调用getTarget获取目标 |
| AdvisorChainFactory | 创建拦截器链的工厂            |
| List\<Advisor\>     | 存储的低级切面列表            |

> 代理创建

通过ProxyFactory.getProxy()创建代理对象。

ProxyFactory间接继承AdvisedSupport。

DefaultAopProxyFactory的createAopProxy方法中会选择创建代理的方式。

包含JdkDynamicAopProxy和Cglib动态代理ObjenesisCglibAopProxy。


> 高级切面转换为低级切面

通过@Aspect定义的高级切面，最终都会转换为Advisor低级切面。

> 统一转换为环绕通知

MethodInterceptor为环绕通知。注意，这个MethodInterceptor和cglib中的MethodInterceptor并不是同一个。

诸如MethodBeforeAdvice，AfterAdvice都会转换为MethodInterceptor
采用通知

> 拦截器链

MethodInvocation

> 和IOC集成

通过Bean后处理器将AOP和beanFactory集成在一起。




> 切面







### 事务

### 设计模式

#### 代理模式

#### 适配器模式

#### 责任链模式

#### 工厂模式

#### 策略模式

#### 模版方法模式



### 整合第三方框架

xml方式整合第三方框架有两种方案：

不需要自定义命名空间，不需要使用Spring配置文件配置第三方框架本身内容，如Mybatis

需要引入第三方框架命名空间，需要使用Spring配置文件配置第三方框架本身内容，如Dubbo

#### Mybatis

回顾一下Mybatis的用法

```java
public class MybatisTest {
    public static void main(String[] args) throws IOException {
        InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
        SqlSessionFactory sqlSessionFactory = builder.build(inputStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    }
}
```

整合步骤：

1. 导入Mybatis整合Spring的坐标
```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.6</version>
</dependency>
```
导入了一些用于整合的类，核心类有

| 类                      | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| SqlSessionFactoryBean   | 需要配置，为FactoryBean，用于产生SqlSessionFactory           |
| MapperScannerConfigurer | 需要配置，为BeanFactoryPostProcessor，根据报名扫描mapper并注册BeanDefinition |
| MapperFactoryBean       | 无需配置，为Mapper的FactoryBean                              |
| ClassPathMapperScanner  | 修改了自动注入状态，MapperFactoryBean的setSqlSessionFactory会自动注入 |

2. 编写Mapper和Mapper.xml
3. 配置SqlSessionFactoryBean和MapperScannerConfigurer

```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property>...</property>
</bean>
<!-- 将SqlSessionFactory存储到容器 -->
<bean class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource">
    </property>
</bean>
<!-- 指定扫描路径 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage"></property>
</bean>
```

配置完毕后，可以自动注入需要的Mapper。

> 原理剖析

- SqlSessionFactoryBean

实现了FactoryBean\<SqlSessionFactory>, InitializingBean接口。

依赖注入时，会setDataSource。

由于其实现了InitializingBean，在afterPropertiesSet时，会buildSqlSessionFactory。

getObject时，返回sqlSessionFactory对象。

- MapperScannerConfigurer

实现了BeanDefinitionRegistryPostProcessor，为一个工厂后处理器。

关键逻辑：创建了ClassPathMapperScanner扫描器，调用了scan方法。

- ClassPathMapperScanner

该类会重新setBeanClass，将自定义Mapper的class设置为MapperFactoryBean，最终会调用其getObject()方法创建对象。
```java
definition.setBeanClass(this.mapperFactoryBeanClass);
```
另外一行关键代码
```java
definition.setAutowireMode(2);  // 根据类型自动注入
```

- MapperFactoryBean
```java
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }
```

#### 自定义命名空间

以数据源DataSource配置为例，现在想把这部分的配置放到一个properties下。

在resources下新建jdbc.properties

```properties
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis
jdbc.username=root
jdbc.password=root
```

在xml中引入

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:property-placeholder location="classpath:jdbc.properties"/>

    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${jdbc.driver}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
    </bean>
</beans>
```

深入研究：自定义命名空间的解析

源码阅读：

1. 找到AbstractApplicationContext，进入其refresh方法。
2. 进入obtainFreshBeanFactory方法。
3. 进入子类AbstractRefreshableApplicationContext的refreshBeanFactory方法。
4. 查看AbstractXmlApplicationContext中的loadBeanDefinitions方法实现。注意loadBeanDefinitions方法有多个重载实现。
5. 进入BeanDefinitionReader中的loadBeanDefinitions方法，查看XmlBeanDefinitionReader的实现，进入doLoadBeanDefinitions方法。
6. 进入registerBeanDefinitions方法。
7. 进入DefaultBeanDefinitionDocumentReader类的doRegisterBeanDefinitions方法。
8. 找到关键代码parseBeanDefinitions。

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element)node;
                    if (delegate.isDefaultNamespace(ele)) {
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }
```

默认命名空间：xmlns="http://www.springframework.org/schema/beans"，其解析对应着方法parseDefaultElement

```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, "import")) {
            this.importBeanDefinitionResource(ele);
        } else if (delegate.nodeNameEquals(ele, "alias")) {
            this.processAliasRegistration(ele);
        } else if (delegate.nodeNameEquals(ele, "bean")) {
            this.processBeanDefinition(ele, delegate);
        } else if (delegate.nodeNameEquals(ele, "beans")) {
            this.doRegisterBeanDefinitions(ele);
        }
    }
```

而像xmlns:context="http://www.springframework.org/schema/context"，其不是默认命名空间，会进入delegate.parseCustomElement方法中。

```java
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        // uri，如http://www.springframework.org/schema/context
        String namespaceUri = this.getNamespaceURI(ele);
        if (namespaceUri == null) {
            return null;
        } else {
            NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
            if (handler == null) {
                this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
                return null;
            } else {
                return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
            }
        }
    }
```

根据url，会获取一个NamespaceHandler。进入resolve方法查看。

```java
	@Nullable
    public NamespaceHandler resolve(String namespaceUri) {
        Map<String, Object> handlerMappings = this.getHandlerMappings();
        Object handlerOrClassName = handlerMappings.get(namespaceUri);
        if (handlerOrClassName == null) {
            return null;
        } else if (handlerOrClassName instanceof NamespaceHandler) {
            return (NamespaceHandler)handlerOrClassName;
        } else {
            String className = (String)handlerOrClassName;

            try {
                Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
                if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                    throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri + "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
                } else {
                    NamespaceHandler namespaceHandler = (NamespaceHandler)BeanUtils.instantiateClass(handlerClass);
                    namespaceHandler.init();
                    handlerMappings.put(namespaceUri, namespaceHandler);
                    return namespaceHandler;
                }
            } catch (ClassNotFoundException var7) {
                throw new FatalBeanException("Could not find NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var7);
            } catch (LinkageError var8) {
                throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var8);
            }
        }
    }
```

handlerMappings即维护了一个URL到NamespaceHandler的映射。

可以看到：http://www.springframework.org/schema/context对应的Handler是ContextNamespaceHandler。

而这部分映射配置数据来源于哪呢？

查看到DefaultNamespaceHandlerResolver下有一个常量

```java
public static final String DEFAULT_HANDLER_MAPPINGS_LOCATION = "META-INF/spring.handlers";
```

即这部分处理器配置在META-INF/spring.handlers文件中。

查看ContextNamespaceHandler

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
    public ContextNamespaceHandler() {
    }

    public void init() {
        this.registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
        this.registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
        this.registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
        this.registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
        this.registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
        this.registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
        this.registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
        this.registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
    }
}
```

可以看到每个标签都有一个BeanDefinitionParser，定义了parse方法。每个Parser重写了doParse方法。

此外，

http://www.springframework.org/schema/context

http://www.springframework.org/schema/context/spring-context.xsd

这两行配置定义的是context标签的schema，并不是真正的URL，而是与本地路径的META-INFO/spring.schemas下的配置相对应。

```text
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context.xsd
```

实战：通过指定一个标签，向Spring容器中自动注入一个BeanPostProcessor

1. 先在resources下的com/alipay/demo/onion下编写一个onion.xsd文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<xsd:schema xmlns="http://com.alipay.demo/onion"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="http://com.alipay.demo/onion">
    <xsd:element name="annotation-driven"></xsd:element>
</xsd:schema>
```

2. 然后定义spring.schemas文件

```text
http\://com.alipay.demo/onion/onion.xsd=com/alipay/demo/onion/onion.xsd
```

3. 在com.alipay.demo.onion下定义OnionNamespaceHandler

```java
public class OnionNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        this.registerBeanDefinitionParser("annotation-driven", new OnionBeanDefinitionParser());
    }
}
```

4. 定义OnionBeanDefinitionParser类，注册一个自定义的BeanPostProcessor，重写beforeInitialization和afterInitialization，初始化前后打印字符串before和after。

```java
public class OnionBeanDefinitionParser implements BeanDefinitionParser {
    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        BeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClassName("com.alipay.demo.onion.OnionBeanPostProcessor");
        parserContext.getRegistry().registerBeanDefinition("onionBeanPostProcessor", beanDefinition);
        return beanDefinition;
    }
}
```

5. 在spring.handlers中配置url和handler的映射

```text
http\://com.alipay.demo/onion=com.alipay.demo.onion.OnionNamespaceHandler
```

至此，自定义命名空间与Spring整合已完成。

测试是否整合成功：

1. 在spring.xml中配置自定义的namespace

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:onion="http://com.alipay.demo/onion"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://com.alipay.demo/onion
       http://com.alipay.demo/onion/onion.xsd">
    <!--把对象的创建交给spring来管理-->
    <onion:annotation-driven></onion:annotation-driven>
    <bean id="userService" class="com.alipay.demo.onion.UserService"></bean>
</beans>
```

2. 创建一个ClassPathXmlApplicationContext，指定spring.xml，通过context获取UserService的bean，查看是否会正确打印before和after。







### SpringBoot