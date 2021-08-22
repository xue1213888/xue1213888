---
title: "docker学习笔记"
date: 2021-08-10T14:52:05+08:00
tags: ["rpc","微服务","grpc","docker"]
categories: ["golang","微服务","docker"]
draft: false
---
# Docker 官方文档 学习笔记

## 开始学习

### 介绍

在这个笔记中，记录了使用的docker的每一个步骤。

- 构建一个镜像并以容器的形式运行。
- 上传镜像到DockerHub。
- 使用多个Docker容器连接同一个数据库部署一个Docker应用。
- 使用Docker Compose运行项目

### 下载并安装docker

#### Centos Docker安装

卸载系统上的docker保证从一个干净的环境开始

```shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine     
```

设置阿里云的镜像源来拉取docker

```                  shell
$ sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

安装docker

``` shell
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

显示出版本号证明安装成功

```shell
[root@VM-0-9-centos ~]# docker -v
Docker version 20.10.8, build 3967b7d
```

设置docker镜像加速，不然你下载镜像太慢了

```shell
[root@VM-0-9-centos ~]# vim /etc/docker/daemon.json
```

在json文件里面追加

```json
{
  "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

记得重启一下docker。

### 教程开始

运行一下命令开始我们的教程，我们运行一下docker run

```shell
[root@VM-0-9-centos ~]# docker run -d -p 80:80 docker/getting-started
Unable to find image 'docker/getting-started:latest' locally
latest: Pulling from docker/getting-started
540db60ca938: Pull complete
0ae30075c5da: Pull complete
9da81141e74e: Pull complete
b2e41dd2ded0: Pull complete
7f40e809fb2d: Pull complete
758848c48411: Pull complete
23ded5c3e3fe: Pull complete
38a847d4d941: Pull complete
Digest: sha256:10555bb0c50e13fc4dd965ddb5f00e948ffa53c13ff15dcdc85b7ab65e1f240b
Status: Downloaded newer image for docker/getting-started:latest
a84a9acc505419028aff855f3d861cad7c86b50487e2f298fafb6401ac26918b
```







### Docker 容器





### Docker 仓库
