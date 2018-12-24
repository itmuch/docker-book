# 理解K8s对象

这个页面描述了Kubernetes对象在Kubernetes API中的表现，以及如何用`.yaml` 格式表达它们。





## 理解Kubernetes对象

*Kubernetes对象*是Kubernetes系统中的持久实体。Kubernetes使用这些实体来表示集群的状态。具体来说，它们可描述：

- 容器化的应用程序正在运行什么（以及在哪些Node上运行）
- 这些应用可用的资源
- 这些应用的策略，例如重启策略，升级和容错

Kubernetes对象是一种“意图记录”——一旦您创建对象，Kubernetes系统将持续工作以确保对象存在。通过创建一个对象，您可以有效地告诉Kubernetes系统您希望集群的工作负载是怎样的？这是您集群的**期望状态** 。

要使用Kubernetes对象——无论是创建、修改还是删除它们，您都需要使用 [Kubernetes API](https://git.k8s.io/community/contributors/devel/api-conventions.md) 。 例如，当您使用`kubectl` 命令行时，CLI会为您提供必要的Kubernetes API调用；您也可直接在自己的程序中使用Kubernetes API。Kubernetes目前提供了一个`golang` [client library](https://github.com/kubernetes/client-go) ，并且正在开发其他语言的客户端库（如 [Python](https://github.com/kubernetes-incubator/client-python) ）。



### 对象Spec和Status（规格和状态）

每个Kubernetes对象都包含两个嵌套的对象字段，它们控制着对象的配置：对象*spec*和对象*status* 。您必须提供*spec* ，它描述了对象*所期望的状态*——您希望对象所具有的特性。*status*描述对象的*实际状态*，由Kubernetes系统提供和更新。在任何时候，Kubernetes Control Plane都会主动管理对象的实际状态，从而让其匹配你所期望的状态。

例如，Kubernetes Deployment是一个表示在集群上运行的应用程序的对象。在创建Deployment时，可设置Deployment spec，例如指定有三个应用程序的replicas（副本）正在运行。这样，Kubernetes系统就会读取Deployment spec，并启动您想要的、应用程序的三个实例——根据您的spec更新status。如果任何一个实例失败（status发生改变），Kubernetes系统就会响应spec和status之间的差异，并进行更正——在这本例中，会启动一个替换实例。

有关对象spec，status和metadata的更多信息，详见： [Kubernetes API Conventions](https://git.k8s.io/community/contributors/devel/api-conventions.md) 。



### 描述Kubernetes对象

在Kubernetes中创建对象时，必须提供对象的spec，spec描述对象所期望的状态，以及一些关于对象的基本信息（如名称）。当您使用Kubernetes API创建对象（直接或通过`kubectl` ）时，该API请求必须在请求体中包含该信息（以JSON格式）。 **最常见的，可在一个.yaml文件中向kubectl提供信息。** 在进行API请求时， `kubectl` 会将信息转换为JSON。

如下是一个示例`.yaml` 文件，显示Kubernetes Deployment所需的字段和对象spec：

```Yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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

使用该`.yaml` 文件创建Deployment的一种方法是在`kubectl` 命令行界面中使用 [`kubectl create`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#create)  命令，将`.yaml`文件作为参数传递。 例如：

```shell
$ kubectl create -f docs/user-guide/nginx-deployment.yaml --record
```

将会输出类似如下的内容：

```
deployment "nginx-deployment" created
```



### 必填字段

在Kubernetes对象的`.yaml` 文件中，您需要为以下字段设置值：

- `apiVersion` ——指定Kubernetes API的版本
- `kind` ——你想创建什么类型的对象
- `metadata` ——有助于唯一标识对象的数据，包括`name` 字符串，UID和可选的`namespace` 

您还需要提供`spec` 字段。对于不同的Kubernetes对象来说， `spec`的格式都是不同的。 [Kubernetes API reference](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) 可帮助您找到所有对象的spec格式。



## 原文

<https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/>