# 在生产环境中使用Docker Compose

在development中使用Compose定义应用程序时，可使用此定义，在不同环境（如CI，staging和production）中运行应用程序。

部署应用最简单的方法是在单机服务器上运行，类似于运行development环境的方式。如果要对应用程序扩容，可在Swarm集群上运行Compose应用程序。





### Modify your Compose file for production（为生产环境修改您的Compose文件）

您几乎肯定会对您的应用配置进行更改，从而使这些配置更适合线上环境。 这些更改可能包括：

* 删除任何绑定到应用程序代码的Volume，以便代码保持在容器内，不能从外部更改
* 绑定到主机上的不同端口
* 设置不同的环境变量（例如，减少日志的冗长程度或启用email发送）
  * DEBUG    INFO     WARN     ERROR      FETAL
* 指定重启策略（例如， `restart: always` ），从而避免停机
* 添加额外服务（例如，日志聚合器）

因此，您可能需要定义一个额外的Compose文件，比如`production.yml` ，它指定了适用于生产的配置。此配置文件只需包含从原始Compose文件的修改。该附加Compose文件，可在原始的`docker-compose.yml` 基础上被应用，从而创建新的配置。

一旦获得了第二个配置文件，可使用`-f` 选项告诉Compose：

```
docker-compose -f docker-compose.yml -f production.yml up -d
```

请参阅  [Using multiple compose files](https://docs.docker.com/compose/extends/#different-environments)  获取更完整的示例。





### Deploying changes（部署修改）

当您更改应用代码时，您需要重新构建镜像并重新创建容器。例如，重新部署名为`web` 的服务，可使用：

```
$ docker-compose build web
$ docker-compose up --no-deps -d web
```

这将会先重新构建`web` 的镜像，然后停止、销毁、重新创建`web` 服务。 `--no-deps` 标志可防止Compose重新创建任何`web` 依赖的服务。





### Running Compose on a single server（单机服务器上运行Compose）

通过适当地设置`DOCKER_HOST` 、`DOCKER_TLS_VERIFY` 和`DOCKER_CERT_PATH` 等环境变量，可使用Compose将应用程序部署到远程的Docker主机。 对于像这样的任务，[Docker Machine](https://docs.docker.com/machine/overview/) 可使本地/远程Docker主机管理变得非常简单，即使您没有远程部署也推荐使用Docker Machine。

一旦您设置了如上环境变量，所有正常的`docker-compose` 命令将无需进一步的配置。





### Running Compose on a Swarm cluster（在Swarm集群上运行Compose）

[Docker Swarm](https://docs.docker.com/swarm/overview/) ，是一款Docker原生的集群系统，它暴露了与单个Docker主机相同的API，这意味着您可在Swarm实例上使用Compose，并在多个主机上运行应用程序。

阅读更多关于集成指Compose/Swarm整合的内容，请详见 [integration guide](https://docs.docker.com/compose/swarm/) 。





## 原文

<https://docs.docker.com/compose/production/> 