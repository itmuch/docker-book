# 用户定义网络中的内嵌DNS服务器

本节中的信息涵盖用户自定义网络中的容器的内嵌DNS服务器操作。连接到用户自定义网络的容器的DNS lookup与连接到默认`bridge` 网络的容器的工作机制不同。

> **注意** ：为了保持向后兼容性， 默认` bridge` 网络的DNS配置保持不变， 有关默认网桥中DNS配置的详细信息，请参阅[默认网桥中的DNS](https://docs.docker.com/engine/userguide/networking/default_network/configure-dns/) 。

从Docker 1.10开始，Docker daemon实现了一个内嵌的DNS服务器，它为任何使用有效`name` 、`net-alias` 或使用`link` 别名所创建的容器提供内置的服务发现能力。 Docker如何管理容器内DNS配置的具体细节可随着Docker版本的改变而改变。 所以你不应该自己管理容器内的`/etc/hosts` 、`/etc/resolv.conf` 等文件，而是使用以下的Docker选项。

影响容器域名服务的各种容器选项。

| `--name=CONTAINER-NAME`       | 使用`--name `配置的容器名称用于发现用户自定义网络中的容器。 内嵌DNS服务器维护容器名称及其IP地址（在容器连接的网络上）之间的映射。 |
| ----------------------------- | ---------------------------------------- |
| `--network-alias=ALIAS`       | 除如上所述的`--name` 以外，容器可使用用户自定义网络中的一个或多个`--network-alias` （或`docker network connect` 命令中的`--alias` 选项）发现。 内嵌DNS服务器维护特定用户自定义网络中所有容器别名及IP之间的映射。 通过在 `docker network connect` 命令中使用`--alias` 选项，容器可在不同的网络中具有不同的别名。 |
| `--link=CONTAINER_NAME:ALIAS` | 在`run` 容器时使用此选项为嵌入式DNS提供了一个名为`ALIAS` 的额外条目，指向由`CONTAINER_NAME` 标识的IP地址。 当使用`--link` 时，嵌入式DNS将确保只在使用了`--link` 选项的容器上进行本地化查找。 这允许新容器内的进程连接到容器，而不必知道其名称或IP。 |
| `--dns=[IP_ADDRESS...]`       | 如果嵌入式DNS服务器无法从容器中解析名称、解析请求，嵌入式DNS服务器将使用`--dns` 选项传递的IP地址转发DNS查询。 这些`--dns` IP地址由嵌入式DNS服务器管理，不会在容器的`/etc/resolv.conf` 文件中更新。 |
| `--dns-search=DOMAIN...`      | 当容器内使用主机名不合格时所设置的域名。这些`--dns-search` 选项由嵌入式DNS服务器管理，不会在容器的`/etc/resolv.conf` 文件中更新。当容器进程尝试访问`host` 并且搜索域 `example.com`被设置时，例如，DNS逻辑不仅将查找`host` ，还将查找`host.example.com` 。 |
| `--dns-opt=OPTION...`         | 设置DNS解析器使用的选项。 这些选项由嵌入式DNS服务器管理，不会在容器的`/etc/resolv.conf` 文件中更新。有关有效选项的列表，请参阅`resolv.conf`文档。 |



在没有`--dns=IP_ADDRESS...` ，`--dns-search=DOMAIN...` 或`--dns-opt=OPTION...` 选项的情况下，**Docker使用宿主机的`/etc/resolv.conf`** （ `docker daemon` 运行的地方）。 在执行此操作时，damon会从宿主机的原始文件中过滤出所有localhost IP地址`nameserver` 条目。

过滤是必要的，因为宿主机上的所有localhost地址都不可从容器的网络中访问。过滤之后，如果容器的`/etc/resolv.conf` 文件中没有更多的`nameserver` 条目，daemon会将公共Google DNS名称服务器（8.8.8.8和8.8.4.4）添加到容器的DNS配置中。 如果daemon启用了IPv6，则也会添加公共IPv6 Google DNS名称服务器（2001:4860:4860::8888 以及 2001:4860:4860::8844）。

> **注意** ：如果您需要访问宿主机的localhost解析器，则必须修改宿主机上的DNS服务，以便侦听从容器内可访问的non-localhost地址。

> **注意** ：DNS服务器始终为`127.0.0.11` 。



## 原文

<https://docs.docker.com/engine/userguide/networking/configure-dns/>



## 参考&拓展阅读

Docker内置DNS：<https://jimmysong.io/blogs/docker-embedded-dns/>

Dns: http://blog.csdn.net/waltonwang/article/details/54098592