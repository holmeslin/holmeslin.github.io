---
title: "編譯原始碼安裝 Nginx 在 REHL / CentOS 環境下"
date: 2020-07-09T14:01:14+08:00
lastmod: 2020-07-09T14:01:14+08:00
draft: true
keywords: []
description: ""
tags: ["nginx","server","centos"]
categories: ["紀錄"]
author: ""
---

<!--more-->

## REHL / CentOS 7 編譯原始碼安裝 Nginx

### 安裝所需編譯器套件

```shell
yum -y install gcc gcc-c++ make zlib-devel pcre-devel openssl-devel 
```

> 如需離線安裝 可參考 [這一篇](https://holmeslin.github.io/post/2020/07/server-offlne-install-software/)

### 下載原始碼

```shell
mkdir /opt/nginx && cd /opt/nginx
wget http://nginx.org/download/nginx-1.X.X.tar.gz
tar zxvf nginx-1.X.X.tar.gz
```

> 如需要裝 Module , 以 nginx-module-vts 模組為例

```shell
wget https://github.com/vozlt/nginx-module-vts/archive/v0.1.18.tar.gz
tar zxvf nginx-module-vts-0.1.18.tar.gz
```

### 編譯安裝

```shell
cd /opt/nginx/nginx-1.x.x/
./configure \
    --user=nginx \
    --group=nginx \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --lock-path=/var/run/nginx.lock \
    --with-http_ssl_module \
    --with-pcre
make && make install
useradd -d /etc/nginx/ -s /sbin/nologin nginx
```

> 如要裝模組 , configure 需加上 --add-module=/opt/nginx/nginx-module-vts

### 啟動檔

> systemd

```shell
vi /usr/lib/systemd/system/nginx.service

# 加入以下內容
[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target

# 啟動
systemctl start nginx
systemctl enable nginx
```

> init.d

```shell
vi /etc/init.d/nginx
加入以下內容

#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# pidfile:     /var/run/nginx.pid
# user:        nginx
# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/sbin/nginx"
prog=$(basename $nginx)
NGINX_CONF_FILE="/etc/nginx/nginx.conf"
lockfile=/var/run/nginx.lock

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
   *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac

# 啟動
chmod +x /etc/init.d/nginx
chkconfig --add nginx
chkconfig nginx on
```
