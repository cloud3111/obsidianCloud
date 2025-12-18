---
banner: "[[pixel-banner-image.png]]"
title: "深入缓冲区: 理解PageCache, Socket与零拷贝"
tags:
  - ZeroCopy
  - Buffer
  - Kernel
categories: 编程
date: 2025-09-17T10:51:00
---

	良言难劝该死的鬼, 慈悲不度自绝人💔

# 深入缓冲区: 理解PageCache, Socket与零拷贝
- 内核空间的数据是通过操作系统层面的 I/O 接口从磁盘读取或写入。

代码通常如下，一般会需要两个系统调用：read()  write()
## 用户态与内核态
- 内核态：操作系统内核运行的空间，权限最高，可以直接操作硬件资源, 像文件管理, 进程管理, 内存管理这类 
	- 凡是与系统态级别的资源有关的操作（如文件管理、进程管理、内存管理等)，都必须通过系统调用方式向操作系统提出服务请求，并由操作系统代为完成
- 用户态：应用程序运行的空间，权限受限，不能直接操控硬件，必须通过系统调用向内核请求服务。  
- 关键点：应用与硬件之间的数据交互，必须经过内核的参与。  
- 进入内核态需要付出较高的开销（需要进行一系列的上下文切换和权限检查）
![17583324712651758332470345.png|700x176](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17583324712651758332470345.png)
## 缓冲区(Buffer)
### 页缓存（Page Cache）
- 定义：内核为磁盘文件数据提供的缓存区，一页差不多4KB; 核心概念 -> 用读内存替换读磁盘
- 作用：降低磁盘 I/O 开销。 
  - 读：文件若已存在于 Page Cache，`read()` 会直接从内存返回
  - 写：数据先写入 Page Cache，再由内核异步写回磁盘（延迟写）  
- 当要从硬盘读取GB大小的文件时, DMA拷贝文件时, 也会使用PageCache缓存文件, 但是这时候内存缓存空间也会被占用, 导致其他热点文件无法使用PageCache, 最终磁盘IO能力下降
### Socket 缓冲区
- 位置：内核态，用于存放即将发送或接收的网络数据
- 作用：维持数据传输的速率和稳定性，确保不会丢包
- 应用：应用层的 `send/recv` 实际是向 Socket 缓冲区写入或读取数据
### 用户缓冲区
- 位置：用户态的内存空间
- 举例: 像`BufferedInputStream`就是一种伪用户缓冲区
- 作用：应用程序处理数据的中转区
- 缺陷：与内核缓冲区之间的数据交互需要额外的 CPU 拷贝
## DMA
- DMA 是 直接内存访问 的缩写  Directly Memory Access 
- 它是一个硬件上的小芯片，一定程度上替代了CPU的职责, 减少了CPU的性能消耗, 从而提高 CPU 利用效率
- 在进行 I/O 设备和内存的数据传输的时候，数据搬运的工作全部交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情，这样 CPU 就可以去处理别的事务
## 文件传输方式对比
### 1. 普通 I/O (`read` + `write`)
1. 磁盘数据 → DMA 拷贝 → Page Cache。  
2. Page Cache → CPU 拷贝 → 用户缓冲区。  
3. 用户缓冲区 → CPU 拷贝 → Socket 缓冲区。  
4. Socket 缓冲区 → DMA 拷贝 → 网卡。  
![17580782241251758078223678.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17580782241251758078223678.png)
### 2. `mmap` + `write`（用户态零拷贝）
- **将磁盘上的物理文件直接映射到用户态的内存地址中, 相当于一片共享空间**  
- 用 mmap() 替换 read() 系统调用函数 buf = mmap(file, len)
1. `mmap` 将文件页映射到用户态地址空间，应用可直接操作 Page Cache。  
2. 写 socket 时：Page Cache → CPU 拷贝 → Socket 缓冲区 → DMA 拷贝 → 网卡。 
![17580783321201758078331448.png|700x439](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17580783321201758078331448.png)
### 3. `sendfile`(网络零拷贝)
1. 磁盘数据 → DMA 拷贝 → Page Cache。  
2. Page Cache 的页直缓冲区描述符和数据长度传到 socket 缓冲区(元数据传递，无实际数据拷贝)
3. PageCache → DMA 拷贝 → 网卡。  
![17582840954991758284095244.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17582840954991758284095244.png)
### 4. Direct I/O
- 绕过 Page Cache，应用直接与磁盘交互。  
- 优点：避免双重缓存（例如数据库有自己的缓存机制）。  
- 缺点：每次读写都要访问磁盘，性能较低。  
## 用户态零拷贝至网络传输的流程
1. 用户进程调用mmap方法, 将用户态切换至内核态, 然后建立起用户缓冲区和内核缓冲区(PageCache)的内存地址映射, 也就是所谓的共享区域
2. CPU调度DMA控制器将磁盘数据拷贝至内核缓冲区; 这是用户如果要修改文件的话, 先将内核态切换为用户态;
3. 用户对数据进行修改后调用write(), 再将用户态切回内核态;
4. CPU将内核缓冲区(PageCache)的数据拷贝至Socket缓冲区;
5. CPU调度DMA将数据从Socket缓冲区拷贝至网卡;
## 用户态零拷贝和网络零拷贝的区别
- **mmap**(MappedByteBuffer) → 用户态可直接访问，**适合文件处理**，减少用户态 ↔ 内核态拷贝。
- **sendfile** → **文件直接发往网络**，Java 层无法访问数据, 也无法修改, 只能用于**网络传输**。
- 选择哪种方式取决于 你要做的 I/O 类型：
    - 文件处理 / 修改 → mmap
    - 文件传输 / 网络发送 → sendfile
![17580801240121758080123618.png|700x176](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17580801240121758080123618.png)
## 总结
- 三大缓冲区：用户缓冲区, 内核缓冲区分为 页缓冲区(PageCache) 和 Socket缓冲区
- 零拷贝的本质：**减少用户态与内核态的上下文切换**和**内存拷贝的次数** 
- Kafka、Nginx、 RocketMQ等高性能框架，均大量使用**零拷贝技术**来提升I/O效率(减少IO次数)
- RocketMQ 内部主要是使用基于 mmap 实现的零拷贝(其实就是调用上述提到的 api)，用来读写文件，这也是 RocketMQ 为什么快的一个很重要原因。
- 零拷贝 无论是mmap还是sendfile都用到了PageCache技术