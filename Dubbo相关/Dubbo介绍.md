# 服务开发
## 1、开发服务
> 提供端实现，接口定义根据需要可以单独抽成一个模块
- 接口定义
```java
public interface DemoService {
    String hello(String arg);
}
```
- 接口实现
```java
@DubboService
public class DemoServiceImpl implements DemoService {
    public String hello(String arg) {
        // put your microservice logic here
    }
}
```
## 2、发布服务
> 在提供端的application.yml中配置

```yaml
dubbo:
  application:
    name: dubbo-demo
  protocol:
    name: dubbo
    port: -1
  registry:
    # address: zookeeper://127.0.0.1:2181
    address: nacos://${nacos.address:127.0.0.1}:8848?username=nacos&password=123456
```
## 3、调用服务
> 首先需要引入相关jar包

```java
@Bean
public class Consumer {
    @DubboReference
    private DemoService demoService;
}
```

# 服务发现
> 支持常用的注册中心如 Nacos、Consul、Zookeeper 等
![](https://cn.dubbo.apache.org/imgs/v3/feature/service-discovery/arc.png)
[参考](https://cn.dubbo.apache.org/zh-cn/overview/what/core-features/service-discovery/)


# 负载均衡
> Dubbo 提供了多种均衡策略，缺省为 `weighted random` 基于权重的随机负载均衡策略（由 Consumer 通过负载均衡算法得出需要将请求提交到哪个 Provider 实例）

| 算法                | 特性                  | 备注                             |
| ----------------- | ------------------- | ------------------------------ |
| Weighted Random   | 加权随机                | 默认算法，默认权重相同                    |
| RoundRobin        | 加权轮询                | 借鉴于 Nginx 的平滑加权轮询算法，默认权重相同，    |
| LeastActive       | 最少活跃优先 + 加权随机       | 背后是能者多劳的思想                     |
| Shortest-Response | 最短响应优先 + 加权随机       | 更加关注响应速度                       |
| ConsistentHash    | 一致性哈希               | 确定的入参，确定的提供者，适用于有状态请求          |
| P2C               | Power of Two Choice | 两次随机选择两个节点后，继续选择“连接数”较小的那个节点。  |
| Adaptive          | 自适应负载均衡             | 在 P2C 算法基础上，选择二者中 load 最小的那个节点 |
## 1、加权随机
- **加权随机**，按权重设置随机概率
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重
- 缺点：存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上

## 2、加权轮询
- **加权轮询**，按公约后的权重设置轮询比率，循环调用节点
- 缺点：同样存在慢的提供者累积请求的问题
加权轮询过程中，如果某节点权重过大，会存在某段时间内调用过于集中的问题。 例如 ABC 三节点有如下权重：`{A: 3, B: 2, C: 1}` 那么按照最原始的轮询算法，调用过程将变成：`A A A B B C`

对此，Dubbo 借鉴 Nginx 的平滑加权轮询算法，对此做了优化，调用过程可抽象成下表:

| 轮前加和权重              | 本轮胜者                   | 合计权重 | 轮后权重（胜者减去合计权重）       |
| ------------------- | ---------------------- | ---- | -------------------- |
| 起始轮                 | \|\|`A(0), B(0), C(0)` |      |                      |
| `A(3), B(2), C(1)`  | A                      | 6    | `A(-3), B(2), C(1)`  |
| `A(0), B(4), C(2)`  | B                      | 6    | `A(0), B(-2), C(2)`  |
| `A(3), B(0), C(3)`  | A                      | 6    | `A(-3), B(0), C(3)`  |
| `A(0), B(2), C(4)`  | C                      | 6    | `A(0), B(2), C(-2)`  |
| `A(3), B(4), C(-1)` | B                      | 6    | `A(3), B(-2), C(-1)` |
| `A(6), B(0), C(0)`  | A                      | 6    | `A(0), B(0), C(0)`   |
我们发现经过合计权重（3+2+1）轮次后，循环又回到了起点，整个过程中节点流量是平滑的，且哪怕在很短的时间周期内，概率都是按期望分布的。如果用户有加权轮询的需求，可放心使用该算法
## 3、最少活跃优先 + 加权随机
- **加权最少活跃调用优先**，活跃数越低，越优先调用，相同活跃数的进行加权随机。活跃数指调用前后计数差（针对特定提供者：请求发送数 - 响应返回数），表示特定提供者的任务堆积量，活跃数越低，代表该提供者处理能力越强。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大；相对的，处理能力越强的节点，处理更多的请求
## 4、最短响应优先 + 加权随机
- **加权最短响应优先**，在最近一个滑动窗口中，响应时间越短，越优先调用。相同响应时间的进行加权随机。
- 使得响应时间越快的提供者，处理更多的请求。
- 缺点：可能会造成流量过于集中于高性能节点的问题
## 5、一致性哈希
- **一致性 Hash**，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 算法参见：[Consistent Hashing | WIKIPEDIA](http://en.wikipedia.org/wiki/Consistent_hashing)
- 缺省只对第一个参数 Hash，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
- 缺省用 160 份虚拟节点，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`
## 6、P2C
1. 对于每次调用，从可用的provider列表中做两次随机选择，选出两个节点providerA和providerB。
2. 比较providerA和providerB两个节点，选择其“当前正在处理的连接数”较小的那个节点

## 7、自适应
> 一种能根据后端实例负载自动调整流量分布的算法实现，它总是尝试将请求转发到负载最小的节点

## 8、配置方式
持在服务提供者一侧配置默认的负载均衡策略，这样所有的消费者都将默认使用提供者指定的负载均衡策略，消费者可以自己配置要使用的负载均衡策略，如果都没有任何配置， 则默认使用随机负载均衡策略
- 全局配置
```yaml
dubbo:
  consumer:
    loadbalance: roundrobin
```
- provider端
```java
@DubboService(loadbalance = "roundrobin")
public class DemoServiceImpl implements DemoService {}
```
- consumer端
```java
@DubboReference(loadbalance = "roundrobin")
private DemoService demoService;
```
- 方法级配置1
```java
@Configuration
public class DubboConfiguration {
    @Bean
    public ServiceBean demoService() {
    	MethodConfig method = new MethodConfig();
		method.setName("sayHello");
		method.setLoadbalance("roundrobin");

        ServiceBean service = new ServiceBean();
        service.setInterface(DemoService.class);
        service.setRef(new DemoServiceImpl());
        service.addMethod(method)
        return service;
    }
}
```
- 方法级配置2
```java
@Autowire
private DemoService demoService;

@Configuration
public class DubboConfiguration {
    @Bean
    public ReferenceBean demoService() {
    	MethodConfig method = new MethodConfig();
		method.setName("sayHello");
		method.setLoadbalance("roundrobin");

        ReferenceBean<DemoService> reference = new ReferenceBean<>();
		reference.setInterface(DemoService.class);
		reference.addMethod(method);
        return reference;
    }
}
```
- 方法级配置3
```
dubbo.reference.org.apache.dubbo.samples.api.DemoService.sayHello.loadbalance=roundrobin
```
