---
title: Consul与kubernetes整合公告[翻译]
keywords: [Consul,kubernetes]
description: Consul与kubernetes整合公告[翻译]
---

# Consul与kubernetes整合公告[翻译]
## 导读
> Consul是目前业界比较火的服务发现与配置产品，它率先将服务发现和配置管理等分布式服务当中使用到的基础服务进行整合，对外提供分布式及高可用的服务。Consul目前有开源版本和商业化版本同时演进，这也是国内可以借鉴的一种开源策略。同时，Consul对于新技术趋势的跟进和整合，也是值得我们学习和参考的。
> 
> 本文翻译了Consul对于Kubernetes的整合所发布的公告文章（[原文地址](https://www.hashicorp.com/blog/consul-plus-kubernetes)）。Consul通过支持Service Mesh，并提供对Kubernetes的无缝支持，与目前最受社区热捧的产品进行绑定，并通过功能预告的形式，来达到对产品宣传效果的最大化。
> 
> 与Consul产品对应的，阿里巴巴在近期开源了其服务发现与配置管理产品[Nacos](https://nacos.io)。Nacos是阿里巴巴集团内部VIPServer、ConfigServer和Diamond三个支撑双十一的重要中间件产品整合而来。Nacos主要关注产品的极致易用以及与云原生的深度整合，主要支持服务发现、配置管理等功能。很快，Nacos也会与Service Mesh进行整合，同时在集团内部和开源进行发布，利用阿里巴巴丰富的场景和开源社区的力量，将Nacos打造成云原生生态中不可或缺的基础产品。

我们很高兴地宣布HashiCorp Consul与Kubernetes深度整合的多项功能。 这篇文章将分享将在未来几周内发布的一系列初步功能。

这些功能包括用于在Kubernetes上安装Consul的官方Helm Chart，与Consul自动同步Kubernetes服务（反之亦然），外部Consul Agent自动加入Kubernetes中的集群，用于Connect自动保护pod的注入器，以及对Envoy的支持。

除了与Kubernetes本地集成之外，这些功能还有助于解决多个Kubernetes集群之间以及Kubernetes服务与非Kubernetes服务交互多项重要跨集群挑战。 我们很高兴与您分享这项工作。

## 功能点

以下是将在未来几周内公布和发布的功能列表。后续公告博客将详细介绍每个项目，每个项目都将链接到该公告帖子。
* __Helm Chart__。用于在Kubernetes上安装，配置和升级Consul的官方Helm Chart。此Helm Chart还将支持Kubernetes的其他功能的自动安装和配置，例如目录同步。
* __Kubernetes自动加入__。Consul的云自动加入功能将更新，以支持发现和加入基于Kubernetes的Agent代理。这将使外部Consul Agent加入在Kubernetes中运行的Consul集群。
* __服务目录同步：从K8S到Consul__。适当的Kubernetes服务将自动同步到Consul目录，使非Kubernetes服务能够发现并连接到Kubernetes中运行的服务。
* __服务目录同步：从Consul到K8S__。Consul服务将同步到Kubernetes服务，以便应用程序可以使用Kubernetes本地服务发现来发现和连接在Kubernetes之外运行的服务。
* __Connect自动注入__。部署在Kubernetes中的Pod可以配置为自动使用Connect通过TLS进行安全通信。
* 支持__Envoy Proxy__。配置为使用Connect自动注入的Pod可以使用Envoy Proxy进行第4层通信，通过Connect进行保护。 Envoy也将用于非Kubernetes的Connect部署。

## 与Kubernetes整合

我们目前正在与Kubernetes在多个产品上进行密切整合。通过使我们的产品更易于运行以及集成和增强Kubernetes功能，我们看到了解决纯Kubernetes用户挑战的机会。

这种集成的核心原则是增强现有功能而不是替换。服务、ConfigMaps及Secrets等功能是Kubernetes核心工作流程的一部分。更高级别的工具和扩展利用这些核心原语。因此，我们也在整合和增强这些核心原语。例如，Consul目录同步将Consul目录中的外部服务转换为一级的Kubernetes服务资源。然后，在Kubernetes中运行的应用程序可以发现并连接到非Kubernetes服务。

除了使我们的产品在Kubernetes中更易用和更自然之外，这些集成还允许用户在与非Kubernetes工作负载共享的环境中更好地工作。 虽然新用户很容易在纯Kubernetes环境中启动，但大多数部署必须与在云计算环境中运行的外部服务，本地数据中心等进行交互。 像Consul这样的HashiCorp产品专为这些异构环境而设计。 通过实现更自然的Kubernetes体验，非Kubernetes应用程序与Kubernetes应用程序交互变得同样自然。

# 下一步

我们很高兴地宣布第一批HashiCorp Consul和Kubernetes的功能。 这些功能使得在Kubernetes上运行Consul，与非Kubernetes服务进行交互，在Kubernetes内外进行安全通信等等更加容易。 这些功能中的每一项都将在未来几周内全面公布和发布，从下周的Helm Chart开始。

Terraform和Vault也与Kubernetes紧密集成。 Terraform Kubernetes提供商现在拥有一名专职工程师，并且在未来几个月内应该会迅速改进。 Vault正在开发新的集成，也将很快公布。

如果您对Kubernetes，我们的工具以及改进这些集成感兴趣，请加入我们！ 我们有一些针对Kubernetes集成的生态系统工程师岗位需求。
