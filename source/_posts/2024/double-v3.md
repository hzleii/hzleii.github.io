---
title: 'dubbo3.0 服务导入导出原理'
date: 2024-03-25
description: 'dubbo3.0 服务导入导出原理。'
# topic: leetcode
author: hzlei
banner: /assets/post/2024/double-v3/index.webp
cover: /assets/post/2024/double-v3/index.webp
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: dubbo3.0 服务导入导出原理。 # 必选
  # caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  color: hsl(184deg 67% 53%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---


作者：京东物流 张士欣

不管是服务导出还是服务引入，都发生在应用启动过程中。
比如：在启动类上加上`@EnableDubbo`时，该注解上有一个`@DubboComponentScan`注解，`@DubboComponentScan` 注解Import了一个DubboComponentScanRegistrar，DubboComponentScanRegistrar中会调用`DubboSpringInitializer.initialize()`，该方法中会注册一个`DubboDeployApplicationListener`，而`DubboDeployApplicationListener`会监听Spring容器启动完成事件ContextRefreshedEvent，一旦接收到这个事件后，就会开始Dubbo的启动流程，就会执行`DefaultModuleDeployer`的`start()`进行服务导出与服务引入。

**在启动过程中，在做完服务导出与服务引入后，还会做几件非常重要的事情：**

1. 导出一个应用元数据服务（就是一个 MetadataService 服务，这个服务也会注册到注册中心），或者将应用元数据注册到元数据中心；

2. 生成当前应用的实例信息对象 ServiceInstance，比如：应用名、实例 ip、实例 port，并将实例信息注册到注册中心，也就是应用级注册；

## 服务导出

当在某个接口的实现类上加上 `@DubboService` 后，就表示定义了一个 Dubbo 服务，应用启动时 Dubbo 只要扫描到了 `@DubboService`，就会解析对应的类，得到服务相关的配置信息，比如：

1. 服务的类型，也就是接口，接口名就是服务名；

2. 服务的具体实现类，也就是当前类；

3. 服务的 version、timeout 等信息，就是 @DubboService 中所定义的各种配置；

解析完服务的配置信息后，就会把这些配置信息封装成为一个 ServiceConfig 对象，并调用其 export() 进行服务导出，此时一个 ServiceConfig 对象就表示一个 Dubbo 服务。

而所谓的服务导出，主要就是完成三件事情：

1. 确定服务的最终参数配置；

2. 按不同协议启动对应的 Server（服务暴露）；

3. 将服务注册到注册中心（服务注册）；

### 确定服务参数

一个 Dubbo 服务，除开服务的名字，也就是接口名，还会有很多其他的属性，比如：超时时间、版本号、服务所属应用名、所支持的协议及绑定的端口等众多信息。

但是，通常这些信息并不会全部在 `@DubboService` 中进行定义，比如：一个 Dubbo 服务肯定是属于某个应用的，而一个应用下可以有多个 Dubbo 服务，所以可以在应用级别定义一些通用的配置，比如协议。

在 application.yml 中定义：

```yaml
dubbo:
 application:
  name: dubbo-springboot-demo-provider
 protocol:
  name: tri
  port: 20880
```

表示当前应用下所有的 Dubbo 服务都支持通过 tri 协议进行访问，并且访问端口为 20880，所以在进行某个服务的服务导出时，就需要将应用中的这些配置信息合并到当前服务的配置信息中。

另外，除开可以通过 `@DubboService` 来配置服务，也可以在配置中心对服务进行配置，比如：在配置中心中配置：

`dubbo.service.org.apache.dubbo.samples.api.DemoService.timeout=5000`

表示当前服务的超时时间为 5s。

所以，在服务导出时，也需要从配置中心获取当前服务的配置，如果在 `@DubboService` 中也定义了 timeout，那么就用配置中心的覆盖掉，配置中心的配置优先级更高。

最终确定出服务的各种参数，这块内容和 Dubbo2.7 一致。

## 服务注册

当确定好了最终的服务配置后，Dubbo 就会根据这些配置信息生成对应的服务 URL，比如：

`tri://192.168.65.221:20880/org.apache.dubbo.springboot.demo.DemoService?application=dubbo-springboot-demo-provider&timeout=3000`

这个 URL 就表示了一个 Dubbo 服务，服务消费者只要能获得到这个服务 URL，就知道了关于这个 Dubbo 服务的全部信息，包括服务名、支持的协议、ip、port、各种配置。

确定了服务 URL 之后，服务注册要做的事情就是把这个服务 URL 存到注册中心（比如：Zookeeper）中去，说的再简单一点，就是把这个字符串存到 Zookeeper中去，这个步骤其实是非常简单的，实现这个功能的源码在 RegistryProtocol 中的 export() 方法中，最终服务 URL 存在了 Zookeeper 的 `/dubbo/接口名/providers` 目录下。

但是服务注册并不仅仅就这么简单，既然上面的这个 URL 表示一个服务，并且还包括了服务的一些配置信息，那这些配置信息如果改变了呢？比如：利用 Dubbo管理台中的动态配置功能（注意，并不是配置中心）来修改服务配置，动态配置可以应用运行过程中动态的修改服务的配置，并实时生效。

如果利用动态配置功能修改了服务的参数，那此时就要重新生成服务 URL 并重新注册到注册中心，这样服务消费者就能及时的获取到服务配置信息。

而对于服务提供者而言，在服务注册过程中，还需要能监听到动态配置的变化，一旦发生了变化，就根据最新的配置重新生成服务 URL，并重新注册到中心。

## 应用级注册

在 Dubbo3.0 之前，Dubbo 是接口级注册，服务注册就是把接口名以及服务配置信息注册到注册中心中，注册中心存储的数据格式大概为：

```ini
接口名1：tri://192.168.1.221:20880/接口名1?application=应用名
接口名2：tri://192.168.1.221:20880/接口名2?application=应用名
接口名3：tri://192.168.1.221:20880/接口名3?application=应用名
```

key 是接口名，value 就是服务 URL，上面的内容就表示现在有一个应用，该应用下有 3 个接口，应用实例部署在 192.168.1.221，此时，如果给该应用增加一个实例，实例 ip 为192.168.1.222，那么新的实例也需要进行服务注册，会向注册中心新增 3 条数据：

```ini
接口名1：tri://192.168.1.221:20880/接口名1?application=应用名
接口名2：tri://192.168.1.221:20880/接口名2?application=应用名
接口名3：tri://192.168.1.221:20880/接口名3?application=应用名

接口名1：tri://192.168.1.222:20880/接口名1?application=应用名
接口名2：tri://192.168.1.222:20880/接口名2?application=应用名
接口名3：tri://192.168.1.222:20880/接口名3?application=应用名
```

可以发现，如果一个应用中有 3 个 Dubbo 服务，那么每增加一个实例，就会向注册中心增加 3 条记录，那如果一个应用中有 10 个 Dubbo 服务，那么每增加一个实例，就会向注册中心增加 10 条记录，注册中心的压力会随着应用实例的增加而剧烈增加。

反过来，如果一个应用有 3 个 Dubbo 服务，5 个实例，那么注册中心就有 15 条记录，此时增加一个 Dubbo 服务，那么注册中心就会新增 5 条记录，注册中心的压力也会剧烈增加。

注册中心的数据越多，数据就变化的越频繁，比如：修改服务的 timeout，那么对于注册中心和应用都需要消耗资源用来处理数据变化。

所以为了降低注册中心的压力，Dubbo3.0 支持了应用级注册，同时也兼容接口级注册，用户可以逐步迁移成应用级注册，而一旦采用应用级注册，最终注册中心的数据存储就变成为：

```makefile
应用名：192.168.1.221:20880
应用名：192.168.1.222:20880
```

表示在注册中心中，只记录应用所对应的实例信息（IP + 绑定的端口），这样只有一个应用的实例增加了，那么注册中心的数据才会增加，而不关心一个应用中到底有多少个 Dubbo 服务。

这样带来的好处就是，注册中心存储的数据变少了，注册中心中数据的变化频率变小了，并且使用应用级注册，使得 Dubbo3 能实现与异构微服务体系如：Spring Cloud、Kubernetes Service 等在地址发现层面更容易互通， 为连通 Dubbo 与其他微服务体系提供可行方案。

应用级注册带来了好处，但是对于 Dubbo 来说又出现了一些新的问题，比如：原本，服务消费者可以直接从注册中心就知道某个 Dubbo 服务的所有服务提供者以及相关的协议、ip、port、配置等信息，那现在注册中心上只有 ip、port，那对于服务消费者而言：服务消费者怎么知道现在它要用的某个 Dubbo 服务，也就是某个接口对应的应用是哪个呢？

对于这个问题，在进行服务导出的过程中，会在 Zookeeper 中存一个映射关系，在服务导出的最后一步，在 ServiceConfig 的 exported() 方法中，会保存这个映射关系：`接口名：应用名`。这个映射关系存在 Zookeeper 的 /dubbo/mapping 目录下，存了这个信息后，消费者就能根据接口名找到所对应的应用名了。

消费者知道了要使用的 Dubbo 服务在哪个应用，那也就能从注册中心中根据应用名查到应用的所有实例信息（ ip + port ），也就是可以发送方法调用请求了，但是在真正发送请求之前，还得知道服务的配置信息，对于消费者而言，它得知道当前要调用的这个 Dubbo 服务支持什么协议、timeout 是多少，那服务的配置信息从哪里获取呢？

之前的服务配置信息是直接从注册中心就可以获取到的，就是服务 URL 后面，但是现在不行了，现在需要从服务提供者的元数据服务获取，前面提到过，在应用启动过程中会进行服务导出和服务引入，然后就会暴露一个应用元数据服务，其实这个应用元数据服务就是一个 Dubbo 服务（Dubbo 框架内置的，自己实现的 ），消费者可以调用这个服务来获取某个应用中所提供的所有 Dubbo 服务以及服务配置信息，这样也就能知道服务的配置信息了。

知道了应用注册的好处，以及相关问题的解决方式，那么来看它到底是如何实现的。

首先，我们可以通过配置 dubbo.application.register-mode 来控制：

1. instance：表示只进行应用级注册；

2. interface：表示只进行接口级注册；

3. all：表示应用级注册和接口级注册都进行，默认；

不管是什么注册，都需要存数据到注册中心，而 Dubbo3 的源码实现中会根据所配置的注册中心生成两个 URL（不是服务 URL，可以理解为注册中心 URL，用来访问注册中心的）:

1. service-discovery-registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbospringboot-demoprovider& dubbo=2.0.2&pid=13072&qos.enable=false®istry=zookeeper×tamp=1651755501660

2. registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-springboot-demoprovider& dubbo=2.0.2&pid=13072&qos.enable=false®istry=zookeeper×tamp=1651755501660

这两个 URL 只有 schema 不一样，一个是 service-discovery-registry，一个是 registry，而 registry 是 Dubbo3 之前就存在的，也就代表接口级服务注册，而service-discovery-registry 就表示应用级服务注册。

在服务注册相关的源码中，当调用 RegistryProtocol 的 export() 方法处理 registry:// 时，会利用 ZookeeperRegistry 把服务 URL 注册到 Zookeeper 中去，这就是接口级注册。

而类似，当调用 RegistryProtocol 的 export() 方法处理 service-discovery-registry:// 时，会利用 ServiceDiscoveryRegistry 来进行相关逻辑的处理，那是不是就是在这里把应用信息注册到注册中心去呢？并没有这么简单。

1. 首先，不可能每导出一个服务就进行一次应用注册，太浪费了，应用注册只要做一次就行了

2. 其次，如果一个应用支持了多个端口，那么应用注册时只要挑选其中一个端口作为实例端口就可以了（该端口只要能接收到数据就行）

3. 前面提到，应用启动过程中要暴露应用元数据服务，所以在此处也还是要收集当前所暴露的服务配置信息，以提供给应用元数据服务

所以 ServiceDiscoveryRegistry 在注册一个服务 URL 时，并不会往注册中心存数据，而只是把服务 URL 存到到一个 MetadataInfo 对象中，MetadataInfo 对象中就保存了当前应用中所有的 Dubbo 服务信息：服务名、支持的协议、绑定的端口、timeout 等。

前面提到过，在应用启动的最后，才会进行应用级注册，而应用级注册就是当前的应用实例上相关的信息存入注册中心，包括：

1. 应用的名字；

2. 获取应用元数据的方式；

3. 当前实例的 ip 和 port；

4. 当前实例支持哪些协议以及对应的 port；

比如：

```python
{
  "name":"dubbo-springboot-demo-provider",
  "id":"192.168.65.221:20882",
  "address":"192.168.65.221",
  "port":20882,
  "sslPort":null,
  "payload":{
    "@class":"org.apache.dubbo.registry.zookeeper.ZookeeperInstance",
    "id":"192.168.65.221:20882",
    "name":"dubbo-springboot-demo-provider",
    "metadata":{
      "dubbo.endpoints":"[{"port":20882,"protocol":"dubbo"},{"port":50051,"protocol":"tri"}]",
      "dubbo.metadata-service.url-params":"{"connections":"1","version":"1.0.0","dubbo":"2.0.2","side":"provider","port":"20882","protocol":"dubbo"}",
      "dubbo.metadata.revision":"65d5c7b814616ab10d32860b54781686",
      "dubbo.metadata.storage-type":"local"
    } 
  },
  "registrationTimeUTC":1654585977352,
  "serviceType":"DYNAMIC",
  "uriSpec":null
}
```

一个实例上可能支持多个协议以及多个端口，那如何确定实例的 ip 和端口呢？

答案是：获取 MetadataInfo 对象中保存的所有服务 URL，优先取 dubbo 协议对应 ip 和 port，没有 dubbo 协议则所有服务 URL 中的第一个 URL 的 ip 和 port。

另外一个协议一般只会对应一个端口，但是如何就是对应了多个，比如：

```yaml
dubbo:
  application:
    name: dubbo-springboot-demo-provider
  protocols:
    p1:
      name: dubbo
      port: 20881
    p2:
      name: dubbo
      port: 20882
    p3:
      name: tri
      port: 50051
```

如果是这样，最终存入 endpoint 中的会保证一个协议只对应一个端口，另外那个将被忽略，最终服务消费者在进行服务引入时将会用到这个 endpoint 信息。

确定好实例信息后之后，就进行最终的应用注册了，就把实例信息存入注册中心的 `/services/应用名`，目录下：

![目录](/assets/post/2024/ruoyi-rce//assets/post/2024/double-v3/1.webp)

可以看出 services 节点下存的是应用名，应用名的节点下存的是实例 ip 和实例 port，而 ip 和 port 这个节点中的内容就是实例的一些基本信息。

额外，我们可以配置 dubbo.metadata.storage-type，默认时 local，可以通过配置改为 remote：

```yaml
dubbo:
	application:
		name: dubbo-springboot-demo-provider
		metadata-type: remote
```

这个配置其实跟应用元数据服务有关系：

1. 如果为 local，那就会启用应用元数据服务，最终服务消费者就会调用元数据服务获取到应用元数据信息；

2. 如果为 remote，那就不会暴露应用元数据服务，那么服务消费者从元数据中心获取应用元数据；

在 Dubbo2.7 中就有了元数据中心，它其实就是用来减轻注册中心的压力的，Dubbo 会把服务信息完整的存一份到元数据中心，元数据中心也可以用 Zookeeper来实现，在暴露完元数据服务之后，在注册实例信息到注册中心之前，就会把 MetadataInfo 存入元数据中心，比如：

![示例](/assets/post/2024/ruoyi-rce//assets/post/2024/double-v3/2.webp)

节点内容为：

```json
{
    "app":"dubbo-springboot-demo-provider",
    "revision":"64e68950e300068e6b5f8632d9fd141d",
    "services":{
        "org.apache.dubbo.springboot.demo.HelloService:tri":{
            "name":"org.apache.dubbo.springboot.demo.HelloService",
            "protocol":"tri",
            "path":"org.apache.dubbo.springboot.demo.HelloService",
            "params":{
                "side":"provider",
                "release":"",
                "methods":"sayHello",
                "deprecated":"false",
                "dubbo":"2.0.2",
                "interface":"org.apache.dubbo.springboot.demo.HelloService",
                "service-name-mapping":"true",
                "generic":"false",
                "metadata-type":"remote",
                "application":"dubbo-springboot-demo-provider",
                "background":"false",
                "dynamic":"true",
                "anyhost":"true"
            }
        },
        "org.apache.dubbo.springboot.demo.DemoService:tri":{
            "name":"org.apache.dubbo.springboot.demo.DemoService",
            "protocol":"tri",
            "path":"org.apache.dubbo.springboot.demo.DemoService",
            "params":{
                "side":"provider",
                "release":"",
                "methods":"sayHelloStream,sayHello,sayHelloServerStream",
                "deprecated":"false",
                "dubbo":"2.0.2",
                "interface":"org.apache.dubbo.springboot.demo.DemoService",
                "service-name-mapping":"true",
                "generic":"false",
                "metadata-type":"remote",
                "application":"dubbo-springboot-demo-provider",
                "background":"false",
                "dynamic":"true",
                "anyhost":"true"
            }
        }
    }
}
```

这里面就记录了当前实例上提供了哪些服务以及对应的协议，注意并没有保存对应的端口......，所以后面服务消费者得利用实例信息中的 endpoint，因为endpoint 中记录了协议对应的端口....

其实元数据中心和元数据服务提供的功能是一样的，都可以用来获取某个实例的 MetadataInfo，上面中的 UUID 表示实例编号，只不过元数据中心是集中式的，元数据服务式分散在各个提供者实例中的，如果整个微服务集群压力不大，那么效果差不多，如果微服务集群压力大，那么元数据中心的压力就大，此时单个元数据服务就更适合，所以默认也是采用的元数据服务。

至此，应用级服务注册的原理就分析完了，总结一下：

1. 在导出某个 Dubbo 服务 URL 时，会把服务 URL 存入 MetadataInfo 中；

2. 导出完某个 Dubbo 服务后，就会把服务接口名:应用名存入元数据中心（可以用 Zookeeper 实现）；

3. 导出所有服务后，完成服务引入后；

4. 判断要不要启动元数据服务，如果要就进行导出，固定使用 Dubbo 协议；

5. 将 MetadataInfo 存入元数据中心；

6. 确定当前实例信息（应用名、ip、port、endpoint）；

7. 将实例信息存入注册中心，完成应用注册；

## 服务暴露

服务暴露就是根据不同的协议启动不同的 Server，比如：dubbo 和 tri 协议启动的都是 Netty，像 Dubbo2.7 中的 http 协议启动的就是 Tomcat，这块在服务调用的时候再来分析。

## 服务引入

```java
@DubboReference
private DemoService demoService;
```

需要利用 `@DubboReference` 注解来引入某一个 Dubbo 服务，应用在启动过程中，进行完服务导出之后，就会进行服务引入，属性的类型就是一个 Dubbo 服务接口，而服务引入最终要做到的就是给这个属性赋值一个接口代理对象。

在 Dubbo2.7 中，只有接口级服务注册，服务消费者会利用接口名从注册中心找到该服务接口所有的服务 URL，服务消费者会根据每个服务 URL 的 protocol、ip、port 生成对应的 Invoker 对象，比如生成 TripleInvoker、DubboInvoker 等，调用这些 Invoker 的 invoke() 方法就会发送数据到对应的 ip、port，生成好所有的 Invoker 对象之后，就会把这些 Invoker 对象进行封装并生成一个服务接口的代理对象，代理对象调用某个方法时，会把所调用的方法信息生成一个 Invocation 对象，并最终通过某一个 Invoker 的 invoke() 方法把 Invocation 对象发送出去，所以代理对象中的 Invoker 对象是关键，服务引入最核心的就是要生成这些 Invoker 对象。

Invoker 是非常核心的一个概念，也有非常多种类，比如：

1. TripleInvoker：表示利用 tri 协议把 Invocation 对象发送出去；

2. DubboInvoker：表示利用 dubbo 协议把 Invocation 对象发送出去；

3. ClusterInvoker：有负载均衡功能；

4. MigrationInvoker：迁移功能，Dubbo3.0 新增的

像 TripleInvoker 和 DubboInvoker 对应的就是具体服务提供者，包含了服务提供者的 ip 地址和端口，并且会负责跟对应的 ip 和 port 建立 Socket 连接，后续就可以基于这个 Socket 连接并按协议格式发送 Invocation 对象。

比如现在引入了 DemoService 这个服务，那如果该服务支持：

1. 一个 tri 协议，绑定的端口为 20881；

2. 一个 tri 协议，绑定的端口为 20882；

3. 一个 dubbo 协议，绑定的端口为 20883；

那么在服务消费端这边，就会生成两个 TripleInvoker 和一个 DubboInvoker，代理对象执行方法时就会进行负载均衡选择其中一个 Invoker 进行调用。

## 接口级服务引入

在服务导出时，Dubbo3.0 默认情况下即会进行接口级注册，也会进行应用级注册，目的就是为了兼容服务消费者应用，用的还是 Dubbo2.7，用 Dubbo2.7 就只能老老实实的进行接口级服务引入。

接口级服务引入核心就是要找到当前所引入的服务有哪些服务 URL，然后根据每个服务 URL 生成对应的 Invoker，流程为：

1. 首先，根据当前引入的服务接口生成一个 RegistryDirectory 对象，表示动态服务目录，用来查询并缓存服务提供者信息；

2. RegistryDirectory 对象会根据服务接口名去注册中心，比如：Zookeeper 中的 `/dubbo/服务接口名/providers/` 节点下查找所有的服务 URL；

3. 根据每个服务 URL 生成对应的 Invoker 对象，并把 Invoker 对象存在 RegistryDirectory 对象的 invokers 属性中；

4. RegistryDirectory 对象也会监听 `/dubbo/服务接口名/providers/` 节点的数据变化，一旦发生了变化就要进行相应的改变；

5. 最后将 RegistryDirectory 对象生成一个 ClusterInvoker 对象，到时候调用 ClusterInvoker 对象的 invoke() 方法就会进行负载均衡选出某一个 Invoker 进行调用；

![](/assets/post/2024/ruoyi-rce/https://p6-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a19dcbbc57c84ddcb8f0afe04be898a8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5Lqs5Lic5LqR5byA5Y-R6ICF:q75.awebp?rk3s=f64ab15b&x-expires=1731302660&x-signature=XRdz0wTjPcMe8mnWxfrK5uClToU%3D)

## 应用级服务引入

在 Dubbo 中，应用级服务引入，并不是指引入某个应用，这里和 SpringCloud 是有区别的，在 SpringCloud 中，服务消费者只要从注册中心找到要调用的应用的所有实例地址就可以了，但是在 Dubbo 中找到应用的实例地址还远远不够，因为在 Dubbo 中是直接使用的接口，所以在 Dubbo 中就算是应用级服务引入，最终还是得找到服务接口有哪些服务提供者。

所以，对于服务消费者而言，不管是使用接口级服务引入，还是应用级服务引入，最终的结果应该得是一样的，也就是某个服务接口的提供者 Invoker 是一样的，不可能使用应用级服务引入得到的 Invoker 多一个或少一个，但是！！！，目前会有情况不一致，就是一个协议有多个端口时，比如在服务提供者应用这边支持：

```yaml
dubbo:
  application:
    name: dubbo-springboot-demo-provider
  protocols:
    p1:
      name: dubbo
      port: 20881
    p2:
      name: tri
      port: 20882
    p3:
      name: tri
      port: 50051
```

那么在消费端进行服务引入时，比如：引入 DemoService 时，接口级服务引入会生成 3 个 Invoker（2个 TripleInvoker，1个DubboInvoker），而应用级服务引入只会生成 2 个 Invoker（1个TripleInvoker，1个DubboInvoker），原因就是在进行应用级注册时是按照一个协议对应一个port存的。

那既然接口级服务引入和应用级服务引入最终的结果差不多，可能就不理解了，那应用级服务引入有什么好处呢？要知道应用级服务引入和应用级服务注册是对应，服务提供者应用如果只做应用级注册，那么对应的服务消费者就只能进行应用级服务引入，好处就是前面所说的，减轻了注册中心的压力等，那么带来的影响就是服务消费者端寻找服务 URL 的逻辑更复杂了。

只要找到了当前引入服务对应的服务 URL，然后生成对应的 Invoker，并最终生成一个 ClusterInvoker。

在进行应用级服务引入时：

1. 首先，根据当前引入的服务接口生成一个 ServiceDiscoveryRegistryDirectory 对象，表示动态服务目录，用来查询并缓存服务提供者信息；

2. 根据接口名去获取 /dubbo/mapping/服务接口名节点的内容，拿到的就是该接口所对应的应用名；

3. 有了应用名之后，再去获取 /services/应用名节点下的实例信息；

4. 依次遍历每个实例，每个实例都有一个编号 revision

5. 根据 metadata-type 进行判断如果是 local：则调用实例上的元数据服务获取应用元数据（MetadataInfo）；如果是 remote：则根据应用名从元数据中心获取应用元数据（MetadataInfo）；

6. 获取到应用元数据之后就进行缓存，key 为 revision，MetadataInfo 对象为 value

7. 这里为什么要去每个实例上获取应用的元数据信息呢？因为有可能不一样，虽然是同一个应用，但是在运行不同的实例的时候，可以指定不同的参数，比如不同的协议，不同的端口，虽然在生产上基本不会这么做，但是 Dubbo 还是支持了这种情况

8. 根据从所有实例上获取到的 MetadataInfo 以及 endpoint 信息，就能知道所有实例上所有的服务 URL（注意：一个接口+一个协议+一个实例 : 对应一个服务URL）

9. 拿到了这些服务 URL 之后，就根据当前引入服务的信息进行过滤，会根据引入服务的接口名+协议名，消费者可以在 @DubboReference 中指定协议，表示只使用这个协议调用当前服务，如果没有指定协议，那么就会去获取 tri、dubbo、rest 这三个协议对应的服务 URL（Dubbo3.0 默认只支持这三个协议）

10. 这样，经过过滤之后，就得到了当前所引入的服务对应的服务 URL 了

11. 根据每个服务 URL 生成对应的 Invoker 对象，并把 Invoker 对象存在 ServiceDiscoveryRegistryDirectory 对象的 invokers 属性中

12. 最后将 ServiceDiscoveryRegistryDirectory 对象生成一个 ClusterInvoker 对象，到时候调用 ClusterInvoker 对象的 invoke() 方法就会进行负载均衡选出某一个 Invoker 进行调用

## MigrationInvoker 的生成

上面分析了接口级服务引入和应用级服务引入，最终都是得到某个服务对应的服务提供者 Invoker，那最终进行服务调用时，到底该怎么选择呢？

所以在 Dubbo3.0 中，可以配置：

```ini
# dubbo.application.service-discovery.migration 仅支持通过 -D 以及 全局配置中心 两种方式进行配置。

dubbo.application.service-discovery.migration=APPLICATION_FIRST

 # 可选值
 # FORCE_INTERFACE，强制使用接口级服务引入
 # FORCE_APPLICATION，强制使用应用级服务引入
 # APPLICATION_FIRST，智能选择是接口级还是应用级，默认就是这个
```

对于前两种强制的方式，没什么特殊，就是上面走上面分析的两个过程，没有额外的逻辑，那对于 APPLICATION\_FIRST 就需要有额外的逻辑了，也就是 Dubbo 要判断，当前所引入的这个服务，应该走接口级还是应用级，这该如何判断呢？

事实上，在进行某个服务的服务引入时，会统一利用 InterfaceCompatibleRegistryProtocol 的 refer 来生成一个 MigrationInvoker 对象，在 MigrationInvoker 中有三个属性：

```java
// 用来记录接口级ClusterInvoker
private volatile ClusterInvoker<T> invoker;

// 用来记录应用级的ClusterInvoker
private volatile ClusterInvoker<T> serviceDiscoveryInvoker; 

// 用来记录当前使用的ClusterInvoker，要么是接口级，要么应用级
private volatile ClusterInvoker<T> currentAvailableInvoker; 
```

一开始构造出来的 MigrationInvoker 对象中三个属性都为空，接下来会利用 MigrationRuleListener 来处理 MigrationInvoker 对象，也就是给这三个属性赋值。

在 MigrationRuleListener 的构造方法中，会从配置中心读取 DUBBO\_SERVICEDISCOVERY\_MIGRATION 组下面的"当前应用名+.migration"的配置项，配置项为 yml 格式，对应的对象为 MigrationRule，也就是可以配置具体的迁移规则，比如：某个接口或某个应用的 MigrationStep（FORCE\_INTERFACE、APPLICATION\_FIRST、FORCE\_APPLICATION），还可以配置 threshold，表示一个阈值，比如：配置为 2，表示应用级 Invoker 数量是接口级 Invoker 数量的两倍时才使用应用级 Invoker，不然就使用接口级数量，可以参考：[cn.dubbo.apache.org/zh/docs/adv…](/assets/post/2024/ruoyi-rce/https://cn.dubbo.apache.org/zh/docs/advanced/migration-invoker/ "https://cn.dubbo.apache.org/zh/docs/advanced/migration-invoker/")

如果没有配置迁移规则，则会看当前应用中是否配置了 migration.step，如果没有，那就从全局配置中心读取 dubbo.application.service-discovery.migration 来获取 MigrationStep，如果也没有配置，那 MigrationStep 默认为 APPLICATION\_FIRST

如果没有配置迁移规则，则会看当前应用中是否配置了 migration.threshold，如果没有配，则 threshold 默认为 -1。

在应用中可以这么配置：

```yaml
dubbo:
	application:
		name: dubbo-springboot-demo-consumer
		parameters:
			migration.step: FORCE_APPLICATION
			migration.threshold: 2
```

确定了 step 和 threshold 之后，就要真正开始给 MigrationInvoker 对象中的三个属性赋值了，先根据 step 调用不同的方法

```java
switch (step) {
 case APPLICATION_FIRST:
 // 先进行接口级服务引入得到对应的ClusterInvoker，并赋值给invoker属性
 // 再进行应用级服务引入得到对应的ClusterInvoker，并赋值给serviceDiscoveryInvoker属性
 // 再根据两者的数量判断到底用哪个，并且把确定的ClusterInvoker赋值给currentAvailableInvoker属性

 migrationInvoker.migrateToApplicationFirstInvoker(newRule);
 break;
 case FORCE_APPLICATION:
 // 只进行应用级服务引入得到对应的ClusterInvoker，
 // 并赋值给serviceDiscoveryInvoker和currentAvailableInvoker属性

 success = migrationInvoker.migrateToForceApplicationInvoker(newRule);
 break;
 case FORCE_INTERFACE:
 default:
 // 只进行接口级服务引入得到对应的ClusterInvoker，
 // 并赋值给invoker和currentAvailableInvoker属性

 success = migrationInvoker.migrateToForceInterfaceInvoker(newRule);
 }
```

这里只需要分析当 step 为 APPLICATION\_FIRST 时，是如何确定最终要使用的 ClusterInvoker 的。

得到了接口级 ClusterInvoker 和应用级 ClusterInvoker 之后，就会利用 DefaultMigrationAddressComparator 来进行判断：

1. 如果应用级 ClusterInvoker 中没有具体的 Invoker，那就表示只能用接口级 Invoker

2. 如果接口级 ClusterInvoker 中没有具体的 Invoker，那就表示只能用应用级 Invoker

3. 如果应用级 ClusterInvoker 和接口级 ClusterInvoker 中都有具体的 Invoker，则获取对应的 Invoker 个数

4. 如果在迁移规则和应用参数中都没有配置 threshold，那就读取全局配置中心的 dubbo.application.migration.threshold 参数，如果也没有配置，则threshold 默认为 0（不是-1了）

5. 用应用级 Invoker 数量 / 接口级 Invoker 数量，得到的结果如果大于等于 threshold，那就用应用级 ClusterInvoker，否则用接口级 ClusterInvoker

threshold 默认为 0，那就表示在既有应用级 Invoker 又有接口级 Invoker 的情况下，就一定会用应用级 Invoker，两个正数相除，结果肯定为正数，当然你自己可以控制 threshold，如果既有既有应用级 Invoker 又有接口级 Invoker 的情况下，你想在应用级 Invoker 的个数大于接口级 Invoker 的个数时采用应用级Invoker，那就可以把 threshold 设置为 1，表示个数相等，或者个数相除之后的结果大于 1 时用应用级 Invoker，否者用接口级 Invoker

这样 MigrationInvoker 对象中的三个数据就能确定好值了，和在最终的接口代理对象执行某个方法时，就会调用 MigrationInvoker 对象的 invoke，在这个invoke 方法中会直接执行 currentAvailableInvoker 对应的 invoker 的 invoker 方法，从而进入到了接口级 ClusterInvoker 或应用级 ClusterInvoker 中，从而进行负载均衡，选择出具体的 DubboInvoer 或 TripleInvoker，完成真正的服务调用。

## 服务调用底层原理

在 Dubbo2.7 中，默认的是 Dubbo 协议，因为 Dubbo 协议相比较于 Http1.1 而言，Dubbo 协议性能上是要更好的。

但是 Dubbo 协议自己的缺点就是不通用，假如现在通过 Dubbo 协议提供了一个服务，那如果想要调用该服务就必须要求服务消费者也要支持 Dubbo 协议，比如想通过浏览器直接调用 Dubbo 服务是不行的，想通过 Nginx 调 Dubbo 服务也是不行得。

而随着企业的发展，往往可能会出现公司内部使用多种技术栈，可能这个部门使用 Dubbo，另外一个部门使用 Spring Cloud，另外一个部门使用 gRPC，那此时部门之间要想相互调用服务就比较复杂了，所以需要一个通用的、性能也好的协议，这就是 Triple 协议。

Triple 协议是基于 Http2 协议的，也就是在使用 Triple 协议发送数据时，会按 HTTP2 协议的格式来发送数据，而 HTTP2 协议相比较于 HTTP1 协议而言，HTTP2是 HTTP1 的升级版，完全兼容 HTTP1，而且 HTTP2 协议从设计层面就解决了 HTTP1 性能低的问题。

另外，Google 公司开发的 gRPC，也基于的 HTTP2，目前 gRPC 是云原生事实上协议标准，包括 k8s/etcd 等都支持 gRPC 协议。

所以 Dubbo3.0 为了能够更方便的和 k8s 进行通信，在实现 Triple 的时候也兼容了 gRPC，也就是可以用 gPRC 的客户端调用 Dubbo3.0 所提供的 triple 服务，也可以用 triple 服务调用 gRPC 的服务。

### Triple 的底层原理分析

就是因为 HTTP2 中的数据帧机制，Triple 协议才能支持 UNARY、SERVER\_STREAM、BI\_STREAM 三种模式。

1. UNARY：就是最普通的，服务端只有在接收到完请求包括的所有的 HEADERS 帧和 DATA 帧之后(通过调用 onCompleted() 发送最后一个 DATA 帧)，才会处理数据，客户端也只有接收完响应包括的所有的 HEADERS 帧和 DATA 帧之后，才会处理响应结果。

2. SERVER\_STREAM：服务端流，特殊的地方在于，服务端在接收完请求包括的所有的 DATA 帧之后，才会处理数据，不过在处理数据的过程中，可以多次发送响应 DATA 帧（第一个 DATA 帧发送之前会发送一个 HEADERS 帧），客户端每接收到一个响应 DATA 帧就可以直接处理该响应 DATA 帧，这个模式下，客户端只能发一次数据，但能多次处理响应 DATA 帧。

3. BI\_STREAM：双端流，或者客户端流，特殊的地方在于，客户端可以控制发送多个请求 DATA 帧（第一个 DATA 帧发送之前会发送一个 HEADERS 帧），服务端会不断的接收到请求 DATA 帧并进行处理，并且及时的把处理结果作为响应 DATA 帧发送给客户端（第一个 DATA 帧发送之前会发送一个 HEADERS 帧），而客户端每接收到一个响应结果 DATA 帧也会直接处理，这种模式下，客户端和服务端都在不断的接收和发送 DATA 帧并进行处理，注意请求 HEADER 帧和响应 HEADERS 帧都只发了一个。

### Triple 请求调用和响应处理

创建一个 Stream 的前提是先得有一个 Socket 连接，所以我们得先知道 Socket 连接是在哪创建的。

在服务提供者进行服务导出时，会按照协议以及对应的端口启动 Server，比如：Triple 协议就会启动 Netty 并绑定指定的端口，等待 Socket 连接，在进行服务消费者进行服务引入的过程中，会生成 TripleInvoker 对象，在构造 TripleInvoker 对象的构造方法中，会利用 ConnectionManager 创建一个 Connection 对象，而Connection 对象中包含了一个 Bootstrap 对象（Netty 中用来建立 Socket 连接的），不过以上都只是创建对象，并不会真正和服务去建立 Socket 连接，所以在生成 TripleInvoker 对象过程中不会真正去创建 Socket 连接，那什么时候创建的呢？

当我们在服务消费端执行以下代码时：`demoService.sayHello("habit")`

demoService 是一个代理对象，在执行方法的过程中，最终会调用 TripleInvoker 的 doInvoke() 方法，在 doInvoke() 方法中，会利用 Connection 对象来判断Socket 连接是否可用，如果不可用并且没有初始化，那就会创建 Socket 连接。

一个 Connection 对象就表示一个 Socket 连接，在 TripleInvoker 对象中也只有一个 Connection 对象，也就是一个 TripleInvoker 对象只对应一个 Socket 连接，这个和 DubboInvoker 不太一样，一个 DubboInvoker 中可以有多个 ExchangeClient，每个 ExchangeClient 都会与服务端创建一个 Socket 连接，所以一个DubboInvoker 可以对应多个 Socket 连接，当然多个 Socket 连接的目的就是提高并发，不过在 TripleInvoker 对象中就不需要这么来设计了，因为可以 Stream机制来提高并发。

以上，我们知道了，当我们利用服务接口的代理对象执行方法时就会创建一个 Socket 连接，就算这个代理对象再次执行方法时也不会再次创建 Socket 连接了，值得注意的是，有可能两个服务接口对应的是一个 Socket 连接，举个例子。

比如服务提供者应用 A，提供了 DemoService 和 HelloService 两个服务，服务消费者应用 B 引入了这两个服务，那么在服务消费者这端，这个两个接口对应的代理对象对应的 TripleInvoker 是不同的两个，但是这两个 TripleInvoker 会公用一个 Socket 连接，因为 ConnectionManager 在创建 Connection 对象时会根据服务 URL 的 address 进行缓存，后续这两个代理对象在执行方法时使用的就是同一个 Socket 连接，但是是不同的 Stream。

Socket 连接创建好之后，就需要发送 Invocation 对象给服务提供者了，因为是基于的 HTTP2，所以要先创建一个 Stream，然后再通过 Stream 来发送数据。

TripleInvoker 中用的是 Netty，所以最终会利用 Netty 来创建 Stream，对应的对象为 Http2StreamChannel，消费端的 TripleInvoker 最终会利用Http2StreamChannel 来发送和接收数据帧，数据帧对应的对象为 Http2Frame，它又分为 Http2DataFrame、Http2HeadersFrame 等具体类型。

正常情况下，会每生成一个数据帧就会通过 Http2StreamChannel 发送出去，但是在 Triple 中有一个小小的优化，会有一个批量发送的思想，当要发送一个数据帧时，会先把数据帧放入一个 WriteQueue 中，然后会从线程池中拿到一个线程调用 WriteQueue 的 flush 方法，该方法的实现为：

```java
private void flush() {
        try {
            QueuedCommand cmd;
            int i = 0;
            boolean flushedOnce = false;
            // 只要队列中有元素就取出来，没有则退出while
            while ((cmd = queue.poll()) != null) {
                // 把数据帧添加到Http2StreamChannel中，
                // 添加并不会立马发送，调用了flush才发送
                cmd.run(channel);
                i++;

                // DEQUE_CHUNK_SIZE=128
                // 连续从队列中取到了128个数据帧就flush一次
                if (i == DEQUE_CHUNK_SIZE) {
                    i = 0;
                    channel.flush();
                    flushedOnce = true;
                }
            }

            // i != 0 表示从队列中取到了数据但是没满128个
            // 如果i=0，flushedOnce=false也flush一次
            if (i != 0 || !flushedOnce) {
                channel.flush();
            }
        } finally {
            scheduled.set(false);

            // 如果队列中又有数据了，则继续会递归调用flush
            if (!queue.isEmpty()) {
                scheduleFlush();
            }
        }
    }
```

总体思想是，只要向 WriteQueue 中添加一个数据帧之后，那就会尝试开启一个线程，要不要开启线程要看 CAS，比如现在有 10 个线程同时向 WriteQueue 中添加了一个数据帧，那么这 10 个线程中的某一个会 CAS 成功，其他会 CAS 失败，那么此时 CAS 成功的线程会负责从线程池中获取另外一个线程执行上面的 flush 方法，从而获取 WriteQueue 中的数据帧然后发送出去。

有了底层这套设计之后，对于 TripleInvoker 而 ，它只需要把要发送的数据封装为数据帧，然后添加到 WriteQueue 中就可以了。

在 TripleInvoker 的 doInvoke() 源码中，在创建完成 Socket 连接后，就会：

1. 基于 Socket 连接先构造一个 ClientCall 对象

2. 根据当前调用的方法信息构造一个 RequestMetadata 对象，这个对象表示，当前调用的是哪个接口的哪个方法，并且记录了所配置的序列化方式，压缩方式，超时时间等

3. 紧接着构造一个 ClientCall.Listener，这个 Listener 是用来处理响应结果的，针对不同的流式调用类型，会构造出不同的 ClientCall.Listener：UNARY：会构造出一个 UnaryClientCallListener，内部包含了一个 DeadlineFuture，DeadlineFuture 是用来控制 timeout 的SERVER\_STREAM：会构造出一个 ObserverToClientCallListenerAdapter，内部包含了调用方法时传入进来的 StreamObserver 对象，最终就是由这个StreamObserver 对象来处理响应结果的BI\_STREAM：和 SERVER\_STREAM 一样，也会构造出来一个 ObserverToClientCallListenerAdapter

4. 紧着着，就会调用 ClientCall 对象的 start 方法创建一个 Stream，并且返回一个 StreamObserver 对象

5. 得到了 StreamObserver 对象后，会根据不同的流式调用类型来使用这个 StreamObserver 对象UNARY：直接调用 StreamObserver 对象的 onNext() 方法来发送方法参数，然后调用 onCompleted 方法，然后返回一个 new AsyncRpcResult(future, invocation)，future 就是 DeadlineFuture，后续会通过 DeadlineFuture 同步等待响应结果的到来，并最终把获取到的响应结果返回给业务方法。SERVER\_STREAM：直接调用 StreamObserver 对象的 onNext() 方法来发送方法参数，然后调用 onCompleted 方法，然后返回一个 new AsyncRpcResult(CompletableFuture.completedFuture(new AppResponse()), invocation)，后续不会同步了，并且返回 null 给业务方法。BI\_STREAM：直接返回 new AsyncRpcResult( CompletableFuture.completedFuture(new AppResponse(requestObserver)), invocation)，也不同同步等待响应结果了，而是直接把 requestObserver 对象返回给了业务方法。所以我们可以发现，不管是哪种流式调用类型，都会先创建一个 Stream，得到对应的一个 StreamObserver 对象，然后调用 StreamObserver 对象的onNext 方法来发送数据，比如发送服务接口方法的入参值，比如一个 User 对象：在发送 User 对象之前，会先发送请求头，请求头中包含了当前调用的是哪个接口、哪个方法、版本号、序列化方式、压缩方式等信息，注意请求头中会包含一些 gRPC 相关的 key，主要就是为了兼容 gRPC然后就是发送请求体然后再对 User 对象进行序列化，得到字节数组然后再压缩字节数组然后把压缩之后的字节数组以及是否压缩标记生成一个 DataQueueCommand 对象，并且把这个对象添加到 writeQueue 中去，然后执行scheduleFlush()，该方法就会开启一个线程从 writeQueue 中获取数据进行发送，发送时就会触发 DataQueueCommand 对象的 doSend 方法进行发送，该方法中会构造一个 DefaultHttp2DataFrame 对象，该对象中由两个属性 endStream，表示是不是 Stream 中的最后一帧，另外一个属性为content，表示帧携带的核心数据，该数据格式为：第一个字节记录请求体是否被压缩紧着的四个字节记录字节数组的长度后面就真正的字节数据以上是 TripleInvoker 发送数据的流程，接下来就是 TripleInvoker 接收响应数据的流程，ClientCall.Listener 就是用来监听是否接收到的响应数据的，不同的流式调用方式会对应不同的 ClientCall.Listener：UNARY：UnaryClientCallListener，内部包含了一个 DeadlineFuture，DeadlineFuture 是用来控制 timeout 的SERVER\_STREAM：ObserverToClientCallListenerAdapter，内部包含了调用方法时传入进来的 StreamObserver 对象，最终就是由这个StreamObserver 对象来处理响应结果的BI\_STREAM：和 SERVER\_STREAM 一样，也会构造出来一个 ObserverToClientCallListenerAdapter那现在要了解的就是，如何知道某个 Stream 中有响应数据，然后触发调用 ClientCall.Listener 对象的相应的方法。要监听某个 Stream 中是否有响应数据，这个肯定是 Netty 来做的，实际上，在之前创建 Stream 时，会向 Http2StreamChannel 绑定一个TripleHttp2ClientResponseHandler，很明显这个 Handler 就是用来处理接收到的响应数据的。在 TripleHttp2ClientResponseHandler 的 channelRead0 方法中，每接收一个响应数据就会判断是 Http2HeadersFrame 还是 Http2DataFrame，然后调用 ClientTransportListener 中对应的 onHeader 方法和 onData 方法：onHeader 方法通过处理响应头，会生成一个 TriDecoder，它是用来解压并处理响应体的onData 方法会利用 TriDecoder 的 deframe() 方法来处理响应体另外如果服务提供者那边调用了 onCompleted 方法，会向客户端响应一个请求头，endStream 为 true，表示响应结束，也会触发执行 onHeader 方法，从而会调用 TriDecoder 的 close() 方法.TriDecoder 的 deframe() 方法在处理响应体数据时，会分为两个步骤：先解析前 5 个字节，先解析第 1 个字节确定该响应体是否压缩了，再解析后续 4 个字节确定响应体内容的字节长度然后再取出该长度的字节作为响应体数据，如果压缩了，那就进行解压，然后把解压之后的字节数组传递给 ClientStreamListenerImpl 的onMessage() 方法，该方法就会按对应的序列化方式进行反序列化，得到最终的对象，然后再调用到最终的 UnaryClientCallListener 或者ObserverToClientCallListenerAdapter 的 onMessage() 方法。TriDecoder 的 close() 方法最终也会调用到 UnaryClientCallListener 或者 ObserverToClientCallListenerAdapter 的 close() 方法。UnaryClientCallListener，构造它时传递了一个 DeadlineFuture 对象：onMessage() 接收到响应结果对象后，会把结果对象赋值给 appResponse 属性onClose() 会取出 appResponse 属性记录的结果对象构造出来一个 AppResponse 对象，然后调用 DeadlineFuture 的 received 方法，从而将方法调用线程接阻塞，并得到响应结果对象。ObserverToClientCallListenerAdapter，构造它时传递了一个 StreamObserver 对象：onMessage() 接收到响应结果对象后，会调用 StreamObserver 对象的 onNext()，并把结果对象传给 onNext() 方法，从触发了程序员的 onNext() 方法逻辑。onClose() 就会调用 StreamObserver 对象的 onCompleted()，或者调用 onError() 方法

### Triple 请求处理和响应结果发送

其实这部分内容和发送请求和处理响应是非常类似的，无非就是把视角从消费端切换到服务端，前面分析的是消费端发送和接收数据，现在要分析的是服务端接收和发送数据。

消费端在创建一个 Stream 后，会生成一个对应的 StreamObserver 对象用来发送数据和一个 ClientCall.Listener 用来接收响应数据，对于服务端其实也一样，在接收到消费端创建 Stream 的命令后，也需要生成一个对应的 StreamObserver 对象用来响应数据以及一个 ServerCall.Listener 用来接收请求数据。

在服务导出时，TripleProtocol 的 export 方法中会开启一个 ServerBootstrap，并绑定指定的端口，并且最重要的是，Netty 会负责接收创建 Stream 的信息，一旦就收到这个信号，就会生成一个 ChannelPipeline，并给 ChannelPipeline 绑定一个 TripleHttp2FrameServerHandler，而这个TripleHttp2FrameServerHandler 就可以用来处理 Http2HeadersFrame 和 Http2DataFrame。

比如在接收到请求头后，会构造一个 ServerStream 对象，该对象有一个 ServerTransportObserver 对象，ServerTransportObserver 对象就会真正来处理请求头和请求体：

1. onHeader() 方法，用来处理请求头比如从请求头中得到当前请求调用的是哪个服务接口，哪个方法构造一个 TriDecoder 对象，TriDecoder 对象用来处理请求体构造一个 ReflectionServerCall 对象并调用它的 doStartCall() 方法，从而生成不同的 ServerCall.ListenerUNARY：UnaryServerCallListenerSERVER\_STREAM：ServerStreamServerCallListenerBI\_STREAM：BiStreamServerCallListener并且在构造这些 ServerCall.Listener 时会把 ReflectionServerCall 对象传入进去，ReflectionServerCall 对象可以用来向客户端发送数据

2. onData() 方法，用来处理请求体，调用 TriDecoder 对象的 deframe 方法来处理请求体，如果是 endStream，那还会调用 TriDecoder 对象的 close 方法

#### TriDecoder：

deframe()：这个方法的作用和客户端时一样的，都是先解析请求体的前 5 个字节，然后解压请全体，然后反序列化得到请求参数对象，然后调用不同的ServerCall.Listener 中的 onMessage()

close()：当客户端调用 onCompleted 方法时，就表示发送数据完毕，此时会发送一个 DefaultHttp2DataFrame 并且 endStream 为 true，从而会触发服务端TriDecoder 对象的 close() 方法，从而调用不同的 ServerCall.Listener 中的 onComplete()

#### UnaryServerCallListener：

1. 在接收到请求头时，会构造 UnaryServerCallListener 对象，没什么特殊的

2. 然后接收到请求体时，请求体中的数据就是调用接口方法的入参值，比如 User 对象，那么就会调用 UnaryServerCallListener 的 onMessage() 方法，在这个方法中会把 User 对象设置到 invocation 对象中

3. 当消费端调用 onCompleted() 方法，表示请求体数据发送完毕，从而触发 UnaryServerCallListener 的 onComplete() 方法，在该方法中会调用 invoke() 方法，从而执行服务方法，并得到结果，得到结果后，会调用 UnaryServerCallListener 的 onReturn() 方法，把结果通过 responseObserver 发送给消费端，并调用 responseObserver 的 onCompleted() 方法，表示响应数据发送完毕，responseObserver 是 ReflectionServerCall 对象的一个 StreamObserver 适配对象（ServerCallToObserverAdapter）。

#### 再来看 ServerStreamServerCallListener：

1. 在接收到请求头时，会构造 ServerStreamServerCallListener 对象，没什么特殊的

2. 然后接收到请求体时，请求体中的数据就是调用接口方法的入参值，比如 User 对象，那么就会调用 ServerStreamServerCallListener 的 onMessage() 方法，在这个方法中会把 User 对象以及 responseObserver 对象设置到 invocation 对象中，这是和 UnaryServerCallListener 不同的地方，UnaryServerCallListener 只会把 User 对象设置给 invocation，而 ServerStreamServerCallListener 还会把 responseObserver 对象设置进去，因为服务 端流需要这个 responseObserver 对象，服务方法拿到这个对象后就可以自己来控制如何发送响应体，并什么时候调用 onCompleted() 方法来表示响应体发送完毕。

3. 当消费端调用 onCompleted() 方法，表示请求体数据发送完毕，从而触发 ServerStreamServerCallListener 的 onComplete() 方法，在该方法中会调用invoke() 方法，从而执行服务方法，从而会通过 responseObserver 对象来发送数据

4. 方法执行完后，仍然会调用 ServerStreamServerCallListener的onReturn() 方法，但是个空方法

#### 再来看最后一个 BiStreamServerCallListener：

1. 在接收到请求头时，会构造 BiStreamServerCallListener 对象，这里比较特殊，会把 responseObserver 设置给 invocation 并执行 invoke() 方法，从而执行服务方法，并执行 onReturn() 方法，onReturn() 方法中会把服务方法的执行结果，也是一个 StreamObserver 对象，赋值给 BiStreamServerCallListener 对象的requestObserver 属性。

2. 这样，在接收到请求头时，服务方法就会执行了，并且得到了一个 requestObserver，它是程序员定义的，是用来处理请求体的，另外的 responseObserver是用来发送响应体的。

3. 紧接着就会收到请求体，从而触发 onMessage() 方法，该方法中会调用 requestObserver 的 onNext() 方法，这样就可以做到，服务端能实时的接收到消费端每次所发送过来的数据，并且进行处理，处理过程中，如果需要响应就可以利用 responseObserver 进行响应

4. 一旦消费端那边调用了 onCompleted() 方法，那么就会触发 BiStreamServerCallListener 的 onComplete 方法，该方法中也就是调用 requestObserver 的onCompleted()，主要就触发程序员自己写的 StreamObserver 对象中的 onCompleted()，并没有针对底层的 Stream 做什么事情。

### 总结

不管是 Unary，还是 ServerStream，还是 BiStream，底层客户端和服务端之前都只有一个 Stream，它们三者的区别在于：

1. Unary：通过流，将方法入参值作为请求体发送出去，而且只发送一次，服务端这边接收到请求体之后，会执行服务方法，得到结果，把结果返回给客户端，也只响应一次。

2. ServerStream：通过流，将方法入参值作为请求体发送出去，而且只发送一次，服务端这边接收到请求体之后，会执行服务方法，并且会把当前流对应的StreamObserver 对象也传给服务方法，由服务方法自己控制如何响应，响应几次，响应什么数据，什么时候响应结束，都由服务方法自己控制。

3. BiStream，通过流，客户端和服务端，都可以发送和响应多次。
