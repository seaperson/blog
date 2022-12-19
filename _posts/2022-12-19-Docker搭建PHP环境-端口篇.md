---
layout: post
title:  "Docker搭建PHP环境-端口篇"
date:   2022-12-19 16:30:03 +0800
categories: Docker PHP Docker-compose
---
>本文构建起来的项目目前只支持端口访问

# 背景
前几天升级了下mac系统，结果php-fpm就起不来了，最后还是重装fpm才解决的。本来想着在本机上搭建php开发环境一来方便，二来不用被docker占用太多系统资源，现在看来本机环境还是存在很大缺陷：
- 容易受本地系统更新的影响
- 移植性不高，换一台设备就得重新安装lnmp一整套，谁知道会出来多少问题

再想想docker的好处，打包好镜像放到镜像仓库后，换了设备只要安装docker就可以一键部署，不由得心头一热，心动不如行动，那就干吧！

# 系统环境

- Mac Monterey 12.6.1
- Docker Desktop 4.15.0 
- docker 20.10.21
- Docker Compose version v2.13.0

# 过程

## 项目结构
```
.
├── docker-compose.yml # docker-compose 配置文件
├──log
    ├── nginx   # nginx日志目录
    ├── php-fpm # php-fpm日志目录
├── mysql
│   ├── conf.d # mysql 配置文件目录
│   │   ├── my.cnf
│   └── data # mysql 数据文件目录
├── nginx
│   ├── conf.d # nginx 配置目录
│   │   ├── hosts.conf
├── php
│   ├── conf # php 配置目录
│   │   └── php.ini
    |   └── php-fpm.conf
│   └── Dockerfile # php 的 Dockerfile 配置
```

## docker-compose.yml
```
version: "3"

networks: 
  dev:

services:

##################nginx################################
  nginx:
      image: nginx:alpine
      container_name: nginx
      restart: always
      links:
        - php56
      networks:
        - dev
      ports:
        - "80:80"
      volumes:
        # 1. 挂载项目目录
        - /Volumes/kaifa/php:/var/www
        # 2. 挂载配置文件目录
        - ./nginx/conf.d:/etc/nginx/conf.d
        # 3. 挂载日志目录
        - ./log/nginx:/var/log/nginx
  ######################################################

  #################php################################
  php56:
      image: ./php
      container_name: php56
      restart: always
      networks:
        - dev
      ports:
        - "9000:9000"
      volumes:
        - /Volumes/kaifa/php:/var/www
        - ./php/php.ini:/usr/local/etc/php/php.ini
        - ./log/php-fpm:/var/log/php-fpm
  #####################################################

  #################php################################
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    networks:
      - ydxs-dev
    ports:
      - "3307:3306"
    expose:
      - "3306"
    volumes:
      - ./mysql/conf.d:/usr/local/etc/mysql/conf.d
      - ./mysql/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: 123456 # 改为自定义密码
  #####################################################
```
### mysql
只需要将配置目录及数据目录做下映射即可
### php
php需要开启一些常用扩展，所以需要在原始基础镜像上进行依赖安装或配置，Dockerfile如下：
```
FROM php:5.6-fpm-alpine as php56

#替换国内镜像
RUN sed -i "s/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g" /etc/apk/repositories

#时区配置
ENV TIMEZONE Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TIMEZONE /etc/localtime
RUN echo $TIMEZONE > /etc/timezone

RUN apk update && apk upgrade \
    && apk add tzdata \
    && apk --no-cache add pcre-dev ${PHPIZE_DEPS} \
    && pecl install xdebug-2.5.0 \
    && docker-php-ext-enable xdebug \
    apk add --no-cache freetype-dev libjpeg-turbo-dev libpng-dev jpeg-dev libwebp-dev gettext-dev libmcrypt-dev libmemcached-dev zlib-dev curl-dev libxml2-dev  \
    && docker-php-ext-install -j$(nproc) bcmath mbstring opcache pdo pdo_mysql mysqli iconv mcrypt gettext curl zip xml simplexml sockets hash soap \
    && docker-php-ext-configure gd  --with-freetype-dir  --with-freetype-dir=/usr/include/ --with-png-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-zlib-dir=/usr  \
    && NPROC=$(grep -c ^processor /proc/cpuinfo 2>/dev/null || 1)  \
    && docker-php-ext-install -j${NPROC} gd

EXPOSE 9000
CMD ["php-fpm"]
```

### nginx
映射nginx的配置：
```
server {
         listen       82;
         server_name  sample.com;
         root   /var/www/yunkerefactor/extapi/web;

         access_log /var/log/nginx/sample.com.access.log;
         error_log /var/log/nginx/sample.com.error.log;

        location / {
             index  index.php index.html index.htm;
         }


         location ~ \.php(.*)$ {
              fastcgi_pass   php56:9000;
              fastcgi_index  index.php;
              fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
              fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
              fastcgi_param  PATH_INFO  $fastcgi_path_info;
              fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
              include        fastcgi_params;
          }

}
```

## 构建验证
>docker-compose up

![docker-compose](https://raw.githubusercontent.com/seaperson/blog-pictures/main/202212191645696.png)
构建成功！

浏览器输入http://localhost
![phpinfo](https://raw.githubusercontent.com/seaperson/blog-pictures/main/202212191647851.png)
成功访问！

# 最后
![](https://raw.githubusercontent.com/seaperson/blog-pictures/main/202212191648046.png)
看了下docker的系统资源占用情况，真香！