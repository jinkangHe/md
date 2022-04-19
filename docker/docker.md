## Docker相关知识

**docker：是一个引擎，类似于一个可以驱动操作系统的平台**

**image：操作系统镜像，一般是由很小的一个操作系统作为底层，然后安装特定的软件，打包成image**，**可以理解为模板模具

**container：容器，image实例化的产物，可以真正运行的系统，可以理解为由模板产生的实体**

下图进行一个类比

|  docker   |   java   |        解释        |
| :-------: | :------: | :----------------: |
|  docker   |   jvm    | 一个平台，一种引擎 |
|   image   |  class   |     抽象的物体     |
| container | instance |  由抽象生成的实例  |

镜像分为很多不同的版本（tag），使用imageName:tag区分

容器的状态分为

+ **创建 create**
+ **运行 up 27 hours**
+ **暂停 up 27 hours（Paused）** 
+ **退出 Exited**
+ ![image-20210527164345307](docker.assets/image-20210527164345307.png)

## Docker相关命令

### 启动docker

```shell
systemctl start docker
```

### 拉取镜像

```shell
docker image pull <imageName>:<tag>
```

从指定image仓库拉取镜像，默认为Docker Hub，其中第一个参数是镜像名称，第二个参数是镜像标签，如果不加默认为latest

![image-20210526093249594](docker.assets/image-20210526093249594.png)

### 查看镜像

```shell
docker images
docker image ls
```

![image-20210526093313832](docker.assets/image-20210526093313832.png)

### 删除镜像

```shell
docker image rm <imageName>
```

![image-20210526135934157](docker.assets/image-20210526135934157.png)

### 创建容器

创建容器之后，容器并没有启动

```
docker create <imageName>
```

![image-20210526140526500](docker.assets/image-20210526140526500.png)

### 启动容器

```shell
# -a 表示输出到前台
sudo docker start -a 715
```

![image-20210526142113056](docker.assets/image-20210526142113056.png)

也可以直接使用run命令来运行，run = create+start

```shell
docker run hello-world
```

![image-20210526144202522](docker.assets/image-20210526144202522.png)

### 查看容器列表

```shell
# 查看运行中的容器
docker ps 
# 查看所有容器
docker ps -a
```

![image-20210526163607396](docker.assets/image-20210526163607396.png)

### 运行参数

实际上，启动一个容器往往需要设置很多参数，比如容器如果想和外部通信，则需要端口映射；如果想和外部的磁盘交互，则需要文件路径映射；启动时需要一些内部的设置，则需要设置环境变量，输入docker run --help可以看到所有参数设置

![image-20210526151110692](docker.assets/image-20210526151110692.png)

![image-20210526151130541](docker.assets/image-20210526151130541.png)

![image-20210526151148151](docker.assets/image-20210526151148151.png)

以下对几个常用的参数作一个大概说明

| 参数名 |        说明        |                             用法                             |
| :----: | :----------------: | :----------------------------------------------------------: |
|   -d   |      后台运行      |                                                              |
|   -p   |      端口映射      |  -p 3306:3306 docker主机上的3306端口映射容器里面的3306端口   |
|   -v   | 外部文件挂载(映射) | -v /etc/data:/data 将docker主机的 /etc/data 目录和容器内部的/data映射起来 |
|   -e   |      环境变量      |  这个可以在官方镜像里面的使用说明找到运行容器的一些环境变量  |

### 进入容器内部

![image-20210527162108672](docker.assets/image-20210527162108672.png)

### 停止容器

```shell
docker stop hello-world
```

![image-20210526163423898](docker.assets/image-20210526163423898.png)

### 杀掉容器进程

==推荐使用stop==

![image-20210527163249336](docker.assets/image-20210527163249336.png)

### 删除容器

删除容器之前请确保容器已经停止

```shell
docker rm hello-world
```

![image-20210526164324080](docker.assets/image-20210526164324080.png)

## Docker实战1——安装运行redis

### 1.获取镜像

可以去dockerhub 官方仓库https://registry.hub.docker.com/搜索需要的镜像，然后选择自己需要的版本

![image-20210526165531965](docker.assets/image-20210526165531965.png)

本例使用5.0版本进行演示

![image-20210527151949267](docker.assets/image-20210527151949267.png)

### 2.目录映射

下载好镜像之后，可以在官方实例看到一些基础的命令和相关配置

![image-20210527152821702](docker.assets/image-20210527152821702.png)

redis启动时需要配置文件你，如果我们想指定某些配置，需要做一个配置文件的映射；如果想持久化数据也可以做一个持久化目录的映射，可以看到容器里配置文件的目录是**/usr/local/etc/redis**，持久目录是**/data**，我们在docker主机上建立对应的目录映射起来

![image-20210527153254653](docker.assets/image-20210527153254653.png)

这是我的主机目录配置

### 3.配置文件配置

在创建好配置文件的目录之后，在配置文件目录里面创建一个名为**redis.conf**的文件，加入自己需要的配置

![image-20210527154614868](docker.assets/image-20210527154614868.png)

更多详细配置请参考redis官网

### 4.启动

|                           启动参数                           |                说明                |
| :----------------------------------------------------------: | :--------------------------------: |
| **-v /home/hejinkang/docker/redis5.0/conf:/usr/local/etc/redis** |          **配置文件映射**          |
|      **-v /home/hejinkang/docker/redis5.0/data:/data**       |         **持久化目录映射**         |
|                       **-p 6780:6739**                       | **端口映射，主机6380映射容器6379** |
|                            **-d**                            |            **后台运行**            |
|                     **--name redis5.0**                      |      **设置容器名为redis5.0**      |
|                     **--appendonly yes**                     |       **开启aof持久化模式**        |

最后命令如下

```shell
docker run -v /home/hejinkang/docker/redis5.0/conf:/usr/local/etc/redis -v /home/hejinkang/docker/redis5.0/data:/data -p 6380:6379 -d --name redis5.0 redis:5.0 redis-server /usr/local/etc/redis/redis.conf --appendonly yes
```

运行会返回容器id，查看容器进程列表能找到该进程

![image-20210527160134995](docker.assets/image-20210527160134995.png)

### 5.客户端连接测试

客户端测试输入服务器ip，端口和设置好的密码，可以看到redis连接成功显示信息

![image-20210527160306367](docker.assets/image-20210527160306367.png)

连接信息

![image-20210527160418226](docker.assets/image-20210527160418226.png)

## Docker实战2——安装运行mysql

安装mysql和redis大同小异，重复部分不再赘述，特殊之处在于启动的时候可以通过环境变量设置root密码，其他环境变量也可以在官网上找到

![image-20210527161639780](docker.assets/image-20210527161639780.png)

下面是使用环境变量的启动命令，仅供参考，可以查询官网获取更多信息

```shell
sudo docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.0
```

