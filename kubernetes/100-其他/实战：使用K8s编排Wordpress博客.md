# 标题

## MySQL

### 创建deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment                      # 定义一个deployment
metadata:
  name: mysql                         # deployment名称，全局唯一
  labels:
    app: mysql123
    release: stable
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql123                   # deployment的POD标签选择器，即：监控和管理拥有这些标签的POD实例，确保当前集群中有且只有replicas个POD实例在运行
  template:
    metadata:
      labels:                         # 指定该POD的标签
        app: mysql123                 # POD副本拥有的标签，需要与deployment的selector一致
    spec:
      containers:
      - name: mysql
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```

### 创建SVC

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql           # Service名称，全局唯一
  labels:
    app: mysql123
spec:
  ports:
  - port: 3306          # Service提供服务的端口号
  selector:
    app: mysql123       # 选择器
```

## Wordpress

### 创建deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:4.8-apache
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: "mysql"
        - name: WORDPRESS_DB_PASSWORD
          value: "123456"
```

### 创建SVC

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: NodePort    # 为该Service开启NodePort方式的外网访问模式
  ports:
  - port: 80        # Service提供服务的端口号
    targetPort: 80  # 将Service的80端口转发到Pod中容器的80端口上
    nodePort: 32001 # 在k8s集群外访问的端口，如果设置了NodePort类型，但没设置nodePort，将会随机映射一个端口，可使用kubectl get svc wordpress看到
  selector:
    app: wordpress
```



## 删除

```shell
kubectl delete deployment,svc -l app=mysql123 
kubectl delete deployment,svc -l app=wordpress
```


