Filter 过滤器动态拦截请求（request）或响应（response）以转换或使用请求或响应中包含的信息。过滤器本身通常不会创建响应，而是提供可以“附加”到任何一次 RPC 请求的通用函数。Dubbo Filter 是可插拔的，我们可以在一次 RPC 请求中插入任意类型的、任意多个 Filter。
Filter 工作原理如下图所示：
![](https://cn.dubbo.apache.org/imgs/v3/tasks/framework/filter.png)
可以通过 Filter 实现的一些典型能力如下：
- 记录请求参数、响应结果等到日志文件
- 为 RPC 请求添加认证或校验逻辑
- 在发送或执行请求之前，格式化请求体或 header 参数
- 压缩响应结果
- 对请求数据进行埋点，统计调用耗时、成功、失败次数等
- 监测并发执行的请求数量，实现限流降级能力

```java
@SPI(scope = ExtensionScope.MODULE)
public interface Filter extends BaseFilter {}

@SPI(scope = ExtensionScope.MODULE)
public interface ClusterFilter extends BaseFilter {}
```

基于以上 BaseFilter 定义，Dubbo 定义了两个 SPI 接口：ClusterFilter 与 Filter。这两个 SPI 实现能实现的效果基本是一致的，之所以定义两个主要是出于性能优化考虑，建议用户关注 Filter SPI 即可，仅在有严苛性能需求的情况下（如集群 provider 提供者实例数量庞大）才关注 ClusterFilter
![](https://cn.dubbo.apache.org/imgs/v3/tasks/framework/cluster-filter.png)

## 1、实现FIlter接口
```java
@Activate(group = PROVIDER)
public class AccessLogFilter implements Filter {}
```
## 2、注册配置
在`resources/META-INF/dubbo/org.apache.dubbo.rpc.Filter`文件中添加如下配置：

```properties
accesslog=org.apache.dubbo.samples.extensibility.filter.provider.AccessLogFilter
```
## 3、全局关闭开启Filter
- 关闭
```yaml
dubbo:
  consumer:
    filter: "-accesslog,-tps"
```
- 开启
```yaml
dubbo:
  consumer:
    filter: "accesslog,tps"
```
## 4、指定服务关闭开启Filter
- 关闭
```java
@DubboReference(filter="-accesslog,-tps")
private DemoService demoService;
```
- 开启
```java
@DubboReference(filter="accesslog,tps")
private DemoService demoService;
```

## 5、provider与consumer配置区别
- provider，提供者，接收方
- consumer，消费者，调用方
以日志traceId透传为例
- consumer端
```java
String traceId = MDC.get("traceId"); 
// 没有就随机生成一个
RpcContext.getClientAttachment().setAttachment("traceId", traceId);
```
- provider端
```java
String traceId = RpcContext.getServerAttachment().getAttachment("traceId"); MDC.put("traceId", traceId);
```
