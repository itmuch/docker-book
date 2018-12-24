# 第十六章 使用Maven插件构建Docker镜像

我们知道，Maven是一个强大的项目管理与构建工具。如果可以使用Maven构建Docker镜像，那么我们的工作就能得到进一步的简化。

经过调研，以下几款Maven的Docker插件进入笔者视野，如表13-1所示。

| 插件名称                | 官方地址                                     |
| ------------------- | ---------------------------------------- |
| docker-maven-plugin | https://github.com/spotify/docker-maven-plugin |
| docker-maven-plugin | https://github.com/fabric8io/docker-maven-plugin |
| docker-maven-plugin | https://github.com/bibryam/docker-maven-plugin |

表13-1 Maven的Docker插件列表

笔者从各项目的功能性、文档易用性、更新频率、社区活跃度、Stars等几个纬度考虑，选用了第一款。这是一款由Spotify公司开发的Maven插件。

下面我们来详细探讨如何使用Maven插件构建Docker镜像。





## 快速入门

以项目`microservice-discovery-eureka` 为例。

(1) 在pom.xml中添加Maven的Docker插件。

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.4.13</version>
  <configuration>
    <imageName>itmuch/microservice-discovery-eureka:0.0.1</imageName>
    <baseImage>java</baseImage>
    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>${project.build.finalName}.jar</include>
      </resource>
    </resources>
  </configuration>
</plugin>
```
简要说明一下插件的配置：

① imageName：用于指定镜像名称，其中itmuch是仓库名称，microservice-discovery-eureka是镜像名称，0.0.1是标签名称。

② baseImage：用于指定基础镜像，类似于Dockerfile中的FROM指令。

③ entrypoint：类似于Dockerfile的ENTRYPOINT指令。

④ resources.resource.directory：用于指定需要复制的根目录，${project.build.directory}表示target目录。

⑤ resources.resource.include：用于指定需要复制的文件。${project.build.finalName}.jar指的是打包后的jar包文件。

(2) 执行以下命令，构建Docker镜像。

```shell
mvn clean package docker:build
```
我们会发现终端输出类似于如下的内容：

```shell
[INFO] Building image itmuch/microservice-discovery-eureka:0.0.1
Step 1 : FROM java
 ---> 861e95c114d6
Step 2 : ADD /microservice-discovery-eureka-0.0.1-SNAPSHOT.jar //
 ---> 035a03f5b389
Removing intermediate container 2b0e70056f1d
Step 3 : ENTRYPOINT java -jar /microservice-discovery-eureka-0.0.1-SNAPSHOT.jar
 ---> Running in a0149704b949
 ---> eb96ca1402aa
Removing intermediate container a0149704b949
Successfully built eb96ca1402aa
```

由以上日志可知，我们已成功构建了一个Docker镜像。

(3)执行 `docker images` 命令，即可查看刚刚构建的镜像。

```
REPOSITORY                             TAG        IMAGE ID          CREATED             SIZE
itmuch/microservice-discovery-eureka   0.0.1      eb96ca1402aa      39 seconds ago      685 MB
```
(4) 启动镜像

```shell
docker run -d -p 8761:8761 itmuch/microservice-discovery-eureka:0.0.1
```

我们会发现该Docker镜像会很快地启动

(5) 访问测试

访问[http://Docker宿主机IP:8761](http://Docker宿主机IP:8761) ，能够看到Eureka Server的首页。





## 插件读取Dockerfile进行构建

之前的示例，我们直接在pom.xml中设置了一些构建的参数。很多场景下，我们希望使用Dockerfile更精确、有可读性地构建镜像。

(1) 首先我们在`/microservice-discovery-eureka/src/main/docker` 目录下，新建一个Dockerfile文件，例如：

```dockerfile
FROM java:8
VOLUME /tmp
ADD microservice-discovery-eureka-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'
EXPOSE 9000
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

(2) 修改pom.xml

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.4.13</version>
  <configuration>
    <imageName>itmuch/microservice-discovery-eureka:0.0.2</imageName>
    <dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>${project.build.finalName}.jar</include>
      </resource>
    </resources>
  </configuration>
</plugin>
```

可以看到，我们不再指定baseImage和entrypoint，而是使用dockerDirectory指定Dockerfile所在的路径。这样，我们就可以使用Dockerfile构建Docker镜像了。





## 将插件绑定在某个phase执行

很多场景下，我们有这样的需求，执行例如`mvn clean package` 时，插件就自动为我们构建Docker镜像。要想实现这点，我们只需将插件的goal绑定在某个phase即可。

phase和goal可以这样理解：maven命令格式是：`mvn phase:goal` ，例如`mvn package docker:build` 。那么，package和docker都是phase，build则是goal 。示例：

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.4.13</version>
  <executions>
    <execution>
      <id>build-image</id>
      <phase>package</phase>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <imageName>itmuch/microservice-discovery-eureka:0.0.3</imageName>
    <baseImage>java</baseImage>
    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>${project.build.finalName}.jar</include>
      </resource>
    </resources>
  </configuration>
</plugin>
```

由配置可知，我们只需添加如下配置：

```xml
<executions>
  <execution>
    <id>build-image</id>
    <phase>package</phase>
    <goals>
      <goal>build</goal>
    </goals>
  </execution>
</executions>
```

就可将插件绑定在package这个phase上。也就是说，用户只需执行`mvn package` ，就会自动执行`mvn docker:build` 。当然，读者也可按照需求，将插件绑定到其他的phase。





## 推送镜像

前文我们使用`docker push` 命令实现了镜像的推送，我们也可使用Maven插件推送镜像。我们不妨使用Maven插件推送一个Docker镜像到Docker Hub。

(1) 修改Maven的全局配置文件setttings.xml，在其中添加以下内容，配置Docker Hub的用户信息。

```xml
<server>
  <id>docker-hub</id>
  <username>你的DockerHub用户名</username>
  <password>你的DockerHub密码</password>
  <configuration>
    <email>你的DockerHub邮箱</email>
  </configuration>
</server>
```

(2) 修改pom.xml，示例：

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.4.13</version>
  <configuration>
    <imageName>itmuch/microservice-discovery-eureka:0.0.4</imageName>
    <baseImage>java</baseImage>
    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
    <resources>
      <resource>
        <targetPath>/</targetPath>
        <directory>${project.build.directory}</directory>
        <include>${project.build.finalName}.jar</include>
      </resource>
    </resources>

    <!-- 与maven配置文件settings.xml中配置的server.id一致，用于推送镜像 -->
    <serverId>docker-hub</serverId>
  </configuration>
</plugin>
```

如上，添加serverId段落，并引用settings.xml中的server的id即可。

(3) 执行以下命令，添加pushImage的标识，表示推送镜像。

```shell
mvn clean package docker:build  -DpushImage
```

经过一段时间的等待，我们会发现Docker镜像已经被push到Docker Hub了。同理，我们也可推送镜像到私有仓库，只需要将imageName指定成类似于如下的形式即可：

```xml
<imageName>localhost:5000/itmuch/microservice-discovery-eureka:0.0.4</imageName>
```





**TIPS**

(1) 以上示例中，我们是通过imageName指定镜像名称和标签的，例如：

```xml
<imageName>itmuch/microservice-discovery-eureka:0.0.4</imageName>
```

我们也可借助imageTags元素更为灵活地指定镜像名称和标签，例如：

```xml
<configuration>
  <imageName>itmuch/microservice-discovery-eureka</imageName>
  <imageTags>
    <imageTag>0.0.5</imageTag>
    <imageTag>latest</imageTag>
  </imageTags>
  ...
<configuration>
```

这样就可为同一个镜像指定两个标签。

(2) 我们也可在执行构建命令时，使用dockerImageTags参数指定标签名称，例如：

```shell
mvn clean package docker:build -DpushImageTags -DdockerImageTags=latest -DdockerImageTags=another-tag
```

(3) 如需重复构建相同标签名称的镜像，可将forceTags设为true，这样就会覆盖构建相同标签的镜像。

```xml
<configuration>
  <!-- optionally overwrite tags every time image is built with docker:build -->
  <forceTags>true</forceTags>
<configuration>
```






## 拓展阅读

(1) Spotify是全球最大的正版流媒体音乐服务平台。