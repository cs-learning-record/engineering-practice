![image](./img/timg.jpg)
<br>

## 前言

本人平时学习及收集内容，欢迎参入一起讨论。

## 关于作者

一个工作八年的草根程序员。

## 内容

- [Ngnix介绍](#一Ngnix介绍)
- [安装](#二安装)
- [Nginx基本配置与参数说明](#三Nginx基本配置与参数说明)
- [Nginx负载均衡](#四Nginx负载均衡)
- [Nginx反向代理](#五Nginx反向代理)
- [配置Gzip](#六配置Gzip)

### 一、Ngnix介绍

Nginx是一款轻量级的网页服务器、反向代理器以及电子邮件代理服务器。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、示例配置文件和低系统资源的消耗而闻名。类似的web服务器有Apache、IIS。

### 二、安装

### 2.1 安装依赖包

```
//一键安装上面四个依赖
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel
```

### 2.2 下载并解压安装包

```

//创建一个文件夹
cd /usr/local
mkdir nginx
cd nginx
//下载tar包
wget http://nginx.org/download/nginx-1.13.7.tar.gz
tar -xvf nginx-1.13.7.tar.gz
```

### 2.3 安装nginx

```
//进入nginx目录
cd /usr/local/nginx
//执行命令
./configure
//执行make命令
make
//执行make install命令
make install
```

#### 2.4 启动

```
/usr/local/nginx/sbin/nginx -s reload
```

查看nginx进程是否启动：

`ps -ef | grep nginx`

#### 2.5 关闭防火墙

- centOS6及以前版本使用命令： `systemctl stop iptables.service`
- centOS7关闭防火墙命令： `systemctl stop firewalld.service`

#### 2.6 启动，关闭，重启，命令

进入安装目录中，命令：`cd /usr/local/nginx/sbin`，`./nginx` 启动，`./nginx -s stop` 关闭，`./nginx -s reload`重启。

### 三、Nginx基本配置与参数说明

nginx配置文件主要分为六个区域：main(全局设置)、events(nginx工作模式)、http(http设置)、server(主机设置)、location(URL匹配)、upstream(负载均衡器设置)。
  
#### 3.1 main模块

```
user nobody nobody;
worker_processes 2;
error_log /usr/local/var/log/nginx/error.log  notice;
pid        /usr/local/var/run/nginx/nginx.pid;
worker_rlimit_nofile 1024;
   
```

- `user`：来指定Nginx Worker进程运行用户以及用户组，默认同nobody账号运行。
- `worker_processes`：来指定了Nginx要开启的子进程数。每个Nginx进程平均耗费10M~12M内存。根据经验，一般指定1个进程就足够了，如果是多核CPU，建议指定和CPU的数量一样的进程数即可。我这里写2，那么就会开启2个子进程，总共3个进程。
- `error_log`：用来定义全局错误日志文件。日志输出级别有debug、info、notice、warn、error、crit可供选择，其中，debug输出日志最为最详细，而crit输出日志最少。
- `pid`：用来指定进程id的存储文件位置。
- `worker_rlimit_nofile`：用于指定一个nginx进程可以打开的最多文件描述符数目，这里是65535，需要使用命令“ulimit -n 65535”来设置。

#### 3.2 events 模块

events模块来用指定nginx的工作模式和工作模式及连接数上限，一般是这样：
   
```
events {
    use kqueue; #mac平台
    worker_connections  1024;
}

```

- `use`：用来指定Nginx的工作模式。Nginx支持的工作模式有select、poll、kqueue、epoll、rtsig和/dev/poll。其中select和poll都是标准的工作模式，kqueue和epoll是高效的工作模式，不同的是epoll用在Linux平台上，而kqueue用在BSD系统中，因为Mac基于BSD,所以Mac也得用这个模式，对于Linux系统，epoll工作模式是首选。
- `worker_connections`：用于定义`Nginx`每个进程的最大连接数，即接收前端的最大请求数，默认是`1024`。最大客户端连接数由`worker_processes`和`worker_connections`决定，即`Max_clients=worker_processes*worker_connections`，在作为反向代理时，`Max_clients`变为：`Max_clients = worker_processes * worker_connections/4`。进程的最大连接数受Linux系统进程的最大打开文件数限制，在执行操作系统命令“ulimit -n 65536”后`worker_connections`的设置才能生效。

#### 3.3 http 模块

http模块可以说是最核心的模块了，它负责HTTP服务器相关属性的配置，它里面的server和upstream子模块，至关重要。

```
http{
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" 
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /usr/local/var/log/nginx/access.log  main;
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  10;
    #gzip  on;
    upstream myproject {
        .....
    }
    server {
        ....
    }
}
```

下面详细介绍下这段代码中每个配置选项的含义。 

- `include`：来用设定文件的`mime`类型,类型在配置文件目录下的`mime.type`文件定义，来告诉`nginx`来识别文件类型。
- `default_type`：设定了默认的类型为二进制流，也就是当文件类型未定义时使用这种方式，例如在没有配置`asp`的`locate`环境时，Nginx是不予解析的，此时，用浏览器访问asp文件就会出现下载了。
- `log_format`：用于设置日志的格式，和记录哪些参数，这里设置为`main`，刚好用于`access_log`来纪录这种类型。
- `access_log`：用来纪录每次的访问日志的文件地址，后面的`main`是日志的格式样式，对应于`log_format`的`main`。
- `sendfile`：参数用于开启高效文件传输模式。将`tcp_nopush`和`tcp_nodelay`两个指令设置为`on`用于防止网络阻塞。
- `keepalive_timeout`：设置客户端连接保持活动的超时时间。在超过这个时间之后，服务器会关闭该连接。

#### 3.4 server 模块

`sever`模块是`http`的子模块，它用来定一个虚拟主机，我们先讲最基本的配置，这些在后面再讲。

```
server {
        listen       8080;
        server_name  localhost 192.168.12.10 www.yangyi.com;
        # 全局定义，如果都是这一个目录，这样定义最简单。
        root   /Users/yangyi/www;
        index  index.php index.html index.htm; 
        charset utf-8;
        access_log  usr/local/var/log/host.access.log  main;
        aerror_log  usr/local/var/log/host.error.log  error;
        ....
}
```

- `server`：标志定义虚拟主机开始。
- `listen`：用于指定虚拟主机的服务端口。
- `server_name`：用来指定IP地址或者域名，多个域名之间用空格分开。
- `root`：表示在这整个`server`虚拟主机内，全部的`root web`根目录。注意要和`locate {}`下面定义的区分开来。
- `index`：全局定义访问的默认首页地址。注意要和`locate {}`下面定义的区分开来。
- `charset`：用于设置网页的默认编码格式。
- `access_log`：用来指定此虚拟主机的访问日志存放路径，最后的`main`用于指定访问日志的输出格式。

#### 3.5 location 模块

`location`模块是nginx中用的最多的，也是最重要的模块了，什么负载均衡啊、反向代理啊、虚拟域名啊都与它相关;location 根据它字面意思就知道是来定位的，定位URL，解析URL，所以，它也提供了强大的正则匹配功能，也支持条件判断匹配，用户可以通过location指令实现Nginx对动、静态网页进行过滤处理。像我们的php环境搭建就是用到了它。

```
location / {
    root   /Users/yangyi/www;
    index  index.php index.html index.htm;
}

```

- `location`： `/`表示匹配访问根目录。
- `root`：指令用于指定访问根目录时，虚拟主机的web目录，这个目录可以是相对路径（相对路径是相对于nginx的安装目录）。也可以是绝对路径。
- `index`：用于设定我们只输入域名后访问的默认首页地址，有个先后顺序：index.php index.html index.htm，如果没有开启目录浏览权限，又找不到这些默认首页，就会报403错误。
- `location`：还有一种方式就是正则匹配，开启正则匹配这样：location ~。后面加个~。

下面这个例子是运用正则匹配来链接php。我们之前搭建环境也是这样做：

```
location ~ \.php$ {
    root           /Users/yangyi/www;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    include        fastcgi.conf;
}
```

`\.php$`熟悉正则的我们直到，这是匹配.php结尾的URL，用来解析php文件。里面的root也是一样，用来表示虚拟主机的根目录。
fast_pass链接的是php-fpm 的地址，之前我们也搭建过。

### 四、Nginx负载均衡

- 轮询（默认），请求过来后，Nginx 随机分配流量到任一服务器

```
upstream backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}
```

- `weight=number`设置服务器的权重，默认为1，权重大的会被优先分配

```
upstream backend {
    server 127.0.0.1:3000 weight=2;
    server 127.0.0.1:3001 weight=1;
}
```

- `backup` 标记为备份服务器。当主服务器不可用时，将传递与备份服务器的连接。

```
upstream backend {
    server 127.0.0.1:3000 backup;
    server 127.0.0.1:3001;
}
```

- `ip_hash` 保持会话，保证同一客户端始终访问一台服务器。

```
upstream backend {
    ip_hash;  
    server 127.0.0.1:3000 backup;
    server 127.0.0.1:3001;
}
```

- `least_conn` 优先分配最少连接数的服务器，避免服务器超载请求过多。

```
upstream backend {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}
```

### 五、Nginx反向代理

- 什么是反向代理
- 反射代理的作用
- 反射代理配置

#### 5.1 什么是反向代理
  
反向代理是指以代理服务器来接受internet上的连接请求，然后奖请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就是表现为一个服务器.
  
代理情况：`client —(send request)—> clinet proxy –(send request)—> server`

#### 5.2 反射代理的作用

保护网站安全：任何来自internet的请求都必须先经过代理器；
通过配置缓存功能加速web请求：可以缓存真实web服务器上的某些静态资源，减轻真实web服务器的负载压力；

实现负载均衡：充当负载均衡服务器均衡分发请求，平衡集群中各个服务器的负载压力。

#### 5.3 反向代理配置

```
location /api {   
    # 请求host传给后端
    proxy_set_header Host $http_host;
    # 请求ip 传给后端
    proxy_set_header X-Real-IP $remote_addr;
    # 请求协议传给后端
    proxy_set_header X-Scheme $scheme;
    # 路径重写
    rewrite  /api/(.*)  /$1  break;
    # 代理服务器
    proxy_pass http://localhost:9000;
}

```

- 拦截路径`/api`, 可以通过正则匹配。
- `proxy_set_header`允许重新定义或添加字段传递给代理服务器的请求头。
- `$http_host`、`$remote_addr`、`$scheme` 为Nginx内置变量。
- `rewrite`根据`rewrite`后的请求`URI`，将路径重写，如：接口路径为`/user`, 我们可以请求`/api/user`。（为什么需要重写uri？因为在使用Nginx做反向代理的时候，需要匹配到跨域的接口再做转发，为了方便匹配，会人为的在原接口中添加一段路径（或标示， 如例子中的api），因此需要在匹配之后、转发之前把添加的那段去掉，因此需要rewrite。）
- `break`继续本次请求后面的处理 ,停止匹配下面的location。需要注意的是与之类似的last执行过程则是停止当前这个请求，并根据rewrite匹配的规则重新发起一个请求，从上到下依次匹配location后面的规则。
- `proxy_pass`代理服务器。

### 六、配置Gzip

```
server {
    # 开启gzip 压缩
    gzip on;
    # 设置gzip所需的http协议最低版本 （HTTP/1.1, HTTP/1.0）
    gzip_http_version 1.1;
    # 设置压缩级别，压缩级别越高压缩时间越长  （1-9）
    gzip_comp_level 4;
    # 设置压缩的最小字节数， 页面Content-Length获取
    gzip_min_length 1000;
    # 设置压缩文件的类型  （text/html)
    gzip_types text/plain application/javascript text/css;
}
```

### 参考资料

- [nginx基本配置与参数说明](http://www.nginx.cn/76.html)
- [Nginx标准HTTP模块](http://manual.51yip.com/nginx/StandardHTTPModules/index.html)
- [nginx从入门到实践](https://juejin.im/post/5a2600bdf265da432b4aaaba)
- [前端想要了解的Nginx](https://juejin.im/post/5cae9de95188251ae2324ec3)

## 联系作者

<div align="center">
    <p>
        平凡世界，贵在坚持。
    </p>
    <img src="./img/contact.png" />
</div>