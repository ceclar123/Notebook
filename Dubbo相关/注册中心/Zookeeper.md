## 增加 Maven 依赖
```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
<!-- 推荐 Zookeeper Server 3.8.0+ 版本 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-zookeeper-curator5-spring-boot-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
<!-- 仅当 Zookeeper Server 版本是 3.4.x 及以下时，使用此依赖
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-zookeeper-spring-boot-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
-->
```

## Zookeeper版本选择
由于 Dubbo 使用 Curator 作为与 Zookeeper Server 交互的编程客户端，因此，要特别注意 Zookeeper Server 与 Dubbo 版本依赖的兼容性。

Dubbo 提供了 Zookeeper 依赖的辅助管理组件，开发者可根据当前使用的 Zookeeper Server 版本选择依赖版本：

**1. 如果您是 Dubbo3 3.3 版本及以上用户，请根据如下表格选择组件：**

|**Zookeeper Server 版本**|**Dubbo 依赖**|**Dubbo Starter 依赖（SpringBoot用户）**|
|---|---|---|
|3.4.x 及以下|dubbo-dependencies-zookeeper|dubbo-zookeeper-spring-boot-starter|
|3.5.x 及以上|dubbo-dependencies-zookeeper-curator5|dubbo-zookeeper-curator5-spring-boot-starter|

**2. 如果您是 Dubbo3 3.2 及以下、Dubbo2 2.7.x 用户：**

|**Zookeeper Server 版本**|**Dubbo 依赖**|**Dubbo Starter 依赖（SpringBoot用户）**|
|---|---|---|
|3.4.x 及以下|dubbo-dependencies-zookeeper|不支持（自行管理）|
|3.5.x 及以上|不支持（自行管理）|不支持（自行管理）|

**注意**
- Dubbo 3.3.0 版本开始正式支持 JDK 17，如果您使用 JDK 17，则必须选用 dubbo-dependencies-zookeeper-curator5 或 dubbo-zookeeper-curator5-spring-boot-starter 依赖，对应的 Zookeeper Server 推荐是 3.8.0 版本及以上。
- 如果是自行管理 zookeeper 依赖，则须确保在项目中引入正确的 zookeeper、curator 版本依赖，可参考 Dubbo 3.3.0 版本中的 `dubbo-dependencies-zookeeper` 或 `dubbo-dependencies-zookeeper-curator5` 组件是如何实现的

## 配置Zookeeper
```yaml
# application.yml
dubbo
 registry
   address: zookeeper://localhost:2181
   # zookeeper://10.20.153.10:2181?backup=10.20.153.11:2181,10.20.153.12:2181
```

或

```properties
# dubbo.properties
dubbo.registry.address=zookeeper://localhost:2181
# dubbo.registry.address=zookeeper://10.20.153.10:2181backup=10.20.153.11:2181,10.20.153.12:2181
```

或

```xml
<dubbo:registry address="zookeeper://localhost:2181" />
<dubbo:registry address="zookeeper://localhost:2181" />
<dubbo:registry protocol="zookeeper" address="10.20.153.10:2181,10.20.153.11:2181,10.20.153.12:2181" />
```

## 高级配置
### 认证与鉴权
如果 Zookeeper 开启认证，Dubbo 支持指定 username、password 的方式传入身份标识。

```yaml
# application.yml
dubbo
 registry
   address: zookeeper://localhost:2181
   username: hello
   password: 1234
```
也可以直接将参数扩展在 address 上 `address=zookeeper://hello:1234@localhost:2181`
### 分组隔离
通过指定 `group` 属性，可以在同一个 Zookeeper 集群内实现微服务地址的逻辑隔离。比如可以在一套集群内隔离出多套开发环境，在地址发现层面实现隔离。

```yaml
# application.yml
dubbo
 registry
   address: zookeeper://localhost:2181
   group: prod
```
### 其他配置
配置连接、会话过期时间，[详细参数](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/config/properties/#dubbo.registry)

```yaml
# application.yml
dubbo
 registry
   address: zookeeper://localhost:2181
   timeout: 30 * 1000* # 连接超时时间，默认 30s
   session: 60 * 1000* # 会话超时时间，默认 60s
```

## 工作原理
Zookeeper 注册中心实现支持以下高可用能力：
- 当提供者出现断电等异常停机时，注册中心能自动删除提供者信息
- 当注册中心重启时，能自动恢复注册数据，以及订阅请求
- 当会话过期时，能自动恢复注册数据，以及订阅请求
- 当设置 `registry.check=false` 时，记录失败注册和订阅请求，后台定时重试
## 接口级节点结构
![](https://cn.dubbo.apache.org/imgs/user/zookeeper.jpg)
流程：
- 服务提供者启动时: 向 `/dubbo/com.foo.BarService/providers` 目录下写入自己的 URL 地址
- 服务消费者启动时: 订阅 `/dubbo/com.foo.BarService/providers` 目录下的提供者 URL 地址,并向 `/dubbo/com.foo.BarService/consumers` 目录下写入自己的 URL 地址
- 监控中心启动时: 订阅 `/dubbo/com.foo.BarService` 目录下的所有提供者和消费者 URL 地址
可通过 `registry.group` 设置 zookeeper 的根节点，不配置将使用默认的 `/dubbo` 根节点

### 应用级节点结构
#### 地址列表
![](https://cn.dubbo.apache.org/imgs/v3/tasks/registry/zookeeper-hierarchy.png)
应用级服务发现的地址结构比接口级更精简，它以应用名为粒度分发地址列表。服务提供者启动时，向 `/services/app` 目录下写入自己的 URL 地址，相比于接口级别的 URL，应用级别的 URL 更简单，只包含一些实例级别的参数，如 `tri://ip:port?region=hangzhou`。

可通过 `registry.group` 设置 zookeeper 的根节点，如设置 `registry.group=dubbo` 后，地址根节点变为 `/dubbo`。不配置将使用默认的 `/services` 根节点。在与 Spring Cloud Gateway 共用情况下，使用 `/services` 根节点会导致 dubbo 地址被 gateway 消费，此时可考虑设置独立 group

**注意**
在应用级服务发现模型中，接口级别的配置信息由消费者与提供者之间自行协商同步，不再由注册中心负责同步，从而大大减少了注册中心地址同步压力
#### 接口应用映射
在应用级服务发现中，zookeeper 注册中心还会存储一份额外的元数据，用于解决 `接口名到应用名` 之间的映射关系，其存储结构如下：
![](https://cn.dubbo.apache.org/imgs/v3/tasks/registry/zookeeper-mapping.png)
service1 节点的 value 值是应用列表，可通过 `get /dubbo/mapping/service1` 查看：app1,app2

#### 元数据
如果您用的是应用级服务发现的集中式元数据模式（默认是点对点的元数据模式，可通过 `dubbo.registry.metadata-type=remote` 开启）。在开启集中式元数据模式后，zookeeper 中还会发现以下节点内容：
![](https://cn.dubbo.apache.org/imgs/v3/tasks/registry/zookeeper-metadata.png)
每个 revision 下是该应用的部署元数据信息，包含完整的接口服务列表及其配置信息