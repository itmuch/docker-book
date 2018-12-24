# 第十一章 实战：使用Dockerfile构建一个Spring Boot应用镜像
有一个Java应用，在裸机中的启动命令是`java -jar xxx.jar` ，请将其制作成Docker镜像，并启动。



## 答案

```dockerfile
# 基于哪个镜像
FROM java:8

# 将本地文件夹挂载到当前容器
VOLUME /tmp

# 拷贝文件到容器，也可以直接写成ADD xxxxx.jar /app.jar
ADD xxxxx.jar app.jar /app.jar'

# 声明需要暴露的端口
EXPOSE 8761

# 配置容器启动后执行的命令
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

