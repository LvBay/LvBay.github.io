---
title: docker-compose部署wordpress
date: 2019-02-13 19:18:37
tags:
---

# 体验wordpress

使用docker-compose安装wordpress

## 安装docker

移除旧的版本：

    $ sudo yum remove docker \
        docker-client \
        docker-client-latest \
        docker-common \
        docker-latest \
        docker-latest-logrotate \
        docker-logrotate \
        docker-selinux \
        docker-engine-selinux \
        docker-engine

安装一些必要的系统工具：

    sudo yum install -y yum-utils device-mapper-persistent-data lvm2

添加软件源信息：

    sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

更新 yum 缓存：

    sudo yum makecache fast

安装 Docker-ce：

    sudo yum -y install docker-ce

启动 Docker 后台服务

    sudo systemctl start docker

## 安装docker-compose

需要先安装企业版linux附加包（epel)

    yum -y install epel-release

安装pip

    yum -y install python-pip

更新pip

    pip install --upgrade pip

安装docker-compose

    pip install docker-compose

## docker-compose.yml

``` yml
version: "3"
services:

   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: wordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
  db_data:

```

## 遇到的坑

中间出现过一次Access denied for user 'wordpress'@'172.18.0.3'的错误，原因是之前有一份yml文件，设置的 MYSQL_PASSWORD和本次不一样。

网上资料上的解决方案是

``` bash
docker-compose stop
docker-compose rm -v 删除之前容器的的volumes文件（一般是在/var/lib/docker/volumes）
```

但是我使用 docker-compose rm -v，并没有删除文件

最后我索性直接把docker卸载重装了