# 实战：修改Nginx首页

## 6.1 需求

* 启动一个Nginx容器。
* 将Nginx容器的首页改为`Welcome to 51CTO docker class` 。
* 将容器保存下来。





## 6.2 提示

* Nginx默认首页目录在：`/usr/share/nginx/html/index.html` 




## 答案

```
docker exec -it nginx容器ID /bin/bash   # 进入容器
```

执行如下命令，修改/usr/share/nginx/html/index.html

```
tee index.html <<-'EOF'
Welcome to 51CTO docker class
EOF
```

