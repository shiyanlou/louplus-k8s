# 挑战：在 Swarm 集群中使用 Compose 部署简单的服务

安装 Docker Swarm 和 Docker Compose：

```bash
# 安装 docker swarm 集群
$ docker swarm init --advertise-addr=eth0
# 安装 docker compose
$ wget http://labfile.oss-cn-hangzhou.aliyuncs.com/courses/980/software/docker-compose-Linux-x86_64
$ sudo mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ mkdir challenge && cd challenge
$ touch docker-compose.yml
```

docker-compose.yml 文件内容如下所示：

```yml
# docker compose 的版本为 3
version: "3"

# 由 3 个服务共同组成：db、wordpress、visualizer
services:
  db:
    image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/mysql:5.7
    volumes:
      - db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    depends_on:
      - db
    image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/wordpress:latest
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    deploy:
      mode: replicated
      replicas: 3

  visualizer:
    image: registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  db-data:
```

执行部署：

```bash
$ docker stack deploy -c docker-compose.yml wordpress
Creating network wordpress_default
Creating service wordpress_db
Creating service wordpress_visualizer
Creating service wordpress_wordpress
$ docker service ls
ID                  NAME                   MODE                REPLICAS            IMAGE                             PORTS
wy5rdzqnytxx        wordpress_db           replicated          1/1                 mysql:latest
ymees9w7hbt6        wordpress_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
xjqwpfnd9y0t        wordpress_wordpress    replicated          3/3                 wordpress:latest                  *:80->80/tcp
$ docker stack ls
NAME                SERVICES            ORCHESTRATOR
wordpress           3                   Swarm
```

在浏览器访问 `localhost:8080`，可以看到在当前节点运行的服务的可视化图形：

![此处输入图片的描述](https://doc.shiyanlou.com/courses/1457/600404/17235af8c0bc76110b93b2be1564f37c/wm)

![此处输入图片的描述](https://doc.shiyanlou.com/courses/1457/600404/db6ca14da9bcd323a4c048a18f9231b3/wm)

由于当前只有一个节点，所以所有服务都运行在这个节点上。

最后移除服务，运行命令：

```bash
docker stack down wordpress
```
