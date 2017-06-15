---
title: 部署HTTP2踩坑记
date: 2017-06-15 23:01:16
tags:
- HTTP2
categories: 
- 浏览器
---

## 实现HTTP/2很简单

参考Nginx实现HTTP/2白皮书

https://www.nginx.com/wp-content/uploads/2015/09/NGINX_HTTP2_White_Paper_v4.pdf

根据白皮书的描述，我们只需要在 `Nginx` 的配置文件中增加如下几行配置即可实现HTTP/2的部署。

```text
server {
    listen 443 ssl http2 default_server;
    ssl_certificate server.crt;
    ssl_certificate_key server.key;
    …
}
```

不管你信不信，反正我我就天真的信了！！！然后在白皮书的指导下开始了一个下午的踩坑之旅。

<!-- more -->

## 证书

在部署了HTTPS的基础上，证书上的坑已经踩过去了，问题基本解决。可参考[自签证书](http://blog.yancoder.com/2017/06/06/%E8%87%AA%E7%AD%BE%E8%AF%81%E4%B9%A6/);

## Nginx

## 前奏

白皮书中明确指出，需要 `Nginx 1.9.6` 及其后的版本。

```text
NGINX Plus users simply need to use the HTTP/2-enabled package nginx-plus-http2. For
NGINX open source, use Version 1.9.6 or later. 
```

在我的服务器上运行 `nginx -v` 

```text
nginx version: nginx/1.6.3
```

=.= 得嘞，先升级吧

## Nginx升级

1、在 [nginx: download](http://nginx.org/en/download.html) 上下载稳定版 `Nginx-1.12.0`。

2、上传到服务器上，解压。

```text
$ tar -zxvf nginx-1.12.0.tar.gz
```

3、编译

`nginx`源码的编译需要使用`configure`脚本自动生成`Makefile`脚本，`configure`脚本支持许多选项，来控制安装的各种属性。

```text
$ ./configure --prefix=/etc/nginx-1.12.0
# --prefix=/etc/nginx-1.12.0 指明软件安装的路径
```

结果GG

![](/images/20150804205132556.jpg)

`nginx` 的一些模块需要依赖其他第三方的库，通常有 `pcre` 库（支持rewrite模块）、`zlib` 库（支持gzip模块）和 `openssl` 库（支持ssl模块）等。
这种情况，说明缺少 `pcre` 库。

```text
$ sudo apt-get update
$ sudo apt-get install libpcre3 libpcre3-dev
```

再次执行 `configure` 即可。

4、安装

```text
$ make
$ make install
```

命令完成后，将当前目录定位到 `/etc/nginx-1.12.0` 下，就可以查看 `nginx` 的全部资源了，主要有4个文件夹组成：conf、html、logs、sbin。

- conf目录中存放nginx的所有配置文件
- html目录中存放了nginx服务器在运行过程中调用的一些html网页文件
- logs用来存放nginx服务器日志的
- sbin目录存放的是nginx的主程序

## 编译模块

终于装好支持HTTP/2的 `Nginx` 版本了，按照白皮书修改配置文件，检查 `nginx` ：

```text
$ nginx -t
nginx: [emerg] the "ssl" parameter requires ngx_http_ssl_module in /usr/local/nginx/conf/nginx.conf:99
```

好吧，重新编译，加上 `ngx_http_ssl_module` 模块：

```text
$ ./configure --prefix=/etc/nginx-1.12.0 --with-http_ssl_module
$ make
$ make install
```

```text
$ nginx -t
nginx: [emerg] the "http2" parameter requires ngx_http_v2_module in /etc/nginx-1.12.0/conf/nginx.conf:118
```

？？？ 一脸问号，重新编译！加上 `ngx_http_v2_module` 模块：

```text
$ ./configure --prefix=/etc/nginx-1.12.0 --with-http_ssl_module --with-http_v2_module
$ make
$ make install
```

```text
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

哇，终于好了！赶紧打开浏览器试一下！

？？？又是一脸问号，为什么还是`HTTP1.1`，说好的`h2`呢？

## openssl升级

继续面向Google编程！

`nginx -V` 查看编译参数也带有 `--with-http_v2_module`，应该没什么问题才对。
Google 了一把发现是 OpenSSL 版本的问题，我使用的是 `ubuntu 14.04` ，这个系统上的 OpenSSL 版本是 1.0.1 ，所以我的 Nginx 是在 OpenSSL 1.0.1 上编译而成的，而 OpenSSL 的这个版本不支持 ALPN，所以无法开启 HTTP/2。

```text
$ openssl version
OpenSSL 1.0.1f 6 Jan 2014
```
> Note that accepting HTTP/2 connections over TLS requires the “Application-Layer Protocol Negotiation” (ALPN) TLS extension support, which is available only since OpenSSL version 1.0.2. Using the “Next Protocol Negotiation” (NPN) TLS extension for this purpose (available since OpenSSL version 1.0.1) is not guaranteed.

解决方法有两种：

1、升级本地 OpenSSL ，然后重新编译。

留个坑吧。升级OpenSSL很麻烦的样子。

2、添加一个源，用 `ape-get` 安装 Nginx。

这么懒的我果断选择这种方法。

```text
$ sudo add-apt-repository ppa:ondrej/nginx
$ sudo apt-get update
$ sudo apt-get install -f nginx nginx-common nginx-full

$ nginx -V
nginx version: nginx/1.12.0
built with OpenSSL 1.0.2k  26 Jan 2017 (running with OpenSSL 1.0.2l  25 May 2017)
TLS SNI support enabled
```

再次修改配置文件：
```text
server {
    listen 443 ssl http2;
    server_name tencentserver.yancoder.com;

    root /usr/share/nginx/html;
    index index.html index.htm;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    ssl_session_timeout 5m;

    ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

可爱的 `h2` 终于出来了，泪流满面！