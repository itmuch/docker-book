# 从Pod说起

* Pod是Kubernetes中可以创建和管理的最小部署单元。是K8s种最小的调度单位
* 可以认为是一个“虚拟机”
* 是一个逻辑概念

> **拓展与提问**：如果没有Pod，而把容器作为最小的调度单位，由于容器是“单进程模型”，会出现什么问题？
>
> 举个例子：rsyslogd包括三个模块（进程）：imklog、imuxsock、rsyslogd。三个进程必须运行在一台机器上，否则基于Socket的通信和文件交换会出问题。于是，我们把三个模块分别制作成三个不同的容器，并未三个容器分别分配1G的内存。
>
> 如果没有Pod，那么为了让三个容器运行在一台node上，我们可能会这么做：首先启用imklog，让它调度到一台服务器（记为Node1）上，然后将imuxsock、rsyslogd也调度到imklog所在的机器上（一般通过设置亲和性的方式）。但假如Node1总共只有2G的内存，那么就势必调度失败；但是呢，由于设置了亲和性，又意味着这三个容器必须调度到Node1上。
>
> 这是一个非常经典的调度问题。不同的容器编排框架采用不同的方式去解决以上问题。例如：Mesos采用资源囤积的方式解决这个问题（即：当且仅当所有设置亲和性约束的任务都达到时，才统一调度）
>
> Kubernetes采用Pod的方式，也良好地解决了该问题。



## 什么是Pod

Pod（像鲸鱼群或豌豆荚）由一个或多个容器（例如Docker容器）组成的容器组，组内容器具有共享存储、网络，以及容器的运行规范。 Pod的内容始终是被同时调度。 Pod可被认为是运行特定应用程序的“逻辑主机”——它包含一个或多个相对紧密耦合的应用容器——在容器启动之前，应用容器总是会被调度到在相同的Node上。

尽管Kubernetes比Docker支持更多的容器运行时，但Docker是最常见的运行时，这样有助于使用Docker术语中描述Pod。

Pod的共享上下文是一组Linux命名空间、cgroups和潜在的其他方面的隔离机制——这一点与Docker容器的隔离机制一致。 在Pod的上下文中，各个应用程序可能会有更小的子隔离环境。

Pod中的容器共享IP地址和端口，并且可通过`localhost` 找到对方。它们还可使用标准的进程间通信（IPC）（例如SystemV信号量或POSIX共享内存）进行通信。不同Pod中的容器有不同的IP地址，不能通过IPC进行通信。

Pod中的应用程序还可以访问共享卷，这些卷定义为Pod的一部分并挂载到每个应用程序的文件系统中。

根据Docker的结构，Pod被建模为一组具有共享namespace和[共享卷](https://kubernetes.io/docs/concepts/storage/volumes/) 的Docker容器。 暂不支持PID namespace的共享。

和应用容器一样，Pod被认为是相对短暂的（非持久）实体。 如在[Pod 生命周期](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) 所讨论的，创建Pod后，会分配唯一的ID（UID），并将其调度到Node上，直到终止（根据重启策略）或删除。 如果一个Node挂掉，那么在经过超时时间后，调度到该节点的Pod被将被删除。 一个给定的（例如UID定义的）Pod不会“重新调度”到新的Node上；而是被相同的Pod取代。如有需要，甚至可使用相同的名称，但会使用新的UID（有关更多详细信息，请参阅 [replication controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) ）。（将来，会有更高级别的API将支持Pod迁移。）

当某些东西与一个Pod的生命周期（例如Volume）相同时，这意味着只要该Pod（具有该UID）存在就存在。 如果由于任何原因删除了该Pod，那么，即使创建了相同的Pod（以替代之前的Pod），相关内容（例如Volume）也会被销毁并重新创建。

![包含文件下载器和使用持久卷在容器之间共享存储的Web服务器的多容器Pod](images/pod.svg)





## Pod的动机

### 管理

Pod是一个服务中多个进程的聚合单位，通过更高级别的抽象来简化应用程序的部署和管理。 在K8s中，Pod是部署，横向缩放和复制的单位。Pod中的容器可自动处理托管（共同调度）、共享命运（例如终止）、协调复制，资源共享和依赖关系管理。

### 资源共享和通信

Pods中的成员之间可进行数据共享和通信。

Pod中的应用程序都使用相同的网络namespace（相同的IP和端口），因此可使用 `localhost` 互相发现。 因此，Pod中的应用程序必须协调其端口的使用。 每个Pod都会在一个扁平的共享网络空间中分配一个IP地址，从而与其他物理机以及Pod通信。

对于Pod中的容器，主机名会被设为Pod的名称。 [更多相关网络的内容](https://kubernetes.io/docs/concepts/cluster-administration/networking/) 。

除定义应用程序容器之外，Pod还指定了一组共享Volume。Volume允许数据在容器重启之后不丢失，并在Pod中的应用程序之间共享。



## Pod的使用

Pods可用于托管垂直集成的应用栈（例如LAMP），但其主要动机是支持共同协作、共同管理的工作程序，例如：

- 内容管理系统，文件和数据加载器，本地缓存管理等
- 日志和检查点备份，压缩，旋转，快照等
- 数据更改观察者，日志分配器，日志记录和监视适配器，事件发布者等
- 代理，桥接器和适配器
- 控制器，管理器，配置器和更新器

**一般来说，单个Pod不会运行同一应用的多个实例。**

详情请看[The Distributed System ToolKit: Patterns for Composite Containers](http://blog.kubernetes.io/2015/06/the-distributed-system-toolkit-patterns.html)



## 考虑替代方案

*为什么不在单个（Docker）容器中运行多个程序？*

1. 透明。 使Pod中的容器对基础设施可见，以便基础设施为这些容器提供服务，例如进程管理和资源监控。 这为用户提供了一些便利。
2. 解耦软件依赖。各个容器可以独立进行版本管理、重建和重新部署。 未来，Kubernetes甚至有可能支持单个容器的的实时更新。
3. 使用方便。 用户无需运行自己的进程管理器，担心信号以及退出代码传播等。
4. 效率。 因为基础设施承担起更多的责任，容器更加轻量级。

*为什么不支持基于亲和性部署的容器协同调度？*

这种方法将会提供协同定位，但无法提供Pod的大部分优势，例如资源共享，IPC（进程间通信），保证命运共享和简化管理。



## Pod的持久性（或缺乏持久性）

Pod不能被视为持久的实体。 它们不会因调度失败，节点故障或其他问题而生存，例如由于缺乏资源，或者在节点维护的情况下。

一般来说，用户无需直接创建Pod。他们几乎总是使用Controller（例如 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) ），即使是创建单个Pod时。 Controller提供集群范围的自我修复、复制和升级管理。

集体API作为面向用户的语言的方式，在在集群调度系统中相对比较常见，包括 [Borg](https://research.google.com/pubs/pub43438.html) 、 [Marathon](https://mesosphere.github.io/marathon/docs/rest-api.html) 、 [Aurora](http://aurora.apache.org/documentation/latest/reference/configuration/#job-schema) 以及 [Tupperware](http://www.slideshare.net/Docker/aravindnarayanan-facebook140613153626phpapp02-37588997) 都采用这种方式。

Pod被暴露为原始API，以便于：

- 调度程序和控制器可插拔性
- 支持Pod级操作，而无需通过控制器API“代理”它们
- 将Pod生命周期与控制器生命周期解耦
- 控制器和服务的解耦——端点控制器秩序观察Pod
- 清晰地将Kubelet级的功能与集群级的功能组合——Kubelet实际上是“Pod控制器”
- 高可用性应用，Pod可在其终止或删除之前被替换，例如在计划的驱逐，图像预取或Pod实时移植的情况下[＃3949](http://issue.k8s.io/3949)

[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset.md) 控制器（目前处于Beta测试阶段），支持有状态Pod。该功能在1.4中是alpha测试阶段，被称为PetSet。对于先前版本的Kubernetes，有状态Pod的最佳做法是创建一个replication controller，其`replicas` 等于`1` 并提供相应的服务，详见[this MySQL deployment example](https://kubernetes.io/docs/tutorials/stateful-application/run-stateful-application/)。



## Pod的终止

因为Pod代表集群中Node上运行的进程，所以当不再需要这些进程时，允许这些进程正常终止（比起使用KILL信号这种暴力的方式）是非常重要的。用户应该能够请求删除并知道进程何时终止，但也能够确保删除最终完成。 当用户请求删除Pod时，系统会在Pod被强制杀死之前记录预期的优雅关闭时间，并且TERM信号被发送到每个容器的主进程。一旦超过优雅关闭时间，就会向这些进程发送KILL信号，然后从API Server中删除该Pod。如果在等待进程终止的过程中Kubelet或Container Manager重启了，那么，在重启后仍会重试完整优雅关闭。

示例流程：

1. 用户发送删除Pod的命令，默认优雅关闭时间是30s
2. 随着时间的推移，API Server中的Pod状态会被更新，Pod会被标记为“dead”，并开始进入优雅关闭时间
3. 当在客户端命令行中列出Pod时，状态显示为“Terminating”
4. （与3同时）当Kubelet看到Pod已被标记为Terminating，因为2中的时间已经设置，它将开始Pod关闭进程。
5. 如果Pod定义了[preStop hook](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-details) ，它将在Pod中被调用。如果 `preStop` Hook在优雅关闭时间到期后仍在运行，则会在第二步中增加2秒的优雅关闭时间。
6. Pod中的进程发送TERM信号。
7. （与3同时），Pod将从该服务的端点列表中删除，并且不再被认为是replication controllers正在运行的Pod的一部分。 缓慢关闭的Pod可以继续接收从load balancer转发过来的流量，直到load balancer将其从可用列表中移除。
8. 当优雅关闭时间到期时，仍在Pod中运行的任何进程都会被SIGKILL杀死。
9. 通过设置优雅关闭时间为0（立即删除），Kubelet将在API Server上完成Pod的删除。 Pod从API消失，并且在客户端中也不可见。

默认情况下，优雅删除时间是30秒。 `kubectl delete` 命令支持`--grace-period=<seconds>` 选项，允许用户覆盖默认值并指定自己的值。 如果设置为`0`，则表示 [强制删除](https://kubernetes.io/docs/concepts/workloads/pods/pod/#force-deletion-of-pods)  Pod。 当kubectl version >= 1.5时，您必须同时使用标志`--force` 与`--grace-period=0` 来强制删除Pod。



### 强制删除Pod

一个Pod的强制删除被定义为从群集状态和etcd立即删除一个Pod。 当执行强制删除时，API Server不等待来自Kubelet的确认，而是直接在其运行的Node上终止该Pod。它立即删除API Server中的Pod，以便创建一个新的同名Pod。在Node上，被设为立即终止的Pod在被强制杀死之前仍然会有一个较小的优雅关闭时间。

**强制删除对于某些Pod可能是危险的，应慎用**。在StatefulSet Pod的情况下，请参阅[deleting Pods from a StatefulSet](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/) 。



## Pod phase

Pod的`status` 字段是一个[PodStatus](https://kubernetes.io/docs/resources-reference/v1.8/#podstatus-v1-core) 对象，它有一个`phase` 字段。

Pod的phase是Pod在其生命周期中的简单、高级的概述。phase并不是对容器或Pod状态的综合汇总，也不是作为了做为状态机。

phase值的数量和含义被严格保护。除了这里记录的内容，不应该假定`phase` 有其他值。

以下是`phase` 可能的取值：

- Pending：Pod已被Kubernetes系统接受，但一个或多个容器镜像尚未创建。该时间调度Pod所花费的时间以及以及通过网络下载镜像的时间，这可能需要一段时间。
- Running：Pod已绑定到一个节点，并且所有的容器都已创建。至少有一个容器仍在运行，或正在启动或重新启动。
- Succeeded：Pod中的所有容器已成功终止，不会重新启动。
- Failed：Pod中的所有容器都已终止，并且至少有一个容器已终止失败。也就是说，容器以非零状态退出或被系统终止。
- Unknown：由于某种原因，无法获得Pod状态，通常是由于与Pod所在主机通信时出现错误。





## Pod Condition

Pod有一个PodStatus，它有一个[PodConditions](https://kubernetes.io/docs/resources-reference/v1.8/#podcondition-v1-core) 数组。 PodCondition数组的每个元素都有一个`type` 字段和一个`status` 字段。 `type` 字段是一个字符串，可能的值为`PodScheduled、Ready、Initialized以及 Unschedulable` 。 `status` 字段是一个字符串，可能的值为`True，False和Unknown` 。





## 容器探针

 [Probe](https://kubernetes.io/docs/resources-reference/v1.8/#probe-v1-core) 是由[kubelet](https://kubernetes.io/docs/admin/kubelet/) 对容器定期执行的诊断。要执行诊断，Kubelet调用由Container实现的[Handler](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler) 。 有三种类型的handler：

- [ExecAction](https://kubernetes.io/docs/resources-reference/v1.8/#execaction-v1-core)：在Container中执行指定的命令。 如果命令退出的状态码为0，则认为诊断成功。
- [TCPSocketAction](https://kubernetes.io/docs/resources-reference/v1.8/#tcpsocketaction-v1-core) ：对容器IP的指定端口执行TCP检查。如果端口打开，则认为诊断成功。
- [HTTPGetAction](https://kubernetes.io/docs/resources-reference/v1.8/#httpgetaction-v1-core) ：对容器IP的指定端口和路径执行HTTP Get请求。如果响应的状态码大于等于200且小于400，则认为诊断成功。

每个探针都有三个结果之一：

- Success：容器已通过诊断。
- Failure：容器未通过诊断。
- Unknown：诊断失败，因此不会采取任何行动。



kubelet可以选择对运行容器上的两种探针执行和反应：

- `livenessProbe` （活动探针）：指示容器是否正在运行。如活动探测失败，那么kubelet就会杀死容器，并且容器将受到其 [重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 的影响。 如容器不提供活动探针，则默认状态为 `Success` 。
- `readinessProbe` （就绪探针）：指示容器是否准备好服务请求。 如就绪探测失败，Endpoint Controller将会从与Pod匹配的所有Service的端点中删除该Pod的IP地址。初始延迟之前的就绪状态默认为`Failure` 。 如果容器不提供就绪探针，则默认状态为`Success` 。



### 什么时候应该使用活动探针或就绪探针？

如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则**不一定需要活动探针**；kubelet将根据Pod的`restartPolicy`自动执行正确的操作。

如果您希望容器在探测失败时被杀死并重新启动，那么请指定一个**活动探针**，并指定`restartPolicy` 为Always或OnFailure。

如果要仅在探测成功时才开始向Pod发送流量，请指定**就绪探针**。在这种情况下，就绪探针可能与活动探针相同，但是spec中存在就绪探针，就意味着Pod会在没有接收到任何流量的情况下启动，并且只在探针探测成功后才开始接收流量。

如果您希望容器能够自己维护，可指定一个**就绪探针**，该探针检查的端点与活动探针不同。

请注意，如果您只想在Pod被删除时排除请求，则**不一定需要就绪探针**；在删除时，Pod会自动将自身置于未完成状态，无论就绪探针是否存在。在等待Pod中的容器停止的过程中，Pod仍处于未完成状态。



## Pod及容器状态

有关Pod容器状态的详细信息，请参阅 [PodStatus](https://kubernetes.io/docs/resources-reference/v1.8/#podstatus-v1-core) 和 [ContainerStatus](https://kubernetes.io/docs/resources-reference/v1.8/#containerstatus-v1-core) 。 请注意，作为Pod状态报告的信息取决于当前的 [ContainerState](https://kubernetes.io/docs/resources-reference/v1.8/#containerstatus-v1-core) 。



## 重启策略

PodSpec有一个`restartPolicy` 字段，可能的值为Always，OnFailure和Never，默认值为Always。 `restartPolicy`适用于Pod中的所有容器。 `restartPolicy` 仅指同一Node上kubelet重启容器时所使用的策略。 失败的容器由kubelet重启，以五分钟上限的指数退避延迟（10秒，20秒，40秒...），并在成功执行十分钟后重置。 如 [Pods document](https://kubernetes.io/docs/user-guide/pods/#durability-of-pods-or-lack-thereof) 中所述，一旦绑定到一个Node，Pod将永远不会重新绑定到另一个Node。





## Pod的寿命

一般来说，Pod不会消失，直到有人销毁它们——可能是人工或Controller去销毁Pod。 这个规则的唯一例外是Pod的`phase` 成功或失败超过一段时间（由Master确定）的Pod将过期并被自动销毁。

有三种类型的Controller可用：

- [Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) ，Pod预期会终止，例如批量计算。Job仅适用于`restartPolicy` 为OnFailure或Never的Pod。
- [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 、 [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 以及 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) ，Pod预期不会终止，例如Web服务器。 ReplicationController仅适用于`restartPolicy` 为Always的Pod。
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) ，每台机器都需要运行一个Pod，因为它们提供特定于机器的系统服务。

以上三种类型的Controller都包含一个PodTemplate。建议创建适当的Controller，让Controller创建Pod，而非直接创建Pod。这是因为单独的Pod在机器故障发生时无法自我修复，而Controller可以。

如果Node挂掉或与集群断开，那么Kubernetes应用一种策略，将故障Node上的所有Pod的`phase`设为Failed。





## 示例



### 高级活动探针示例

活动探测由kubelet执行，因此所有请求都会在kubelet网络命名空间中进行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: gcr.io/google_containers/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
          - name: X-Custom-Header
            value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```

容器的源码：

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }})
```

参考：<http://blog.51cto.com/kusorz/2069412>



### 状态示例

- Pod正在运行，并只有一个容器。容器退出成功。
  - 记录完成事件
  - 如果`restartPolicy`是：
    - Always: 重启容器，Pod的`phase` 保持Running
    - OnFailure: Pod的`phase` 变成Succeeded.
    - Never: Pod的`phase` 变成Succeeded.
- Pod正在运行，并只有一个容器。容器退出失败。
  - 记录失败事件
  - 如果`restartPolicy` 是：
    - Always: 重启容器，Pod的`phase` 保持Running.
    - OnFailure: 重启容器，Pod的`phase` 保持Running.
    - Never: Pod的`phase` 变成Failed.
- Pod正在运行，并且有两个容器，Container 1退出失败。
  - 记录失败事件
  - 如果`restartPolicy` 是：
    - Always：重启容器，Pod的`phase` 保持Running.
    - OnFailure：重启容器; Pod的`phase` 保持Running.
    - Never：不会重启容器，Pod的`phase` 保持Running.
  - 如果Container 1不在运行，Container退出
    - 记录失败事件
    - 如果`restartPolicy` 是：
      - Always：重启容器，Pod的`phase` 保持Running.
      - OnFailure：重启容器，Pod的`phase` 保持Running.
      - Never：Pod的`phase` 变成Failed.
- Pod正在运行，并且只有一个容器，容器内存溢出：
  - 容器失败并终止
  - 记录OOM事件
  - 如果`restartPolicy` 是：
    - Always: 重启容器，Pod的`phase` 保持Running.
    - OnFailure: 重启容器，Pod `phase` 保持Running.
    - Never: 记录失败事件，Pod `phase` 变成Failed.
- Pod正在运行，磁盘损坏：
  - 杀死所有容器
  - 记录适当事件
  - Pod的`phase` 变成Failed
  - 如果Pod是用Controller创建的，Pod将在别处重建
- Pod正在运行，Node被分段
  - Node Controller等待，直到超时
  - Node Controller将Pod的`phase` 设为Failed
  - 如果Pod是用Controller创建的，Pod将在别处重建



