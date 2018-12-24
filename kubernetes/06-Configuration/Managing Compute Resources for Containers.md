# Managing Compute Resources for Containers（管理容器的计算资源）

> 译者按：本节中，笔者将request翻译成最小需求，limit翻译成最大限制。由于出现的次数太多，故而绝大多数地方直接不翻译了，大家可以当做术语来阅读。

指定 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 时，可选择指定每个容器需要多少CPU和内存（RAM）。当容器指定了最小资源需求时，Scheduler可对Pod调度到哪个Node上进行更好的决策。当容器具有指定的资源限制时，可以指定的方式，处理Node上资源的争抢。有关资源的最小需求和最大限制之间的差异的更多信息，请参阅 [Resource QoS](https://git.k8s.io/community/contributors/design-proposals/node/resource-qos.md) 。





## Resource types（资源类型）

*CPU*和*内存*都是*资源类型* 。资源类型有基本单元。CPU以核心为单位指定，内存以字节为单位指定。

CPU和内存统称为*计算资源* ，也可称为*资源* 。 计算资源是可以请求、分配和消费的，可测量的数量。它们与 [API resources](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) 。 API资源（如Pods和 [Services](https://kubernetes.io/docs/concepts/services-networking/service/) 是可通过Kubernetes API Server读取和修改的对象。





## Resource requests and limits of Pod and Container（Pod和容器资源的最小需求与最大限制）

Pod的每个容器可指定以下一个或多个：

- `spec.containers[].resources.limits.cpu`
- `spec.containers[].resources.limits.memory`
- `spec.containers[].resources.requests.cpu`
- `spec.containers[].resources.requests.memory`

尽管只能在每个容器上指定request和limit，但这样既可方便地算出Pod资源的request和limit。特定资源类型的Pod resource request/limit是Pod中每个容器该类型资源的request/limit的总和。





## Meaning of CPU（CPU的含义）

CPU资源的request和limit以*cpu为*单位。在Kubernetes中，一个cpu相当于：

- 1 AWS vCPU
- 1 GCP Core
- 1 Azure vCore
- 1 *Hyperthread* on a bare-metal Intel processor with Hyperthreading

允许小数。具有 `spec.containers[].resources.requests.cpu=0.5` 的容器，保证其所需的CPU资源是需要`1cpu` 容器资源的一半。表达式 `0.1` 等价于表达式 `100m` ，可看作“100millicpu”。有些人说“100 millicore”，表达的也是一个意思。具有小数点的请求（如 `0.1` ，会由API转换为 `100m` ，精度不超过 `1m` 。

CPU始终被要求作为绝对数量，从不作为相对数量；0.1在单核、双核或48核机器中，表示的是相同数量的CPU。





## Meaning of memory（内存的含义）

`memory` 的request和limit以字节为单位。可使用整数或定点整数来表示内存，并使用如下后缀之一：E、P、T、G、M、K；也可使用：Ei，Pi，Ti ，Gi，Mi，Ki。 例如，以下代表大致相同的值：

```
128974848, 129e6, 129M, 123Mi
```

如下是一个例子。如下Pod有两个容器。每个容器都有0.25 cpu和64MiB（226字节）内存的request。 每个容器的内存限制为0.5 cpu和128MiB。你可以说Pod有0.5 cpu和128 MiB内存的request，有1 cpu和256MiB内存的limit。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```





## How Pods with resource requests are scheduled（如何调度带有request的Pods）

当您创建一个Pod时，Kubernetes Scheduler将为Pod选择一个Node。对于各种资源类型，每个Node都有最大容量：可为Pod提供的CPU和内存量。Scheduler确保对于每种资源类型，调度到该Node的所有容器的request之和小于该Node的容量。请注意，尽管Node上的实际内存或CPU资源使用量非常低，但如果容量检查失败，那么Scheduler仍会拒绝在该Node上放置一个Pod。这样可在资源使用稍后增加时，例如在请求的高峰期，防止Node上的资源短缺。





## How Pods with resource limits are run（带有资源limit的Pod是如何运行的）

当kubelet启动Pod的容器时，它将CPU和内存限制传递到容器运行时。

使用Docker时：

-  `spec.containers[].resources.requests.cpu` 转换为其核心值，该值可能是小数，乘以1024。该数字中的较大值或2用作 `docker run` 命令中 [`--cpu-shares`](https://docs.docker.com/engine/reference/run/#/cpu-share-constraint) 的值。

-  `spec.containers[].resources.limits.cpu` 转换为其millicore值并乘以100。结果值是容器每100ms可以使用的CPU时间总量。 在此间隔期间，容器不能占用超过其CPU时间的份额。

  > **注意** ：默认配额期限为100ms。 CPU配额的最小分辨率为1ms。


-  `spec.containers[].resources.limits.memory` 会被转换为一个整数，并用作 `docker run`命令中 [`--memory`](https://docs.docker.com/engine/reference/run/#/user-memory-constraints) 标志的值。


如果容器超出其内存limit，则可能会被终止。如果容器能够重新启动，则与所有其他类型的运行时故障一样，kubelet将重新启动它。

如果一个容器超出其内存request，那么当Node内存不满足要求时，Pod可能会被逐出。

容器可能被允许或不允许长时间超过其CPU limit。 然而，即使CPU使用量过大，容器也不会被杀死。

要确定容器是否由于资源limit而无法调度或被杀死，请参阅 [Troubleshooting](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#troubleshooting) 部分。





## 监控计算资源使用情况（Monitoring compute resource usage）

Pod的资源使用情况被报告为Pod status的一部分。

如果为集群配置了 [optional monitoring](http://releases.k8s.io/master/cluster/addons/cluster-monitoring/README.md) ，那么即可从监控系统查询Pod资源的使用情况。





## Troubleshooting（故障排查）



### My Pods are pending with event message failedScheduling

如果Scheduler找不到任何Pod能够匹配的Node，则Pod将保持unscheduled状态。每当调度程序找不到地方调度Pod时，会产生一个事件，如下所示：

```shell
$ kubectl describe pod frontend | grep -A 3 Events
Events:
  FirstSeen LastSeen   Count  From          Subobject   PathReason      Message
  36s   5s     6      {scheduler }              FailedScheduling  Failed for reason PodExceedsFreeCPU and possibly others
```

在上述示例中，由于Node上的CPU资源不足，名为“frontend”的Pod无法调度。 如果内存不足，也可能会导致失败，并提示类似的错误消息（PodExceedsFreeMemory）。一般来说，如果一个Pod处于pending状态，并带有这种类型的消息，有几件事情要尝试：

- 向集群添加更多Node。
- 终止不需要的Pod，为处于pending的Pod腾出空间。
- 检查Pod是否不大于所有Node。例如，如果所有Node的容量为 `cpu: 1` ，那么request = `cpu: 1.1` 的Pod将永远不会被调度。

可使用 `kubectl describe nodes` 命令检查Node的容量和数量。 例如：

```shell
$ kubectl describe nodes e2e-test-minion-group-4lw4
Name:            e2e-test-minion-group-4lw4
[ ... lines removed for clarity ...]
Capacity:
 alpha.kubernetes.io/nvidia-gpu:    0
 cpu:                               2
 memory:                            7679792Ki
 pods:                              110
Allocatable:
 alpha.kubernetes.io/nvidia-gpu:    0
 cpu:                               1800m
 memory:                            7474992Ki
 pods:                              110
[ ... lines removed for clarity ...]
Non-terminated Pods:        (5 in total)
  Namespace    Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------    ----                                  ------------  ----------  ---------------  -------------
  kube-system  fluentd-gcp-v1.38-28bv1               100m (5%)     0 (0%)      200Mi (2%)       200Mi (2%)
  kube-system  kube-dns-3297075139-61lj3             260m (13%)    0 (0%)      100Mi (1%)       170Mi (2%)
  kube-system  kube-proxy-e2e-test-...               100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system  monitoring-influxdb-grafana-v4-z1m12  200m (10%)    200m (10%)  600Mi (8%)       600Mi (8%)
  kube-system  node-problem-detector-v0.1-fj7m3      20m (1%)      200m (10%)  20Mi (0%)        100Mi (1%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests    CPU Limits    Memory Requests    Memory Limits
  ------------    ----------    ---------------    -------------
  680m (34%)      400m (20%)    920Mi (12%)        1070Mi (14%)
```

由如上输出可知，如果一个Pod的request超过1120mCPU或6.23Gi内存，它将不适合该Node。

通过查看 `Pods` 部分，可查看哪些Pod占用Node上的空间。

> 译者按：CPU 1120m是这么算的：1800m（Allocatable） - 680m（Allocated）。同理，内存是7474992Ki - 1070Mi

Pods所用的资源量必须小于Node容量，因为系统守护程序需要使用一部分资源。 `allocatable` 字段 [NodeStatus](https://kubernetes.io/docs/resources-reference/v1.8/#nodestatus-v1-core) 给出了Pod可用的资源量。有关更多信息，请参阅 [Node Allocatable Resources](https://git.k8s.io/community/contributors/design-proposals/node/node-allocatable.md) 。

可配置 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 功能，从而限制能够使用的资源总量。 如果与Namespace一起使用，则可防止一个团队占用所有资源。



### My Container is terminated

由于资源不足，容器可能会被终止。要查看容器是否因为资源限制而被杀死，请在感兴趣的Pod上调用 `kubectl describe pod` ：

```shell
[12:54:41] $ kubectl describe pod simmemleak-hra99
Name:                           simmemleak-hra99
Namespace:                      default
Image(s):                       saadali/simmemleak
Node:                           kubernetes-node-tf0f/10.240.216.66
Labels:                         name=simmemleak
Status:                         Running
Reason:
Message:
IP:                             10.244.2.75
Replication Controllers:        simmemleak (1/1 replicas created)
Containers:
  simmemleak:
    Image:  saadali/simmemleak
    Limits:
      cpu:                      100m
      memory:                   50Mi
    State:                      Running
      Started:                  Tue, 07 Jul 2015 12:54:41 -0700
    Last Termination State:     Terminated
      Exit Code:                1
      Started:                  Fri, 07 Jul 2015 12:54:30 -0700
      Finished:                 Fri, 07 Jul 2015 12:54:33 -0700
    Ready:                      False
    Restart Count:              5
Conditions:
  Type      Status
  Ready     False
Events:
  FirstSeen                         LastSeen                         Count  From                              SubobjectPath                       Reason      Message
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {scheduler }                                                          scheduled   Successfully assigned simmemleak-hra99 to kubernetes-node-tf0f
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   pulled      Pod container image "gcr.io/google_containers/pause:0.8.0" already present on machine
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   created     Created with docker id 6a41280f516d
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    implicitly required container POD   started     Started with docker id 6a41280f516d
  Tue, 07 Jul 2015 12:53:51 -0700   Tue, 07 Jul 2015 12:53:51 -0700  1      {kubelet kubernetes-node-tf0f}    spec.containers{simmemleak}         created     Created with docker id 87348f12526a
```

在上述示例中， `Restart Count: 5` 表示Pod中的 `simmemleak` 容器已终止并重启了5次。

可使用 `kubectl get pod` 的`-o go-template=...` 选项来获取先前终止的Containers的状态：

```shell
[13:59:01] $ kubectl get pod -o go-template='{{range.status.containerStatuses}}{{"Container Name: "}}{{.name}}{{"\r\nLastState: "}}{{.lastState}}{{end}}'  simmemleak-hra99
Container Name: simmemleak
LastState: map[terminated:map[exitCode:137 reason:OOM Killed startedAt:2015-07-07T20:58:43Z finishedAt:2015-07-07T20:58:43Z containerID:docker://0e4095bba1feccdfe7ef9fb6ebffe972b4b14285d5acdec6f0d3ae8a22fad8b2]]
```

您可以看到容器由于 `reason:OOM Killed` 而终止，其中`OOM`代表Out Of Memory。





## Local ephemeral storage (alpha feature)（ephemeral-storage，本地临时存储（Alpha功能））

Kubernetes 1.8版本引入了一种新的资源，用于管理本地临时存储的ephemeral-storage。 在每个Kubernetes Node中，kubelet的根目录（默认 `/var/lib/kubelet` ）和日志目录（ `/var/log` ）存储在Node的根分区上。 此分区也可由Pod通过EmptyDir Volume、容器日志、镜像层以及容器可写层等进行共享和使用。

该分区是“短暂的”，应用程序不能对此分区的性能SLA（例如磁盘IOPS）有期望。 Local ephemeral storage管理仅适用于根分区；镜像层和可写层的可选分区超出了Local ephemeral storage的范围。

**注意：**如果使用可选的运行时分区，根分区将不会保存任何镜像层或可写层。

> 译者按：
>
> SLA：<https://baike.baidu.com/item/SLA/2957862>
>
> IOPS：<https://baike.baidu.com/item/IOPS/3105194>
>
> 系统SLA和监控流程：<http://www.doc88.com/p-9082091179407.html> 



### Requests and limits setting for local ephemeral storage（local ephemeral storage的request和limit设置）

Pod的每个容器可指定以下一个或多个：

- `spec.containers[].resources.limits.ephemeral-storage`
- `spec.containers[].resources.requests.ephemeral-storage`

`ephemeral-storage` 的request和limit以字节为单位。可使用整数或定点整数来表示内存，并使用如下后缀之一：E、P、T、G、M、K。也可使用：Ei，Pi，Ti ，Gi，Mi，Ki。 例如，以下代表大致相同的值：

```
128974848, 129e6, 129M, 123Mi
```

例如，以下Pod有两个容器。每个容器有一个2GiB的local ephemeral storage的request。每个容器的local ephemeral storage的limit是4GiB。因此，Pod有4GiB的local ephemeral storage的request，limit为8GiB。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
  - name: wp
    image: wordpress
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
```



### How Pods with ephemeral-storage requests are scheduled（如何调度设置了ephemeral-storage request的Pod）

当您创建一个Pod时，Kubernetes Scheduler将为Pod选择一个Node。每个Node具有能够为Pod提供的local ephemeral storage最大量值。（有关详细信息，请参见 [“Node Allocatable”](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) 。Scheduler确保调度的容器的资源需求总和小于Node的容量。



### How Pods with ephemeral-storage limits run（如何运行设置了ephemeral-storage limit的Pod）

对于容器级别的隔离，如果容器可写层和日志的使用超出其存储限制，则该Pod将被驱逐。对于Pod级别的隔离，如果所有容器的local ephemeral storage使用量的综合超过限制，则Pod将被驱逐，同理，Pod的EmptyDir也是如此。





## Opaque integer resources (alpha feature) （不透明的整数资源（alpha特征））

**废弃通知：**从 `Kubernetes v1.8` 开始，该特性已被 [deprecated](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#) 。

既已废弃，就没有翻译的必要了。多抱半小时老婆吧。该功能的替代品是Extended Resources。





## Extended Resources（扩展资源）

Kubernetes 1.8版引入了Extended Resources。Extended Resources是 `kubernetes.io` 域名之外的完全资格的资源名称。Extended Resources允许集群运营商发布新的Node级别的资源，否则系统将无法识别这些资源。 Extended Resources数量必须是整数，不能过大。

用户可像CPU和内存一样使用Pod spec中的Extended Resources。 Scheduler负责资源计算，以便分配给Pod的资源部超过可用的资源量。

API Server将Extended Resources的数量限制为整数，例如 `3Ki` 和 `3Ki` 是有效的，`0.5`和`1500m` 是无效的。

**注意：**扩展资源替代 [Opaque Integer Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#opaque-integer-resources-alpha-feature) 。 用户可使用 `kubernetes.io/` 域名之外的任何域名前缀，而非以前的 `pod.alpha.kubernetes.io/opaque-int-resource-` 前缀。

使用Extended Resources需要两步。首先，集群操作员必须在一个或多个Node上发布per-node Extended Resource。第二，用户必须在Pod中请求Extended Resource。

要发布新的Extended Resource，集群操作员应向API Server提交 `PATCH` HTTP请求，从而指定集群中Node的 `status.capacity` 。在此操作之后，Node的 `status.capacity` 将包含一个新的资源。 `status.allocatable` 字段由kubelet异步地使用新资源自动更新。请注意，由于Scheduler在评估Pod适应度时，会使用Node的`status.allocatable`值，所以在 使用新资源PATCH到Node容量 和 第一个Pod请求该Node上资源 之间可能会有短暂的延迟。

**示例：**

如下是一个示例，显示如何使用 `curl` 构建一个HTTP请求，该请求在Node `k8s-node-1` （Master是`k8s-master` ）上发布了5个“example.com/foo”资源。

```shell
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1foo", "value": "5"}]' \
http://k8s-master:8080/api/v1/nodes/k8s-node-1/status
```

**注意** ：在上述请求中， `~1` 是PATCH路径中字符 `/` 的编码。 JSON-Patch中的操作路径值被拦截为JSON指针。 有关更多详细信息，请参阅 [IETF RFC 6901, section 3](https://tools.ietf.org/html/rfc6901#section-3) 。

要在Pod中使用Extended Resource，请将资源名称作为 `spec.containers[].resources.requests` map中key。

**注意：**Extended resources不能提交过大的值，因此如果request和limit都存在于容器spec中，则两者必须相等。

> TODO：这是什么意思？

只有当所有资源的request都满足时（包括cpu、内存和任何Extended Resources），Pod才会被调度。只要资源的request无法被任何Node满足，Pod将保持在 `PENDING` 状态。

**示例：**

下面的Pod有如下request：2 cpus和1“example.com/foo”（extended resource）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: myimage
    resources:
      requests:
        cpu: 2
        example.com/foo: 1
```





## Planned Improvements（计划改进）

Kubernetes 1.5仅允许在容器上指定资源量。计划对Pod中所有容器共享资源的计费进行改进，例如 [emptyDir volumes](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) 。

Kubernetes 1.5仅支持容器级别的CPU/内存的request/limit。 计划添加新的资源类型，包括node disk space resource和用于添加自定义 [resource types](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/resources.md) 的框架。

Kubernetes通过支持多层的 [Quality of Service](http://issue.k8s.io/168) 支持overcommitment of resources。

> overcommitment of resources：笔者理解就是资源超售。
>
> Quality of Service在部分K8s文档上也被简写成QoS。

在Kubernetes 1.5中，对于不同云提供商，或对于同一个云提供商中的不同机器类型，一个CPU单位表达的是不同的意思。 例如，在AWS上，Node的容量在 [ECUs](http://aws.amazon.com/ec2/faqs/) 中报告，而在GCE中报告为逻辑内核。我们计划修改cpu资源的定义，从而使得在提供商和平台之间更一致。





## What’s next

- 掌握 [assigning Memory resources to containers and pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/) 的实践经验。
- 掌握 [assigning CPU resources to containers and pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/) 的实践经验。
- [Container](https://kubernetes.io/docs/api-reference/v1.8/#container-v1-core)
- [ResourceRequirements](https://kubernetes.io/docs/resources-reference/v1.8/#resourcerequirements-v1-core)






## 原文

<https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/>