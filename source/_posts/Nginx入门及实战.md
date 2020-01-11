---
title: Nginx入门及实战
date: 2019-12-27 10:47:43
categories: Nginx
tags:
- Nginx
- 服务器
- 负载均衡
- 反向代理
- Openresty
---
一说到Nginx，想到的就是反向代理，负载均衡，动静分离，这些都只是Nginx的一些模块，也是我们经常在生产环境中使用到的主要模块，其实Nginx远远不止这些模块，他还可以做RTMP推拉流服务器，
可以做直播服务器等等，还可以自定义扩展。

## Nginx安装
* [下载](http://nginx.org/download/nginx-1.17.7.tar.gz)
* 解压：`tar -zxvf nginx-1.17.7.tar.gz`
* `./configure {--prefix} {--with-xxx-module}`:--prefix是可选的，表示指定编译安装目录，默认/usr/local/nginx。以及安装其他模块使用--with-xxx-module
* `make && make install`
* 启动和停止：`./nginx`/`./nginx -s stop`

### 安装问题解决
* 问题一
```shell
[root@localhost nginx-1.16.1]# ./configure 
checking for OS
 + Linux 3.10.0-514.el7.x86_64 x86_64
checking for C compiler ... not found

./configure: error: C compiler cc is not found
```
需要安装gcc，`yum -y install gcc`

* 问题二
```shell
./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.
```
安装`yum install pcre-devel`

* 问题三
```shell
./configure: error: the HTTP gzip module requires the zlib library.
You can either disable the module by using --without-http_gzip_module
option, or install the zlib library into the system, or build the zlib library
statically from the source with nginx by using --with-zlib=<path> option.
```
安装`yum install zlib-devel`

## 反向代理
Nginx反向代理既可以是IP+PORT也可以是域名，只需要配置proxy_pass就可以实现反向代理。配置如下：
```
server {
    listen 80;
    server_name localhost;
    location / {
        proxy_pass http://192.168.1.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
proxy_set_header：表示设置HTTP Header信息，因为在后端Servlet容器中无法获取客户端的真实IP，只能获取反向代理的IP，所以我们可以在反向代理这一层做一次转发，将客户端的真实IP转发到Servlet容器。

## 负载均衡
负载均衡是通过一定的算法分配策略将请求分摊到集群中的各个节点上，使得各个节点并行处理客户端请求，以及故障转移等功能，来达到高可用、高性能的一种手段。在Nginx中，配置负载均衡非常简单，
直接将反向代理从原来的IP+PORT或者域名代理到upstream上就可以实现负载均衡，配置如下：
```
upstream tomcat {
    server 192.168.11.161:8080 max_fails=2 fail_timeout=60s;
    server 192.168.11.159:8080;
}
server {
    listen 80;
    server_name localhost;
    location / {
        proxy_pass http://tomcat;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_500 http_503;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET,POST,DELETE';
        add_header 'Aceess-Control-Allow-Header' 'Content-Type,*';
    }
}
```
如上，配置一个upstream名为tomcat，将`proxy_pass http://tomcat;`即可实现负载均衡。
proxy_next_upstream：表示当请求的一台服务器出错时，自动切换到下一个节点继续执行客户端的请求，错误类型可以指定，例如:`error timeout http_500 http_503`，表示错误、超时、500、503
proxy_connect_timeout：用于设置Nginx于upstream server连接的超时时间，当超出该时间范围，则会报错。
proxy_send_timeout：向后端写数据的超时时间，两次写操作的时间间隔超过该值，连接会被关闭。
proxy_read_timeout：从后端服务器读取数据的超时时间，两次读取操作的时间间隔超过该值，连接会被关闭，如果后端处理逻辑较复杂导致请求响应很慢时可以将此值设置大一些。

### 负载均衡策略（算法）
Nginx有三种负载均衡算法，分别是：
* 轮询算法（默认）：将客户端请求轮流分配到集群中的节点，当有节点出现宕机时，Nginx会自动将其剔除。
* ip_hash：根据客户端请求的IP进行hash得到一个值，往后该客户端会永远访问同一个节点，除非客户端更换IP
* 权重轮询：在轮询的基础上添加权重。

## 动静分离
我们可以吧静态资源放到Nginx上，把动态资源放到后端服务器上，这样就可以减少后端服务器的请求，减小了服务器的请求压力。实现动静分离配置如下：
```
server {
    #...
    location / {
        #...
    }
    location ~ .*\.(js|css|png|svg|ico|jpg)$ {
        root static-resource;
        expires 1d;
    }
}
```
把指定后缀的请求转发到指定静态文件目录中就实现了动静分离，其中`static-resource`就是静态文件的存放路径。
动静分离过后我们可以对静态资源做一些特定的优化，例如：
* 静态资源缓存：我们可以通过`expires 1d;`来指定资源在客户端缓存的过期时间。
* 压缩：静态资源包含很多html、css、js、图片、视频等文件，这些文件本身就很大，客户端请求要返回这些资源，可能就会影响整个系统的渲染速度。所以我们可以使用压缩，将资源压缩过后再传输
到客户端，在浏览器中解压渲染。配置如下:
```
http {
    gzip on;                                                    #开启压缩功能
    gzip_min_length 5k;                                         #达到最小压缩的条件
    gzip_comp_level 3;                                          #压缩级别，该值越大压缩率越高，性能损耗越大，范围[1-9]
    gzip_types application/javascript image/jpeg image/svg+xml; #压缩的资源mine-type
    gzip_buffers 4 32k;                                         #设置压缩申请内存的大小，作用是按照指定大小的倍数申请内存，4 32k表示按照原始数据大小以32k单位的4倍申请内存
    gzip_vary on;                                               #是否传输压缩标志给客户端，避免有的浏览器不支持压缩而造成的资源浪费。
}
```

## 防盗链
有的时候我们并不希望静态资源被其他网站使用，例如：图片、视频，那么为这些资源配置防盗链是非常有必要的。防盗链的原理是判断HTTP请求头中的refer，如果refer的值不在我们允许的范围内
则进行其他操作。配置如下：
```
location ~ .*\.(js|css|png|svg|ico|jpg)$ {
    valid_referers none blocked 192.168.1.1 easyjava.xyz;
    if ($invalid_referer) {
        return 404;
    }
    root static-resource;
    expires 1d;
}
```

## 跨域访问
如果客户端和服务器端的协议、域名、端口、子域名不同，那么所有的请求都是跨域的，是浏览器对跨域资源访问的一种限制手段。我们可以在应用层面去解决跨域问题，也可以在代理层面（Nginx）
来解决跨域访问问题，配置也是非常简单：
```
location / {
    #...
    add_header 'Access-Control-Allow-Origin' '*';
    add_header 'Access-Control-Allow-Methods' 'GET,POST,DELETE';
    add_header 'Aceess-Control-Allow-Header' 'Content-Type,*';
}
```

## Nginx进程模型
Nginx采用多进程单线程的方式来运行的，在Nginx中有一个Master进程和多个Worker进程，Master进程管理着多个Worker进程，Worker进程主要处理客户端请求，Master进程Worker进程是通过共享内存的方式
和信号量进行通信的，我们可以再ngixn.conf配置文件中配置nginx的Worker进程数和每个Worker进程的最大连接数，通过这两个配置，就可以计算出当前这台Nginx节点就处理最大的连接数，
最大连接数 = Worker进程数 * 单个Worker最大连接数。配置如下：
```
worker_processes 1;         #尽量设置成CPU的核心数
events {
 worker_connections 1024;   #理论上 processes* connections
}
```

## Nginx高可用方案
Nginx作为我们系统的入口，所以Nginx本身的可用性也是我们首先考虑的问题。
//TODO

## OpenResty
OpenResty是基于Nginx内核新增了Lua支持的扩展，便于在Nginx的基础上编写Lua脚本达到用户自定义的功能实现，使之更加健壮。

### 安装
OpenResty是在Nginx上做的扩展，所以实际上也是一个Nginx，安装方式和Nginx一致。

### Nginx执行阶段和OpenResty
Nginx把一次请求划分了11个阶段，各个阶段按照顺序执行，顺序是post-read、server-rewrite、find-config、rewrite、post-rewrite、preaccess、access、post-access、try-files、content、log，下面来一一介绍一下这些
阶段都是干啥用的，并且在什么时候被调用。
* post-read：Nginx读取并解析完请求头过后立即执行该阶段。
* server-rewrite：URI和Location匹配前，修改URI，可用于重定向，该阶段执行位于Server语句块内，Location块外
* find-config：根据URI匹配Location，该阶段可能会执行多次
* rewrite：匹配到Location过后的URI重写，该阶段可能执行多次
* post-rewrite：检查上个阶段是否有URI重写，根据重写的URI跳转到合适的阶段
* preaccess：访问权限控制的前一阶段，一般用于访问控制
* access：访问权限控制阶段，判断该请求是否允许进入Nginx服务器
* post-access：权限控制的后一阶段，根据前一阶段的执行结果进行相应的处理
* try-files：为访问静态文件资源设定，如果没有配置try-files指令，该阶段会被跳过
* content：处理HTTP请求内容的阶段，该阶段产生响应，并返回到客户端
* log：日志记录阶段，记录请求访问日志
OpenResty也有如下阶段
* init_by_lua：Master进程加载conf配置文件时执行该阶段，一般用来注册全局变量和预加载Lua库
* init_worker_by_lua：各个worker进程启动时会执行该阶段，可以用来做健康检查
* ssl_certificate_by_lua：在Nginx和下游服务器进行SSL握手之前执行该阶段
* set_by_lua: 流程分之处理判断变量初始化
* rewrite_by_lua: 转发、重定向、缓存等功能(例如特定请求代理到外网)
* access_by_lua: IP准入、接口权限等情况集中处理(例如配合iptable完成简单防火墙)
* content_by_lua: 内容生成
* balancer_by_lua：实现动态负载均衡
* header_filter_by_lua: 应答HTTP过滤处理(例如添加头部信息)
* body_filter_by_lua: 应答BODY过滤处理(例如完成应答内容统一成大写)
* log_by_lua: 回话完成后本地异步完成日志记录(日志可以记录在本地，还可以同步到其他机器)
