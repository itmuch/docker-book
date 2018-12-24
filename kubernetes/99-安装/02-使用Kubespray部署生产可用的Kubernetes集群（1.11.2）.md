# 使用Kubespray部署生产可用的Kubernetes集群（1.11.2）

**前提**：科学上网，或自行将gcr.io的镜像转成其他镜像仓库的镜像。



Kubernetes的安装部署是难中之难，每个版本安装方式都略有区别。笔者一直想找一种`支持多平台`、`相对简单` 、`适用于生产环境` 的部署方案。经过一段时间的调研，有如下几种解决方案进入笔者视野：

| 部署方案                                                     | 优点                                                   | 缺点                   |
| ------------------------------------------------------------ | ------------------------------------------------------ | ---------------------- |
| [Kubeadm](https://github.com/kubernetes/kubeadm)             | 官方出品                                               | 部署较麻烦、不够透明   |
| [Kubespray](https://github.com/kubernetes-incubator/kubespray) | 官方出品、部署较简单、懂Ansible就能上手                | 不够透明               |
| [RKE](https://github.com/rancher/rke)                        | 部署较简单、需要花一些时间了解RKE的cluster.yml配置文件 | 不够透明               |
| 手动部署 [第三方操作文档](https://github.com/opsnull/follow-me-install-kubernetes-cluster) | 完全透明、可配置、便于理解K8s各组件之间的关系          | 部署非常麻烦，容易出错 |

其他诸如Kops之类的方案，由于无法跨平台，或者其他因素，被我pass了。

最终，笔者决定使用Kubespray部署Kubernetes集群。**也希望大家能够一起讨论，总结出更加好的部署方案**。

废话不多说，以下是操作步骤。





> 注：撰写本文时，笔者临时租赁了几台海外阿里云机器，实现了科学上网。如果您的机器在国内，请：
>
> - 考虑科学上网
> - 或修改Kubespray中的gcr地址，改为其他仓库地址，例如阿里云镜像地址。

## 主机规划

| IP          | 作用           |
| ----------- | -------------- |
| 172.20.0.87 | ansible-client |
| 172.20.0.88 | master,node    |
| 172.20.0.89 | master,node    |
| 172.20.0.90 | node           |
| 172.20.0.91 | node           |
| 172.20.0.92 | node           |

## 准备工作

### 关闭selinux

所有机器都必须关闭selinux，执行如下命令即可。

```
~]# setenforce 0
~]# sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

### 网络配置

#### 在master机器上

```
~]# firewall-cmd --permanent --add-port=6443/tcp
~]# firewall-cmd --permanent --add-port=2379-2380/tcp
~]# firewall-cmd --permanent --add-port=10250/tcp
~]# firewall-cmd --permanent --add-port=10251/tcp
~]# firewall-cmd --permanent --add-port=10252/tcp
~]# firewall-cmd --permanent --add-port=10255/tcp
~]# firewall-cmd --reload
~]# modprobe br_netfilter
~]# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
~]# sysctl -w net.ipv4.ip_forward=1
```

如果关闭了防火墙，则只需执行最下面三行。

#### 在node机器上

```
~]# firewall-cmd --permanent --add-port=10250/tcp
~]# firewall-cmd --permanent --add-port=10255/tcp
~]# firewall-cmd --permanent --add-port=30000-32767/tcp
~]# firewall-cmd --permanent --add-port=6783/tcp
~]# firewall-cmd  --reload
~]# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
~]# sysctl -w net.ipv4.ip_forward=1
```

如果关闭了防火墙，则只需执行最下面两行。

#### 【可选】关闭防火墙

```
systemctl stop firewalld
```

## 在ansible-client机器上安装ansible

### 安装ansible

```
~]# sudo yum install epel-release
~]# sudo yum install ansible
```

### 安装jinja2

```
~]# easy_install pip
~]# pip2 install jinja2 --upgrade
```

如果执行`pip2 install jinja2 --upgrade` 出现类似如下的提示：

```
You are using pip version 9.0.1, however version 18.0 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```

则执行`pip install --upgrade pip` 升级pip，再执行`pip2 install jinja2 --upgrade`

### 安装Python 3.6

```
~]# sudo yum install python36 –y
```

## 在ansible-client机器上配置免密登录

### 生成ssh公钥和私钥

在ansible-cilent机器上执行：

```
~]# ssh-keygen
```

然后三次回车，生成ssh公钥和私钥。

### 建立ssh单向通道

在ansible-cilent机器上执行：

```
~]# ssh-copy-id root@172.20.0.88		#将公钥分发给88机器
~]# ssh-copy-id root@172.20.0.89
~]# ssh-copy-id root@172.20.0.90
~]# ssh-copy-id root@172.20.0.91
~]# ssh-copy-id root@172.20.0.92
```

## 在ansible-client机器上安装kubespray

- 下载kubespray

  > **TIPS**：本文下载的是master分支，如果大家要部署到线上环境，建议下载RELEASE分支。笔者撰写本文时，最新的RELEASE是2.6.0，RELEASE版本下载地址：<https://github.com/kubernetes-incubator/kubespray/releases>）

  ```
  ~]# git clone https://github.com/kubernetes-incubator/kubespray.git
  ```

- 安装kubespray需要的包：

  ```
  ~]# cd kubespray
  ~]# sudo pip install -r requirements.txt
  ```

- 拷贝`inventory/sample` ，命名为`inventory/mycluster` ，mycluster可以改为其他你喜欢的名字

  ```
  cp -r inventory/sample inventory/mycluster
  ```

- 使用inventory_builder，初始化inventory文件

  ```
  ~]# declare -a IPS=(172.20.0.88 172.20.0.89 172.20.0.90 172.20.0.91 172.20.0.92)
  ~]# CONFIG_FILE=inventory/mycluster/hosts.ini python36 contrib/inventory_builder/inventory.py ${IPS[@]}
  ```

  此时，会看到`inventory/mycluster/host.ini` 文件内容类似如下：

  ```
  [k8s-cluster:children]
  kube-master      
  kube-node        
  
  [all]
  node1    ansible_host=172.20.0.88 ip=172.20.0.88
  node2    ansible_host=172.20.0.89 ip=172.20.0.89
  node3    ansible_host=172.20.0.90 ip=172.20.0.90
  node4    ansible_host=172.20.0.91 ip=172.20.0.91
  node5    ansible_host=172.20.0.92 ip=172.20.0.92
  
  [kube-master]
  node1    
  node2    
  
  [kube-node]
  node1    
  node2    
  node3    
  node4    
  node5    
  
  [etcd]
  node1    
  node2    
  node3    
  
  [calico-rr]
  
  [vault]
  node1    
  node2    
  node3
  ```

- 使用ansible playbook部署kubespray

  ```
  ~]# ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml
  ```

- 大概20分钟左右，Kubernetes即可安装完毕。

## 验证

验证1：查看Node状态

```
]# kubectl get nodes
NAME      STATUS    ROLES         AGE       VERSION
node1     Ready     master,node   2m        v1.11.2
node2     Ready     master,node   2m        v1.11.2
node3     Ready     node          2m        v1.11.2
node4     Ready     node          2m        v1.11.2
node5     Ready     node          2m        v1.11.2
```

每个node都是ready的，说明OK。

验证2：部署一个NGINX

```
# 启动一个单节点nginx
]# kubectl run nginx --image=nginx:1.7.9 --port=80

# 为“nginx”服务暴露端口
]# kubectl expose deployment nginx --type=NodePort

# 查看nginx服务详情
]# kubectl get svc nginx
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx     NodePort   10.233.29.96   <none>        80:32345/TCP   14s

# 访问测试，如果能够正常返回NGINX首页，说明正常
]# curl localhost:32345
```

## 卸载

```
]# ansible-playbook -i inventory/mycluster/hosts.ini reset.yml
```

## 遇到的问题

Calico网络插件部署失效。这是Calico 3.2所带来的问题，原因详见：<https://github.com/kubernetes-incubator/kubespray/issues/3223>

解决方法：<https://github.com/wilmardo/kubespray/commit/1c87a49d1443bcdd237500a714f1a60d680c1ad8>，即：将Calico降级到3.1.3。

## 参考文档：

- Kubespray – 10 Simple Steps for Installing a Production-Ready, Multi-Master HA Kubernetes Cluster：<https://dzone.com/articles/kubespray-10-simple-steps-for-installing-a-product>

  > **TIPS**：主要的参考文档，里面还讲解了Kubespray的一些配置，与可能会遇到的问题及解决方案。

- 使用Kubespray 部署kubernetes 高可用集群：<https://yq.aliyun.com/articles/505382>

- kubespray(ansible)自动化安装k8s集群：<https://www.cnblogs.com/iiiiher/p/8128184.html>

  > **TIPS**：里面有将如何替换gcr镜像为国内镜像

- Installing Kubernetes On-premises/Cloud Providers with Kubespray:<https://kubernetes.io/docs/setup/custom-cloud/kubespray/>