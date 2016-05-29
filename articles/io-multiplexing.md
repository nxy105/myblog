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

#### seletc()

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

### 结语

未完待续……
