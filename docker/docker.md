## Docker相关知识

## Docker相关命令

### 拉取镜像

```shell
docker image pull <imageName>:<tag>
```

从指定image仓库拉取镜像，默认为Docker Hub，其中第一个参数是镜像名称，第二个参数是镜像标签，如果不加默认为latest

### 查看镜像

```shell
docker images
docker image ls
```

### 删除镜像

```shell
docker image rm <imageName>
```

## Docker实战——安装redis
