---
layout: post
title: Nginx配置文件及模块
date: 2020-08-07
tags: WEB服务
---

# 一、Nginx是什么？
Nginx是一个基于c语言开发的高性能http服务器及反向代理服务器。由俄罗斯的程序设计师Igor Sysoev所开发，官方测试nginx能够支支撑5万并发链接，并且cpu、内存等资源消耗却非常低，运行非常稳定。

# 二、为什么要用Nginx？

### 1.理由一

- 传统的小型网站并发量小，用户使用的少，所以在低并发的情况下，用户可以直接访问tomcat服务器，然后tomcat服务器返回消息给用户。为了解决并发，可以使用负载均衡，也就是多增加几个tomcat服务器，当用户访问的时候，请求可以提交到空闲的tomcat服务器上。
- 但是这种情况下可能会出现一种问题：假设把图片上传到了tomcat1上了，当要访问这个图片的时候，tomcat1正好在工作，所以访问的请求就交给其他的tomcat操作，而tomcat之间的数据没有进行同步，所以就发生了我们要请求的图片找不到。
- 为了解决这种情况，我们专门建立一个图片服务器，用来存储图片。这样当都把图片上传的时候，不管是哪个服务器接收到图片，都把图片上传到图片服务器。而图片服务器上需要安装一个http服务，可以使用tomcat、apache、nginx。

### 2.理由二

- nginx常用做静态内容服务和代理服务器，直面外来请求转发给后面的应用服务（tomcat，django什么的），tomcat更多用来做做一个应用容器，让java web app跑在里面的东西。

### 3.理由三

- nginx作反向代理：
反向代理就是后端服务不直接对外暴露,请求首先发送到nginx,然后nginx将请求转发到后端服务器,比如tomcat等.如果后端服务只有一台服务器,nginx在这里只有一个作用就是起到了代理后端服务接收请求的作用.称之为反向代理.

### 4.理由四

- nginx作负载均衡：
在现实的应用场景中,一台后端服务器出现单点故障的概率很大或者单台机器的吞吐量有限,无法承担过多请求.这时候就需要在nginx后端配置多台服务器,利用nginx内置的规则讲请求转发到后端不同的机器上.这时候就起到了负载均衡的作用.

# 三、Nginx配置文件

### 1.主要组成部分
- main（全局设置）
main部分设置的指令将影响其它所有部分的设置；
- server（主机设置）
server部分的指令主要用于指定虚拟主机域名、IP和端口；
- upstream
upstream的指令用于设置一系列的后端服务器，设置反向代理及后端服务器的负载均衡；
- location
location部分用于匹配网页位置（比如，根目录“/”,“/images”,等等）；



### 2.它们之间的关系
server继承main，location继承server；upstream既不会继承指令也不会被继承。它有自己的特殊指令，不需要在其他地方的应用。



### 3.举例说明
下面的nginx.conf简单的实现nginx在前端做反向代理服务器的例子，处理js、png等静态文件，jsp等动态请求转发到其它服务器tomcat：

```
user www www;
worker_processes 2;
error_log logs/error.log;
#error_log logs/error.log notice;
#error_log logs/error.log info;
pid logs/nginx.pid;
events {
use epoll;
worker_connections 2048;
}
http {
include mime.types;
default_type application/octet-stream;
#log_format main '$remote_addr - $remote_user [$time_local] "$request" '
# '$status $body_bytes_sent "$http_referer" '
# '"$http_user_agent" "$http_x_forwarded_for"';
#access_log logs/access.log main;
sendfile on;
# tcp_nopush on;
keepalive_timeout 65;
# gzip压缩功能设置
gzip on;
gzip_min_length 1k;
gzip_buffers 4 16k;
gzip_http_version 1.0;
gzip_comp_level 6;
gzip_types text/html text/plain text/css text/javascript application/json application/javascript application/x-javascript application/xml;
gzip_vary on;
# http_proxy 设置
client_max_body_size 10m;
client_body_buffer_size 128k;
proxy_connect_timeout 75;
proxy_send_timeout 75;
proxy_read_timeout 75;
proxy_buffer_size 4k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k;
proxy_temp_file_write_size 64k;
proxy_temp_path /usr/local/nginx/proxy_temp 1 2;
# 设定负载均衡后台服务器列表
upstream backend {
#ip_hash;
server 192.168.10.100:8080 max_fails=2 fail_timeout=30s ;
server 192.168.10.101:8080 max_fails=2 fail_timeout=30s ;
}
# 很重要的虚拟主机配置
server {
listen 80;
server_name itoatest.example.com;
root /apps/oaapp;
charset utf-8;
access_log logs/host.access.log main;
#对 / 所有做负载均衡+反向代理
location / {
root /apps/oaapp;
index index.jsp index.html index.htm;
proxy_pass http://backend;
proxy_redirect off;
# 后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
}
#静态文件，nginx自己处理，不去backend请求tomcat
location ~* /download/ {
root /apps/oa/fs;
}
location ~ .*\.(gif|jpg|jpeg|bmp|png|ico|txt|js|css)$
{
root /apps/oaapp;
expires 7d;
}
location /nginx_status {
stub_status on;
access_log off;
allow 192.168.10.0/24;
deny all;
}
location ~ ^/(WEB-INF)/ {
deny all;
}
#error_page 404 /404.html;
# redirect server error pages to the static page /50x.html
#
error_page 500 502 503 504 /50x.html;
location = /50x.html {
root html;
}
}
## 其它虚拟主机，server 指令开始
}
```

# 四、常用指令说明

## 1.main全局配置
### woker_processes 2;
在配置文件的顶级main部分，worker角色的工作进程的个数，master进程是接收并分配请求给worker处理。这个数值简单一点可以设置为cpu的核数grep ^processor /proc/cpuinfo | wc -l，也是 auto 值，如果开启了ssl和gzip更应该设置成与逻辑CPU数量一样甚至为2倍，可以减少I/O操作。如果nginx服务器还有其它服务，可以考虑适当减少。

### worker_cpu_affinity;

也是写在main部分。在高并发情况下，通过设置cpu粘性来降低由于多CPU核切换造成的寄存器等现场重建带来的性能损耗。如worker_cpu_affinity 0001 0010 0100 1000; （四核）。

### worker_connections 2048;

写在events部分。每一个worker进程能并发处理（发起）的最大连接数（包含与客户端或后端被代理服务器间等所有连接数）。nginx作为反向代理服务器，计算公式 最大连接数 = worker_processes * 

### worker_rlimit_nofile 10240;

写在main部分。默认是没有设置，可以限制为操作系统最大的限制65535。

### use epoll;

写在events部分。在Linux操作系统下，nginx默认使用epoll事件模型，得益于此，nginx在Linux操作系统下效率相当高。同时Nginx在OpenBSD或FreeBSD操作系统上采用类似于epoll的高效事件模型kqueue。在操作系统不支持这些高效模型时才使用select。

## 2.http服务器

### sendfile on;

开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，减少用户空间到内核空间的上下文切换。对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载；

### keepalive_timeout 65; 

长连接超时时间，单位是秒，这个参数很敏感，涉及浏览器的种类、后端服务器的超时设置、操作系统的设置，可以另外起一片文章了。长连接请求大量小文件的时候，可以减少重建连接的开销，但假如有大文件上传，65s内没上传完成会导致失败。如果设置时间过长，用户又多，长时间保持连接会占用大量资源；

### send_timeout ;

用于指定响应客户端的超时时间。这个超时仅限于两个连接活动之间的时间，如果超过这个时间，客户端没有任何活动，Nginx将会关闭连接；

### client_max_body_size 10m;

允许客户端请求的最大单文件字节数。如果有上传较大文件，请设置它的限制值；

### client_body_buffer_size 128k;

缓冲区代理缓冲用户端请求的最大字节数；

## 3.模块http_proxy
### proxy_read_timeout 60;

连接成功后，与后端服务器两个成功的响应操作之间超时时间(代理接收超时)

### proxy_buffer_size 4k;

设置代理服务器（nginx）从后端realserver读取并保存用户头信息的缓冲区大小，默认与proxy_buffers大小相同，其实可以将这个指令值设的小一点
### proxy_buffers 4 32k;

proxy_buffers缓冲区，nginx针对单个连接缓存来自后端realserver的响应，网页平均在32k以下的话，这样设置
### proxy_busy_buffers_size 64k;

高负荷下缓冲大小（proxy_buffers*2）
###proxy_max_temp_file_size;

当 proxy_buffers 放不下后端服务器的响应内容时，会将一部分保存到硬盘的临时文件中，这个值用来设置最大临时文件大小，默认1024M，它与 proxy_cache 没有关系。大于这个值，将从upstream服务器传回。设置为0禁用。
### proxy_temp_file_write_size 64k;

当缓存被代理的服务器响应到临时文件时，这个选项限制每次写临时文件的大小。proxy_temp_path（可以在编译的时候）指定写到哪那个目录。

## 4.模块http_gzip：

### gzip on :

开启gzip压缩输出，减少网络传输。

### gzip_min_length 1k ：

设置允许压缩的页面最小字节数，页面字节数从header头得content-length中进行获取。默认值是20。建议设置成大于1k的字节数，小于1k可能会越压越大。

### gzip_buffers 4 16k ：

设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。4 16k代表以16k为单位，安装原始数据大小以16k为单位的4倍申请内存。

### gzip_http_version 1.0 ：

用于识别 http 协议的版本，早期的浏览器不支持 Gzip 压缩，用户就会看到乱码，所以为了支持前期版本加上了这个选项，如果你用了 Nginx 的反向代理并期望也启用 Gzip 压缩的话，由于末端通信是 http/1.0，故请设置为 1.0。
### gzip_comp_level 6 ：

gzip压缩比，1压缩比最小处理速度最快，9压缩比最大但处理速度最慢(传输快但比较消耗cpu)
### gzip_types ：

匹配mime类型进行压缩，无论是否指定,”text/html”类型总是会被压缩的。
### gzip_proxied any ：

Nginx作为反向代理的时候启用，决定开启或者关闭后端服务器返回的结果是否压缩，匹配的前提是后端服务器必须要返回包含”Via”的 header头。
### gzip_vary on ：

和http头有关系，会在响应头加个 Vary: Accept-Encoding ，可以让前端的缓存服务器缓存经过gzip压缩的页面，例如，用Squid缓存经过Nginx压缩的数据。

## 5.server虚拟主机
### listen;

监听端口，默认80，小于1024的要以root启动。可以为listen *:80、listen 127.0.0.1:80等形式。
### server_name;

服务器名，如localhost、www.example.com，可以通过正则匹配。


## 6.模块http_stream

### host:port options; 

这个模块通过一个简单的调度算法来实现客户端IP到后端服务器的负载均衡，upstream后接负载均衡器的名字，后端realserver以 host:port options; 方式组织在 {} 中。如果后端被代理的只有一台，也可以直接写在 proxy_pass 。

## 7.location
### root /var/www/html;

定义服务器的默认网站根目录位置。如果locationURL匹配的是子目录或文件，root没什么作用，一般放在server指令里面或/下。
### index index.jsp index.html index.htm;

定义路径下默认访问的文件名，一般跟着root放
### proxy_pass http:/backend;

请求转向backend定义的服务器列表，即反向代理，对应upstream负载均衡器。也可以proxy_pass http://ip:port。

## 8.访问控制 allow/deny

> Nginx 的访问控制模块默认就会安装，而且写法也非常简单，可以分别有多个allow,deny，允许或禁止某个ip或ip段访问，依次满足任何一个规则就停止往下匹配。如：

```
location /nginx-status {
stub_status on;
access_log off;
# auth_basic "NginxStatus";
# auth_basic_user_file /usr/local/nginx-1.6/htpasswd;
allow 192.168.10.100;
allow 172.29.73.0/24;
deny all;
}
```

> 也常用 httpd-devel 工具的 htpasswd 来为访问的路径设置登录密码：

```
# htpasswd -c htpasswd admin
New passwd:
Re-type new password:
Adding password for user admin
# htpasswd htpasswd admin //修改admin密码
# htpasswd htpasswd sean //多添加一个认证用户
```
> 这样就生成了默认使用CRYPT加密的密码文件。打开上面nginx-status的两行注释，重启nginx生效。

## 9.列出目录 autoindex

Nginx默认是不允许列出整个目录的。如需此功能，打开nginx.conf文件，在location，server 或 http段中加入autoindex on;，另外两个参数最好也加上去:

### autoindex_exact_size off; 

默认为on，显示出文件的确切大小，单位是bytes。改为off后，显示出文件的大概大小，单位是kB或者MB或者GB

### autoindex_localtime on;

默认为off，显示的文件时间为GMT时间。改为on后，显示的文件时间为文件的服务器时间

```
location /images {
root /var/www/nginx-default/images;
autoindex on;
autoindex_exact_size off;
autoindex_localtime on;
}
```