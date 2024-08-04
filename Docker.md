## 理解容器

![img](image/Docker.assets/-17227827981153.assets)

## Docker 命令

![img](image/Docker.assets/-17227827981141.assets)

### 基本操作

docker container 操作可省略为 docker 操作

```Bash
#查看运行中的容器
docker ps
#查看所有容器
docker ps -a
#打印所有容器的id
docker ps -aq


#搜索镜像
docker search nginx
#下载镜像
docker pull nginx
#下载指定版本镜像
docker pull nginx:1.26.0
#查看所有镜像
docker image ls
docker images
#删除指定id的镜像
docker image rm e784f4560448
docker rmi e784f4560448


#运行一个新容器
docker run nginx
#停止容器
docker stop keen_blackwell
#启动容器
docker start 592
#重启容器
docker restart 592
#查看容器资源占用情况
docker stats 592
#查看容器日志
docker logs 592
#删除指定容器
docker rm 592
#删除所有容器
docker rm -f $(docker ps -aq)
#强制删除指定容器
docker rm -f 592
# 后台启动容器
docker run -d --name mynginx nginx
# 后台启动并暴露端口
docker run -d --name mynginx -p 80:80 nginx
# 进入容器内部
docker exec -it mynginx /bin/bash


# 提交容器变化打成一个新的镜像
docker commit -m "update index.html" mynginx mynginx:v1.0
# 保存镜像为指定文件
docker save -o mynginx.tar mynginx:v1.0
# 删除多个镜像
docker rmi bde7d154a67f 94543a6c1aef e784f4560448
# 加载镜像
docker load -i mynginx.tar 


# 登录 docker hub
docker login
# 重新给镜像打标签
docker tag mynginx:v1.0 spongehah/mynginx:v1.0
# 推送镜像
docker push spongehah/mynginx:v1.0
```

### docker run 参数

Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

> [COMMAND] [ARG...]具有默认值，多数情况下不用指定

| **参数**                   | **说明**                         |
| -------------------------- | -------------------------------- |
| -d                         | 后台启动                         |
| -p 80:80                   | 指定端口（对外：对内）           |
| -v /aaa/bbb:/xxx/yyy       | 目录挂载（对外：对内）           |
| -v bbb:/xxx/yyy            | 卷挂载（对外：对内）             |
| -e <环境变量/配置参数>=xxx | 传入环境变量/配置参数的值        |
| --network xxx              | 指定自定义网络                   |
| --restart always           | 设置启动 Docker 时自动启动该容器 |
| --name                     | 指定容器名称                     |

容器启动时，**在删除容器后数据会丢失**，不会保存容器内部修改的数据，且只能使用类似 echo xxx >> xxx 的方式修改文件内容

所以我们需要做目录挂载或者卷挂载：

- 目录挂载：对外部分是一个目录，将自动进行映射
- 卷挂载：对外部分会自动创建 /var/lib/docker/volumes/<volume-name>，自动映射对内的文件夹，与目录挂载不同的是：会在对外的卷目录下，**自动创建对内文件夹下本来就存在的文件**（常用于配置文件目录）
  - 可通过 `docker volume ls`命令查看所有的卷

> 在删除容器之后，若我们之后再启动相同的容器，使用相同映射的目录或卷，那么数据将会存在

### 示例：启动 Redis 主从

> 容器的具体启动命令可以参考 docker hub 中提供该 image 的官方建议，特别是-e 参数对应的环境变量，各镜像提供方的环境变量可能不同

```Bash
docker run -d -p 6379:6379 \
-v /app/rfd1:/bitnami/redis/data \
-e REDIS_REPLICATION_MODE=master \
-e REDIS_PASSWORD=123456 \
--network mynet --name redis01 \
bitnami/redis
docker run -d -p 6379:6380 \
-v /app/rfd2:/bitnami/redis/data \
-e REDIS_REPLICATION_MODE=slave\
-e REDIS_MASTER_HOST=redis01 \
-e REDIS_MASTER_PORT_NUMBER=6379 \
-e REDIS_MASTER_PASSWORD=123456 \
-e REDIS_PASSWORD=123456 \
--network mynet --name redis02 \
bitnami/redis
```

## docker network

### docker0 网桥网络

现在启动两个 Nginx，一个对外暴露 88 端口，一个对外暴露 99 端口，现在 nginx1 内部想要访问 nginx2，需要通过 **"主机的** **ip****:99"** 来访问，但是这样的访问方式相当于，出了容器，然后又进入容器，再去访问到 nginx2

Docker 自带有这些网络：

```Bash
[root@iZf8zd3w9o76ozrsaqcs3sZ ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
98057dc4fbf3   bridge    bridge    local
53c9770e7429   host      host      local
02432d62b292   none      null      local
```

我们可以使用以下命令查看 Docker 启动时自带的网络：

```Bash
[root@iZf8zd3w9o76ozrsaqcs3sZ ~]# ip a
...
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:72:a6:ed:8b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:72ff:fea6:ed8b/64 scope link 
       valid_lft forever preferred_lft forever
...
```

可以看到一个叫做 **docker0** 的网络，就是我们 docker network ls 中打印出的 **bridge** 网络

我们可以通过以下命令查看一个容器的细节，其中包括其网关地址及其分配的 ip 地址：

```Bash
docker [container] inspect <container>
```

在打印内容的最后：我们可以看到该容器分配的 ip 地址是：172.17.0.2

```Bash
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "MacAddress": "02:42:ac:11:00:02",
                    "NetworkID": "98057dc4fbf352ea29ad77dbf6f427f1b9c8d84d96b6e62154f347c195caed03",
                    "EndpointID": "8216d3a215eb82889ca9f8728a74b0a43a37b126d5cc00a5fb8d1500096ec831",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "DriverOpts": null,
                    "DNSNames": null
                }
            }
        }
    }
]
```

我们就可以通过 **"172.17.0.2：80（对内****端口****）"** 的方式去访问另一个容器了

### docker 自定义网络

但是，ip 由于各种原因可能会变化，所以我们一般采取域名访问的方式

但是 **docker0 默认不支持主机域名**

我们可以创建自定义网络，容器名就是稳定域名

```Bash
# 创建自定义网络
docker network create mynet
# 启动容器时加入自定义网络
docker run -d -p 88:80 --name app1 --network mynet nginx
```

这样该容器的网关地址以及 ip 地址就是该自定义网络内的地址了，自定义网络内的容器，可以通过容器名作为域名进行相互访问：`<containter-name:对内端口>`

## docker compose

### compose 命令

作用：我们可以通过 docker compose 来**管理容器**，不用记住繁琐的 docker run 命令，也可以通过 compose 来**批量启动/下线容器**

> docker compse [-f compose.yaml]：若不指定-f 参数，将会自动寻找当前路径下的 compose.yaml 作为配置文件

- 全部上线：docker compose up [-d]
- 全部下线：docker compose down
  - 下线时删除镜像和卷：docker compose down [--rmi all/service-name] [--v]
- 指定启动：docker compose start x1 x2 x3
- 指定下线：docker compose stop x1 x3
- 指定扩容：docker compose scale x2=3      将 x2 启动到 3 份

### compose 文件基本语法

官方手册：https://docs.docker.com/compose/compose-file/

compose.yaml： 以启动一个 WordPress 博客为示例：

```YAML
name: myblog
services: 
    mysql: 
        container_name: mysql
        image: mysql:8.0
        ports: 
            - 3306:3306
        environment: 
            - MYSQL_ROOT_PASSWORD=123456
            - MYSQL_DATABASE=wordpress
        volumes: 
            - mysql-data:/var/lib/mysql
            - /app/myconf:/etc/mysql/conf.d
        restart: always
        networks: 
            - blog

    wordpress: 
        container_name: wordpress
        image: wordpress:latest
        ports: 
            - 8080:80
        environment: 
            - WORDPRESS_DB_HOST=mysql
            - WORDPRESS_DB_USER=root
            - WORDPRESS_DB_PASSWORD=123456
            - WORDPRESS_DB_NAME=wordpress
        volumes: 
            - wordpress:/var/www/html
        restart: always
        networks: 
            - blog
        # 启动依赖项（即先启动指定services，再启动此service）
        depends_on: 
            - mysql
    
volumes: 
    mysql-data: 
    wordpress:
    
networks: 
    blog: 
```

> 注意事项：
>
> - services 中用到的卷挂载需要在最下面的顶级元素 volumes 中声明（目录挂载不用管）
> - services 中用到的网络需要在最下面的顶级元素 networks 中声明

## Dockerfile

官方手册：https://docs.docker.com/reference/dockerfile/

常见指令及其作用：

| 常见指令       | 作用                 |
| :------------- | :------------------- |
| **FROM**       | **指定镜像基础环境** |
| RUN            | 运行自定义命令       |
| CMD            | 容器启动命令或参数   |
| **LABEL**      | **自定义标签**       |
| **EXPOSE**     | **指定暴露端口**     |
| ENV            | 环境变量             |
| ADD            | 添加文件到镜像       |
| **COPY**       | **复制文件到镜像**   |
| **ENTRYPOINT** | **容器固定启动命令** |
| VOLUME         | 数据卷               |
| USER           | 指定用户和用户组     |
| WORKDIR        | 指定默认工作目录     |
| ARG            | 指定构建参数         |

示例：将一个 java jar 包打包成一个镜像

编写 Dockerfile：

```Dockerfile
FROM openjdk:17

LABEL author=spongehah

EXPOSE 8080

COPY app.jar /app.jar

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

使用 docker build 命令构建镜像：

```Bash
docker build -f Dockerfile -t myjavaapp:v1.0 .
# 参数全写：（最后一个 . 代表当前目录）
docker build --file Dockerfile --tag myjavaapp:v1.0 .
```

## 镜像分层机制

![img](image/Docker.assets/-17227827981152.assets)

同一个 image 启动的容器，他们具有独立的读写层，image 本身的大小是固定的大小，**启动的多个容器都会依赖于镜像本身的存储大小（virtual size）**，容器自身的修改会单独有一个读写层，用于存储自己的改变（将修改后的容器重新打 tag 为新镜像也是如此，共用的部分都是同一片存储）

```Bash
[root@iZf8zd3w9o76ozrsaqcs3sZ ~]# docker ps -s
CONTAINER ID   IMAGE         PORTS     NAMES            SIZE
a342de836047   mysql:8.0     3306      mysql-master-1   339B (virtual 603MB)
```

如上：该容器的存在实际只多占用了 339B 的空间

一个原则：能共用就共用

## NameSpace & Cgroups

- **NameSpace:** 
  - NameSpace 其实是一种实现不同进程间资源隔离的机制，不同 NameSpace 的程序，可以享有一份独立的系统资源
  - 两个不同的 namespaces 下部署了两个进程，这两个进程的 pid 可能是相同的
  - **Docker 的每个进程独享一份 namespaces**
  - 我们可以进入到 Linux 操作系统的/proc/$pid/ns 目录下去查看指定进程的 NameSpaces 信息，可以发现 **Linux 自己的不同进程的 namespace 信息是相同**的，而 **docker 下的进程间的 namespace 是不同的**
- **Cgroups**
  - Cgroups 全称 Control Groups，是 Linux 内核提供的**物理资源隔离机制**，通过这种机制，可以实现对 Linux 进程或者进程组的**资源限制、隔离、统计**功能

**Cgroups 的核心组成：**

1. cpu: 限制进程的 cpu 使用率
2. memory: 限制进程的 memory 使用量
3. ns: 控制 cgroups 中的进程使用不同的 namespace

> 举例：限制进程 cpu 的使用率：
>
> 1. cpu.shares： cgroup 对时间的分配。比如 cgroup A 设置的是 1，cgroup B 设置的是 2，那么 B 中的任务获取 cpu 的时间，是 A 中任务的 2 倍
> 2. cgroup.procs： 当前系统正在运行的进程（若是新建的 demo 目录案例，需要手动添加进程号）
> 3. cpu.cfs_period_us: 完全公平调度器的调整时间配额的周期，默认值是 100000
> 4. cpu.cfs_quota_us: 完全公平调度器的周期当中可以占用的时间，默认值是 -1 表示不做限制
>
>  cd /sys/fs/cgroup/cpu
>
> 进入到该目录，就可以使用 cat 。/xxx 命令查看 Linux 下上面参数的默认值
>
> **demo：** 先运行一个 java 程序，只有一个死循环 while true，然后通过 top 命令可以看到该进程占用 cpu 100% 然后在 cpu 目录下，新建 demo：mkdir demo，会自动在新目录下生成配套文件 使用 jps 查看 java 程序进程号 将 java 程序的进程号添加到 cgroup.procs：**echo [****pid****] > 。/cgroup.procs** cpu.cfs_period_us 的默认值是 100000，cpu.cfs_quota_us 默认值是 -1 表示不做限制，我们将 java 程序的 cpu 使用率限制为 10%，使用命令：**echo 10000 > 。/cpu.cfs_quota_us**，因为 10000/100000=0.1 再次使用 top 命令查看，发现 java 进程 cpu 只占用 10%左右

## 附录：常用 dev-soft compse 文件

准备工作：关闭系统交换区、设置最大卷映射数

```Bash
#Disable memory paging and swapping performance
sudo swapoff -a

# Edit the sysctl config file
sudo vi /etc/sysctl.conf

# Add a line to define the desired value
# or change the value if the key exists,
# and then save your changes.
vm.max_map_count=262144

# Reload the kernel parameters using sysctl
sudo sysctl -p

# Verify that the change was applied by checking the value
cat /proc/sys/vm/max_map_count
```

compose.yaml: 

> 注意：
>
> - 将下面文件中 `kafka` 的  `119.45.147.122` 改为你自己的服务器 IP。
> - 所有容器都做了时间同步，这样容器的时间和 linux 主机的时间就一致了

```YAML
name: devsoft
services:
  redis:
    image: bitnami/redis:latest
    restart: always
    container_name: redis
    environment:
      - REDIS_PASSWORD=123456
    ports:
      - '6379:6379'
    volumes:
      - redis-data:/bitnami/redis/data
      - redis-conf:/opt/bitnami/redis/mounted-etc
      - /etc/localtime:/etc/localtime:ro

  mysql:
    image: mysql:8.0.31
    restart: always
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    ports:
      - '3306:3306'
      - '33060:33060'
    volumes:
      - mysql-conf:/etc/mysql/conf.d
      - mysql-data:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro

  rabbit:
    image: rabbitmq:3-management
    restart: always
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=rabbit
      - RABBITMQ_DEFAULT_PASS=rabbit
      - RABBITMQ_DEFAULT_VHOST=dev
    volumes:
      - rabbit-data:/var/lib/rabbitmq
      - rabbit-app:/etc/rabbitmq
      - /etc/localtime:/etc/localtime:ro
  opensearch-node1:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node1 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - opensearch-data1:/usr/share/opensearch/data # Creates volume called opensearch-data1 and mounts it to the container
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 9200:9200 # REST API
      - 9600:9600 # Performance Analyzer

  opensearch-node2:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node2 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - opensearch-data2:/usr/share/opensearch/data # Creates volume called opensearch-data2 and mounts it to the container

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.13.0
    container_name: opensearch-dashboards
    ports:
      - 5601:5601 # Map host port 5601 to container port 5601
    expose:
      - "5601" # Expose port 5601 for web access to OpenSearch Dashboards
    environment:
      - 'OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]'
      - "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true" # disables security dashboards plugin in OpenSearch Dashboards
    volumes:
      - /etc/localtime:/etc/localtime:ro
  zookeeper:
    image: bitnami/zookeeper:3.9
    container_name: zookeeper
    restart: always
    ports:
      - "2181:2181"
    volumes:
      - "zookeeper_data:/bitnami"
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: 'bitnami/kafka:3.4'
    container_name: kafka
    restart: always
    hostname: kafka
    ports:
      - '9092:9092'
      - '9094:9094'
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://0.0.0.0:9094
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://119.45.147.122:9094
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - ALLOW_PLAINTEXT_LISTENER=yes
      - "KAFKA_HEAP_OPTS=-Xmx512m -Xms512m"
    volumes:
      - kafka-conf:/bitnami/kafka/config
      - kafka-data:/bitnami/kafka/data
      - /etc/localtime:/etc/localtime:ro
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    restart: always
    ports:
      - 8080:8080
    environment:
      DYNAMIC_CONFIG_ENABLED: true
      KAFKA_CLUSTERS_0_NAME: kafka-dev
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    volumes:
      - kafkaui-app:/etc/kafkaui
      - /etc/localtime:/etc/localtime:ro

  nacos:
    image: nacos/nacos-server:v2.3.1
    container_name: nacos
    ports:
      - 8848:8848
      - 9848:9848
    environment:
      - PREFER_HOST_MODE=hostname
      - MODE=standalone
      - JVM_XMX=512m
      - JVM_XMS=512m
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=nacos-mysql
      - MYSQL_SERVICE_DB_NAME=nacos_devtest
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos
      - MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
      - NACOS_AUTH_IDENTITY_KEY=2222
      - NACOS_AUTH_IDENTITY_VALUE=2xxx
      - NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789
      - NACOS_AUTH_ENABLE=true
    volumes:
      - /app/nacos/standalone-logs/:/home/nacos/logs
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      nacos-mysql:
        condition: service_healthy
  nacos-mysql:
    container_name: nacos-mysql
    build:
      context: .
      dockerfile_inline: |
        FROM mysql:8.0.31
        ADD https://raw.githubusercontent.com/alibaba/nacos/2.3.2/distribution/conf/mysql-schema.sql /docker-entrypoint-initdb.d/nacos-mysql.sql
        RUN chown -R mysql:mysql /docker-entrypoint-initdb.d/nacos-mysql.sql
        EXPOSE 3306
        CMD ["mysqld", "--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
    image: nacos/mysql:8.0.30
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=nacos_devtest
      - MYSQL_USER=nacos
      - MYSQL_PASSWORD=nacos
      - LANG=C.UTF-8
    volumes:
      - nacos-mysqldata:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "13306:3306"
    healthcheck:
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]
      interval: 5s
      timeout: 10s
      retries: 10
  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: prometheus
    restart: always
    ports:
      - 9090:9090
    volumes:
      - prometheus-data:/prometheus
      - prometheus-conf:/etc/prometheus
      - /etc/localtime:/etc/localtime:ro

  grafana:
    image: grafana/grafana:10.4.2
    container_name: grafana
    restart: always
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana
      - /etc/localtime:/etc/localtime:ro

volumes:
  redis-data:
  redis-conf:
  mysql-conf:
  mysql-data:
  rabbit-data:
  rabbit-app:
  opensearch-data1:
  opensearch-data2:
  nacos-mysqldata:
  zookeeper_data:
  kafka-conf:
  kafka-data:
  kafkaui-app:
  prometheus-data:
  prometheus-conf:
  grafana-data:
```

| 组件（容器名）                 | 介绍             | 访问地址                    | 账号/密码          | 特性                    |
| :----------------------------- | :--------------- | :-------------------------- | :----------------- | :---------------------- |
| Redis(redis)                   | k-v 库           | 你的 ip:6379                | 单密码模式：123456 | 已开启 AOF              |
| MySQL(mysql)                   | 数据库           | 你的 ip:3306                | root/123456        | 默认 utf8mb4 字符集     |
| Rabbit(rabbit)                 | 消息队列         | 你的 ip:15672               | rabbit/rabbit      | 暴露 5672 和 15672 端口 |
| OpenSearch(opensearch-node1/2) | 检索引擎         | 你的 ip:9200                |                    | 内存 512mb；两个节点    |
| opensearch-dashboards          | search 可视化    | 你的 ip:5601                |                    |                         |
| Zookeeper(zookeeper)           | 分布式协调       | 你的 ip:2181                |                    | 允许匿名登录            |
| kafka(kafka)                   | 消息队列         | 你的 ip:9092 外部访问：9094 |                    | 占用内存 512mb          |
| kafka-ui(kafka-ui)             | kafka 可视化     | 你的 ip:8080                |                    |                         |
| nacos(nacos)                   | 注册/配置中心    | 你的 ip:8848                | nacos/nacos        | 持久化数据到 MySQL      |
| nacos-mysql(nacos-mysql)       | nacos 配套数据库 | 你的 ip:13306               | root/root          |                         |
| prometheus(prometheus)         | 时序数据库       | 你的 ip:9090                |                    |                         |
| grafana(grafana)               |                  | 你的 ip:3000                | admin/admin        |                         |