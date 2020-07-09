---
title: "RHEL(CentOS) 離線安裝套件"
date: 2020-07-09T14:14:35+08:00
lastmod: 2020-07-09T14:14:35+08:00
keywords: []
description: ""
tags: ["server","centos","offline"]
categories: ["紀錄"]
author: ""
---

<!--more-->

## RHEL(CentOS) 離線安裝套件

> 以 docker 為例

### 本機端

- 先 run container 並連入

    ```shell
    docker run --rm -it -v /data:/data centos:7 bash
    ```

- 設定 docker stable repository

    ```shell
    [root@1a1e5c5e761a /]# yum install -y yum-utils device-mapper-persistent-data lvm2
    [root@1a1e5c5e761a /]# 
    yum-config-manager --add-repo \ 
    https://download.docker.com/linux/centos/docker-ce.repo  
    ```

- 列出可安裝版本

    ```shell
    [root@1a1e5c5e761a /]# yum list docker-ce --showduplicates | sort -r
    docker-ce.x86_64            3:19.03.8-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:19.03.7-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:19.03.6-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:19.03.5-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:19.03.4-3.el7                     docker-ce-stable
    ...
    ```

- 把要安裝的套件下載下來

    ```shell
    [root@1a1e5c5e761a /]# yum install docker-ce-19.03.8 -y \
        --downloadonly \
        --downloaddir=/data/docker-ce
    ```

- 打包為 tar 

    ```shell
    [root@1a1e5c5e761a /]# tar cvf offline.tar /data/docker-ce/*.rpm
    ```

### 遠端

- 上傳所需檔案,並解壓 tar

    ```shell
    [root@docker ~]# tar xvf docker-offline.tar
    ```

- 安裝 rpm

    ```shell
    [root@docker ~]# rpm -ivh --replacepkgs --replacefiles *.rpm
    ```

- 啟動

    ```shell
    [root@docker ~]# systemctl enable docker && systemctl start docker
    ```
