# Docker容器网络

本节概述了Docker默认的网络行为，包括默认情况下创建的网络类型以及如何创建用户自定义网络。 本文也描述了在单个主机或集群上创建网络所需的资源。

有关Docker如何在Linux主机上与`iptables`进行交互的详细信息，请参阅[Docker和`iptables`](https://docs.docker.com/engine/userguide/networking/#docker-and-iptables) 。



## 默认网络

当您安装Docker时，它会自动创建三个网络，可使用`docker network ls`命令列出这些网络：

```
$ docker network ls

NETWORK ID          NAME                DRIVER
7fca4eb8c647        bridge              bridge
9f904ee27bf5        none                null
cf03ee007fb4        host                host
```

Docker内置如上三个网络。 运行容器时，可使用`--network` 标志来指定容器应连接到哪些网络。

`bridge` 网络代表所有Docker安装中存在的`docker0` 网络。 除非您使用`docker run --network=<NETWORK>` 选项，否则Docker守护程序默认将容器连接到此网络。 可使用`ip addr show` 命令（或简写形式， `ip a` ），xia显示该网桥的信息。 （ `ifconfig`命令已被弃用，根据系统的不同，还可能会`command not found`错误。）

```
$ ip addr show

docker0   Link encap:Ethernet  HWaddr 02:42:47:bc:3a:eb
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:47ff:febc:3aeb/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1100 (1.1 KB)  TX bytes:648 (648.0 B)
```

> **如何在Docker for Mac或Docker for Windows上运行？**
>
> 如果您使用Docker for Mac（或在Docker for Windows上运行Linux容器）， `docker network ls` 命令将按照上述方式工作，但`ip addr show` 和`ifconfig` 命令可能会展示结果，但会给你本地主机的IP地址信息，而不是Docker容器网络。 这是因为Docker使用虚拟机中运行的网卡，而并非在宿主机的网卡。
>
> 要使用`ip addr show`或`ifconfig`命令浏览Docker网络，请前往[Docker Machine](https://docs.docker.com/machine/overview/) 查看相关文档；如您使用的是云提供商，如AWS上的[Docker Machine](https://docs.docker.com/machine/examples/ocean/)或Digital Ocean上的[Docker Machine](https://docs.docker.com/machine/examples/ocean/) 。可使用`docker-machine ssh <machine-name>` 登录到本地或云托管的机器，也可根据云提供商站点上的描述，直接`ssh` 。

`none` 网络将容器添加到容器特定的网络，该容器缺少网卡。Attach到一个网络为`none` 模式的容器，将会看到类似如下的内容：

```
$ docker attach nonenetcontainer

root@0cb243cd1293:/# cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
root@0cb243cd1293:/# ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@0cb243cd1293:/#
```

> **注意** ：可使用`CTRL-p CTRL-q` 断开容器连接并离开。

`host` 网络模式将容器添加到在宿主机的网络栈上。就网络而言，宿主机和容器之间没有隔离。例如，如果您使用`host` 网络运行在80端口上运行一个Web服务器容器，则该容器可在宿主机的80端口上使用。

在Docker中，`none` 和 `host` 网络模式不能直接配置。 但是，您可以配置默认的`bridge` 网络，以及用户自定义的网桥。

## 默认网桥

所有Docker主机上都有默认的`bridge` 网络。 如不指定网络，容器将自动连接到默认的`bridge` 网络。

`docker network inspect`命令返回有关网络的信息：

```
$ docker network inspect bridge

[
   {
       "Name": "bridge",
       "Id": "f7ab26d71dbd6f557852c7156ae0574bbf62c42f539b50c8ebde0f728a253b6f",
       "Scope": "local",
       "Driver": "bridge",
       "IPAM": {
           "Driver": "default",
           "Config": [
               {
                   "Subnet": "172.17.0.1/16",
                   "Gateway": "172.17.0.1"
               }
           ]
       },
       "Containers": {},
       "Options": {
           "com.docker.network.bridge.default_bridge": "true",
           "com.docker.network.bridge.enable_icc": "true",
           "com.docker.network.bridge.enable_ip_masquerade": "true",
           "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
           "com.docker.network.bridge.name": "docker0",
           "com.docker.network.driver.mtu": "9001"
       },
       "Labels": {}
   }
]
```

运行以下两个命令启动两个`busybox` 容器，两个容器都连接到默认的`bridge` 网络。

```
$ docker run -itd --name=container1 busybox

3386a527aa08b37ea9232cbcace2d2458d49f44bb05a6b775fba7ddd40d8f92c

$ docker run -itd --name=container2 busybox

94447ca479852d29aeddca75c28f7104df3c3196d7b6d83061879e339946805c
```

启动两个容器后再检查`bridge` 网络。 这两个`busybox`容器都连接到网络。 可看到类似如下的结果：

```
$ docker network inspect bridge

{[
    {
        "Name": "bridge",
        "Id": "f7ab26d71dbd6f557852c7156ae0574bbf62c42f539b50c8ebde0f728a253b6f",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.17.0.1/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Containers": {
            "3386a527aa08b37ea9232cbcace2d2458d49f44bb05a6b775fba7ddd40d8f92c": {
                "EndpointID": "647c12443e91faf0fd508b6edfe59c30b642abb60dfab890b4bdccee38750bc1",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "94447ca479852d29aeddca75c28f7104df3c3196d7b6d83061879e339946805c": {
                "EndpointID": "b047d090f446ac49747d3c37d63e4307be745876db7f0ceef7b311cbba615f48",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "9001"
        },
        "Labels": {}
    }
]
```

连接到默认`bridge` 网络的容器可通过IP地址进行通信。 **Docker不支持在默认网桥上自动发现服务。如果您希望容器能够通过容器名称来解析IP地址，那么可使用用户自定义网络** 。您可以使用遗留的`docker run --link` 选项将两个容器连接在一起，但在大多数情况下不推荐使用。

您可以`attach` 到正在运行的容器，查看容器内部的IP是什么。 

```
$ docker attach container1

root@3386a527aa08:/# ifconfig

eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:16 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1296 (1.2 KiB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

从容器内部，使用`ping` 命令测试与其他容器的网络连接。

```
root@3386a527aa08:/# ping -w3 172.17.0.3

PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.096 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.074 ms

--- 172.17.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.074/0.083/0.096 ms
```

使用`cat`命令查看容器上的`/etc/hosts` 文件。 该命令显示容器识别的主机名和IP地址。

```
root@3386a527aa08:/# cat /etc/hosts

172.17.0.2	3386a527aa08
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
```

要从`container1` 容器离开，并保持容器的运行，请依次使用**CTRL-p CTRL-q** 。 如果你愿意，也可attch到`container2` ，并重复上面的命令。

默认的`docker0` 桥接网络支持使用端口映射和`docker run --link` ，以便在`docker0` 网络中的容器之间进行通信。 不推荐这种方法。 如果可以，请使用[用户定义的桥接网络](hhttps://docs.docker.com/engine/userguide/networking/#user-defined-networks)。

## 用户自定义的网络

建议使用用户自定义网桥来控制哪些容器可以相互通信，这样也可启用自动DNS去解析容器名称到IP地址。 Docker提供了创建这些网络的默认**网络驱动程序**。您可以创建一个新的**桥接网络**， **覆盖网络**或**MACVLAN网络** 。 您还可以创建一个**网络插件**或**远程网络**进行完整的自定义和控制。

**您可以根据需要创建任意数量的网络，并且可在任意时间将容器连接到这些网络中的零个或多个**。 此外，您可以将运行着的容器连接或断开网络，而无需重启容器。当容器连接到多个网络时，**其外部连接通过第一个非内部网络以词汇顺序提供**。

接下来的几节将详细介绍Docker的内置网络驱动程序。

### 网桥网络

`bridge` 网络是Docker中最常见的网络类型。 桥接网络类似于默认的`bridge` 网络，但添加一些新功能并删除一些旧的能力。 以下示例创建了桥接网络，并对这些网络上的容器执行一些实验。

```
$ docker network create --driver bridge isolated_nw

1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b

$ docker network inspect isolated_nw

[
    {
        "Name": "isolated_nw",
        "Id": "1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1/16"
                }
            ]
        },
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

$ docker network ls

NETWORK ID          NAME                DRIVER
9f904ee27bf5        none                null
cf03ee007fb4        host                host
7fca4eb8c647        bridge              bridge
c5ee82f76de3        isolated_nw         bridge
```

创建网络后，您可以使用`docker run --network=<NETWORK>` 选项启动容器。

```
$ docker run --network=isolated_nw -itd --name=container3 busybox

8c1a0a5be480921d669a073393ade66a3fc49933f08bcc5515b37b8144f6d47c

$ docker network inspect isolated_nw
[
    {
        "Name": "isolated_nw",
        "Id": "1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {}
            ]
        },
        "Containers": {
            "8c1a0a5be480921d669a073393ade66a3fc49933f08bcc5515b37b8144f6d47c": {
                "EndpointID": "93b2db4a9b9a997beb912d28bcfc117f7b0eb924ff91d48cfa251d473e6a9b08",
                "MacAddress": "02:42:ac:15:00:02",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

您启动到此网络的容器必须驻留在同一个Docker主机上。网络中的每个容器可以立即与其他容器通信。 **虽然网络本身将容器与外部网络隔离开来**。

![](images/bridge_network.png)

在用户定义的桥接网络中，不支持链接（link）。 您可以在此网络中的容器上[暴露和发布容器端口](https://docs.docker.com/engine/userguide/networking/#exposing-and-publishing-ports) 。 如果您希望使一部分`bridge` 网络可用于外部网络，这将非常有用。

![](images/network_access.png)

如果您希望在单个主机上运行相对较小的网络，桥接网络将非常有用。 但是，您可以通过创建`overlay`网络来创建更大的网络。

### `docker_gwbridge`网络

`docker_gwbridge` 是由Docker在两种不同情况下自动创建的本地桥接网络：

- 当您初始化或加入swarm时，Docker会创建`docker_gwbridge` 网络，并将其用于不同主机上swarm节点之间的通信。
- 当容器网络不能提供外部连接时，除了容器的其他网络之外，Docker将容器连接到`docker_gwbridge` 网络，以便容器可以连接到外部网络或其他swarm节点。

如果您需要自定义配置，您可以提前创建`docker_gwbridge` 网络，否则Docker会根据需要创建它。 以下示例使用一些自定义选项创建`docker_gwbridge`网络。

```
$ docker network create --subnet 172.30.0.0/16 \
                        --opt com.docker.network.bridge.name=docker_gwbridge \
			--opt com.docker.network.bridge.enable_icc=false \
			docker_gwbridge
```

当您使用`overlay` 网络时， `docker_gwbridge` 网络始终存在。

### swarm模式下的覆盖网络

当Docker在swarm模式下运行时，您可以在管理节点上创建覆盖网络，而无需外部key-value存储。swarm使覆盖网络仅可用于需要服务的swarm节点。 当您创建使用覆盖网络的服务时，管理节点会自动将覆盖网络扩展到运行服务任务的节点。

要了解有关在swarm模式下运行Docker Engine的更多信息，请参阅[Swarm模式概述](https://docs.docker.com/engine/swarm/) 。

下面的示例显示了如何创建网络并将其用于来自swarm管理节点的服务：

```
$ docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  my-multi-host-network

400g6bwzd68jizzdx5pgyoe95

$ docker service create --replicas 2 --network my-multi-host-network --name my-web nginx

716thylsndqma81j6kkkb5aus
```

只有swarm服务可以连接到覆盖网络，而不是独立的容器。 有关群集的更多信息，请参阅[Docker swarm模式覆盖网络安全模型](https://docs.docker.com/engine/userguide/networking/overlay-security-model/) 以及 [将服务附加到覆盖网络](https://docs.docker.com/engine/swarm/networking/) 。



### 非swarm模式下的覆盖网络

如果您不是在swarm模式下使用Docker Engine，那么`overlay`网络需要有效的key-value存储。 支持的key-value存储包括Consul，Etcd和ZooKeeper（分布式存储）。 在以这种方式创建网络之前，您必须安装并配置您所选择的key-value存储服务。 网络中的Docker宿主机、服务必须能够进行通信。

> **注意** ：以swarm模式运行的Docker Engine与使用外部key-value存储的网络不兼容。

对于大多数Docker用户，不推荐这种使用覆盖网络的方法。它可以与独立的swarm一起使用，可能对在Docker顶部构建解决方案的系统开发人员有用。 将来可能会被弃用。 如果您认为可能需要以这种方式使用覆盖网络，请参阅[本指南](https://docs.docker.com/engine/userguide/networking/get-started-overlay/) 。

### 自定义网络插件

如果任何上述网络机制无法满足您的需求，您可以使用Docker的插件基础架构编写自己的网络驱动插件。 该插件将在运行Docker deamon的主机上作为单独的进程运行。 使用网络插件是一个高级主题。

网络插件遵循与其他插件相同的限制和安装规则。 所有插件都使用插件API，并具有包含了安装，启动，停止和激活的生命周期。

创建并安装自定义网络驱动后，您可以使用`--driver` 标志创建一个使用该驱动的网络。

```
 $ docker network create --driver weave mynet 
```

您可以检查该网络、让容器连接或断开该网络，删除该网络。 特定的插件为特定的需求而生。 检查插件文档的具体信息。 有关编写插件的更多信息，请参阅[扩展Docker](https://docs.docker.com/engine/extend/legacy_plugins/) 以及 [编写网络驱动程序插件](https://docs.docker.com/engine/extend/plugins_network/) 。

### 内嵌DNS服务器

Docker daemon运行一个嵌入式的DNS服务器，从而为连接到同一用户自定义网络的容器之间提供DNS解析——这样，这些容器即可将容器名称解析为IP地址。 如果内嵌DNS服务器无法解析请求，它将被转发到为容器配置的任意外部DNS服务器。 **为了方便，当容器创建时，只有`127.0.0.11` 可访问的内嵌DNS服务器会列在容器的`resolv.conf`文件中。** 有关在用户自定义网络的内嵌DNS服务器的更多信息，请参阅用户定义网络中的[内嵌DNS服务器](https://docs.docker.com/engine/userguide/networking/configure-dns/)



## 暴露和发布端口

在Docker网络中，有两种不同的机制可以直接涉及网络端口：暴露端口和发布端口。 这适用于默认网桥和用户定义的网桥。

- 您使用`Dockerfile` 中的`EXPOSE` 关键字或 `docker run` 命令中的`--expose` 标志来暴露端口。 **暴露端口是记录使用哪些端口，但实际上并不映射或打开任何端口的一种方式**。 暴露端口是可选的。

- 您可以使用`Dockerfile` 中的`PUBLISH` 关键字或 `docker run` 命令中的`--publish`标志来发布端口。 这告诉Docker在容器的网络接口上打开哪些端口。当端口发布时，它将映射到宿主机上可用的高阶端口（高于`30000` ），除非您在运行时指定要映射到宿主机的哪个端口。 您不能在Dockerfile中指定要映射的端口，因为无法保证端口在运行image的宿主机上可用。

  此示例将容器中的端口80发布到宿主机上的随机高阶端口（在这种情况下为`32768` ）。 `-d` 标志使容器在后台运行，因此您可以发出`docker ps` 命令。

  ```
  $ docker run -it -d -p 80 nginx

  $ docker ps

  64879472feea        nginx               "nginx -g 'daemon ..."   43 hours ago        Up About a minute   443/tcp, 0.0.0.0:32768->80/tcp   blissful_mclean
  ```

  下一个示例指定80端口应映射到宿主机上的8080端口。 如果端口8080不可用，将失败。

  ```
  $ docker run -it -d -p 8080:80 nginx

  $ docker ps

  b9788c7adca3        nginx               "nginx -g 'daemon ..."   43 hours ago        Up 3 seconds        80/tcp, 443/tcp, 0.0.0.0:8080->80/tcp   goofy_brahmagupta
  ```




## 容器与代理服务器

如果您的容器需要使用HTTP、HTTPS或者FTP代理，你可以使用如下两种方式进行配置：

* 对于Docker 17.07或更高版本，你可以配置Docker客户端从而将代理信息自动传递给容器。
* 对于Docker 17.06或更低版本，你必须在容器内设置环境变量。你可以在构建镜像（这样不太好移植）或启动容器时执行此操作。

### 配置Docker客户端

> **仅限Edge版本** ：此选项仅适用于Docker CE Edge版本。 请参阅[Docker CE Edge](https://docs.docker.com/edge/) 。

1. 在Docker客户端上，在启动容器所使用的用户的主目录中创建或编辑`~/.config.json` 文件。在其中添加如类似下所示的JSON，如果需要，使用`httpsproxy` 或`ftpproxy` 替换代理类型，然后替换代理服务器的地址和端口。 您可以同时配置多个代理服务器。

   您可以通过将`noProxy` 键设置为一个或多个逗号分隔的IP地址或主机名来选择将指定主机或指定范围排除使用代理服务器。 支持使用`*`字符作为通配符，如此示例所示。

   ```
   {
     "proxies":
     {
       "httpProxy": "http://127.0.0.1:3001",
       "noProxy": "*.test.example.com,.example2.com"
     }
   }
   ```

   保存文件。

2. 当您创建或启动新容器时，环境变量将在容器内自动设置。

### 手动设置环境变量

在构建映像时，或在创建或运行容器时使用`--env` 标志，可将下表中的一个或多个变量设置为适当的值。 这种方法使镜像不太可移植，因此如果您使用Docker 17.07或更高版本，则应该配置Docker客户端。

| 变量            | Dockerfile示例                             | `docker run`示例                           |
| ------------- | ---------------------------------------- | ---------------------------------------- |
| `HTTP_PROXY`  | `ENV HTTP_PROXY "http://127.0.0.1:3001"` | `--env HTTP_PROXY "http://127.0.0.1:3001"` |
| `HTTPS_PROXY` | `ENV HTTPS_PROXY "https://127.0.0.1:3001"` | `--env HTTPS_PROXY "https://127.0.0.1:3001"` |
| `FTP_PROXY`   | `ENV FTP_PROXY "ftp://127.0.0.1:3001"`   | `--env FTP_PROXY "ftp://127.0.0.1:3001"` |
| `NO_PROXY`    | `ENV NO_PROXY "*.test.example.com,.example2.com" |` -env NO_PROXY“* .test.example.com，.example2.com”` |                                          |



## 链接

在Docker包含“用户自定义网络”功能之前，您可以使用Docker `--link` 功能来允许容器将另一个容器的名称解析为IP地址，还可以访问你所链接的容器的环境变量。 如果可以，您应该避免使用 `--link` 标志。

当您创建连接时，当您使用默认`bridge` 或用户自定义网桥时，它们的行为会有所不同。 有关详细信息，请参阅默认`bridge`链接功能的[遗留链接](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&ie=UTF8&prev=_t&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=http://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/&usg=ALkJrhj19xH8k_g7aLJ5Euc_TvEJNo2Hkw)以及在[用户自定义网络中链接容器的链接容器](https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks) 。



## Docker和iptables

Linux主机使用内核模块`iptables` 来管理对网络设备的访问，包括路由，端口转发，网络地址转换（NAT）等问题。 Docker会在启动或停止发布端口的容器、创建或修改网络、attach到容器或其他与网络相关的操作时修改`iptables` 规则。

对`iptables` 全面讨论超出了本主题的范围。 要查看哪个`iptables` 规则在任何时间生效，可以使用`iptables -L` 。 如存在多个表，例如`nat` ， `prerouting` 或`postrouting` ，您可以使用诸如`iptables -t nat -L` 类的命令列出特定的表。 有关`iptables` 的完整文档，请参阅[netfilter / iptables](https://netfilter.org/documentation/) 。

通常， `iptables`规则由初始化脚本或守护进程创建，例如`firewalld` 。 规则在系统重新启动时不会持久存在，因此脚本或程序必须在系统引导时执行，通常在运行级别3或直接在网络初始化之后运行。请参阅您的Linux发行版的网络相关的文档，了解如何使`iptables` 规则持续存在。

Docker动态管理Docker daemon、容器，服务和网络的`iptables` 规则。 在Docker 17.06及更高版本中，您可以向名为`DOCKER-USER`的新表添加规则，这些规则会在Docker自动创建任何规则之前加载。 如果您需要在Docker运行之前预先设置需要使用的`iptables` 规则，这将非常有用。



## 相关信息

- [Work with network commands](https://docs.docker.com/engine/userguide/networking/work-with-networks/)
- [Get started with multi-host networking](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)
- [Managing Data in Containers](https://docs.docker.com/engine/tutorials/dockervolumes/)
- [Docker Machine overview](https://docs.docker.com/machine)
- [Docker Swarm overview](https://docs.docker.com/swarm)
- [Investigate the LibNetwork project](https://github.com/docker/libnetwork)




## 原文

<https://docs.docker.com/engine/userguide/networking/>


