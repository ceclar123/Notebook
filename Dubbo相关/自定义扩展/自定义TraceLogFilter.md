建一个通用模块，每个服务都引用
## 1、创建TraceLogFilter
```java
@Activate(group = {CommonConstants.CONSUMER, CommonConstants.PROVIDER})  
public class TraceLogFilter implements Filter {  
    private static final String TRACE_ID_KEY = "traceId";  
  
    @Override  
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {  
        // 系统的不拦截，否则isConsumerSide 可能会导致NPE  
        if (invoker.getUrl().getServiceInterface().startsWith("org.apache.dubbo")) {  
            return invoker.invoke(invocation);  
        }  
//        RpcContext rpcContext = RpcContext.getServiceContext();  
//        boolean isConsumer = rpcContext.isConsumerSide();  
//        boolean isProvider = rpcContext.isProviderSide();  
  
        URL url = invoker.getUrl();  
        if (url == null) {  
            return invoker.invoke(invocation);  
        }  
  
        String side = url.getSide();  
        boolean isConsumer = CommonConstants.CONSUMER.equals(side);  
        boolean isProvider = CommonConstants.PROVIDER.equals(side);  
  
        if (isConsumer) {  
            handleConsumer(invocation);  
        } else {  
            handleProvider(invocation);  
        }  
  
        try {  
            return invoker.invoke(invocation);  
        } finally {  
            // Provider 侧必须清理，防止线程池复用导致 MDC 污染  
            if (isProvider) {  
                MDC.remove(TRACE_ID_KEY);  
            }  
        }  
    }  
  
    private void handleConsumer(Invocation invocation) {  
        String traceId = MDC.get(TRACE_ID_KEY);  
  
        if (traceId == null || traceId.isEmpty()) {  
            traceId = generateTraceId();  
            MDC.put(TRACE_ID_KEY, traceId);  
        }  
  
        invocation.setAttachment(TRACE_ID_KEY, traceId);  
    }  
  
    private void handleProvider(Invocation invocation) {  
        String traceId = invocation.getAttachment(TRACE_ID_KEY);  
  
        if (traceId == null || traceId.isEmpty()) {  
            traceId = generateTraceId();  
        }  
  
        MDC.put(TRACE_ID_KEY, traceId);  
    }  
  
    private String generateTraceId() {  
        return UUID.randomUUID().toString();  
    }  
}
```

## 注册配置
> 在应用 `resources/META-INF/dubbo/org.apache.dubbo.rpc.Filter` 文件

```
traceLogFilter=org.a.b.c.common.filter.TraceLogFilter
```

## logback启用
> logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<configuration>  
    <!-- 定义日志输出格式，对应你原来的 ConversionPattern -->    <property name="LOG_PATTERN"  
              value="date: [%d{yyyy-MM-dd HH:mm:ss.SSS}] traceId: [%X{traceId}] %logger{36} %thread %5p %logger{2}: %msg%n"/>  
  
    <!-- 控制台输出配置 -->  
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">  
        <encoder>            <!-- 设置字符集，防止中文乱码 -->  
            <charset>UTF-8</charset>  
            <pattern>${LOG_PATTERN}</pattern>  
        </encoder>    </appender>  
    <!-- 根日志级别设置，对应原来的 log4j.rootLogger=info, stdout -->    <root level="INFO">  
        <appender-ref ref="STDOUT"/>  
    </root></configuration>
```

## 可能dubbo配置logger
``` yml
dubbo:  
  application:  
    name: demo-consumer  
    logger: slf4j
```
