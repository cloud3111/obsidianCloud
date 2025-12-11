---
banner: "[[pixel-banner-image.png]]"
title: Linux总结
tags:
  - Linux
categories: 编程
date: 2024-11-25T16:07:00
---

# Linux Shell 命令整理
## Linux 目录结构
1. /：根目录，所有文件和目录的起点。
2. /bin：存放基础命令的可执行文件，如 ls、cp。
3. /sbin：系统管理命令所在目录，普通用户通常没有权限。
4. /etc：系统配置文件目录，如网络和服务配置。 (source /etc/profile: 刷新系统配置)
5. /home：用户主目录，每个用户有自己的文件存储位置。
6. /root：超级用户（root）的主目录。
7. /var：存放日志、缓存、临时文件等可变数据。
8. /usr：存放用户级应用程序和共享文件。
9. /lib：系统基本库文件，支持 /bin 和 /sbin 命令。
10. /opt：用于安装第三方或可选软件包。
11. /dev：设备文件目录，所有硬件设备在此表示为文件。
12. /proc：虚拟文件系统，存放内核和进程信息。
13. /tmp：临时文件目录，系统重启后可能会清空。
14. /boot：存放启动相关文件，如内核和引导加载程序。
15. /mnt 和 /media：用于临时挂载外部设备。
![17582598048201758259804688.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17582598048201758259804688.png)

## 用户管理

### 创建用户

```bash
useradd -m -s /bin/bash -u 501 King
```

- `-m`：创建用户的家目录 `/home/King`
- `-s /bin/bash`：指定默认 Shell 解释器为 Bash
- `-u 501`：指定用户 ID（UID）为 501

### 设置用户密码

```bash
passwd King
```

- 输入两次新密码即可完成用户密码设置。

### 修改用户名

```bash
usermod -l Queen King
```

- `-l`：更改用户名，将 `King` 修改为 `Queen`

### 查看用户信息(配置文件)

```bash
cat /etc/passwd
```

- 每个用户占一行，格式如下：

  ```java
  用户名:密码(加密后):用户ID:组ID:描述:家目录:默认Shell
  ```

### 查看用户所属组
- 每个用户都有一个用户id和用户名; 用户组也有名和id
- 一个用户可用属于一个或多个用户组
```bash
groups Queen
```

- 显示 `Queen` 用户所属的组。

### 创建用户组

```bash
groupadd -g 501 QueenGroup
```

- `-g 501`：指定组 ID（GID）为 501。

### 修改用户所属组

```bash
groupmod -g QueenGroup Queen
```

- 修改 `Queen` 用户的主组为 `QueenGroup`。

### 查看组信息

```bash
cat -n /etc/group
```

- `-n`：显示行号。
- `/etc/group` 文件包含所有用户组的信息。

### 过滤查询某个用户的组信息

```bash
grep 'King' /etc/group
```

- 仅显示 `King` 用户的组信息。

### 查看当前登录用户

```bash
who -H
```

- 显示当前登录的用户列表。
## 进程管理
- PCB: process control blocker, 一个进程有一个PCB单元, 作用是进程描述, 进程调度(5种)… 
### 查看当前时间

```bash
date "+%Y-%m-%d %H:%M:%S"
```

- 显示当前时间，格式为 `年-月-日 时:分:秒`。

### 查询进程

```bash
ps -l
ps -ef -u root | grep mysqld
```

- `ps -ef`：显示所有进程详细信息。
- `-u root`：筛选 root 用户的进程。
- `grep mysqld`：查找包含 `mysqld` 关键字的进程。

### 强行终止进程

```bash
kill -9 3306
```

- `-9`：强制终止进程，`3306` 是进程 ID。
### 执行进程

```bash
// 后台执行,将命令的信息重定向到指定的文件
nohup command [参数] > output_file 2>&1 &
// 例子:
nohup java -jar -Dspring.profiles.active=dev ./javaProject.jar > ./log.log 2>&1 &
```
### 查看端口是否被占用

```bash
netstat -anp | grep 3306
```

```bash
netstat -nltp
```
- `-a` → 列出所有的端口。
- `-n` → 直接显示 IP 和端口，不做域名解析。
- `-l` → 仅显示处于监听状态的套接字。
- `-t` → 显示 TCP 连接。
- `-p` → 显示进程 PID 和程序名。
### 进程调度算法
- 确定进程执行的先后以实现最大 CPU 利用率
	- **先到先服务调度算法**(FCFS，First Come, First Served) : 从就绪队列中选择一个最先进入该队列的进程为之分配资源，使它立即执行并一直执行到完成或发生某事件而被阻塞放弃占用 CPU 时再重新调度。
	- **短作业优先的调度算法**(SJF，Shortest Job First) : 从就绪队列中选出一个估计运行时间最短的进程为之分配资源，使它立即执行并一直执行到完成或发生某事件而被阻塞放弃占用 CPU 时再重新调度。
	- **时间片轮转调度算法**（RR，Round-Robin） : 时间片轮转调度是一种最古老，最简单，最公平且使用最广的算法。每个进程被分配一个时间段，称作它的时间片，即该进程允许运行的时间。
	- **多级反馈队列调度算法**（MFQ，Multi-level Feedback Queue）：前面介绍的几种进程调度的算法都有一定的局限性。如短进程优先的调度算法，仅照顾了短进程而忽略了长进程 。多级反馈队列调度算法既能使高优先级的作业得到响应又能使短作业（进程）迅速完成，因而它是目前被公认的一种较好的进程调度算法，UNIX 操作系统采取的便是这种调度算法。
	- **优先级调度算法**（Priority）：为每个流程分配优先级，首先执行具有最高优先级的进程，依此类推。具有相同优先级的进程以 FCFS 方式执行。可以根据内存要求，时间要求或任何其他资源要求来确定优先级。
### 僵尸进程和孤儿进程
- 僵尸进程  子进程由于某些原因停止运行, 但是他的父进程仍然还在运行, 必须通过wait()或者wiatpid()来回收父进程
- 孤儿进程  父进程意外终止, 子进程仍然运行, 没有调用wait()或者waitpid()回收进程, 但是操作系统会通过将子进程的父进程设置为init, 由init进程来回收子进程
## 服务管理

### 查看服务状态

```bash
systemctl status firewalld.service
```

- 检查 `firewalld` 防火墙的运行状态。

### 启用服务自启动

```bash
systemctl enable firewalld.service
```

- 设置 `firewalld` 开机自启动。
## 文件和目录操作

### 复制文件(仅在源文件比目标文件新时执行)

```bash
cp -u 源文件 目标文件夹path
```

- `-u`：仅在目标文件较旧或不存在时才复制。

### 移动文件（仅在目标文件较旧时执行）

```bash
mv -u 源文件 目标文件夹path
```

- `-u`：仅在目标文件较旧或不存在时才移动。

### 修改文件权限

```bash
chmod -Rv 777 文件夹/文件
```

- `-R`：递归修改所有子目录。
- `-v`：显示修改的过程。

![17582812784061758281277599.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17582812784061758281277599.png)

### 列出文件（不包括隐藏文件）

```bash
ls -l 文件名
```

- `-l`:  列出当前目录可见文件详细信息
- `-al`: 列出所有文件,包含隐藏文件

### 实时查看文件末尾

```bash
tail -n 10 -f 文件名
```

- `-n 10`：显示最后 10 行。
- `-f`：实时更新输出内容。

### 向文件追加内容

```bash
echo "hello" >> a.txt
```

- `>>` 表示追加内容到 `a.txt` 文件。

### 查找特定类型文件

```bash
find /package -name '*.sh'
```

- 在 `/package` 目录下查找所有 `.sh` 结尾的文件。

### 对文件内容排序

```bash
sort -ur number.txt | head -n 5
```

- `-u`：去重。
- `-r`：降序排序。
- `head -n 5`：取前 5 行。

### 压缩和解压缩文件

压缩文件1和文件2 为一个文件:

```bash
tar -zcvf 压缩后文件名.tar.gz 源文件1 源文件2
```

- `-z`：使用 gzip 压缩。
- `-c`：创建压缩包。
- `-v`：显示详细信息。
- `-f`：指定文件名。

解压缩：

```bash
tar -zxvf 文件.tar.gz
```

- `-z`：使用 gzip 压缩。
- `-x`：解压缩。
- `-v`：显示详细信息。
- `-f`：指定文件名。
## 网络管理

### 设置命令别名

```bash
alias ll='ls -l --color=auto'
```

- `alias` 用于给命令取别名。

### 启动网络服务

```bash
service network start
```

- 开启网络。

### 启用/禁用网卡

```bash
ifconfig ens33 up   # 启用网卡
ifconfig ens33 down # 禁用网卡
```

### 测试网络连通性

```bash
ping -c 3 www.baidu.com
```

- `-c 3`：只执行 3 次 ping。

### 查看端口占用情况

```bash
netstat -anp | grep 3306
```

- `-a`：显示所有连接。
- `-n`：显示数字格式的 IP 地址和端口。
- `-p`：显示进程信息。
## 系统管理

### 查看磁盘使用情况

```bash
df -h --total
```

- `-h`：以人类可读的格式显示。
- `--total`：显示总计信息。

### 查看块设备

```bash
lsblk
```

- 以树状结构显示所有块设备。

### 查看内存、CPU、进程、I/O 等性能状态

```bash
vmstat 1 # 每隔1s显示系统情况
```

### 查看 系统内存的使用情况

```bash
free -h  # -h人类可读
```
### 设置虚拟网卡
```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```
## 脚本Shell
- 指定脚本解释器写,shell的习惯第一行指定解释器 例如  #!jdk17/bin/java.exe
![17583670223771758367021565.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17583670223771758367021565.png)
![17583670422641758367041783.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17583670422641758367041783.png)
```bash
#!/bin/sh   
echo $home  # 查询系统变量
name = 'root'
hello='Hello, I am $name!' 
echo $hello  # 输出{变量}

str="iLoveChina" 
echo ${str} # 输出{变量}

array=(! @ # $ %)  
length=${#array[@]}
echo ${array[1]}

a=1
b=2
# if条件判断 判断a是否大于b
if [ ${a} -eq ${b} ]; then
    echo "a=b"
elif [ ${a} -gt ${b} ]; then
    echo "a>b"
else
    echo "a<b"
fi

# for循环
for loop in A B C D E F G 
do
    echo "顺序输出字母为: $loop"
done

# 函数定义
d() {
    echo "请录入第一个数"
    read number1
    echo "请录入第二个数"
    read number2
    echo "两个数字分别为 ${number1}, ${number2}"
    return $((number1 + number2))
}
d
echo $?
```
## Vim 编辑器

### 普通模式命令

#### 光标移动

```java
h：向左移动
j：向下移动
k：向上移动
l：向右移动
0：移动到行首
$：移动到行尾
gg：移动到文件开头
G：移动到文件结尾
```

#### 文本操作

```java
x：删除光标下的字符
dd：删除当前行
yy：复制当前行
p：粘贴（在光标后插入）
u：撤销
Ctrl + r：重做
```

#### 查找和替换

```java
/text：向下查找 text
?text：向上查找 text
n：跳转到下一个匹配
N：跳转到上一个匹配
```

### 命令行模式命令

#### 文件操作

```java
:w：保存文件
:q：退出 vi
:wq 或 ZZ：保存并退出
:q!：强制退出而不保存
:set nu：显示行号
:set nonu：隐藏行号
```

#### 撤销与重做

```java
u：撤销上一步操作
Ctrl + r：重做上一步操作
```

#### 文件操作

```java
:e filename：打开新文件
:r filename：将文件内容插入当前文件
```

#### 查找内容

```java
/关键字  # 向下查找关键字
?关键字  # 向上查找关键字
```


## AB压力测试
- Apache服务器的性能测试工具, 会创建并发线程访问同一个url(Web服务)
- `ab -c 10 -n 100 http://www.myvick.cn/index.php`：同时处理100个请求并运行10次index.php
	- -c10表示并发用户数为10
	- -n100表示请求总数为100
```bash
Server Software:        nginx/1.13.6   #测试服务器的名字
Server Hostname:        www.myvick.cn  #请求的URL主机名
Server Port:            80             #web服务器监听的端口

Document Path:          /index.php　　  #请求的URL中的根绝对路径，通过该文件的后缀名，我们一般可以了解该请求的类型
Document Length:        799 bytes       #HTTP响应数据的正文长度

Concurrency Level:      10　　　　　　　　# 并发用户数，这是我们设置的参数之一
Time taken for tests:   0.668 seconds   #所有这些请求被处理完成所花费的总时间 单位秒
Complete requests:      100 　　　　　　  # 总请求数量，这是我们设置的参数之一
Failed requests:        0　　　　　　　　  # 表示失败的请求数量，这里的失败是指请求在连接服务器、发送数据等环节发生异常，以及无响应后超时的情况
Write errors:           0
Total transferred:      96200 bytes　　　 #所有请求的响应数据长度总和。包括每个HTTP响应数据的头信息和正文数据的长度
HTML transferred:       79900 bytes　　　　# 所有请求的响应数据中正文数据的总和，也就是减去了Total transferred中HTTP响应数据中的头信息的长度
Requests per second:    149.71 [#/sec] (mean) #吞吐率，计算公式：Complete requests/Time taken for tests  总请求数/处理完成这些请求数所花费的时间
Time per request:       66.797 [ms] (mean)   # 用户平均请求等待时间，计算公式：Time token for tests/（Complete requests/Concurrency Level）。处理完成所有请求数所花费的时间/（总请求数/并发用户数）
Time per request:       6.680 [ms] (mean, across all concurrent requests) #服务器平均请求等待时间，计算公式：Time taken for tests/Complete requests，正好是吞吐率的倒数。也可以这么统计：Time per request/Concurrency Level
Transfer rate:          140.64 [Kbytes/sec] received  #表示这些请求在单位时间内从服务器获取的数据长度，计算公式：Total trnasferred/ Time taken for tests，这个统计很好的说明服务器的处理能力达到极限时，其出口宽带的需求量。

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   0.7      2       5
Processing:     2   26  81.3      3     615
Waiting:        1   26  81.3      3     615
Total:          3   28  81.3      6     618

```
## 问答
### 如何发起Http请求
- 不同网络主机要通信一必须通过**Socket(套接字)**建立**传输层连接**（TCP/UDP）一→然后在上层跑具体的**应用层协议**(HTTP、MQTT、WebSocket等）实现数据格式约定和业务逻辑。
```bash
curl https://baidu.com  # Get请求
curl -X POST https://baidu.com "name=123&password=456"

wget https://baidu.com
``` 
### 什么是线程的死锁
- 互斥条件:  一个资源一个线程, 如果一个线程持有资源, 那么其他线程想要访问这个资源必须等待
- 持有并等待条件:  一个资源被一个线程占有一段时间, 另一个线程必须死等
- 不可剥夺条件:  已经分配资源给进程, 该资源不可被强制剥夺, 并主动释放
- 循环等待条件:  a持有资源1 请求支援2;  b持有资源2 请求资源1;  想当与循环依赖
### 怎么在Linux上安装Java环境
1. 下载Linux版本的jdk安装包
2. tar -zxvf 解压缩安装包
3. 配置环境变量 export JAVA_HOME=/usr/local/java/jdk17
4. 配置路径变量 export PATH=$JAVA-HOME/bin: $PATH
5. 刷新配置 source /etc/profile
6. 使用 java --version验证是否安装成功
7. 使用`echo "$PATH"`查看是否加入到系统bin目录
### `ps`和`netstat`的区别
| 特性   | netstat              | ps                     |
| ---- | -------------------- | ---------------------- |
| 关注点  | 网络连接、端口、协议状态         | 进程运行状态、CPU/内存资源        |
| 典型用途 | 看端口是否被占用，连接是否正常      | 看进程是否运行，消耗多少资源         |
| 输出重点 | TCP/UDP 连接，监听端口，路由   | PID、父子进程关系、命令、用户       |
| 结合场景 | 常和 `ps` 配合用来查哪个进程占端口 | 常和 `netstat` 配合找进程网络行为 |
- netstat注重端口(网络连接);  ps注重进程情况
### 如何统计 HTTPS 服务（通常运行在端口 443）每秒的请求数
```bash
watch -n 1 "netstat -an | grep ':443'""
```
- `netstat -an`：显示所有连接和监听端口。
- `grep ':443 '`：过滤出端口 443 的连接。
- `grep ESTABLISHED`：过滤出已经建立的连接。
- `wc -l`：统计连接数。
- `watch -n 1`：每秒刷新一次命令的输出。
### 什么是软链接和硬链接
| 特性       | 软链接 (Symbolic)       | 硬链接 (Hard)         |
| -------- | -------------------- | ------------------ |
| inode    | 独立 inode，存储路径        | 和源文件 inode 相同      |
| 指向对象     | 文件路径（间接指向数据）         | 文件本身（直接指向数据）       |
| 是否可跨文件系统 | ✅ 可以                 | ❌ 不可以              |
| 是否可链接目录  | ✅ 可以                 | ❌ 不可以              |
| 删除源文件影响  | ❌ 软链失效（死链）           | ✅ 不影响（数据还在）        |
| 文件大小     | 路径字符串长度              | 和源文件大小相同           |
| 典型用途     | 快捷方式、版本切换            | 数据保护、防止误删          |
| 命令       | ln -s target newFile | ln  target newFile |
- 软链接保存的是指向文件的路径, 文件删除则软链接变死链;
- 硬链接是“同一个文件的多个名字”, 删除源文件, 但是有副本