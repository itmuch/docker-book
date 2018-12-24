# docker-compose.yml常用命令

docker-compose.yml是Compose的默认模板文件。该文件有多种写法，例如Version 1 file format、Version 2 file format、Version 2.1 file format、Version 3 file format等。其中，Version 1 file format将逐步被被弃用；Version 2.x及Version 3.x基本兼容，是未来的趋势。考虑到目前业界的使用情况，本节只讨论Version 2 file format下的常用命令。





## (1) build

配置构建时的选项，Compose会利用它自动构建镜像。build的值可以是一个路径，例如：

```yaml
build: ./dir
```

也可以是一个对象，用于指定Dockerfile和参数，例如：

```yaml
build:
  context: ./dir
  dockerfile: Dockerfile-alternate
  args:
    buildno: 1
```





## (2) command

覆盖容器启动后默认执行的命令。示例：

```yaml
command: bundle exec thin -p 3000
```

也可以是一个list，类似于Dockerfile中的CMD指令，格式如下：

```yaml
command: [bundle, exec, thin, -p, 3000]
```





## (3) dns

配置dns服务器。可以是一个值，也可以是一个列表。示例：

```yaml
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
```





## (4) dns_search

配置DNS的搜索域名，可以是一个值，也可以是一个列表。示例：

```yaml
dns_search: example.com
dns_search:
  - dc1.example.com
  - dc2.example.com
```





## (5) environment

环境变量设置，可使用数组或字典两种方式。示例：

```yaml
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```





## (6) env_file

从文件中获取环境变量，可指定一个文件路径或路径列表。如果通过 `docker-compose -f FILE` 指定了Compose文件，那么env_file中的路径是Compose文件所在目录的相对路径。使用environment指定的环境变量会覆盖env_file指定的环境变量。示例：

```yaml
env_file: .env

env_file:
  - ./common.env   # 共用
  - ./apps/web.env # web用
  - /opt/secrets.env # 密码用
```





## (7) expose

暴露端口，只将端口暴露给连接的服务，而不暴露给宿主机。示例：

```yaml
expose:
 - "3000"
 - "8000"
```





## (8) external_links

连接到docker-compose.yml外部的容器，甚至并非Compose管理的容器，特别是提供共享或公共服务的容器。格式跟links类似，例如：

```yaml
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```





## (9) image

指定镜像名称或镜像id，如果本地不存在该镜像，Compose会尝试下载该镜像。

示例：

```yaml
image: java
```





## (10) links

连接到其他服务的容器。可以指定服务名称和服务别名（ `SERVICE:ALIAS` ），也可只指定服务名称。例如：

```yaml
web:
  links:
   - db
   - db:database
   - redis
```





## (11) networks

详见本书《Docker Compose网络设置》一节。





## (12) network_mode

设置网络模式。示例：

```yaml
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```





## (13) ports

暴露端口信息，可使用`HOST:CONTAINER` 的格式，也可只指定容器端口（此时宿主机将会随机选择端口），类似于`docker run -p ` 。

需要注意的是，当使用`HOST:CONTAINER` 格式映射端口时，容器端口小于60将会得到错误的接口，因为yaml会把`xx:yy` 的数字解析为60进制。因此，建议使用字符串的形式。示例：

```yaml
ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
```







## (14) volumes

卷挂载路径设置。可以设置宿主机路径 （`HOST:CONTAINER`） ，也可指定访问模式 （`HOST:CONTAINER:ro`）。示例：

```yaml
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # Named volume
  - datavolume:/var/lib/mysql
```





## (15) volumes_from

从另一个服务或容器挂载卷。可指定只读（ro）或读写（rw），默认是读写（rw）。示例：

```yaml
volumes_from:
 - service_name
 - service_name:ro
 - container:container_name
 - container:container_name:rw
```





## TIPS

(1) docker-compose.yml还有很多其他命令，比如depends_on、pid、devices等。限于篇幅，笔者仅挑选常用的命令进行讲解，其他命令不作赘述。感兴趣的读者们可参考官方文档：[https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/) 。


