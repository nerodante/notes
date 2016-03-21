# 深入理解nginx模块开发与架构解析



### 一 .编译安装

下载地址：[http://nginx.org/en/download.html](http://nginx.org/en/download.html)

`tar -zxvf  nginx-1.6.tar.gz`

进入目录：

* ./configure
* make
* make install

_./configure —help_ 可以查看编译参数

configure 执行成功会生成objs目录，并产生如下目录：

​					 ![F24BDDD1-94A2-456C-B1E1-38919F3DAFEB](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/F24BDDD1-94A2-456C-B1E1-38919F3DAFEB.png)

**<u>ngx_modules.c 里面定义的是一个数组，指明每个模块的优先级</u>**

当需要处理多个模块时按这样的优先级处理，但HTTP过滤模块是相反的，在http框架初始化时，会在ngx_models中先将过滤模块按先后顺序在链表中添加，但每次都是添加到表头，所以对于http过滤模块越靠后的，越会优先处理。



### 二.命令行控制

| nginx -s stop           | 强制停止nginx     |
| ----------------------- | ------------- |
| nginx -V                | 查看nginx编译时的参数 |
| nginx -t                | 测试配置文件是否错误    |
| nginx  -c  …/nginx.conf | 指定配置文件        |
| nginx -s quit           | 正常处理完当前请求再停止  |
| nginx -s reload         | 重新加载配置文件      |

 ![A7106D8D-AA17-4F2A-87DE-2EC0C327A797](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/A7106D8D-AA17-4F2A-87DE-2EC0C327A797.png)



### 三.Nginx 配置

####1.运行中nginx进程间的关系

​    nginx中一个master进程不会去处理请求，而是由worker进程处理，worker进程之间共享内存，woker进程的数量与CPU的核数相等。master会去管理worker进程，因此master需要更大的权限，以root用户启动，当某个worker进程由于错误退出时，master会重新启动新的worker进程。

与apache相比较有何不同？

​    apache每个进程只能在同一时刻处理一个请求，如果需要处理更多请求，就需要启动更多进程。这样大量的进程间的切换，会有很大的资源消耗。

​    而nginx不同，同一时刻一个worker可以同时处理多个请求，数量受限于内存的大小，不同的worker进程间没有同步锁的限制。因此woker数量设置为CPU的核数，进程间的切换代价是最小。 ![F062F79C-75E5-49B9-BAC5-A1DDD0DE6414](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/F062F79C-75E5-49B9-BAC5-A1DDD0DE6414.png)

#### 2.Nginx的配置通用语法

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    debug_connection 10.211.55.2;
}


http {
	log_format  main  '$remote_addr - $request_body - $remote_user 	[$time_local] "$request" ''$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

access_log  /var/log/nginx/access.log  main;

sendfile            on;
tcp_nopush          on;
tcp_nodelay         on;
keepalive_timeout   65;
types_hash_max_size 2048;
gzip                off;
include             /etc/nginx/mime.types;
default_type        application/octet-stream;

# Load modular configuration files from the /etc/nginx/conf.d directory.
# See http://nginx.org/en/docs/ngx_core_module.html#include
# for more information.
include /etc/nginx/conf.d/*.conf;

server {
    listen       80;
    #listen       [::]:80 default_server;
    server_name  localhost;
    root         /usr/share/nginx/html;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```

上面的events，http等被称作块配置项。块配置项可以嵌套，**内层直接继承外层**

**2.1 语法格式**

行首的是配置名，其次是配置项的值，多个值之间由多个空格符分割，每行配置项结尾需要加**分号**

 ![14071123-495D-4E2C-9559-E8310C4AE8E8](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/14071123-495D-4E2C-9559-E8310C4AE8E8.png)

> 如果配置项值中包括语法符号，比如空格符，那么需要使用单引号或双引号括住配置项值，否则Nginx会报错


#### 3.Nginx配置优化

* nginx worker进程数

```nginx
worker processes 1; // 默认为1
```

一般来讲用户要配置和CPU核数相等，如果数量多于CPU内核，则会带来进程间的切换消耗。

* 绑定worker进程到CPU内核

![DE23FE67-ADD7-4373-92BC-BA9D055DDAF1](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/DE23FE67-ADD7-4373-92BC-BA9D055DDAF1.png)

```nginx
worker processes 4;
worker_cpu_affinity 1000 0100 0010 0001;
```



#### 4. Nginx基本配置

* location 

  语法：location[=|~|~*|^~|@/url/ {…..}

  配置块：server

  如果匹配则会执行location里的配置, " = " 表示做完全匹配，例如 "＝ /images/ " 只会匹配_http://www.url.com/images/_ , 不加 “ ＝” 则可配_http://www.url.com/images/.../..._

```nginx
location = / {
  // 只有当请求是 / 时，才会使用location里的配置
}
```

"~" 表示匹配url时对大小写敏感，" ~* "表示忽略大小写敏感，"^~" 表示只需其前半部分匹配。

```nginx
location ^~/images/ {
  // 以/images/ 开始都会匹配
}
```

 ![690C8E8E-F7F2-4166-B180-333F778995E5](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/690C8E8E-F7F2-4166-B180-333F778995E5.png)



#### 5. 配置nginx 反向代理 ![C4ABAAB3-1267-4DAA-9BE0-E8613CE6BC40](/Users/nero/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1254052488/QQ/Temp.db/C4ABAAB3-1267-4DAA-9BE0-E8613CE6BC40.png)

当客户端发来请求，nginx不会立即将请求转发到后端，而是先将请求缓存到本地，然后再转发。

简单配置例子：







main ：代理服务器

```nginx
http {
   upstream backend{
       server 10.211.55.12:80;
    }
  
  	server{
         listen 80;
         server_name local.test.com;
         root /Users/nero/code/www/test;
         access_log /Users/nero/code/log/access.log;
         error_log /Users/nero/code/log/error.log debug;
         location /{
            proxy_pass http://backend;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
			index index.html index.php;
            try_files $uri /index.php;
  		 }

         location ~*\.(jpg|gif)$ {
            expires 1d;
         }
         location ^~ /images/ {
            return 401;
         }
	}
}
```

A: 后端服务器（10.211.55.12）

```nginx
server {
        listen       80;
        #listen       [::]:80 default_server;
        server_name  localhost;
        root         /usr/share/nginx/html;
}
```

