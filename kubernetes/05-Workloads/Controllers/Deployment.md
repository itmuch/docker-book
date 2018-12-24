# Deployment

> * RC只支持基于等式的selector（env=dev或environment!=qa），但Replica Set还支持新的，基于集合的selector（version in (v1.0, v2.0)或env notin (dev, qa)），这对复杂的运维管理很方便。
> * 使用Deployment升级Pod，只需要定义Pod的最终状态，k8s会为你执行必要的操作，虽然能够使用命令`# kubectl rolling-update`完成升级，但它是在客户端与服务端多次交互控制RC完成的，所以REST API中并没有rolling-update的接口，这为定制自己的管理系统带来了一些麻烦。
> * Deployment拥有更加灵活强大的升级、回滚功能。



*Deployment* Controller为 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 和 [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 提供声明式的更新。

只需在Deployment对象中描述*所期望的状态* ，Deployment Controller就会以受控的速率将实际状态逐步转变为你所期望的状态。您可以定义Deployment以创建新的ReplicaSet，也可删除现有Deployment并让新的Deployment采用其所有资源。

> **注意：**您不应该管理Deployment所拥有的ReplicaSet。而应该操作Deployment对象从而管理ReplicaSet。如果你认为有必须直接管理Deployment所拥有的ReplicaSet的场景，请考虑在Kubernetes repo中提Issue。





## 用例

以下是Deployment的典型用例：

- [Create a Deployment to rollout a ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment) （创建Deployment从而升级ReplicaSet）。 ReplicaSet在后台创建Pod。检查升级的状态，看是否成功。
- 通过更新Deployment的PodTemplateSpec来 [Declare the new state of the Pods](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment) （声明Pod的新状态）。 这样，会创建一个新的ReplicaSet，并且Deployment会以受控的速率，将Pod从旧ReplicaSet移到新的ReplicaSet。每个新的ReplicaSet都会更新Deployment的修订版本。
- 如果的Deployment当前状态不稳定，则 [Rollback to an earlier Deployment revision](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment) （回滚到之前的Deployment修订版本）。每次回滚都会更新Deployment的修订版本。
- [Scale up the Deployment to facilitate more load](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment) （扩展Deployment，以便更多的负载）


- [Pause the Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment) （暂停Deployment），从而将多个补丁应用于其PodTemplateSpec，然后恢复它，开始新的升级。
- [Use the status of the Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status) （使用Deployment的状态）作为升级卡住的指示器。
- 清理您不再需要的 [Clean up older ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#clean-up-policy) （清理旧的ReplicaSet）





## 创建Deployment

以下是Deployment的示例。它创建一个包含三个`nginx` Pod的ReplicaSet：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

在本例中：

- 将创建名为`nginx-deployment` 的Deployment，由`metadata: name` 字段定义。
- Deployment创建三个Pod副本，由`replicas` 字段定义。
- Pod模板的规范，即：`template: spec`字段定义Pod运行一个 `nginx` 容器，它运行1.7.9版的`nginx`  [Docker Hub](https://hub.docker.com/) 镜像。
- Deployment打开80端口供Pod使用。

`template` 字段包含以下说明：

- Pod被打上了`app: nginx` 的标签
- 创建一个名为`nginx` 的容器。
- 运行`nginx 1.7.9` 镜像。
- 打开端口`80` ，以便容器可以发送和接收流量。

要创建此Deployment，请运行以下命令：

```shell
kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes.github.io/master/docs/concepts/workloads/controllers/nginx-deployment.yaml
```

注意：您可以将`--record` 附加到此命令以在资源的annotation中记录当前命令。这对将来的审查（review）很有用，例如调查在每个Deployment修订版中执行了哪些命令。

接下来，运行`kubectl get deployments` ，将会输出类似如下内容：

```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

当您inspect集群中的“Deployment”时，将会显示以下字段：

- `NAME` 列出了集群中Deployment的名称。

- `DESIRED` 显示您创建Deployment时所定义的期望*副本*数。这是*所需的状态* 。

- `CURRENT` 显示当前正在运行的副本数。

- `UP-TO-DATE` 显示已更新，从而实现期望状态的副本数。

  - > 译者按：该字段常用于在滚动升级时，显示有多少个Pod副本已成功更新到最新版本。

- `AVAILABLE` 显示当前有可用副本的数量。

- `AGE` 显示应用程序已经运行了多久。

注意每个字段中的值如何对应于Deployment规范中的值：

- 根据`spec: replicas` 字段，期望的副本数量是3。
- 根据`.status.replicas` 字段，当前副本的数量为0。
- 根据`.status.updatedReplicas` 字段，最新副本的数量为0。
- 根据`.status.availableReplicas` 字段，可用副本的数量为0。

要查看Deployment升级的状态，请运行 `kubectl rollout status deployment/nginx-deployment` 。此命令将会返回类似如下的输出：

```
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

几秒钟后再次运行`kubectl get deployments` ：

```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           18s
```

由结果可知，Deployment已创建三个Pod副本，并且所有副本都是最新的（它们包含最新的Pod模板）并且可用（Pod状态为Ready的持续时间至少得达到`.spec.minReadySeconds` 字段定义的值）。

> 编者按：拓展阅读：<http://blog.csdn.net/WaltonWang/article/details/77461697>

要查看Deployment创建的ReplicaSet（ `rs` ），可运行`kubectl get rs` ：

```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-2035384211   3         3         3       18s
```

请注意，ReplicaSet的名称的格式始终为 `[DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]` 。创建Deployment时会自动生成该hash。

要查看为每个Pod自动生成的label，请运行 `kubectl get pods --show-labels` 。 可返回类似以下输出：

```
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-2035384211-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
```

ReplicaSet会确保在任何时候都有三个`nginx` Pod运行。

**注意：**您必须在Deployment中指定适当的选择器和Pod模板标签（在本例中为`app: nginx` ）。不要与其他Controller（包括其他Deployment和StatefulSet）所使用的标签或选择器重叠。Kubernetes不会阻止您重叠，如果多个Controller选择器发生重叠，那么这些Controller可能会出现冲突并且出现意外。



### Pod-template-hash标签

**注意**：请勿修改此标签。

`pod-template-hash label`由Deployment Controller添加到其创建或采用的每个ReplicaSet上。

此标签可确保Deployment的子ReplicaSet不重叠。 它通过将ReplicaSet的`PodTemplate` 进行hash，并将生成的hash值作为标签，添加到ReplicaSet选择器、Pod模板标签、ReplicaSet所拥有的任何现有Pod中。





## 升级Deployment

**注意：**当且仅当Deployment的Pod模板（即`.spec.template` ）发生变化时，Deployment才会发生升级。例如，如果模板的标签或容器镜像被更新，则会触发Deployment的更新。 其他更新，例如对Deployment伸缩，不会触发更新。



假设我们现在想要升级nginx Pod，让其使用`nginx:1.9.1` 镜像，而非`nginx:1.7.9` 镜像。

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

也可`edit` Deployment，并将`.spec.template.spec.containers[0].image` 从`nginx:1.7.9` 改为  `nginx:1.9.1` ：

```shell
$ kubectl edit deployment/nginx-deployment
deployment "nginx-deployment" edited
```

要想查看升级的状态，可运行：

```shell
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

升级成功后，您可能希望`get` Deployment：

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           36s
```

* up-to-date：表示Deployment已升级为最新配置的副本数量
* current：表示此Deployment管理的副本总数
* available：表示当前可用副本的数量。

运行`kubectl get rs` 即可看到，Deployment通过创建一个新的ReplicaSet并将其扩展到3个副本，并将旧ReplicaSet减少到0个副本的方式更新Pod。

```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```

运行`get pods` ，则只会显示新的Pod：

```
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

当下次我们要更新这些Pod时，只需再次更新Deployment的Pod模板。

Deployment可确保在升级时，只有一定数量的Pod会被关闭。默认情况下，它确保至少少有于1个期望数量的Pod运行（最多1个不可用）。

Deployment还可确保在期望数量的Pod上，只能创建一定数量的Pod。 默认情况下，它确保最多有多于1个期望数量的Pod运行（最多1个波动）。

> 译者注：以上两段表示：如果期望的Pod数目是3，那么在升级的过程中，保持运行状态的Pod最小是2，最大是4。

未来，默认值将从1-1变为25％-25％。

例如，如果您仔细观察上述Deployment，您将看到它首先创建了一个新Pod，然后删除一些旧Pod并创建新Pod。直到新新Pod的数量已经足够，它才会杀死旧Pod；直到足够数量的老Pod被杀死才创建新Pod。 它确保可用的Pod数量至少为2，并且Pod总数量至多为4。

```yaml
$ kubectl describe deployments
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 12:01:06 -0700
Labels:         app=nginx
Annotations:    deployment.kubernetes.io/revision=2
Selector:       app=nginx
Replicas:       3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:       app=nginx
  Containers:
   nginx:
    Image:              nginx:1.9.1
    Port:               80/TCP
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
OldReplicaSets:     <none>
NewReplicaSet:      nginx-deployment-1564180365 (3/3 replicas created)
Events:
  FirstSeen LastSeen    Count   From                     SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                     -------------   --------    ------              -------
  36s       36s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
```

由上可知，当我们第一次创建Deployment时，它创建了一个ReplicaSet（nginx-deployment-2035384211），并直接将其扩容到3个副本。当我们升级Deployment时，它创建了一个新ReplicaSet（nginx-deployment-1564180365），并将其扩容到1，然后将旧ReplicaSet缩容到2。这样，在任何时候，至少有2个Pod可用，并且至多创建4个Pod。 然后，继续按照相同的滚动更新策略对新旧ReplicaSet进行扩容/缩容。最后，新ReplicaSet中将会有3个可用副本，旧ReplicaSet副本缩小到0。



### Rollover（翻转）（也称为multiple updates in-flight[不停机多更新、多个升级并行]）

每次Deployment Controller观察到新的Deployment对象时，则会创建一个ReplicaSet来启动所期望的Pod（如果没有现有的ReplicaSet执行此操作）。现有ReplicaSet控制那些标签与`.spec.selector` 匹配，但其模板不与`.spec.template` 匹配的对象缩容。最终，新ReplicaSet将缩放到`.spec.replicas` ，所有旧ReplicaSets将被缩放为0。

如果在更新Deployment时，另一个更新正在进行中，那么Deployment就会为每个更新创建一个新的ReplicaSet，并开始扩容，并将滚动以前扩容的ReplicaSet——它会将其添加到旧ReplicaSet列表中，并开始缩容它。

例如，假设您创建一个Deployment，让它创建5个`nginx:1.7.9` 副本，当仅3个`nginx:1.7.9` 副本完成创建时，你开始更新Deployment，让它创建5个`nginx:1.9.1` 副本。在这种情况下，Deployment将会立即开始杀死已创建的3个`nginx:1.7.9` Pod，并将开始创建`nginx:1.9.1` Pod。它不会等待5个副本的`nginx:1.7.9` 都创建完成后再开始创建1.9.1的Pod。



### 标签选择器更新

通常不鼓励更新标签选择器，建议您预先规划好您的选择器。无论如何，如果您需要更新标签选择器，请务必谨慎，并确保您已经掌握了所有的含义。

**注意：**在API版本`apps/v1beta2` 中，Deployment的标签选择器创建后不可变。

- 添加选择器要求Deployment规范中的Pod模板标签也更新为新标签，否则将会返回验证错误。此更改是不重叠的，这意味着新选择器不会选择使用旧选择器所创建的ReplicaSet和Pod，也就是说，所有旧版本的ReplicaSet都会被丢弃，并创建新ReplicaSet。
- 更新选择器——即，更改选择器中key中的现有value，会导致与添加相同的行为。
- 删除选择器——即从Deployment选择器中删除现有key——不需要对Pod模板标签进行任何更改。现有的ReplicaSet不会被孤立，也不会创建新的ReplicaSet，但请注意，删除的标签仍然存在于任何现有的Pod和ReplicaSet中。





## 回滚Deployment

有时您可能想要回滚Deployment；例如，当Deployment不稳定时，例如循环崩溃。 默认情况下，所有Deployment的升级历史记录都保留在系统中，以便能随时回滚（可通过修改“版本历史记录限制”进行更改）。

**注意：**当Deployment的升级被触发时，会创建Deployment的修订版本。这意味着当且仅当更改Deployment的Pod模板（ `.spec.template` ）时，才会创建新版本，例如，模板的标签或容器镜像发生改变。其他更新（例如伸缩Deployment）不会创建Deployment修订版本，以便我们可以方便同时进行手动或自动缩放。这意味着，当您回滚到较早的版本时，对于一个Deployment，只有Pod模板部分会被回滚。

假设我们在更新Deployment时写错了字，将镜像名称写成了`nginx:1.91` 而非`nginx:1.9.1` ：

```
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment "nginx-deployment" image updated
```

升级将被卡住。

```
$ kubectl rollout status deployments nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```

按下Ctrl-C，即可停止查阅上述状态。有关升级卡住的更多信息，请 [read more here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status) 。

您还将看到旧副本（nginx-deployment-1564180365和nginx-deployment-2035384211）和新副本（nginx-deployment-3066724191）的数量都是2。

```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   2         2         0       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   2         2         2       6s
```

查看创建的Pod，您将看到由新ReplicaSet创建的2个Pod卡在拉取镜像的过程中。

```
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
nginx-deployment-3066724191-eocby   0/1       ImagePullBackOff   0          6s
```

**注意：**Deployment Controller将自动停止不良的升级，并将停止扩容新的ReplicaSet。这取决于您指定的rollingUpdate参数（具体为`maxUnavailable` ）。默认情况下，Kubernetes将maxUnavailable设为1，而spec.replicas也为1，因此，如果您没有关注过这些参数设置，则默认情况下，您的Deployment可能100％不可用！这将在未来版本的Kubernetes中修复。

```
$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       2 updated | 3 total | 2 available | 2 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     nginx-deployment-1564180365 (2/2 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (2/2 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
```

为解决以上问题，我们需要回滚到以前稳定版本的Deployment。



### 检查Deployment的升级历史记录

首先，检查此Deployment的修订版本：

```
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f docs/user-guide/nginx-deployment.yaml --record
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

因为我们在创建此Deployment时，使用`--record` 记录了命令，所以我们能够轻松看到我们在每个版本中所做的更改。

要进一步查看每个版本的详细信息，请运行：

```
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
  Containers:
   nginx:
    Image:      nginx:1.9.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```



### 回滚到以前的版本

现在我们决定：撤消当前的升级并回滚到以前的版本：

```
$ kubectl rollout undo deployment/nginx-deployment
deployment "nginx-deployment" rolled back
```

或者，可通过`--to-revision` 参数指定回滚到特定修订版本：

```
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment "nginx-deployment" rolled back
```

有关回滚命令相关的信息，详见 [`kubectl rollout`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rollout) 。

现在，Deployment就会回滚到以前的稳定版本。如下可知，Deployment Controller会生成`DeploymentRollback` 事件。

```
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           30m

$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     <none>
NewReplicaSet:      nginx-deployment-1564180365 (3/3 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  30m       30m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  2m        2m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-3066724191 to 0
  2m        2m          1       {deployment-controller }                Normal      DeploymentRollback  Rolled back deployment "nginx-deployment" to revision 2
  29m       2m          2       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
```





## 伸缩Deployment

可使用如下命令伸缩Deployment：

```
$ kubectl scale deployment nginx-deployment --replicas=10
deployment "nginx-deployment" scaled
```

假设您的群集启用了 [horizontal pod autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) 功能，您可以为Deployment设置一个autoscaler，并根据现有Pod的CPU利用率，选择要运行的Pod的最小和最大个数。

```
$ kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
deployment "nginx-deployment" autoscaled
```





### Proportional scaling（比例缩放）

RollingUpdate Deployment支持同时运行一个应用程序的多个版本。当您或autoscaler伸缩一个正处于升级中（正在进行或暂停）的RollingUpdate Deployment时，Deployment Controller就会平衡现有的、正在活动的ReplicaSet（ReplicaSet with Pod）中新增的副本，以减轻风险。 这称为*比例缩放* 。

例如，您正在运行一个具有10个副本的Deployment，[maxSurge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge)=3， [maxUnavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable)=2。

```
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

你更新到一个新的镜像，该镜像在集群内部无法找到。

```
$ kubectl set image deploy/nginx-deployment nginx=nginx:sometag
deployment "nginx-deployment" image updated
```

镜像使用ReplicaSet nginx-deployment-1989198191开始升级，但是由于上面设置了maxUnavailable=2，升级将被阻塞。

```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

然后，发起一个新的Deployment扩容请求。autoscaler将Deployment副本增加到15个。Deployment Controller需要决定在哪里添加这个5个新副本。如果我们不使用比例缩放，那么5个副本都将会被添加到新的ReplicaSet中。使用比例缩放，则新添加的副本将传播到所有ReplicaSet中。较大比例会被加入到有更多的副本的ReplicaSet，较低比例会被加入到较少的副本的ReplicaSet。剩余部分将添加到具有最多副本的ReplicaSet中。具有零个副本的ReplicaSet不会被扩容。

在上面的示例中，3个副本将被添加到旧ReplicaSet中，2个副本将添加到新ReplicaSet中。升级进程最终会将所有副本移动到新的ReplicaSet中，假设新副本变为健康状态。

```
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```





## 暂停与恢复Deployment

在触发一个或多个更新之前，您可以暂停Deployment，然后恢复。这将允许您在暂停和恢复之间应用多个补丁，而不会触发不必要的升级。

例如，使用刚刚创建的Deployment：

```
$ kubectl get deploy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```

通过运行以下命令暂停：

```
$ kubectl rollout pause deployment/nginx-deployment
deployment "nginx-deployment" paused
```

然后升级Deployment的镜像：

```
$ kubectl set image deploy/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

请注意，并不会产生新的ReplicaSet：

```
$ kubectl rollout history deploy/nginx-deployment
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>

$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```

您可以根据需要进行多次更新，例如，更新将要使用的资源：

```
$ kubectl set resources deployment nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
deployment "nginx-deployment" resource requirements updated
```

在暂停之前，Deployment的初始状态将继续执行其功能；只要Deployment暂停，Deployment的更新将不会有任何影响。

最终，恢复Deployment并观察一个新的ReplicaSet提供了所有新的更新：

```
$ kubectl rollout resume deploy/nginx-deployment
deployment "nginx" resumed
$ kubectl get rs -w
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
^C
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

**注意：**恢复暂停的Deployment之前，您无法回滚。





## Deployment状态

Deployment在其生命周期中有各种状态。在升级新ReplicaSet时是 [progressing](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#progressing-deployment) ，也可以是 [complete](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#complete-deployment) 或 [fail to progress](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment) 状态。

### Progressing Deployment（进行中的Deployment）

当执行以下任务之一时，Kubernetes将Deployment标记为*progressing* ：

- Deployment创建一个新ReplicaSet。
- 该Deployment正在扩容其最新ReplicaSet。
- Deployment正在缩容其旧版ReplicaSet。
- 新Pod已准备就绪或可用（Ready状态至少持续了 [MinReadySeconds](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#min-ready-seconds) 时间）。

您可使用`kubectl rollout status` 监视部署的进度。



### Complete Deployment（完成的Deployment）

Deployment具有以下特点时候，Kubernetes将Deployment标记为*complete* ：

- 与Deployment相关联的所有副本都已被更新为您所指定的最新版本，也就是说您所请求的任何更新都已完成。
- 与Deployment相关联的所有副本都可用。
- Deployment的旧副本都不运行。

可使用`kubectl rollout status` 来检查Deployment是否已经完成。如果升级成功，则`kubectl rollout status` 将会返回一个为零的退出代码。

```
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx" successfully rolled out
$ echo $?
0
```



### Failed Deployment（失败的Deployment）

您的Deployment在尝试部署最新ReplicaSet的过程中可能会阻塞，永远也无法完成。这可能是由于以下一些因素造成的：

- 配额不足
- 就绪探针探测失败
- 镜像拉取错误
- 权限不足
- 范围限制
- 应用程序运行时配置错误

您可以检测到这种情况的一种方法，是在Deployment spec中指定一个期限参数：（ [`spec.progressDeadlineSeconds`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#progress-deadline-seconds) ）。 `spec.progressDeadlineSeconds` 表示Deployment Controller等待多少秒后认为Deployment进程已停止。

以下`kubectl` 命令使用`progressDeadlineSeconds` 设置spec，使Controller在10分钟后缺少Deployment进度：

```
$ kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
"nginx-deployment" patched
```

一旦超过截止时间，Deployment Controller就会将一个DeploymentCondition添加到Deployment的`status.conditions` ，DeploymentCondition包含以下属性：

- Type=Progressing
- Status=False
- Reason=ProgressDeadlineExceeded

有关状态条件的更多信息，请参阅 [Kubernetes API conventions](https://git.k8s.io/community/contributors/devel/api-conventions.md#typical-status-properties) 。

**注意：**除报告`Reason=ProgressDeadlineExceeded` 以外，Kubernetes不会对停滞的Deployment采取任何行动。 更高级别的协调者，可利用它并采取相应的行动，例如，将Deployment恢复到之前的版本。

**注意：**如果暂停Deployment，Kubernetes不会根据您指定的截止时间检查进度。 您可以在升级过程中安全地暂停Deployment，然后再恢复，这样不会触发超过截止时间的条件。

您的Deployments可能会遇到短暂的错误，这可能是由于您设置的超时时间偏低，或者可能是由于可被视为“短暂”的其他类型的错误。例如，配额不足。 如果您describe Deployment，将可看到以下部分的内容：

```
$ kubectl describe deployment nginx-deployment
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

如果运行`kubectl get deployment nginx-deployment -o yaml` ，则Deployment状态可能如下所示：

```
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

最终，一旦Deployment进度超过了截止时间，Kubernetes将会更新状态以及导致Progressing的原因：

```
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

可缩容Deployment、缩容正在运行的其他Controller或通过增加namespace中的配额，从而解决配额不足的问题。当您满足配额条件后，Deployment Controller就会完成升级，您将看到Deployment的状态更新为成功（ `Status=True`和`Reason=NewReplicaSetAvailable` ）。

```
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

`Type=Available` ，并且`Status=True` 表示您的Deployment具有最低可用性。 最低可用性由部署策略中指定的参数决定。 `Type=Progressing` ，并且 `Status=True` 表示您的Deployment正在升级中；正处于progressing状态；抑或已成功完成其进度，并且达到所需的最小可用新副本个数（请查看特定状态的原因——本例中，`Reason=NewReplicaSetAvailable` 意味着Deployment完成）。

可使用`kubectl rollout status` 来检查Deployment进程是否失败。如果Deployment已超过截止之间，则`kubectl rollout status` 将会返回非零的退出代码。

```
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
$ echo $?
1
```



### 操作失败的Deployment

应用于完成的Deployment的所有操作也适用于失败的Deployment。如果需要在Deployment的Pod模板中应用多个调整，您可以进行扩容/缩容、回滚到先前的版本，甚至是暂停。





## 清理策略

可在Deployment中设置`.spec.revisionHistoryLimit` 字段，从而指定该Deployment要保留多少个旧版本ReplicaSet。 其余的将在后台垃圾收集。 默认情况下，所有修订历史都将被保留。在将来的版本中，默认是2。

**注意：**明确将此字段设置为0将会导致清除Deployment的所有历史记录，这会导致Deployment无法回滚。





## 用例

### Canary Deployment（金丝雀部署）

如果要使用Deployment向一部分用户或服务器发布版本，则可以按照 [managing resources](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments) 描述的金丝雀模式，创建多个Deployment，每个Deployment对应各自的版本。





## 编写Deployment Spec

与所有其他Kubernetes配置一样，Deployment需要`apiVersion` 、 `kind` 和 `metadata` 等字段。有关使用配置文件的一般信息，请参阅 [deploying applications](https://kubernetes.io/docs/tutorials/stateless-application/run-stateless-application-deployment/) ，配置容器以及 [using kubectl to manage resources](https://kubernetes.io/docs/tutorials/object-management-kubectl/object-management/) 文档。

Deployment还需要一个 [`.spec` section](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) 。



### Pod Template

`.spec.template`是`.spec` 唯一必需的字段。

`.spec.template`是一个 [pod template](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates) 。它与 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 有完全相同的schema，除了它是嵌套的，并且没有`apiVersion` 或`kind` 。

除了Pod必需的字段之外，Deployment中的Pod模板必须指定适当的标签和重启策略。对于标签，请确保不与其他Controller重叠。详见 [selector](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#selector) ）。

[`.spec.template.spec.restartPolicy`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) 只允许等于 `Always` ，如果未指定，则为默认值。



### 副本

`.spec.replicas` 是一个可选字段，用于指定期望的`.spec.replicas` 的数量，默认为1。



### 选择器

`.spec.selector` 是一个可选字段，指定此Deployment所关联的Pod的 [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 。

`.spec.selector`必须与`.spec.template.metadata.labels` 相匹配，否则将被API拒绝。

在API版本`apps/v1beta2` 中，如果未经设置，则`.spec.selector` 和`.metadata.labels` 不再默认与`.spec.template.metadata.labels` 相同。 所以，必须明确设定这些字段。 另请注意，在`apps/v1beta2` 中，`.spec.selector` 在Deployment创建后是不可变的。

对于模板与`.spec.template` 不同，抑或副本总数超过`.spec.replicas` 定义的Pod，Deployment可能会终止这些Pod。如果Pod的数量小于所需的数量，它将会使用`.spec.template` 的定义启动新Pod。

> 译者按：“模板与`.spec.template` 不同”的场景：Deployment先创建，然后修改YAML定义文件，升级镜像的版本。此时，旧Pod的镜像字段就与`.spec.template` 不同了

**注意：**你不应该建立其他与该选择器相匹配的Pod，无论是直接创建，还是通过另一个Deployment创建，抑或通过另一个Controller创建（例如ReplicaSet或ReplicationController）。如果这样做，第一个Deployment将会认为是它创造了这些Pod。 Kubernetes并不会阻止你这么做。

如果多个Controller的选择器发生重叠，Controller会发生冲突，并导致不正常的行为。





### 策略

`.spec.strategy` 指定使用新Pod代替旧Pod的策略。`.spec.strategy.type` 可有“Recreate”或“RollingUpdate” 两种取值。默认为“RollingUpdate”。



#### Recreate Deployment

当`.spec.strategy.type==Recreate` 时，创建新Pod前，会先杀死所有现有的Pod。



#### Rolling Update Deployment（滚动更新Deployment）

当`.spec.strategy.type==RollingUpdate` 时，Deployment以 [rolling update](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/) 的方式更新Pod。可指定`maxUnavailable` 和`maxSurge` 控制滚动更新的过程。

##### Max Unavailable

`.spec.strategy.rollingUpdate.maxUnavailable` 是一个可选字段，用于指定在更新的过程中，不可用Pod的最大数量。该值可以是一个绝对值（例如5），也可以是期望Pod数量的百分比（例如10％）。通过百分比计算出来的绝对值会向下取整。如果 `.spec.strategy.rollingUpdate.maxSurge`是0，那么该值不能为0。默认值是25％。

例如，此值被设置为30％，当滚动更新开始时，旧ReplicaSet会立即缩容到期望Pod数量的70％。一旦新的Pod进入Ready状态，老ReplicaSet可进一步缩容，然后扩容新ReplicaSet，确保在更新过程中，任何时候都会有至少70％的可用Pod。

##### Max Surge

`.spec.strategy.rollingUpdate.maxSurge` 是一个可选字段，用于指定在更新的过程中，超过Pod期望数量的最大数量。该值可以是一个绝对值（例如5），也可以是期望Pod数量的百分比（例如10％）。如果`MaxUnavailable`为0，该值不能为0。通过百分比计算出来的绝对值会绝对会向上取整。默认值是25％。

例如，此值被设置为30％，当滚动更新开始时，新ReplicaSet会立即扩容， 新旧Pod的总数不超过期望Pod数量的130％。一旦老Pod已被杀死，新ReplicaSet可进一步扩容，确保在更新过程中，任何时候运行的Pod总数不超过期望Pod数量的130％。



### Progress Deadline Seconds

`.spec.progressDeadlineSeconds` 是一个可选字段，用于指定表示Deployment Controller等待多少秒后认为Deployment进程已  [failed progressing](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment)  ——表现为在资源状态中有： `Type=Progressing` 、`Status=False`，以及`Reason=ProgressDeadlineExceeded` 。Deployment Controller将继续重试该Deployment。在未来，一旦实现自动回滚，Deployment Controller观察到这种状况时，就会尽快回滚 Deployment。

如需设置本字段，值必须大于`.spec.minReadySeconds` 。



### Min Ready Seconds

`.spec.minReadySeconds` 是一个可选字段，用于指定新创建的Pod进入Ready状态（Pod的容器持续多少秒不发生崩溃，就被认为可用）的最小秒数。默认为0（ Pod在Ready后就会被认为是可用状态）。要了解什么时候Pod会被认为已Ready，详见 [Container Probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) 。





### Rollback To

在API版本`extensions/v1beta1` 和`apps/v1beta1` 中，`.spec.rollbackTo` 已被弃用，并且在API版本`apps/v1beta2` 中不再支持该字段。取而代之的是，建议使用`kubectl rollout undo` ，详见 [Rolling Back to a Previous Revision](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-to-a-previous-revision) 。



### Revision History Limit（修订历史限制）

Deployment的修订历史记录存储在它所控制的ReplicaSet内。

`.spec.revisionHistoryLimit` 是一个可选字段，用于指定要保留的ReplicaSet数量，以便回滚。它的理想取值取决于新Deployment的频率和稳定性。默认情况下，所有老ReplicaSet都被保存，将资源存储在`etcd` 中，使用`kubectl get rs` 查询ReplicaSet信息。每个Deployment修订的配置都被存储在其ReplicaSet；因此，一旦旧ReplicaSet被删除，Deployment将无法回滚到那个修订版本。

更具体地讲，将该字段设为零，意味着所有0副本的旧ReplicaSet将被清理。在这种情况下，一个新的Deployment无法回滚，因为其修订历史都被清除了。



### Paused

`.spec.paused` 是一个可选的布尔类型的字段，用于暂停和恢复Deployment。Deployment暂停和未暂停之间唯一的区别是：对暂停Deployment的PodTemplateSpec所做的任何更改，不会触发新的升级。当Deployment创建后，默认情况下不会暂停。





## Deployment的替代方案



### kubectl滚动更新

[Kubectl rolling update](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update) 以类似的方式更新Pod和ReplicationControllers。但是建议使用Deployment，因为是声明式、服务器端的，并有额外的功能，例如滚动更新完成后可回滚到历史版本。





## 原文

<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>