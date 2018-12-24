# Daemon Set





## What is a DaemonSet?（什么是DaemonSet？）

*DaemonSet*确保所有（或一些）Node运行Pod的副本。随着Node被添加到集群，Pod被添加到Node上。随着从集群中删除Node，这些Pods被垃圾回收。删除DaemonSet将清理它创建的Pod。

DaemonSet的典型用途是：

- 在每个Node上运行集群存储daemon，例如在每个Node上运行`glusterd` 、 `ceph` 。
- 在每个Node上运行日志收集daemon，例如`fluentd` 或`logstash` 。
- 在每个Node上运行Node监控daemon，如 [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) 、`collectd` 、`Datadog Agent` ，New Relic Agent或Ganglia `gmond` 。

在简单的情况下，一个DaemonSet，覆盖所有Node，将用于每种类型的daemon。 更复杂的设置可能对单一类型的daemon使用多个DaemonSet，但对不同硬件类型使用不同标志和/或不同内存和cpu最小需求。





## Writing a DaemonSet Spec（编写DaemonSet Spec）



### Create a DaemonSet（创建DaemonSet）

您可以在YAML文件中描述DaemonSet。 例如，下面的 `daemonset.yaml` 文件描述了运行`fluentd-elasticsearch` Docker镜像的DaemonSet：

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: gcr.io/google-containers/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

- 基于YAML文件创建DaemonSet： `kubectl create -f daemonset.yaml`




### Required Fields（必填字段）

与所有其他Kubernetes配置一样，DaemonSet需要 `apiVersion` ， `kind` 和 `metadata` 字段。有关使用配置文件的信息，请参阅 [deploying applications](https://kubernetes.io/docs/user-guide/deploying-applications/) 、[configuring containers](https://kubernetes.io/docs/tasks/) 以及 [working with resources](https://kubernetes.io/docs/concepts/tools/kubectl/object-management-overview/) 。

DaemonSet还需要一个 [`.spec`](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) 部分。



### Pod Template（Pod模板）

`.spec.template` 是`.spec` 中必需的字段之一。

`.spec.template`是一个 [pod template](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates) 。 它与 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 具有完全相同的模式，只不过它是嵌套的，没有`apiVersion` 或`kind` 。

除Pod的必需字段外，DaemonSet中的Pod模板必须指定适当的标签（请参阅 [pod selector](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#pod-selector) ）。

DaemonSet中的Pod模板必须具有等于`Always`的 [`RestartPolicy`](https://kubernetes.io/docs/user-guide/pod-states) ，也可留空，默认为`Always` 。



### Pod Selector（Pod选择器）

`.spec.selector` 字段是一个pod选择器。 它的作用与 [Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 的`.spec.selector` 相同。

从Kubernetes 1.8起，您必须指定与 `.spec.template` 的Label相匹配的Pod选择器。如果该字段留空，Pod Selector将不再有默认值。Selector默认与 `kubectl apply` 不兼容。另外，一旦DaemonSet被创建，它的`spec.selector` 就不可变。改变Pod Selector可能会导致Pod的孤立，从而引起用户的困惑。

`spec.selector` 是由两个字段组成的对象：

- `matchLabels` - 与 [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 的`.spec.selector`相同。
- `matchExpressions` - 允许通过指定key列表、value列表、以及与key/value相关联的运算符的方式，来构建更复杂的选择器。

当同时指定以上两个字段时，两者之间AND关系。

如果指定了`.spec.selector` ，它必须与 `.spec.template.metadata.labels` 匹配。 如果未指定，则默认为相等。 不匹配的配置将会被API拒绝。

此外，您通常不应创建其标签与此选择器相匹配的任何Pod，不管是通过另一个DaemonSet还是通过其他Controller（如ReplicaSet）进行匹配。 否则，DaemonSet Controller会认为那些Pod是由它创建的。 Kubernetes不会阻止你这样做。您可能想要执行此操作的一种情况，是手动在一个Node上创建具有不同值的Pod进行测试。

如果您尝试创建一个DaemonSet，那么

// TODO



### Running Pods on Only Some Nodes（只在一些Node上运行Pod）

如果指定了`.spec.template.spec.nodeSelector` ，则DaemonSet Controller将在与该 [node selector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 匹配的Node上创建Pod。 同样，如果您指定了`.spec.template.spec.affinity` ，则DaemonSet控制器将在与该 [node affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 匹配的Node上创建Pod。 如果不指定，则DaemonSet Controller将会在所有Node上创建Pod。

> 译者按：拓展阅读：《 k8s nodeSelector&affinity》：<http://blog.csdn.net/yevvzi/article/details/54585686> 







## How Daemon Pods are Scheduled（如何调度Daemon Pods）

通常，Pod在哪台机器上运行是由Kubernetes Scheduler选择的。但是，由DaemonSet Controller创建的Pod已经事先确定在哪台机器上运行（在创建Pod时指定了`.spec.nodeName` ，因此Scheduler将忽略调度这些Pod）。因此：

- Node的 [`unschedulable`](https://kubernetes.io/docs/admin/node/#manual-node-administration) 字段不受DaemonSet Controller的约束。
- 即使Scheduler尚未启动，DaemonSet Controller也可以创建Pod，这样设计对集群启动是有帮助的。

Daemon Pod 关注所设置的 [taints and tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration) 规则，但它们会为如下未指定 `tolerationSeconds` 的taint创建`NoExecute` toleration。

- `node.kubernetes.io/not-ready` 
- `node.alpha.kubernetes.io/unreachable`

这样可以确保在启用 `TaintBasedEvictions` alpha功能时，当Node出现问题（如网络分区）时，它们将不被驱逐。（当 `TaintBasedEvictions` 功能未启用时，它们在这些情况下也不会被驱逐，但会NodeController的硬编码行为而被驱逐，而会因为toleration被驱逐）。

它们也tolerate以下的 `NoSchedule` taints：

- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`

当启用对critical Pod的支持时，并且DaemonSet中的Pod被标记为critical时，将会创建Daemon pod，Daemon Pod会为 `node.kubernetes.io/out-of-disk` 的taint创建额外 `NoSchedule` 的toleration。

请注意，如果启用了Alpha功能 `TaintNodesByCondition` ，上述`NoSchedule` Taints仅在1.8或更高版本中创建。





## Communicating with Daemon Pods（与Daemon Pod通信）

与DaemonSet中的Pod通信的方式有：

- **Push** ：配置DaemonSet中的Pod，将更新发送到其他Service服务，例如统计数据库。 它们没有客户端。
- **NodeIP and Known Port** ：DaemonSet中的Pod可以使用 `hostPort` ，以便通过Node的IP访问Pod。客户能通过某种方式知道Node IP的列表，并知道端口。
- **DNS** ：使用相同的Pod选择器创建 [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) ，然后使用 `endpoints` 资源或从DNS查询多个A记录发现DaemonSet。
- **Service** ：使用相同的Pod选择器创建Service，并使用该Service来访问随机Node上的daemon。 （无法到达特定Node）





## Updating a DaemonSet（更新DaemonSet）

如果Node的Label被更改，DaemonSet会立即将Pods添加到新匹配的Node，并从不匹配Node删除Pod。

您可以修改DaemonSet创建的Pod。但是，Pod不允许更新所有字段。此外，在下次创建一个Node（即使使用相同的名称）时，DaemonSet Controller依然会使用原始模板。

您可以删除DaemonSet。 如果使用`kubectl` 指定`--cascade=false` ，那么Pod将保留在Node上。然后，您可以使用不同的模板创建一个新的DaemonSet。新DaemonSet将识别所有具有匹配的标签的、已存在的Pod。尽管Pod模板并不匹配，但它不会修改或删除这些Pod。您将需要通过删除Pod或删除Node的方式，强制创建新的Pod。

在Kubernetes 1.6及更高版本中，您可以在DaemonSet上 [perform a rolling update](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/) 。

未来的Kubernetes版本将支持Node的受控更新。





## Alternatives to DaemonSet（DaemonSet的替代方案）

### Init Scripts（初始化脚本）

通过在Node上直接启动它们（例如使用`init` 、 `upstartd`或`systemd` ），可以运行daemon进程。这是非常好的。但是，通过DaemonSet运行这些进程有几个优点：


- 能够监控和管理daemon进程的日志，就像应用程序一样。
- 相同的配置语言和工具（如Pod模板， `kubectl` ）用于daemon和应用程序。
- 未来版本的Kubernetes可能支持 DaemonSet创建的Pod 与 Node升级工作流 之间的集成。
- 在具有资源限制的容器中运行daemon会增强应用容器和daemon隔离性。 当然，也可通过在容器中运行而不在Pod中运行daemon的方式，实现该目标（例如，直接通过Docker启动）。




### Bare Pods（裸Pod）

可以直接创建Pods，指定Pod运行在特定的Node上。但是，DaemonSet会替换由于任何原因而被删除或终止的Pod，例如在Node故障或Node被中断维护（例如内核升级）的情况下。因此，您应该使用DaemonSet而非单独创建Pod。



### Static Pods（静态Pod）

可通过将文件写入由Kubelet监视的某个目录来的方式来创建Pods。 这些被称为 [static pods](https://kubernetes.io/docs/concepts/cluster-administration/static-pod/) 。 与DaemonSet不同，静态Pod无法使用kubectl或其他Kubernetes API客户端进行管理。静态Pod不依赖于apiserver，这使其在集群启动的情况下很有用。 此外，静态Pod可能在将来会被废弃。



### Deployments

DaemonSets类似于 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment.md) ，因为它们都创建了Pods，并且这些Pod都有不期望终止的进程（例如web服务器，存储服务器）。

为无状态服务使用Deployment，例如前端，缩放副本数量和滚动升级比控制Pod运行在哪个主机上要重要得多。 当一个Pod副本始终在所有或某些主机上运行，并需要先于其他Pod启动时，可使用DaemonSet。





## 原文

<https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/>