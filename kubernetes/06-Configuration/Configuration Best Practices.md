# Configuration Best Practices

本文档强调并整合了整个用户指南、入门文档和示例中引入的配置最佳实践。

这是一个“活”的文件。如果你想到的东西不在这个名单上，但可能对他人有用，请不要犹豫，提交issue或提交PR（pull request）。





## General Config Tips（常规配置Tips）

- 定义配置时，指定最新的稳定API版本（当前为v1）。

- 配置文件应该被推送到集群之前，应存储在版本控制软件中。这样，如果需要，可以快速回滚配置。如果需要，还可以帮助集群重新创建和恢复。

- 使用YAML而非JSON编写配置文件。尽管这些格式在几乎所有情况下可以互换使用，但YAML往往对用户更加友好。

- 只要有意义，将相关对象组合成单个文件。一个文件通常比多个文件更容易管理。请参阅 [guestbook-all-in-one.yaml](https://github.com/kubernetes/examples/tree/master/guestbook/all-in-one/guestbook-all-in-one.yaml) 文件作为此语法的示例。

  还要注意，可以在目录上调用许多`kubectl` 命令，因此您还可以在配置文件所在目录中调用`kubectl create` 。 请参阅下面的更多细节。

- 不要指定不必要的默认值 - 简单和最小的配置将减少错误。

- 将对象描述放在Annotation中。






## “Naked” Pods vs Replication Controllers and Jobs（“裸”Pod vs Replication Controllers/Job）

- 如果有一个可行的方案替代裸Pod（换句话说：Pod没有绑定到 [replication controller](https://kubernetes.io/docs/user-guide/replication-controller) ），请使用替代方案。 在Node发生故障的情况下，裸Pod将不会被重新调度。

  除了某些明确的 [`restartPolicy: Never`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 方案外，Replication Controller总是比直接创建Pod更为可取。一个 [Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 对象（目前处于Beta状态）也可能适合你。






## Services

- 通常最好在相应的 [replication controllers](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 之前创建一个 [service](https://kubernetes.io/docs/concepts/services-networking/service/) 。这使得scheduler能够分散构成Service的Pod。

  您还可以使用此过程来确保至少一个副本在创建许多副本之前起作用：

  1. 创建Replication Controller而不指定副本（这将会设置replicas = 1）;
  2. 创建Service
  3. 然后扩容Replication Controller。


- 除非绝对必要，否则不要使用`hostPort` （例如：对于node daemon）。 它指定要在Node上暴露的端口号。当您将Pod绑定到 `hostPort` ，由于端口冲突，调度Pod的数量有限 - 您只能调度与Kubernetes集群中的Node一样多的Pod。

  如果只需访问端口进行调试，可以使用 [kubectl proxy and apiserver proxy](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/) 或 [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) 。 您可以使用 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 对象进行外部服务访问。

  如果您需要在特定主机上暴露Pod的端口，请优先使用 [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) Service，再考虑使用 `hostPort` 。


- 由于与`hostPort`相同的原因，避免使用 `hostNetwork` 。
- 当您不需要kube-proxy负载均衡时，使用*headless services* 轻松发现服务。 见[headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 。







## Using Labels（标签的使用）

- 定义、使用能够标识应用程序或部署的**语义属性**的 [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 。例如，不要将Label附加到一组Pod来显式表示某些Service（例如 `service: myservice` ），或者显式地表示管理Pod的replication controller（例如`controller: mycontroller` ），你应该附加标识语义的Label属性，例如 `{ app: myapp, tier: frontend, phase: test, deployment: v3 }`。 这将允许您选择适用于上下文的对象组——例如，所有标记了`tier: frontend` 的Pod的Service；或“myapp”应用的所有“test”阶段的组件。有关此方法的示例，请参阅  [guestbook](https://github.com/kubernetes/examples/tree/master/guestbook/) 应用程序。

  可通过简单的、从其选择器中省略特定于发行版的标签，而非更新Service的选择器以完全Replication Controller的选择器的方式，来实现跨越多个部署的Service，例如 [rolling updates](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/) 。


- 为方便滚动更新，请在Replication Controller名称中包含版本信息，例如作为名称的后缀。 设置`version`标签也很有用。 滚动更新创建一个新的Controller，而不是修改现有的Controller。因此，版本无关的Controller名称可能会有问题。有关更多详细信息，请参阅rolling-update命令中的 [documentation](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/) 。

  请注意， [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 对象避免了管理Replication Controller `version names`的需要。 对象的期望状态由Deployment描述，如果应用对该spec的更改，则Deployment Controller将以可控的速率，来将实际状态更改为期望状态。（Deployment对象当前是 [`extensions` API Group](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups) 的一部分。）


- 您可以操作Label进行调试。由于Kubernetes Replication Controller和Service使用Label来匹配Pod，因此可通过删除相关Label来删除Controller或Service的Pod。如果您删除现有Pod的Label，其Controller将会创建一个新的Pod来取代它。这是一个有用的方法，用来调试隔离环境中之前“活着”的Pod。请参阅 [`kubectl label`](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 命令。






## Container Images（容器镜像）

-  [default container image pull policy](https://kubernetes.io/docs/concepts/containers/images/) 是`IfNotPresent` ，如果镜像已经存在，则会导致 [Kubelet](https://kubernetes.io/docs/admin/kubelet/) 不会拉取镜像的问题。如果您想始终强制拉取镜像，则必须在.yaml文件中指定`Always`的拉取策略（ `imagePullPolicy: Always` ），或在镜像上指定 `:latest` 标签。

  也就是说，如果您指定的镜像不是 `:latest` 标签，例如 `myimage:v1` ，并且更新了有相同标签的镜像，则Kubelet将不会拉取新的镜像。 可通过更新镜像标签时同时更新镜像的标签（例如改为 `myimage:v2` ），并确保您的配置指向正确的版本，从而解决此问题。

  **注意：**在生产中部署容器时，应避免使用 `:latest` 标签，因为这很难跟踪正在运行哪个版本的镜像并且很难回滚。


- 要使用特定版本的镜像，可使用其摘要（SHA256）来指定镜像。这种方法保证镜像永远不会更新。有关使用镜像摘要的详细信息，请参阅 [the Docker documentation](https://docs.docker.com/engine/reference/commandline/pull/#pull-an-image-by-digest-immutable-identifier) 。





## Using kubectl（kubectl的使用）

- 在可能的情况下使用 `kubectl create -f <directory>` 。这将会`<directory>` 中查找所有`.yaml` ， `.yml`和`.json`文件中查找配置对象，并将其传给`create` 命令。
- 使用`kubectl delete` 而不是 `stop` 。 `delete` 具有`stop` 功能的超集， `stop` 已被弃用。
- 使用kubectl批量操作（通过文件和/或Label）进行get和delete操作。 请参阅 [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)and [using labels effectively](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively) 。
- 使用`kubectl run` 和 `expose` 快速创建和暴露单个容器部署。有关示例，请参阅[quick start guide](https://kubernetes.io/docs/user-guide/quick-start/) 。





## 原文

<https://kubernetes.io/docs/concepts/configuration/overview/>