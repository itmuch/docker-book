# Docker数据持久化

容器中数据持久化主要有两种方式：

* 数据卷（Data Volumes）
* 数据卷容器（Data Volumes Dontainers）



## 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，可以绕过UFS（Unix File System）。

* 数据卷可以在容器之间共享和重用
* 对数据卷的修改会立马生效
* 对数据卷的更新，不会影响镜像
* 数据卷默认会一直存在，即使容器被删除
* 一个容器可以挂载多个数据卷

**注意**：数据卷的使用，类似于 Linux 下对目录或文件进行 mount。



### 创建数据卷

示例：

```shell
docker run --name nginx-data -v /mydir nginx
```

执行如下命令即可查看容器构造的详情：

```Shell
docker inspect 容器ID
```

由测试可知：

* Docker会自动生成一个目录作为挂载的目录。


* **即使容器被删除，宿主机中的目录也不会被删除。**





### 删除数据卷

数据卷是被设计来持久化数据的，因此，删除容器并不会删除数据卷。如果想要在删除容器时同时删除数据卷，可使用如下命令：

```shell
docker rm -v 容器ID
```

这样既可在删除容器的同时也将数据卷删除。



### 挂载宿主机目录作为数据卷

```shell
docker run --name nginx-data2 -v /host-dir:/container-dir nginx
```

这样既可将宿主机的/host-dir路径加载到容器的/container-dir中。

需要注意的是：

* 宿主机路径**尽量设置**绝对路径——如果使用相对路径会怎样？
  * 测试给答案
* 如果宿主机路径不存在，Docker会自动创建

**TIPS**

Dockerfile暂时不支持这种形式。



### 挂载宿主机文件作为数据卷

```Shell
docker run --name nginx-data3 -v /文件路径:/container路径 nginx
```



### 指定权限

默认情况下，挂载的权限是读写权限。也可使用`:ro` 参数指定只读权限。

示例：

```Shell
docker run --name nginx-data4 -v /host-dir:/container-dir:ro nginx
```

这样，在容器中就只能读取/container-dir中的文件，而不能修改了。





## 数据卷容器

如果有数据需要在多个容器之间共享，此时可考虑使用数据卷容器。

创建数据卷容器：

```dockerfile
docker run --name nginx-volume -v /data nginx
```

在其他容器中使用`-volumes-from` 来挂载nginx-volume容器中的数据卷。

```dockerfile
docker run --name v1 --volumes-from nginx-volume nginx
docker run --name v2 --volumes-from nginx-volume nginx
```

这样：

* v1、v2两个容器即可共享nginx-volume这个容器中的文件。
* 即使nginx-volume停止，也不会有任何影响。





