# Nacos打通CMDB实现就近访问

CMDB在企业中，一般用于存放与机器设备、应用、服务等相关的元数据。一般当企业的机器及应用达到一定规模后就需要这样一个系统来存储和管理它们的元数据。有一些广泛使用的属性例如机器的IP、主机名、机房、应用、region等，这些数据一般会在机器部署时录入到CMDB，运维或者监控平台会使用这些数据进行展示或者相关的运维操作。

在服务进行多机房或者多地域部署时，跨地域的服务访问往往延迟较高，一个城市内的机房间的典型网络延迟在1ms左右，而跨城市的网络延迟，例如上海到北京大概为30ms。此时自然而然的一个想法就是能不能让服务消费者和服务提供者进行同地域访问。阿里巴巴集团内部很早就意识到了这样的需求，在内部的实践中，这样的需求是通过和CMDB打通来实现的。在服务发现组件中，对接CMDB，然后通过配置的访问规则，来实现服务消费者到服务提供者的同地域优先，这样的调用每天都在阿里巴巴集团内部大量执行。

![](https://cdn.nlark.com/lark/0/2018/png/15356/1544702277705-0bbfca60-6629-477c-92bb-1a690e68f9cd.png#align=left&display=inline&height=330&originHeight=330&originWidth=448&status=done&width=448)<br />图1 服务的同地域优先访问

这实际上就是一种负载均衡策略，在Nacos的规划中，丰富的服务端的可配置负载均衡策略是我们的重要发展方向，这与当前已有的注册中心产品不太一样。在设计如何在开源的场景中，支持就近访问的时候，与企业自带的CMDB集成是我们考虑的一个核心问题。除此之外，我们也在考虑将Nacos自身扩展为一个实现基础功能的CMDB。无论如何，我们都需要能够从某个地方获取IP的环境信息，这些信息要么是从企业的CMDB中查询而来，要么是从自己内置的存储中查询而来。

<a name="pwyxgn"></a>

## [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#pwyxgn)CMDB插件机制

先不考虑如何将CMDB的数据应用于负载均衡，我们需要首先在Nacos里将CMDB的数据通过某种方法获取。在实际使用中，基本上每个公司都会通过购买或者自研搭建自己的CMDB，那么为了能够解耦各个企业的CMDB具体实现，一个比较好的策略是使用SPI机制，约定CMDB的抽象调用接口，由各个企业添加自己的CMDB插件，无需任何代码上的重新构建，即可在运行状态下对接上企业的CMDB。

![](https://cdn.nlark.com/lark/0/2018/png/15356/1544842539697-cca20e3d-0f78-45b8-92b9-3b7559e838b2.png#align=left&display=inline&height=394&originHeight=394&originWidth=295&status=done&width=295)<br />图2 Nacos CMDB SPI机制原理

如图2所示，Nacos定义了一个SPI接口，里面包含了与第三方CMDB约定的一些方法。用户依照约定实现了相应的SPI接口后，将实现打成jar包放置到Nacos安装目录下，重启Nacos即可让Nacos与CMDB的数据打通。整个流程并不复杂，但是理解CMDB SPI接口里方法和相应概念的含义不太简单。在这里对CMDB机制的相关概念和接口含义做一个详细说明。

<a name="ga38al"></a>

## [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#ga38al)CMDB抽象概念

<a name="d1gdtg"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#d1gdtg)实体（Entity）
实体是作为CMDB里数据的承载方，在一般的CMDB中，一个实体可以指一个IP、应用或者服务。而这个实体会有很多属性，例如IP的机房信息，服务的版本信息等。

<a name="hig8ag"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#hig8ag)实体类型（Entity Type）
我们并不限定实体一定是IP、应用或者服务，这取决于实际的业务场景。Nacos有计划在未来支持不同的实体类型，不过就目前来说，服务发现需要的实体类型是IP。

<a name="bm37ew"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#bm37ew)标签（Label）
Label是我们抽象出的Entity属性，Label定义为一个描述Entity属性的K-V键值对。Label的key和value的取值范围一般都是预先定义好的，当需要对Label进行变更，如增加新的key或者value时，需要调用单独的接口并触发相应的事件。一个常见的Label的例子是IP的机房信息，我们认为机房（site）是Label的key，而机房的集合（site1, site2, site3）是Label的value，这个Label的定义就是：site: {site1, site2, site3}。

<a name="1osqbb"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#1osqbb)实体事件（Entity Event）
实体的标签的变更事件。当CMDB的实体属性发生变化，需要有一个事件机制来通知所有订阅方。为了保证实体事件携带的变更信息是最新准确的，这个事件里只会包含变更的实体的标识以及变更事件的类型，不会包含变更的标签的值。

<a name="3vu8pv"></a>

## [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#3vu8pv)CMDB约定接口

在设计与CMDB交互接口的时候，我们参考了内部对CMDB的访问接口，并与若干个外部客户进行了讨论。我们最终确定了以下要求第三方CMDB插件必须实现的接口：

<a name="hc8tsu"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#hc8tsu)获取标签列表
```java
Set<String> getLabelNames();
```
这个方法将返回CMDB中需要被Nacos识别的标签名集合，CMDB插件可以按需决定返回什么标签个Nacos。不在这个集合的标签将会被Nacos忽略，即使这个标签出现在实体的属性里。我们允许这个集合会在运行时动态变化，Nacos会定时去调用这个接口刷新标签集合。

<a name="2v2vks"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#2v2vks)获取实体类型
```java
Set<String> getEntityTypes();
```
获取CMDB里的实体的类型集合，不在这个集合的实体类型会被Nacos忽略。服务发现模块目前需要的实体类似是ip，如果想要通过打通CMDB数据来实现服务的高级负载均衡，请务必在返回集合里包含“ip”。
<a name="sw9ryi"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#sw9ryi)获取标签详情
```java
Label getLabel(String labelName);
```
获取标签的详细信息。返回的Label类里包含标签的名字和标签值的集合。如果某个实体的这个标签的值不在标签值集合里，将会被视为无效。

<a name="va70wg"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#va70wg)查询实体的标签值
```java
String getLabelValue(String entityName, String entityType, String labelName);
String getLabelValues(String entityName, String entityType);
```
这里包含两个方法，一个是获取实体某一个标签名对应的值，一个是获取实体所有标签的键值对。参数里包含实体的值和实体的类型。注意，这个方法并不会在每次在Nacos内部触发查询时去调用，Nacos内部有一个CMDB数据的缓存，只有当这个缓存失效或者不存在时，才会去访问CMDB插件查询数据。为了让CMDB插件的实现尽量简单，我们在Nacos内部实现了相应的缓存和刷新逻辑。
<a name="byohax"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#byohax)查询实体
```java
String getAllEntities();
Entity getEntity(String entityName, String entityType);
```
查询实体包含两个方法：查询所有实体和查询单个实体。查询单个实体目前其实就是查询这个实体的所有标签，不过我们将这个方法与获取所有标签的方法区分开来，因为查询单个实体方法后面可能会进行扩展，比查询所有标签获取的信息要更多。

查询所有实体则是一次性将CMDB的所有数据拉取过来，该方法可能会比较消耗性能，无论是对于Nacos还是CMDB。Nacos内部调用该方法的策略是通过可配置的定时任务周期来定时拉取所有数据，在实现该CMDB插件时，也请关注CMDB服务本身的性能，采取合适的策略。

<a name="tgn5ut"></a>

### [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#tgn5ut)查询实体事件
```java
List<EntityEvent> getEntityEvents(long timestamp);
```
这个方法意在获取最近一段时间内实体的变更消息，增量的去拉取变更的实体。因为Nacos不会实时去访问CMDB插件查询实体，需要这个拉取事件的方法来获取实体的更新。参数里的timestamp为上一次拉取事件的时间，CMDB插件可以选择使用或者忽略这个参数。
<a name="p7g6dw"></a>

## [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#p7g6dw)CMDB插件开发流程
参考 [https://github.com/nacos-group/nacos-examples](https://github.com/nacos-group/nacos-examples)，这里已经给出了一个示例plugin实现。<br />具体步骤如下：

1. 新建一个maven工程，引入依赖nacos-api:
```
<dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-api</artifactId>
            <version>0.7.0</version>
        </dependency>
```

2. 引入打包插件：
```
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
            </plugin>
```

3. 定义实现类，继承com.alibaba.nacos.api.cmdb.CmdbService，并实现相关方法。<br />![](https://cdn.nlark.com/lark/0/2018/png/15356/1543916500193-213df77a-096d-4fd9-a283-85241a856fbf.png#align=left&display=inline&height=116&originHeight=116&originWidth=585&status=done&width=585)<br />
3. 在src/main/resource/目录下新建目录：META-INF/services<br />![](https://cdn.nlark.com/lark/0/2018/png/15356/1543916595978-fd322205-16c1-4a95-9cdc-4a6292ee3b66.png#align=left&display=inline&height=96&originHeight=96&originWidth=379&status=done&width=379)<br />
3. 在src/main/resources/META-INF/services目录下新建文件com.alibaba.nacos.api.cmdb.CmdbService，并在文件里将第三步中创建的实现类全名写入该文件:<br />![](https://cdn.nlark.com/lark/0/2018/png/15356/1545036650034-75d11aee-8738-485f-9426-52e560b059cd.png#align=left&display=inline&height=136&originHeight=178&originWidth=944&status=done&width=719)<br />
3. 代码自测完成后，执行命令进行打包：
```
mvn package assembly:single -Dmaven.test.skip=true
```

7. 将target目录下的包含依赖的jar包上传到nacos CMDB插件目录：
```
{nacos.home}/plugins/cmdb
```

8. 在nacos的application.properties里打开加载插件开关：
```
nacos.cmdb.loadDataAtStart=true
```

9. 重启nacos Server，即可加载到您实现的nacos-cmdb插件获取您的CMDB数据。<br />

<a name="5mpctx"></a>

## [](https://yuque.alibaba-inc.com/nacos/opensource/uk8inc/edit#5mpctx)使用Selector实现同机房优先访问
在拿到CMDB的数据之后，就可以运用CMDB数据的强大威力来实现多种灵活的负载均衡策略了，下面举例来说明如何使用CMDB数据和Selector来实现就近访问。

假设目前Nacos已经通过CMDB拿到了一些IP的机房信息，且它们对应的标签信息如下：
```
11.11.11.11
    site: x11

22.22.22.22
    site: x12

33.33.33.33
    site: x11

44.44.44.44
    site: x12

55.55.55.55
    site: x13
```

11.11.11.11、22.22.22.22、33.33.33.33、44.44.44.44和55.55.55.55.55都包含了标签site，且它们对应的值分别为x11、x12、x11、x12、x13。我们先注册一个服务，下面挂载IP11.11.11.11和22.22.22.22。<br />![](https://cdn.nlark.com/lark/0/2018/png/15356/1545035855381-5d9dcfad-75ab-43ad-a084-8ae4a65f914c.png#align=left&display=inline&height=307&originHeight=516&originWidth=1254&status=done&width=747)<br />图3 服务详情

然后我们修改服务的“服务路由类型”，并配置为基于同site优先的服务路由：<br />![](https://cdn.nlark.com/lark/0/2018/png/15356/1545035973200-497c0649-b652-4c36-bf6c-7cddfc5b75c6.png#align=left&display=inline&height=499&originHeight=499&originWidth=610&status=done&width=610)<br />图4 编辑服务路由类型

这里我们将服务路由类型选择为标签，然后输入标签的表达式：

```
CONSUMER.label.site = PROVIDER.label.site
```

这个表达式的格式和我们抽象的Selector机制有关，具体将会在另外一篇文章中介绍。在这里您需要记住的就是，任何一个如下格式的表达式：

```
CONSUMER.label.labelName = PROVIDER.label.labelName
```
将能够实现基于同labelName优先的负载均衡策略。

然后假设服务消费者的IP分别为33.33.33.33、44.44.44.44和55.55.55.55，它们在使用如下接口查询服务实例列表：

```
naming.selectInstances("nacos.test.1", true)
```

那么不同的消费者，将获取到不同的实例列表。33.33.33.33获取到11.11.11.11，44.44.44.44将获取到22.22.22.22，而55.55.55.55将同时获取到11.11.11.11和22.22.22.22。