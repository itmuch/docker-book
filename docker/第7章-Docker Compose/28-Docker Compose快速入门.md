# Docker Compose快速入门

本节我们来探讨Compose使用的基本步骤，并编写一个简单示例快速入门。



# 基本步骤

使用Compose大致有三个步骤：

* 使用Dockerfile（或其他方式）定义应用程序环境，以便在任何地方重现该环境。
* 在docker-compose.yml文件中定义组成应用程序的服务，以便各个服务在一个隔离的环境中一起运行。
* 运行docker-compose up命令，启动并运行整个应用程序。





# 入门示例

下面笔者以之前课上用到的Eureka为例讲解Compose的基本步骤。

* 在`microservice-discovery-eureka-0.0.1-SNAPSHOT.jar` 所在路径（默认是项目的target目录）创建Dockerfile文件，并在其中添加如下内容。

  ```dockerfile
  FROM java:8
  VOLUME /tmp
  ADD microservice-discovery-eureka-0.0.1-SNAPSHOT.jar app.jar
  RUN bash -c 'touch /app.jar'
  EXPOSE 9000
  ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
  ```

* 在`microservice-discovery-eureka-0.0.1-SNAPSHOT.jar` 所在路径创建文件docker-compose.yml，在其中添加如下内容。

  ```yaml
  version: '2'			# 表示该docker-compose.yml文件使用的是Version 2 file format
  services:
    eureka:				# 指定服务名称
      build: .			# 指定Dockerfile所在路径
      ports:
        - "8761:8761"		# 指定端口映射，类似docker run的-p选项，注意使用字符串形式
  ```

* 在`docker-compose.yml` 所在路径执行以下命令。

  ```shell
  docker-compose up
  ```

  Compose就会自动构建镜像并使用镜像启动容器。我们也可使用`docker-compose up -d` 后台启动并运行这些容器。

* 访问：`http://宿主机IP:8761/`  ，即可访问Eureka Server首页。





# 工程、服务、容器

Docker Compose将所管理的容器分为三层，分别是工程（project），服务（service）以及容器（container）。Docker Compose运行目录下的所有文件（docker-compose.yml, extends文件或环境变量文件等）组成一个工程（默认为docker-compose.yml所在目录的目录名称）。一个工程可包含多个服务；每个服务中定义了容器运行的镜像、参数和依赖，一个服务可包括多个容器实例。

对应《入门示例》一节，工程名称是docker-compose.yml所在的目录名。该工程包含了1个服务，服务名称是eureka；执行docker-compose up时，启动了eureka服务的1个容器实例。

