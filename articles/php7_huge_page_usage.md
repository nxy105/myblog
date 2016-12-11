# [译] PHP7 Huge Page 使用

## 那些内存分页的回忆

内存分页是操作系统管理用户态进程的内存的一种手段。每个进程访问的内存都是虚拟的，操作系统和 MMU 必须将其转换为在主存（RAM）中真实的物理地址。

分页内存将内存分成了各个固定大小的块，每块叫做页。Linux 系统中通常每页的大小为 4KB。每个进程中都有一张内存页地址的转换表，这张转换表叫做页表。为了避免每次访问内存都要读取这张转换表（导致每次读取内存的数据都需要两次操作），硬件会使用一些缓存保存转换表：称之为后备缓冲器（TLB）。存储到 MMU 中并且一旦在TLB 中找到转换，就非常快速和高效。没有命中时，就为记为 TLB 未命中，并且需要访问缓慢的内存来更新 TLB。

## 为什么使用 Huge Page？

概念很简单。如果我们让操作系统内核使用更大的页，意味着可以通过单页访问到更多的数据。同样意味着一旦转换表存储到 TLB 中，TLB 的命中率也相应提高。Linux 内核在 X86/64 架构上过去使用默认的 4KB 大小的页，但到了 2.6.20 版本时，内核引入了 *huge page* 的概念。一个 *huge page* 通常是传统页大小的512倍：大小为 20MB。你可以阅读 Linux 内核的相关资料，从中了解到内核让 huge page 映射关系透明化了：内核将虚拟内存映射到 4KB 的传统页上，但尝试将相邻的页合并成 *huge page*。尤其针对进程的堆段，表示一段非常巨大的地址空间，但是需要小心的是，这部分内存可能会被分成小块返回给操作系统（释放），这样浪费了大量的页空间，可能导致内核撤销当前的任务，并且将一个 huge page 缩小成 512 个 4KB 的页。

然而，用户进程需要可以根据自身需求使用 huge page。如果你确认能够填充满 huge page 的空间，那么请求内核使用 huge page 是更好的决定。huge page 意味着更少的页管理，意味着内核扫描的列表更小，意味着 更低频地使用 TLB 缓存。系统将会更有效率，更快速。

## PHP7 OPCache 实现

在 PHP7，我们致力于让内存的使用更高效。我们重制了 PHP 内部的结构，让 CPU 缓存的使用更加高效。我们致力于数据局部化，这意味着更多相邻的数据可以从 CPU 的缓存中获取，而不是访问主存。这是 PHP7 性能提升的基本设计思想，这些恰恰是C语言比C++能容易实现的。

综上所述，我们将优化内存使用的思路放到内核 huge page 的使用上，OPCache 扩展添加了一些面向 huge page 的特性。

## 请求 huge page

由于 UNIX 版本的多样性，你可以使用我们提供的两个 API 来管理虚拟内存映射。`mmap()` 方法（推荐，因为这个函数能够真正分配我们所需的 huge page），或者 `madivse()` （该函数只是提示内核当前页需要转换成 huge page，而不是真正转换）。首先，你需要确认操作系统是否支持 huge page，如果不支持，分配 huge page 将会失败。使用 sysctl 调整 *vm.nr_hugepages* 值。然后 cat /proc/meminfo 确认 huge page 是否可用。

```
> cat /proc/meminfo
HugePages_Total:      20
HugePages_Free:       20
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

以上，20个 huge pages 是可用状态，每个 huge page 是 2MB（在 X86/64 平台上，Linux可用将 huge page 设置到 1GB，我不推荐 PHP 使用这样的大小，关系数据库系统将更适合这样的大小）。

然后我们使用 API，当然，你不能映射一段边界与 huge page 无法对齐的内存段到 huge page 上，所以，首先，你必须对其你的地址做对齐操作，然后请求内核通过 huge page 映射。我们使用C语言做对齐作为示例，需要对齐的缓冲来自于堆，虽然已经有现成的对齐函数，但为了跨平台兼容性，我们不会使用。

以下是简单的示例：

```
#include 
#include 
#include 
#include 

#define ALIGN 1024*1024*2 /* We assume huge pages are 2Mb */
#define SIZE 1024*1024*32 /* Let's allocate 32Mb */

int main(int argc, char *argv[])
{
    void *addr;
    void *buf = NULL;
    void *aligned_buf;

    /* As we're gonna align on 2Mb, we need to allocate 34Mb if
    we want to be sure we can use huge pages on 32Mb total */
    buf = malloc(SIZE + ALIGN);

    if (!buf) {
        perror("Could not allocate memory");
        exit(1);
    }

    printf("buf is at: %p\n", buf);

    /* Align on ALIGN boundary */
    aligned_buf = (void *) ( ((unsigned long)buf + ALIGN - 1) & ~(ALIGN -1) );

    printf("aligned buf: %p\n", aligned_buf);

    /* Turn the address to huge page backed address, using MAP_HUGETLB */
    addr = mmap(aligned_buf, SIZE, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE | MAP_HUGETLB | MAP_FIXED, -1, 0);

    if (addr == MAP_FAILED) {
        printf("failed mapping, check address or huge page support\n");
        exit(0);
    }

    printf("mmapped: %p with huge page usage\n", addr);

    return 0;
}
```

如果你是C语言开发者，上面的代码不需要特别说明。内存不会显示的释放，因为程序很快就结束，这里只是向你阐述概念。

当这段程序分配了内存并且即将退出时，我们查看当时的内存信息，我们可以看到内核中被申请使用的 huge page：

```
HugePages_Total:      20
HugePages_Free:       20
HugePages_Rsvd:       16
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

16页被标记为存储，并且 16 * 2 MB 的内存段通过 `mmap()` 分配。

当目前为止都很不错，这个 API 正如我们预期的运行。

## PHP7 代码与 huge page

如果你看到为 PHP7 代码分配的内存段，你可以看到其占用的内存相当巨大。在我的 X86/64 传统机器上，大约有 9MB：

```
> cat /proc/8435/maps
00400000-00db8000 r-xp 00000000 08:01 4196579                            /home/julien.pauli/php70/nzts/bin/php /* text segment */
00fb8000-01056000 rw-p 009b8000 08:01 4196579                            /home/julien.pauli/php70/nzts/bin/php
01056000-01073000 rw-p 00000000 00:00 0 
02bd0000-02ce8000 rw-p 00000000 00:00 0                                  [heap]
... ... ...
```

从虚拟内存段 *00400000* 到 *00db8000*，有 9MB 大小。这意味着 PHP 的二进制指令总共占用 9MB 内存。是的，PHP 的开销越来越大。它加入了越来越多的特性，需要更多的C代码，从而转换成更多的 CPU 指令。

如果，我们继续往下看内存信息，我们会发现内核使用了传统的 4KB 页做内存映射：

```
> cat /proc/8435/smaps
00400000-00db8000 r-xp 00000000 08:01 4196579  /home/julien.pauli/php70/nzts/bin/php
Size:               9952 kB /* VM size */
Rss:                1276 kB /* PM busy load */
Pss:                1276 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:      1276 kB
Private_Dirty:         0 kB
Referenced:         1276 kB
Anonymous:             0 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB /* page size is 4Kb */
MMUPageSize:           4 kB
Locked:                0 kB
```

如你所见，内核并没有使用 huge page 映射这段内存。也许内核会在某个时刻使用 huge page？我并不清楚，因为我不是内核开发者，并且不想深入到内核的源码进行分析。

我们现在能做的，是使用 huge page 重新映射。这就是 OPCache 做的事情。

我们之所以这么做，是因为代码段不会随着进程的退出了增加或减少。如果会变化，那 huge page 就显得不够好了。但我们知道，这段内存不会移动，并且 9952KB 大小的代码将会使用 4 个 huge page，并且将剩余的内存使用 4KB 的页。


