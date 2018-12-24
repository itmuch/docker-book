# Taints and Tolerations

> 译者按：
>
> * Taints：在本文中根据上下文，有的地方直接叫Taints，有的地方翻译成“污点”或者“污染”。
> * Tolerations：在本文中根据上下文，有的地方直接叫Tolerations，有的地方翻译成“容忍”或“容忍度”。

 [here](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature) 描述的节点亲和性是一个Pod属性，使用它可将Pod吸引到一组Node（作为优选或硬性要求）。Taint功能相反，它允许Node排斥一组Pod。

Taints和Tolerations一起工作，以确保Pod不被安排在不适当的Node上。一个或多个Taints应用于Node；这标志着Node不应该接受任何不能容忍这些Taints的Pod。 Tolerations应用于Pod，并允许（但不要求）Pod可以调度到具有匹配Taints的Node上。





## Concepts（概念）

您可以使用 [kubectl taint](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#taint) 向节点添加Taint。例如，

```
kubectl taint nodes node1 key=value:NoSchedule
```

在 `node1` 上设置了一个Taint。Taint具有kay、value以及污点效果`NoSchedule` 。这意味着没有Pod能够调度到`node1` ，除非它具有匹配的Toleration。可在PodSpec中为Pod指定Toleration。以下两种Toleration都与上述`kubectl taint` 所造成的Taint“匹配”，因此这些Pod将能够调度到`node1` 上：

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

如果key相同，并且效果相同，那么Toleration匹配Taint，并且：

- `operator` 为 `Exists` （在这种情况下不应指定`value` ），或
- `operator` 为 `Equal` 并且 `value` 相等

如不指定，则`Operator`默认为 `Equal` 。

**注意：**有两种特殊情况：

-  `key` 留空配合操作符 `Exists` 可匹配所有key、value和效果，这意味着容忍一切。

  ```yaml
  tolerations:
  - operator: "Exists"
  ```


-  `effect` 留空匹配所有带有该`key` 效果。

  ```yaml
  tolerations:
  - key: "key"
    operator: "Exists"
  ```

上述示例使用`NoSchedule` 这个  `effect` 。或者，可使用`PreferNoSchedule` 这个 `effect` 。它是 `NoSchedule`的“偏好”或“软”版本——系统将**尽量**避免放置无法容忍Node上Taint的Pod，但不是必需的。 第三种`effect` 是 `NoExecute` ，稍后描述。

您可以在同一个Node上设置多个Taint，并在同一个Pod上设置多个Toleration。Kubernetes处理多个Taint和Toleration的方式就像过滤器：从Node的所有Taint开始，然后忽略具有匹配Toleration的那些Pod；剩下的不被忽视的Taints对Pod有明显影响。 尤其是，

- 如果至少有一个无法忽视的Taint，效果为 `NoSchedule` ，那么Kubernetes不会将Pod调度到该Node上；
- 如果没有不受忽视的Taint，效果为 `NoSchedule` ；但至少有一个无法忽视的Taint，效果为 `PreferNoSchedule` ，那么Kubernetes会将*尝试*不将Pod调度到该Node上；
- 如果至少有一个不受忽视的污点，效果为`NoExecute` ，则该Pod将从Node中驱逐（如果已经在Node上运行），并且不会被调度到该Node上（如果尚未运行在该Node上）。

例如，假设你污染了这样一个Node：

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

一个Pod有两个Tolerations：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

在本例中，由于没有符合第三个Taint的Toleration，该Pod将无法调度到`node1` 上。但如果在添加Taint时已经在该Node上运行，则能继续运行。

通常情况下，如果一个Node添加了一个带有 `NoExecute` 效果的Taint，那么任何不能Toleration该Taint的Pod将立即被驱逐，任何Toleration该Taint的Pod都不会被驱逐。但是，使用 `NoExecute` 效果的Toleration可指定一个可选的 `tolerationSeconds` 字段，该字段表示在添加Taint后，Pod停驻在该Node的时间。例如，

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

意味着如果该Pod正在运行，并且Taint被添加到Node，则Pod将在该Node上停驻3600秒，然后被驱逐。 如果在该时间内删除了Taint，则Pod不会被驱逐。

> 译者按：`tolerationSeconds` 可以理解成：容忍多少秒，过了这么多秒后，就不容忍了，然后就会被驱逐。





## Example Use Cases（使用案例示例）

Taints和Tolerations是一种能让将Pod”远离“Node或驱逐不应该运行的Pod的方式，该方式非常灵活。一些用例是

- **Dedicated Nodes（专用Node）** ：如果要将一组Node专用于一组特定的用户，您可以向这些Node添加一个污点（例如， `kubectl taint nodes nodename dedicated=groupName:NoSchedule` ），然后将相应的Toleration添加到它们的Pod（这可以通过编写自定义 [admission controller](https://kubernetes.io/docs/admin/admission-controllers/) 来轻松完成）。然后具有Tolerations的Pod将被允许使用污染（专用）Node以及集群中的任何其他Node。如果要将Node专用于它们， 并确保它们**只使用**专用Node，那么您应该额外添加标签，这些标签类似于同一组Node（例如`dedicated=groupName` ）上的Taint，并且Admission Controller应该另外添加一个Node的亲和性要求，Pod只能调度到标有 `dedicated=groupName` 的Node上。


- **Nodes with Special Hardware（具有特殊硬件的Node）** ：在一小部分Node具有专用硬件（例如GPU）的集群中，希望将不需要专用硬件的Pod原理这些Node，从而为需要使用专用硬件的Pod留出空间。这可以通过污染具有专门硬件的Node（例如， `kubectl taint nodes nodename special=true:NoSchedule` 或 `kubectl taint nodes nodename special=true:PreferNoSchedule` ）来完成，并将相应的Toleration添加到需使用特殊硬件的Pod。在专用Node的用例中，使用自定义 [admission controller](https://kubernetes.io/docs/admin/admission-controllers/) 应用Tolerations可能是最简单的）。例如，Admission Controller可使用Pod的一些特性来确定应该允许该Pod使用特殊Node。 为确保需要特殊硬件的Pod*只*被调度到具有特殊硬件的Node上，您将需要一些额外的机制，例如您可以使用  [opaque integer resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#opaque-integer-resources-alpha-feature) 来表示特殊资源，并将其作为PodSpec中的资源最小需求，或您可以标记具有特殊硬件的Node，并在需要硬件的Pod上使用Node Affinity。
- **Taint based Evictions (alpha feature)（基于Taint的驱逐（alpha功能））** ：当Node出现问题时，per-pod-configurable的驱逐行为，这将在下一节中描述。





## Taint based Evictions（基于Taint的驱逐）

之前我们提到了`NoExecute` taint的效果，它会影响已经在Node上运行的Pod，如下所示

- 不能容忍污点的Pod被立即驱逐
- 容忍污点的Pod，而不指定`tolerationSeconds` 将永远被绑定
- 容许污点的Pod，指定`tolerationSeconds` ，在指定的时间内保持绑定

上述行为是一个beta功能。 此外，Kubernetes 1.6具有表示Node问题的alpha支持。换句话说，当某些条件为真时，Node Controller会自动给Node添加Taint。 目前内置的Taint包括：

- `node.kubernetes.io/not-ready` ：Node尚未准备就绪。 这对应于NodeCondition `Ready` 字段为 `False` 。
- `node.alpha.kubernetes.io/unreachable` ：Node无法从Node Controller访问。这对应于NodeCondition `Ready` 字段为 `Unknown` 。
- `node.kubernetes.io/out-of-disk` ：Node磁盘不可用。
- `node.kubernetes.io/memory-pressure` ：Node内存有压力。
- `node.kubernetes.io/disk-pressure` ：Node磁盘有压力。
- `node.kubernetes.io/network-unavailable` ：Node的网络不可用。
- `node.cloudprovider.kubernetes.io/uninitialized` ：当kubelet以外部cloud provider启动时，它会为Node设置一个Taint，将其标记为未使用。当来自cloud-controller-manager的Controller初始化此Node时，kubelet将删除此Taint。

当启用`TaintBasedEvictions` alpha功能时（您可以通过在Kubernetes controller manager的`--feature-gates` 包含 `TaintBasedEvictions=true`来实现此功能，例如 `--feature-gates=FooBar=true,TaintBasedEvictions=true` ），这样，NodeController（或kubelet）将会自动添加Taint，并且基于Ready NodeCondition从Node驱出Pod的逻辑将会被禁用。（注意：为保持由于Node问题而导致的现有 [rate limiting](https://kubernetes.io/docs/concepts/architecture/nodes/) 行为，系统实际上以rate-limited的方式添加Taints，从而防止在Master节点在发生网络分区故障的场景中有大量的Pod被驱逐。 该alpha特征与`tolerationSeconds`相结合，从而允许Pod指定应该保持绑定到具有这些问题中的Node的时长。

例如，具有很多本地状态的应用可能希望在网络分区故障发生的情况下长时间保持绑定到Node，希望分区恢复时避免该驱逐。 在这种情况下，Pod将使用的Toleration情况看起来像：

```yaml
tolerations:
- key: "node.alpha.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

请注意，除非用户提供的Pod配置已经具有 `node.kubernetes.io/not-ready` 的Toleration，否则Kubernetes会自动为 `node.kubernetes.io/not-ready` 添加 `tolerationSeconds=300` 的Toleration。 同样地，对于`node.alpha.kubernetes.io/unreachable` ，也会有 `tolerationSeconds=300` 用户自行设置。

这些自动添加的Toleration确保在检测到这些问题之一后默认保持绑定5分钟。是 [DefaultTolerationSeconds admission controller](https://git.k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds) 添加了这两个默认Toleration。

对于以下没有未设置`tolerationSeconds` 的Taint， [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) Pod是通过 `NoExecute` Toleration来创建的：

- `node.alpha.kubernetes.io/unreachable`
- `node.kubernetes.io/not-ready`

这确保了DaemonSet Pod对这些问题容忍而永远不会被驱逐，这与禁用此功能时的行为相匹配。





## Taint Nodes by Condition（使用Condition为Node添加Taint）

1.8版引入了一个Alpha功能，导致Node Controller创建与Node状态相对应的Taint。启用此功能时（可以通过在Scheduler的 `--feature-gates` 命令中添加 `TaintNodesByCondition=true` 来启用该功能，例如 `--feature-gates=FooBar=true,TaintNodesByCondition=true` ），Scheduler不会检查Node的Condition，而是检查Taint。这样可确保Node Condition不会影响调度到该Node上的内容。用户可通过添加适当的Pod Toleration来选择忽略Node的一些问题（在Node Condition中显示）。

为了确保打开此功能不会破坏DaemonSet的特性，从1.8版本开始，DaemonSet Controller会自动将以下 `NoSchedule` 的Toleration添加到所有daemon中：

- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/out-of-disk` (*only for critical pods*)

上述设置确保向后兼容性，但是我们了解，它们可能无法适应所有用户的需求，这就是为什么集群管理员可能需要向DaemonSet添加Arbitrary Toleration的原因。





## 原文

<https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/>

