---
layout: article
title: LNMP环境下，利用Nginx反代Wikipedia
tags: ["VPS"]
page:
  key: Nginx-Wikipedia
  comment: true
---

>维基百科
>海纳百川，有容乃大
>人人可编辑的自由百科全书

<!--more-->

# 前记

之前一直使用ShadowsocksR代理网络，但是这需要安装软件，还是很麻烦的。

考虑到访问外网的主要需求为维基百科，所以就着手于制作一个维基镜像站。

了解到一共有两种方法，分别为**通过Kiwix访问本地存储的ZIM文件**和**通过Nginx反代wikipedia.org**。

前者不仅时效性差还需要占用大量空间，对于我寒酸的服务器来说是个大问题，因此采用了第二个方案。

特别感谢Johnson的指点、网上大佬们的教程，才有了如今[这个](https://wiki.ericdrive.ml)维基镜像站。

# 准备

1. 一台港澳台或国外的VPS
2. Linux系统（笔者使用的是`Ubuntu 18.04 LTS`）
3. LNMP环境（笔者使用的是`LNMP 1.6`，安装详见[LNMP官网](https://lnmp.org/install.html)）

# 教程

## 1. 为已安装的LNMP编译`ngx_http_substitutions_filter_module`模块

### 下载`ngx_http_substitutions_filter_module`模块

```bash
cd /root
git clone "https://github.com/yaoweibin/ngx_http_substitutions_filter_module.git"
```

### 编辑`lnmp.conf`文件

```bash
vim /root/lnmp1.6/lnmp.conf
```

在`Nginx_Modules_Options`一项后的单引号间添加  
`--add-module=/root/ngx_http_substitutions_filter_module`

### 查看`Nginx`版本号

```bash
nginx -V
```

### 安装`ngx_http_substitutions_filter_module`模块

```bash
cd /root/lnmp1.6
./upgrade.sh
```

选择`Update Nginx`

在`Current Nginx Version`中输入你当前的`Nginx`版本号

不停回车即可

编译结束时，输入`nginx -V`检查编译

你将看到`--add-module=/root/ngx_http_substitutions_filter_module`

## 2.添加vhost

### 确保相关域名已添加至你的DNS解析服务商

以本站为例，分别为：

`wiki.ericdrive.ml`

`*.wiki.ericdrive.ml`

实际使用的为：

`wiki.ericdrive.ml`

`www.wiki.ericdrive.ml`

`lang.wiki.ericdrive.ml`

`lang.m.wiki.ericdrive.ml`

`up.wiki.ericdrive.ml`

**注意：**

**将`ericdrive.ml`替换成你自己的域名**

**将`lang`替换为你希望添加的维基语言，如`en`、`zh`分别对应英文中文维基，下同**

### 新建配置文件

```bash
lnmp vhost add
```

依次添加

```
wiki.ericdrive.ml
```

```
www.wiki.ericdrive.ml
lang.wiki.ericdrive.ml
lang.m.wiki.ericdrive.ml
up.wiki.ericdrive.ml
```

路径默认即可

选`n`直到出现`Add SSL Certificate (y/n)` 一项，选`y`

```
1: Use your Own SSL Certificate and Key  
2: Use Let's Encrypt to create SSL Certificate and Key  
Enter 1 or 2:
```

然后填2

完成配置即可

## 3.编辑配置文件

### 配置`wiki.ericdrive.ml.conf`

```bash
vim /usr/local/nginx/conf/vhost/wiki.ericdrive.ml.conf
```

### 用以下代码覆盖原文件

**注意：**

**将`ericdrive.ml`替换成你自己的域名**

```nginx
server {
    server_name up.wiki.ericdrive.ml;
    listen 80;
    listen 443 ssl http2;
    resolver 8.8.8.8;
#    resolver 127.0.0.1 valid=30s;

    ssl_certificate /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/fullchain.cer;
    ssl_certificate_key /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/wiki.ericdrive.ml.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
    ssl_session_cache builtin:1000 shared:SSL:10m;
    # openssl dhparam -out /usr/local/nginx/conf/ssl/dhparam.pem 2048
    ssl_dhparam /usr/local/nginx/conf/ssl/dhparam.pem;
    
    if ($http_x_forwarded_proto = 'http')
    {
        return 301 $server_name$request_uri;
    }

    location / {
        proxy_pass https://upload.wikimedia.org;
        proxy_cookie_domain upload.wikimedia.org up.wiki.ericdrive.ml;
        proxy_buffering off;
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header referer "https://upload.wikimedia.org$request_uri";
    }
}

server {
    server_name  wiki.ericdrive.ml;
    listen 80;
    listen 443 ssl http2;
    resolver 8.8.8.8;

    ssl_certificate /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/fullchain.cer;
    ssl_certificate_key /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/wiki.ericdrive.ml.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
    ssl_session_cache builtin:1000 shared:SSL:10m;
    # openssl dhparam -out /usr/local/nginx/conf/ssl/dhparam.pem 2048
    ssl_dhparam /usr/local/nginx/conf/ssl/dhparam.pem;
    
    if ($http_x_forwarded_proto = 'http')
    {
        return 301 $server_name$request_uri;
    }

    location / {
        proxy_pass https://www.wikipedia.org;
        proxy_buffering off;

        proxy_redirect https://www.wikipedia.org/ http://wiki.ericdrive.ml/;
        proxy_cookie_domain www.wikipedia.org wiki.ericdrive.ml;
        proxy_redirect ~^https://([\w\.]+).wikipedia.org/(.*?)$ http://$1.wiki.ericdrive.ml/$2;

        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Accept-Encoding '';
        proxy_set_header referer "https://$proxy_host$request_uri";

        subs_filter_types text/css text/xml text/javascript application/javascript application/json;
        subs_filter .wikipedia.org .wiki.ericdrive.ml;
        subs_filter //wikipedia.org //wiki.ericdrive.ml;
        subs_filter upload.wikimedia.org up.wiki.ericdrive.ml;
    }
}

server {
    server_name  ~^(?<subdomain>[^.]+)\.wiki\.ericdrive\.ml$;
    listen 80;
    listen 443 ssl http2;
    resolver 8.8.8.8;

    ssl_certificate /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/fullchain.cer;
    ssl_certificate_key /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/wiki.ericdrive.ml.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
    ssl_session_cache builtin:1000 shared:SSL:10m;
    # openssl dhparam -out /usr/local/nginx/conf/ssl/dhparam.pem 2048
    ssl_dhparam /usr/local/nginx/conf/ssl/dhparam.pem;
    
    if ($http_x_forwarded_proto = 'http')
    {
        return 301 $subdomain.wiki.ericdrive.ml$request_uri;
    }

    location / {
        proxy_pass https://$subdomain.wikipedia.org;
        proxy_buffering off;

        proxy_redirect https://$subdomain.wikipedia.org/ http://$subdomain.wiki.ericdrive.ml/;
        proxy_redirect https://$subdomain.m.wikipedia.org/ http://$subdomain.m.wiki.ericdrive.ml/;
        proxy_cookie_domain $subdomain.wikipedia.org $subdomain.wiki.ericdrive.ml;
        proxy_redirect ~^https://([\w\.]+).wikipedia.org/(.*?)$ http://$1.wiki.ericdrive.ml/$2;

        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Accept-Encoding ''; 
        proxy_set_header referer "https://$proxy_host$request_uri";

        subs_filter_types text/css text/xml text/javascript application/javascript application/json;
        subs_filter .wikipedia.org .wiki.ericdrive.ml;
        subs_filter //wikipedia.org //wiki.ericdrive.ml;
        subs_filter 'https://([^.]+).wiki' 'http://$1.wiki' igr; 
        subs_filter upload.wikimedia.org up.wiki.ericdrive.ml;
    }
}

server {
    server_name ~^(?<subdomain>[^.]+)\.m\.wiki\.ericdrive\.ml$;
    listen 80;
    listen 443 ssl http2;
    resolver 8.8.8.8;

    ssl_certificate /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/fullchain.cer;
    ssl_certificate_key /usr/local/nginx/conf/ssl/wiki.ericdrive.ml/wiki.ericdrive.ml.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
    ssl_session_cache builtin:1000 shared:SSL:10m;
    # openssl dhparam -out /usr/local/nginx/conf/ssl/dhparam.pem 2048
    ssl_dhparam /usr/local/nginx/conf/ssl/dhparam.pem;
    
    if ($http_x_forwarded_proto = 'http')
    {
        return 301 $subdomain.m.wiki.ericdrive.ml$request_uri;
    }

    location / {
        proxy_pass https://$subdomain.m.wikipedia.org;
        proxy_buffering off;

        proxy_redirect https://$subdomain.m.wikipedia.org/ http://$subdomain.m.wiki.ericdrive.ml/;
        proxy_cookie_domain $subdomain.m.wikipedia.org $subdomain.m.wiki.ericdrive.ml;

        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Accept-Encoding ''; 
        proxy_set_header referer "https://$proxy_host$request_uri";

        subs_filter_types text/css text/xml text/javascript application/javascript application/json;
        subs_filter .wikipedia.org .wiki.ericdrive.ml;
        subs_filter //wikipedia.org //wiki.ericdrive.ml;
        subs_filter 'https://([^.]+).m.wiki' 'http://$1.m.wiki' igr; 
        subs_filter upload.wikimedia.org up.wiki.ericdrive.ml;
    }
}
```

## 4.重启`Nginx`

```bash
lnmp nginx restart
```