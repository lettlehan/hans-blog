## 1. 服务调用核心概念  
  
### 1.1 服务调用模式  
  
```mermaid  
graph TD  
    A[服务调用] --> B[同步调用]  
    A --> C[异步调用]  
    B --> D[HTTP/REST]    B --> E[RPC]    C --> F[消息队列]  
    C --> G[事件驱动]  
```  
  
### 1.2 Spring Cloud 调用组件对比  
  
| 组件         | 类型       | 协议支持       | 负载均衡 | 熔断支持 | 适用场景              |  
|--------------|-----------|----------------|----------|----------|-----------------------|  
| Feign        | 声明式    | HTTP/REST      | 内置     | 支持     | 微服务间REST调用      |  
| RestTemplate | 编程式    | HTTP/REST      | 需配置   | 需集成   | 传统REST调用          |  
| WebClient    | 响应式    | HTTP/REST/WebSocket | 内置     | 支持     | 响应式应用/SSE/WebSocket |  
  
## 2. Feign 深度解析  
  
### 2.1 基础集成  
  
**Maven依赖**：  
```xml  
<dependency>  
    <groupId>org.springframework.cloud</groupId>    <artifactId>spring-cloud-starter-openfeign</artifactId>    <version>3.1.0</version></dependency>  
```  
  
**启动类配置**：  
```java  
@EnableFeignClients(basePackages = "com.example.clients")  
@SpringBootApplication  
public class Application {  
    public static void main(String[] args) {        SpringApplication.run(Application.class, args);    }}  
```  
  
### 2.2 客户端定义  
  
**基础定义**：  
```java  
@FeignClient(  
    name = "inventory-service",    url = "${feign.client.inventory.url:}",  
    configuration = InventoryFeignConfig.class)  
public interface InventoryClient {  
        @GetMapping("/api/inventory/{sku}")  
    Inventory getInventory(@PathVariable String sku);        @PostMapping(  
        value = "/api/inventory",        consumes = MediaType.APPLICATION_JSON_VALUE  
    )    Inventory updateInventory(@RequestBody InventoryUpdateRequest request);}  
```  
  
### 2.3 高级特性  
  
**自定义编解码器**：  
```java  
public class CustomEncoder extends SpringEncoder {  
    public CustomEncoder(ObjectFactory<HttpMessageConverters> messageConverters) {        super(messageConverters);    }        @Override  
    public void encode(Object object, Type bodyType, RequestTemplate template) {        // 自定义编码逻辑  
        super.encode(object, bodyType, template);    }}  
```  
  
**请求拦截器**：  
```java  
public class AuthRequestInterceptor implements RequestInterceptor {  
    @Override    public void apply(RequestTemplate template) {        template.header("Authorization", "Bearer " + getCurrentToken());        template.header("X-Request-ID", UUID.randomUUID().toString());    }}  
```  
  
## 3. RestTemplate 专业指南  
  
### 3.1 最佳实践配置  
  
```java  
@Configuration  
public class RestTemplateConfig {  
  
    @Bean    @LoadBalanced    public RestTemplate restTemplate(RestTemplateBuilder builder) {        return builder            .setConnectTimeout(Duration.ofSeconds(5))            .setReadTimeout(Duration.ofSeconds(10))            .additionalInterceptors(                new LoggingInterceptor(),                new RetryInterceptor(3, 1000L)            )            .requestFactory(() -> {                HttpComponentsClientHttpRequestFactory factory =                    new HttpComponentsClientHttpRequestFactory();  
                factory.setConnectionRequestTimeout(3000);                return factory;            })            .build();    }}  
```  
  
## 4. WebClient 响应式调用  
  
### 4.1 基础使用  
  
```java  
@Service  
public class ProductService {  
        private final WebClient webClient;  
        public ProductService(WebClient.Builder webClientBuilder) {  
        this.webClient = webClientBuilder            .baseUrl("http://product-service")            .build();    }        public Mono<Product> getProduct(String id) {  
        return webClient.get()            .uri("/api/products/{id}", id)            .retrieve()            .bodyToMono(Product.class)            .timeout(Duration.ofSeconds(3))            .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));    }}  
```  
  
## 5. 关键特性实现  
  
### 5.1 重试机制对比  
  
| 特性         | Feign                     | RestTemplate              | WebClient                 |  
|--------------|---------------------------|---------------------------|---------------------------|  
| 配置方式     | 注解/配置文件             | 拦截器                    | 操作符                    |  
| 重试策略     | 固定间隔                  | 自定义                    | 指数退避                  |  
| 条件控制     | 异常类型                  | 响应状态                  | 异常/状态                 |  
| 线程模型     | 同步                      | 同步                      | 异步非阻塞                |  
  
**Feign重试配置**：  
```properties  
feign.client.config.default.retryer=com.example.CustomRetryer  
feign.client.config.default.retryable-exceptions=java.io.IOException  
```  
  
## 6. 性能优化  
  
### 6.1 连接池配置  
  
```yaml  
feign:  
  httpclient:    enabled: true    max-connections: 200    max-connections-per-route: 50    connection-timeout: 2000    time-to-live: 900000  
```  
  
## 7. 安全实践  
  
### 7.1 认证传递  
  
```java  
@Bean  
public OAuth2FeignRequestInterceptor oauth2FeignRequestInterceptor(  
        OAuth2ClientContext oauth2ClientContext,        ClientCredentialsResourceDetails resource) {    return new OAuth2FeignRequestInterceptor(oauth2ClientContext, resource);}  
```  
  
## 8. 生产环境建议  
  
**日志规范**：  
```java  
@Aspect  
@Component  
@Slf4j  
public class FeignLoggingAspect {  
        @Around("@within(org.springframework.cloud.openfeign.FeignClient)")  
    public Object logFeignCall(ProceedingJoinPoint joinPoint) throws Throwable {        long start = System.currentTimeMillis();        try {            Object result = joinPoint.proceed();            log.info("Feign call success - {}: {}ms",                joinPoint.getSignature(),   
                System.currentTimeMillis() - start);  
            return result;        } catch (Exception e) {            log.error("Feign call failed - {}: {}",                joinPoint.getSignature(), e.getMessage());  
            throw e;        }    }}  
```