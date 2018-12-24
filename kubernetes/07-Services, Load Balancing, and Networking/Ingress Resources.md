# Ingress Resources



**术语**

在本文中，您将看到有些术语，这些术语在其他地方往往被交叉使用。这可能会导致混淆。本节试图澄清它们。

- Node：Kubernetes集群中的单个虚拟机或物理机。
- Cluster：一组位于互联网防火墙之后的Node，这是Kubernetes管理的主要计算资源。
- Edge router：为集群强制执行防火墙策略的路由器。这可能是由cloud provider或物理硬件管理的网关。
- Cluster network：一组逻辑或物理链接，可根据 [Kubernetes networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/) 实现集群内的通信。集群网络的示例包括诸如 [flannel](https://github.com/coreos/flannel#flannel) 的Overlay网络或诸如 [OVS](https://kubernetes.io/docs/admin/ovs-networking/) 的SDN网络。
- Service：Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 使用Label Selector识别一组Pod。除非另有说明，否则Service假定在集群网络内仅可通过虚拟IP进行路由。





## What is Ingress?

通常，Service和Pod的IP只能在集群网络内部访问。所有到达edge router的所有流量会被丢弃或转发到其他地方。在概念上，这可能看起来像：

```
    internet
        |
  ------------
  [ Services ]
```

Ingress是允许入站连接到达集群Service的规则的集合。

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

可配置Ingress，从而提供外部可访问的URL、负载均衡流量、SSL、提供基于名称的虚拟主机等。用户通过POST Ingress资源到API Server的方式来请求Ingress。 [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) 负责实现Ingress，通常使用负载均衡器，也可配置edge router或其他前端，这有助于以HA方式处理流量。





## Prerequisites（先决条件）

在开始使用Ingress资源之前，您应该了解一些事情。 Ingress是一个Beta资源，在1.1之前的任何Kubernetes版本中都不可用。您需要一个Ingress Controller才能满足Ingress，单纯创建Ingress不起作用。

GCE/GKE会在Master上部署一个Ingress Controller。您可以在Pod中部署任意数量的自定义Ingress Controller。您必须使用适当的类去为每个Ingress添加Annotation，如此 [here](https://git.k8s.io/ingress#running-multiple-ingress-controllers) 和 [here](https://git.k8s.io/ingress-gce/BETA_LIMITATIONS.md#disabling-glbc) 所示。

确保您看过此Controller的 [beta limitations](https://github.com/kubernetes/ingress-gce/blob/master/BETA_LIMITATIONS.md#glbc-beta-limitations) 。在GCE/GKE以外的环境中，您需要 [deploy a controller](https://git.k8s.io/ingress-nginx/README.md) 为pod。





## The Ingress Resource

最简单的Ingress可能如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```

*如果您尚未配置Ingress Controller，那么将其发送到API Server将不起作用。*

**1-6行** ：与所有其他Kubernetes配置一样，Ingress需要 `apiVersion` 、 `kind` 和 `metadata` 字段。有关使用配置文件的信息，请参阅 [deploying applications](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) 、 [configuring containers](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/) 、 [managing resources](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/) 以及 [ingress configuration rewrite](https://github.com/kubernetes/ingress-nginx/blob/master/examples/rewrite/README.md#rewrite) 。

**7-9行** ：Ingress [spec](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) 有配置负载均衡器或代理服务器所需的所有信息。 最重要的是，它包含了一个匹配所有入站请求的rule列表。 目前，Ingress资源只支持http rule。

**10-11行** ：每个http rule包含以下信息：host（例如：foo.bar.com，本例中默认为*）；path列表（例如：/ testpath），每个path都有关联的backend（比如test:80）。 在负载均衡器将流量导入到backend之前，host和path都必须都与入站请求的内容匹配。

**行12-14** ：backend是  [services doc](https://kubernetes.io/docs/concepts/services-networking/service/) 描述的 service:port 组合。 Ingress流量通常直接发送到与backend匹配的端点。

**全局参数** ：简单起见，Ingress示例没有全局参数，要查看资源的完整定义，请阅读 [API reference](https://releases.k8s.io/master/staging/src/k8s.io/api/extensions/v1beta1/types.go) 。可指定全局缺省backend，在所有请求都无法与spec中path匹配的情况下，会发送给缺省backend。





## Ingress controllers

为使Ingress资源正常工作，集群必须运行Ingress Controller。这与其他类型的Controller不同，它们通常作为`kube-controller-manager` 二进制文件的一部分运行，通常作为集群创建的一部分自动启动。 您需要选择最适合您集群的Ingress Controller实现，或自己实现一个。我们目前支持和维护 [GCE ](https://git.k8s.io/ingress-gce/README.md) 和 [nginx](https://git.k8s.io/ingress-nginx/README.md) Controller。





## Before you begin

以下文档描述了通过Ingress资源暴露的一组跨平台功能。 理想情况下，所有Ingress Controller都应符合此规范，但还没有实现。 GCE和nginx Controller的文档分别 [here](https://git.k8s.io/ingress-gce/README.md) 和 [here](https://git.k8s.io/ingress-nginx/README.md) 。**确保您查看Controller的文档，以便您了解每个文档的注意事项** 。





## Types of Ingress



### Single Service Ingress

现有的Kubernetes概念允许您暴露单个Service（请参阅 [alternatives](https://kubernetes.io/docs/concepts/services-networking/ingress/#alternatives) ），但是您也可通过Ingress来暴露，通过指定不带rule的*默认backend*来实现。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```

使用 `kubectl create -f`创建它，即可看到：

```shell
$ kubectl get ing
NAME                RULE          BACKEND        ADDRESS
test-ingress        -             testsvc:80     107.178.254.228
```

其中， `107.178.254.228`是由Ingress Controller为此Ingress分配的IP。  `RULE` 列显示发送到IP的所有流量都被转发到在BACKEND列所列出的Kubernetes服务。



### Simple fanout

如上文所述，kubernetes Pod的IP只能在集群网络可见，所以我们需要一些边缘组件，边缘接受Ingress流量并将其代理到正确的端点。 该组件通常是高可用的负载均衡器。 Ingress允许您将负载均衡器的数量降至最低，例如：如果你想创建类似如下的配置：

```
foo.bar.com -> 178.91.123.132 -> / foo    s1:80
                                 / bar    s2:80
```

那么将需要一个Ingress，例如：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

当您使用 `kubectl create -f` 创建Ingress时：

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -
          foo.bar.com
          /foo          s1:80
          /bar          s2:80
```

只要存在Service（s1，s2），Ingress Controller就会提供满足Ingress的特定负载均衡器的实现。完成这个步骤后，您将在Ingress的最后一列看到负载均衡器的地址。



### Name based virtual hosting（基于名称的虚拟主机）

Name-based virtual hosts为相同IP使用多个主机名。

```
foo.bar.com --|                 |-> foo.bar.com s1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com s2:80
```

以下Ingress告诉后端负载均衡器根据 [Host header](https://tools.ietf.org/html/rfc7230#section-5.4) 进行路由请求。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```

**默认后端** ：一个没有rule的Ingress，如上一节所示，所有流量都会发送到一个默认的backend。您可以使用相同的技术，通过指定一组rule和默认backend，来告诉负载均衡器找到您网站的404页面。如果您的Ingress中的所有Host都无法与请求头中的Host匹配，或者，没有path与请求的URL匹配，则流量将路由到您的默认backend。



### TLS

您可以通过指定包含TLS私钥和证书的 [secret](https://kubernetes.io/docs/user-guide/secrets) 来加密Ingress。目前，Ingress仅支持单个TLS端口443，并假定TLS termination（TLS终止）。如果Ingress中的TLS配置部分指定了不同的host，那么它们将根据通过SNI TLS扩展指定的主机名复用同一端口（如果你提供的Ingress Controller支持SNI）。TLS secret必须包含名为 `tls.crt` 和 `tls.key` 的密钥，其中包含用于TLS的证书和私钥，例如：

```yaml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret
  namespace: default
type: Opaque
```

在Ingress中引用这个secret会告诉Ingress Controller，使用TLS加密从客户端到负载均衡器器的channel：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec:
  tls:
  - secretName: testsecret
  backend:
    serviceName: s1
    servicePort: 80
```

请注意，不同Ingress Controller支持的TLS功能存在差距。请参阅有关 [nginx](https://git.k8s.io/ingress-nginx/README.md#https) 、 [GCE](https://git.k8s.io/ingress-gce/README.md#frontend-https) 或任何其他平台特定的Ingress Controller的文档，以了解TLS在您的环境中的工作原理。



### Loadbalancing

Ingress Controller启动时，会带上适用于所有Ingress的负载均衡策略设置，例如负载均衡算法、后端权重方案等。 更高级的负载均衡概念（例如：持续会话、动态权重）尚未通过Ingress公开。您仍然可以通过 [service loadbalancer](https://git.k8s.io/contrib/service-loadbalancer) 获得这些功能。随着时间的推移，我们计划将跨平台适用的负载均衡模式提取到Ingress资源中。

另外，尽管健康检查不直接通过Ingress暴露，但是在Kubernetes中存在parallel（并行）的概念，例如 [readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) ，可让您实现相同的最终结果。请查看特定Controller的文档，以了解他们如何处理健康检查（ [nginx](https://git.k8s.io/ingress-nginx/README.md) 、 [GCE](https://git.k8s.io/ingress-gce/README.md#health-checks) ）。



## Updating an Ingress

如果您想将新的Host添加到现有Ingress中，可通过编辑资源进行更新：

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -                       178.91.123.132
          foo.bar.com
          /foo          s1:80
$ kubectl edit ing test
```

将会弹出一个编辑器，让你修改现有yaml，修改它，添加新的Host：

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
        path: /foo
  - host: bar.baz.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
        path: /foo
..
```

保存yaml，就会更新API Server中的资源，这将会告诉Ingress Controller重新配置负载均衡器。

```shell
$ kubectl get ing
NAME      RULE          BACKEND   ADDRESS
test      -                       178.91.123.132
          foo.bar.com
          /foo          s1:80
          bar.baz.com
          /foo          s2:80
```

在一个被修改过的Ingress yaml文件上，调用 `kubectl replace -f` 也可实现相同的效果。







## Failing across availability zones（跨可用区的故障）

不同cloud provider之间，跨故障域的流量传播技术有所不同。有关详细信息，请查看Ingress Controller相关的文档。 有关在federated cluster中部署Ingress的详细信息，请参阅federation [doc](https://kubernetes.io/docs/concepts/cluster-administration/federation/) 。





## Future Work

- 各种模式的HTTPS/TLS支持（例如：SNI、re-encryption）
- 通过声明请求IP或主机名
- 结合L4和L7 Ingress
- 更多Ingress Controllers

请追踪 [L7 and Ingress proposal](https://github.com/kubernetes/kubernetes/pull/12827) （L7和Ingress提案），了解有关资源演进的更多细节，以及 [Ingress repository](https://github.com/kubernetes/ingress/tree/master) ，了解有关各种Ingress Controller演进的更多细节。





## Alternatives（备选方案）

有多种方式暴露Service，而无需直接涉及Ingress资源：

- 使用 [Service.Type=LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#type-loadbalancer) 
- 使用 [Service.Type=NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) 
- 使用 [Port Proxy](https://git.k8s.io/contrib/for-demos/proxy-to-service) 
- 部署 [Service loadbalancer](https://git.k8s.io/contrib/service-loadbalancer) 。这允许您在多个Service之间共享一个IP，并通过Service Annotation实现更高级的负载均衡。





## 原文

<https://kubernetes.io/docs/concepts/services-networking/ingress/> 

