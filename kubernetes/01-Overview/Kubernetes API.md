# Kubernetes API

 [API conventions doc](https://git.k8s.io/community/contributors/devel/api-conventions.md) 中描述了API的总体规范。

 [API Reference](https://kubernetes.io/docs/reference) 中描述了API端点、资源类型和样本。

 [access doc](https://kubernetes.io/docs/admin/accessing-the-api) 讨论了API的远程访问。

Kubernetes API也是系统声明式配置模式的基础。  [Kubectl](https://kubernetes.io/docs/user-guide/kubectl) 命令行工具可用于创建、更新、删除以及获取API对象。

Kubernetes也会存储其API资源方面的序列化状态（目前存在 [etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/) 中）。

Kubernetes本身被分解成了多个组件，通过其API进行交互。





## API更改

根据我们的经验，任何成功的系统都需要随着新用例的出现或现有的变化而发展和变化。因此，我们预计Kubernetes API将会不断变化和发展。但是，在很长一段时间内并不会破坏与现有客户端的兼容性。一般来说，新的API资源和新的资源字段通常可被频繁添加。消除资源或字段将需遵循 [API deprecation policy](https://kubernetes.io/docs/reference/deprecation-policy/) 。

[API change document](https://git.k8s.io/community/contributors/devel/api_changes.md) 详细介绍了兼容更改以及如何更改API的内容。





## OpenAPI与Swagger定义

完整的API详情使用 [Swagger v1.2](http://swagger.io/) 和 [OpenAPI](https://www.openapis.org/) 记录。Kubernetes apiserver（又名“master”）公开了一个路径是`/swaggerapi` API，该API使用Swagger v1.2 Kubernetes API规范。 您也可以通过将`--enable-swagger-ui=true` 标志传递给apiserver，从而浏览`/swagger-ui` 的启用UI的API文档。

从Kubernetes 1.4开始，OpenAPI规范也可在 [`/swagger.json`](https://git.k8s.io/kubernetes/api/openapi-spec/swagger.json) 。当我们从Swagger v1.2转换到OpenAPI（又名Swagger v2.0）时，一些工具（例如：kubectl和swagger-ui）仍在使用v1.2规范。OpenAPI规范在Kubernetes 1.5中，进入Beta阶段。

Kubernetes为主要用于集群内通信的API实现了另一种基于Protobuf的序列化格式，在 [design proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md) 有记录，每个schema的IDL文件都存放在定义该API对象的Go语言包中。





## API版本

为了更容易地消除字段或重组资源表示，Kubernetes支持多种API版本，每种API版本都有不同的API路径，例如`/api/v1`或`/apis/extensions/v1beta1` 。

我们选择在API级别，而非资源级别/字段级别使用版本控制，从而确保API提供清晰、一致的系统资源和行为视图，以及控制对终极API/实验API的访问。JSON和Protobuf序列化schema遵循相同的schema更改准则——以下所有描述都涵盖了两种格式。

请注意，API版本控制和软件版本控制仅仅是间接相关的关系。 [API and release versioning proposa（API和版本发布提案）l](https://git.k8s.io/community/contributors/design-proposals/release/versioning.md) 描述了API版本控制和软件版本控制之间的关系。

不同的API版本意味着不同程度的稳定性和支持。[API Changes documentation](https://git.k8s.io/community/contributors/devel/api_changes.md#alpha-beta-and-stable-versions) 详细描述了每个级别的标准。概括如下：

- Alpha级别:
  - 版本名称包含`alpha` （例如`v1alpha1` ）
  - 可能有一些bug，启用该功能可能会显示错误。 默认禁用
  - 一些功能可能随时会被废弃，恕不另行通知
  - API可能会以不兼容的方式更改，恕不另行通知
  - 建议仅在短期测试集群中使用，因为增加了bug带来的风险，而且缺乏长期支持
- Beta级别:
  - 版本名称包含`beta` （例如`v2beta3` ）
  - 代码经过了良好的测试。启用该功能被认为是安全的。 默认启用
  - 整体功能不会被删除，尽管细节可能会改变
  - 对象的schema/语义可能会在后续的beta版/稳定版本中以不兼容的方式发生变化。发生这种情况时，我们将提供迁移到下一个版本的说明。 这可能需要删除、编辑和重新创建API对象。编辑进程可能需要一些思考。依赖该功能的应用程序可能需要停机。
  - 推荐仅用于非关键业务用途，因为后续版本中可能会发生不兼容的更改。如果您有多个可独立升级的集群，则可放宽此限制。
  - **请尝试我们的beta功能并给他们反馈！一旦他们退出beta状态，我们就不会做更多的改变。**


- Stable等级：
  - 版本名称为`vX` ，其中`X`为整数。
  - 稳定版本的功能将会出现在许多后续版本中。





## API组

为了使Kubernetes API扩展更容易，我们实现了 [*API groups*](https://git.k8s.io/community/contributors/design-proposals/api-machinery/api-group.md) 。 API组可在序列化对象的`apiVersion` 字段中使用一个REST路径指定。

目前可使用的几个API组：

1. “core”（由于没有明确的组名称，通常称为“legacy”）组，它的REST路径是`/api/v1` 。例如`apiVersion: v1` 。
2. 命名组是REST路径`/apis/$GROUP_NAME/$VERSION` ，并使用`apiVersion: $GROUP_NAME/$VERSION` （例如`apiVersion: batch/v1` ）。 支持的API组的完整列表可详见：[Kubernetes API reference](https://kubernetes.io/docs/reference/) 。

使用 [custom resources](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) 扩展API有两个支持的路径：

1. [CustomResourceDefinition](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/) 适用于非常基本的CRUD需求的用户。
2. 即将推出：用户需要完整的Kubernetes API语法，这些语法可实现自己的apiserver，并使用 [aggregator](https://git.k8s.io/community/contributors/design-proposals/api-machinery/aggregated-api-servers.md) 无缝连接客户端。





## 启用API组

默认情况下，某些资源和API组已被启用。可通过在apiserver上设置`--runtime-config` 来启用或禁用它们。 `--runtime-config`接受逗号分隔的值。例如：要禁用`batch / v1` ，请设置`--runtime-config=batch/v1=false` ；想启用`batch/v2alpha1` ，可设置`--runtime-config=batch/v2alpha1` 。 该标志接受逗号分隔的一组键值对，键值对描述了apiserver的运行时配置。

**重要信息**：启用或禁用组或资源，需要重启apiserver和controller-manager来获取`--runtime-config` 的更改。





## 启用组中的资源

默认情况下，DaemonSets、Deployments、HorizontalPodAutoscalers、Ingress、Jobs和ReplicaSets都被启用。可通过在apiserver上设置`--runtime-config` 来启用其他扩展资源。`--runtime-config` 接受逗号分隔值。 例如：要禁用Deployments和Ingress，可设置

```
--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/ingress=false
```





## 原文

<https://kubernetes.io/docs/concepts/overview/kubernetes-api/>