## 1. SpringBoot 的 Listener 机制与事件

Spring Boot的事件机制是基于Spring框架的`ApplicationEvent`和`ApplicationListener`接口扩展而来，提供了应用生命周期中各个阶段的事件通知。

### 1.1. 事件发布机制源码分析

Spring Boot事件发布的核心是`SpringApplicationRunListeners`类，它封装了所有的`SpringApplicationRunListener`实现：

```java  
// SpringApplicationRunListeners.java 核心源码  
class SpringApplicationRunListeners {  
    private final List<SpringApplicationRunListener> listeners;        SpringApplicationRunListeners(Collection<? extends SpringApplicationRunListener> listeners) {  
        this.listeners = new ArrayList<>(listeners);    }    // 发布应用启动事件  
    void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {        doWithListeners("spring.boot.application.starting",            (listener) -> listener.starting(bootstrapContext, mainApplicationClass));  
    }    // 发布环境准备事件  
    void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {        doWithListeners("spring.boot.application.environment-prepared",            (listener) -> listener.environmentPrepared(bootstrapContext, environment));    }    // 发布上下文准备事件  
    void contextPrepared(ConfigurableApplicationContext context) {        doWithListeners("spring.boot.application.context-prepared",            (listener) -> listener.contextPrepared(context));  
    }    // 发布上下文加载事件  
    void contextLoaded(ConfigurableApplicationContext context) {        doWithListeners("spring.boot.application.context-loaded",            (listener) -> listener.contextLoaded(context));  
    }    // 发布应用启动完成事件  
    void started(ConfigurableApplicationContext context, Duration timeTaken) {        doWithListeners("spring.boot.application.started",            (listener) -> listener.started(context, timeTaken));  
    }    // 发布应用就绪事件  
    void ready(ConfigurableApplicationContext context, Duration timeTaken) {        doWithListeners("spring.boot.application.ready",            (listener) -> listener.ready(context, timeTaken));  
    }    // 发布应用失败事件  
    void failed(ConfigurableApplicationContext context, Throwable exception) {        doWithListeners("spring.boot.application.failed",            (listener) -> callFailedListener(listener, context, exception));  
    }}  
```  

### 1.2. 核心事件与触发时机

| 事件类型                                  | 触发时机        | 源码位置                         | 用途      |  
|---------------------------------------|-------------|------------------------------|---------|  
| `ApplicationStartingEvent`            | 启动开始        | `SpringApplication.run()` 开始 | 进行早期初始化 |  
| `ApplicationEnvironmentPreparedEvent` | 环境准备完成      | `prepareEnvironment()` 方法中   | 配置环境变量  |  
| `ApplicationContextInitializedEvent`  | 上下文初始化      | `prepareContext()` 方法中       | 初始化操作   |  
| `ApplicationPreparedEvent`            | Bean 定义加载完成 | `prepareContext()` 方法末尾      | 预处理     |  
| `ApplicationStartedEvent`             | 上下文刷新完成     | `refreshContext()` 之后        | 启动后处理   |  
| `ApplicationReadyEvent`               | 应用准备就绪      | `callRunners()` 之后           | 最终处理    |  
| `ApplicationFailedEvent`              | 启动失败        | 捕获异常处理中                      | 失败处理    |  

### 1.3. 事件监听器实现方式

#### 1.3.1 实现ApplicationListener接口

```java  
@Component  
public class MyListener implements ApplicationListener<ApplicationStartedEvent> {  
    @Override    public void onApplicationEvent(ApplicationStartedEvent event) {        // 处理逻辑  
    }}  
```  

#### 1.3.2 使用@EventListener注解

```java  
@Component  
public class AnnotationBasedEventListener {  
    @EventListener    public void handleApplicationStarted(ApplicationStartedEvent event) {        // 处理逻辑  
    }        @EventListener(condition = "#event.source.profiles.contains('dev')")  
    public void handleConditionalEvent(ApplicationEnvironmentPreparedEvent event) {        // 条件处理逻辑  
    }}  
```  

#### 1.3.3 监听器注册源码分析

```java  
// EventPublishingRunListener.java 核心源码  
public class EventPublishingRunListener implements SpringApplicationRunListener {  
    private final SpringApplication application;    private final String[] args;    private final SimpleApplicationEventMulticaster initialMulticaster;  
    public EventPublishingRunListener(SpringApplication application, String[] args) {        this.application = application;        this.args = args;        // 创建事件广播器  
        this.initialMulticaster = new SimpleApplicationEventMulticaster();        // 注册应用中的所有监听器  
        for (ApplicationListener<?> listener : application.getListeners()) {            this.initialMulticaster.addApplicationListener(listener);        }    }        @Override  
    public void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {        // 发布ApplicationStartingEvent事件  
        this.initialMulticaster.multicastEvent(                new ApplicationStartingEvent(bootstrapContext, this.application, this.args));    }    // 其他事件发布方法...  
}  
```  

### 1.4. 自定义事件示例

```java  
// 1. 定义自定义事件  
public class MyCustomEvent extends ApplicationEvent {  
    private final String message;        public MyCustomEvent(Object source, String message) {  
        super(source);        this.message = message;    }        public String getMessage() {  
        return message;    }}  
  
// 2. 发布事件  
@Component  
public class MyEventPublisher {  
    private final ApplicationEventPublisher publisher;        public MyEventPublisher(ApplicationEventPublisher publisher) {  
        this.publisher = publisher;    }        public void publishEvent(String message) {  
        publisher.publishEvent(new MyCustomEvent(this, message));    }}  
  
// 3. 监听事件  
@Component  
public class MyEventListener {  
    @EventListener    public void handleMyCustomEvent(MyCustomEvent event) {        System.out.println("Received custom event: " + event.getMessage());    }}  
```