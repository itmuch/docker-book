# Namespace（命名空间）

Kubernetes支持在同一物理集群中创建多个虚拟集群。 这些虚拟集群被称为Namespace。





## 使用多个Namespace的场景

Namespace旨在用于这种环境：有许多的用户，这些用户分布在多个团队/项目中。对于只有几个或几十个用户的集群，您根本不需要创建或考虑使用Namespace。 当您需要使用Namespace提供的功能时，再考虑使用Namespace。

Namespace为Name（名称）提供了范围。在Namespace中，资源的名称必须唯一，但不能跨Namespace。

Namespace是一种在多种用途之间划分集群资源的方法（通过  [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) ）。

在未来的Kubernetes版本中，同一Namespace中的对象默认有相同的访问控制策略。

没有必要使用多个Namespace来分隔稍微不同的资源，例如同一软件的不同版本，可使用  [labels](https://kubernetes.io/docs/user-guide/labels) 来区分同一Namespace中的资源。





## 使用Namespace

Namespace的创建和删除在 [Admin Guide documentation for namespaces](https://kubernetes.io/docs/admin/namespaces) 有描述。



### 查看Namespace

可使用如下命令列出集群中当前的Namespace：

```Shell
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
```

Kubernetes初始有两个Namespace：

- `default` ：对于没有其他Namespace的对象的默认Namespace
- `kube-system` ：由Kubernetes系统所创建的对象的Namespace



### 为请求设置Namespace

要临时设置请求的Namespace，可使用`--namespace` 标志。

例如：

```shell
$ kubectl --namespace=<insert-namespace-name-here> run nginx --image=nginx
$ kubectl --namespace=<insert-namespace-name-here> get pods
```



### 设置Namespace首选项

可在上下文中永久保存所有后续`kubectl` 命令的Namespace。

```shell
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
# Validate it
$ kubectl config view | grep namespace:
```





## Namespace和DNS

当你创建 [Service](https://kubernetes.io/docs/user-guide/services) 时，会创建一个相应的 [DNS entry](https://kubernetes.io/docs/admin/dns) 。此条目的形式为`<service-name>.<namespace-name>.svc.cluster.local` ，这意味着如果容器只使用`<service-name>` ，则它将解析为Namespace本地的服务。 这对在多个Namespace（例如Development、Staging以及Production）中使用相同的配置的场景非常有用。 如果要跨越Namespace，请使用完全限定域名（FQDN）。

> 译者按：FQDN是fully qualified domain name的缩写，即：完全限定域名





## 并非所有对象都在Namespace中

大多数Kubernetes资源（例如：Pod、Service、Replication Controllers等）都在某些Namespace中。但Namespace资源本身并不在Namespace中。低级资源（例如： [nodes](https://kubernetes.io/docs/admin/node) 和persistentVolumes）也不在任何Namespace中。事件是一个例外：它们可能有也可能没有Namespace，具体取决于事件的对象。

