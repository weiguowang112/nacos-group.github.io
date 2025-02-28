---
title: Nacos 的一条注册请求会经历什么？
keywords: [Nacos,注册]
description: 详解 Nacos 的一条注册请求会经历什么？

---

转载且编辑自[Nacos 的一条注册请求会经历什么?](http://www.passjava.cn/#/13.SpringCloud架构剖析/07.Nacos配置注册中心/02.Nacos架构原理①：一条注册请求会经历什么？)

# Nacos 的一条注册请求会经历什么？

# 前言

Nacos 作为注册中心，用来接收客户端（服务实例）发起的注册请求，并将注册信息存放到注册中心进行管理。

> 那么一条注册请求到底会经历哪些步骤呢？

### 知识点预告

先上一张整体的流程图：

![](img/blog/nacos-reigster-mechanism/image-20220413171451070bFSgRzW3Rx0S.png)

- **集群环境**：如果是 Nacos 集群环境，那么拓扑结构是什么样的。
- **组装请求**：客户端组装注册请求，下一步对 Nacos 服务发起远程调用。
- **随机节点**：客户端随机选择集群中的一个 Nacos 节点发起注册，实现负载均衡。
- **路由转发**：Nacos 节点收到注册请求后，看下是不是属于自己的，不是的话，就进行路由转发。
- **处理请求**：转发给指定的节点后，该节点就会将注册请求中的实例信息解析出来，存到自定义的内存结构中。
- **最终一致性**：通过 Nacos 自研的 Distro 协议执行`延迟异步任务`，将注册信息同步给集群中的其他节点，保证了数据的最终一致性。
- **异步重试**：如果注册失败，客户端将会切换 Nacos 节点，再次发起注册请求，保证高可用性。

这些知识点里面还有很多细节，我会通过画图 + 源码剖析的方式给大家解答。如果遇到源码看不太懂的地方，可以多看下我画的图，然后翻下源码，对照着一起看。

小 Tip：本文使用的 Nacos 版本： 2.0.4。

## 一、源头：发起注册

### 1.1 阅读源码的小技巧

在使用 Nacos 组件的的时候，我们加上一个注解 `@EnableDiscoveryClient` 就可以使服务自动注册到 Nacos。

> 那么这个发起注册的地方到底在哪呢？注册信息又是长什么样的呢？

告诉大家一个看源码的小技巧，拿到源码后，不是直接各个文件都看一篇，而是先看源码中带的 example 文件夹。如下图所示，找到 example 的 App 类，里面就有发起注册的实例代码。如下图所示：

![](img/blog/nacos-reigster-mechanism/image-20220412071138017zxK9mw.png)

当然，我们也可以通过官网给的 curl 命令发起 HTTP 请求：

``` SH
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.11&port=8080'
```

**留个问题**：我们都是加一个 Nacos 注解 `@EnableDiscoveryClient`，就会自动把服务实例注册到 Nacos，这个是怎么做到的？

### 1.2 发起注册的流程图

先来看一下代码的流程图：

![](img/blog/nacos-reigster-mechanism/image-20220412071400192DVNL4d.png)

跟着这个流程图，我们 debug 来看下。

### 1.3 组装注册的实例信息

入口的核心代码如下图所示，它会组装注册的`实例信息`，放到一个 instance 变量里面：

![](img/blog/nacos-reigster-mechanism/image-202204111612410590Vmd01.png)

通过代码调试，我们可以看到里面的实例信息长这样：

![](img/blog/nacos-reigster-mechanism/image-20220411160413112vLIk1i.png)

### 1.4 组装注册请求 request

发起注册的核心方法是 doRegisterService()，组装的 request 如下图所示，里面有之前组装的实例信息 instance，还有指定的  namespace（Nacos 的命名空间）、serviceName（服务名），groupName（Nacos 的分组）。

![image-20220411162322668](img/blog/nacos-reigster-mechanism/image-202204111623226683NnG6U.png)

### 1.5 发起远程调用

requestToServer() 方法里面会调用 RpcClient 的 request() 方法：

```java
response = this.currentConnection.request(request, timeoutMills);
```

就是向 Nacos 发起远程调用，如果是 Nacos 集群，则是向集群中的某个 Nacos 节点发起远程调用。

接下来我们看下客户端是如何选择一个 Nacos 节点进行注册的。

## 二、集群环境：分布式的前提

如果是 Nacos 集群环境，客户端会随机选择一个 Nacos 节点发起注册。

### 2.1 搭建好一套Nacos 集群环境

为了讲解客户端是如何注册到 Nacos 集群环境的底层原理，我在本地搭建了一个 Nacos 集群环境，有 3 个 Nacos 服务，它们的 IP 相同，端口号不同。

``` sh
192.168.10.197:8848
192.168.10.197:8858
192.168.10.197:8868
```

![](img/blog/nacos-reigster-mechanism/image-20220408100844549MpxWbbx40la1.png)

然后**服务 A 和服务 B 都是配置了 Nacos 集群**的 IP 和 端口号的，配置如下所示

```java
spring.cloud.nacos.discovery.server-addr
  =192.168.10.197:8848,192.168.10.197:8858,192.168.10.197:8868
```

整体的结构如下图所示，服务 A 和 服务 B 都往 Nacos 集群进行注册。

![](img/blog/nacos-reigster-mechanism/image-20220408101723181kPWAUaDBK5jL.png)

**但是里面有一个问题**：服务 A 注册时，是向所有 Nacos 节点发起注册呢？还是只向其中一个节点发起注册？如果只向一个节点注册，要向哪个节点注册呢？

> 答案：在 Client 发起注册之前，会有一个后台线程随机拿到 Nacos 集群服务列表中的一个地址。

**Nacos 为什么会这样设计？**

- 这其实就是一个负载均衡的思想在里面，每个节点都均匀的分摊请求。
- 保证高可用，当某个节点宕机后，重新拿到其他的 Nacos 节点来建立连接。

接下来我们看下服务 A 是怎么随机拿到一个 Nacos 节点的。

## 三、随机节点：平等的世界

我们来看下客户端是如何随机选择一个节点的，流程图如下：

![](img/blog/nacos-reigster-mechanism/image-20220412085821355AZgLcJ.png)

那么如何找到这些代码逻辑呢？思路是怎么样的？

我们之前讲过，RpcClient 会发起 request 请求，用的是和 Nacos 建立 `currentConnection` 连接来发起调用，代码如下：

```java
// 发起调用
response = this.currentConnection.request(request, timeoutMills);
```

这个 `currentConnection` 是客户端和 Nacos 集群中的某个节点建立的连接，我们找下它在哪里赋值的。代码如下：

```java
// 拿到 Nacos 节点信息
serverInfo = recommendServer.get() == null ? nextRpcServer() : recommendServer.get();
// 连接 Nacos 节点
connectToServer = connectToServer(serverInfo);
// 赋值 currentConnection
this.currentConnection = connectToServer;
```

而连接的信息是通过参数 serverInfo 传进去的，所以我们再看下 serverInfo 在哪里赋值的。

这个 nextRpcServer() 方法里面会**拿到一个随机的 Nacos 地址**：

```java
// 一个 int 随机数，范围 [0 ~ Nacos 个数)
currentIndex.set(new Random().nextInt(serverList.size()));
// index 自增 1
int index = currentIndex.incrementAndGet() % getServerList().size();
// 返回 Nacos 地址
return getServerList().get(index);
```

**小结**：客户端生成一个随机数，然后通过这个随机数从 Nacos 服务列表中拿到一个 Nacos 服务地址返回给客户端，然后客户端通过这个地址和 Nacos 服务建立连接。Nacos 服务列表中的节点都是平等的，随机拿到的任何一个节点都是可以用来发起调用的。

## 四、路由转发：不是我的菜

### 4.1 发起和转发请求的流程

为了演示发起注册的流程，我在这里模拟了一个注册请求。

用的是 curl 命令，对 Nacos 节点（127.0.0.1:8848）发起注册请求：

``` SH
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.11&port=8080'
```

**请求 URL**：/nacos/v1/ns/instance

**请求参数**：

- serviceName=nacos.naming.serviceName
- ip=20.18.7.11
- port=8080'

之前我们讲到，Nacos 的有多个节点可以分别处理请求，当节点发现这个请求不是属于自己的，就会进行转发。

如下图所示：

服务 A 随机选择一个 Nacos 节点（图中为 Nacos1）发起注册请求，请求参数中包含了实例信息，Nacos 1 根据实例信息 hash + 取模拿到正确的节点，如果不属于自己，则将请求转发给其他节点（图中为 Nacos2）

![](img/blog/nacos-reigster-mechanism/image-20220412215250738gU1BYV.png)

那么路由转发的细节是怎么样的？这个就涉及到 Distro 协议了，我们接着往下看。

### 4.1 路由转发的逻辑

其实 Nacos 节点的路由转发逻辑比较简单，先来看下流程图：

![](img/blog/nacos-reigster-mechanism/image-20220412184102530MvbD7W.png)

步骤如下：

- ①  Nacos 节点从客户端发起的 request 中拿到客户端的实例信息生成 distroTag，如 IP + port 或 service name。
- ②  Nacos 根据 distroTag 生成 hash 值。
- ③  用 hash 值对 Nacos 节点数进行`取余`，拿到余数，比如 0、1、2、3。
- ④ 根据余数从 Nacos 节点列表中拿到指定的节点地址。

**我没看懂的点**：我这里启动了三个 Nacos 节点，如下图所示的 三个 Running 节点。但是为什么 Nacos 的 ServersList 会多了一个 192.168.10.197:8848的节点？

> 开发者回答：
> nacos-server存在一个机制，在启动的时候会检查cluster.conf中配置的member内容，如果发现自身不在member列表中，就会将自身地址加入到member列表中。
> 图中member显示有127.0.0.1的3个节点和一个192.168.10.197节点，说明nacos-server获取到的ip是192.168.10.197，但是cluster.conf中配置的ip是127.0.0.1。
> 需要解决的话有两种方法，一种是严格按照上文中的配置，配置ip为192.168.10.197，另一种方式是在启动服务是设置JVM参数-Dnacos.server.ip=127.0.0.1
> 这个配置错误同样常见于使用K8S搭建Nacos集群，域名和hostname互相混用时出现。

![IDEA 启动了三个 nacos 节点](img/blog/nacos-reigster-mechanism/image-202204122033417675s0J4F.png)

![nacos 控制台有四个节点](img/blog/nacos-reigster-mechanism/image-20220413153431838s2Wj4W.png)

### 4.2 路由转发源码分析

入口文件是 DistroFilter.java：

```SH
naming/src/main/java/com/alibaba/nacos/naming/web/DistroFilter.java
```

请求会先到 DistroFilter 类的 doFilter() 方法，拿到正确的节点地址后，将请求转发出去。

获取需要转发节点地址的代码如下：

```java
// 找到 Nacos 集群中的目标节点
final String targetServer = distroMapper.mapSrv(distroTag);

// mapSrv 方法会先 hash，然后再取模，responsibleTag的值类似这样："20.18.7.11:8080"
int index = distroHash(responsibleTag) % servers.size();

// distroHash 方法里面会对 客户端的 ip+port 字符串或者服务名字符串 进行 hash
Math.abs(responsibleTag.hashCode() % Integer.MAX_VALUE);
```

不论是自己处理注册请求还是转发给其他节点来处理，都会把实例信息存储起来，那么是如何进行存储的？

## 五、处理请求：快到碗里来

Nacos 目前有两个版本，v1 和 v2，如果是 v1，则是 instanceController 来处理注册请求，否则用 instanceControllerV2。本篇我们只讲解 v1 版本是怎么处理请求的。

先上流程图：

![添加实例信息的流程](img/blog/nacos-reigster-mechanism/image-20220413164932907KHTvVM.png)

测试用的发起注册的命令：

``` SH
curl -X POST 'http://127.0.0.1:8858/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.11&port=8080'
```

核心代码就是这个：

![服务端注册实例的方法](img/blog/nacos-reigster-mechanism/image-20220413160148289ylcS1n.png)

有一个 synchronized 锁，将临时的实例信息存放起来，所以重点看下 这个 consistencyService.put() 方法做了什么事情。

先看下源码：

```java
onPut(key, value);
// 开启 1s 的延迟任务，将数据同步给其他 Nacos 节点
distroProtocol.sync(new      DistroKey(key,KeyBuilder.INSTANCE_LIST_KEY_PREFIX),DataOperation.CHANGE,
                DistroConfig.getInstance().getSyncDelayMillis());
```

这里面做了三件事情：

- ① 将实例信息存放到内存缓存 concurrentHashMap 里面。
- ② 添加一个任务到 BlockingQueue 里面，这个任务就是将最新的实例列表通过 UDP 的方式推送给所有客户端（服务实例），这样客户端就拿到了最新的服务实例列表。
- ③ 开启 1s 的延迟任务，将数据通过给其他 Nacos 节点。

**注意**：针对第二点和第三点，属于 Distro 一致性协议的一部分，里面的内容还比较多，我们放到下一讲专门来讲。

**知识点预告**：

- 这里的存储实例和同步的方式和 Eureka 有什么区别？Eureka 用的三层缓存架构，Nacos 用的 CopyOnWrite 技术。

- 如何推送给所有客户端的？UDP 方式。
- 如何同步给 Nacos 其他节点的？Distro 一致性协议。

## 六、总结

本文通过发起一条注册请求，讲解了 Nacos 客户端如何随机选择节点、Nacos Server 如何路由、Nacos Server 如何存储注册实例。

**核心流程**：

![](img/blog/nacos-reigster-mechanism/image-20220413171451070bFSgRz-20220530201153679.png)
