---
title: 利用Docker和Nginx部署Hexo静态页面数据
toc: true 
date: 2021/04/30 20:39:01
categories:
- docker
---
本文讲的是如何用Docker部署hexo，为啥要用nginx呢？hexo是支持生成静态页面的，
访问速度上静态页面是快于```hexo s```这种开启服务式的。除了本文实现方式，还有另一种方式。
用docker部署nginx，挂载nginx静态文件目录（卷），在本地生成hexo静态文件，然后丢到挂载的目录里即可。
这种方式相对简约点，但是笔者并不想在本地生成文件丢到服务器上，所以在上式中多加了一个hexo容器，在这个容器里生成静态文件（同样也挂载生成的静态文件卷），只需拷贝新增的md文件至服务器卷中
<!--more-->

首先blog目录结构
```
blog
│  # 这个是主题配置文件，因主题而异
├─ _config.icarus.yml
├─ _config.landscape.yml
├─ _config.yml
├─ db.json
├─ docker-compose.yml
├─ dockerfiles
│    └─ NodeFile
├─ package-lock.json
├─ package.json
├─ yarn.lock
├─ node.sh
│  # md文件
├─ source
│  # 主题配置
└─ themes
```

## 准备环境
Docker
docker-compose 

## 安装Docker
<a href="https://vxdf.icu/docker/docker-install/" target="_blank">可查看本文</a>

## 配置文件 
NodeFile 配置如下
```dockerfile NodeFile
FROM node:alpine
# 切换工作目录，后续的操作都会在这个目录执行
WORKDIR /root/blog
# 复制blog目录下所有的文件至工作目录
COPY . .
# 因为node镜像自带yarn，这一步设置一下仓库源，安装hexo
RUN yarn config set registry https://registry.npm.taobao.org \
    && yarn global add hexo-cli \
    && yarn
# 赋予可执行权限
RUN chmod +x node.sh
# 在 {WORKDIR} 中执行脚本 (是在docker容器里执行)
ENTRYPOINT ["/bin/sh"，"node.sh"]
# 声明容器运行时提供服务的端口，实际并不会自动进行映射宿主端口映射。
# 只是理解该镜像的守护端口，除非在run命令用 -p 绑定映射关系或者 -P 会随机映射宿主端口
EXPOSE 8080
# 指定命名卷 （容器中目录）
VOLUME ["/root/blog/source", "/root/blog/public"]
```

```shell node.sh
# 清除静态页面以及缓存
hexo clean
# 生成静态页面
hexo g
# 开启hexo 服务
hexo s

```

docker-compose.yml 配置
```yaml  docker-compose.yml 
version: "3.9"
services:
  blog:
    build: 
      # 镜像构建上下文，也就是 /home/admin/blog 目录
      context: .
      # 指定Dockerfile文件
      dockerfile: dockerfiles/NodeFile
    # 容器名称  
    container_name: blog
    # 镜像名称
    image: "blog"
    ports:
      - "8080:4000"
    # 挂载卷 (前者本机目录，后者容器内目录)
    volumes:
      # 绑定blog source目录,也就是md文件目录。后续新增的md文件丢到这里
      - "/home/admin/blog/source:/root/blog/source"
      # 绑定hexo生成静态文件目录，后续用来同步nginx
      - "/home/admin/nginx/html:/root/blog/public"
  nginx:
    # 为啥这里不需要Dockerfile文件呢? 
    # 没有Dockerfile的话会先从registry中心pull下来
    container_name: nginx
    image: "nginx:alpine"
    ports:
      - "443:443"
      - "80:80"
    volumes:
      # nginx配置
      - "/home/admin/nginx/conf/nginx.conf:/etc/nginx/nginx.conf"
      # 默认nginx server配置
      - "/home/admin/nginx/conf.d:/etc/nginx/conf.d"
      # nginx静态页面目录     
      - "/home/admin/nginx/html:/usr/share/nginx/html"
      # 证书位置
      - "/home/admin/nginx/ssl/cert:/etc/nginx/ssl/cert"
```
## Nginx配置
因为nginx.conf和default.conf文件在host上是不存在的，对于挂载host上不存在的文件，docker会在host上为其创建一个目录，并且启动容器报错。容器中的配置会被host覆盖。推荐挂载目录而不是文件，挂载单个文件时，使用vim命令修改并不会实时同步至container中。[具体查看](http://localhost:4000/docker/docker-volume)

我们host上并不存在这些配置文件，先后台启动一个nginx容器, 为了拷贝配置文件

```shell
docker run -d --name nginx nginx:alpine 
```
拷贝配置容器中的default.conf和nginx.conf至host中
```shell
docker cp nginx:/etc/nginx/conf.d /home/admin/nginx/conf.d
```
```shell
docker cp nginx:/etc/nginx/nginx.conf /home/admin/nginx/conf/nginx.conf 
```
当然，如果需要配置SSL的话，修改配置default.conf
```shell /home/admin/nginx/conf.d/default.conf
server {
    # 开启SSl
    listen 443 ssl;
    server_name yourdomain.com;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    # 注意，这个是container中的路径。如果不是绝对路径，默认是container中的/etc/nginx目录下
    ssl_certificate ssl/cert/you.pem;
    ssl_certificate_key ssl/cert/you.key;
    ssl_session_timeout 5m;
    # 加密套件类型
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    # 使用TLS协议类型
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page  404              404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
# HTTP强制重定向HTTPS
server {
   listen 80;
    server_name yourdomain.com; #需要将yourdomain.com替换成证书绑定的域名。
    rewrite ^(.*)$ https://$host$1; #将所有HTTP请求通过rewrite指令重定向到HTTPS。
    location / {
        index index.html index.htm;
    }
}
```


## 开始部署
将blog目录下的文件拷贝至服务器上的```/home/admin/blog```下
执行命令 ``dokcer-compose up -d`` 后台启动并运行所有的容器。

## 后续使用

服务启动好了，那我怎么更新文章呢？只需把需要更新的文章（md文件），拷贝至服务器`/home/admin/blog/source`下。

然后执行 `docker exec -it blog hexo g` 生成静态文件



