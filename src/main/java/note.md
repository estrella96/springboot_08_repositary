# Spring Boot 原理

## 启动运行流程
- 创建SpringApplication对象
```java

public SpringApplication(ResourceLoader resourceLoader, Class... primarySources) {
//保存主配置类        
    this.sources = new LinkedHashSet();
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.addConversionService = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = new HashSet();
        this.isCustomEnvironment = false;
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
//从类路径下找到META-INF/spring.factories配置的所有ApplicationContextInitializer 然后保存起来     
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
//   从类路径下找到META-INF/spring.factories配置的所有ApplicationListener     
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
       //从多个配置类中找到有main方法的主配置类
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }

```
- 运行run方法
```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        
        //获取SpringApplicationRunListeners 从从类路径下META-INF/spring.factories获取
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        //回调所有的SpringApplicationRunListeners.starting方法
        listeners.starting();

        Collection exceptionReporters;
        try {
            
            //封装命令行参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //准备环境
            ConfigurableEnvironment environment = this.prepareEnvironment(
                    listeners, applicationArguments);
                    //创建环境完成后回调SpringApplicationRunListeners.environmentPrepared() 表示环境准备完成
                    
            
            this.configureIgnoreBeanInfo(environment);
            
            
            Banner printedBanner = this.printBanner(environment);
            //创建ApplicationContext 决定创建web的IOC还是普通的IOC
            context = this.createApplicationContext();
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            //准备上下文环境 将environment保存到IOC中 而且applyInitializers()
            //applyInitializers(): 回调之前保存的所有的ApplicationContextInitializer的initialize方法
            //回调所有的SpringApplicationRunListener的contextPrepared()方法
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //prepareContext的最后回调所有的SpringApplicationRunListener的contextLoading()方法
            
            //刷新容器：IOC初始化 如果是web应用会创建嵌入的tomcat
            this.refreshContext(context);
            //从IOC容器中获取所有的ApplicationRunner（先）和CommandLineRunner（后）进行回调 
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            listeners.running(context);
            //整个SpringBoot应用启动完成后 返回启动的IOC容器
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }

    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        listeners.environmentPrepared((ConfigurableEnvironment)environment);
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
            environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
    }

```
## 事件监听机制

- 重要的事件回调机制
    ApplicationContextInitializer 需要配置 META-INF/spring.factories
    SpringApplicationRunListener  需要配置 META-INF/spring.factories
    ApplicationRunner ioc容器中 @Component
    CommandLineRunner ioc容器中

## 自定义starter
- starter
    1 这个场景需要使用到的依赖是什么
    2 如何编写自动配置 
    3 自动配置类要放在META-INF/spring.factories
```properties

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\

org.springframework.boot.autoconfigure.aop.AopAutoConfiguration=\
  
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration=\
  
```        
```java
// /Users/lyj1996/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.1.7.RELEASE/spring-boot-autoconfigure-2.1.7.RELEASE.jar!/org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration.class
@Configuration //指定这个类是一个配置类
@ConditionalOnXXX//指定条件成立的情况下自动配置类生效
@AutoConfigureOrder(-2147483638) //指定自动配置类的顺序
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {
    public static final String DEFAULT_PREFIX = "";
    public static final String DEFAULT_SUFFIX = "";
    private static final String[] SERVLET_LOCATIONS = new String[]{"/"};

    public WebMvcAutoConfiguration() {
    }

    @Bean //给容器中添加组件
    @ConditionalOnMissingBean({HiddenHttpMethodFilter.class})
    @ConditionalOnProperty(
        prefix = "spring.mvc.hiddenmethod.filter",
        name = {"enabled"},
        matchIfMissing = true
    )
    public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
        return new OrderedHiddenHttpMethodFilter();
    }
}
```
   4 模式
    启动器：只用来做依赖导入 xxxx-spring-boot-starter
    专门写自动配置模块 启动器依赖自动配置
    
    