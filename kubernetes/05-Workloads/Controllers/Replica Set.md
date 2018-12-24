# Replica Set

**ReplicaSet是下一代Replication Controller。***ReplicaSet*和 [*Replication Controller*](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 之间的唯一区别就是选择器支持。ReplicaSet支持 [labels user guide](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) 描述的新的set-based selector requirement，而Replication Controller仅支持equality-based selector requirement。

> 关于Label Selector：<https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/> ，本文的示例种也有用到两种selector。





## 如何使用ReplicaSet

支持Replication Controller的大多数 [`kubectl`](https://kubernetes.io/docs/user-guide/kubectl/) 命令也支持ReplicaSets。 **[`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update) 命令是一个例外。**如果您想要滚动更新功能，请考虑使用Deployment。 此外， [`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update) 命令是必不可少的，而Deployment是声明式的，因此我们建议您通过 [`rollout`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rollout) 命令使用Deployment。

虽然ReplicaSet可独立使用，但是目前它主要被 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 作为编排Pod创建、删除和更新的机制。当您使用Deployment时，您不必担心如何管理Deployment创建的ReplicaSet。Deployment拥有并管理其ReplicaSet。





## 何时使用ReplicaSet

ReplicaSet确保在任何时间都会运行指定数量的Pod副本。但是，Deployment是一个更高层次的概念——它管理ReplicaSet，并提供对Pod的声明性更新以及许多其他有用的功能。因此，我们建议您使用Deployment而不是直接使用ReplicaSet，除非您需要自定义更新编排或根本不需要更新。

这实际上意味着您可能永远不需要操作ReplicaSet对象：使用Deployment替代，并在spec部分中定义应用程序。





## 示例

```yaml
apiVersion: apps/v1beta2 # for versions before 1.6.0 use extensions/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 3
  selector:
  	# 下面的是equality-based selector requirement
    matchLabels:
      tier: frontend
    # 下面的是set-based selector requirement
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

将此清单保存为`frontend.yaml` ，并将其提交给Kubernetes集群，即可创建你所定义的ReplicaSet以及ReplicaSet管理的Pod。

```shell
$ kubectl create -f frontend.yaml
replicaset "frontend" created
$ kubectl describe rs/frontend
Name:		frontend
Namespace:	default
Selector:	tier=frontend,tier in (frontend)
Labels:		app=guestbook
		tier=frontend
Annotations:	<none>
Replicas:	3 current / 3 desired
Pods Status:	3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=guestbook
                tier=frontend
  Containers:
   php-redis:
    Image:      gcr.io/google_samples/gb-frontend:v3
    Port:       80/TCP
    Requests:
      cpu:      100m
      memory:   100Mi
    Environment:
      GET_HOSTS_FROM:   dns
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----                -------------    --------    ------            -------
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-qhloh
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-dnjpy
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-9si5l
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
frontend-9si5l   1/1       Running   0          1m
frontend-dnjpy   1/1       Running   0          1m
frontend-qhloh   1/1       Running   0          1m
```





## 编写ReplicaSet的spec

与所有其他Kubernetes API对象一样，ReplicaSet需要`apiVersion` 、 `kind`和`metadata` 等字段。关于使用清单的一般信息，请参阅 [here](https://kubernetes.io/docs/user-guide/simple-yaml/) 、 [here](https://kubernetes.io/docs/tasks/) 和 [here](https://kubernetes.io/docs/concepts/tools/kubectl/object-management-overview/) 。

ReplicaSet还需要一个  [`.spec` section](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) 。



### Pod模板

`.spec.template` 是`.spec` 唯一必需的字段。 `.spec.template`是一个 [pod template](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates) 。 它与 [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 有完全相同的模式——除了是嵌套的，没有`apiVersion` 以及`kind` 以外。

除Pod必需的字段外，ReplicaSet中的Pod模板必须指定适当的标签和适当的重新启动策略。

对于标签，请确保不会与其他Controller重叠。 有关更多信息，请参阅 [pod selector](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#pod-selector) 。

对于 [restart policy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) ， `.spec.template.spec.restartPolicy` 的唯一允许的值是`Always` ，这是默认值。

对于本地容器重新启动，ReplicaSet将委托给Node上的代理，例如 [Kubelet](https://kubernetes.io/docs/admin/kubelet/) 或Docker。



### Pod选择器

`.spec.selector` 字段是一个 [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 。 一个ReplicaSet管理所有与标签选择器相匹配的Pod。它不区分其创建或删除的Pod，也不区分另一个人或进程创建或删除的Pod。这允许我们在不影响正在运行的Pod的前提下替换ReplicaSet。

`.spec.template.metadata.labels` 必须与`.spec.selector`  匹配，否则将被API拒绝。

在Kubernetes 1.8中，ReplicaSet类型上当前的API版本是`apps/v1beta2` ，默认启用。API版本`extensions/v1beta1` 已被弃用。 在API版本`apps/v1beta2` 中，如果没有设置，则`.spec.selector`和`.metadata.labels` 默认不再与`.spec.template.metadata.labels` 一致。 因此，必须明确设定这些字段。 另外，在API版本`apps/v1beta2` 中，一旦ReplicaSet创建， `.spec.selector` 是不可变的。

此外，您通常不应创建任何标签与此选择器匹配的Pod，不管是直接创建、使用ReplicaSet或其他Controller（例如Deployment）。 如果这样做，ReplicaSet会认为它创建了其他Pod。Kubernetes并不会阻止你这样做。

如果您最终使用具有重叠选择器的多个Controller，则必须自行管理删除。



### ReplicaSet上的标签

ReplicaSet本身可以有标签（ `.metadata.labels` ）。 通常，您应该将这些该字段设置为与`.spec.template.metadata.labels` 相同。 但是，也允许不同，`.metadata.labels` 不会影响ReplicaSet的行为。



### Replicas（副本）

您可以通过设置`.spec.replicas` 来指定同时运行多少个Pod。任何时间运行的Pod个数都可能会更高或更低，例如，如果replica刚刚增加或减少；或者如果Pod优雅关闭，而替换提前启动。

如果不指定`.spec.replicas` ，则默认为1。





## 使用ReplicaSet

### 删除ReplicaSet和其Pod

可使用 [`kubectl delete`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#delete) 删除ReplicaSet及其所有pod。Kubectl将ReplicaSet缩放为零，并等待它删除每个Pod，然后再删除ReplicaSet本身。 如果这个kubectl命令被中断，可以重启。

当使用REST API或Go语言客户端库时，需要明确执行这些步骤（将副本缩放为0，等待Pod删除，然后删除ReplicaSet）。



### 只删除ReplicaSet

可以只删除ReplicaSet，而不影响ReplicaSet的任何Pod，使用 [`kubectl delete`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#delete) 和`--cascade=false` 选项即可。

使用REST API或Go语言客户端库时，只需删除ReplicaSet对象即可。

原始的ReplicaSet被删除后，您可以创建一个新的ReplicaSet来替换它。只要新旧ReplicaSet的`.spec.selector` 相同，那么新ReplicaSet就会使用旧ReplicaSet的Pod。然而，它不会努力使已存在的Pod去匹配一个新的、不同的Pod模板。要想以可控的方式将Pod更新为新的spec，请使用 [rolling update](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#rolling-updates) 。



### 从ReplicaSet隔离Pod

可以通过更改其标签的方式，从ReplicaSet中删除Pod。此技术可用于从Service中删除Pod，从而进行debug、数据恢复等。以这种方式删除的Pod将被自动替换（假设副本的数量也不会改变）。



### ReplicaSet伸缩

只需更新`.spec.replicas` 字段即可轻松缩放ReplicaSet。ReplicaSet controller会确保集群中有指定数量的Pod可用并可操作。



### ReplicaSet作为Horizontal Pod Autoscaler（HPA）目标

ReplicaSet也可以是 [Horizontal Pod Autoscalers (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 的目标。也就是说，ReplicaSet可以由HPA自动伸缩。以下是一个HPA指向我们在上一个示例中创建的ReplicaSet的示例。

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

将此清单保存为`hpa-rs.yaml` 并将其提交到Kubernetes集群，这样就会创建一个HPA，根据Pod副本的CPU使用率自动调整目标ReplicaSet。

```shell
 kubectl create -f hpa-rs.yaml 
```

或者，您可以使用`kubectl autoscale` 命令来完成相同的操作（并且更容易！）

```shell
 kubectl autoscale rs frontend 
```





## ReplicaSet的替代方案

### Deployment（推荐）

[`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 是一种更高级的API对象，它以与`kubectl rolling-update` 类似的方式，更新其底层的ReplicaSet及其Pod。如果您想要滚动更新的功能，则建议使用Deployment，因为与`kubectl rolling-update` 不同，它们是声明式、服务器端的，并且具有其他特性。有关使用Deployment运行无状态应用的更多信息，请阅读 [Run a Stateless Application Using a Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) 。



### Bare Pod（裸Pod）

与用户直接创建Pod的情况不同，ReplicaSet会替换由于任何原因被删除或终止的pod，例如在Node故障或Node中断维护（例如内核升级）的情况下。 因此，我们建议您使用ReplicaSet，即使您的应用程序只需要一个Pod。 与process supervisor（进程管理器）类似，ReplicaSet只能监视多个Node上的多个Pod，而不是单个Node上的单个进程。ReplicaSet将本地容器的重启委托给Node上的某个代理（例如，Kubelet或Docker）。



### Job（作业）

对于可预期会终止的Pod（即批处理作业），可以使用 [`Job`](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 而非ReplicaSet。



### DaemonSet

对于提供机器级功能（例如机器监控或日志）的Pod，请使用 [`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 而非ReplicaSet。 这些Pod的生命周期与机器的生命周期相关：在其他Pod启动之前，这些Pod需要在机器上运行；当机器准备重启/关闭时，可安全终止这些Pod。





## 原文

<https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/>