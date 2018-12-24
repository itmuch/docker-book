# 安装Kubernetes（单机）

## 对于Mac/Windows 10

* 前提：保持网络畅通
* 系统版本满足要求

对于macOS或者Windows 10，Docker已经原生支持了Kubernetes。你所要做的只是启用Kubernetes即可，如下图：

![](images/kubernetes-install.png)





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