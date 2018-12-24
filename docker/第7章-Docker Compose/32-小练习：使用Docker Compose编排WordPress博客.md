# 小练习：使用Docker Compose编排WordPress博客

```yaml
version: '2'
services:
  mysql:
    image: mysql:5.7
    expose:
      - "3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
  wordpress:
    image: wordpress
    ports:
      - "80:80"
    environment:
      - WORDPRESS_DB_HOST=mysql
      - WORDPRESS_DB_USER=root
      - WORDPRESS_DB_PASSWORD=123456
```



**WARNING**

这里，MySQL镜像只能用5.x的镜像，不能使用8.x的镜像。否则WordPress无法正常连接到MySQL。原因是：目前PHP 7.x（例如7.1.4）所使用的字符集与MySQL 8.x所使用的默认字符集不同：<https://bugs.php.net/bug.php?id=74461>