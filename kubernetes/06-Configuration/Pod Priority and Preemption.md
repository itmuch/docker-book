# Pod Priority and Preemption（Pod优先级和抢占）

> 本节把priority翻译成优先级，Preemption翻译成抢占。



**特性状态：** `Kubernetes v1.8` alpha

在Kubernetes 1.8或更高版本中，[Pods](https://kubernetes.io/docs/user-guide/pods) 有priority的概念。priority表示某个Pod相对于其他Pod的重要性。当一个Pod不能被调度时，Scheduler试图抢占（驱逐）较低priority的Pod，从而使调度处于pending状态的Pod成为可能。 在未来的Kubernetes版本中，priority还将影响Node上资源的驱逐排序。

**注意：**抢占不遵守PodDisruptionBudget；有关详细信息，请参阅 [the limitations section](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#poddisruptionbudget-is-not-supported) 。





## How to use priority and preemption（如何使用优先级和抢占）

要在Kubernetes 1.8中使用priority和preemption，请按照如下步骤操作：

1. 启用该功能。
2. 添加一个或多个PriorityClasses。
3. 创建Pod，并将 `PriorityClassName` 设为你所添加的PriorityClasses之一。当然，您无需直接创建Pod；通常可将 `PriorityClassName` 添加到集合对象的Pod模板（如Deployment）。

以下部分提供有关这些步骤的更多信息。







## Enabling priority and preemption（启用优先级和抢占）

默认情况下，Kubernetes 1.8中的Pod priority和preemption功能是禁用的。要启用该功能，请为API Server和Scheduler设置此命令行标志：

```shell
--feature-gates=PodPriority=true
```

并为API Server设置此标志：

```shell
--runtime-config=scheduling.k8s.io/v1alpha1=true
```

启用该功能后，您可以创建 [PriorityClasses](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass) 并创建具有 [`PriorityClassName`](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#pod-priority) 设置的Pods。

如果您尝试过该功能后，决定禁用它，那么您必须删除PodPriority命令行标志或将其设置为false，然后重新启动API Server和Scheduler。禁用该功能后，现有的Pod将保留其priority字段，但禁用preemption，priority字段将被忽略；并且，您不能在新Pod中设置PriorityClassName。





## PriorityClass

PriorityClass是一个non-namespaced对象，它定义了从priority class到priority整数值的映射。该名称在PriorityClass对象元数据的 `name` 字段中指定。该值在必需的 `value` 字段中指定。value值越大，优先级越高。

PriorityClass对象可以有小于或等于10亿的、任意32位整数值。较大的数字保留给关键系统Pod，这些Pod通常不应该被抢占或被驱逐。 集群管理员应为每个映射创建一个PriorityClass对象。

PriorityClass还有两个可选字段： `globalDefault` 和 `description` 。对于未设置 `PriorityClassName` 的Pod， `globalDefault` 字段表示该PriorityClass的值。只有一个 `globalDefault=true` 的PriorityClass能够存在于系统中。如果没有设置了 `globalDefault` 的PriorityClass，那么，未设置 `PriorityClassName` 的Pod的优先级为零。

`description` 字段是一个任意字符串。这是为了告诉集群用户，他们什么时候该使用这个PriorityClass。

**注1** ：如果升级现有集群并启用此功能，则现有Pod的优先级为零。

**注2** ：为Pod动态添加 `globalDefault=true` 的PriorityClass，不会更改现有Pod的优先级。此类PriorityClass的值仅用于添加PriorityClass后创建的Pod。

> TODO 注2 原文有点歧义，待测试、改进。

**注3** ：如果您删除了PriorityClass，则引用该PriorityClass名称的现有Pod将保持不变，但您无法继续创建引用该PriorityClass名称的Pod。



### Example PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1alpha1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```





## Pod priority

创建一个或多个PriorityClass之后，可创建Pod，并在其spec中指定其中一个PriorityClass名称。priority admission controller使用 `priorityClassName` 字段并填充该字段的整数值。如果未找到priority class，则Pod被拒绝。

如下YAML是一个Pod的定义，该Pod使用上述示例中所创建的PriorityClass。priority admission controller检查spec，并将Pod的priority设为1000000。

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
  priorityClassName: high-priority
```





## Preemption（抢占）

当Pods被创建时，它们会进入队列并等待被调度。Scheduler从队列中选择一个Pod，并尝试将其调度到Node上。 如果未能找到满足Pod所有要求的Node，则为处于pending状态的Pod触发preemption逻辑。下面我们称这个处于pending状态的Pod为P。抢占逻辑尝试找到一个Node，在删除一个或多个优先级低于P的Pod后，就能在该Node上调度P。 如果能找到这样的Node，则会从Node中删除一个或多个优先级较低的Pod。 在其删除后，P就能在Node上调度。



### Limitations of preemption (alpha version)

#### Starvation of preempting Pod

当Pod被抢占时，受害者获得了 [graceful termination period](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods) 。他们有graceful termination period所定义的时长完成工作并退出。如果无法优雅关闭，就会被杀死。这个graceful termination period会在   scheduler抢占Pod的时间点 与 能在Node（N）上调度pending Pod（P）的时间点 之间创建时间间隔。 在此期间，Scheduler会调度其他pending的Pod。当受害者退出或终止时，Scheduler会尝试调度在待处理队列中的Pod，并且在Scheduler将P调度到N之前，可能会将其中的一个或多个考虑并调度到N。在这种情况下，所有的受害者退出，Pod P不再适用于Node N。因此，Scheduler将必须抢占Node N或其他Node上的其他Pod，以便能够调度P。对于第二次和后续的抢占，这种情况可能会再次重复，P可能会在一定时间内无法得到调度。这种情况可能会导致各种集群中的问题，在Pod创建速率较高的集群中尤其成问题。

我们将在Pod preemption的Beta版本中解决这个问题。  [provided here（在这里提供了）](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/pod-preemption.md#preemption-mechanics) 我们计划实施的解决方案。

#### PodDisruptionBudget is not supported（不支持PodDisruptionBudget）

 [Pod Disruption Budget (PDB)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) 允许应用程序所有者限制从voluntary disruptions中资源下降的Pod的数量。

然而，在选择preemption的受害者时，alpha版本的preemption并不遵守PDB规则。我们计划在beta版本中添加PDB支持，但即使在beta版中，也只是尽可能地遵守PDB。Scheduler将尝试寻找那些 PDB不会被preemption违反的 受害者，但如果未发现这样的受害者，preemption仍然会发生，优先级较低的Pod将被删除，尽管这会违反PDB。

> 译者按：PDB简介：<http://blog.csdn.net/horsefoot/article/details/76496496>

#### Inter-Pod affinity on lower-priority Pods（低优先级Pod的Inter-Pod 亲和性）

在版本1.8中，只有当该问题的答案为yes时，才将Node视为可抢占：“如果从Node中删除所有优先级低于pending Pod的Pod，就能在该Node上调度pending Pod吗？

**注意：**抢占并不一定会清除所有优先级较低的Pod。如果删除部分优先级较低的Pod就能调度pending Pod，那么只有一部分优先级较低的Pod会被删除。即使如此，上述问题的答案也必须是yes。否则，则Node不被视为可抢占。

如果pending Pod具有与 Node上一个或多个较低优先级Pod 的inter-pod affinity，则在那些较低优先级Pod缺失的情况下，不能满足 inter-Pod affinity规则。在这种情况下，Scheduler不会抢占Node上的任何Pod。相反，它寻找另一个Node。 Scheduler可能会找到一个合适的Node，又或者它找不到合适的Node。不能保证pending Pod能够被调度。

我们可能会在将来的版本中解决这个问题，但还没有一个明确的计划。我们不会认为它是Beta或GA的阻止者。部分原因是找到满足所有inter-Pod affinity规则的较低优先级的Pod集合，计算代价比较高，并且对抢占逻辑增加了相当大的复杂性。此外，即使preemption保留优先级较低的Pod来满足inter-Pod affinity，较低优先级的Pod也可能被其他Pod稍后抢占，这消除了遵守inter-Pod affinity的复杂逻辑的好处。

我们针对此问题的建议解决方案是仅在相同或更高优先级的Pod上创建inter-Pod affinity。

#### Cross node preemption（跨Node抢占）

假设Node N正被考虑用于抢占，以便可在N上调度pending Pod P。只有当另一个Node上的Pod被抢占时，P才可能在N上可行。这里有一个例子：

- Node N正在考虑Pod P。
- Pod Q在与Node N相同zone中的另一个Node上运行。
- Pod P与Pod Q具有anit-affinity。
- Pod P和其他Pod之间没有anti-affinity的情况。
- 为在Node N上安排Pod P，Pod Q应该被抢占，但是Scheduler不执行跨节点抢占。 所以，Pod P在Node N上被视为unschedulable的。

如果Pod Q从其Node中删除，那么久不违反anti-affinity了，并且Pod P可能会在Node N上进行调度。

如果我们找到某种性能合理的算法，可能会考虑在将来的版本中添加跨Node抢占。在这一点上我们不能作任何承诺，并且跨Node抢占不会被认为是Beta或GA的阻止者。





## 原文

<https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/>

