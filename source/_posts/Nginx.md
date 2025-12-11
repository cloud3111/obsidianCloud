---
title: Nginx配置解析
date: 2024-10-23 11:29:59
tags:
  - Nginx
  - chuangTu
categories: 编程
---

# Nginx

## `Nginx`配置页解析
- **proxy_pass 是隐式的, 从前端看不到后端地址, 这是一个误区, nginx保护隐私隐藏代理后的ip和端口**
- **Nginx可以通过添加请求头参数CORS解决跨域问题**
```nginx
#配置允许运行Nginx工作进程的用户和用户组
user www;

#配置运行Nginx进程生成的worker进程数
worker_processes 2;

#配置Nginx服务器运行对错误日志存放的路径
error_log logs/error.log;

#配置Nginx服务器允许时记录Nginx的master进程的PID文件路径和名称
pid logs/nginx.pid;

#配置Nginx服务是否以守护进程方法启动
#daemon on;

updtream cluster{			# 选择性代理后端服务器的地址
	# nginx负载均衡策略: 默认轮询, 还有ip_hash(同一个ip每次都会定位到同一个后端)  
    server 192.168.71.100:8080 weight = 1;		
    server 192.168.71.100:8081 weight = 2;
    server 192.168.71.100:8082 weight = 1;
}

events {
    #设置Nginx网络连接序列化
    accept_mutex on;

    #设置Nginx的worker进程是否可以同时接收多个请求
    multi_accept on;

    #设置Nginx的worker进程最大的连接数
    worker_connections 1024;

    #设置Nginx使用的事件驱动模型
    use epoll;
}
```

```nginx
http {
    include			mime.types;   	# 根据文件后缀判断文件类型
    	
	# 每一个server块就是一个虚拟主机
	# 项目演示
	server{	
		# Nginx静态资源服务器
		listen       1000;              
		server_name  192.168.71.100;

		location / {					# 代理本地 /根目录下的资源
			root   html;				# 静态根目录
			index   index.html;			# 静态首页
		}

	   # 注意 nginx保护隐私隐藏代理后的ip和端口
	   # 将代理以/api开头的请求,然后重写清除/api前缀
		location /api/ {
			rewrite ^/api/(.*)$ /$1 break;
			proxy_pass http://cluster;
			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			add_header Access-Control-Allow-Origin *;
			add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
			add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept';
		}

		# error_page:设置网站的错误页面 可以指定具体跳转的地址
		error_page 404 /404.html;
		error_page 500 502 503 504 /50x.html

		# 错误页面精确导向
		location = /404.html {
			root html;
			index 404.html;
		}  

		# 指定重定向地址
		location =/50x.html {
			root html;
			index 50x.html;
		}
			}

	# 利用nginx代理本地床图
    server {
        listen       3111;
        server_name  www.iloveyou.*  www.other.com;

        location / {
            root html;
            index index.html;
        }

        
        location /img/ {
        	alias html/img/;
            autoindex on;  # 启用目录列表查看 
            autoindex_exact_size on; # 是否在目录列表展示文件的详细大小
            autoindex_format html; # 设置目录列表的格式
            autoindex_localtime on;   # 是否在目录列表上显示时间
    }

    # 演示rewrite重定向 
    server {
        listen 3112;
        server_name localhost;
        rewrite ^/ http://www.baidu.com permanent;

        location rewrite {
            rewrite ^/rewrite/url\w*$ https://www.baidu.com;
            rewrite ^/rewrite/(test)\w*$ /$1;
            rewrite ^/rewrite/(demo)\w*$ /$1;
        }
        location /test{
            default_type text/plain;
            return 200 test_success;
        }
    }
}
```

![17312463677901731246367666.png|700x558](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17312463677901731246367666.png)
## `Nginx`原理

| **原理**：有一个 `index.html` 静态资源部署在 `192.168.71.100:80`。当用户通过浏览器访问 `192.168.71.100:80`（即 `www.iloveyou.com`）时，服务器将静态页面返回给用户。用户的浏览器在渲染页面时，如果发现需求，会发送 AJAX 或 Fetch 请求（由前端定义）到 **Nginx**，然后 **Nginx 反向代理**请求到后端的 Tomcat 服务器（`192.168.71.200:8080/get`） |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ![17305165582881730516558235.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17305165582881730516558235.png)                                                                                                                  |

## `Nginx`路径重写
#### 1.路径重写的两种方式
- **`rewrite` 指令**
- `rewrite` 用于修改客户端请求的 URI（路径），在请求进入 Nginx 处理流程时生效。
- **`proxy_pass` 指令**
- `proxy_pass` 指令用于转发请求到后端服务器，并可根据路径配置进行自动拼接或替换。
#### 2.rewrite详解
- 语法
  **`regex`**：**正则表达式**，用于匹配需要重写的路径。k;  
  **`replacement`**：目标路径，替换 `regex` 匹配到的路径部分。
  **`flag`**：
  - `last`：重写完成后，停止当前 location 配置，重新进入新的处理流程。
  - `break`：停止重写，但继续在当前 location 中处理请求。
  - `redirect`：返回临时重定向（302）。
  - `permanent`：返回永久重定向（301）。
- 例子:   
  - 重写为新路径: rewrite     ^/old-path/(.*)$      /new-path/$1 brea;  即请求 `/old-path/test` -> `/new-path/test`
  - 重定向到完整 URL: rewrite    ^/google$      <https://www.google.com>     permanent;   即请求 `/google` -> 301 重定向到 `https://www.google.com`
#### 3.proxy_pass详解
**规则 1**：如果 `proxy_pass` 以 `/` 结尾：
- Nginx 会将匹配的 `location` 中的 URI 前缀移除，并拼接到 `proxy_pass` 的 URL 后。
**示例**：
```nginx
nginx复制代码location /api/ {
    proxy_pass http://backend/service/;
}
```
- 请求 `/api/test` 转发为 `http://backend/service/test`。
**规则 2**：如果 `proxy_pass` 不以 `/` 结尾：
- Nginx 会保留 `location` 中的 URI 前缀，并直接拼接到 `proxy_pass` 的 URL 后。
**示例**：
```nginx
nginx复制代码location /api/ {
    proxy_pass http://backend/service;
}
```
- 请求 `/api/test` 转发为 `http://backend/service/api/test`。
**规则3:移除路径前缀并转发**
```nginx
nginx复制代码location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://backend/;
}
```
- 请求 `/api/login` -> 先重写为 `/login`，然后转发 `http://backend/login`。
#### 4.方式区别
![17318103314111731810330458.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17318103314111731810330458.png)
## `Tips`
#### 为什么要动静分离?
- 前面我们介绍过**Nginx在处理静态资源的时候，效率是非常高的**，而且**Nginx的并发访问量也是名列前茅**，而Tomcat则相对比较弱一些，所以把静态资源交个Nginx后，可以减轻Tomcat服务器的访问压力并提高静态资源的访问速度。
#### 如何实现动静分离?
- 实现动静分离的方式很多，比如静态资源可以部署到CDN、Nginx等服务器上，动态资源可以部署到Tomcat,weblogic或者websphere上。
```nginx
#动态资源
location /demo {
    proxy_pass http://webservice;
}
#静态资源
location ~/.*\.(png|jpg|gif|js){
    root html/web;
    gzip on;
}

```
#### 正向代理和反向代理的区别?
- 正向代理是**客户端的代理**, 是由**客户端架设代理软件**之类的, 比如说**vpn**, **隐藏了客户端地址**
- 反向代理是**服务端的代理**, 是由**服务端设置一个反向代理服务器**, 比如**nginx**, 可以**隐藏后端真实的ip地址**
- 正向代理和反向代理的作用和目的不同。正向代理主要是用来**解决访问限制问题**；而反向代理则是提供**负载均衡**、安全防护等作用；但是二者**均能提高访问速度**
#### 请求出现异常的可能情况
- 跨域option预检失败 
- 跨域请求
- 未处理/api后缀  
- 检查请求头参数 
- 请求参数不正确
