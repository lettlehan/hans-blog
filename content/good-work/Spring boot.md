# 1. 核心启动流程(源码级)

## 1.1. 初始化 SpringApplication

```
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class[]{primarySource}, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }
```

主要初始化参数，**配置基本的环境变量、资源、构造器、监听器**，初始化阶段的主要作用是为运行`SpringApplication`实例对象启动环境变量准备以及进行必要的资源构造器的初始化动作

```
	public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
	}

	@SuppressWarnings({ "unchecked", "rawtypes" })
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
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

## 1.2. 执行 run 方法

```
    public ConfigurableApplicationContext run(String... args) {
        long startTime = System.nanoTime();
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        ConfigurableApplicationContext context = null;
        // 开启 Java AWT Headless模式
        this.configureHeadlessProperty();
        // 
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        // // 发布 ApplicationStartingEvent
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            // 打印 banner
            Banner printedBanner = this.printBanner(environment);
            // 创建上下文（AnnotationConfigServletWebServerApplicationContext）
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 核心：刷新容器，触发 bean 的加载
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
            }

            // // 发布 ApplicationStartedEvent
            listeners.started(context, timeTakenToStartup);
            // / 执行 ApplicationRunner、CommandLineRunner
            this.callRunners(context, applicationArguments);
        } catch (Throwable var12) {
            this.handleRunFailure(context, var12, listeners);
            throw new IllegalStateException(var12);
        }

        try {
            Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
            // // 发布 ApplicationReadyEvent
            listeners.ready(context, timeTakenToReady);
            return context;
        } catch (Throwable var11) {
            this.handleRunFailure(context, var11, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var11);
        }
    }
```



# 2. spring的bean生命周期，如何解决循环依赖


# 3. springboot的listener机制以及都有哪些event事件


# 4. spring的spi机制，对比idk区别


# 5. springboot常用注解，以及进阶注解(bean装配相关，aop等等)


# 6. sprinqboot停机流程，优雅停机(源码级)

