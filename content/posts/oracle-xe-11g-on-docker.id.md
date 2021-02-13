---
title: "Oracle XE 11g dengan Docker"
date: 2017-07-23T06:26:06+07:00
categories: ["Mini Tips"]
tags: ["Oracle", "Docker"]
draft: false
aliases: [/id/post/2017-07-23-oracle-xe-11g-on-docker/]
toc: true
---


```shell
$ docker pull wnameless/oracle-xe-11g
$ docker run -d -p 1022:22 -p 1521:1521 -e ORACLE_ALLOW_REMOTE=true wnameless/oracle-xe-11g
$ docker ps -a
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS                    PORTS               NAMES
4fe7a1024ed3        wnameless/oracle-xe-11g   "/bin/sh -c '/usr/..."   5 weeks ago         Exited (137) 4 days ago                       practical_williams
$ docker start practical_williams
# or
$ docker start 4fe7a1024ed3
```

<!--more-->

Konfigurasi koneksi database :
```
hostname: localhost
port: 1521
sid: xe
username: system
password: oracle
```

Password untuk `SYS` dan `SYSTEM` :
```
oracle
```

Login dengan SSH :
```
ssh root@localhost -p 1022
password: admin
```

Sumber: https://github.com/wnameless/docker-oracle-xe-11g
