# Zuul 整合 Sentinel 实现租户接口流控

本文旨在使用sentinel在zuul网关实现某租户对某接口粒度的流控



## 1. 启动Sentinel dashboard

1. [下载](https://github.com/alibaba/Sentinel/releases)最新的jar包

2. 使用命令启动：`java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar`

3. 默认用户名密码sentinel

## 2. zuul网关

### 2.1 引入依赖

```xml
<!--引入 Transport 模块来与 Sentinel 控制台进行通信-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>x.y.z</version>
</dependency>

<!--适配模块-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-zuul-adapter</artifactId>
    <version>x.y.z</version>
</dependency>

<!--规则配置源-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-redis</artifactId>
    <version>x.y.z</version>
</dependency>
```



### 2.2 声明bean

```java
@Configuration
public class ZuulConfig {

    @Bean
    public ZuulFilter sentinelZuulPreFilter() {
        // We can also provider the filter order in the constructor.
        return new SentinelZuulPreFilter();
    }

    @Bean
    public ZuulFilter sentinelZuulPostFilter() {
        return new SentinelZuulPostFilter();
    }

    @Bean
    public ZuulFilter sentinelZuulErrorFilter() {
        return new SentinelZuulErrorFilter();
    }
}
```



### 2.3 启动

使用以下参数启动

`-Dproject.name=zuul-gateway -Dcsp.sentinel.dashboard.server=localhost:8080 -Dcsp.sentinel.api.port=8720 -Dcsp.sentinel.app.type=1`



## 3. zuul网关配置

使用 Sentinel 来进行资源保护，主要分为几个步骤:

1. 定义资源
2. 定义规则
3. 检验规则是否生效



Sentinel 提供了 Zuul 1.x 的适配模块，可以为 Zuul Gateway 提供两种资源维度的限流：

- route 维度：即在 Spring 配置文件中配置的路由条目，资源名为对应的 route ID（对应 `RequestContext` 中的 `proxy` 字段）
- 自定义 API 维度：用户可以利用 Sentinel 提供的 API 来自定义一些 API 分组



#### 3.1 发生限流之后的处理流程

- 发生限流之后可自定义返回参数，通过实现 `SentinelFallbackProvider` 接口，默认的实现是 `DefaultBlockFallbackProvider`。
- 默认的 fallback route 的规则是 route ID 或自定义的 API 分组名称。

```java
// 自定义 FallbackProvider 
public class MyBlockFallbackProvider implements ZuulBlockFallbackProvider {

    private Logger logger = LoggerFactory.getLogger(DefaultBlockFallbackProvider.class);
    
    // you can define route as service level 
    @Override
    public String getRoute() {
        return "/book/app";
    }

    @Override
        public BlockResponse fallbackResponse(String route, Throwable cause) {
            RecordLog.info(String.format("[Sentinel DefaultBlockFallbackProvider] Run fallback route: %s", route));
            if (cause instanceof BlockException) {
                //这里429是httpcode
                return new BlockResponse(429, "Sentinel block exception", route);
            } else {
                return new BlockResponse(500, "System Error", route);
            }
        }
 }
 
 // 注册 FallbackProvider
 ZuulBlockFallbackManager.registerProvider(new MyBlockFallbackProvider());
```

限流发生之后的默认返回：

```json
{
    "code":429,
    "message":"Sentinel block exception",
    "route":"/"
}
```



##### 3.1.1 自定义response body

ZuulBlockFallbackProvider 只提供自定义response code，若需要自定义response body，需override SentinelZuulPreFilter，在catch BlockException处理逻辑中，添加`ctx.addZuulResponseHeader(XXX,XXX)`



## 4. 动态规则扩展

### 4.1 规则

Sentinel 的理念是开发者只需要关注资源的定义，当资源定义成功后可以动态增加各种流控降级规则。Sentinel 提供两种方式修改规则：

- 通过 API 直接修改 (`loadRules`)
- 通过 `DataSource` 适配不同数据源修改

手动通过 API 修改比较直观，可以通过以下几个 API 修改不同的规则：

```java
FlowRuleManager.loadRules(List<FlowRule> rules); // 修改流控规则
DegradeRuleManager.loadRules(List<DegradeRule> rules); // 修改降级规则
```

手动修改规则（硬编码方式）一般仅用于测试和演示，生产上一般通过动态规则源的方式来动态管理规则。



### 4.2 DataSource 扩展

上述 `loadRules()` 方法只接受内存态的规则对象，但更多时候规则存储在文件、数据库或者配置中心当中。`DataSource` 接口给我们提供了对接任意配置源的能力。相比直接通过 API 修改规则，实现 `DataSource` 接口是更加可靠的做法。

我们推荐**通过控制台设置规则后将规则推送到统一的规则中心，客户端实现** `ReadableDataSource` **接口端监听规则中心实时获取变更**，流程如下：

![push-rules-from-dashboard-to-config-center](https://user-images.githubusercontent.com/9434884/45406233-645e8380-b698-11e8-8199-0c917403238f.png)

`DataSource` 扩展常见的实现方式有:

- **拉模式**：客户端主动向某个规则管理中心定期轮询拉取规则，这个规则中心可以是 RDBMS、文件，甚至是 VCS 等。这样做的方式是简单，缺点是无法及时获取变更；
- **推模式**：规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用 [Nacos](https://github.com/alibaba/nacos)、Zookeeper 等配置中心。这种方式有更好的实时性和一致性保证。

Sentinel 目前支持以下数据源扩展：

- Pull-based: 文件、Consul (since 1.7.0)
- Push-based: ZooKeeper, Redis, Nacos, Apollo



#### 4.2.1 拉模式拓展

实现拉模式的数据源最简单的方式是继承 [`AutoRefreshDataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/AutoRefreshDataSource.java) 抽象类，然后实现 `readSource()`方法，在该方法里从指定数据源读取字符串格式的配置数据。比如 [基于文件的数据源](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-dynamic-file-rule/src/main/java/com/alibaba/csp/sentinel/demo/file/rule/FileDataSourceDemo.java)。



#### 4.2.2 推模式拓展

实现推模式的数据源最简单的方式是继承 [`AbstractDataSource`](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-extension/src/main/java/com/alibaba/csp/sentinel/datasource/AbstractDataSource.java) 抽象类，在其构造方法中添加监听器，并实现 `readSource()` 从指定数据源读取字符串格式的配置数据。比如 [基于 Nacos 的数据源](https://github.com/alibaba/Sentinel/blob/master/sentinel-extension/sentinel-datasource-nacos/src/main/java/com/alibaba/csp/sentinel/datasource/nacos/NacosDataSource.java)。

控制台通常需要做一些改造来直接推送应用维度的规则到配置中心。功能示例可以参考 [AHAS Sentinel 控制台的规则推送功能](https://github.com/alibaba/Sentinel/wiki/AHAS-Sentinel-%E6%8E%A7%E5%88%B6%E5%8F%B0#44-%E8%A7%84%E5%88%99%E7%AE%A1%E7%90%86)。改造指南可以参考 [在生产环境中使用 Sentinel 控制台](https://github.com/alibaba/Sentinel/wiki/%E5%9C%A8%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%B8%AD%E4%BD%BF%E7%94%A8-Sentinel)。



### 4.3 示例

#### 4.3.1 推模式：使用 ZooKeeper 配置规则

Sentinel 针对 ZooKeeper 作了相应适配，底层可以采用 ZooKeeper 作为规则配置数据源。使用时只需添加以下依赖：

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-zookeeper</artifactId>
    <version>x.y.z</version>
</dependency>
```

然后创建 `ZookeeperDataSource` 并将其注册至对应的 RuleManager 上即可。比如：

```java
// remoteAddress 代表 ZooKeeper 服务端的地址
// path 对应 ZK 中的数据路径
ReadableDataSource<String, List<FlowRule>> flowRuleDataSource = new ZookeeperDataSource<>(remoteAddress, path, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
```

详细示例可以参见 [sentinel-demo-zookeeper-datasource](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo/sentinel-demo-zookeeper-datasource)。



####4.3.2 推模式：使用 Redis 配置规则(截止1.6.3版本 不支持redis cluster)

Sentinel 针对 Redis 作了相应适配，底层可以采用 Redis 作为规则配置数据源。使用时只需添加以下依赖：

```xml
<!-- 仅支持 JDK 1.8+ -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-redis</artifactId>
    <version>x.y.z</version>
</dependency>
```

Redis 动态配置源采用 Redis PUB-SUB 机制实现，详细文档参考：<https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-datasource-redis>



## 5. 参考资料

alibaba sentinel wiki: https://github.com/alibaba/Sentinel/wiki/