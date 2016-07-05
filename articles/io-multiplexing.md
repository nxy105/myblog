# I/O 多路复用 

### 什么是多路复用
I/O 多路复用是为了解决进程或者线程被某个 I/O 系统调用而阻塞的技术。I/O 多路复用通过一种机制，监听多个描述符，一旦某个描述符处于就绪状态，能通知程序进行读写操作。

因此，I/O 多路复用机制设计上遵循以下原则：

- 当任何一个文件描述符 I/O 就绪时进行通知。
- 在有可用的文件描述符之前一直处于睡眠状态。
- 处理所有 I/O 就绪的文件描述符，没有阻塞。 

### 多路复用的实现模型
Linux 提供了三种 I/O 多路复用的方案：select，poll，epoll。

### 多路复用的实现

#### select()

```
#include <sys/select.h>
#include <sys/time.h>

int select(int nfds,fd_set *readset,fd_set *writeset,fd_set *exceptset, struct timeval *timeout)
```

监听并等待多个文件描述符的属性变化，在给定的文件描述符 I/O 就绪之前并且没有超出指定的时间限制，`select()`调用就会被阻塞。监听的文件描述符分为3类，等待不同的事件。

- **nfds**：要监视的文件描述符的范围，一般取监视的描述符数的最大值+1。
- **readset**：监视的可读描述符集合，只要有文件描述符即将进行读操作。
- **writefds**：监视的可写描述符集合。
- **exceptset**：监视的错误异常描述符集合。
- **timeout**：如果该参数不是 NULL，在 tv_sec 秒 tv_usec 微秒后，select()调用会返回。以下是该参数的结构体`timeval`定义；

```
struct timeval {
	long tv_sec;   
	long tv_usec; 
};
```

成功返回时，每个集合都修改成只包含响应类型的 I/O 就绪的文件描述符。举个例子，假定 readset 集合中有两个文件描述符 7 和 9。当调用返回时，如果文件描述符 7 还在集合中，它在 I/O 读取时不会阻塞。如果描述符 9 不在集合中，它在读取时很可能会发生阻塞。（因为在调用完成后，数据可能已经就绪了。）

系统提供了 4 个宏操作文件描述符集合：

```
// 清空集合
void FD_ZERO(fd_set *fdset);

//将一个给定的文件描述符加入集合之中
void FD_SET(int fd, fd_set *fdset);

//将一个给定的文件描述符从集合中删除
void FD_CLR(int fd, fd_set *fdset);

// 检查集合中指定的文件描述符是否可以读写 
int FD_ISSET(int fd, fd_set *fdset)
```

接下来，我们通过一个示例代码，了解`select()`的用法。在这个例子中，会阻塞等待 stdin 的输入，超时设置是 5 秒。由于只是监视单个文件描述符，该示例不算 I/O 多路复用，但它清晰的说明了如何使用系统调用：

```
#include <stdio.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

#define TIMEOUT 5 /* 设置超时时间为5秒 */
#define BUF_LEN 1024 /* 读取的buffer长度 */

int main (void)
{
    struct timeval tv;
    fd_set readfds;
    int ret;

    /* 等待键盘输入 */
    FD_ZERO(&readfds);
    FD_SET(STDIN_FILENO, &readfds);

    /* 设置等待时间 */
    tv.tv_sec = TIMEOUT;
    tv.tv_usec = 0;

    ret = select(STDIN_FILENO + 1, &readfds, NULL, NULL, &tv);
    if (ret == -1) {
        perror("select");
        return 1;
    } else if (!ret) {
        printf("%d seconds elapsed.\n", TIMEOUT);
        return 0;
    }

    /**
     * 读取文件描述符
     */
    if (FD_ISSET(STDIN_FILENO, &readfds)) {
        char buf[BUF_LEN];
        int len;

        /* 此时可以确保读取不会被阻塞 */
        len = read(STDIN_FILENO, buf, BUF_LEN);
        if (len == -1) {
            perror("read");
            return 1;
        }

        if (len) {
            buf[len] = '\0';
            printf("read: %s\n", buf);
        }

        return 0;
    }

    fprintf(stderr, "This should not happend!\n");
    return 1;
}
```

#### poll()

`poll()`系统调用是 System V 的 I/O 多路复用的解决方案。它解决了`select()`的不足。


```
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- **fd** `select()`使用了基于文件描述符的三位掩码的解决方案，其效率不高。`poll()`·使用了有 nfds 个 pollfd 结构体构成的数组，fds 指针指向该数组。pollfd 结构体定义如下：

```
struct pollfd{
	int fd;         /* 文件描述符 */
	short events;   /* 等待的事件 */
	short revents;  /* 实际发生了的事件 */
};
```
每个 pollfd 结构体指定一个被监视的文件描述符。可以给`poll()`传递多个 pollfd 结构体，使它能够监视多个文件描述符。每个结构体的 events 变量是要监视的文件描述符的事件的位掩码。用户可以设置该变量。revents 变量是该文件描述符的结果事件的位掩码。以下是合法的 events 值：

> - **POLLIN** 普通或优先级带数据可读
> - **POLLRDNORM** 普通数据可读
> - **POLLRDBAND** 优先级带数据可读
> - **POLLPRI** 高优先级数据可读
> - **POLLOUT** 普通或优先级带数据可写
> - **POLLWRNORM** 普通数据可写
> - **POLLWRBAND** 优先级带数据可写
> - **POLLERR** 发生错误
> - **POLLHUP** 发生挂起
> - **POLLVAL** 描述字不是一个打开的文件

- **nfds** 用来指定第一个参数数组元素个数。
- **timeout** 指定等待的毫秒数，无论 I/O 是否准备好，`poll()`都会返回。当等待时间为 0 时，`poll()`函数立即返回，为 -1 则使`poll()`一直阻塞直到一个指定事件发生。

同样的，我们实现一个`poll()`的示例程序，它同时检测 stdin 读和 stdout 写是否会发生阻塞：

```
#include <stdio.h>
#include <unistd.h>
#include <poll.h>

#define TIMEOUT 5 /** 定义超时时间 **/

int main(void)
{
    struct pollfd fds[2];
    int ret;

    /** 监听键盘输入 **/
    fds[0].fd = STDIN_FILENO;
    fds[0].events = POLLIN;

    /** 监听屏幕输出 **/
    fds[1].fd = STDOUT_FILENO;
    fds[1].events = POLLOUT;

    ret = poll(fds, 2, TIMEOUT * 1000);
    if (ret == -1) {
        perror("poll");
        return 1;
    }

    if (!ret) {
        printf("%d seconds elapsed.\n", TIMEOUT);
        return 1;
    }

    if (fds[0].revents & POLLIN)
        printf("stdin is readable\n");

    if (fds[1].revents & POLLOUT)
        printf("stdout is writable\n");

    return 1;
}
```

#### epoll

由于`poll()`和`select()`的局限，Linux 2.6 内核引入 event poll（epoll）机制。`epoll`的实现比`poll()`和`select()`复杂，但解决了前者存在的性能问题，并增加新的特性。

`poll()`和`select()`，每次调用需要所有被监听的文件描述符列表。内核必须遍历所有被监听的文件描述符。当列表变得很大时，遍历就成为了性能的瓶颈。

`epoll`把监听注册从实际监听中分离出来。一个系统调用会初始化`epoll`上下文，另一个从上下文中加入或删除监听的文件描述符，第三个执行真正的事件等待。

`epoll`操作需要以下三个接口：

```
#include <sys/epoll.h>

int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

`epoll_create1()`调用成功后，会创建一个 epoll 实例，并且返回和该实例关联的文件描述符。这个文件描述符将用于后续的 epoll 方法。

- **flags** 表示支持的 epoll 行为，目前只用 EPOLL_CLOEXEC 是合法值，表示进程被替换时关闭文件描述符。

当使用完成后，`epoll_create1()`创建的文件描述符需要通过`close()`调用来关闭。

```
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

`epoll_ctl()`向指定的 epoll 上下文中加入或者删除需要监听的文件描述符。

- **epfd** epoll 专用的文件描述符，epoll_create1()的返回值。
- **op** 表示操作 epoll 文件描述符的动作。

> - `EPOLL_CTL_ADD`表示注册新的文件描述符到 epoll。
> - `EPOLL_CTL_MOD`表示修改已经注册的文件描述符。
> - `EPOLL_CTL_DEL`表示删除已经注册的文件描述符。

- **fd** 需要监听的文件描述符。
- **event** 表示要监听的事件，结构体定义如下：

```
struct epoll_evnet {
	__u32 events;
	union {
		void *ptr;
		int fd;
		__u32 u32;
		__u64 u64;
	} data;
};
```
以下是合法的 events 值：

> - `EPOLLIN` 表示文件描述符可读。
> - `EPOLLOUT` 表示文件描述符可写。
> - `EPOLLPRI` 表示文件描述符有高优先级的带外数据可读。
> - `EPOLLERR` 表示文件描述符出错。
> - `EPOLLHUP` 表示文件描述符被挂起。
> - `EPOLLET` 表示在监听的文件描述符上开启边缘触发（edge-triggered）。默认的触发是条件触发（level-triggered）。
> - `EPOLLONESHOT` 表示在事件生成并处理后，文件描述符不会再被监听。 


```
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

调用`epoll_wait()`会等待指定 epoll 实例关联的文件描述符上的事件。

- **epfd** epoll 专用的文件描述符，epoll_create1()的返回值。
- **events** 分配好的事件结构体数组，当事件发生时，会将事件赋值到 events 中。
- **maxevents** 最大事件数量。
- **timeout** 超时时间，作用与`select()`和`poll()`的超时时间类似。

接下来，我们将用 epoll 改写`poll()`中的例子：

```
#include <stdio.h>
#include <unistd.h>
#include <sys/epoll.h>

#define TIMEOUT 5 /** 定义超时时间 **/
#define MAX_EVENTS 16 /** 定义最大事件数 **/

int main(void)
{
    struct epoll_event event1;
    struct epoll_event event2;
    struct epoll_event *wait_events;
    int ret, nr_events, epfd, i;

    epfd = epoll_create1(0);
    if (epfd < 0) {
        perror("epoll_create1");
    }

    /** 添加事件到监听列表中 **/
    event1.data.fd = STDIN_FILENO;
    event1.events = EPOLLIN;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &event1);
    if (ret) {
        perror("epoll_ctl on event1");
    }

    event2.data.fd = STDOUT_FILENO;
    event2.events = EPOLLOUT;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &event2);
    if (ret) {
        perror("epoll_ctl on event2");
    }

    /** 等待事件响应 **/
    wait_events = malloc(sizeof(struct epoll_event) * MAX_EVENTS);
    if (!wait_events) {
        perror("malloc");
    }

    nr_events = epoll_wait(epfd, wait_events, MAX_EVENTS, TIMEOUT * 1000);
    if (nr_events < 0) {
        perror("epoll_wait");
        free(wait_events);
        return 1;
    }

    for (i = 0; i < nr_events; i ++) {
        printf("event=%ld on fd=%d\n", wait_events[i].events, wait_events[i].data.fd);
    }

    free(wait_events);

    return 1;
}
```

