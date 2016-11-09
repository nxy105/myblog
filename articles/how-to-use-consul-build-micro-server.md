# Consul 搭建服务框架（基础篇）

近期，公司产品架构面临升级，伴随的产品架构的调整，技术架构应该做为强大的支撑存在。然而，强大的技术架构，面临的最大的问题是如何控制系统的复杂度。为了解决这个问题，我们准备使用 SOA 思想将系统服务化。

服务发现和治理，一直是服务化后所面临的一项课题。目前，致力于解决服务管理的工具有 ZooKeeper，etcd 以及即将介绍的 Consul。这3者各具优势，Consul 作为其中的新星，引起了我尝试和研究的兴趣。

## Consul 简介

Consul 是一个支持多数据中心，分布式，高可用的服务发现和配置共享系统。由 HashiCorp 公司使用 Go 语言开发。

### Consul 的特性

- 简单的服务注册，支持通过 DNS 或者 HTTP 的方式发现服务。
- 支持服务的健康检查。
- 多数据中心，保证高可用。
- 提供 Key/Value 存储，方便配置信息的共享。
- 官方提供 Web 管理界面。

## Consul 架构

### Consul 系统中出现的角色

- Agent - Agent 是在 Consul 集群中每个机器成员中运行的一个常驻进程。Agent 可以被作为 Client 或者 Server 的身份运行。所有 Agent 都可以运行 DNS 或者 HTTP 接口，并且执行监控检查和保持服务信息的同步。
- Client - Client 是一个 Agent，它的职责是将 RPC 调用转向对应的服务。Client 是无状态的。Client 唯一在后台运行的的逻辑是加入 Gossip 同步池。
- Server - Server 同样也是一个 Agent，它的职责包括实现集群的数据一致性，维护集群状态，响应 RPC 请求，与其他数据中心交换数据等。

### Consul 集群

一个基础的 Consul 集群可能是下面这样的：

![image](http://joshuais.me/content/images/consul-datacenter.png)

## Consul 要点

### Consul 如何实现一致性

由于 Consul 是分布式多中心系统，各个数据中心，各个节点之间，需要通过一致性算法保持数据一致。Consul 使用 Raft 算法实现一致性的协议。

这里暂时不展开分析 Raft 算法。提供 ["Raft: In search of an Understandable Consensus Algorithm"](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf) 作为参考。

同时，节点与节点之间，Consul 使用 gossip 协议管理集群和广播消息。实现部分是基于自家的 [Serf](https://www.serfdom.io/) 库。

### Consul 如何发现服务

通过定义服务的方式或者调用 HTTP API 的方式，我们可以将服务注册到 Consul 中。

使用定义服务的方式是将服务信息写入指定的配置文件中，然后重新启动 Agent 将服务信息同步到 Consul 集群中。

另外，通过 HTTP API 调用，我们也可以动态的为集群添加服务信息。

### Consul 如何查询服务

Consul 提供了 DNS 和 HTTP 两种查询服务的方式。每个 Consul 节点默认会运行内置的 DNS 和 HTTP 服务。

DNS 服务的默认端口设置为8600。Consul 服务被放置在 .consul 的命名空间下，DNS 查询的基础规则为 `SERVICENAME.service.consul`。其中 service 子域表示查询服务，`SERVICENAME`正是查询服务的名称。除了通过 DNS API，HTTP 服务也可用于服务查询。

通过 DNS 和 HTTP API 所查询到的服务信息，可以用于服务之间的调用访问。

## 结语

通过一致性实现节点之间的服务信息共享，通过节点本地的访问获取服务信息，Consul 让服务管理变得简单而高效。通过本篇介绍，我们对 Consul 有一个整体的了解。关于如何搭建 Consul 集群以及一些 Consul 原理基础，我们将会在后续的研究中继续不断探索。



