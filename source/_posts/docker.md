---
title: Docker总结
tags:
  - Docker
categories: 编程
date: 2024-11-25 15:24:35
---

# Docker 架构与常用命令详解

## Docker 架构
### 仓库
- 也分为**本地仓库**和**远程仓库**, 可以通过仓库拉取镜像(pull)
- 仓库是用于存储 Docker 镜像的地方, 最常使用的 Registry 公开服务是官方的 **Docker Hub**, 网址为：[搜索镜像官网](https://hub.docker.com/ "https://hub.docker.com/")
### 镜像
- Docker 镜像就像是一个 root 文件系统，包含应用程序和运行环境的静态定义。
- 镜像是 只读的模板，用于创建容器。
### 容器
- 容器是镜像运行时的实例，类似于面向对象中的类和对象。
  - 镜像：静态定义（类）
  - 容器：运行时实例（对象）
- 容器是一个独立运行的应用实例，支持多种操作：创建、启动、停止、删除、暂停等。
- 容器内部就是一个独立系统空间
## Docker 常用命令
### 基础操作命令

| **命令**                    | **功能**           |
| ------------------------- | ---------------- |
| `systemctl status docker` | 查看 Docker 服务状态   |
| `systemctl start docker`  | 启动 Docker 服务     |
| `systemctl enable docker` | 开机自动启动 Docker 服务 |

### 镜像操作命令

| **命令**                            | **功能**                    |
| --------------------------------- | ------------------------- |
| `docker images`                   | 查看本地镜像                    |
| `docker images -q`                | 查看所有镜像的 ID                |
| `docker search 镜像名称`              | 搜索镜像                      |
| `docker pull 镜像名称`                | 拉取镜像                      |
| `docker rmi 镜像ID`                 | 删除指定的本地镜像                 |
| `docker build -t imageName:1.0.0` | 指定一个Dockerfile时, 读取指令构建镜像 |

### 容器操作命令

| **命令**                | **功能**                        |
| --------------------- | ----------------------------- |
| `docker ps`           | 查看正在运行的容器                     |
| `docker ps -a`        | 查看所有容器                        |
| `docker ps -ap`       | 查看所有容器(只带容器id参数)              |
| `docker start 容器名称`   | 启动容器                          |
| `docker inspect 容器名称` | 查看容器详细信息                      |
| `docker rm -f 容器名称`   | 删除指定容器                        |
| `docker status mysql` | 查看指定容器资源占用状况，比如cpu、内存、网络、io状态 |
| **自定义简化命令**           |                               |
| `dps`                 | 查看所有容器并简化信息（别名命令）             |

### 数据卷命令
- 容器的数据卷默认存储在主机的文件夹：   `/var/lib/docker/volumes`
- 每个数据卷在此目录下有一个独立文件夹，格式：   `/数据卷名/_data`

| **命令**                       | **功能**  |
| ---------------------------- | ------- |
| `docker volume create my-vo` | 创建一个数据卷 |
| `docker volume ls`           | 查看所有数据卷 |
| `docker inspect my-vo`       | 查看数据卷详情 |
| `docker volume rm my-vo`     | 删除数据卷   |

### 网络管理命令

| **命令**                                          | **功能**                            |
| ----------------------------------------------- | --------------------------------- |
| `docker network ls`                             | 查看所有网络                            |
| `docker network create 网络名`                     | 创建新的 Docker 网络                    |
| `docker network connect hmall mysql --alias db` | 将 mysql 容器连接到 hmall 网络，指定别名为 `db` |
| `docker network connect hmall db`               | 将 db 容器连接到 hmall 网络               |

### 容器内部命令

| **命令**                     | **功能**       |
| -------------------------- | ------------ |
| `docker exec -it 容器名 bash` | 进入指定容器的命令行环境 |

| 命令(进入容器后的命令)          | 功能说明: 与Linux系统操作一致(相当于一个小系统) |     |
| --------------------- | ---------------------------- | --- |
| `ls`                  | 查看当前目录文件                     |     |
| `cd /路径`              | 切换到指定目录                      |     |
| `pwd`                 | 查看当前路径                       |     |
| `cat 文件名`             | 查看文件内容                       |     |
| `vi 文件名` / `nano 文件名` | 编辑文件（需先安装）                   |     |
| `apt update`          | 更新 apt 包管理源（Debian/Ubuntu）   |     |
| `apt install 软件名`     | 安装常用软件                       |     |
| `top` / `ps aux`      | 查看容器中的运行进程                   |     |
| `env`                 | 查看环境变量                       |     |
| `exit`                | 退出容器命令行                      |     |
## 案例解析
### 在 Nginx 容器中挂载配置文件和静态资源目录
假设需要将 Nginx 的配置文件和静态文件挂载到本地：
1. 创建数据卷目录：  
   ```java
   mkdir -p /mydata/nginx/{conf,html}
   ```
2. 数据卷挂载方式
- `--mount` 可以有两种
	-  一种是挂载到数据卷volume上面(type=volume)
	- 另一种是直接挂载到宿主机上面(type=bind)
- `-v` 对应的是宿主机路径:容器路径
1. 运行容器时挂载数据卷：  
   ```bash
   docker run -d -p 80:80 \
       -v /mydata/nginx/conf:/etc/nginx/conf.d \
       -v /mydata/nginx/html:/usr/share/nginx/html \
       --name my-nginx nginx
   ```
2. 修改本地配置生效：  
   - 修改 `/mydata/nginx/conf` 下的配置文件后，重启容器即可生效。
### 为 Java 项目配置网络
1. 创建网络：  
   ```java
   docker network create hmall
   ```
2. 将 MySQL 容器加入网络，并设置别名：  
   ```java
   docker network connect hmall mysql --alias db
   ```
3. 将 Java 项目容器加入网络：  
   ```java
   docker network connect hmall db
   ```
4. 连接测试：   在 Java 项目中, 数据库连接地址可以直接写为：`db:3306`。
### 如何创建一个实例
```bash
# 模板
docker run -id \
	-p 主机port:容器port \
	--name=实例名 \
	-v ./宿主机目录:实例主目录 \
	-e 环境参数(例如:MYSQL_ROOT_PASSWORD=root) \
	镜像名

docker run -d \
  --name nginx \
  -p 18080:18080 \
  -p 18081:18081 \
  -v /root/nginx/html:/usr/share/nginx/html \
  -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf \
  --network hmall \
  nginx
  
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=123 \
  -v ./mysql/data:/var/lib/mysql \
  -v ./mysql/conf:/etc/mysql/conf.d \
  -v ./mysql/init:/docker-entrypoint-initdb.d \
  --network hmall    
  mysql 
```
			
- `-v ./logs:/logs`：将主机当前目录下的./logs目录挂载到容器的 /logs
- `-d`:让容器在后台运行
- `-P`:将容器内部使用的网络端口映射到我们使用的主机上
- 一个容器也可以被挂载多个数据卷
- 多个容器是可以挂载同一个数据卷
## `Docker Compose`(yaml)
- Docker Compose 是 用来定义和管理多个容器服务。  
- 它通过一个叫 `docker-compose.yml` 的配置文件，把多个 `docker run` 命令整合起来，一键启动，统一管理

| 命令                       | 介绍          |
| ------------------------ | ----------- |
| `docker-compose version` | 查看版本        |
| `docker-compose images`  | 列出所有容器使用的镜像 |
| `docker-compose kill`    | 强制停止服务的容器   |
| `docker-compose exec`    | 在容器中执行命令    |
| `docker-compose logs`    | 查看日志        |
| `docker-compose pause`   | 暂停服务        |
| `docker-compose unpause` | 恢复服务        |
| `docker-compose push`    | 推送服务镜像      |
| `docker-compose start`   | 启动当前停止的某个容器 |
| `docker-compose stop`    | 停止当前运行的某个容器 |
| `docker-compose rm`      | 删除服务停止的容器   |
| `docker-compose top`     | 查看进程        |


```yaml
version: '3.8'  # Docker Compose 文件的版本

services:
  app:  # 服务名称，可以自定义
    image: 镜像  # 使用的镜像（如：node, nginx, mysql等）
    container_name: container_name  # 指定容器名称
    ports:
      - "宿主机port:容器port"  # 映射端口：宿主机:容器（容器端口是8080，宿主机也映射8080）
    volumes:
      - ./app:/app  # 将./app宿主机当前目录下的app挂载到容器的/app目录
    environment:
      - ENV=dev  # 设置环境变量，可以根据需要调整
	build:
	  -          #构建目录
    depends_on:
      - db  # 依赖数据库服务:必须等db服务启动后才能启动
    networks:
      - backend  # 使用的网络，容器间可以通过这个网络相互通信

  db:  # 另一个服务：数据库
    image: mysql:8.0  # 使用 MySQL 8.0 镜像
    container_name: mysql_db  # 数据库容器名称
    ports:
      - "3306:3306"  # 映射数据库端口
    environment:
      - MYSQL_ROOT_PASSWORD=root  # 设置 root 用户密码
      - MYSQL_DATABASE=demo  # 设置数据库名称
      - MYSQL_USER=user  # 设置数据库用户
      - MYSQL_PASSWORD=pass  # 设置用户密码
    volumes:
      - ./mysql/data:/var/lib/mysql  # 持久化数据
    networks:
      - backend  # 使用相同的网络

networks:
  backend:
    driver: bridge  # 使用默认的 bridge 网络驱动
```
## `Dockerfile`
- Dockerfile 是一个**构建镜像**的脚本文件, 用命令docker build，创建一个 Dockerfile 文件
- 比如你有一个 Spring Boot 项目用 MySQL 和 Redis。
	- 用 Dockerfile 构建你自己的 Java 应用镜像。
	- 用 Docker Compose 启动 Java 应用 + MySQL + Redis 三个容器，它会自动组建成一个小型微服务环境。

| 对比点      | Dockerfile      | Docker Compose     |
| -------- | --------------- | ------------------ |
| 用途       | 构建镜像            | 一键式启动镜像容器          |
| 文件格式     | name.Dockerfile | docker-compose.yml |
| 是否可以独立使用 | 是               | 是                  |
| 是否构建镜像   | 是               | 可以（如果用了 `build`）   |
| 是否启动容器   | 否               | 是                  |
| 是否支持多个服务 | 否               | 是                  |
## 如何查看容器错误日志
- 适用于服务没用启动或者服务异常关闭的排错
```bash
例：实时查看docker容器名为user-uat的最后10行日志 
docker logs -f -t --tail 10 user-uat 
例：查看指定时间后的日志，只显示最后100行： docker logs -f -t --since="2018-02-08" --tail=100 user-uat 
例：查看最近30分钟的日志: docker logs --since 30m user-uat 
例：查看某时间之后的日志： docker logs -t --since="2018-02-08T13:23:37" user-uat 
例：查看某时间段日志： docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" user-uat 
例：将错误日志写入文件： docker logs -f -t --since="2018-02-18" user-uat | grep error >> logs_error.txt
```
## 注意
- 容器想要访问宿主机的端口需要通过DNS
	- Docker专门提供了一个特殊DNS名称 `host.docker.internal`，这个名称在容器内表示“宿主机地址"
	- 如果容器想要访问宿主Ollama的服务  可以通过dns映射直接来访问
		- `http://host.docker.internal:11434`
- docker的作用
	- 通过docker我们可以快速打包并一键式部署我们自己的服务在各种环境中, 不管你是在 Windows、Linux、Mac，还是在云服务器、K8s 集群