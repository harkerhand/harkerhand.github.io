---
title: 几种IO方式
date: 2026-03-02 14:34:01
tags: 
categories: TechMagic 
index_img: ./io/demux.png
banner_img:
---

> 面试问到了，重新梳理一下

# 一次 IO 到底发生了什么？

以读 socket 数据为例。
一次 read 实际上包含两个阶段：

1. 等待数据准备好（内核态）
    * 数据从网卡 → DMA → 内核缓冲区
    * 这个过程应用程序**完全无法参与**
    * 只能等

2. 数据从内核拷贝到用户空间
    * 内核 buffer → 用户 buffer
    * 这是一次内存拷贝


一次 IO = 等待数据 + 拷贝数据

所有 IO 模型的区别，都来自：

* 等待阶段谁在等？
* 拷贝阶段谁在等？
* 是否阻塞线程？
* 是否需要主动轮询？

# 阻塞 IO（Blocking IO）

这是最传统的模型。

```c
read(fd, buffer, size);
```

如果数据没到：

* 线程直接睡眠
* 内核等数据
* 数据到达后拷贝
* read 返回

## 特点

* 等待阶段：阻塞
* 拷贝阶段：阻塞
* 整个线程完全卡住

### 优点

* 简单
* 易写
* 易理解

### 缺点

* 1个线程只能处理1个连接
* 高并发会炸线程

{% fold info @示例代码 %}
```c
#include <stdio.h>    
#include <stdlib.h>   
#include <string.h>   
#include <unistd.h>   
#include <arpa/inet.h>
#define PORT 9999
#define BUFFER_SIZE 1024

int main()
{
    int server_fd, client_fd;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    char buffer[BUFFER_SIZE] = {0};

    // 1. 创建套接字 (IPv4, TCP)
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0)
    {
        perror("Socket 创建失败");
        exit(EXIT_FAILURE);
    }

    // 2. 准备地址和端口
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY; // 监听所有网卡 (0.0.0.0)
    address.sin_port = htons(PORT);       // 转换为网络字节序

    // 3. 绑定
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0)
    {
        perror("绑定失败");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 4. 监听
    if (listen(server_fd, 3) < 0)
    {
        perror("监听失败");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("服务端启动，正在监听端口 %d...\n", PORT);

    // 5. 接受连接 阻塞 线程睡眠，等待客户端连接
    printf("等待客户端连接...\n");
    if ((client_fd = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen)) < 0)
    {
        perror("接受连接失败");
        close(server_fd);
        exit(EXIT_FAILURE);
    }
    printf("客户端已连接！地址: %s\n", inet_ntoa(address.sin_addr));

    // 6. 循环读取数据
    while (1)
    {
        memset(buffer, 0, BUFFER_SIZE); // 清空缓冲区

        // recv 阻塞 线程睡眠，等待数据到达
        ssize_t valread = recv(client_fd, buffer, BUFFER_SIZE - 1, 0);

        if (valread > 0)
        {
            printf("[客户端]: %s", buffer);
        }
        else if (valread == 0)
        {
            printf("客户端断开连接。\n");
            break;
        }
        else
        {
            perror("读取出错");
            break;
        }
    }

    // 7. 关闭
    close(client_fd);
    close(server_fd);
    return 0;
}
```

{% endfold %}


# 非阻塞 IO（Non-blocking IO）

设置描述符为非阻塞

```c
fcntl(fd, F_SETFL, O_NONBLOCK);

read(fd, buffer, size);
```

如果数据没到：

* 立刻返回 -1
* errno = EAGAIN

不会睡眠


但它只是**不等待数据**，它没有解决**什么时候知道数据来了**的问题

所以必须轮询，这就导致了：
* CPU 空转
* 效率低
* 不能规模化

{% fold info @示例代码 %}
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <arpa/inet.h>

#define PORT 9999

void set_nonblocking(int fd)
{
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main()
{
    int server_fd, client_fd = -1;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    char buffer[1024];

    // 1. 创建并设置监听 Socket 为非阻塞
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblocking(server_fd);

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, 3);

    printf("非阻塞服务端已启动，监听 %d...\n", PORT);

    while (1)
    {
        // 2. 尝试接受连接 (非阻塞 accept)
        client_fd = accept(server_fd, (struct sockaddr *)&address, (socklen_t *)&addrlen);

        if (client_fd < 0)
        {
            if (errno == EAGAIN || errno == EWOULDBLOCK)
            {
                // 没有新连接
                printf("等待连接中...\n");
            }
            else
            {
                perror("accept error");
            }
        }
        else
        {
            printf("客户端已连接！\n");
            set_nonblocking(client_fd); // 把客户端也设为非阻塞

            // 3. 尝试读取数据 (非阻塞 recv)
            while (1)
            {
                memset(buffer, 0, 1024);
                ssize_t bytes = recv(client_fd, buffer, 1023, 0);

                if (bytes > 0)
                {
                    printf("[收到数据]: %s", buffer);
                }
                else if (bytes == -1)
                {
                    if (errno == EAGAIN || errno == EWOULDBLOCK)
                    {
                        printf("暂无数据...\n");
                        sleep(1);
                    }
                    else
                    {
                        perror("读取错误");
                        close(client_fd);
                        client_fd = -1;
                        break;
                    }
                }
                else if (bytes == 0)
                {
                    printf("客户端断开了。\n");
                    close(client_fd);
                    client_fd = -1;
                    break;
                }
            }
        }
        sleep(1);
    }
    close(server_fd);
    return 0;
}
```
{% endfold %}


# IO 多路复用（select / poll / epoll）

它解决的是**如何高效等待多个 fd**

不对每个 fd 调 read

而是先问内核：哪些 fd 可以读？

内核来等。

## 时间线

```
epoll_wait()
    ↓
[ 等待数据 ] ← 阻塞在这里
    ↓
返回就绪fd
    ↓
read()
    ↓
[ 数据拷贝 ] ← 阻塞
    ↓
返回
```

## 关键理解

IO多路复用：

* 等待阶段：由内核统一等待
* 拷贝阶段：仍然是同步的

所以它本质是同步 IO，只是等待更高效

## Select / Poll / Epoll


### select（最原始，效率最低）

* **原理**：传给内核一个固定大小的位图，里面存了所有要监控的 FD。
* 痛点 1（限制）：位图长度有限，默认只能监控 1024 个 FD。
* 痛点 2（拷贝）：每次调用都要把这个位图从用户态全量拷贝到内核态。
* 痛点 3（遍历）：内核发现有数据了，它不告诉你是哪个，而是把位图还给你。你得在用户态亲自写个循环从 1 扫到 1024，看看到底是谁有数。

### poll（select 的小幅改良版）

* 原理：不用位图了，改用**结构体数组**（链表实现）。
* 进步：没有了 1024 的数量限制。
* 原地踏步：它依然需要全量拷贝数组，内核依然不告诉你是谁有数，你还是得手动 O(n) 遍历整个数组。

### epoll（Linux 的终极方案，性能王者）

它之所以快，是因为在内核里做了三件大事：

1. 红黑树：在内核里开了一棵树，想监控哪个 FD 就往树上挂。这样不用每次调用都拷贝全量列表，只需告诉内核变动了哪一个。
2. 回调机制：内核给每个 FD 挂了钩子。一旦数据到了，内核自动执行回调，把这个 FD 塞进一个就绪链表。
3. 就绪链表：当你调用 `wait` 时，内核直接把这个“有数”的链表丢给你。复杂度是 O(1)，不需要去遍历那些没动静的死链接。


{% fold info @示例代码 %}
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <arpa/inet.h>
#include <sys/epoll.h> // epoll 核心头文件

#define PORT 9999
#define MAX_EVENTS 10 // 一次最多处理多少个事件
#define BUFFER_SIZE 1024

// 将文件描述符设为非阻塞
void set_nonblocking(int fd)
{
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main()
{
    int server_fd, client_fd, epoll_fd;
    struct sockaddr_in address;
    struct epoll_event ev, events[MAX_EVENTS];

    // 1. 创建并设置监听 Socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblocking(server_fd);

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&address, sizeof(address));
    listen(server_fd, 128);

    // 2. 创建 epoll 实例
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1)
    {
        perror("epoll_create1 失败");
        exit(EXIT_FAILURE);
    }

    // 3. 将监听 Socket 加入 epoll 监视列表
    ev.events = EPOLLIN; // 关注读事件（有人连接或发消息）
    ev.data.fd = server_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1)
    {
        perror("epoll_ctl 添加监听 socket 失败");
        exit(EXIT_FAILURE);
    }

    printf("epoll 服务端已启动，监听端口 %d...\n", PORT);

    while (1)
    {
        // 4. 等待事件发生（无限期阻塞，直到有事发生，不消耗 CPU）
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds == -1)
        {
            perror("epoll_wait 出错");
            break;
        }

        // 5. 遍历所有发生的事件
        for (int i = 0; i < nfds; ++i)
        {
            if (events[i].data.fd == server_fd)
            {
                // 如果是 server_fd 有消息，说明有新客户端连接
                struct sockaddr_in client_addr;
                socklen_t client_len = sizeof(client_addr);
                client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);

                printf("新客户端连接: %s\n", inet_ntoa(client_addr.sin_addr));

                set_nonblocking(client_fd);    // 设为非阻塞
                ev.events = EPOLLIN | EPOLLET; // 监听读事件，使用边缘触发(ET)模式
                ev.data.fd = client_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
            }
            else
            {
                // 如果是 client_fd 有消息，说明客户端发来了字符
                char buffer[BUFFER_SIZE];
                int fd = events[i].data.fd;
                ssize_t bytes = recv(fd, buffer, sizeof(buffer) - 1, 0);

                if (bytes > 0)
                {
                    buffer[bytes] = '\0';
                    printf("[收到数据]: %s", buffer);
                }
                else if (bytes == 0)
                {
                    printf("客户端断开连接。\n");
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL); // 从监视列表中移除
                    close(fd);
                }
                else
                {
                    if (errno != EAGAIN)
                    {
                        perror("读取错误");
                        close(fd);
                    }
                }
            }
        }
    }

    close(server_fd);
    close(epoll_fd);
    return 0;
}
```

{% endfold %}

# 信号驱动 IO（Signal-driven IO）

```c
sigaction(SIGIO, handler);
fcntl(fd, F_SETFL, O_ASYNC);
```

调用后：
* 立刻返回
* 内核负责等待
* 完成后通知（信号）
* 数据拷贝阶段仍然是同步的

{% fold info @示例代码 %}
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <signal.h>
#include <errno.h>
#include <arpa/inet.h>

int server_fd, client_fd;

// 信号处理函数：当内核发送 SIGIO 时执行
void io_handler(int sig)
{
    // 1. 尝试接受新连接
    if (client_fd == -1)
    {
        struct sockaddr_in addr;
        socklen_t addrlen = sizeof(addr);
        int new_client = accept(server_fd, (struct sockaddr *)&addr, &addrlen);
        if (new_client >= 0)
        {
            client_fd = new_client;
            printf("\n[信号触发] 客户端已连接！地址: %s\n", inet_ntoa(addr.sin_addr));

            // 对新的 client_fd 也设置信号驱动
            struct sigaction sa;
            memset(&sa, 0, sizeof(sa));
            sa.sa_handler = io_handler;
            sigaction(SIGIO, &sa, NULL);

            fcntl(client_fd, F_SETOWN, getpid());
            int flags = fcntl(client_fd, F_GETFL, 0);
            fcntl(client_fd, F_SETFL, flags | O_ASYNC | O_NONBLOCK);
        }
        return;
    }

    // 2. 接收数据
    char buffer[1024];
    ssize_t bytes = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (bytes > 0)
    {
        buffer[bytes] = '\0';
        printf("\n[信号触发] 收到数据: %s", buffer);
    }
    else if (bytes == 0)
    {
        printf("\n[信号触发] 客户端断开。\n");
        close(client_fd);
        client_fd = -1;
    }
}

int main()
{
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(9999);

    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 5);

    // --- 对 server_fd 设置信号驱动 I/O ---
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = io_handler;
    sigaction(SIGIO, &sa, NULL);

    fcntl(server_fd, F_SETOWN, getpid());
    int flags = fcntl(server_fd, F_GETFL, 0);
    fcntl(server_fd, F_SETFL, flags | O_ASYNC | O_NONBLOCK);

    client_fd = -1; // 初始化为无效
    printf("等待连接...\n");

    // --- 主程序干自己的活 ---
    while (1)
    {
        printf(".");
        fflush(stdout);
        sleep(1); // 模拟主线程繁忙
    }

    return 0;
}
```

{% endfold %}


# 同步 IO vs 异步 IO

这是最容易混淆的地方。

区别标准：数据拷贝阶段是否由用户线程等待


## 同步 IO（Synchronous IO）

特点：

调用 `read` 数据拷贝完成前线程不能继续

包括：

* 阻塞 IO
* 非阻塞 IO
* IO 多路复用

它们都是同步 IO。


## 异步 IO（Asynchronous IO）

```c
aio_read(...)
```

调用后：

* 立刻返回
* 内核负责等待
* 内核负责拷贝
* 完成后通知（信号 / 回调 / 事件）


从头到尾都没有阻塞

{% fold info @示例代码 %}
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <aio.h>
#include <unistd.h>
#include <pthread.h>

#define BUF_SIZE 1024

void aio_callback(union sigval sv)
{
    struct aiocb *cb = (struct aiocb *)sv.sival_ptr;
    ssize_t bytes_read = aio_return(cb);
    if (bytes_read > 0)
    {
        printf("\n[回调触发] 读取完成！内容如下：\n%.*s\n", (int)bytes_read, (char *)cb->aio_buf);
    }
    else if (bytes_read == 0)
    {
        printf("\n[回调触发] 文件为空。\n");
    }
    else
    {
        printf("\n[回调触发] 读取失败：%s\n", strerror(aio_error(cb)));
    }
    close(cb->aio_fildes);
    free((void *)cb->aio_buf);
    free(cb);
}

int main()
{
    int fd = open("aio_read.c", O_RDONLY);
    if (fd < 0)
    {
        perror("Open 失败");
        return 1;
    }

    struct aiocb *cb = malloc(sizeof(struct aiocb));
    char *buffer = malloc(BUF_SIZE);

    memset(cb, 0, sizeof(struct aiocb));
    cb->aio_fildes = fd;
    cb->aio_buf = buffer;
    cb->aio_nbytes = BUF_SIZE;
    cb->aio_offset = 0;

    // 设置异步通知方式并注册回调函数
    cb->aio_sigevent.sigev_notify = SIGEV_THREAD;
    cb->aio_sigevent.sigev_notify_function = aio_callback;
    cb->aio_sigevent.sigev_value.sival_ptr = cb;

    if (aio_read(cb) < 0)
    {
        perror("AIO 读取失败");
        close(fd);
        free(buffer);
        free(cb);
        return 1;
    }
    printf("异步读取已发起，等待回调...\n");

    // 主线程可以继续执行其他任务
    for (int i = 0; i < 5; i++)
    {
        printf("主线程正在执行其他任务... %d\n", i + 1);
        sleep(1);
    }

    printf("主程序退出\n");
    return 0;
}
```

{% endfold %}

## IO_uring

io_uring 是 Linux 5.1 引入的一个全新的异步 IO 框架，旨在提供更高效、更灵活的异步 IO 处理方式。它通过使用共享内存环形缓冲区来减少系统调用的开销，实现了真正的零拷贝异步 IO.

{% fold info @示例代码 %}

```c
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <stdlib.h>
#include <liburing.h>
#include <unistd.h>

#define ENTRIES 4 // 队列深度
#define BUF_SIZE 1024

int main()
{
    struct io_uring ring;
    struct io_uring_sqe *sqe; // 提交项
    struct io_uring_cqe *cqe; // 完成项
    char buffer[BUF_SIZE];
    struct iovec iov; // 描述缓冲区的结构

    // 1. 初始化 io_uring 实例
    io_uring_queue_init(ENTRIES, &ring, 0);

    // 打开一个文件进行测试
    int fd = open("io_uring.c", O_RDONLY);
    if (fd < 0)
    {
        perror("open");
        return 1;
    }

    // 2. 准备缓冲区信息
    iov.iov_base = buffer;
    iov.iov_len = sizeof(buffer);

    // 3. 获取一个提交项 (SQE)
    sqe = io_uring_get_sqe(&ring);

    // 4. 设置异步读取任务
    // 告诉内核：帮我读 fd，读到 iov 里，偏移量为 0
    io_uring_prep_readv(sqe, fd, &iov, 1, 0);

    // 5. 提交任务给内核
    // 这个动作类似发令枪，内核收到后就在后台开始搬运数据了
    io_uring_submit(&ring);

    printf("任务已提交至 io_uring，主线程可以去处理其他逻辑了...\n");

    // 6. 等待并获取任务完成结果 (阻塞直到有一个任务完成)
    int ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0)
    {
        perror("io_uring_wait_cqe");
        return 1;
    }

    if (cqe->res > 0)
    {
        printf("读取完成！内容：\n%.*s\n", cqe->res, buffer);
    }

    // 7. 标记该完成项已被处理
    io_uring_cqe_seen(&ring, cqe);

    // 8. 清理
    close(fd);
    io_uring_queue_exit(&ring);

    return 0;
}
```

{% endfold %}


# 整体对比与总结

| 模型        | 等待阶段     | 拷贝阶段 | 是否阻塞线程 |
| ----------- | ------------ | -------- | ------------ |
| 阻塞 IO     | 阻塞         | 阻塞     | 是           |
| 非阻塞 IO   | 不阻塞       | 阻塞     | 部分         |
| IO 多路复用 | 阻塞在select | 阻塞     | 是           |
| 信号驱动 IO | 内核通知     | 阻塞     | 部分         |
| 异步 IO     | 内核处理     | 内核处理 | 否           |
