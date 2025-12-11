---
title: 深入了解SpringBoot自动装配原理
date: 2024-11-04 09:25:48
tags:
  - spring
  - Annotation
  - springBoot
  - BeanFactory
categories: 编程
---

# Spring And SpringBoot
## 序言

> - 本文梳理了 `SpringApplication` 启动流程的原理，并分析了入口类返回的 IOC 容器的作用，特别是自动装配的过程。通过对关键组件如 `@Import` 注解、`BeanFactory` 和调试流程的理解，能够更加深入地了解 Spring Boot 应用的启动过程及其背后的工作机制。
> - `SpringApplication` 返回容器的过程中，自动装配(读取加载注入刷新)是关键的工作，只有在所有工作都完成后，容器才会最终返回
## Spring启动预览
- 模拟SpringApplication主类: 通过下面的例子让你快速理解Spring启动加载原理
```java
@SpringBootApplication
@EnableConfigurationProperties
public class testSpringApplicant {
    public static void main(String[] args) {
        // 1.启动Spring应用
        SpringApplication app = new SpringApplication(bootClass.class);
        app.run(args);  // 启动 Spring Boot 应用并初始化容器

        // 2.使用 AnnotationConfigApplicationContext 初始化 IOC 容器
        ApplicationContext context = new AnnotationConfigApplicationContext();

        // 3.手动注册一个 Bean 示例
        ConfigurableListableBeanFactory beanFactory = new DefaultListableBeanFactory();         // 工作bean工厂: 不需要IOC容器来做事  注册单例 
        beanFactory.registerSingleton("bean", new Bean()); 

        // 4.刷新容器
        ((AnnotationConfigApplicationContext) context).refresh(); 
        
        // 5.等所有准备工作都做完了,可以拿到run方法返回的一个IOC容器
        ConfigurableApplicationContext returnValue = app.run(args);
//returnValue应该是ConfigurableApplicationContext的实现AnnotationConfigApplicationContext
    }
}
```
### SpringApplication类解析
- 把所有的组件、环境配置、初始化逻辑整合起来，确保应用程序顺利启动。
- 这个类既是入口类,又是结束类,关系到IOC容器的生命周期,也关系到整个程序的启动与停止
```java
// 最外层
public static void main(String[] args) {
        SpringApplication.run(ProjectTestApplication.class, args);
    } 
```

```java
// 第二层
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class[]{primarySource}, args);
    }
```

```java
// 第三层: 实例化一个SpringApplication实例,执行run方法(不是静态的run方法)
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }
```

```java
// 最里层: 逻辑层->程序的启动流程
public ConfigurableApplicationContext run(String... args) {
		//创建 DefaultBootstrapContext 对象，它代表了应用程序的引导上下文。引导上下文通常是 Spring Boot 启动时的早期阶段，用于初始化一些关键组件。
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
    	// 初始化容器对象
        ConfigurableApplicationContext context = null;
        this.configureHeadlessProperty();
    	// 监听 Spring Boot 应用程序启动生命周期的监听器。
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
    	// 开启监听器
        listeners.starting(bootstrapContext, this.mainApplicationClass);
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 准备应用程序的环境配置
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            // 打印横幅
            Banner printedBanner = this.printBanner(environment);
            // AnnotationConfigApplicationContext IOC容器
            context = this.createApplicationContext();
            // 开启监听器
            context.setApplicationStartup(this.applicationStartup);
            // 准备应用程序的上下文（context),并执行一些初始化任务
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 刷新 Spring 上下文  触发 BeanFactory 的加载和初始化，容器会开始准备好为应用程序提供服务。
            this.refreshContext(context);
            // 刷新后的一些后处理逻辑，通常用于执行一些应用程序启动后的任务。
            this.afterRefresh(context, applicationArguments);
            // 标记应用程序的启动过程已经完成。
            startup.started();
            // 返回配置好的 ConfigurableApplicationContext 实例。该上下文是整个应用程序的核心容器，所有的 Spring Beans 都会在其中管理。
            return context;
        }
```
## IOC容器
- IOC容器的实现类`AnnotationConfigApplicationContext` 实现了`AnnotationConfigRegistry` 继承了`GenericApplicationContext`
```java
  public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry

  // 获取Bean名单的方式: 通过register和scan
  void scan(String… basePackages); // 加载需要扫描的包路径@Compoent
  void register(Class<?>… componentClasses); // 加载手动注册的注解配置类@Configuration
```
### IOC容器实现类有两个方法专门用来扫描Bean
- **void scan  -> 扫描组件**
	- 所有方法自动添加@bean
	- 通过@ComponentScan扫描带有注解@Component @Controller @Service @Repository(dao层专用,spring框架下) @Mapper(Mybatis专用)  的类
- **void register -> 扫描配置类**
	- 需手动添加@bean
	- @Configuration 下带有@Bean注解的方法
### IOC容器执行Refresh方法加载注册Bean流程
  - 加载bean到上下文中(包括解析Bean定义、创建Bean实例、解决依赖关系、初始化Bean以及销毁Bean)
  - 执行依赖注入
  - 标注已刷新,完成工作
### 了解@Import注解: 加载Bean名单(自动装配)
- @Import注解是spring第三种定义bean的方法, **支持对第三方包的导入**
- 共有三种实现方法:
  1. 基础数组: 注解形式通过反射`Class.forName()`把bean交给IOC容器
    ```java
    // 这些在容器中bean名称是该类的全类名 ，比如com.spring.ImBean
    @Import({ 类名.class , 类名.class... })
    public class TestDemo {
    }
    ```
  2. 实现`ImportSelect`接口:这种方式也是Spring的用法, 返回值就是我们实际上要导入到容器中的组件全限定类名(Bean名单)【**重点** 】
    ```java
    public class Myclass implements ImportSelector {
    // 重写导入Bean名单方法
        @Override
        public String[] selectImports(AnnotationMetadata annotationMetadata) {
            return new String[]{"com.spring.ImBean", "com.spring.ImBean2"};
        }
    }
    ```
  3. 实现`ImportBeanDefinitionRegister`接口: 跟上面的实现方法接近, 区别在于能修改bean名
    ```java
    public class Myclass2 implements ImportBeanDefinitionRegistrar {
        @Override
        public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
            //指定bean定义信息（包括bean的类型、作用域...）
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(ImBean.class);
            //注册一个bean指定bean名字（id）
        beanDefinitionRegistry.registerBeanDefinition("reNameBean",rootBeanDefinition);
        }
    }
    ```
## 自动装配分析(重点)
- 首先`@SpringBootApplication`启动类注解带有3个关键注解
	- 一个就是下文提到的`@EnableAutoConfiguration`, 用于自动装配第三方Bean依赖
	- `@ComponentScan`用于扫描`ClassPath`下的本地Bean
	- `@SpringBootConfiguration`注解嵌套了一个`@Configuration`, 实际上它也是一个配置类
- **@EnableAutoConfiguration**启用Spring Boot的自动配置机制，将第三方依赖bean注入IOC容器
- 通过`SpringFactoriesLoader`从类路径下去读取`META-INF/spring.factories`文件信息，此文件中有一个key为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`，定义了一组需要自动配置的bean名单 如下:
- [x] ConfigurationPropertiesAutoConfiguration 
- [x] AOPAutoConfiguration
- [x] DataSourceAutoConfiguration
- [x] ElasticsearchAutoConfiguration
- [x] RedisAutoConfiguration
- [x] GsonAutoConfiguration
- [x] HttpMessageConvertersAutoConfiguration
- [x] JacksonAutoConfiguration
- [x] DispathcherAutoConfiguration
- [x] WebMvcAutoConfiguration
### 具体的实现流程
- `@EnableAutoConfiguration`只是一个引导注解, 真正实现加载bena名单功能的是实现类**AutoConfigurationImportSelector**
- import注解将`AutoConfigurationImportSelector`这个伪配置类交给IOC
```java
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})  // 先是通过import第一种用法:基础数组
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```
1. 通过`getCandidateConfigurations`去加载配置文件的bean全类名名单, 
2. 由`getAutoConfigurationEntry`做名单过滤, 比如像判断自动转配开关是否打开, 是否设置了排除项, 然后把名单设置到类的成员变量中, 
3. 由`selectImports`方法去拿到成员变量名单去执行加载Bean的操作, 加载Bean是通过实现import接口重写`selectImports`方法实现的
- 其中加载的配置文件地址  采取逐行读取的方式`readline()`
	- 旧地址: META-INF/spring.factories(兼容旧版本) 
	- 新地址: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. 
```java
// DeferredImportSelector实现了了ImportSelect, AutoConfigurationImportSelector实现了DeferredImportSelector
public class AutoConfigurationImportSelector implements DeferredImportSelector{

    // 用于返回空bean名单的空数组
    private static final String[] NO_IMPORTS = new String[0]; 

    // 这个才是存储Bean数组和排除的Bean数组的底层对象,里面是List和Set
    private static final AutoConfigurationEntry EMPTY_ENTRY = new AutoConfigurationEntry();

    // 关于底层对象的定义
    protected static class AutoConfigurationEntry {
        private final List<String> configurations;  // 存储bean名单的底层容器
        private final Set<String> exclusions;       // 排除项名单
    }
    
    // 作用: 得到bean名单 全类名 
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 要加载的接口类型  类加载器类型
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader()); 
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        // 返回bean名单
        return configurations;
    }
}
    
    // 作用: 将得到的bean名单进一步啥筛选
    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        // 判断自动装配开关是否打开: 默认spring.boot.enableautoconfiguration=true
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            // 返回bean名单
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes); 
            // 去重操作: 一般为0
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            // 检查排除项
            this.checkExcludedClasses(configurations, exclusions); 
            // 去除排除项
            configurations.removeAll(exclusions);
            // 过滤器过滤bean 逐个检查每个配置类的条件（如@ConditionalOnClass, @ConditionalOnMissingClass)
            configurations = this.getConfigurationClassFilter().filter(configurations);
            // 监听器用在自动配置类被导入到 ApplicationContext 之前，执行一些自定义的处理逻辑
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            // 返回底层对象
            return new AutoConfigurationEntry(configurations, exclusions); 
        }
    }

    // 作用:获取所有符合条件的类[]的全限定类名,判断和准备要导入的自动配置类,交给IOC刷新
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) { 
        if (!this.isEnabled(annotationMetadata)) {  
	        // 再次判断自动装配开关是否打开 如果关 则返回空数组
            return NO_IMPORTS;  
        } else {
            AutoConfigurationEntry autoConfigurationEntry = 				 this.getAutoConfigurationEntry(annotationMetadata);	
            return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations()); 
        }
    }
```
- 注意
	- 调用`getCandidateConfigurations()`方法读取外部文件这个操作是**在IOC容器的Refresh()方法中触发的**
	- 也就是说，在IOC容器启动的时候通过调用`getCandidateConfigurations()`方法把外部文件中指定的类读取进来，然后再使用反射机制(CGLIB或者JDK代理)将它们实例化成为Bean对象载入到IOC容器中
![17307293786901730729377814.png|700x440](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307293786901730729377814.png)
## Bean的生命周期
### Bean的执行流程
**核心金句: 在refresh方法内完成 -> *读取加载注入刷新*** 
- 获取bean名单      包括本地bean和第三方依赖bean
- 加载类                  CGLIB代理或者是反射Instance
- 注入类                  依赖注入一般在所有bean都加载完成以后
- 初始化和销毁       调用@PostConstruct、InitializingBean的afterPropertiesSet方法等，以保证Bean的状态正常
`这四个阶段并非严格独立，但可以理解为顺序执行的一个过程。每个阶段都会依次触发特定的操作，以确保Bean的正确注册、加载和注入，从而构建一个完整的IOC容器
### BeanFactory
-  自定义Bean工厂的核心: **继承了一个抽象模板类, 实现了基础性接口**
- 使用到了**工厂设计模式**
    ```java
    public class testBeanFactory extends AbstractApplicationContext implements BeanFactory {
        private ConfigurableListableBeanFactory beanFactory; 
        @Override
        protected void refreshBeanFactory() {  // IOC核心核心方法,读取加载注入刷新
            // 继承自AbstractBeanFactory的实现类
            beanFactory = new DefaultListableBeanFactory(); 
            // 调用BeanFactory的注册单例bean方法
            beanFactory.registerSingleton("ImBean", new ImBean());
            System.out.println(beanFactory.containsBean("ImBean"));
        }
    }
    ```
  - `ApplicationContext`:是 `BeanFactory` 的一个子接口，扩展了其功能,项目启动就加载bean,是提前加载
![17307676058781730767605785.png|700x600](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307676058781730767605785.png)

![17307841307381730784130182.png|700x180](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307841307381730784130182.png)

![17321672134681732167213381.png|700x259](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17321672134681732167213381.png)
### BeanPostProcessor
- Bean后置处理器, 用来做bean加载完成后的增强操作: 如转换bean
- 利用后置处理器(实例化, 初始化): 记录日志, 修改bean的属性方法, 返回另一个bean
```java
@Component
public class beanUtil implements ApplicationContextAware, BeanPostProcessor {

    @Autowired
    private ApplicationContext applicationContext;


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        if (bean instanceof interface2Abstract) {
            ((interface2Abstract) bean).setMessage("bean被拦截了");
        }
        return bean;
    }

    public Object getBean(String beanName) {
        return applicationContext.getBean(beanName);
    }

    public boolean ContainsBean(String beanName) {
        return applicationContext.containsBean(beanName);
    }
}
```
## Aware接口
- 例如ApplicationContextAware: 允许通过setter方法注入了ApplicationContext实例
- Spring 会在容器初始化该 bean 的时候调用 `setApplicationContext(ApplicationContext context)` 方法，并将当前的 `ApplicationContext` 实例传递给该 bean。因此，`ApplicationContext` 会自动注入到实现了 `ApplicationContextAware` 的类中，而无需显式使用 `@Autowired` 注解。
```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```
## SpringMVC执行流程
![17601617070631760161706121.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17601617070631760161706121.png)
## 思考
- 为什么能在主类底下创建Bean:?
	- 因为有@SpringBootConfiguration注解 里层又嵌套了一个@Configuration, 表明主类也是一个配置类, 程序会在启动时自动加载配置类的bean
- 如何保证Bean线程安全
	- 不使用无状态的成员变量
	- 加锁: 利用 `synchronized` 或 `ReentrantLock`
	- 使用`ThreadLocal`替代
- 为什么一个程序中只需要一个bean工厂?
	1. 便于对bean进行统一管理,而且ApplicationContext相比较底层的BeanFactory有更多的拓展功能: 支持事件发布与监听(记录日志), 而且是BeanFactory的子接口,具有BeanFactory的所有功能
	2. 保持单例, 避免重复bean的载入
	3. 内存占用和性能的角度
	4. Spring 的设计理念是通过简化配置来减少开发者的负担。通过一个统一的 `ApplicationContext`，Spring 实现了自动化和集中化管理，使得开发者无需手动管理多个工厂实例
- @Bean是怎么实现注入且单例的?
	- `@Configuration` 注解的类在 Spring 容器启动时会被 CGLIB 代理，目的是确保每个 `@Bean` 方法在被调用时，返回的是同一个 Bean 实例，而不是每次都创建新的实例
- 如果一个 Bean 被标记为懒加载，那么它不会在 Spring IoC 容器启动时立即实例化?
	- 首先 Spring 会去创建 A 的 Bean，创建时需要注入 B 的属性；
	- 由于在 A 上的 B 属性标注了 `@Lazy` 注解，因此 Spring 会去创建一个 B 的代理对象，将这个代理对象注入到 A 中的 B 属性；
	- 之后开始执行 B 的实例化、初始化，在注入 B 中的 A 属性时，此时 A 已经创建完毕了，就可以将 A 给注入进去。

- `spring.factories`中这么多配置，每次启动都要全部加载么？
	- 不会, 有过滤器进行@Condition注解的判断: 只有存在某些特定的类才能实现加载
  ```java
    @ConditionalOnClass({ RabbitTemplate.class, Channel.class })
    @EnableConfigurationProperties(RabbitProperties.class)
    @Import(RabbitAnnotationDrivenConfiguration.class)
    public class RabbitAutoConfiguration {
    }
    ```
  - 补充:`SpringCloud`这类注解要常用到条件型注解
    - `@ConditionalOnBean`：当容器里有指定 Bean 的条件下
    - `@ConditionalOnMissingBean`：当容器里没有指定 Bean 的情况下
    - `@ConditionalOnSingleCandidate`：当指定 Bean 在容器中只有一个，或者虽然有多个但是指定首选 Bean
    - `@ConditionalOnClass`：当类路径下有指定类的条件下
    - `@ConditionalOnMissingClass`：当类路径下没有指定类的条件下  
    - `@ConditionalOnProperty`：指定的属性是否有指定的值
    - `@ConditionalOnResource`：类路径是否有指定的值
    - `@ConditionalOnExpression`：基于 `SpEL` 表达式作为判断条件
    - `@ConditionalOnJava`：基于 `Java` 版本作为判断条件
    - `@ConditionalOnJndi`：在 `JNDI` 存在的条件下差在指定的位置
    - `@ConditionalOnNotWebApplication`：当前项目不是 `Web` 项目的条件下
    - `@ConditionalOnWebApplication`：当前项目是 `Web` 项 目的条件下
## Debug
`annotationMetaData`存储的是注解的信息
![注解信息.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307755237431730775521453.png)
location标注的是配置文件所在地  url是对location做了字符处理得来的
![17307755677371730775567276.png|700x159](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307755677371730775567276.png)

![17307762407371730776239863.png|700x230](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307762407371730776239863.png)

![17307769407391730776940482.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307769407391730776940482.png)

![17307779357381730777935451.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307779357381730777935451.png)

![17307780087391730778008460.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307780087391730778008460.png)

![17307782007431730778199872.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307782007431730778199872.png)

![17307788857381730778885604.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307788857381730778885604.png)

![17307834977441730783497666.png|700x362](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17307834977441730783497666.png)

