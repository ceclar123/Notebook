## 资源
只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源
## 规则
围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则以及系统保护规则。所有规则可以动态实时调整

| Rule 类型       | 作用   |
| ------------- | ---- |
| FlowRule      | 限流规则 |
| DegradeRule   | 熔断规则 |
| SystemRule    | 系统保护 |
| ParamFlowRule | 热点参数 |
| AuthorityRule | 权限规则 |
流量控制有以下几个角度:
- 资源的调用关系，例如资源的调用链路，资源和资源之间的关系；
- 运行指标，例如 QPS、线程池、系统负载等；
- 控制的效果，例如直接限流、冷启动、排队等。

### 系统规则有哪几种维度
> `SystemSlot` 在链路前部检查，超限抛 `SystemBlockException`。它保护的是**整个进程/机器**，不是单资源。
1. **Load 自适应**（OS 平均负载，仅 Linux/Unix 有效，mac 用 X 架构取值，Windows 返回 -1 不支持）；
2. **全局平均 RT**（`avgResponseTime`，单位 ms）；
3. **全局入口 QPS**（`qps`，只统计入口资源 `ContextUtil.isCurContextEntry()` 的请求）；
4. **全局并发线程数**（`maxThreadCount`）；
5. **最小服务比**（`minServiceRatio` 1.8.x 预留/特定版本能力，表述时谨慎，可说"高版本引入服务比例保护"）

### ProcessorSlotChain 的默认 Slot 顺序是什么？为什么是这个顺序
> 默认顺序（定义在 `sentinel-core` 的 SPI 文件 `META-INF/services/com.alibaba.csp.sentinel.slotchain.ProcessorSlot`）

```java
NodeSelectorSlot → ClusterBuilderSlot → LogSlot → StatisticSlot →
AuthoritySlot → SystemSlot → ParamFlowSlot → FlowSlot → DegradeSlot
```

## 示例demo
- 引入jar包
```xml
<dependency> 
	<groupId>com.alibaba.csp</groupId> 
	<artifactId>sentinel-core</artifactId> 
	<version>1.8.6</version> 
</dependency>
```
- 具体使用
```java
public static void main(String[] args) {
    // 配置规则.
    initFlowRules();

    while (true) {
        // 1.5.0 版本开始可以直接利用 try-with-resources 特性
        try (Entry entry = SphU.entry("HelloWorld")) {
            // 被保护的逻辑
            System.out.println("hello world");
	} catch (BlockException ex) {
            // 处理被流控的逻辑
	    System.out.println("blocked!");
	}
    }
}

// 支持注解
@SentinelResource("HelloWorld")
public void helloWorld() {
    // 资源中的逻辑
    System.out.println("hello world");
}

// 定义规则
private static void initFlowRules(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    rule.setResource("HelloWorld");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    // Set limit QPS to 20.
    rule.setCount(20);
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```
### SphU.entry 和 SphO.entry 的区别？为什么官方推荐 SphU？
- `SphO`：布尔式
```java
if (SphO.entry("res")) { 
	try{
		//
	} catch(Exception e){
		//
	}
	finally{ 
		SphO.exit(); 
	} 
} else { 
	// fallback 
}
```
容易**忘记 exit**、且不支持带参数的 exit 与上下文传参；1.8.x 里 SphO 已不推荐，新代码一律 SphU；
- `SphU`：异常式
```java
try (Entry e = SphU.entry("res", EntryType.IN, 1, args)) {
	//
} catch (BlockException ex) {
	//
}
``` 
-自动资源管理保证 exit，`entry` 重载支持 traffic 数量、参数列表（热点用）、`EntryType.IN/OUT`（区分入口/出口流量，系统规则只统计 IN）。  
    追问：BlockException 要单独 catch 还是 catch Throwable？——业务异常应让其在 try 内传播，Sentinel 自动记录（1.8.x CtEntry 在 `entry.exit` 时会处理 TraceEntry 的异常）；catch 后必须手动 trace 的场景是 catch 吞掉了异常
## 介入Sentinel 控制台
- 部署好[控制台](https://sentinelguard.io/zh-cn/docs/dashboard.html)
- 客户端引入通信模块
```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.6</version>
</dependency>
```
- 修改启动命令
```shell
-Dcsp.sentinel.dashboard.server=consoleIp:port
```
- 对接阿里云Sentinel控制台
```shell
-Dproject.name=AppName -Dahas.license=<License>
```

## 熔断和降级的区别？Sentinel 里两者是什么关系？

**熔断**是手段：检测到某资源异常/慢调用超阈值，短时间内直接拒绝请求（OPEN），不再真实调用，防止故障扩散与线程占用；
**降级**是目的/结果：熔断后给调用方一个兜底响应（cache、默认值、友好文案）。Sentinel 里"熔断降级规则"（DegradeRule）触发后抛 `DegradeException`（属于 `BlockException` 子类），由 fallback/blockHandler 实现降级语义

## push 模式和 pull 模式的区别？Sentinel 属于哪种？

- pull：客户端定时拉取（如早期 Ribbon/Hystrix 配置）
- push：配置变更后服务端主动通知
- **Sentinel 原生控制台→客户端是 pull**：控制台改规则存自己的内存，客户端通过心跳/控制台拉取模式获取指标、控制台主动下发（`/setRules` 由控制台调用客户端 8719 命令接口，方向是控制台 push 命令）
- **规则数据源（Nacos 等）是 push**（长轮询/Watch 语义）。改造生产版 dashboard 的标准姿势：dashboard 写 Nacos（WritableDataSource），SDK 读 Nacos（ReadableDataSource），彻底解耦控制台——这就是市面上"sentinel-dashboard 改造版"的原理。