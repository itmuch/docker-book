# network命令

本文提供可用于与Docker网络及与网络中容器进行交互的network子命令的示例。这些命令可通过Docker Engine CLI获得。 这些命令是：

- `docker network create`
- `docker network connect`
- `docker network ls`
- `docker network rm`
- `docker network disconnect`
- `docker network inspect`

虽然不是必需的，但在尝试本节中的示例之前，先阅读 [了解Docker网络](https://docs.docker.com/engine/userguide/networking/) 更佳。 示例使用默认`bridge` 网络以便您可以立即尝试。要实验`overlay`网络，请参阅 [多主机网络入门指南](https://docs.docker.com/engine/userguide/networking/get-started-overlay/) 。



## 创建网络

Docker Engine在安装时自动创建`bridge` 网络。 该网络对应于Engine传统依赖的`docker0` 网桥。除该网络外，也可创建自己的`bridge` 或`overlay` 网络。

`bridge` 网络驻留在运行Docker Engine实例的单个主机上。 `overlay` 网络可跨越运行Docker Engine的多个主机。 如果您运行`docker network create` 并仅提供网络名称，它将为您创建一个桥接网络。

```
$ docker network create simple-network

69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a

$ docker network inspect simple-network
[
    {
        "Name": "simple-network",
        "Id": "69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

与`bridge` 网络不同， `overlay` 网络需要一些预制条件才能创建——

- 访问key-value存储。 引擎支持Consul，Etcd和ZooKeeper（分布式存储）key-value存储。
- 与key-value存储连接的主机集群。
- 在swarm中的每个主机上正确配置的`Docker daemon` 。

支持`overlay` 网络的`dockerd` 选项有：

- `--cluster-store`
- `--cluster-store-opt`
- `--cluster-advertise`

在创建网络时，Docker引擎默认会为网络创建一个不重叠的子网。 您可以覆盖此默认值，并使用`--subnet` 选项直接指定子网。 对于`bridge ` 网络，只可指定一个子网。 `overlay` 网络支持多个子网。

> **注意** ：强烈建议在创建网络时使用`--subnet` 选项。 如果未指定`--subnet` 则Docker daemon会自动为网络选择并分配子网，这可能会导致与您基础结构中的另一个子网（该子网不受`--subnet` 管理）重叠。 当容器连接到该网络时，这种重叠可能导致连接问题或故障。

除`--subnet` 选项以外，您还可以指定`--gateway` ， `--ip-range` `--gateway` `--ip-range`和`--aux-address`选项。

```
$ docker network create -d overlay \
  --subnet=192.168.0.0/16 \
  --subnet=192.170.0.0/16 \
  --gateway=192.168.0.100 \
  --gateway=192.170.0.100 \
  --ip-range=192.168.1.0/24 \
  --aux-address="my-router=192.168.1.5" --aux-address="my-switch=192.168.1.6" \
  --aux-address="my-printer=192.170.1.5" --aux-address="my-nas=192.170.1.6" \
  my-multihost-network
```

确保您的子网不重叠。 如果重叠，那么网络将会创建失败，Docker Engine返回错误。

创建自定义网络时，您可以向驱动传递其他选项。 `bridge` 驱动程序接受以下选项：

| Option                                   | Equivalent  | Description        |
| ---------------------------------------- | ----------- | ------------------ |
| `com.docker.network.bridge.name`         | -           | 创建Linux网桥时要使用的网桥名称 |
| `com.docker.network.bridge.enable_ip_masquerade` | `--ip-masq` | 启用IP伪装             |
| `com.docker.network.bridge.enable_icc`   | `--icc`     | 启用或禁用跨容器连接         |
| `com.docker.network.bridge.host_binding_ipv4` | `--ip`      | 绑定容器端口时的默认IP       |
| `com.docker.network.driver.mtu`          | `--mtu`     | 设置容器网络MTU          |

`overlay` 驱动也支持`com.docker.network.driver.mtu` 选项。

以下参数可以传递给任何网络驱动的`docker network create` 。

| Argument     | Equivalent | Description |
| ------------ | ---------- | ----------- |
| `--internal` | -          | 限制对网络的外部访问  |
| `--ipv6`     | `--ipv6`   | 启用IPv6网络    |

以下示例使用`-o` 选项，在绑定端口时绑定到指定的IP地址，然后使用`docker network inspect` 来检查网络，最后将新容器attach到新网络。

```
$ docker network create -o "com.docker.network.bridge.host_binding_ipv4"="172.23.0.1" my-network

b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a

$ docker network inspect my-network

[
    {
        "Name": "my-network",
        "Id": "b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.23.0.0/16",
                    "Gateway": "172.23.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.host_binding_ipv4": "172.23.0.1"
        },
        "Labels": {}
    }
]

$ docker run -d -P --name redis --network my-network redis

bafb0c808c53104b2c90346f284bda33a69beadcab4fc83ab8f2c5a4410cd129

$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                        NAMES
bafb0c808c53        redis               "/entrypoint.sh redis"   4 seconds ago       Up 3 seconds        172.23.0.1:32770->6379/tcp   redis
```

## 连接容器

您可以将一个现有容器连接到一个或多个网络。 容器可连接到使用不同网络驱动的网络。 一旦连接，容器即可使用另一个容器的IP地址或名称进行通信。

对于支持多主机连接的`overlay` 网络或自定义插件，不同主机上的容器，只要连接到同一multi-host network多主机网络，也可以这种方式进行通信。

此示例使用六个容器，并指示您根据需要创建它们。













### 基本容器网络示例

1. 首先，创建并运行两个容器， `container1`和`container2` ：

   ```
   $ docker run -itd --name=container1 busybox

   18c062ef45ac0c026ee48a83afa39d25635ee5f02b58de4abc8f467bcaa28731

   $ docker run -itd --name=container2 busybox

   498eaaaf328e1018042c04b2de04036fc04719a6e39a097a4f4866043a2c2152
   ```

2. 创建一个隔离的`bridge` 网络进行测试。

   ```
   $ docker network create -d bridge --subnet 172.25.0.0/16 isolated_nw

   06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8
   ```

3. 将`container2` 连接到网络，然后`inspect`网络以验证连接：

   ```
   $ docker network connect isolated_nw container2

   $ docker network inspect isolated_nw

   [
       {
           "Name": "isolated_nw",
           "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
           "Scope": "local",
           "Driver": "bridge",
           "IPAM": {
               "Driver": "default",
               "Config": [
                   {
                       "Subnet": "172.25.0.0/16",
                       "Gateway": "172.25.0.1/16"
                   }
               ]
           },
           "Containers": {
               "90e1f3ec71caf82ae776a827e0712a68a110a3f175954e5bd4222fd142ac9428": {
                   "Name": "container2",
                   "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
                   "MacAddress": "02:42:ac:19:00:02",
                   "IPv4Address": "172.25.0.2/16",
                   "IPv6Address": ""
               }
           },
           "Options": {}
       }
   ]
   ```

   请注意， `container2` 自动分配了一个IP地址。 因为在创建网络时指定了`--subnet` 选项，所以IP地址会从该子网选择。

   **作为提醒**， `container1` 仅连接到默认`bridge` 。

4. 启动第三个容器，但这次使用`--ip` 标志分配一个IP地址，并使用 `docker run`命令的`--network`选项将其连接到`--isolated_nw`网络：

   ```
   $ docker run --network=isolated_nw --ip=172.25.3.3 -itd --name=container3 busybox

   467a7863c3f0277ef8e661b38427737f28099b61fa55622d6c30fb288d88c551
   ```

   只要您为容器指定的IP地址是如上子网的一部分，那就可使用`--ip` 或`--ip6` 标志将IPv4或IPv6地址分配给容器，将其连接到以上网络。 当您在使用用户自定义的网络时以这种方式指定IP地址时，配置将作为容器配置的一部分进行保留，并在容器重新加载时进行应用。 使用非用户自定义网络时，分配的IP地址将被保留，因为不保证Docker daemon重启时容器的子网不会改变，除非您使用用户定义的网络。【这一段官方文档是不是有问题？？？】

5. 检查`container3` 所使用的网络资源。 简洁起见，截断以下输出。

   ```
   $ docker inspect --format=''  container3

   {"isolated_nw":
     {"IPAMConfig":
       {
         "IPv4Address":"172.25.3.3"},
         "NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
         "EndpointID":"dffc7ec2915af58cc827d995e6ebdc897342be0420123277103c40ae35579103",
         "Gateway":"172.25.0.1",
         "IPAddress":"172.25.3.3",
         "IPPrefixLen":16,
         "IPv6Gateway":"",
         "GlobalIPv6Address":"",
         "GlobalIPv6PrefixLen":0,
         "MacAddress":"02:42:ac:19:03:03"}
       }
     }
   }
   ```

   因为在启动时将`container3` 连接到`isolated_nw`  ，所以它根本没有连接到默认的`bridge` 网络。

6. 检查`container2` 所使用的网络。 如果你安装了Python，你可以打印输出格式化。

   ```
   $ docker inspect --format=''  container2 | python -m json.tool

   {
       "bridge": {
           "NetworkID":"7ea29fc1412292a2d7bba362f9253545fecdfa8ce9a6e37dd10ba8bee7129812",
           "EndpointID": "0099f9efb5a3727f6a554f176b1e96fca34cae773da68b3b6a26d046c12cb365",
           "Gateway": "172.17.0.1",
           "GlobalIPv6Address": "",
           "GlobalIPv6PrefixLen": 0,
           "IPAMConfig": null,
           "IPAddress": "172.17.0.3",
           "IPPrefixLen": 16,
           "IPv6Gateway": "",
           "MacAddress": "02:42:ac:11:00:03"
       },
       "isolated_nw": {
           "NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
           "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
           "Gateway": "172.25.0.1",
           "GlobalIPv6Address": "",
           "GlobalIPv6PrefixLen": 0,
           "IPAMConfig": null,
           "IPAddress": "172.25.0.2",
           "IPPrefixLen": 16,
           "IPv6Gateway": "",
           "MacAddress": "02:42:ac:19:00:02"
       }
   }
   ```

   请注意， `container2` 属于两个网络。 当您启动它时，它加入了默认`bridge` 网络，并在步骤3中将其连接到`isolated_nw` 。

   ![](images/working.png)

   eth0 Link encap:Ethernet HWaddr 02:42:AC:11:00:03

   eth1 Link encap:Ethernet HWaddr 02:42:AC:15:00:02

7. 使用`docker attach` 命令连接到正在运行的`container2` 并检查它的网络堆栈：

   ```
   $ docker attach container2
   ```

   使用`ifconfig` 命令检查容器的网络堆栈。 您应该看到两个以太网卡，一个用于默认`bridge` ，另一个用于`isolated_nw` 网络。

   ```
   $ sudo ifconfig -a

   eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
             inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
             inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
             UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
             RX packets:8 errors:0 dropped:0 overruns:0 frame:0
             TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

   eth1      Link encap:Ethernet  HWaddr 02:42:AC:15:00:02
             inet addr:172.25.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
             inet6 addr: fe80::42:acff:fe19:2/64 Scope:Link
             UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
             RX packets:8 errors:0 dropped:0 overruns:0 frame:0
             TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

   lo        Link encap:Local Loopback
             inet addr:127.0.0.1  Mask:255.0.0.0
             inet6 addr: ::1/128 Scope:Host
             UP LOOPBACK RUNNING  MTU:65536  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
   ```

8. Docker内嵌DNS服务器可使用容器名称解析连接到给定网络的容器。 这意味着网络内的容器可以通过容器名称ping在同一网络中的另一个容器。 例如，从`container2` 可以按名称ping `container3` 。

   ```
   / # ping -w 4 container3
   PING container3 (172.25.3.3): 56 data bytes
   64 bytes from 172.25.3.3: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.3.3: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.3.3: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.3.3: seq=3 ttl=64 time=0.097 ms

   --- container3 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   此功能不适用于默认`bridge` 网络。 `container1`和`container2`都连接到默认的`bridge` 网络，但是并不能使用容器名称从`container2` ping `container1` 。

   ```
   / # ping -w 4 container1
   ping: bad address 'container1'
   ```

   但依然可直接ping IP地址：

   ```
   / # ping -w 4 172.17.0.2
   PING 172.17.0.2 (172.17.0.2): 56 data bytes
   64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.095 ms
   64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.075 ms
   64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.072 ms
   64 bytes from 172.17.0.2: seq=3 ttl=64 time=0.101 ms

   --- 172.17.0.2 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.072/0.085/0.101 ms
   ```

   离开`container2` 容器，并使用`CTRL-p CTRL-q` 保持容器运行。

9. 当前， `container2` 连接到默认`bridge`网络和`isolated_nw` 网络，因此，`container2` 可与`container1` 以及`container3`进行通信。 但是，`container3` 和`container1` 没有任何共同的网络，所以它们不能通信。 要验证这一点，请附加到`container3`并尝试通过IP地址ping `container1` 。

   ```
   $ docker attach container3

   $ ping 172.17.0.2
   PING 172.17.0.2 (172.17.0.2): 56 data bytes
   ^C

   --- 172.17.0.2 ping statistics ---
   10 packets transmitted, 0 packets received, 100% packet loss
   ```

   离开`container3` 容器，并使用`CTRL-p CTRL-q`保持容器运行。

> 即使容器未运行，也可以将容器连接到网络。 但是， `docker network inspect` 仅显示运行容器的信息。









### 链接容器而不使用用户定义的网络

完成基本容器网络示例中的步骤后， `container2` 可以自动解析`container3` 的名称，因为两个容器都连接到`isolated_nw` 网络。 但是，连接到默认`bridge` 的容器无法解析彼此的容器名称。 如果您需要容器能够通过`bridge` 网络进行通信，则需要使用[遗留的连接](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)功能。 这是唯一的建议使用`--link` 的情况。 您应该强烈地考虑使用用户定义的网络。

使用遗留的`link` 标志为可为默认的`bridge` 网络添加以下功能进行通信：

- 将容器名称解析为IP地址的能力
- 使用`--link=CONTAINER-NAME:ALIAS` 定义一个网络别名去连接容器的能力
- 安全的容器连接（通过`--icc=false` 隔离）
- 环境变量注入

需要重申的是，当您使用用户自定义网络时，默认情况下提供所有这些功能，无需额外的配置。 **此外，您可以动态attach到多个网络，也可动态从多个网络中离开。**

- 使用DNS进行自动名称解析
- 支持`--link` 选项为链接的容器提供名称别名
- 网络中容器的自动安全隔离环境
- 环境变量注入

以下示例简要介绍如何使用`--link` 。

1. 继续上面的例子，创建一个新的容器`container4` ，并将其连接到网络`isolated_nw` 。 另外，使用`--link`标志链接到容器`container5` （不存在！）！

   ```
   $ docker run --network=isolated_nw -itd --name=container4 --link container5:c5 busybox

   01b5df970834b77a9eadbaff39051f237957bd35c4c56f11193e0594cfd5117c
   ```

   这有点棘手，因为`container5` 还不存在。 当`container5`被创建时， `container4`将能够将名称`c5` 解析为`container5` 的IP地址。

   > **注意** ：使用遗留的link功能创建的容器之间的任何链接本质上都是静态的，并且通过别名强制绑定容器。 它无法容忍链接的容器重新启动。 用户自定义网络中的新链接功能支持容器之间的动态链接，并且允许链接容器中的重新启动和IP地址更改。

   由于您尚未创建容器`container5` 尝试ping它将导致错误。 attach到`container4`并尝试ping任何`container5`或`c5` ：

   ```
   $ docker attach container4

   $ ping container5

   ping: bad address 'container5'

   $ ping c5

   ping: bad address 'c5'
   ```

   从`container4` 离开，并使用`CTRL-p CTRL-q` 使其保持运行。

2. 创建一个容器，名为`container5` ，并使用别名`c4`将其链接到`container4` 。

   ```
   $ docker run --network=isolated_nw -itd --name=container5 --link container4:c4 busybox

   72eccf2208336f31e9e33ba327734125af00d1e1d2657878e2ee8154fbb23c7a
   ```

   现在attach到`container4` ，尝试ping `c5` 和`container5` 。

   ```
   $ docker attach container4

   / # ping -w 4 c5
   PING c5 (172.25.0.5): 56 data bytes
   64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

   --- c5 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms

   / # ping -w 4 container5
   PING container5 (172.25.0.5): 56 data bytes
   64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

   --- container5 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   从`container4` 分离，并使用`CTRL-p CTRL-q` 使其保持运行。

3. 最后，附加到`container5` ，验证你可以ping `container4` 。

   ```
   $ docker attach container5

   / # ping -w 4 c4
   PING c4 (172.25.0.4): 56 data bytes
   64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.065 ms
   64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.4: seq=2 ttl=64 time=0.067 ms
   64 bytes from 172.25.0.4: seq=3 ttl=64 time=0.082 ms

   --- c4 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.065/0.070/0.082 ms

   / # ping -w 4 container4
   PING container4 (172.25.0.4): 56 data bytes
   64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.065 ms
   64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.4: seq=2 ttl=64 time=0.067 ms
   64 bytes from 172.25.0.4: seq=3 ttl=64 time=0.082 ms

   --- container4 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.065/0.070/0.082 ms
   ```

   从`container5` 离开，并使用`CTRL-p CTRL-q` 使其保持运行。



### 网络范围的别名示例

链接容器时，无论是使用遗留的`link`方法还是使用用户自定义网络，您指定的任何别名只对指定的容器有意义，并且不能在默认`bridge` 上的其他容器上运行。

另外，如果容器属于多个网络，则给定的链接别名与给定的网络范围一致。 因此，容器可以链接到不同网络中的不同别名，并且别名将不适用于不在同一网络上的容器。

以下示例说明了这些要点。

1. 创建另一个名为`local_alias` 网络：

   ```
   $ docker network create -d bridge --subnet 172.26.0.0/24 local_alias
   76b7dc932e037589e6553f59f76008e5b76fa069638cd39776b890607f567aaa
   ```

2. 接下来，使用别名`foo` 和`bar` 将`container4` 和`container5` 连接到新的网络`local_alias` ：

   ```
   $ docker network connect --link container5:foo local_alias container4
   $ docker network connect --link container4:bar local_alias container5
   ```

3. attach到`container4` 并尝试使用别名`foo` ping `container4` （是的，同一个），然后尝试使用别名`c5` ping容器`container5` ：

   ```
    $ docker attach container4

    / # ping -w 4 foo
    PING foo (172.26.0.3): 56 data bytes
    64 bytes from 172.26.0.3: seq=0 ttl=64 time=0.070 ms
    64 bytes from 172.26.0.3: seq=1 ttl=64 time=0.080 ms
    64 bytes from 172.26.0.3: seq=2 ttl=64 time=0.080 ms
    64 bytes from 172.26.0.3: seq=3 ttl=64 time=0.097 ms

    --- foo ping statistics ---
    4 packets transmitted, 4 packets received, 0% packet loss
    round-trip min/avg/max = 0.070/0.081/0.097 ms

    / # ping -w 4 c5
    PING c5 (172.25.0.5): 56 data bytes
    64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
    64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
    64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
    64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

    --- c5 ping statistics ---
    4 packets transmitted, 4 packets received, 0% packet loss
    round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   两个ping都成功了，但子网不同，这意味着网络不同。

   离开`container4`，并使用`CTRL-p CTRL-q` 使其保持运行。

4. 从`isolated_nw` 网络断开`container5` 。 附加到`container4` 并尝试ping `c5` 和`foo` 。

   ```
   $ docker network disconnect isolated_nw container5

   $ docker attach container4

   / # ping -w 4 c5
   ping: bad address 'c5'

   / # ping -w 4 foo
   PING foo (172.26.0.3): 56 data bytes
   64 bytes from 172.26.0.3: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.26.0.3: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.26.0.3: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.26.0.3: seq=3 ttl=64 time=0.097 ms

   --- foo ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   您不能再从`container5` 收到`isolated_nw` 网络上的`container5` 。 但是，您仍然可以使用别名`foo`到达`container4` （从`container4` ）。

   离开`container4`，并使用`CTRL-p CTRL-q` 使其保持运行。



### `docker network` 限制

虽然`docker network` 是控制您的容器使用的网络的推荐方法，但它确实有一些限制。

#### 环境变量注入

环境变量注入是静态的，环境变量在容器启动后无法更改。 遗留的`--link` 标志将所有环境变量共享到链接的容器，但`docker network` 命令没有等效选项。 当您使用`docker network` 将容器连接到网络时，不能在容器之间动态共享环境变量。

#### 使用网络范围的别名

遗留的link提供传出名称解析，隔离在配置别名的容器内。 网络范围的别名不允许这种单向隔离，而是为网络的所有成员提供别名。

以下示例说明了此限制。

1. 在网络`isolated_nw` 创建另一个容器`container6` ，并给它网络别名`app` 。

   ```
   $ docker run --network=isolated_nw -itd --name=container6 --network-alias app busybox

   8ebe6767c1e0361f27433090060b33200aac054a68476c3be87ef4005eb1df17
   ```

2. attach到`container4` 。 尝试通过名称（ `container6` ）和网络别名（ `app` ）ping容器。 请注意，IP地址是一样的。

   ```
   $ docker attach container4

   / # ping -w 4 app
   PING app (172.25.0.6): 56 data bytes
   64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms

   --- app ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms

   / # ping -w 4 container6
   PING container5 (172.25.0.6): 56 data bytes
   64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms

   --- container6 ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   从`container4` 离开，并使用`CTRL-p CTRL-q` 使其保持运行。

3. 将`container6` 连接到`local_alias` 网络，并为其赋予网络范围的别名`scoped-app` 。

   ```
   $ docker network connect --alias scoped-app local_alias container6
   ```

   现在`container6` 在网络`isolated_nw` 中的别名为`app` ，在网络`local_alias` 中别名为`scoped-app` 。

4. 尝试从`container4` （连接到这两个网络）和`container5` （仅连接到`isolated_nw` ）连接到这些别名。

   ```
   $ docker attach container4

   / # ping -w 4 scoped-app
   PING foo (172.26.0.5): 56 data bytes
   64 bytes from 172.26.0.5: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.26.0.5: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.26.0.5: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.26.0.5: seq=3 ttl=64 time=0.097 ms

   --- foo ping statistics ---
   4 packets transmitted, 4 packets received, 0% packet loss
   round-trip min/avg/max = 0.070/0.081/0.097 ms
   ```

   离开`container4` ，并使用`CTRL-p CTRL-q` 使其保持运行。

   ```
   $ docker attach container5

   / # ping -w 4 scoped-app
   ping: bad address 'scoped-app'
   ```

   离开`container5`，并使用`CTRL-p CTRL-q` 使其保持运行。

   这表明将别名仅在定义它的网络上生效，只有连接到该网络的容器才能访问该别名。

#### 将多个容器解析为一个别名

多个容器可在同一网络内共享相同的网络范围别名。 这提供了一种DNS轮询（round-robbin）高可用性。 当使用诸如Nginx这样的软件时，这可能不可靠，Nginx通过IP地址来缓存客户端。

以下示例说明了如何设置和使用网络别名。

> **注意** ：使用网络别名进行DNS轮询高可用的用户应考虑使用swarm服务。 Swarm服务提供了开箱即用的、类似的负载均衡功能。 如果连接到任何节点，即使是不参与服务的节点。 Docker将请求发送到正在参与服务的随机节点，并管理所有的通信。





1. 在`isolated_nw` 中启动`container7` ，别名与`container6`相同，即`app` 。

   ```
   $ docker run --network=isolated_nw -itd --name=container7 --network-alias app busybox

   3138c678c123b8799f4c7cc6a0cecc595acbdfa8bf81f621834103cd4f504554
   ```

   当多个容器共享相同的别名时，其中一个容器将解析为别名。 如果该容器不可用，则另一个具有别名的容器将被解析。 这提供了群集中的高可用性。

   > **注意** ：**在IP地址解析时，所选择的容器是不完全可预测的。 因此，在下面的练习中，您可能会在一些步骤中获得不同的结果。** 如果步骤假定返回的结果是`container6` 但是您收到`container7` ，这就是为什么。

2. 从`container4` 开始连续ping到`app` 别名。

   ```
   $ docker attach container4

   $ ping app
   PING app (172.25.0.6): 56 data bytes
   64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms
   ...
   ```

   返回的IP地址属于`container6` 。

3. 在另一个终端，停止`container6` 。

   ```
    $ docker stop container6 
   ```

   在连接到`container4` 的终端 ，观察`ping`输出。 当`container6`关闭时，它将暂停，因为`ping` 命令在首次调用时查找IP，并且发现该IP不再可用。 但是， `ping`命令在默认情况下具有非常长的超时时间，因此不会发生错误。

4. 使用`CTRL+C`退出`ping`命令并再次运行。

   ```
   $ ping app

   PING app (172.25.0.7): 56 data bytes
   64 bytes from 172.25.0.7: seq=0 ttl=64 time=0.095 ms
   64 bytes from 172.25.0.7: seq=1 ttl=64 time=0.075 ms
   64 bytes from 172.25.0.7: seq=2 ttl=64 time=0.072 ms
   64 bytes from 172.25.0.7: seq=3 ttl=64 time=0.101 ms
   ...
   ```

   `app`别名现在解析为`container7` 的IP地址。

5. 最后一次测试，重新启动`container6` 。

   ```
   $ docker start container6
   ```

   在连接到`container4` 的终端，再次运行`ping` 命令。 现在可能会再次解决`container6` 。 如果您几次启动和停止`ping` ，您将看到每个容器的响应。

   ```
   $ docker attach container4

   $ ping app
   PING app (172.25.0.6): 56 data bytes
   64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
   64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
   64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms
   ...
   ```

   用`CTRL+C` 停止ping。 从`container4` 离开，并使用`CTRL-p CTRL-q` 使其保持运行。








## 断开容器

您可以随时使用`docker network disconnect` 命令断开容器与网络的连接。

1. 从`isolated_nw` 网络断开`container2` ，然后检查`container2` 和`isolated_nw` 网络。

   ```
   $ docker network disconnect isolated_nw container2

   $ docker inspect --format=''  container2 | python -m json.tool

   {
       "bridge": {
           "NetworkID":"7ea29fc1412292a2d7bba362f9253545fecdfa8ce9a6e37dd10ba8bee7129812",
           "EndpointID": "9e4575f7f61c0f9d69317b7a4b92eefc133347836dd83ef65deffa16b9985dc0",
           "Gateway": "172.17.0.1",
           "GlobalIPv6Address": "",
           "GlobalIPv6PrefixLen": 0,
           "IPAddress": "172.17.0.3",
           "IPPrefixLen": 16,
           "IPv6Gateway": "",
           "MacAddress": "02:42:ac:11:00:03"
       }
   }
   $ docker network inspect isolated_nw

   [
       {
           "Name": "isolated_nw",
           "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
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
           "Containers": {
               "467a7863c3f0277ef8e661b38427737f28099b61fa55622d6c30fb288d88c551": {
                   "Name": "container3",
                   "EndpointID": "dffc7ec2915af58cc827d995e6ebdc897342be0420123277103c40ae35579103",
                   "MacAddress": "02:42:ac:19:03:03",
                   "IPv4Address": "172.25.3.3/16",
                   "IPv6Address": ""
               }
           },
           "Options": {}
       }
   ]
   ```

2. 当容器与网络断开连接时，它不能再与连接到该网络的其他容器进行通信，除非它与其他容器具有g共用他网络。 验证`container2` 不能再到达`isolated_nw` 上的`container3` 。

   ```
   $ docker attach container2

   / # ifconfig
   eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03  
             inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
             inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
             UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
             RX packets:8 errors:0 dropped:0 overruns:0 frame:0
             TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

   lo        Link encap:Local Loopback  
             inet addr:127.0.0.1  Mask:255.0.0.0
             inet6 addr: ::1/128 Scope:Host
             UP LOOPBACK RUNNING  MTU:65536  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

   / # ping container3
   PING container3 (172.25.3.3): 56 data bytes
   ^C
   --- container3 ping statistics ---
   2 packets transmitted, 0 packets received, 100% packet loss
   ```

3. 验证`container2` 是否仍具有与默认`bridge`完全连接。

   ```
   / # ping container1
   PING container1 (172.17.0.2): 56 data bytes
   64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.119 ms
   64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.174 ms
   ^C
   --- container1 ping statistics ---
   2 packets transmitted, 2 packets received, 0% packet loss
   round-trip min/avg/max = 0.119/0.146/0.174 ms
   / #
   ```

4. 移除`container4` ， `container5` ， `container6`和`container7` 。

   ```
   $ docker stop container4 container5 container6 container7

   $ docker rm container4 container5 container6 container7
   ```



### 处理过时的网络端点

在某些情况下，例如在多主机网络中以非优雅的方式重新启动Docker daemon，Docker daemon将无法清除过时的连接端点。 如果新的容器连接到具有与过期端点相同的名称的网络，则此类过时的端点可能会导致错误：

   ```
ERROR: Cannot start container bc0b19c089978f7845633027aa3435624ca3d12dd4f4f764b61eac4c0610f32e: container already connected to network multihost
   ```

要清理这些过时的端点，可移除容器并强制将其与网络断开（ `docker network disconnect -f` ）。 这样，您就可将容器成功连接到网络。

```
$ docker run -d --name redis_db --network multihost redis

ERROR: Cannot start container bc0b19c089978f7845633027aa3435624ca3d12dd4f4f764b61eac4c0610f32e: container already connected to network multihost

$ docker rm -f redis_db

$ docker network disconnect -f multihost redis_db

$ docker run -d --name redis_db --network multihost redis

7d986da974aeea5e9f7aca7e510bdb216d58682faa83a9040c2f2adc0544795a
```



## 删除网络

当网络中的所有容器都已停止或断开连接时，您可以删除网络。 如果网络连接了端点，则会发生错误。

1. 断开`container3` 与`isolated_nw` 连接。

   ```
   $ docker network disconnect isolated_nw container3
   ```


2. 检查`isolated_nw` 以验证没有其他端点连接到它。

   ```
   $ docker network inspect isolated_nw

   [
       {
           "Name": "isolated_nw",
           "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
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
           "Options": {}
       }
   ]
   ```

3. 删除`isolated_nw` 网络。

   ```
   $ docker network rm isolated_nw
   ```

4. 列出所有网络以验证`isolated_nw `不再存在：
   ```
   $ docker network ls

   NETWORK ID          NAME                DRIVER              SCOPE
   4bb8c9bf4292        bridge              bridge              local
   43575911a2bd        host                host                local
   76b7dc932e03        local_alias         bridge              local
   b1a086897963        my-network          bridge              local
   3eb020e70bfd        none                null                local
   69568e6336d8        simple-network      bridge              local
   ```




## 相关信息

- [network create](https://docs.docker.com/engine/reference/commandline/network_create/)
- [network inspect](https://docs.docker.com/engine/reference/commandline/network_inspect/)
- [network connect](https://docs.docker.com/engine/reference/commandline/network_connect/)
- [network disconnect](https://docs.docker.com/engine/reference/commandline/network_disconnect/)
- [network ls](https://docs.docker.com/engine/reference/commandline/network_ls/)
- [network rm](https://docs.docker.com/engine/reference/commandline/network_rm/)




## 原文

<https://docs.docker.com/engine/userguide/networking/work-with-networks/> 
