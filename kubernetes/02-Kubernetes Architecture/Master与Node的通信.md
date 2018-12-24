# Master与Node的通信

## 概要

本文列出了Master（其实是apiserver）和Kubernetes集群之间的通信路径。目的是允许用户自定义其安装，从而加强网络配置，以便集群可在不受信任的网络（或云提供商上的公共IP）上运行。





## Cluster -> Master

从集群到Master的所有通信路径终止于apiserver（其他Master组件都不是设计来暴露远程服务的）。在典型的部署中，我们会为apiserver配置监听启用了一种或多种形式的客户端 [authentication](https://kubernetes.io/docs/admin/authentication/) 的安全HTTPS端口（443）。应启用一种或多种 [authorization](https://kubernetes.io/docs/admin/authorization/) 形式，特别是允许 [anonymous requests](https://kubernetes.io/docs/admin/authentication/#anonymous-requests) 或 [service account tokens](https://kubernetes.io/docs/admin/authentication/#service-account-tokens) 的情况下

应为Node配置集群的公共根证书，以便安全地连接到apiserver。例如，在默认的GCE部署中，提供给kubelet的客户端凭证采用客户端证书的形式。请参阅用于自动配置kubelet客户端证书的 [kubelet TLS bootstrapping](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/) 。

希望连接到apiserver的Pod可通过服务帐户安全地进行，这样，Kubernetes就会在实例化时，自动将公共根证书和有效的承载令牌注入到该Pod中。 `kubernetes` 服务（在所有namespace中）配置了一个虚拟IP地址（通过kube-proxy）重定向到apiserver上的HTTPS端点。

Master组件通过不安全（未加密或验证）端口与集群apiserver通信。该端口通常仅在Master机器上的localhost接口上公开，以便所有运行在同一台机器上的Master组件可与集群的apiserver进行通信。 随着时间的推移，Master组件将被迁移以使用安全端口进行认证和授权（参见 [#13598](https://github.com/kubernetes/kubernetes/issues/13598) ）。

因此，默认情况下，来自集群（Node，以及Node上运行的Pod）到Master的连接的默认操作模式将被保护，并且可在不可信和/公共网络上运行。





## Master -> Cluster

从Master（apiserver）到集群主要有两个通信路径。一是从apiserver到集群中的每个Node上运行的kubelet进程。二是通过apiserver的proxy功能，从apiserver到任意Node、Pod、或Service。



### apiserver - > kubelet

从apiserver到kubelet的连接用于：

- 获取Pod的日志。
- 通过kubectl连接到运行的Pod。
- 提供kubelet的端口转发功能。

这些连接终止于kubelet的HTTPS端点。默认情况下，apiserver不验证kubelet的证书，这使得连接可能会受到中间人的攻击，并且在不可信/公共网络上运行是不安全的。

要验证此连接，请使用`--kubelet-certificate-authority` 标志，为apiserver提供一个根证书包，用于验证kubelet的证书。

如果不能这样做，如果需要，请在apiiserver和kubelet之间使用 [SSH tunneling](https://kubernetes.io/docs/concepts/architecture/master-node-communication/#ssh-tunnels) ，以避免通过不可信或公共网络进行连接。

最后，应启用 [Kubelet authentication and/or authorization](https://kubernetes.io/docs/admin/kubelet-authentication-authorization/) 来保护kubelet API。



### apiserver -> nodes, pods, and services

从apiserver到Node、Pod或Service的连接默认为纯HTTP连接，因此不会被认证或加密。可通过将`https:` 前缀添加到到API URL中的Node、Pod、Service名称，从而运行安全的HTTPS连接，但不会验证HTTPS端点提供的证书，也不会提供客户端凭据——因此，尽管连接将被加密，它不会提供任何诚信保证。这些连接在不受信任/公共网络上运行**并不安全**。



### SSH隧道

[Google Container Engine](https://cloud.google.com/container-engine/docs/) 使用SSH隧道来保护Master -> 集群的通信路径。在此配置中，apiserver会启动一个SSH隧道到集群中的每个Node（连接到监听端口22的ssh服务器），并通过隧道传递目标为kubelet，Node，Pod或Service的所有流量。该隧道确保流量不会暴露到私有GCE网络以外。



## 原文

<https://kubernetes.io/docs/concepts/architecture/master-node-communication/> 