---
title: Nacos服务发现控制台预览
keywords: [nacos]
description: Nacos服务发现控制台预览
---

# Nacos服务发现控制台预览

Nacos是阿里巴巴中间件部门最近开源的一款用于服务发现和配置管理的产品。在既0.1版本发布基本功能和0.2版本发布与Spring生态结合的功能后，0.3版本将释放全新的控制台界面。配置管理功能相关的控制台，将会由阿里云商业产品ACM控制台改造而来，而服务发现的控制台界面，则将以首次露面的姿态，开放给开源社区。本文就将服务发现控制台相关的界面UI初版设计公布，欢迎大家参与讨论，希望通过大家的批评和建议，将服务发现控制台这块的功能和界面，设计的更加美观和易用。

服务发现控制台的主要功能是服务列表的展示和搜索，以及服务配置、集群配置、实例配置的查询和更新。在0.3版本中，主要会有两个页面：服务列表和服务详情。

# 服务列表
服务列表页面主要展示已经在Nacos注册的服务列表，以及服务的基本信息，服务的基本信息有：服务的名称、服务下集群的数目、服务下实例的数目、服务的健康程度以及进入服务详情的按钮。同时右上角还有一个支持根据服务名搜索服务的搜索框和搜索按钮。

![图1 服务列表页面](https://cdn.nlark.com/lark/0/2018/png/15356/1538701093629-9880a456-8a37-4663-bd88-853441dab3f4.png)
<!-- <div data-type="alignment" data-value="center" style="text-align:center">
  <div data-type="p">
    <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701093629-9880a456-8a37-4663-bd88-853441dab3f4.png" data-width="572">
      <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701093629-9880a456-8a37-4663-bd88-853441dab3f4.png" width="572" />
    </div>
  </div>
  <div data-type="p"></div>
</div> -->


# 服务详情
在服务列表页面点击“detail”按钮，就会进入服务详情页面。服务详情页面展示的是一个服务的所有关键信息，包括服务的配置和元数据、集群列表和示例列表，以及一些操作的按钮。

![图2 服务详情页面](https://cdn.nlark.com/lark/0/2018/png/15356/1538701093629-9880a456-8a37-4663-bd88-853441dab3f4.png)

<!-- <div data-type="alignment" data-value="center" style="text-align:center">
  <div data-type="p">
    <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701116839-4753cafa-f5f6-4866-a19d-f1293489053d.png" data-width="607">
      <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701116839-4753cafa-f5f6-4866-a19d-f1293489053d.png" width="607" />
    </div>
  </div>
  <div data-type="p">图2 服务详情页面</div>
</div> -->


在该页面的上方，是服务的配置和元信息，目前包含服务名、保护阈值、健康检查模式以及元数据metadata。右上方是编辑服务按钮，点击后会有对话框弹出，可以对服务的配置进行编辑。

![图2 服务详情页面](https://cdn.nlark.com/lark/0/2018/png/15356/1538701150783-fa6d58cf-01f4-430c-a5d0-6278c9590404.png)
<!-- <div data-type="alignment" data-value="center" style="text-align:center">
  <div data-type="p">
    <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701150783-fa6d58cf-01f4-430c-a5d0-6278c9590404.png" data-width="362">
      <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701150783-fa6d58cf-01f4-430c-a5d0-6278c9590404.png" width="362" />
    </div>
  </div>
  <div data-type="p">图3 更新服务对话框</div>
</div> -->


服务详情的下方，是集群列表和集群下的实例列表。每个集群会显示一个集群名，和相应的查看&更新集群详情按钮。点击该按钮后，会是一个更新集群的对话框。

![图4 更新集群（TCP健康检查）](https://cdn.nlark.com/lark/0/2018/png/15356/1538701200952-f9dcb51e-100e-4501-a3db-d665dfaf7188.png)
<!-- <div data-type="alignment" data-value="center" style="text-align:center">
  <div data-type="p">
    <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701200952-f9dcb51e-100e-4501-a3db-d665dfaf7188.png" data-width="362">
      <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701200952-f9dcb51e-100e-4501-a3db-d665dfaf7188.png" width="362" />
    </div>
  </div>
  <div data-type="p"></div>
</div> -->

![图5 更新集群（HTTP健康检查）](https://cdn.nlark.com/lark/0/2018/png/15356/1538701223427-284aaf1c-1cd3-412e-9f22-d5baae2cee25.png)

<!-- <div data-type="alignment" data-value="center" style="text-align:center">
  <div data-type="p">
    <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701223427-284aaf1c-1cd3-412e-9f22-d5baae2cee25.png" data-width="362">
      <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701223427-284aaf1c-1cd3-412e-9f22-d5baae2cee25.png" width="362" />
    </div>
  </div>
  <div data-type="p">图5 更新集群（HTTP健康检查）</div>
</div> -->


图4和图5分别展示了对集群更新的两种对话框展示，两者的区别是选择了不同的健康检查方式。TCP健康检查方式可以配置检查的端口；HTTP健康检查方式可以配置检查的端口、检查的路径和HTTP头部信息。同时还可以配置是否采用实例的端口进行健康检查，如果配置为true，则健康检查将使用实例注册的端口进行通信。该对话框还可以编辑集群的元信息。

每个集群下面都会有实例列表，实例列表将会分页展示该集群下注册的所有实例，展示的信息有IP、端口、权重、是否健康、元信息和对应的编辑及下线按钮。下线按钮点击后，该实例将不会返回给订阅端，无论该实例是否健康。“下线”文本会改成“上线”，用于对应的实例上线操作。点击编辑按钮，则会进入编辑实例对话框。

![图6 编辑实例对话框](https://cdn.nlark.com/lark/0/2018/png/15356/1538701250740-ffb38cd0-a15d-4425-a2ca-48c5d0d2698e.png)
<!-- <div>
  <div data-type="alignment" data-value="center" style="text-align:center">
    <div data-type="p">
      <div id="soktqz" data-type="image" data-display="block" data-align="center" data-src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701250740-ffb38cd0-a15d-4425-a2ca-48c5d0d2698e.png" data-width="346">
        <img src="https://cdn.nlark.com/lark/0/2018/png/15356/1538701250740-ffb38cd0-a15d-4425-a2ca-48c5d0d2698e.png" width="346" />
      </div>
    </div>
    <div data-type="p"></div>
  </div>
</div> -->


编辑实例对话框，可以编辑的信息有实例的权重、是否上下线和元信息。

0.3版本的服务发现页面，基本就是这样，欢迎大家的反馈。服务注册客户端也可以编辑服务、集群、实例元信息，这些可能会和控制台的编辑相冲突，目前的机制是，不管是控制台更新和客户端更新，都将被Nacos服务端所接受，这点也欢迎大家给出自己的看法。最后也预祝大家国庆放假愉快！



