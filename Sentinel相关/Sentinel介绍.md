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
