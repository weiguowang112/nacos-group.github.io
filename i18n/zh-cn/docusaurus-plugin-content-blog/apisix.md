---
title: Apache APISIX 基于 Nacos 实现服务发现
keywords: [Apache APISIX]
description: 本文为您介绍 Apache APISIX、Nacos 基本概念以及注册中心的作用，并为您展示了 Apache APISIX 基于 Nacos 实现服务发现的具体操作
---

# 背景信息

## 关于 Apache APISIX

Apache APISIX 是一个动态、实时、高性能的 API 网关，提供负载均衡、动态上游、灰度发布、服务熔断、身份认证、可观测性等丰富的流量管理功能。Apache APISIX 不仅拥有众多实用的插件，而且支持插件动态变更和热插拔。

## 关于 Nacos

**Nacos** 是阿里巴巴开源的一个易于使用的动态服务发现、配置和服务管理平台。它提供了一组简单易用的特性集，可以帮助您快速实现动态服务发现，服务配置，服务元数据及流量管理，让您更敏捷和容易地构建，交付和管理微服务平台。Nacos 是构建以“服务”为中心的现代应用架构（例如微服务范式、云原生范式）的服务基础设施。

# 注册中心

## 什么是注册中心

服务注册中心是服务要实现服务化管理的核心组件，类似于目录服务的作用，也是微服务架构中最基础的设施之一，主要用来存储服务信息，譬如服务提供者 URL 、路由信息等。注册中心的实现是通过一种映射的方式，将复杂的服务端信息映射为简单易懂的信息提供给客户端。

注册中心的核心功能为以下三点：

1. 服务注册：**服务提供方**向**注册中心**进行注册。
2. 服务发现：**服务消费方**可以通过注册中心寻找到服务提供方的调用路由信息。
3. 健康检测：确保注册到注册中心的服务节点是可以被正常调用的，避免无效节点导致的调用资源浪费等问题。

## 为什么需要注册中心？

注册中心本质上是为了**解耦服务提供者和服务消费者**，在微服务体系中，各个业务服务之间会频繁互相调用，并且需要对各个服务的 IP、port 等路由信息进行统一的管理。但是要如何进行管理呢？我们可以通过注册中心的 **服务注册** 功能将已有服务的相关信息提供到统一的注册中心进行管理。

通过上述描述，您可以了解到注册中心可以帮助用户通过映射快速找到服务和服务地址。随着业务更新迭代，服务会频繁发生变化，在服务端中注册了新的服务或者服务宕机后，客户端仍然可以通过注册中心的 **服务发现** 功能拉取服务列表，如果注册中心的服务节点发生变更，注册中心会发送请求通知客户端重新拉取。

如果服务端的服务突然宕机，并且没有向注册中心反馈，客户端可以通过注册中心的**健康检查**功能，进行固定时间间隔的主动上报心跳方式向服务端表明自己的服务状态。如果服务状态异常，则会通知注册中心，注册中心可以及时把已经宕机的服务节点进行剔除，避免资源的浪费。

## Apache APISIX + Nacos 为用户提供了什么应用场景？

Apache APISIX + Nacos 可以将各个微服务节点中与业务无关的各项控制，集中在 Apache APISIX 中进行统一管理，即通过Apache APISIX 实现接口服务的代理和路由转发的能力。各个微服务在 Nacos 上注册后，Apache APISIX 可以通过 Nacos 的服务发现功能获取服务列表，查找对应的服务地址从而实现动态代理。

![img](/img/blog/apisix.png)

# Apache APISIX 基于 Nacos 实现服务发现

## 前提条件

本文操作基于以下环境进行。

- 操作系统 Centos 7.9。
- 已安装 Apache APISIX 12.1.0，详情请参考：[Apache APISIX how-to-bulid](https://apisix.apache.org/zh/docs/apisix/how-to-buildhttps://apisix.apache.org/zh/docs/apisix/how-to-build)。
- 已安装 Nacos 2.0.4 及以上版本，详情请参考：[quick start](https://nacos.io/zh-cn/docs/quick-start.html)。
- 已安装 Node.js，详情请参考：[node.js Installation](https://github.com/nodejs/help/wiki/Installation)。

## 步骤一：服务注册

1. 使用 Node.js 的 Koa 框架在 3005 端口启动一个简单的测试服务作为[上游（Upstream）](https://apisix.apache.org/zh/docs/apisix/admin-api#upstream)。

```JavaScript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3005);
```

2. 在命令行中通过请求 Nacos Open API 的方式进行服务注册。

```Bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=APISIX-NACOS&ip=127.0.0.1&port=3005&ephemeral=false'
```

3. 执行服务注册后使用以下命令查询当前服务情况。

```Bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=APISIX-NACOS'
```

正确返回结果示例如下：

```JSON
{
  "name": "DEFAULT_GROUP@@APISIX-NACOS",
  "groupName": "DEFAULT_GROUP",
  "clusters": "",
  "cacheMillis": 10000,
  "hosts": [
    {
      "instanceId": "127.0.0.1#3005#DEFAULT#DEFAULT_GROUP@@APISIX-NACOS",
      "ip": "127.0.0.1",
      "port": 3005,
      "weight": 1.0,
      "healthy": true,
      "enabled": true,
      "ephemeral": true,
      "clusterName": "DEFAULT",
      "serviceName": "DEFAULT_GROUP@@APISIX-NACOS",
      "metadata": {},
      "instanceHeartBeatInterval": 5000,
      "instanceHeartBeatTimeOut": 15000,
      "ipDeleteTimeout": 30000,
      "instanceIdGenerator": "simple"
    }
  ],
  "lastRefTime": 1643191399694,
  "checksum": "",
  "allIPs": false,
  "reachProtectionThreshold": false,
  "valid": true
}
```

## 步骤二：新增 Nacos 路由

使用 Apache APISIX 提供的 Admin API 创建一个新的[路由（Route）](https://apisix.apache.org/zh/docs/apisix/admin-api#route)，APISIX 通过 `upstream.discovery_type` 字段选择使用的服务发现类型，`upstream.service_name` 需要与注册中心的对应服务名进行关联，因此创建路由时指定服务发现类型为 `nacos` 。

```Shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "uri": "/nacos/*",
    "upstream": {
        "service_name": "APISIX-NACOS",
        "type": "roundrobin",
        "discovery_type": "nacos"
    }
}'
```

在上述命令中，请求头 `X-API-KEY` 是 Admin API 的访问 token，可以在 `conf/config.yaml` 文件中的 `apisix.admin_key.key` 查看。

添加成功后，正确返回结果示例如下：

```JSON
{
  "action": "set",
  "node": {
    "key": "\/apisix\/routes\/1",
    "value": {
      "update_time": 1643191044,
      "create_time": 1643176603,
      "priority": 0,
      "uri": "\/nacos\/*",
      "upstream": {
        "hash_on": "vars",
        "discovery_type": "nacos",
        "scheme": "http",
        "pass_host": "pass",
        "type": "roundrobin",
        "service_name": "APISIX-NACOS"
      },
      "id": "1",
      "status": 1
    }
  }
}
```

除此之外，您还可以在 `upstream.discovery_args` 中传递其他服务相关参数用于指定服务所在的命名空间或组别，具体内容可参考下表：

| **名字**     | **类型** | **可选项** | **默认值**    | **有效值** | **说明**           |
| ------------ | -------- | ---------- | ------------- | ---------- | ------------------ |
| namespace_id | string   | 可选       | public        |            | 服务所在的命名空间 |
| group_name   | string   | 可选       | DEFAULT_GROUP |            | 服务所在的组       |

## 步骤三：验证配置结果

使用以下命令发送请求至需要配置的路由。

```Shell
curl -i http://127.0.0.1:9080/nacos/
```

正常返回结果示例如下：

```Apache
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 11
Connection: keep-alive
Date: Thu, 27 Jan 2022 00:48:26 GMT
Server: APISIX/2.12.0

Hello World
```

通过示例看到，Apache APISIX 中新增的路由已经可以通过 Nacos 服务发现找到正确的服务地址，并正常响应。

# 总结

本文为大家介绍了注册中心的概念以及 Apache APISIX 如何配合 Nacos 实现基于服务发现的路由代理。实际场景中如何进行 Apache APISIX 与 Nacos 的配合使用，您需要看具体的业务需求和过往技术架构。关于 `nacos` 插件的更多说明和完整配置信息，可参考官网文档：[nacos](https://apisix.apache.org/zh/docs/apisix/discovery/nacos)。
