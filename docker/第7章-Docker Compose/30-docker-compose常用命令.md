# docker-compose常用命令

和docker命令一样，docker-compose命令也有很多选项。下面我们来详细探讨docker-compose的常用命令。





## build

构建或重新构建服务。服务被构建后将会以`project_service` 的形式标记，例如：`composetest_db` 。





## help

查看指定命令的帮助文档，该命令非常实用。docker-compose所有命令的帮助文档都可通过该命令查看。

```shell
docker-compose help COMMAND
```

示例：

```shell
docker-compose help build		# 查看docker-compose build的帮助
```





## kill

通过发送`SIGKILL` 信号停止指定服务的容器。示例：

```
docker-compose kill eureka
```

该命令也支持通过参数来指定发送的信号，例如：

```shell
docker-compose kill -s SIGINT
```





##  logs

查看服务的日志输出。





##  port

打印绑定的公共端口。示例：

```shell
docker-compose port eureka 8761
```

这样就可输出eureka服务8761端口所绑定的公共端口。





## ps

列出所有容器。示例：

```shell
docker-compose ps
```

也可列出指定服务的容器，示例：

```shell
docker-compose ps eureka
```






## pull

下载服务镜像。





## rm

删除指定服务的容器。示例：

```shell
docker-compose rm eureka
```





## run

在一个服务上执行一个命令。示例：

```shell
docker-compose run web bash
```

这样即可启动一个web服务，同时执行bash命令。





## scale

设置指定服务运行容器的个数，以service=num的形式指定。示例：

```shell
docker-compose scale user=3 movie=3
```





## start

启动指定服务已存在的容器。示例：

```shell
docker-compose start eureka
```





## stop

停止已运行的容器。示例：

```shell
docker-compose stop eureka
```

停止后，可使用`docker-compose start` 再次启动这些容器。





## up

构建、创建、重新创建、启动，连接服务的相关容器。所有连接的服务都会启动，除非它们已经运行。

`docker-compose up` 命令会聚合所有容器的输出，当命令退出时，所有容器都会停止。

使用`docker-compose up -d` 可在后台启动并运行所有容器。





## TIPS

(1) 本节仅讨论常用的docker-compose命令，其他命令可详见Docker官方文档：[https://docs.docker.com/compose/reference/overview/](https://docs.docker.com/compose/reference/overview/) 。