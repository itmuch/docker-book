# 第十章 实战：使用Dockerfile在CentOS 7中安装Nginx

基于CentOS 7镜像，在其中安装Nginx，并启动。

**提示**：默认Nginx不在官方Yum仓库中，需要先安装RPMS仓库包，这样才能用Yum安装Nginx。安装RPMS包的命令如下：

```Shell
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```



## 答案

```dockerfile
FROM centos:7
RUN rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
RUN yum -y install nginx
RUN sed -i '1i\daemon off;' /etc/nginx/nginx.conf
ENTRYPOINT nginx
```

