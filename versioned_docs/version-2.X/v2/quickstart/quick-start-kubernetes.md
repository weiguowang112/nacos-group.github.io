---
title: Kubernetes Nacos
keywords: [nacos,kubernetes]
description: 本项目包含一个可构建的Nacos Docker Image，旨在利用 StatefulSets 在 Kubernetes上部署 Nacos。
---

# Kubernetes Nacos

本项目包含一个可构建的Nacos Docker Image，旨在利用StatefulSets在[Kubernetes](https://kubernetes.io/)上部署[Nacos](https://nacos.io)
# Tips
* 推荐使用[Nacos Operator](https://github.com/nacos-group/nacos-k8s/blob/master/operator/README-CN.md)在Kubernetes部署Nacos Server.

# 快速开始

* **Clone 项目**

```shell
git clone https://github.com/nacos-group/nacos-k8s.git
```

* **简单例子**

> 如果你使用简单方式快速启动,请注意这是没有使用持久化卷的,可能存在数据丢失风险:

```shell
cd nacos-k8s
chmod +x quick-startup.sh
./quick-startup.sh
```
* **测试**

  * **服务注册**

  ```bash
  curl -X POST 'http://cluster-ip:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
  ```
  
  * **服务发现**

  ```bash
  curl -X GET 'http://cluster-ip:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
  ```
  
  * **发布配置**

  ```bash
  curl -X POST "http://cluster-ip:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"
  ```
  
  * **获取配置**

  ```bash
  curl -X GET "http://cluster-ip:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
  ```

# 高级使用

> 在高级使用中,Nacos在K8S拥有自动扩容缩容和数据持久特性,请注意如果需要使用这部分功能请使用PVC持久卷,Nacos的自动扩容缩容需要依赖持久卷,以及数据持久化也是一样,本例中使用的是NFS来使用PVC.
>

## 部署 NFS

* 创建角色 

```shell
kubectl create -f deploy/nfs/rbac.yaml
```

> 如果的K8S命名空间不是**default**，请在部署RBAC之前执行以下脚本:

```shell
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/nfs/rbac.yaml

```

* 创建 `ServiceAccount` 和部署 `NFS-Client Provisioner`

```shell
kubectl create -f deploy/nfs/deployment.yaml
```

* 创建 NFS StorageClass

```shell
kubectl create -f deploy/nfs/class.yaml
```

* 验证NFS部署成功

```shell
kubectl get pod -l app=nfs-client-provisioner
```

## 部署数据库




```shell

cd nacos-k8s

kubectl create -f deploy/mysql/mysql-nfs.yaml
```




* 验证数据库是否正常工作

```shell

kubectl get pod 
NAME                         READY   STATUS    RESTARTS   AGE
mysql-gf2vd                        1/1     Running   0          111m

```
## 执行数据库初始化语句

数据库初始化语句位置  <https://github.com/alibaba/nacos/blob/develop/distribution/conf/nacos-mysql.sql>


## 部署Nacos

* 修改  **deploy/nacos/nacos-pvc-nfs.yaml**

```yaml
data:
  mysql.host: "数据库地址"
  mysql.db.name: "数据库名称"
  mysql.port: "端口"
  mysql.user: "用户名"
  mysql.password: "密码"
```

* 创建 Nacos

``` shell
kubectl create -f nacos-k8s/deploy/nacos/nacos-pvc-nfs.yaml
```

* 验证Nacos节点启动成功

```shell
kubectl get pod -l app=nacos


NAME      READY   STATUS    RESTARTS   AGE
nacos-0   1/1     Running   0          19h
nacos-1   1/1     Running   0          19h
nacos-2   1/1     Running   0          19h
```

## 扩容测试

* 在扩容前，使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)获取在pod中的Nacos集群配置文件信息

```powershell
for i in 0 1; do echo nacos-$i; kubectl exec nacos-$i cat conf/cluster.conf; done
```

StatefulSet控制器根据其序数索引为每个Pod提供唯一的主机名。 主机名采用<statefulset name />  -  <ordinal index />的形式。 因为nacos StatefulSet的副本字段设置为2，所以当前集群文件中只有两个Nacos节点地址

![k8s](https://cdn.nlark.com/yuque/0/2019/gif/338441/1562846123635-e361d2b5-4bbe-4347-acad-8f11f75e6d38.gif)

* 使用kubectl scale 对Nacos动态扩容

```bash
kubectl scale sts nacos --replicas=3
```

![scale](https://cdn.nlark.com/yuque/0/2019/gif/338441/1562846139093-7a79b709-9afa-448a-b7d6-f57571d3a902.gif)

* 在扩容后，使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)获取在pod中的Nacos集群配置文件信息

```bash
for i in 0 1 2; do echo nacos-$i; kubectl exec nacos-$i cat conf/cluster.conf; done
```

![get_cluster_after](https://cdn.nlark.com/yuque/0/2019/gif/338441/1562846177553-c1c7f379-1b41-4026-9f0b-23e15dde02a8.gif)

* 使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)执行Nacos API 在每台节点上获取当前**Leader**是否一致

```bash
for i in 0 1 2; do echo nacos-$i; kubectl exec nacos-$i curl -X GET "http://localhost:8848/nacos/v1/ns/raft/state"; done
```

到这里你可以发现新节点已经正常加入Nacos集群当中

# 例子部署环境

- 机器配置

| 内网IP       | 主机名      | 配置                                                         |
| ----------- | ---------- | ------------------------------------------------------------ |
| 172.17.79.3 | k8s-master | CentOS Linux release 7.4.1708 (Core) Single-core processor Mem 4G Cloud disk 40G |
| 172.17.79.4 | node01     | CentOS Linux release 7.4.1708 (Core) Single-core processor Mem 4G Cloud disk 40G |
| 172.17.79.5 | node02     | CentOS Linux release 7.4.1708 (Core) Single-core processor Mem 4G Cloud disk 40G |

- Kubernetes 版本：**1.20.11** （如果你和我一样只使用了三台机器，那么记得开启master节点的部署功能）
- NFS 版本：**4.1** 在k8s-master进行安装Server端，并且指定共享目录，本项目指定的**/data/nfs-share**
- Git

# 限制

* 必须要使用持久卷，否则会出现数据丢失的情况

# 项目目录

| 目录 | 描述                                |
| ------ | ----------------------------------- |
| `plugin` | 帮助Nacos集群进行动态扩容的插件Docker镜像源码 |
| `deploy` | K8s 部署文件              |

# 配置属性

* nacos-pvc-nfs.yaml or nacos-quick-start.yaml 

| 名称                     | 必要 | 描述                                    |
| ----------------------- | -------- | --------------------------------------- |
| `mysql.host`            | Y       | 自建数据库地址,使用外部数据库时必须指定                      |
| `mysql.db.name`         | Y       | 数据库名称                      |
| `mysql.port`            | N       | 数据库端口                        |
| `mysql.user`            | Y       | 数据库用户名(请不要含有符号```,```)     |
| `mysql.password`        | Y       | 数据库密码(请不要含有符号```,```)                     |
| `SPRING_DATASOURCE_PLATFORM`        | Y       |   数据库类型,默认为embedded嵌入式数据库,参数只支持mysql或embedded	                   |
| `NACOS_REPLICAS`        | N      | 确定执行Nacos启动节点数量,如果不适用动态扩容插件,就必须配置这个属性，否则使用扩容插件后不会生效 |
| `NACOS_SERVER_PORT`     | N       | Nacos 端口 为peer_finder插件提供端口|
| `NACOS_APPLICATION_PORT`     | N       | Nacos 端口|
| `PREFER_HOST_MODE`      | Y       | 启动Nacos集群按域名解析 |

* **nfs** deployment.yaml 

| 名称          | 必要 | 描述                     |
| ------------ | --------| ------------------------ |
| `NFS_SERVER` | Y       | NFS 服务端地址         |
| `NFS_PATH`   | Y       | NFS 共享目录 |
| `server`     | Y       | NFS 服务端地址  |
| `path`       | Y       | NFS 共享目录 |

* mysql 

| 名称                     | 必要 | 描述                                                      |
| -------------------------- | -------- | ----------------------------------------------------------- |
| `MYSQL_ROOT_PASSWORD`        | N       | ROOT 密码                                                    |
| `MYSQL_DATABASE`             | Y       | 数据库名称                                   |
| `MYSQL_USER`                 | Y       | 数据库用户名                                  |
| `MYSQL_PASSWORD`             | Y       | 数据库密码                              |
| `MYSQL_REPLICATION_USER`     | Y       | 数据库复制用户            |
| `MYSQL_REPLICATION_PASSWORD` | Y       | 数据库复制用户密码      |
| `Nfs:server`                 | N      | NFS 服务端地址，如果使用本地部署不需要配置 |
| `Nfs:path`                   | N     | NFS 共享目录，如果使用本地部署不需要配置 |
