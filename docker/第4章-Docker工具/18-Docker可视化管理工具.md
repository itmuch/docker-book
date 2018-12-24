# Docker可视化管理工具

## DockerUI(ui for Docker)

官方GitHub：<https://github.com/kevana/ui-for-docker> 

项目已废弃，现在转投Portainer项目。



## Portainer

简介：Portainer是一个轻量级的管理界面，可以让您轻松地管理不同的Docker环境（Docker主机或Swarm集群）。Portainer提供状态显示面板、应用模板快速部署、容器镜像网络数据卷的基本操作、事件日志显示、容器控制台操作、Swarm集群和服务等集中管理和操作、登录用户管理和控制等功能。功能十分全面，基本能满足中小型单位对容器管理的全部需求。

官方GitHub：<https://github.com/portainer/portainer>

使用：

```
docker run -d --privileged -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /opt/portainer:/data portainer/portainer
```

如开启了SELinux，可执行如下命令启动：

```
docker run -d --privileged -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /opt/portainer:/data portainer/portainer
```

官方文档：<https://portainer.readthedocs.io/en/latest/deployment.html>



## Kitematic

简介：Kitematic是一个Docker GUI。

官方GitHub：<https://github.com/docker/kitematic>

使用：演示



## Shipyard

简介：Shipyard 是一个基于 Web 的 Docker 管理工具，支持多 host，可以把多个 Docker host 上的 containers 统一管理；可以查看 images，甚至 build images；并提供 RESTful API 等。

官方GitHub：<https://github.com/shipyard/shipyard>

安装：

```
curl -s https://shipyard-project.com/deploy | bash -s
```

展示所有参数：

```
curl -s https://shipyard-project.com/deploy | bash -s -- -h
```

使用：访问<http://localhost:8080> ，输入账号/密码：admin/shipyard即可访问Shipyard。

官方文档：<https://shipyard-project.com/>



## 各种可视化界面的比较

参考：<http://m.blog.csdn.net/qq273681448/article/details/75007828>