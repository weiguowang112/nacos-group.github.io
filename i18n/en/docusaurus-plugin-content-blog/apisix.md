---
title: Apache APISIX Realizes Service Discovery Based on Nacos
keywords: [Apache APISIX]
description: This article introduces the basic concepts of Apache APISIX and Nacos and Service Registry, and shows you the specific operation of Apache APISIX to realize service discovery based on Nacos.
---

# Background information

## Apache APISIX Introduction

**Apache APISIX** is a dynamic, real-time, high-performance API gateway.

APISIX provides rich traffic management features such as load balancing, dynamic upstream, canary release, circuit breaking, authentication, observability, and more. Apache APISIX not only has many practical plug-ins, but also supports plugin dynamic change and Hot-loading.

## Nacos Introduction

Nacos is an easy-to-use, open source platform for dynamic service discovery, service configuration and service management.  It provides a set of simple and useful features enabling you to realize dynamic service discovery, service configuration, service metadata and traffic management. Nacos makes it easier and faster to construct, deliver and manage your microservices platform. It is the infrastructure that supports a service-centered modern application architecture with a microservices or cloud-native approach.

# Service Registry

Service Registry is the core component of service management, similar to the role of directory service, and one of the most basic facilities in the microservices architecture. It is mainly used to store service information, such as service provider URL, routing information, and so on. The service registry is implemented by mapping complex service-side information to simple and understandable information for the client.

The core functions of the Service Registry are as follows:

- Service registration: Service providers register with the Service Registration Center.

- Service discovery: Service consumers can find the call routing information of service providers through the registry.

- Health check: Ensure that service nodes registered with the service registry can be invoked normally, and avoid the waste of call resources caused by invalid nodes.

## Why do you need a service registry?

The registry is essentially to decouple service providers and service consumers. In the microservice system, each business service will call each other frequently, and the IP, port and other routing information of each service need to be managed uniformly. But how do you manage it? You can provide information about existing services to a unified service registry for management through the Service Registration function of the Service Registry.

From the above description, you can know that the registry can help users quickly find services and service addresses through mapping. As business updates iterate, services change frequently. Clients can still pull a list of services through the service discovery function of the registry after registering new services or service downtime on the service side. If the service node of the registry changes, the registry sends a request to notify the client to pull again.

If the service on the server side suddenly goes down and there is no feedback to the service registry, the client can show the service side its service status by actively reporting the heartbeat at regular intervals through the health check function of the service registry. If the service status is abnormal, the service registry will be notified, and the service registry can remove the down service nodes in time to avoid waste of resources.If the service on the server side suddenly goes down and there is no feedback to the service registry, the client can show the service side its service status by actively reporting the heartbeat at regular intervals through the health check function of the service registry. If the service status is abnormal, the service registry will be notified, and the service registry can remove the down service nodes in time to avoid waste of resources.

## What application scenarios does Apache APISIX + Nacos provide for users?

Apache APISIX + Nacos can centralize business-independent control of each microservice node into Apache APISIX for unified management, that is, the ability to proxy and route forwarding interface services through Apache APISIX. After each microservice is registered on Nacos, Apache APISIX can obtain a list of services through Nacos's service discovery function and find the corresponding service address to implement dynamic proxy.

![img](/img/blog/apisix_en.png)

# Apache APISIX realizes service discovery based on Nacos

## Prerequisites

This article is based on the following environments.

- OS: Centos 7.9.
- Apache APISIX 12.1.0, please refer to: [Apache APISIX how-to-bulid](https://apisix.apache.org/docs/apisix/how-to-build).
- Nacos 2.0.4, please refer to: [Nacos quick start](https://nacos.io/en-us/docs/quick-start.html)
- Node.js, please refer to: [node.js Installation](https://github.com/nodejs/help/wiki/Installation)

## Step 1: Service Register

1. Use Node.js's Koa framework starts a simple test service on port 3005 as [upstream](https://apisix.apache.org/docs/apisix/admin-api#upstream).

```JavaScript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3005);
```

2. Register the service on the command line by requesting the Nacos Open API.

```Bash
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=APISIX-NACOS&ip=127.0.0.1&port=3005&ephemeral=false'
```

3. After service registration, use the following command to query the current service.

```Bash
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=APISIX-NACOS'
```

Examples of correct returned results are as follows:

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

## Step 2:Added Nacos Route

Create a new route using the Admin API provided by Apache APISIX. APISIX selects the service discovery type to use through the `upstream.discovery_type` field. `upstream.service_name`  needs to be associated with the corresponding service name of the registry. Therefore, when creating a route, specify the service discovery type as Nacos.

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

In the above command, the request header `X-API-KEY` is the access token of the admin API, which can be viewed under `apisix.admin_key.key` in the`conf/config.yaml` file.

After successful addition, examples of correct returned results are as follows:

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

In addition, you can also pass other service related parameters in `upstream.discovery_args`   to specify the namespace or group where the service is located. For details, please refer to the following table:

| **Name**     | **Type** | **Requirement** | **Default**    | **Valid** | **Description**           |
| ------------ | -------- | --------------- | -------------- | --------- | ------------------------- |
| namespace_id | string   | optional        | public         |           | This parameter is used to specify the namespace of the corresponding service |
| group_name   | string   | optional        | DEFAULT_GROUP  |           | This parameter is used to specify the group of the corresponding service     |

## Step 3: Verify configuration results

Use the following command to send the request to the route to be configured.

```Shell
curl -i http://127.0.0.1:9080/nacos/
```

Examples of correct returned results are as follows:

```Apache
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 11
Connection: keep-alive
Date: Thu, 27 Jan 2022 00:48:26 GMT
Server: APISIX/2.12.0

Hello World
```

It can be seen from the example that the new route in Apache APISIX can find the correct service address through Nacos service discovery and respond normally.

# Summary

This article introduces the concept of registry and how Apache APISIX cooperates with Nacos to implement routing proxy based on service discovery. How to use Apache APISIX with Nacos in actual scenarios depends on the specific business requirements and past technical architecture.

To get more information about the `nacos` plugin description and full configuration list, you can refer to the [official documentation](https://apisix.apache.org/zh/docs/apisix/discovery/nacos). 
