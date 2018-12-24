# 安装Docker Compose

本节我们来讨论如何安装Compose。





## 安装Compose

Compose有多种安装方式，例如通过Shell、pip以及将Compose作为容器安装等。本书讲解通过Shell来安装的方式，其他安装方式可详见官方文档：[https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

(1) 通过以下命令自动下载并安装适应系统版本的Compose

```shell
sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

(2) 为安装脚本添加执行权限

```shell
chmod +x /usr/local/bin/docker-compose
```

这样，Compose就安装完成了。

可使用以下命令测试安装结果。

```shell
docker-compose --version
```

可输出类似于如下的内容。

```shell
docker-compose version 1.16.1, build 1719ceb
```

说明Compose已成功安装。





## 安装Compose命令补全工具

我们已成功安装Compose，然而，当我们输入`docker-compose` 并按下Tab键时，Compose并没有为我们补全命令。要想使用Compose的命令补全，我们需要安装命令补全工具。

命令补全工具在Bash和Zsh下的安装方式不同，本书演示的是Bash下的安装。其他Shell以及其他操作系统上的安装，可详见Docker的官方文档：[https://docs.docker.com/compose/completion/](https://docs.docker.com/compose/completion/) ，笔者不作赘述。

* 执行以下命令，即可安装命令补全工具。

```shell
curl -L https://raw.githubusercontent.com/docker/compose/$(docker-compose version --short)/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
```

这样，在重新登录后，输入`docker-compose` 并按下Tab键，Compose就可自动补全命令了。



## 官方文档

Docker Compose安装：<https://docs.docker.com/compose/install/>

命令补全工具安装：<https://docs.docker.com/compose/completion/>