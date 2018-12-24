# StatefulSet

> StatefulSet有两个维度的抽象：
>
> * 拓扑状态：有时候多个实例之间并非对等关系，可能需要按顺序启动，又或者，某个Pod挂掉后，新创建的Pod必须和原先Pod的网络标识一样，这样，原先的访问者才可能使用访问原先Pod的方法，访问到新Pod。
> * 存储状态：即使Pod挂掉被重建，访问的存储应该是同一份。
> * 本质：StatefulSet是一种特殊的Deployment，只不过这个Deployment的Pod名称+编号固定，编号的顺序固定了Pod之间的拓扑关系；而编号对应的Persist Volume，绑定了Pod与持久化存储之间的关系。因此，Pod即使被重建，状态也不会改变。·

**StatefulSet是用于管理有状态应用的工作负载API对象。StatefulSet在1.8中是beta版本。**

管理Deployment以及一组 [Pods](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#term-pod) 的伸缩，并为这些Pod的排序和唯一性提供保证。

像 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#term-deployment) 一样，StatefulSet管理基于相同容器spec的Pod。与Deployment不同，StatefulSet会为每个Pod保留粘性身份。这些Pod是从相同的spec创建的，但不可互换：每个Pod都有一个持久化的标识符，即使重新调度也仍然会使用该标识符。

StatefulSet与任何其他Controller的模式相同。您可在StatefulSet对象中定义期望状态，StatefulSet Controller会从当前状态进行必要的更新，从而达到期望状态。





## Using StatefulSets（使用StatefulSet）

StatefulSet适用于以下场景：

- 稳定、唯一的网络标识。
- 稳定、持久化的存储。
- 有序、优雅的部署和缩放（即：部署有顺序、扩展也有顺序）。
- 有序、优雅的删除和终止。
- 有序、自动的滚动更新。

在上文中，“稳定”是Pod调度（重新调度）持久化的代名词。如果应用程序不需要任何稳定的标识符或有序部署、删除或缩放，则应该使用提供无状态副本的Controller部署应用。 诸如 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 或者 [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 可能更适合您的无状态需求。





## Limitations（限制）

- StatefulSet是一个beta资源（处于Beta阶段的资源），在1.5之前的Kubernetes版本中不可用。
- 与所有其他alpha/beta资源一样，可传给apiserver一个`--runtime-config` 选项来禁用StatefulSet。
- 给定Pod的存储必须由 [PersistentVolume Provisioner](https://github.com/kubernetes/examples/tree/master/staging/persistent-volume-provisioning/README.md) 根据请求的 `storage class` 提供，或由管理员预先设置。
- 删除/缩容StatefulSet将*不会*删除与StatefulSet关联的Volume。这样做是为了确保数据安全性，这通常比自动清除StatefulSet所有相关资源更有价值。
- StatefulSet目前需要一个 [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 负责Pod的网络身份。您有责任创建此Service。





## Components（组件）

下面的示例演示了StatefulSet的组件。

- 名为nginx的Headless Service，用于控制网络域名。
- 名为web的StatefulSet，它有一个Spec，表示有3个nginx副本。
- volumeClaimTemplates将使用PersistentVolume Provisioner提供的 [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/volumes/) ，从而提供稳定的存储。

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: task-pv-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: task-pv-storage
        persistentVolumeClaim:
          claimName: task-pv-claim
```





## Pod Selector（Pod选择器）

您必须设置StatefulSet的`spec.selector` 字段以匹配其`.spec.template.metadata.labels` 的Label。在Kubernetes 1.8之前， `spec.selector` 字段被默认为省略。 在1.8及更高版本中，如未指定匹配的Pod Selector将会导致在StatefulSet创建过程中验证错误。





## Pod Identity（Pod身份）

StatefulSet Pod有唯一的身份，包括序数（ordinal）、稳定的网络标识和稳定的存储。 身份会绑定到Pod（具有粘性），不管Pod被调度到哪个Node上。



### Ordinal Index（有序的索引）

对于一个有N个副本的StatefulSet，StatefulSet中的每个Pod将被分配一个整数序号，范围[0，N），并且唯一。



### Stable Network ID（稳定的网络ID）

StatefulSet中的每个Pod，从StatefulSet的名称和Pod的序数派生其主机名。 构造的主机名的模式是 `$(statefulset name)-$(ordinal)` 。上面的例子中，将会创建名为`web-0,web-1,web-2` 的Pod。StatefulSet可以使用 [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 来控制其Pod的域名。此Service管理的域名的格式为： `$(service name).$(namespace).svc.cluster.local` ，其中“cluster.local”是集群域名。在创建每个Pod时，它将会获得一个匹配的DNS子域，采用以下形式： `$(podname).$(governing service domain)` ，其中governing service由StatefulSet上的`serviceName`字段定义。

以下是Cluster Domain、Service名称、StatefulSet名称以及如何影响StatefulSet的Pod的DNS名称的一些示例。

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                  | Pod Hostname |
| -------------- | ----------------- | --------------------- | ------------------------------- | ---------------------------------------- | ------------ |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local | web-{0..N-1} |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local    | web-{0..N-1} |

请注意，除非 [otherwise configured](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#how-it-works) ，Cluster Domain将被设为`cluster.local` 。



### Stable Storage（稳定的存储）

Kubernetes为每个VolumeClaimTemplate创建一个 [PersistentVolume](https://kubernetes.io/docs/concepts/storage/volumes/) 。 在上面的nginx示例中，每个Pod将收到一个PersistentVolume，其中包含名为`my-storage-class` 的StorageClass和1 Gib的存储。如果未指定StorageClass，则将使用默认StorageClass。当一个Pod被重新调度到一个Node上时，它的`volumeMounts` 将挂载与其PersistentVolume Claims相关联的PersistentVolume。请注意，当Pod或StatefulSet被删除时，与Pod的PersistentVolume Claims相关联的PersistentVolumes不会被删除。这必须手动完成。





## Deployment and Scaling Guarantees（部署和缩放保证）

- 对于具有N个副本的StatefulSet，当部署Pod时，它们将按从{0..N-1}的顺序创建。
- 当Pod被删除时，它们按从{N-1..0}的顺序终止。
- 在将扩容操作应用于Pod之前，它的所有“前辈”必须Running and Ready。
- 在Pod终止之前，所有的“后辈”必须完全关闭。

StatefulSet不应该将 `pod.Spec.TerminationGracePeriodSeconds` 设为0。这种做法是不安全，强烈不建议您这么做。 有关进一步说明，请参阅 [force deleting StatefulSet Pods](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/) 。

当创建上述的nginx示例时，将以web-0、web-1、web-2的顺序部署三个Pod。在web-0正在 [Running and Ready](https://kubernetes.io/docs/user-guide/pod-states/) ，web-1将不被部署，而在web-1 Running and Ready之前web-2将不被部署。 如果web-0失败，在web-1 Running and Ready之后，但在启动web-2之前，web-2将不会启动，直到web-0成功重启并Running and Ready。

如果用户通过patch StatefulSet来scale部署的示例，例如设置`replicas=1` ，则web-2将首先被终止。在web-2完全关闭和删除之前，web-1不会被终止。如果web-0失败发生在web-2终止并完全关闭之后、web-1终止之前，web-1将不会终止，除非web-0已经Running and Ready。



### Pod Management Policies（Pod管理策略）

在Kubernetes 1.7及更高版本中，StatefulSet允许您放松其排序保证，同时通过`.spec.podManagementPolicy` 字段保留其唯一性和身份保证。

#### OrderedReady Pod Management（OrderedReady的Pod管理）

`OrderedReady` Pod管理是StatefulSet的默认值。 它实现了[上述](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#deployment-and-scaling-guarantees) 行为。

#### Parallel Pod Management（并行 Pod管理）

`Parallel` Pod管理告诉StatefulSet Controller 并行启动或终止所有Pod，并且不要等待Pod在启动或终止另一个Pod之前变为“Running”和“Ready”或完全终止。





## Update Strategies（更新策略）

在Kubernetes 1.7及更高版本中，StatefulSet的 `.spec.updateStrategy` 字段允许您配置和禁用StatefulSet中Pod的容器、标签、资源最小要求/最大限制、Annotation的滚动更新。



### On Delete

`OnDelete` 更新策略实现了遗留（1.6和以前）的行为。 当 `spec.updateStrategy` 未指定时，这是默认策略。当StatefulSet的 `.spec.updateStrategy.type` 设置为`OnDelete` ，StatefulSet Controller将不会自动更新StatefulSet中的Pod。用户必须手动删除Pod以使Controller创建新的Pod，以反映对StatefulSet的`.spec.template` 进行的修改。



### Rolling Updates

`RollingUpdate`更新策略为在StatefulSet中的Pod实现自动的滚动更新。当StatefulSet的`.spec.updateStrategy.type`设置为`RollingUpdate` 时，StatefulSet Controller将在StatefulSet中删除并重新创建每个Pod。它将按照Pod终止（从最大序号到最小）的顺序进行，每次更新一个Pod。 在更新其“前辈”之前，它将等待正在更新的Pod进入Running and Ready状态。

#### Partitions（分区）

可以通过指定 `.spec.updateStrategy.rollingUpdate.partition` 来对 `RollingUpdate` 更新策略进行分区。 如果指定了分区，则当StatefulSet的 `.spec.template` 更新时，具有大于或等于分区的序数的所有 `.spec.template` 被更新。具有小于分区的序数的所有Pods将不会更新，即使删除它们，也会使用以前的版本重新创建。 如果StatefulSet的 `.spec.updateStrategy.rollingUpdate.partition` 大于其 `.spec.replicas` ，则 `.spec.replicas` 更新将不会传播到其Pod。在大多数情况下，您不需要使用分区，但如果您想要进行阶段更新、金丝雀部署或执行分阶段发布，分区将非常有用。





## What’s next（下一步）

- Follow an example of [deploying a stateful application](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/).
- Follow an example of [deploying Cassandra with Stateful Sets](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/).





## 原文

<https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/> 



## 很赞的博客

在K8s中创建StatefulSet：<http://www.cnblogs.com/puyangsky/p/6677308.html>