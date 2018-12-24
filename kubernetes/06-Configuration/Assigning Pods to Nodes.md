# Assigning Pods to Nodes（将Pod分配到Node）

您可以约束一个 [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) ，让其只能在特定 [nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) 上运行，或者更倾向于在特定Node上运行。有几种方法能做到这点，他们都使用 [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 进行选择。通常这种约束不是必要的，因为Scheduler会自动执行合理的放置（例如：在Node之间传播您的Pod，而不是将Pod放在一个没有足够资源的Node上等），但在某些情况下，您可能需要控制一个Pod落在特定的Node上，例如：确保一个Pod在一个有SSD的机器上运行，或者将来自两个不同的服务的Pod放到相同的可用区域。

您可以在 [in our docs repo here](https://github.com/kubernetes/kubernetes.github.io/tree/master/docs/user-guide/node-selection) 找到这些示例的所有文件。





## nodeSelector

`nodeSelector`是最简单的约束形式。 `nodeSelector` 是PodSpec的一个字段。它指定键值对的映射。为了使Pod能够在Node上运行，Node必须将每个指定的键值对作为标签（也可有其他标签）。最常见的用法是使用一个键值对。

下面，我们来来看一下如何使用 `nodeSelector` 。



### Step Zero: Prerequisites（先决条件）

这个例子假设你对Kubernetes Pod有一个基本的了解，并已经 [turned up a Kubernetes cluster](https://github.com/kubernetes/kubernetes#documentation) 。



### Step One: Attach label to the node（将Label附加到Node）

运行 `kubectl get nodes` ，获取集群Node的名称。选择要添加Label的Node，然后运行 `kubectl label nodes <node-name> <label-key>=<label-value>` ，向您所选的Node添加Label。例如，如果我的Node名称是`kubernetes-foo-node-1.ca-robinson.internal` ，而我想要的标签是`disktype=ssd` ，那么可使用 `kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd` 。

如果此命令出现“invalid command”的错误，那么你可能使用的是旧版本的kubectl，它没有 `label` 命令。 在这种情况下，有关如何在Node上手动设置Label的说明，请参阅本指南的 [previous version](https://github.com/kubernetes/kubernetes/blob/a053dbc313572ed60d89dae9821ecab8bfd676dc/examples/node-selection/README.md) 。

另外请注意，Label的key必须采用DNS标签的格式（如 [identifiers doc](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md) 所述），这意味着key不允许包含大写字母。

您可以通过重新运行 `kubectl get nodes --show-labels` 并检查Node是否已经有你所设的标签标签，来验证Label是否成功添加。



### Step Two: Add a nodeSelector field to your pod configuration（将nodeSelector字段添加到您的Pod配置）

在任意一个你想运行的Pod的配置文件中添加nodeSelector部分，如下所示。例如，如果我的Pod配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

需要像这样添加一个nodeSelector：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

当您运行 `kubectl create -f pod.yaml` 时 ，该Pod将在您拥有以上标签的Node上调度！您可以通过运行  `kubectl get pods -o wide` 查看该Pod所在的“NODE”，来验证是否正常工作。





## Interlude: built-in node labels（Interlude：Node的内置标签）

除你 [attach](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-one-attach-label-to-the-node) 的Label以外，Node还预设了一组标准的Label。 从Kubernetes v1.4起，这些Label是：

- `kubernetes.io/hostname`
- `failure-domain.beta.kubernetes.io/zone`
- `failure-domain.beta.kubernetes.io/region`
- `beta.kubernetes.io/instance-type`
- `beta.kubernetes.io/os`
- `beta.kubernetes.io/arch`





## Affinity and anti-affinity（亲和与反亲和）

`nodeSelector` 提供了一种非常简单的方式，从而将Pod约束到具有特定Label的Node。目前处于beta阶段的Affinity/Anti-affinity特性极大地扩展了您可以表达的约束类型。关键的改进是：

1. 语言更具表现力（不仅仅是“使用AND、完全匹配”）
2. 可指定“soft”/“preference”规则，而非影响要求，因此如果Scheduler不能满足你的要求，则该Pod仍将被安排
3. 您可以限制在Node（或其他拓扑域）上运行的其他Pod的标签，而非针对Node本身上的标签，这允许设置规则，指定哪些Pod可以并且不能放到一起。

Affinity特性由两种affinity组成：“node affinity”和“inter-pod affinity/anti-affinity”。Node affinity类似于现有的节点 `nodeSelector` （但具有上面列出的前两个优点）；而inter-pod affinity/anti-affinity约束Pod的Label而非Node的Label，此特性除具有上面列出的前两项性质之外，还有如上述第三项中所述的性质。

`nodeSelector` 继续像往常一样工作，但最终将会被废弃，因为Node Affinity可表示 `nodeSelector` 所能表达的所有内容。



### Node affinity (beta feature)（Node亲和性（beta特性））

Node Affinity在Kubernetes 1.2中作为alpha功能引入。Node Affinity在概念上类似于`nodeSelector` ——它允许您根据Node上的Label来约束您的Pod可被调度哪些Node。

目前有两种类型的Node Affinity，称为 `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution` 。 您可以将它们分别认为是“hard”和“soft”，前者规定了要将Pod调度到Node上时，必须满足的规则（就像`nodeSelector` ，但使用更具表现力的语法），而后者指定调度程序将尝试强制调度但不能保证的首选项。 名称中的“IgnoredDuringExecution”部分表示，与 `nodeSelector` 工作方式类似，如果Node上的Label在运行时更改，导致不再满足Pod上的Affinity规则，则该Pod仍将继续在该Node上运行。在未来，我们计划提供 `requiredDuringSchedulingRequiredDuringExecution`  ，它和 `requiredDuringSchedulingIgnoredDuringExecution` 一样，只是它会从不再满足Pod的Node Affinity要求的Node中驱逐Pod。

因此， `requiredDuringSchedulingIgnoredDuringExecution` 的一个示例是“仅在有Intel CPU的Node上运行Pod”，并且一个`preferredDuringSchedulingIgnoredDuringExecution` 的一个示例是“尝试在可用区XYZ中运行此组Pod，但如果无法做到，则允许在其他地方运行” 。

Node Affinity被指定`nodeAffinity` 字段，它是 `PodSpec` 中 `affinity` 字段的子字段。

以下是一个使用Node Affinity的Pod的示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```

该Node Affinity规则表示，该Pod只能放在带有 `kubernetes.io/e2e-az-name` 的Label的Node上，其值为 `e2e-az1` 或 `e2e-az2` 。 另外，在符合该标准的Node中，优先使用具有 `another-node-label-key` ，值为`another-node-label-value` 的Node。

在本例中可以看到 `In` 操作符。新Node Affinity语法支持以下操作符： `In` 、 `NotIn` 、 `Exists` 、`DoesNotExist` 、 `Gt` 、 `Lt` 。没有明确的“node anti-affinity”概念，但 `NotIn` 和 `DoesNotExist` 提供了这种行为。

如果同时指定 `nodeSelector` 和 `nodeAffinity` ， 则必须同时满足，才能将Pod调度到候选Node上。

如果指定与 `nodeAffinity` 类型相关联的多个 `nodeSelectorTerms` ，那么**如果**满足 `nodeSelectorTerms` **之一** ，即可将Pod调度到节点上。

如果指定与 `matchExpressions` 相关联的多个 `matchExpressions` ，则**只有**满足**所有**  `matchExpressions` 才能将该Pod调度到Node上。

如果删除或更改了调度Pod的Node的Label，则该Pod不会被删除。 换句话说，Affinity选择仅在调度Pod时起作用。

有关Node Affinity的更多信息，请参见 [设计文档](https://git.k8s.io/community/contributors/design-proposals/scheduling/nodeaffinity.md) 。







### Inter-pod affinity and anti-affinity (beta feature)（Pod内的亲和性与反亲和性（Beta特性））

Kubernetes 1.4引入了Inter-pod affinity 和 anti-affinity。Inter-pod affinity和anti-affinity允许您根据已在Node上运行的Pod 上的Label，而非基于Node的Label来约束您的Pod能被调度到哪些Node。规则的形式是“如果X已经运行了一个或多个满足规则Y的Pod，则该Pod应该（或者在anti-affinity的情况下不应该）运行在X中”。Y表示一个与Namespace列表相关联（或“所有”命名空间）的LabelSelector；与Node不同，因为Pod是在Namespace中的（因此Pod上的Label是隐含Namespace的），Pod Label上的Label Selector必须指定选择器要应用的Namespace。 概念上，X是一个拓扑域，如Node、机架、云提供商Zone、云提供商Region等。您可以使用 `topologyKey` 表示它，该key是系统用来表示拓扑域的Node标签，例如参见上面 [Interlude: built-in node labels](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#interlude-built-in-node-labels) 中列出的键。

与Node Affinity一样，目前有两种类型的pod affinity和anti-affinity，称为 `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution` ，表示“hard”对“soft”需求。请参阅上文Node Affinity部分中的说明。一个 `requiredDuringSchedulingIgnoredDuringExecution` 的例子是“将同一个Zone中的Service A和Service B的Pod放到一起，因为它们彼此通信很多”，而 `preferredDuringSchedulingIgnoredDuringExecution` anti-affinity表示“将该Service的Pod跨Zone“（硬性要求没有意义，因为你可能有比Zone更多的Pod）。

Inter-pod affinity用 `podAffinity` 字段指定，它是PodSpec中 `affinity` 字段的子字段。 inter-pod anti-affinity用 `podAntiAffinity` 指定，它是 `affinity` 字段的子字段。



#### An example of a pod that uses pod affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: gcr.io/google_containers/pause:2.0
```

本例中，Pod的Affinity定义了一个Pod Affinity规则和一个Pod anti-affinity规则。 在此示例中， `podAffinity` 在是`requiredDuringSchedulingIgnoredDuringExecution` ，而`podAntiAffinity` 是`preferredDuringSchedulingIgnoredDuringExecution` 。Pod Affinity规则表示，只有当相同Zone中的某个Node至少有一个已经运行的、具有key=security、value=S1的Label的Pod时，该Pod才能调度到Node上。 （更准确地说，Pod会运行在这样的Node N上：Node N具有带有 `failure-domain.beta.kubernetes.io/zone` 的Label key和某些value V，即：集群中至少有一个带有key= `failure-domain.beta.kubernetes.io/zone` 以及value=V的Node，其上运行了key=security并且value=S1标签的Pod）。pod anti-affinity规则表示该Pod更倾向于不往那些已经运行着Label key=security且value=S2的Pod的Node上调度。（如果 `topologyKey` 是 `failure-domain.beta.kubernetes.io/zone` ，那么这意味着如果相同Zone中的Node中有Pod的key=security并且value=S2，则该Pod不能调度到该Node上“）。对于pod affinity以及anti-affinity的更多例子，详见 [design doc](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md) 。

pod affinity and anti-affinity的合法操作符是 `In` 、 `NotIn` 、 `Exists` 、 `DoesNotExist` 。

原则上， `topologyKey` 可以是任何合法的标签key。然而，出于性能和安全的原因，对于topologyKey有一些限制：

1. 对于Affinity以及 `RequiredDuringScheduling` pod anti-affinity,，不允许使用空（empty）的 `topologyKey` 。
2. 对于 `RequiredDuringScheduling` pod anti-affinity，引入了admission controller `LimitPodHardAntiAffinityTopology`  ，从而限制到 `kubernetes.io/hostname` 的 `topologyKey` 。如果要使其可用于自定义拓扑，您可以修改admission controller，或者简单地禁用它。
3. 对于 `PreferredDuringScheduling` pod anti-affinity，空（empty） `topologyKey` `kubernetes.io/hostname`被解释为“所有拓扑”（“所有拓扑”在这里仅限于 `kubernetes.io/hostname` 和`failure-domain.beta.kubernetes.io/region` 的组合）。
4. 除上述情况外， `topologyKey` 可以是任何合法的标签key。

除 `labelSelector` 和 `topologyKey` 外，还可以选择指定 `labelSelector` 应该匹配的Namespace列表（这与 `labelSelector` 和 `topologyKey` 的定义相同）。如果省略，默认为：定义affinity/anti-affinity了的Pod的Namespace。 如果定义为空（empty），则表示“所有Namespace”。

所有与`requiredDuringSchedulingIgnoredDuringExecution` affinity以及anti-affinity相关联的 `matchExpressions` 必须满足，才会将Pod调度到Node上。





#### More Practical Use-cases（更多实用的用例）

Interpod Affinity和AnitAffinity在与更高级别的集合（例如ReplicaSets、Statefulsets、Deployments等）一起使用时可能更为有用。可轻松配置工作负载应当统统位于同一个拓扑中，例如，同一个Node。



##### Always co-located in the same node（始终位于同一个Node）

在一个有3个Node的集群中，web应用有诸如redis的缓存。我们希望web服务器尽可能地与缓存共存。这是一个简单的redis Deployment的yaml片段，包含3个副本和选择器标签 `app=store` 。

```yaml
apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

在webserver Deployment的yaml代码片段下面配置了 `podAffinity` ，它通知scheduler其所有副本将与具有选择器标签 `app=store` 的Pod共同定位

```yaml
apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
```

如果我们创建上述两个Deployment，我们的三节点集群可能如下所示。

| node-1        | node-2        | node-3        |
| ------------- | ------------- | ------------- |
| *webserver-1* | *webserver-2* | *webserver-3* |
| *cache-1*     | *cache-2*     | *cache-3*     |

如您所见， `web-server` 3个副本将按预期自动与缓存共同定位。

```shell
$kubectl get pods -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
redis-cache-1450370735-6dzlj   1/1       Running   0          8m        10.192.4.2   kube-node-3
redis-cache-1450370735-j2j96   1/1       Running   0          8m        10.192.2.2   kube-node-1
redis-cache-1450370735-z73mh   1/1       Running   0          8m        10.192.3.1   kube-node-2
web-server-1287567482-5d4dz    1/1       Running   0          7m        10.192.2.3   kube-node-1
web-server-1287567482-6f7v5    1/1       Running   0          7m        10.192.4.3   kube-node-3
web-server-1287567482-s330j    1/1       Running   0          7m        10.192.3.2   kube-node-2
```

最佳实践是配置这些高可用的有状态工作负载，例如有AntiAffinity规则的redis，以保证更好的扩展，我们将在下一节中看到。



##### Never co-located in the same node（永远不位于同一个Node）

高可用数据库StatefulSet有一主三从，可能不希望数据库实例都位于同一Node中。

| node-1      | node-2         | node-3         | node-4         |
| ----------- | -------------- | -------------- | -------------- |
| *DB-MASTER* | *DB-REPLICA-1* | *DB-REPLICA-2* | *DB-REPLICA-3* |

[Here](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure) 是一个Zookeper StatefulSet的例子，为高可用配置了anti-affinity的。

有关inter-pod affinity/anti-affinity的更多信息，请参阅 [here](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md) 的设计文档。

您也可以检查 [Taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) ，这样可让Node排斥一组Pod。





## 原文

<https://kubernetes.io/docs/concepts/configuration/assign-pod-node/>





## 参考文档

<http://www.php230.com/1491134522.html>