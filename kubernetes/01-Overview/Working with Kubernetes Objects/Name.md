# 名称

Kubernetes REST API中的所有对象都会被Name和UID明确标识。

对于用户提供**非唯一**属性，Kubernetes提供 [labels](https://kubernetes.io/docs/user-guide/labels) 和 [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 来标识。



## 名称（Name）

名称通常由客户提供。对于给定类型的对象，一次只有一个对象可以有一个给定的名称（即：它们在空间上是唯一的）。但是，如果您删除一个对象，则可使用相同的名称创建一个新对象。名称用于引用资源URL中的对象，例如`/api/v1/pods/some-name` 。按照惯例，Kubernetes资源的名称最多可达253个字符，由小写字母、数字、`-` 和`.` 组成，但某些资源有更具体的限制。有关名称的精确语法规则，详见： [identifiers design doc](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md) 。



## UID

UID由Kubernetes生成。 在Kubernetes集群的整个生命周期中创建的每个对象都有不同的UID（即：它们在空间和时间上是唯一的）。