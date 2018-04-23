---
layout:     post
title:      "基于 docker 搭建简单的流媒体服务器 "
subtitle:   " \"教你基于 docker 搭建一个简单的Nginx-rtmp流媒体服务器\""
date:       2018-04-14 23:30:00
author:     "Luob"
header-img: "img/post/post-2018-hello-docker-media.jpg"
catalog: false
tags:
    - docker
    - nginx
---

>开弓没有回头箭，回头即是空。


## 前言
由于本人昨天把系统给换了然后目前正在做的一个业余项目需要牵扯到流媒体服务器，但是之前系统环境是 ubuntu，现在给换成 Arch 了，而 Arch 软件更新比较激进，所以会导致 nginx 一些第三方模块编译出问题。而我个人也懒得折腾了，就想到干脆用 docker 得了。

## 正文
这里只教你怎么做，如果想知道为什么这么做，请找[资料 (3j4wf4)](https://share.weiyun.com/5n8FemQ) 好好学习一下。<br>

首先安装 docker ，没有安装的话[点击这里](https://yeasy.gitbooks.io/docker_practice/content/install/)

安装完 docker 之后请安装 ffmpeg，因为等下需要用到 ffmpeg 进行推流（实际开发中有很多第三方的 SKD 比如七牛云、网易、腾讯等等）。安装教程自己谷歌。

接下来新建一个文件夹，然后里面创建三个文件：Dockerfile、nginx.conf、sources.list

Dockerfile：
```
FROM ubuntu

MAINTAINER lifetrut "lifetrut@yandex.com"

ADD sources.list /etc/apt/

RUN apt-get update
RUN apt-get install git gcc make wget libpcre3 libpcre3-dev openssl libssl-dev -y -q

WORKDIR /usr/local/src

RUN wget http://nginx.org/download/nginx-1.12.2.tar.gz

RUN tar xf nginx-1.12.2.tar.gz

RUN git clone https://github.com/arut/nginx-rtmp-module.git

WORKDIR /usr/local/src/nginx-1.12.2
RUN ./configure --add-module=../nginx-rtmp-module --with-http_flv_module --with-http_mp4_module
RUN make
RUN make install

ADD nginx.conf /usr/local/nginx/conf/

EXPOSE 80 1935
CMD [ "/usr/local/nginx/sbin/nginx", "-c", "/usr/local/nginx/conf/nginx.conf" ]
```

nginx.conf：
```
worker_processes  auto;
daemon off;
error_log  logs/error.log;
pid        logs/nginx.pid;

events {
    worker_connections  6000;
    use epoll;
    multi_accept on;
    accept_mutex off;
}


http {
    include       mime.types;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;

    keepalive_timeout  30;
    reset_timedout_connection on;
    client_body_timeout 10;

    gzip  on;
    gzip_comp_level 5;	

    server {
        listen       80;
        server_name  localhost;
      
        location /stat {
          rtmp_stat all;
          rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
           root /usr/local/src/nginx-rtmp-module;
        }
        location /control {
          rtmp_control all;
        }
        location /hls {
          types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
          }
          root html;
          expires -1;
        }
        location ~\.flv {
          flv;
        }
        location ~\.mp4 {
          mp4;
        }
    }  
}

rtmp {
  server {
    listen 1935;
    chunk_size 4096;

    application music {
      play /usr/local/nginx/html/music;
    }

    application play {
      play /usr/local/nginx/html/play;
    }

    application video {
      play /usr/local/nginx/html/video;
    }    

    application record {
      record all;
      record_path /usr/local/nginx/html/video;
      exec_record_done ffmpeg -y -i $path -codec copy -movflags +faststart /usr/local/nginx/html/video/$name.mp4;
    }

    application hls {
      live on;
      hls on;
      wait_key on;
      hls_path /usr/local/nginx/html/hls;
      hls_fragment 10s;
      hls_playlist_length 10s;
      hls_continuous on;
      hls_cleanup on;
      hls_nested on;
    }
  }
}

```
sources.list：
```
# deb cdrom:[Ubuntu 16.04 LTS _Xenial Xerus_ - Release amd64 (20160420.1)]/ xenial main restricted
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
```
<br>

**构建 docker 镜像**
> docker build -t nginx:hls .
<br>

**运行镜像**
> docker run -p9037:80 -p 1935:1935 -v /var/log/nginx:/usr/local/nginx/logs -v 用户流媒体文件目录:/usr/local/nginx/html/play -v 用户流媒体保存目录:/usr/local/nginx/html/video -v 用户音乐文件目录:/usr/local/nginx/html/music -d 镜像id

参数解释：<br>
-p：指定容器暴露的端口 用户主机IP:容器IP
-v：给容器挂载存储卷，挂载到容器的某个目录。通俗点就是共享文件夹 主机目录:容器目录
-d：指定容器运行于前台还是后台
<br>

**点播**
> ffplay rtmp://localhost:1935/play/a.mp4
<br>

**拉流播放：(直播)**
> ffplay rtmp://localhost:1935/play/a
<br>

**用 ffmpeg 模拟实现推流：**
> ffmpeg -re -i a.mp4 -acodec aac -f flv rtmp://localhost:1935/play/a

## 后记
这只是一个简单的例子，并不会有实质性的教学效果。

如果各位感兴趣的话可以学一学 nginx。

nginx 还有很多种玩法，流媒体服务器只是其中一种而已。

基于这个例子后面还可以有各种花式玩法（比如开启摄像头实施监控）。

由于这两天有事博客暂更了两天，以后正常开启。

—— Luob 后记于 2018.4
