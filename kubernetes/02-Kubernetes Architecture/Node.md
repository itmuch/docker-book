# Node（节点）





## 什么是Node？

`node` 是Kubernetes中的工作机器（worker），以前被称为`minion` 。 集群中的Node可以是VM或物理机。 每个Node上都有运行 [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 所必要的服务，并由Master组件管理。Node上的服务包括Docker、kubelet和kube-proxy。有关更多详细信息，请参阅架构设计文档中的  [The Kubernetes Node](https://git.k8s.io/community/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node)  部分。





## Node状态

Node状态包含如下信息：

- Addresses
- Phase 【**弃用**】
- Condition
- Capacity
- Info

下面详解每个部分：





### Addresses（地址）

这些字段的用法取决于你云提供商或裸机的配置。

- HostName：Node内核报告的hostname。可通过kubelet的 `--hostname-override` 参数配置。
- ExternalIP：通常是可外部路由的Node IP（可从群集外部获得）。
- InternalIP：通常只能在集群内进行路由的Node IP。




### Phase（阶段）

已弃用，Node phase不再使用。



### Condition（状况）

`conditions` 字段描述所有`Running` Node的状态。

| Node Condition       | Description                              |
| -------------------- | ---------------------------------------- |
| `OutOfDisk`          | 如果节点没有足够的可用空间来添加新的Pod，则为`True` ，否则为`False` |
| `Ready`              | 如果节点健康并准备好接受Pod，则为`True` ；如果节点不健康且不接受Pod，则为`False`，如果node controller与Node失联40秒以上，则为“ `Unknown` |
| `MemoryPressure`     | 如果node的内存存在压力，则为`True` ——即node内存容量低；否则为`False` |
| `DiskPressure`       | 如果磁盘存在压力，则为`True` ——即磁盘容量低；否则为`False`    |
| `NetworkUnavailable` | 如果Node的网络未正确配置，则为`True` ，否则为`False`      |

node condition使用JSON对象来表示。 例如，以下描述了一个健康的Node。

```json
"conditions": [
  {
    "kind": "Ready",
    "status": "True"
  }
]
```

如果`Ready` condition的状态为“Unknown”或“False” ，并且持续超过`pod-eviction-timeout` ，则会将一个参数传递给 [kube-controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/) ，并且Node上的所有Pod都会被Node Controller驱逐。默认驱逐的超时时间为**五分钟** 。 在某些情况下，当Node不可访问时，apiserver无法与其上的kubelet进行通信。 在与apiserver恢复通信之前，删除Pod的指令无法传达到kubelet。 同时，计划删除的Pod可能会继续在该Node上运行。

在Kubernetes 1.5之前，Node Controller将强制从apiserver中  [force delete](https://kubernetes.io/docs/concepts/workloads/pods/pod/#force-deletion-of-pods)  这些不可达的pod。但在1.5及更高版本中，Node Controller不会强制删除Pod，直到确认它们已停止运行。 

这些不可达Node上运行的Pod会处于“Terminating”或“Unknown”状态。如果Kubernetes无法从底层基础设施推断出Node已永久离开集群，集群管理员可能需要手动删除Node对象。从Kubernetes删除Node对象会导致运行在其上的所有Pod对象从apiserver中删除，并释放其名称。

Kubernetes 1.8引入了一个自动创建代表condition的 [taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) 功能（目前处于Alpha状态）。要启用此特性，请向API server、controller manager和scheduler传递标志`--feature-gates=...,TaintNodesByCondition=true` 。一旦启用	`TaintNodesByCondition` ，scheduler将会忽略选择Node时的condition；而是看Node的taint（污点）和Pod的toleration（容忍度）。

> 译者按：
>
> 1. “[taints and tolerations](https://kubernetes.io/docs/user-guide/node-selection/#taints-and-toleations-beta-feature)” 的功能是允许你标注（taint）Node，那样Pod就不会调度到这个Node上，除非Pod明确”tolerates”这个”taint”。
> 2. K8s高级调度特性：<http://blog.csdn.net/jettery/article/details/69500150>

现在用户可在旧调度模型和新的更灵活的调度模型之间选择。没有toleration（容忍度）的Pod根据旧的模型进行调度。但是，对特定Node能够容忍污点（tolerates the taints）的Pod可被调度到该Node。

请注意，由于延迟时间小，通常少于1秒，在观察condition和产生污点的时间段内，启用此功能可能会稍微增加成功调度但被kubelet拒绝的Pod的数量。



### Capacity（容量）

描述Node上可用的资源：CPU、内存，以及可调度到该Node的最大Pod数。



### Info（信息）

关于Node的一般信息，如内核版本、Kubernetes版本（kubelet和kube-proxy版本）、Docker版本（如果使用了Docker的话）、OS名称。信息由Kubelet从Node收集。





## Management（管理）

与 [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 、 [services](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 不同，Node不是由Kubernetes创建的：它是由Google Compute Engine等云提供商在外部创建的，或存在于物理机或虚拟机池中。这意味着当Kubernetes创建一个Node时，它只是创建一个表示Node的对象。创建后，Kubernetes将检查Node是否有效。例如，如果您尝试从以下内容创建一个Node：

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

Kubernetes将在内部创建一个Node对象（用来表示Node），并通过基于`metadata.name` 字段的健康检查来验证Node（我们假设`metadata.name` 可以被解析）。如果Node通过验证，即：所有必需的服务都处于运行状态，则它有资格运行Pod；否则，它将被忽略，直到通过验证。 请注意，Kubernetes将保留无效Node的对象，并继续检查它是否有效，除非它被客户端明确删除。

目前，有三个组件与Kubernetes Node接口进行交互：node controller、kubelet和kubectl。





### Node Controller

Node Controller是一个Kubernetes Master组件，管理Node的各个方面。

Node Controller在Node的生命周期中具有多个角色。首先，是在注册时将CIDR块分配给Node（如果CIDR分配已打开）。

> 译者按：CIDR（无类别域间路由）：<http://blog.csdn.net/xinianbuxiu/article/details/53560417>

其次，是使Node Controller的内部列表与云提供商的可用机器列表同步。在云环境中，每当Node不健康时，Node Controller就会请求云提供商，查询该Node的VM是否依然可用。如果不可用，Node Controller就会从其Node列表中删除该Node。

第三，是监视Node的健康状况。当Node变得不可达时，Node Controller负责将NodeStatus的NodeReady condition更新为ConditionUnknown（即：Node Controller由于某些原因停止接收心跳，例如由于Node关闭）。如果Node依然无法访问，那么就会从该Node中驱逐所有Pod （使用优雅关闭的方式）。（Node Controller与Node失联超过40秒，就会报告ConditionUnknown，5分钟后开始驱逐Pod）。Node Controller每隔`--node-monitor-period` 检查一次Node状态。

在Kubernetes 1.4中，我们更新了Node Controller的逻辑，从而更好地处理大量Node无法连接Master的情况（例如，由于Master有网络问题）。 从1.4开始，Node Controller在做关于Pod驱逐的决定时，会查看集群中所有Node的状态。

在大多数情况下，Node Controller将驱逐速率限制为 `--node-eviction-rate` （默认为0.1）每秒，这意味着它不会1个Node驱逐Pod花费的时间不会超过10秒。

当给定可用区（availability zone）中的Node变得不健康时，Node驱逐的行为就会发生变化。Node Controller同时也会检查区域中有多少百分比的Node是不正常的（NodeReady condition是ConditionUnknown或ConditionFalse）。如果不健康Node的阈值达到 `--unhealthy-zone-threshold` （默认为0.55），那么驱逐速率将被降低：如果集群很小（即小于或等于 `--large-cluster-size-threshold` 个Node，默认50），那么驱逐就会停止；否则驱逐速率降低到 `--secondary-node-eviction-rate` （默认0.01）每秒。每个可用区实施这些策略的原因，是因为每个可用区都可能会从Master断开，而另一个可用区仍然保持连接。如果您的集群不会跨越多个云提供商的可用区，那么就只有一个可用区（整个群集）。

> 译者按：可用区举例：AWS可用区：<http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html> 

在可用区之间传播Node的一个关键原因是：当整个区域停止时，工作负载可以转移到健康区域。因此，当一个区域中的所有Node都不健康时，那么Node Controller就以正常速率`--node-eviction-rate` 驱逐。 当所有区域都不健康时（即集群中没有健康的Node），Node Controller就会假定Master的连接有问题，并停止所有驱逐，直到连接恢复。

从Kubernetes 1.6开始，NodeController还负责驱逐运行在“NoExecute taint”的Node上的Pod，当Pod不能忍受这些taint时。 另外，作为默认禁用的Alpha功能，NodeController负责添加taint，这些taint与诸如Node可达或未准备好等问题相关。有关NoExecute taint和Alpha功能的详细信息，请参阅  [this documentation](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) 。

从1.8版开始，Node Controller可负责创建表示节点condition的taint。这是1.8版的Alpha功能。



### Node的自注册

当kubelet标志`--register-node` 为true（默认值）时，kubelet将会尝试向API server注册自己。这是大多数版本所使用的首选模式。

对于自注册，kubelet会使用如下的选项启动：

- `--kubeconfig` ：凭证向apiserver进行身份验证的路径。
- `--cloud-provider` ：如何与云提供商进行会话，从而获取自身的元数据。
- `--register-node` ：自动向API server注册。
- `--register-with-taints` ：注册具有给定taint列表的Node（逗号分隔的`<key>=<value>:<effect>` ）。 如果`register-node` 为false，则为No-op（空操作，啥都不干）。
- `--node-ip` ：Node的IP。
- `--node-labels` ：向集群注册Node时添加的标签。


- `--node-status-update-frequency` ：指定kubelet将Node状态发送到Master的频率。

目前，任何kubelet都被授权创建/修改任何Node资源，但实际上只能创建/修改其自身的资源。（将来，我们计划只允许一个kubelet修改自己的Node资源。）



### 手动管理Node

集群管理员可创建和修改Node对象。

如果管理员希望手动创建Node对象，可设置kubelet标志`--register-node=false` 。

管理员可修改Node资源（忽视`--register-node` 的设置）。修改操作包括：在Node上设置标签，并将其标记为不可调度。

Node上的Label可与Pod上的Node selector（Node选择器）一起使用，从而控制调度——例如，限制一个Pod只能在指定的节点列表上运行。

将Node标记为不可调度，将会阻止新的Pod被调度到该Node，但不会影响Node上的现有的Pod。这对于做Node重启之前的准备工作很有用。例如，要将node标记为不可调度，可使用如下命令：

```shell
kubectl cordon $NODENAME
```

**请注意**，由DaemonSet Controller创建的Pod会绕过Kubernetes调度程序，并且不遵循节点上的unschedulable属性。 因为，我们假设daemon进程属于机器，即使在准备重启时正被耗尽。



### Node容量

Node的容量（CPU数量和内存大小）是Node对象的一部分。 通常来说，Node在创建Node对象时注册自身，并报告其容量。如果您正在进行 [manual node administration](https://kubernetes.io/docs/concepts/architecture/nodes/#manual-node-administration) ，则需要在添加Node时设置Node容量。

Kubernetes调度程序可确保Node上的所有pod都有足够的资源。它会检查节点上容器的请求总和不大于Node容量。它包括由kubelet启动的所有容器，但不包括由Docker直接启动的容器，也不包含那些不运行在容器中的进程。

如果要明确保留非Pod进程的资源，可创建一个“placeholder pod（占位Pod）”。使用以下模板：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-reserver
spec:
  containers:
  - name: sleep-forever
    image: gcr.io/google_containers/pause:0.8.0
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
```

将`cpu` 和`memory` 值设置为您要保留的资源量。将该文件放在清单目录中（kubelet的`--config=DIR` 标志）。 在想要预留资源的每个kubelet上执行此操作。





## API对象

Node是Kubernetes REST API中的顶级资源。有关API对象的更多详细信息，可详见：[Node API object](https://kubernetes.io/docs/api-reference/v1.8/#node-v1-core) 。



## 原文

<https://kubernetes.io/docs/concepts/architecture/nodes/>