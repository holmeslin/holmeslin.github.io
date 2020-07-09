---
title: "Nginx 相關設定"
date: 2020-07-09T14:20:56+08:00
lastmod: 2020-07-09T14:20:56+08:00
keywords: []
description: ""
tags: ["nginx"]
categories: ["紀錄"]
author: ""
---

<!--more-->

## 防盗連

```shell
location ~* \.(gif|jpg|png)$ {
    # 只允許某一個請求資源
    valid_referers none blocked 192.168.0.1;
    if ($invalid_referer) {
       rewrite ^/ http://$host/logo.png;
    }
}
```

## 根據文件類型設置過期時間

```shell
location ~.*\.css$ {
    expires 1d;
    break;
}
location ~.*\.js$ {
    expires 1d;
    break;
}

location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ {
    access_log off;
    expires 15d;    #保存15天
    break;
}
# 測試圖片的max-age
# curl -x 127.0.0.1:80 \
  http://www.test.com/static/image/common/logo.png -I 
```

## 靜態資源訪問

```shell
http {
    # 这个将为打开文件指定缓存，默认是没有启用的，max 指定缓存数量，
    # 建议和打开文件数一致，inactive 是指经过多长时间文件没被请求后删除缓存。
    open_file_cache max=204800 inactive=20s;

    # open_file_cache 指令中的inactive 参数时间内文件的最少使用次数，
    # 如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个
    # 文件在inactive 时间内一次没被使用，它将被移除。
    open_file_cache_min_uses 1;

    # 这个是指多长时间检查一次缓存的有效信息
    open_file_cache_valid 30s;

    # 默认情况下，Nginx的gzip压缩是关闭的， gzip压缩功能就是可以让你节省不
    # 少带宽，但是会增加服务器CPU的开销哦，Nginx默认只对text/html进行压缩 ，
    # 如果要对html之外的内容进行压缩传输，我们需要手动来设置。
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml;

    server {
        listen       80;
        server_name www.test.com;
        charset utf-8;
        root   /data/www.test.com;
        index  index.html index.htm;
    }
}
```

## log 配置

### log 字段說明

| 字段                                | 说明                                                                  |
| ----------------------------------- | --------------------------------------------------------------------- |
| remote_addr 和 http_x_forwarded_for | 客户端 IP 地址                                                        |
| remote_user                         | 客户端用户名称                                                        |
| request                             | 请求的 URI 和 HTTP 协议                                               |
| status                              | 请求状态                                                              |
| body_bytes_sent                     | 返回给客户端的字节数，不包括响应头的大小                              |
| bytes_sent                          | 返回给客户端总字节数                                                  |
| connection                          | 连接的序列号                                                          |
| connection_requests                 | 当前同一个 TCP 连接的的请求数量msec日志写入时间。单位为秒，精度是毫秒 |
| pipe                                | 如果请求是通过HTTP流水线(pipelined)发送，pipe值为"p"，否则为"."       |
| http_referer                        | 记录从哪个页面链接访问过来的                                          |
| http_user_agent                     | 记录客户端浏览器相关信息                                              |
| request_length                      | 请求的长度（包括请求行，请求头和请求正文）                            |
| time_iso8601ISO8601                 | 标准格式下的本地时间                                                  |
| time_local                          | 记录访问时间与时区                                                    |

### access_log

```shell
http {
    log_format  access  '$remote_addr - $remote_user [$time_local] $host "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for" "$clientip"';
    access_log  /srv/log/nginx/talk-fun.access.log  access;
}
```

### error_log

```shell
error_log  /srv/log/nginx/nginx_error.log  error;
# error_log /dev/null; # 真正的关闭错误日志
http {
    # ...
}
```

### log 切割

```shell
# 和apache不同的是，nginx没有apache一样的工具做切割，需要编写脚本实现。
# 在/usr/local/sbin下写脚本

#!/bin/bash
dd=$(date -d '-1 day' +%F)[ -d /tmp/nginx_log ] || \
mkdir /tmp/nginx_log || \
mv /tmp/nginx_access.log /tmp/nginx_log/$dd.log ||\
/etc/init.d/nginx reload > /dev/null
```

## 反向代理

```shell
http {
    include mime.types;
    server_tokens off;

    ## 配置反向代理的参数
    server {
        listen    8080;

        ## 1. 用户访问 http://ip:port，则反向代理到 https://github.com
        location / {
            proxy_pass  https://github.com;
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

        ## 2.用户访问 http://ip:port/README.md，则反向代理到
        ##   https://github.com/zibinli/blog/blob/master/README.md
        location /README.md {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://github.com/zibinli/blog/blob/master/README.md;
        }
    }
}
```

## 禁止指定user_agent

```shell
#虚拟主机的配置文件里加入：

if ($http_user_agent ~* 'baidu|360|sohu') #禁止useragent为baidu、360和sohu，~*表示不区分大小写匹配
{
   return 403;
}

# location /  和  location  ~ /  优先级是不一样的。 
# 结合这个文章研究一下吧 http://blog.itpub.net/27181165/viewspace-777202/

# curl -A "baidu" -x127.0.0.1:80 www.test.com/forum.php -I 
# 该命令指定百度为user_agent,返回403

```

## nginx訪問控制

```shell
# 可以设置一些配置禁止一些ip的访问

deny 127.0.0.1;     #全局定义限制，location里的是局部定义的。如果两者冲突，以location这种精确地优先，

location ~ .*admin\.php$ {
    #auth_basic "cct auth";
    #auth_basic_user_file /usr/local/nginx/conf/.htpasswd;

    allow 127.0.0.1;  只允许127.0.0.1的访问，其他均拒绝
    deny all;
    include fastcgi_params;
    fastcgi_pass unix:/tmp/www.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME /data/www$fastcgi_script_name;
}
```

## 負載平衡

```shell
http {
    upstream test.net {
        ip_hash;
        server 192.168.10.13:80;
        server 192.168.10.14:80  down;
        server 192.168.10.15:8009  max_fails=3  fail_timeout=20s;
        server 192.168.10.16:8080;
    }
    server {
        location / {
            proxy_pass  http://test.net;
        }
    }
}
```
