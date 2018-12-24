# 实战：使用Dockerfile修改Nginx首页

创建一个Dockerfile，内容如下：

```dockerfile
FROM nginx
RUN echo '<h1>Spring Cloud与Docker微服务实战</h1>' > /usr/share/nginx/html/index.html
```

