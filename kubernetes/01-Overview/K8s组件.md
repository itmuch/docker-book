# K8s组件

本文概述了Kubernetes集群中所需的各种组件。



## Master组件

Master组件提供K8s集群的控制面板。Master对集群进行全局决策（例如调度），以及检测和响应集群事件（例如：当replication controller所设置的`replicas` 不够时，启动一个新的Pod）。

Master可在集群中的任意节点上运行。然而，简单起见，设置脚本通常在同一个VM上启动所有Master组件，并且不会在该VM上运行用户的容器。请阅读 [Building High-Availability Clusters](https://kubernetes.io/docs/admin/high-availability/) 以实现多主机VM配置。



### kube-apiserver

[kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver/) 暴露Kubernetes的API。它是Kubernetes控制能力的前端。它被设计为可水平扩展——也就是通过部署更多实例来实现扩容。详见 [Building High-Availability Clusters](https://kubernetes.io/docs/admin/high-availability/) 。



### etcd

[etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) 用作Kubernetes的后端存储。集群的所有数据都存储在此。请为你Kubernetes集群的etcd数据提供备份计划。



### kube-controller-manager

[kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/) 运行Controller，它们是处理集群中常规任务的后台线程。逻辑上来讲，每个Controller都是一个单独的进程，但为了降低复杂性，它们都被编译成独立的二进制文件并运行在一个进程中。

这些控制器包括：

- Node Controller：当节点挂掉时，负责响应。
- Replication Controller：负责维护系统中每个replication controller对象具有正确数量的Pod。
- Endpoints Controller：填充Endpoint对象（即：连接Service＆Pod）。
- Service Account & Token Controllers：为新的namespace创建默认帐户和API access tokens。




### cloud-controller-manager

cloud-controller-manager运行着与底层云提供商交互的Controller。cloud-controller-manager是在Kubernetes 1.6版中引入的，处于Alpha阶段。

cloud-controller-manager仅运行云提供商特定的Controller循环。您必须在kube-controller-manager中禁用这些Controller循环。可在启动kube-controller-manager时将`--cloud-provider` 标志设为 `external` 来禁用控制器循环。

cloud-controller-manager允许云供应商代码和Kubernetes内核独立发展。在以前的版本中，核心的Kubernetes代码依赖于特定云提供商的功能代码。在未来的版本中，云供应商的特定代码应由云供应商自己维护，并在运行Kubernetes时与cloud-controller-manager相关联。

以下控制器存在云提供商依赖：

- Node Controller：用于检查云提供商，从而确定Node在停止响应后从云中删除
- Route Controller：用于在底层云基础设施中设置路由
- Service Controller：用于创建、更新以及删除云提供商负载均衡器
- Volume Controller：用于创建、连接和装载Volume，并与云提供商进行交互，从而协调Volume




### kube-scheduler

[kube-scheduler](https://kubernetes.io/docs/admin/kube-scheduler/) 监视新创建的、还没分配Node的Pod，并选择一个Node供这些Pod运行。



### addons（插件）

Addon是实现集群功能的Pod和Service。Pod可由Deployment、ReplicationController等进行管理。Namespace的插件对象则是在`kube-system` 这个namespace中被创建的。

Addon manager创建并维护addon的资源。详见这里： [here](http://releases.k8s.io/HEAD/cluster/addons) 。



#### DNS

虽然其他Addon不是严格要求的，但所有Kubernetes集群都应该有 [cluster DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) ，许多用例都依赖于它。

Cluster DNS是一个DNS服务器，它为Kubernetes服务提供DNS记录。

Kubernetes启动的容器会自动将该DNS服务器包含在DNS搜索中。



#### Web UI (Dashboard)

[Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) 是一个Kubernetes集群通用、基于Web的UI。它允许用户管理/排错集群中应用程序以及集群本身。



#### Container Resource Monitoring（容器资源监控）

[Container Resource Monitoring](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/) 将容器的通用时序指标记录到一个中心化的数据库中，并提供一个UI以便于浏览该数据。



#### Cluster-level Logging（集群级别的日志）

[Cluster-level logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 机制负责将容器的日志存储到具有搜索/浏览界面的中央日志存储中去。





## Node组件

Node组件在每个Node上运行，维护运行的Pod并提供Kubernetes运行时环境。



### kubelet


[kubelet](https://kubernetes.io/docs/admin/kubelet/) 是主要的Node代理。它监视已分配到其Node上的Pod（通过apiserver或本地配置文件）和：

- 装载Pod所需的Volume。
- 下载Pod的secret。
- 通过Docker（或实验时使用rkt）运行Pod的容器。
- 定期执行任何被请求容器的活动探针（liveness probes）。
- 在必要时创建*mirror pod* ，从而将pod的状态报告回系统的其余部分。
- 将节点的状态报告回系统的其余部分。




### kube-proxy

[kube-proxy](https://kubernetes.io/docs/admin/kube-proxy/) 在主机上维护网络规则并执行连接转发，从而来实现Kubernetes服务抽象。



### docker

`docker` 用于运行容器。



### rkt

`rkt` 是一个Docker的替代品，支持在实验中运行容器



### supervisord

`supervisord` 是一个轻量级的进程监控/控制系统，可用于保持kubelet和docker运行。



### fluentd

`fluentd` 是一个守护进程，利用它可实现 [cluster-level logging](https://kubernetes.io/docs/concepts/overview/components/#cluster-level-logging) 。

