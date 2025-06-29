## 1. SpringBoot 常用注解

Spring Boot大量使用注解来简化配置和开发，这些注解背后有着复杂的实现机制。

### 1.1. 核心注解源码分析

#### 1.1.1 @SpringBootApplication

```java  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@SpringBootConfiguration  
@EnableAutoConfiguration  
@ComponentScan(excludeFilters = {   
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),  
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })  
public @interface SpringBootApplication {  
    // 排除特定的自动配置类  
    @AliasFor(annotation = EnableAutoConfiguration.class)    Class<?>[] exclude() default {};    // 排除特定的自动配置类名  
    @AliasFor(annotation = EnableAutoConfiguration.class)    String[] excludeName() default {};    // 指定扫描的包  
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")    String[] scanBasePackages() default {};    // 指定扫描的类  
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")    Class<?>[] scanBasePackageClasses() default {};    // 是否代理目标类  
    @AliasFor(annotation = Configuration.class)    boolean proxyBeanMethods() default true;}  
```  

**@SpringBootApplication注解原理**：

1. `@SpringBootConfiguration`：标识这是一个Spring Boot配置类，本质上是`@Configuration`注解
2. `@EnableAutoConfiguration`：启用Spring Boot的自动配置机制
3. `@ComponentScan`：启用组件扫描，自动发现和注册Bean

#### 1.1.2 @EnableAutoConfiguration

```java  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@AutoConfigurationPackage  
@Import(AutoConfigurationImportSelector.class)  
public @interface EnableAutoConfiguration {  
    // 排除特定的自动配置类  
    Class<?>[] exclude() default {};    // 排除特定的自动配置类名  
    String[] excludeName() default {};}  
```  

**AutoConfigurationImportSelector源码分析**：

```java  
public class AutoConfigurationImportSelector implements DeferredImportSelector {  
    @Override    public String[] selectImports(AnnotationMetadata annotationMetadata) {        // 加载自动配置元数据  
        AutoConfigurationMetadata autoConfigurationMetadata =            AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);  
        // 获取自动配置条目  
        AutoConfigurationEntry autoConfigurationEntry =            getAutoConfigurationEntry(autoConfigurationMetadata, annotationMetadata);  
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());    }        protected AutoConfigurationEntry getAutoConfigurationEntry(  
            AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {        // 检查是否启用自动配置  
        if (!isEnabled(annotationMetadata)) {            return EMPTY_ENTRY;        }        // 获取注解属性  
        AnnotationAttributes attributes = getAttributes(annotationMetadata);        // 从spring.factories加载所有自动配置类  
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);        // 去重  
        configurations = removeDuplicates(configurations);        // 获取排除项  
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);        // 检查排除项  
        checkExcludedClasses(configurations, exclusions);        // 移除排除项  
        configurations.removeAll(exclusions);        // 根据条件过滤  
        configurations = getConfigurationClassFilter().filter(configurations);        // 触发自动配置导入事件  
        fireAutoConfigurationImportEvents(configurations, exclusions);        return new AutoConfigurationEntry(configurations, exclusions);    }}  
```  

### 1.2. 依赖注入注解原理

#### 1.2.1 @Autowired

```java  
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Autowired {  
    boolean required() default true;}  
```  

**AutowiredAnnotationBeanPostProcessor源码分析**：

```java  
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor, BeanFactoryAware {  
    // 处理@Autowired注解的核心方法  
    @Override    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {        // 查找注入点  
        InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);        try {            // 执行注入  
            metadata.inject(bean, beanName, pvs);        }        catch (BeanCreationException ex) {            throw ex;        }        catch (Throwable ex) {            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);        }        return pvs;    }    // 查找注入点  
    private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {        // 缓存处理  
        InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);        if (InjectionMetadata.needsRefresh(metadata, clazz)) {            synchronized (this.injectionMetadataCache) {                metadata = this.injectionMetadataCache.get(cacheKey);                if (InjectionMetadata.needsRefresh(metadata, clazz)) {                    // 清除旧的元数据  
                    if (metadata != null) {                        metadata.clear(pvs);                    }                    // 构建新的元数据  
                    metadata = buildAutowiringMetadata(clazz);                    this.injectionMetadataCache.put(cacheKey, metadata);                }            }        }        return metadata;    }}  
```  

#### 1.2.2 @Value

```java  
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Value {  
    String value();}  
```  

**@Value注解处理流程**：

1. 由`AutowiredAnnotationBeanPostProcessor`处理
2. 解析表达式，支持`${property}`和`#{spEL}`格式
3. 通过`PropertySourcesPlaceholderConfigurer`解析属性占位符
4. 通过`SpelExpressionParser`解析SpEL表达式

#### 1.2.3 @Resource和@Qualifier

```java  
@Target({ElementType.TYPE, ElementType.FIELD, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Resource {  
    String name() default "";    // 其他属性...  
}  
  
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
@Documented  
public @interface Qualifier {  
    String value() default "";}  
```  

**依赖注入处理流程**：

1. `@Resource`：
    - 由`CommonAnnotationBeanPostProcessor`处理
    - 优先按名称注入，其次按类型注入

2. `@Qualifier`：
    - 与`@Autowired`配合使用
    - 指定注入Bean的限定符
    - 解决同类型多Bean的注入问题

### 1.3. Bean 配置注解原理

#### 1.3.1 @Component及其派生注解

```java  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Indexed  
public @interface Component {  
    String value() default "";}  
```  

**@Component注解处理流程**：

1. 由`ClassPathBeanDefinitionScanner`扫描带有@Component注解的类
2. 创建对应的`BeanDefinition`
3. 注册到`BeanFactory`中

```java  
// ClassPathBeanDefinitionScanner核心源码  
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();    for (String basePackage : basePackages) {        // 查找候选组件  
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);        for (BeanDefinition candidate : candidates) {            // 解析作用域  
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);            candidate.setScope(scopeMetadata.getScopeName());            // 生成bean名称  
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);            // 处理通用注解  
            if (candidate instanceof AbstractBeanDefinition) {                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);            }            if (candidate instanceof AnnotatedBeanDefinition) {                // 处理@Lazy, @Primary等注解  
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);            }            // 检查名称是否已注册  
            if (checkCandidate(beanName, candidate)) {                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);                // 应用作用域代理模式  
                definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(                        scopeMetadata, definitionHolder, this.registry);                beanDefinitions.add(definitionHolder);                // 注册bean定义  
                registerBeanDefinition(definitionHolder, this.registry);            }        }    }    return beanDefinitions;}  
```  

#### 1.3.2 派生注解

```java  
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Service {  
    @AliasFor(annotation = Component.class)    String value() default "";}  
  
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Repository {  
    @AliasFor(annotation = Component.class)    String value() default "";}  
  
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Controller {  
    @AliasFor(annotation = Component.class)    String value() default "";}  
  
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Controller  
@ResponseBody  
public @interface RestController {  
    @AliasFor(annotation = Controller.class)    String value() default "";}  
```  

### 1.4. AOP 相关注解原理

#### 1.4.1 @Aspect和@Pointcut

```java  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Aspect {  
    String value() default "";}  
  
@Target(ElementType.METHOD)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Pointcut {  
    String value();    String argNames() default "";}  
```  

**AspectJ注解处理流程**：

```java  
// AnnotationAwareAspectJAutoProxyCreator核心源码  
@Override  
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {  
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);    if (advisors.isEmpty()) {        return DO_NOT_PROXY;    }    return advisors.toArray();}  
  
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {  
    // 查找所有候选的Advisor  
    List<Advisor> candidateAdvisors = findCandidateAdvisors();    // 筛选适用于当前bean的Advisor  
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);    extendAdvisors(eligibleAdvisors);    if (!eligibleAdvisors.isEmpty()) {        // 排序  
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);    }    return eligibleAdvisors;}  
```  

#### 1.4.2 通知注解(@Before, @After, @Around等)

```java  
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Before {  
    String value();    String argNames() default "";}  
  
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface After {  
    String value();    String argNames() default "";}  
  
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Around {  
    String value();    String argNames() default "";}  
```  

**通知注解处理流程**：

1. 由`AnnotationAwareAspectJAutoProxyCreator`处理
2. 识别所有的@Aspect注解类
3. 解析切点表达式和通知方法
4. 创建代理对象，拦截方法调用
5. 按照通知类型和优先级执行通知方法