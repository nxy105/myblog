# Consul 搭建服务框架（使用篇）

在之前的 [Consul 搭建服务框架（基础篇）](http://joshuais.me/consul-da-jian-fu-wu-kuang-jia-ji-chu-pian/) 我们讨论了 Consul 的基础概念和特性。在这篇文章中，我们通过实践深入了解如何使用 Consul 搭建服务架构。

## 在搭建之前

在搭建之前，需要先明确我们要搭建的目标，如下图所示：

![image]()

节点介绍

- Consul Consul Server 节点，简单的集群化，设置2个节点，其中一个节点设置为 Bootstrap 模式。
- Service 提供业务服务的机器节点，部署 Consul Client。
- Status 提供 Consul 集群状态查询的 Web 服务，部署 Consul Client。
- Demo 提供业务入口的机器节点，部署 Consul Client。

## 搭建工具

为了方便读者在本地模拟集群的环境，我们将使用 [Vagrant](https://www.vagrantup.com/) 作为虚拟环境搭建的工具。我将 Vagrant 配置文件放在 [https://github.com/nxy105/consul-cluster-demo](https://github.com/nxy105/consul-cluster-demo) 项目中提供下载。

```
$ git clone https://github.com/nxy105/consul-cluster-demo.git
```

下一步我们会在这个虚拟的环境中探索如何使用 Consul 治理我们的应用。


## 搭建集群

### 集群中的节点

#### 服务端节点

首先，我们先启动 Consul 服务端的两台集群机器节点。

```
# 启动服务端的 boostrap 节点
$ vagrant up consul1
# 登入节点机器
$ vagrant ssh consul1

Last login: Thu Jul  7 12:37:34 2016 from 10.0.2.2
# 启动 Consul 服务端，daemon 是我们预先安装的进程化工具。
vagrant@consul1:~$ daemon -X "consul agent -server -bootstrap -data-dir /tmp/consul -node=consul1 -bind 172.0.0.10"
```

在启动命令中，我们定义了 agent 类型为 server，指定了节点名称为 consul1，绑定到我们虚拟机指定的内网地址上。`-bootstrap`定义了这台机器节点启动为 boostrap 模式，后续的节点将加入到该节点以该节点为中心的集群中。

```
# 启动另一个服务端节点 
$ vagrant up consul2
$ vagrant ssh consul2

Last login: Thu Jul  7 12:37:34 2016 from 10.0.2.2
vagrant@consul2:~$ daemon -X "consul agent -server -data-dir /tmp/consul -node=consul2 -bind 172.0.0.11 -join 172.0.0.10"

```

在这个启动命令中，我们通过`-join`参数将这台机器节点加入到第一个服务端节点所在的网络中。

这时，我们看一下当前集群中所有的节点。

```
# 在任意一个节点中执行都可以看到整个集群的节点信息
vagrant@consul2:~$ consul members
Node     Address          Status  Type    Build  Protocol  DC
consul1  172.0.0.10:8301  alive   server  0.6.4  2         dc1
consul2  172.0.0.11:8301  alive   server  0.6.4  2         dc1
```

通过这个命令，我们可以看到当前集群中都包含哪些节点，节点当前的身份状态，以及所在的地址。

#### 客户端节点

接下来，我们将要创建提供业务服务的节点。

```
# 启动业务服务节点，在启动的同时，我们将同时启动我们的一个业务服务程序 demo。
# 该业务程序监听8045端口，这些在我们的 Vagrantfile 中已经配置好了。 
$ vagrant up service1
$ vagrant ssh service1

Last login: Thu Jul  7 12:43:11 2016 from 10.0.2.2
vagrant@service1:~$ daemon -X "consul agent -data-dir /tmp/consul -node=service1 -config-dir /etc/consul.d -bind 172.0.0.13 -join 172.0.0.10"
```

启动 Consul 服务，这时，我们将该节点的身份设置为默认的 client，依然加入第一个服务节点所在的集群。在节点启动脚本中，我们已经将业务服务的定义信息写入`/etc/consul.d/mysvc.json`文件，这样在 Consul 客户端服务启动之后，就能将服务的信息同步到集群中的所有节点。我们来看看`mysvc.json`是如何定义业务服务的。

```
vagrant@service1:~$ cat /etc/consul.d/mysvc.json
{"service": {
    "name": "my-svc",
    "tags": ["microservice"],
    "port": 8045,
    "check": {
        "script": "curl localhost:8045/status | grep OK || exit 2",
        "interval": "5s"
    }
}}
```

这个配置文件定义了业务服务的名称，所属 tag，监听的节点，以及健康检查的脚本指令和频率。这些信息如何获取使用，我们将在下一节详细说明。

如法炮制，我们启动 service2，demo 这两个客户端节点。启动 Consul 服务的命令我们已经预置到 Vagrant 的启动命令中，不需要通过 SSH 登录后执行。

```
# service2 提供与 service1 相同的业务服务，服务监听8076端口
# demo 则监听默认的80端口提供 Web 服务
$ vagrant up service2
$ vagrant up demo
```

这时，我们再来查看集群的节点。接下来，我们会使用可视化的工具查看和管理集群中的节点。

```
Last login: Thu Jul  7 12:17:48 2016 from 10.0.2.2
vagrant@demo:~$ consul members
Node      Address          Status  Type    Build  Protocol  DC
consul1   172.0.0.10:8301  alive   server  0.6.4  2         dc1
consul2   172.0.0.11:8301  alive   server  0.6.4  2         dc1
demo      172.0.0.15:8301  alive   client  0.6.4  2         dc1
service1  172.0.0.13:8301  alive   client  0.6.4  2         dc1
service2  172.0.0.14:8301  alive   client  0.6.4  2         dc1
```

#### 状态监控节点

最后，我们将创建一个机器节点，用于查看集群状态。

```
$ vagrant up status
$ vagrant ssh status

Last login: Thu Jul  7 12:34:52 2016 from 10.0.2.2
vagrant@status:~$ daemon -X "consul agent -client=172.0.0.12 -ui-dir /vagrant/web/ -data-dir /tmp/consul -node=status -bind 172.0.0.12 -join 172.0.0.10"
```

Consul 提供了用于状态监控 Web 服务器，将 [Consul Download](https://www.consul.io/downloads.html) 提供的 Web UI 工具下载到`-ui-dir`指定的目录，即可以创建可视化的管理页面 [http://172.0.0.12:8500/ui](http://172.0.0.12:8500/ui)。

![image]()

## 服务管理

### DNS

每个 Consul 节点都提供了 DNS 服务用于查询业务服务所在的位置。我们进入 demo 机器通过 DNS 查找业务提供的服务。

```
vagrant@demo:~$ dig @127.0.0.1 -p 8600 my-svc.service.consul SRV

; <<>> DiG 9.9.5-3ubuntu0.8-Ubuntu <<>> @127.0.0.1 -p 8600 my-svc.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21274
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;my-svc.service.consul.		IN	SRV

;; ANSWER SECTION:
my-svc.service.consul.	0	IN	SRV	1 1 8076 service2.node.dc1.consul.
my-svc.service.consul.	0	IN	SRV	1 1 8045 service1.node.dc1.consul.

;; ADDITIONAL SECTION:
service2.node.dc1.consul. 0	IN	A	172.0.0.14
service1.node.dc1.consul. 0	IN	A	172.0.0.13
```

我们可以看到 DNS 返回了2条 SRV 记录，对应同一个服务，指向不同的 IP 与端口。这时，我们在 demo 的程序中，会随机选择一个访问。[http://172.0.0.15/demo](http://172.0.0.15/demo) 页面返回了本次请求选择的服务。

### REST API

通过 HTTP 的方式同样可以获取业务服务的信息。

```
vagrant@demo:~$ curl http://localhost:8500/v1/catalog/service/my-svc
[{"Node":"service1","Address":"172.0.0.13","ServiceID":"my-svc","ServiceName":"my-svc","ServiceTags":["microservice"],"ServiceAddress":"","ServicePort":8045,"ServiceEnableTagOverride":false,"CreateIndex":51,"ModifyIndex":261},{"Node":"service2","Address":"172.0.0.14","ServiceID":"my-svc","ServiceName":"my-svc","ServiceTags":["microservice"],"ServiceAddress":"","ServicePort":8076,"ServiceEnableTagOverride":false,"CreateIndex":19,"ModifyIndex":291}]
```

### 健康检查

服务不会永远稳定可用，这也是为什么我们选择多个节点做灾备的原因。接下来，我们将 service1 提供的服务设置为不可用，看看 Consul 是如何做容灾的。登录 service1 创建`/tmp/fail-healthcheck`文件。一旦文件创建成功，[http://172.0.0.13:8045/status](http://172.0.0.13:8045/status) 接口返回的状态将会标识为`FAIL`。我们之前定义的健康检查脚本一旦没有查询到`OK`状态则返回一个异常情况。

```
$ vagrant ssh service1

Last login: Sat Jul  9 09:12:15 2016 from 10.0.2.2
vagrant@service1:~$ touch /tmp/fail-healthcheck
```

[http://172.0.0.13:8045/status](http://172.0.0.13:8045/status) 返回值变成了`FAIL`。这时，我们再来看看 DNS 返回的服务列表。

```
vagrant@demo:~$ dig @127.0.0.1 -p 8600 my-svc.service.consul SRV

; <<>> DiG 9.9.5-3ubuntu0.8-Ubuntu <<>> @127.0.0.1 -p 8600 my-svc.service.consul SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6122
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;my-svc.service.consul.		IN	SRV

;; ANSWER SECTION:
my-svc.service.consul.	0	IN	SRV	1 1 8076 service2.node.dc1.consul.

;; ADDITIONAL SECTION:
service2.node.dc1.consul. 0	IN	A	172.0.0.14
```

DNS 返回的 SRV 记录中已经没有 service1 所在的节点了。[http://172.0.0.15/demo](http://172.0.0.15/demo) 也不再请求 service1 了。而我们的 demo Web 服务却没有因此而停止。

## 结语

到此，我们已经使用 Consul 搭建了一个基础的服务器集群环境，并且在这个环境中使用 Consul 搭建了简单的 Web 服务。接下来，我们还会对 Consul 内部使用的技术点更进一步的分析。