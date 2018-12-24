# 默认bridge网络中配置DNS

本节描述如何在Docker默认网桥中配置容器DNS。 当您安装Docker时，就会自动创建一个名为`bridge` 的桥接网络。

> **注意** ： [Docker网络功能](https://docs.docker.com/engine/userguide/networking/) 允许您创建除默认网桥之外的用户自定义网络。 有关用户自定义网络中DNS配置的更多信息，请参阅[Docker嵌入式DNS](https://docs.docker.com/engine/userguide/networking/configure-dns/) 部分。

Docker如何为每个容器提供主机名和DNS配置，而无需在构建自定义Docker镜像时在内部写入主机名？它的诀窍是利用可以写入新信息的虚拟文件，在容器内覆盖三个关键的`/etc` 文件。 你可以通过在一个容器中运行`mount` 来看到这一点：

```
root@f38c87f2a42d:/# mount

...
/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/resolv.conf type ext4 ...
...
```

这样一来，Docker可以让宿主机在稍后通过DHCP接收到新的配置后，使所有容器中的`resolv.conf` 保持最新状态。 Docker在容器中维护这些文件的具体细节可能会可能会随着Docker版本的演进而改变，因此您不该自己管理/etc文件，而应该用以下Docker选项。

四个不同的选项会影响容器域名服务。

| 参数                                     | 描述                                       |
| -------------------------------------- | ---------------------------------------- |
| `-h HOSTNAME` or `--hostname=HOSTNAME` | 设置容器的主机名。 该设置的值将会被写入`/etc/hostname` ；写入`/etc/hosts` **作为容器的面向主机IP地址的名称（笔者按：在/etc/hosts里添加一条记录，IP是宿主机可以访问的IP，host就是你设置的host）**，并且是容器内部`/bin/bash` 在其提示符下显示的名称。 但主机名不容易从容器外面看到。 它不会出现在`docker ps` 或任何其他容器的`/etc/hosts` 文件中。 |
| `--link=CONTAINER_NAME`or `ID:ALIAS`   | 在`run` 容器时使用此选项为新容器的`/etc/hosts` 添加了一个名为`ALIAS`的额外条目，指向由`CONTAINER_NAME_or_ID`标识的`CONTAINER_NAME_or_ID`的IP地址。这使得新容器内的进程可以连接到主机名`ALIAS` 而不必知道其IP。 `--link=`选项将在下面进行更详细的讨论。 因为Docker可以在重新启动时为链接的容器分配不同的IP地址，Docker会更新收件人容器的`/etc/hosts` 文件中的`ALIAS`条目。 |
| `--dns=IP_ADDRESS...`                  | 在容器的`/etc/resolv.conf`文件添加`nameserver` 行，IP地址为指定IP。 容器中的进程在如果需要访问`/etc/hosts` 里的主机名，就会连接到这些IP地址的53端口，寻找名称解析服务。 |
| `--dns-search=DOMAIN...`               | 通过在容器的`/etc/resolv.conf `写入`search` 行，在容器内使用裸不合格的主机名时搜索的域名。 当容器进程尝试访问`host` 并且搜索域`example.com` 被设置时，例如，DNS逻辑不仅将查找`host`  ，还将查找`host.example.com` 。使用`--dns-search=.` 如果您不想设置搜索域。 |
| `--dns-opt=OPTION...`                  | 通过将`options` 行写入容器的`/etc/resolv.conf` 设置DNS解析器使用的选项。有关有效选项的列表，请参阅`resolv.conf`文档 |

在没有`--dns=IP_ADDRESS...` ， `--dns-search=DOMAIN...`或`--dns-opt=OPTION...`选项的情况下，**Docker使每个容器的`/etc/resolv.conf` 看起来像宿主机的`/etc/resolv.conf`** 。当创建容器的`/etc/resolv.conf` ，Docker daemon会从主机的原始文件中过滤掉所有localhost IP地址`nameserver` 条目。

过滤是必要的，因为主机上的所有localhost地址都不可从容器的网络中访问。 过滤之后，如果容器的`/etc/resolv.conf` 文件中没有更多的`nameserver` 条目，Docker daemon会将Google DNS名称服务器（8.8.8.8和8.8.4.4）添加到容器的DNS配置中。 如果守护进程启用了IPv6，则也会添加公共IPv6 Google DNS名称服务器（2001:4860:4860::8888 和 2001:4860:4860::8844）。

> **注意** ：如果您需要访问主机的localhost解析器，则必须在主机上修改DNS服务，以便侦听从容器内可访问的non-localhost地址。

您可能会想知道宿主机的`/etc/resolv.conf` 文件发生了什么变化。 `docker daemon` 有一个文件更改通知程序，它将监视主机DNS配置的更改。

> **注意** ：文件更改通知程序依赖于Linux内核的inotify功能。由于此功能目前与overlay文件系统驱动不兼容，因此使用“overlay”的Docker daemon将无法利用`/etc/resolv.conf` 自动更新的功能。

当宿主机文件更改时，所有`resolv.conf` 与主机匹配的**停止的容器将立即更新到最新的主机配置**。 当宿主机配置更改时，**运行的容器将需要停止并开始接收主机更改**，这是由于缺少设备，以确保在容器运行时对`resolv.conf` 文件的原子写入。 如果容器修改了默认的`resolv.conf` 文件，则不会替换该文件，因为如果替换，将会覆盖容器执行的更改。 如果选项（ `--dns` ， `--dns-search` 或`--dns-opt` ）已被用于修改默认的主机配置，则更换主机的`/etc/resolv.conf` 也不会发生。

> **注意** ：对于在Docker 1.5.0中实现`/etc/resolv.conf` 更新功能之前创建的容器：当主机`resolv.conf`文件更改时，这些容器将**不会**收到更新。 只有使用Docker 1.5.0及以上版本创建的容器才能使用此自动更新功能。



## 原文

<https://docs.docker.com/engine/userguide/networking/default_network/configure-dns/> 



## 拓展阅读

Docker存储驱动的选择：<https://docs.docker.com/engine/userguide/storagedriver/selectadriver/#docker-ce>