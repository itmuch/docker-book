# K8s资源分配

## 准备工作

本节中的示例需要安装Heapster。因此，需要先安装Heapster，对于Minikube，执行如下操作即可。

```shell
minikube addons enable heapster
```

查看Heapster Service是否已在运行：

```
kubectl get services --namespace=kube-system
```



### TIPS

按理，对于Minikube，启用Heapster addon后，即可在Dashboard上看到各种资源的CPU占用等指标。然而，对于`minikube 0.22.x` ，在Dashboard上无法看到，可使用如下命令，直接查询Grafana监控界面：

```shell
minikube addons open heapster
```

相关Issue：<https://github.com/kubernetes/minikube/issues/2013> 





## 限制CPU分配

### 示例

* `cpu-request-limit.yaml` ：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: cpu-demo
  spec:
    containers:
    - name: cpu-demo-ctr
      image: vish/stress
      resources:
        # 最多使用1个CPU单位
        limits:
          cpu: "1"
        # 至少保证0.5个CPU单位
        requests:
          cpu: "0.5"
      # 容器启动时的参数
      args:
      # 使用2个CPU单位
      - -cpus
      - "2"
  ```

* 创建Pod：

  ```shell
  kubectl create -f cpu-request-limit.yaml
  ```

* 查看Pod状态：

  ```shell
  kubectl get pod cpu-demo -o yaml
  ```

* 启动`proxy` ，并查询配额信息：

  ```shell
  kubectl proxy
  # 另起一个终端，输入：
  curl http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/heapster/api/v1/model/namespaces/default/pods/cpu-demo/metrics/cpu/usage_rate
  ```

  即可看到监控信息。


在本例中，尽管容器启动时，尝试使用2个CPU单位，但由于配置了只允许使用1个CPU单位，因此，最终最多只能使用1个CPU单位。



### CPU单位

CPU资源以*cpu为*单位。 在Kubernetes，一个cpu相当于：

- 1 AWS vCPU
- 1个GCP核心
- 1 Azure vCore
- 1个在裸机Intel处理器上的超线程

允许小数值。你可以使用m后缀来表示“毫”。例如100m cpu，100millicpu和0.1cpu表达的含义其实是相同的。精度不允许超过1m。

> 精度不允许超过1m的意思是，你不能指定有500.88m个CPU单位，精度最小是毫，就像人民币中最小的单位是分一样。

CPU配额是一个绝对值，而非相对值；“0.1 CPU”在单核、双核或48核机器表示的配额是相同的。



### 配额超过任何Node的容量

CPU最低要求和最大限制与容器相关联，但将Pod视为具有CPU最小配合和最大限制是有用的。Pod的CPU最小配需求是Pod中所有容器的CPU请求的总和。同样，Pod的CPU最大限制是Pod中所有容器的CPU最大限制的总和。

Pod的调度是基于最低要求的。只有当Node具有足够可用的CPU资源时，Pod才会在Node上运行。

在本练习中，您将创建一个非常大的CPU最低要求，以至于它超出了集群中任何Node的容量。以下是具有一个容器的Pod的配置文件。容器请求100 cpu，这可能超过集群中任何Node的容量。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-2
spec:
  containers:
  - name: cpu-demo-ctr-2
    image: vish/stress
    resources:
      limits:
        cpu: "100"
      requests:
        cpu: "100"
    args:
    - -cpus
    - "2"
```

创建该Pod后，使用`kubectl get pod cpu-demo-2` 查看Pod状态，即可看到类似如下的结果。由输出可知，Pod的状态为Pending。也就是说，Pod并没有被调度到任何Node上运行，它将无限期地保持在Pending状态。

```
kubectl get pod cpu-demo-2
NAME         READY     STATUS    RESTARTS   AGE
cpu-demo-2   0/1       Pending   0          7m
```

使用`kubectl describe pod cpu-demo-2`  命令查看Pod详情，即可看到类似如下的结果。由输出可看到无法调度的具体原因。

```
Events:
  Reason			Message
  ------			-------
  FailedScheduling	No nodes are available that match all of the following predicates:: Insufficient cpu (3).
```





## 限制内存分配

### 示例

* `memory-request-limit.yaml` 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: memory-demo
  spec:
    containers:
    - name: memory-demo-ctr
      image: vish/stress
      resources:
        # 最大允许使用200Mi内存
        limits:
          memory: "200Mi"
        # 至少保证200Mi的内存资源
        requests:
          memory: "100Mi"
      args:
      # 尝试分配150Mi内存给容器
      - -mem-total
      - 150Mi
      - -mem-alloc-size
      - 10Mi
      - -mem-alloc-sleep
      - 1s
  ```

* 查看Pod状态：

  ```shell
  kubectl get pod memory-demo -o yaml
  ```

  可看到类似如下结果：

  ```
  ...
  resources:
    limits:
      memory: 200Mi
    requests:
      memory: 100Mi
  ...
  ```

* 启动`proxy` 并查询配额信息：

  ```
  kubectl proxy
  curl http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/heapster/api/v1/model/namespaces/default/pods/memory-demo/metrics/memory/usage
  ```

  可看到如下结果：

  ```
  {
   "timestamp": "2017-06-20T18:54:00Z",
   "value": 162856960
  }
  ```

  由结果可知，Pod正使用大约162,900,000的内存，大约为150 MiB。这比Pod的100 MiB要求大，但在Pod的200MiB限制之内。




### 超出容器的内存限制

示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
spec:
  containers:
  - name: memory-demo-2-ctr
    image: vish/stress
    resources:
      requests:
        memory: 50Mi
      limits:
        memory: "100Mi"
    args:
    - -mem-total
    - 250Mi
    - -mem-alloc-size
    - 10Mi
    - -mem-alloc-sleep
    - 1s
```

此时，容器将不能运行。Container会因为内存不足（OOM）而被杀死。

类似的，如果指定Pod的内存超过K8s集群中任何Node的内存，Pod也无法运行。



### 内存单位

内存资源以字节为单位。使用一个整数或一个定点整数跟上以下后缀，来描述内存：E，P，T，G，M，K，Ei，Pi，Ti，Gi，Mi，Ki。 例如，以下代表大致相同的值：

```
128974848, 129e6, 129M , 123Mi 
```





## 参考文档

* 官方文档：《Assign CPU Resources to Containers and Pods》：<https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/>
* 官方文档：《Assign Memory Resources to Containers and Pods》：<https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/#memory-units>
* k8s的资源分配：<https://segmentfault.com/a/1190000003506106>





