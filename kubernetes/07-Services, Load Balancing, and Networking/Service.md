# Service

Kubernetes [`Pods`](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 是有生命周期的，它们产生和死亡，一旦死亡就不会复活。[`ReplicationControllers`](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 能动态创建和销毁`Pod` （例如，当扩容、缩容或在 [rolling updates](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update) 时）。尽管每个`Pod` 有自己的IP，但这些IP可能是会变化的。这导致一个问题：如果一些`Pod` （称为backend）为Kubernetes群集中的其他`Pod` （称为frontend）提供功能，那么frontend如何找到，并一直能找到backend呢？

> 译者注：下面将backend译为后端，frontend译为前端。

可用`Services` 来解决该问题。

Kubernetes `Service` 是一个抽象，它定义了一组逻辑`Pod` 和一个访问它们的策略——有时称为微服务。 `Service` 所关联的一组`Pod` （通常）由 [`Label Selector`](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) 决定（详见下文，为什么您可能需要没有选择器的`Service` ）。

例如，假设一个图片处理的后端有三个正在运行的副本。这些副本是可替代的——前端不关心他们使用哪个后端副本。虽然构成后端的实际`Pod` 可能会改变，但前端客户端不应该也没必要跟踪自己的后端列表（例如：知道后端副本列表的地址等待）。 `Service` 抽象可实现这种解耦。

对于Kubernetes-native应用程序，Kubernetes提供了一个简单的`Endpoint` API，只要`Service` 中的`Pod` 发生变化，它将被更新。对于non-native应用程序，Kubernetes为Service提供了一个基于虚拟IP的网桥，该网桥可重定向到后端 `Pod` 。





## 定义Service

Kubernetes中的`Service` 是一个REST对象，类似于`Pod` 。 像所有的REST对象一样， `Service` 定义可被POST到apiserver，从而创建一个新的实例。 例如，假设您有一组`Pod` ，每个`Pod` 都暴露端口9376，并携带标签`"app=MyApp"` 。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

使用此文件，即可创建一个名为“my-service”的新`Service` 对象，该对象会代理那些使用TCP端口9376，并且有`"app=MyApp"` 标签的Pod，该`Service` 还会被分配一个IP地址（有时称为“Cluster IP”），它会被服务的代理使用（见下文）。 `Service` 的选择器将会被持续评估，处理的结果会被POST到一个名为“my-service”的`Endpoint` 对象。

请注意， `Service` 可将传入端口`port` 映射到任意`targetPort`端口。默认情况下， `targetPort` 将被设置为与`port` 字段相同的值。 也许更有趣的是， `targetPort` 可以是字符串，指向到是后端`Pod` 端口的名称。拥有该名称的端口的实际端口在每个后端`Pod` 中可能并不相同。这为`Services` 提供了很大的灵活性。例如，您可以在后端软件的下一个版本中更改该Pod所暴露的端口，而不会影响客户端的调用。

Kubernetes `Service`支持`TCP` 和`UDP` 协议。 默认值为`TCP` 。



### Services without selectors（不带选择器的Service）

Service通常抽象了 `Pod` 的访问，但也可抽象其他类型的后端。 例如：

- 您希望在生产环境中使用外部数据库集群，但在测试中，您将使用自己的数据库。
- 您希望将Service指向另一个 [`Namespace`](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 的Service或其他集群中的Service。
- 您正在将工作负载迁移到Kubernetes，部分后端运行在Kubernetes之外。

在以上这些情况下，您可以定义不带选择器的Service：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

因为该Service没有选择器，所以不会创建相应的`Endpoint` 对象。可手动将Service映射到指定的`Endpoint` ：

```yaml
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

**注意**：Endpoint IP不能是loopback（127.0.0.0/8），link-local （169.254.0.0/16）或link-local multicast（224.0.0.0/24）。

> 译者按：loopback、link-local、link-local multicast拓展阅读：[https://4sysops.com/archives/ipv6-tutorial-part-6-site-local-addresses-and-link-local-addresses/](https://4sysops.com/archives/ipv6-tutorial-part-6-site-local-addresses-and-link-local-addresses/) 

访问不带选择器的`Service` 与访问有选择器Service相同。流量将被路由到用户定义的端点（在本例中为`1.2.3.4:9376` ）。

ExternalName Service一种特殊的Service：它不带选择器，也不定义任何端口或端点。相反，它可以作为将别名返回到外部Service的一种方式。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

当查找主机 `my-service.prod.svc.CLUSTER` 时，集群DNS服务将会返回一条值为`my.database.example.com`的`CNAME` 记录。访问这样的Service与访问其他Service的工作方式相同，唯一的区别是重定向发生在DNS级别，并且不会发生代理或转发。如果您以后决定将数据库移动到集群中，可启动对应的Pod，添加适当的选择器或Endpoint并更改服务的`type` 。





## Virtual IPs and service proxies（虚拟IP与Service代理）

Kubernetes集群中的每个Node都运行着一个`kube-proxy` 。 `kube-proxy`负责为`Service`实现一个虚拟IP形式，`ExternalName` 类型的Service除外。在Kubernetes v1.0中，使用userspace模式进行代理。在Kubernetes v1.1中，添加了iptables代理，但不是默认的操作模式。从Kubernetes v1.2开始，默认使用iptables代理。

从Kubernetes v1.0起， `Services` 是“4层”（TCP / UDP over IP）结构。在Kubernetes v1.1中，添加了`Ingress` API（beta）来表示“7层”（HTTP）服务。



### Proxy-mode: userspace（代理模式：userspace）

在这种模式下，kube-proxy会监视Kubernetes Master对`Service` 和`Endpoints` 对象的添加和删除。对于每个`Service` ，它将在本地Node上打开一个端口（随机选择），记为“代理端口”。与该“代理端口”的任何连接将被代理到`Service` 的其中一个后端`Pod` （由 `Endpoint` 报告）。使用哪个后端`Pod` 是根据Service的`SessionAffinity` 决定的。 最后，它安装iptables规则，捕获到`Service` 的`clusterIP:Port` 的流量，并将流量重定向到代理端口，代理端口再代理后端`Pod` 。

这样，任何绑定`Service IP:端口` 的流量都被代理到合适的后端，而客户端不需要知道有关Kubernetes或`Service` 或`Pod` 的任何内容。

默认情况下，后端使用算法轮询选择Pod。要想实现基于客户端IP的会话亲和性（Client-IP based session affinity），可将`service.spec.sessionAffinity` 设置为`"ClientIP"` （默认为`"None"` ）；此时，可通过`service.spec.sessionAffinityConfig.clientIP.timeoutSeconds` 字段设置最大会话粘性时间（默认值为“10800”）

![](images/services-userspace-overview.svg)



### Proxy-mode: iptables（代理模式：iptables）

在这种模式下，kube-proxy会监视Kubernetes Master对`Service` 和`Endpoints` 对象的添加和删除。它将对于每个`Service` 安装iptables规则，从而捕获到`Service` 的`clusterIP:Port` 的流量，并将流量重定向到`Service` 的其中一个后端Pod。 对于每个`Endpoint` 对象，它会安装选择后端`Pod` 的iptables规则。

默认情况下，后端使用随机算法的选择Pod。要想实现基于客户端IP的会话亲和性（Client-IP based session affinity），可将`service.spec.sessionAffinity` 设为`"ClientIP"` （默认为`"None"` ）；此时，可通过`service.spec.sessionAffinityConfig.clientIP.timeoutSeconds` 字段设置最大会话粘性时间（默认值为“10800”）。

与用户userspace proxy模式一样，最终，任何绑定`Service IP:端口` 的流量都被代理到合适的后端，而客户端无需有关Kubernetes或`Services` 或`Pods` 的任何内容。这种模式应该比userspace proxy模式更快更可靠。但是，与userspace proxy模式不同，如果最初选择的Pod不响应，iptables代理不能自动重试请求另一个Pod，因此它依赖 [readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#defining-readiness-probes) 。

![](images/services-iptables-overview.svg)



> 译者按：
>
> 1. 简而言之，就是userspace方式，使用kubeproxy来转发；而iptables模式中，kube-proxy只生成相应的iptables规则，转发由iptables实现。
> 2. 拓展阅读：<http://blog.csdn.net/liyingke112/article/details/76022267> 





## Multi-Port Services（多端口Service）

某些`Service` 可能需要暴露多个端口。对于这种情况，Kubernetes支持`Service` 对象上定义多个端口。当使用多个端口时，您必须提供所有端口名称，防止Endpoint产生歧义。 例如：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  - name: https
    protocol: TCP
    port: 443
    targetPort: 9377
```





## Choosing your own IP address（选择自己的IP地址）

`Service` 创建时，可指定自己的Cluster IP。可通过`spec.clusterIP` 字段设置Cluster IP。 例如，如果您想替换一条现有DNS条目，或者有一个配置了固定IP、很难重新配置的遗留系统。用户选择的IP地址必须是合法的IP地址，并且该IP在`service-cluster-ip-range` CIDR范围内，该范围由API Server的标识指定。 如果IP不合法，apiserver将返回一个422 HTTP状态码，表示该值不合法。



### Why not use round-robin DNS?（为什么不使用轮询DNS？）

为什么要使用虚拟IP，而非标准的round-robin DNS呢？有几个原因：

- 长久以来，DNS库都未能实现DNS TTL并缓存域名查询结果。
- 许多应用只执行一次DNS查询，并缓存结果。
- 即使应用程序和DNS库都进行了适当的重新解析，客户端重新解析DNS造成的负载难以管理。

我们试图阻止用户吃力不讨好的事情。也就是说，如果有足够的人要求该功能，我们也可实现该方案作为替代方案。





## Discovering services（服务发现）

Kubernetes主要支持两种方式找到 `Service` ：环境变量和DNS。



### Environment variables（环境变量）

当`Pod` 在`Node` 上运行时，kubelet会为每个活动的`Service` 添加一组环境变量。它支持 [Docker links compatible](https://docs.docker.com/userguide/dockerlinks/) 变量（请参阅 [makeLinkVariables](http://releases.k8s.io/master/pkg/kubelet/envvars/envvars.go#L49) ）和更简单的`{SVCNAME}_SERVICE_HOST` 以及 `{SVCNAME}_SERVICE_PORT` 变量，其中服务名称是大写的，中划线将会转换为下划线。

例如，暴露TCP端口6379，并已分配 Cluster IP地址`10.0.0.11` 的服务 `redis-master` ，将会产生以下环境变量：

```properties
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

*这意味着有顺序的需求* —— `Pod` 想要访问的任何`Service` 必须在该`Pod` 之前创建，否则环境变量将不会被赋值。DNS没有这个限制。



### DNS

DNS server是一个可选的（虽然强烈推荐） [cluster add-on](http://releases.k8s.io/master/cluster/addons/README.md) 。DNS server监视创建新`Services` 的Kubernetes API，并为每个Service创建一组DNS记录。如果在整个集群中启用了DNS，那么所有的`Pod` 都应该能够自动进行`Service` 名称解析。

例如，如果在Kubernetes `my-ns` 这个 `Namespace` 中，有一个名为`my-service` 的Service，就会创建`my-service.my-ns` 的DNS记录。 存在于`"my-ns"` 这个`Namespace`中的`Pod` 可通过`my-service` 这个名称查找来找到这个Service。存在于其他`Namespaces` 的`Pod` 将必须使用`my-service.my-ns` 。使用服务名称查询的结果是Cluster IP。

Kubernetes还支持命名端口的DNS SRV（Service）记录。如果`my-service.my-ns` 这个 `Service` 有一个名为`http` 、协议为`TCP` 的端口，则可以使用 `_http._tcp.my-service.my-ns` 的DNS SRV查询来发现`http` 这个端口的端口号 。

Kubernetes DNS Server是访问`ExternalName` 类型Service的唯一方法。 [DNS Pods and Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) 提供了更多信息。





## Headless Service

有时你不需要或不想使用负载均衡，以及一个单独的Service IP。在这种情况下，您可以通过将Cluster IP（ `spec.clusterIP` ）指定`None` 来创建Headless Service。

此选项允许开发人员通过允许由他们实现自己的服务发现方式，从而减少与Kubernetes系统的耦合。应用程序仍可使用自注册的模式，并且可轻松地将此适配器用于其他的服务发现系统。

对于这类`Services` ，不会分配Cluster IP，kube-proxy也不会处理它们，并且平台也不会对它们进行负载均衡或代理。如何自动配置DNS，取决于Service是否定义了选择器。



### With selectors（有选择器）

对于定义了选择器的Headless Services，Endpoint Controller会在API中创建`Endpoint` 记录，并修改DNS配置以返回`Pods` A记录（地址），A记录直接指向Service后端的Pod。



### Without selectors（无选择器）

对于没有选择器的Headless Service，Endpoint Controller不会创建`Endpoint` 记录。但是，DNS系统会寻找并配置：

- `ExternalName` 类型Service的CNAME记录。
- 所有与Service共享名称的`Endpoints` 的记录，以及其他类型。





## Publishing services - service types（暴露Service——服务类型）

对于应用程序的某些部分（例如前端），您可能希望将服务暴露到一个外部（集群外部）IP中。

Kubernetes `ServiceTypes` 允许您指定所需的Service类型。 默认是`ClusterIP` 。

`Type` 值及其行为如下：

- `ClusterIP` ：通过集群内部的IP暴露Service，Service只能从集群内部访问。这是默认的`ServiceType` 。
- `NodePort` ：在每个Node的IP的静态端口上暴露该Service。NodePort Service会路由到 ClusterIP Service。NodePort类型的集群外的请求可通过 `<NodeIP>:<NodePort>` 与访问一个`NodePort` 类型的Service。
- `LoadBalancer` ：使用云提供商提供的负载均衡器外部暴露服务。负载均衡器将会路由`NodePort` 和`ClusterIP` 类型的Service。


- `ExternalName` ：通过返回带有其值的`CNAME` 记录，将Service映射到`externalName` 字段的内容（例如`foo.bar.example.com` ）。没有任何代理设置。这需要`kube-dns` 1.7或更高版本。





### Type NodePort（NodePort类型）

如果将`type` 字段设为`NodePort` ，则Kubernetes Master将会从给定的配置范围（默认30000-32767）分配一个端口，并且每个Node都会将该端口（每个Node上的同一端口）代理到`Service` 。该端口将在`Service` 的`spec.ports[*].nodePort` 字段中报告。

如果您想要一个特定的端口号，可在`nodePort` 字段的值，系统会为您分配该端口，否则API事务将会失败（即：您需要自己关心可能的端口冲突）。您指定的值必须在Node端口的配置范围内（默认30000-32767）。

这使开发人员可自由设置自己的负载均衡器，配置Kubernetes不完全支持的环境，甚至直接暴露一个或多个Node的IP。

请注意，此Service将在 `<NodeIP>:spec.ports[*].nodePort` 和 `spec.clusterIp:spec.ports[*].port` 可见。



### Type LoadBalancer（LoadBalancer类型）

略



### External IPs（外部IP）

如果外部IP路由到一个或多个集群Node上，则可在这些`externalIPs` 上暴露Kubernetes Service。使用外部IP：端口（作为目标IP）进入集群的流量，将被路由到其中一个Service Endpoint。`externalIPs` 不由Kubernetes管理，这属于集群管理员的职责。

在ServiceSpec中，`ServiceTypes` 的Service都可指定`externalIPs` 。 如下示例中，`my-service` 可通过80.11.12.10:80访问（externalIP:port）

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```





## Shortcomings（缺点）

使用VIP的userspace proxy可在中小规模的情况下工作，但无法扩容到有数千个Service的非常大的集群。有关详细信息，请参阅 [the original design proposal for portals（门户网站的原始设计方案）](http://issue.k8s.io/1107) 。

使用userspace proxy会隐藏访问`Service` 的数据包的源IP。这使得某些防火墙无法实现。iptables proxier不会隐藏群集内的源IP，但它仍会影响到使用负载均衡器或node-port的客户端。

> 译者按：但它仍会影响到使用负载均衡器或node-port的客户端是什么意思？

`Type` 字段支持嵌套——每个级别添加之前级别的功能。不会严格要求所有云提供商这么玩（例如Google Compute Engine不需要分配`NodePort` 来使`LoadBalancer` 工作，但AWS），但当前API是强制要求的。





## Future work（未来的工作）

将来，代理策略可能会比简单的round-robin均衡策略更加的细致，例如Master选举或分片。`Services` 也将会拥有“真正的”负载均衡器，在这种情况下，VIP只会将数据包传输到负载均衡器上。

我们打算改进对L7（HTTP） `Service` 的支持。

我们打算为`Service` 提供更灵活的入口模式，包括当前的`ClusterIP` 、 `NodePort` 和`LoadBalancer` 模式等。





## The gory details of virtual IPs（虚拟IP血腥的细节）

对于那些只想使用`Service` 的人来说，之前的信息应该已经足够。不过，幕后还有很多值得我们理解的事情。



### Avoiding collisions（避免冲突）

Kubernetes的主要哲学之一是，用户不应被暴露在那些“可能会导致他们操作失败，但又不是他们的过错”的场景。在这种情况下，我们来检查网络端口——用户不应该必须选择端口，如果该选择可能与其他用户发生冲突。 那就是“隔离失败”。

> 译者按：笔者理解的“隔离失败”，指的是尽管两个用户之间是隔离的，但仍可能发生失败。

为了允许用户为其`Service` 选择端口，我们必须确保两个`Service` 不会相冲突。我们通过为每个`Service` 分配自己的IP地址的方式来实现。

为了确保每个Service接收到唯一的IP，内部分配器将在创建每个Service之前，以原子方式更新etcd中的全局分配映射表。映射表对象必须存在于服务的注册表中，从而获取IP，否则创建将失败，并显示无法分配IP的消息。后台Controller负责创建该映射表（老版本使用的是内存锁），并检查由于管理员干预而造成的无效分配，并清理那些已经分配但尚无Service使用的IP。



### IPs and VIPs

与实际路由到固定目的地的`Pod` IP地址不同， `Service` IP实际上不由单个主机来应答。相反，我们使用`iptables` （Linux中的数据包处理逻辑）定义虚拟IP，从而根据需要透明地进行重定向。当客户端连接到VIP时，其流量将自动传输到适当的端点。Service的环境变量和DNS实际上是根据`Service` 的VIP和端口填写的。

我们支持两种代理模式——userspace和iptables，运行方式略有不同。

#### Userspace

作为示例，考虑上面提到的图像处理应用。创建后端`Service` ，Kubernetes Master分配虚拟IP地址，例如`10.0.0.1` 。 假设`Service` 端口为1234， `Service` 由集群中的所有`kube-proxy` 实例观察。 当kube-proxy看到一个新的`Service` ，它会随机打开一个新的端口，建立从VIP到这个新端口的iptables重定向，然后开始接受连接。

当客户端连接到VIP时，iptables规则开始起作用，并将数据包重定向到`Service proxy` 的端口。`Service proxy` 选择一个后端节点，并开始代理从客户端到后端的流量。

这意味着`Service` 所有者可选择他们想要的任何端口，没有冲突的风险。客户端可简单地连接到IP和端口，而无需知道它们正在访问哪个`Pod` 。

#### Iptables

再次考虑上述图像处理应用。创建后端`Service` ，Kubernetes Master分配虚拟IP地址，例如`10.0.0.1` 。 假设`Service` 端口为1234， `Service` 由集群中的所有`kube-proxy` 实例观察。当代理看到一个新的`Service` 时，它会安装一系列从VIP重定向到per-`Service` 的iptables规则。 per-`Service` 的规则链接到per-`Endpoint` 规则，per-`Endpoint` 规则会重定向（目标NAT）到后端。

当客户端连接到VIP时，iptables规则就会起作用。选择一个后端（基于会话亲和性或随机），并将数据包重定向到后端。与userspace proxy不同，数据包不会复制到userspace，因此无需运行kube-proxy才能使VIP工作，客户端IP也不会被更改。

当流量通过node-port或负载均衡器进入时，同样的基本流程就会执行，尽管在这种情况下，客户端IP会被改变。





## API Object

Service是Kubernetes REST API中的顶级资源。 有关API对象的更多详细信息，请访问： [Service API object](https://kubernetes.io/docs/api-reference/v1.8/#service-v1-core) 。





## For More Information

阅读 [Connecting a Front End to a Back End Using a Service](https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/) 。





## 原文

<https://kubernetes.io/docs/concepts/services-networking/service/> 