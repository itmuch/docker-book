# 安装Kubernetes（单机）

## 对于Mac/Windows 10

* 前提：保持网络畅通
* 系统版本满足要求

对于macOS或者Windows 10，Docker已经原生支持了Kubernetes。你所要做的只是启用Kubernetes即可，如下图：

![](images/kubernetes-install.png)

如果因为网络问题安装不成功，可参考 <https://github.com/AliyunContainerService/k8s-for-docker-desktop> 的说明进行安装。

**TIPS**：

在Mac上:

如果在Kubernetes部署的过程中出现问题，可以通过docker desktop应用日志获得实时日志信息：

```
pred='process matches ".*(ocker|vpnkit).*"
  || (process in {"taskgated-helper", "launchservicesd", "kernel"} && eventMessage contains[c] "docker")'
/usr/bin/log stream --style syslog --level=debug --color=always --predicate "$pred"
```

在Windows上:

如果在Kubernetes部署的过程中出现问题，可以在 C:\ProgramData\DockerDesktop下的service.txt 查看Docker日志; 如果看到 Kubernetes一直在启动状态，请参考 [Issue 3769(comment)](https://github.com/docker/for-win/issues/3769#issuecomment-486046718) 和 [Issue 1962(comment)](https://github.com/docker/for-win/issues/1962#issuecomment-431091114)

## Minikube

一些场景下，安装Minikube是个不错的选择。该方式适用于Windows 10、Linux、macOS

* 官方安装说明文档：<https://github.com/kubernetes/minikube>
* 如何在Windows 10上运行Docker和Kubernetes？：<http://dockone.io/article/8136>



## 启用Kubernetes Dashboard

执行：

```
kubectl proxy
```

访问：

<http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default>

参考：

<https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/>

