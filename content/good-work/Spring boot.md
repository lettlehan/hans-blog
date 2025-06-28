# Spring Boot 核心知识点

![Spring Boot Logo](https://spring.io/img/spring-by-vmware.svg)

> Spring Boot 让创建独立的、生产级的基于Spring的应用变得容易，"约定优于配置"的思想贯穿始终。

---

## 1. 核心启动流程（源码级）

Spring Boot 应用启动过程是一个复杂而精密的过程，主要分为初始化和运行两个阶段。

### 1.1. 初始化 SpringApplication

当我们执行 `SpringApplication.run()` 方法时，首先会创建 `SpringApplication` 实例：

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class[]{primarySource}, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return (new SpringApplication(primarySources)).run(args);
}
```

在构造方法中，主要完成以下初始化工作：

- **配置基本的环境变量**
- **准备必要的资源**
- **初始化构造器**
- **注册监听器**

这个初始化阶段为后续运行 `SpringApplication` 实例做好了准备。

具体的初始化过程如下：

```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断应用类型（SERVLET、REACTIVE 或 NONE）
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    /**
     * 加载初始化器
     * 从 spring.factories 文件中找出 Key 为 ApplicationContextInitiallizer 的类并实例化
     * 并设置到 SpringApplication的 initiallizer 属性中
     */
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    /**
     * 加载监听器
     * 从 spring.factories 文件中找出 Key 为 ApplicationListener 的类并实例化
     * 监听器设置到 SpringApplication的 listeners 属性中
     */
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 获取启动类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

初始化过程中的关键步骤说明：

1. **推断应用类型**：根据类路径判断是Web应用还是普通应用
2. **加载初始化器**：用于在容器刷新前执行一些初始化工作
3. **加载监听器**：用于监听启动过程中的各种事件
4. **获取启动类**：确定主应用类，用于日志显示等

### 1.2. 执行 run 方法

`run` 方法是 Spring Boot 应用启动的核心，它完成了从初始化到最终运行的全过程：

```java
public ConfigurableApplicationContext run(String... args) {
    // 记录启动时间
    long startTime = System.nanoTime();
    // 创建引导上下文
    DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
    ConfigurableApplicationContext context = null;
    // 配置 Headless 属性（用于无显示器环境）
    this.configureHeadlessProperty();
    // 获取 SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    // 发布 ApplicationStartingEvent 事件
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        // 封装命令行参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        this.configureIgnoreBeanInfo(environment);
        // 打印 banner
        Banner printedBanner = this.printBanner(environment);
        // 创建应用上下文
        context = this.createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        // 准备上下文
        this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 核心：刷新容器，触发 bean 的加载、初始化
        this.refreshContext(context);
        // 刷新后的操作
        this.afterRefresh(context, applicationArguments);
        // 计算启动耗时
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
        }

        // 发布 ApplicationStartedEvent 事件
        listeners.started(context, timeTakenToStartup);
        // 执行 ApplicationRunner、CommandLineRunner
        this.callRunners(context, applicationArguments);
    } catch (Throwable var12) {
        this.handleRunFailure(context, var12, listeners);
        throw new IllegalStateException(var12);
    }

    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        // 发布 ApplicationReadyEvent 事件
        listeners.ready(context, timeTakenToReady);
        return context;
    } catch (Throwable var11) {
        this.handleRunFailure(context, var11, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var11);
    }
}
```

**run方法执行流程**：

1. **创建引导上下文**：为应用启动准备初始环境
2. **准备环境**：加载配置文件，处理命令行参数
3. **创建应用上下文**：根据应用类型创建对应的上下文
4. **刷新容器**：实例化并初始化所有单例Bean
5. **执行Runners**：执行所有ApplicationRunner和CommandLineRunner
6. **发布就绪事件**：通知应用已完全启动并就绪

---

## 2. Spring Bean 生命周期与循环依赖

### 2.1. Bean 的生命周期

Bean 的生命周期是 Spring 框架中的核心概念，包含以下主要阶段：

1. **实例化（Instantiation）**
   - 调用构造方法创建对象
   - 使用反射或CGLIB

2. **属性赋值（Populate）**
   - 设置依赖属性
   - 注入依赖对象

3. **初始化（Initialization）**
   - `BeanNameAware.setBeanName()`
   - `BeanFactoryAware.setBeanFactory()`
   - `ApplicationContextAware.setApplicationContext()`
   - `@PostConstruct`
   - `InitializingBean.afterPropertiesSet()`
   - 自定义 init-method

4. **使用（In Use）**
   - Bean 可以被应用程序使用

5. **销毁（Destruction）**
   - `@PreDestroy`
   - `DisposableBean.destroy()`
   - 自定义 destroy-method

### 2.2. 循环依赖解决方案

Spring 通过三级缓存机制解决循环依赖：

| 缓存 | 用途 | 内容 |
|------|------|------|
| 一级缓存 | 存放完全初始化好的 Bean | `singletonObjects` |
| 二级缓存 | 存放原始的 Bean 对象 | `earlySingletonObjects` |
| 三级缓存 | 存放 Bean 工厂对象 | `singletonFactories` |

> 注意：Spring 只能解决单例作用域下的 setter 注入的循环依赖。

## 3. SpringBoot 的 Listener 机制与事件

### 3.1. 核心事件

| 事件类型 | 触发时机 | 用途 |
|----------|----------|------|
| `ApplicationStartingEvent` | 启动开始 | 进行早期初始化 |
| `ApplicationEnvironmentPreparedEvent` | 环境准备完成 | 配置环境变量 |
| `ApplicationContextInitializedEvent` | 上下文初始化 | 初始化操作 |
| `ApplicationPreparedEvent` | Bean 定义加载完成 | 预处理 |
| `ApplicationStartedEvent` | 上下文刷新完成 | 启动后处理 |
| `ApplicationReadyEvent` | 应用准备就绪 | 最终处理 |
| `ApplicationFailedEvent` | 启动失败 | 失败处理 |

### 3.2. 自定义监听器

```java
@Component
public class MyListener implements ApplicationListener<ApplicationStartedEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        // 处理逻辑
    }
}
```

## 4. Spring 的 SPI 机制

### 4.1. Spring SPI vs JDK SPI

| 特性 | Spring SPI | JDK SPI |
|------|------------|----------|
| 配置文件位置 | META-INF/spring.factories | META-INF/services/ |
| 加载方式 | SpringFactoriesLoader | ServiceLoader |
| 实例化时机 | 按需加载 | 全部加载 |
| 扩展性 | 更灵活，支持key-value | 只支持接口-实现类 |

### 4.2. Spring SPI 示例

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

## 5. SpringBoot 常用注解

### 5.1. 核心注解

- `@SpringBootApplication`
- `@Configuration`
- `@ComponentScan`
- `@EnableAutoConfiguration`

### 5.2. 依赖注入

- `@Autowired`
- `@Resource`
- `@Qualifier`
- `@Value`

### 5.3. Bean 配置

- `@Component`
- `@Service`
- `@Repository`
- `@Controller`
- `@RestController`

### 5.4. AOP 相关

- `@Aspect`
- `@Pointcut`
- `@Before`
- `@After`
- `@Around`

## 6. SpringBoot 优雅停机

### 6.1. 停机流程

1. **接收停机信号**
   - 处理 SIGTERM 信号
   - 触发 ContextClosedEvent

2. **停止接收新请求**
   - 关闭 Web 容器
   - 等待当前请求处理完成

3. **关闭应用上下文**
   - 销毁所有 Bean
   - 释放资源

### 6.2. 配置示例

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 6.3. 优雅停机实现

```java
@Configuration
public class GracefulShutdownConfig {
    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }
}
```

> 注意：优雅停机需要 Spring Boot 2.3+ 版本支持。
