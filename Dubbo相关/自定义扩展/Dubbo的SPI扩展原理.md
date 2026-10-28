Dubbo 中的扩展机制与 [JDK 标准的 SPI 扩展点](https://www.baeldung.com/java-spi) 原理类似。Dubbo 对其做了一定的改造与加强：
- JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
- 如果扩展点加载失败，JDK SPI 没给出详细信息，不方便定位问题，Dubbo SPI 在失败时记录真正的失败原因，并打印出来
- 增加 [IOC](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/architecture/dubbo-spi/#23-ioc-%e6%9c%ba%e5%88%b6)、[AOP](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/architecture/dubbo-spi/#24-aop-%e6%9c%ba%e5%88%b6) 能力
- 增加排序能力
- 增加条件激活能力
- 提供了一系列更灵活的 API，如`获取所有 SPI 扩展实现`、`根据名称查询某个扩展实现`、根据类型查询扩展实现、查询匹配条件的扩展实现等
![](https://cn.dubbo.apache.org/imgs/v3/concepts/extension-load.png)
## 1、定义SPI接口
Dubbo 中的 SPI 插件是标准的 Java Interface 定义，并且必须包含 `@org.apache.dubbo.common.extension.SPI` 注解：

```java
@SPI(value = "dubbo", scope = ExtensionScope.FRAMEWORK)
public interface Protocol {
	// ...
}
```
## 2、实现接口
```java
package com.spi.demo;
import org.apache.dubbo.rpc.Protocol;

@Activate
public class CustomizedProtocol implements Protocol {
	// ...
}
```
## 3、注册配置
在应用 `resources/META-INF/dubbo/` 目录下添加 `org.apache.dubbo.rpc.Protocol` 文件，文件中增加如下配置：

```properties
customized=com.spi.demo.CustomizedProtocol
```
- 文件名必须为 SPI 插件定义的 package 全路径名
- 文件中的内容必须是 `key=value` 形式，其中 `key` 可随便定义，不重复即可
## 4、激活
- 方式1
在`resources/application.properties`文件中添加如下配置，激活刚才的自定义 Filter 实现：

```properties
# Apply AppendedFilter
dubbo.protocol.name=customized
dubbo.provider.filter=myFilter
```
- 方式2
可以使用 `@org.apache.dubbo.common.extension.Activate` 来控制 SPI 扩展实现在什么场景下加载生效。相比于任何场景下都生效，能精确的控制扩展实现的生效条件会让实现变得更灵活。
以下是一些使用场景示例：

```java
// 不加 @Activate 注解，getActivateExtension() 时不会加载，其他 getExtension() 方法仍可正常加载
public class MetricsProviderFilter implements Filter{}
```

```java
// 不加任何参数，表示在 getActivateExtension() 时无条件自动返回
@Activate
public class MetricsProviderFilter implements Filter{}
```

```java
// group 支持 consumer、provider 两个固定值，getActivateExtension() 调用加载扩展点时自动过滤
// provider 表示在提供者端会被加载；consumer 表示在消费者端会被加载
@Activate(group="provider")
public class MetricsProviderFilter implements Filter{}
```

```java
// URL 参数中有 cache 这个 key 时，调用 getActivateExtension() 才会加载
@Activate(value="cache")
public class MetricsProviderFilter implements Filter{}
```

```java
// URL 参数中有 cache 这个 key 并且值为 test 时，调用 getActivateExtension() 才会加载
@Activate(value="cache:test")
public class MetricsProviderFilter implements Filter{}
```

## 5、优先级
> internal > dubbo > services
> 官方推荐自定义存放：META-INF/dubbo/
> META-INF/services 是Java SPI目录，例如：META-INF/services/java.sql.Driver

``` java
META-INF/dubbo/internal/  官方扩展
META-INF/dubbo/           自定义扩展
META-INF/services/        Java SPI扩展
```
Dubbo3 为了兼容 Java SPI，也会扫描META-INF/services/ 目录，但是配置内容没有key