---


title: Docker中的volume到底是个什么东西？
toc: true
date: 2021/05/02 19:40:01
categories:
- docker
---

## 概述

container中数据都是在可写层上，如果container不存在，那么数据也将会不存在。我们一般都需要把数据保存至host上，并且持久保存。Docker提供了几种保持数据的方式

Docker中的保存数据类型分为四种：

1. volumes  （数据卷）
2. bind-mounts  （绑定挂载）
3. tmpfs （只存在内存中，不会写入文件系统）
4. pipes （命名管道。用户host和container之前的通信，通常是在container中运行第三方工具）

<!-- more -->

## volumes

存在host文件系统中，和 `bind-mounts` 工作方式类似，这种类型是由Docker管理。默认是存在`/var/lib/docker/volumes`目录下。

* 可以和多个容器共享数据

* 不会自动删除，即使container被干掉，除非手动删除volume

* 创建volume时，如未指定名称，Docker会为其随机生成名称（称为匿名卷，指定名称称为命名卷）

* 不会增加使用volume的container大小

* 容易备份和迁移数据

### Docker CLI  

手动显示创建

``` bash
$ docker volume create dongle-vo
```

使用`-v`让Docker自行创建

```bash
$ docker run --name nginx -v dongle-vo1:/etc/nginx/conf.d  nginx:alpine
```

或者使用`--mount`创建(推荐，相比`-v`语义化，明确)

```bash
$ docker run --name nginx --mount type=volume,source=dongle-vo1,target=/etc/nginx/conf.d  nginx:alpine 
```

> type值有三种：volume（卷），bind（绑定挂载，也就是bind-mount），tmpfs（内存方式）。 
>
> source：对于volume是命名卷的名称，而对于bind类型是host上的绝对路径（不存在会报错）可以简写为src
>
> target：必需，容器路径（绝对路径）可以简写为dst
>
> 其他具体参数请查看[官方文档](https://docs.docker.com/storage/)
>
> 不显示指定type值，默认是volume。不指定source则按创建匿名卷方式挂载



运行一个Nginx容器，创建一个名称为 `dongle-vo1` 的命名卷，挂载到container `/etc/nginx/conf.d`目录中。

> 如果volume中没有内容，container中有内容。Docker则将container中的内容复制至volume中。这点是不同于`bind-mount`，`bind-mount`类型的volume中的内容会覆盖container内容

查看volume列表

```bash
$ docker volume ls

local     dongle-vo1
```

查看volume

```bash
$ docker inspect dongle-vo1

[
    {
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/dongle-vo1/_data",
        "Name": "dongle-vo1",
        "Options": null,
        "Scope": "local"
    }
]

```

进入挂载点查看挂载的内容

```bash
$ cd /var/lib/docker/volumes/dongle-vo1/_data

default.conf
```

### Dockerfile

那Dockerfile里的VOLUME 是怎么用的?

```shell Dockerfile
# 这俩个效果是一样的
VOLUME dongle-vo
VOLUME ["/dongle-vo"]
```

VOLUME可以指定多个。Docker会创建匿名卷挂载到container中的`dongle-vo`目录，也就是说这里指定的目录是container中的，不存在则创建。container目录里有内容会被复制到匿名卷中。相当于 `docker run -v dongle-vo ` 或者  `docker run --mount target=/dongle-vo`

那我可不可以用VOLUME这个绑定host目录呢？是不可以的，挂载host目录是依赖于host系统的，并不能保证通用性。

指定host目录或者命名卷可以这样`docker run --mount type=volume,source=dongle-vo1,target=/dongle-vo`，使用`-v`也是一样的

### docker-compose

前者为host目录，后者container目录

```yaml docker-compose
services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    volumes:
      # 匿名卷方式挂载container中的dongle-vo目录
      - dongle-vo
      # 命名卷挂载container中的目录
      - dongle-vo:/etc/nginx
# 命名卷不存在则用这种方式让docker自行创建      
volumes:
  dongle-vo:
```



## bind-mounts

绑定挂载可以在host上任何位置，需要自行维护绑定挂载的目录，使用绑定挂载时，host上的文件内容会被复制到container中。挂载的文件或目录不存在，则创建

### Docker-CLI

```bash
$ docker run -v ~/dongle:/etc/nginx 
```

`-v`可以用`~`家目录方式或者绝对路径

```bash
$ docker run --mount type=bind,src=/root/dongle,dst=/etc/nginx/conf.d

```

`--mount`source必须是host上的绝对路径，如果不存在则会报错。俩种方式都可以使用`$(pwd)`表示当前目录

### docker-compose

docker-compose使用bind-mount方式，则是这样

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    volumes:
      # 绝对路径挂载
      - /home/dongle-vo:/etc/nginx/conf.d
      # 挂载当前目录下的dongle-vo
      - ./dongle-vo:/etc/nginx/conf.d
      # 家目录下的dongle-vo
      - ~/dongle-vo:/etc/nginx/conf.d
```

Docker绑定挂载卷时，对于不存在的文件或目录，会始终为其创建目录

bind-mount绑定挂载需要注意几点： 

| host             | container        | 结果                                   |
| :--------------- | :--------------- | :------------------------------------- |
| 文件或目录不存在 | 文件或目录不存在 | 在host和container均创建目录            |
| **文件**不存在   | 文件存在         | 在host中为其创建一个目录，启动容器报错 |
| 文件存在         | 文件不存在       | 复制host文件至container中              |
| 文件存在         | 文件存在         | host文件覆盖container文件              |

挂载host上的空目录，那么container中目录内容会被host覆盖。
