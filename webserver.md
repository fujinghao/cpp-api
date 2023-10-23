## 1.  I/O多路复用？
IO即为网络网络 I/O,多路即为多个tcp连接,复用即为共用一个线程或者进程,模型最大的优势是系统开销小,不必创建也不必维护过多的线程或者进程。
## 2.  select , poll 和 epoll的区别？
### 2.1 select
当client连接到server后，server会创建一个该连接的描述符fd，fd有三种状态，分别是读、写、异常，存放在三个fd_set（其实就是三个long型数组）中。select本质是通过设置和检查fd的三个fd_set来进行下一步操作。
<div align=center><img src="C:\Users\13683\vscodeProjrct\cpp\cpp笔记\fig\fd_set.png"height="265"/> </div>

**select大体执行步骤** ：

1. 使用copy_from_user从用户空间拷贝fd_set到内核空间(每次循环都需要要有用户切换内核的开销)
1. 内核遍历[0,nfds)范围内的每个fd,调用fd所对应的设备驱动poll函数,poll函数可以检测fd的可用流, (读流,写流,异常流),轮询方式O(n)从所有文件描述符查找已经就绪的文件描述符
1.  io方式返回后,用户要找到就绪描述符,需要遍历所有文件描述符O(n)

**select缺点：**

1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
2. 每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
3. select支持的文件描述符数量太小了，默认是1024
### 2.2 poll
poll大体执行和select差不多 poll使用pollfd结构类型的数组pollfd将文件描述符和事件类型放在一起,任何事件都可以统一处理,使得api接口简单,而select是3个fd_set,文件描述符并没有与事件类型绑定,仅仅是三个文件描述符集,所以他不能处理更多类型事件。他的最大文件描述符上限也达到了最高，其余和select一样。
### 2.3 epoll

epoll是Linux特有的IO复用函数，它在实现和使用上与select和poll有很大差异。epoll把用户关心的文件描述符上的事件放在内核上的一个事件表中，从而无须像select和poll那样每次调用都要重复传入文件描述符集合事件表。但epoll需要使用一个额外的文件描述符，来唯一标识内核中这个事件表。

**epoll 功能的实现分了三个系统调用完成：**

- epoll_create:创建内核事件表(epoll 池)用于存放描述符和关注的事件，主要数据结构有： struct eventpoll；其中有两个重要的成员 struct rb_root rbr;和 struct list_head rdllist;前者是一棵**红黑树**，也就是内核事件表，后者是就绪事件存放的队列(**双向链表**)。epoll 池只需要遍历这个就绪链表，就能给用户返回所有已经就绪的 fd 数组；
- epoll_ctl:为红黑树添加 ep_insert、移除 ep_remove、修改 ep_modify 节点的操作，每个节点就是一个描述符和事件的结构体，数据结构如：struct epitem;
- epoll_wait:负责收集就绪事件.当关注的描述符上有事件就绪,就会调用回调函数(ep_poll_callback)，把**对应 fd 的结构体**放入就绪队列中，从而把 epoll 从 epoll_wait处唤醒。

**epoll的优缺点：**

- 本身没有最大并发连接的限制，仅受系统中进程能打开的最大文件数目限制（65535);
- 采用回调方式来检测就绪事件，算法时间复杂度为O(1)。
- 如果在并发量低，但socket都比较活跃的情况下，select就不见得比epoll慢了（回调函数触发得过于频繁）。
- epoll的跨平台性不够用 只能工作在linux下。

## 3. epoll中水平触发（LT）和边沿触发（ET）的区别？

- recv的时候:
如果设置为LT，只要 接受缓冲 不为空，就会一直触发EPOLLIN，直到 接受缓冲 为空
如果设置为ET，只要 客户端 发送一次数据，就会触发一次EPOLLIN
- send的时候:
如果设置为LT，只要发送缓冲不满，就会一直触发EPOLLOUT
如果设置为ET，有注册EPOLLOUT事件，才会一次触发一次EPOLLOUT

ET模式 效率要比 LT模式高, 小数据使用边沿触发，大数据使用水平触发
比如listenfd，接受缓冲区 可能存放多个客户端连接请求的信息，这时候要使用水平触发（LT），因为accept每次只能处理一个，需要多次触发。如果用边沿触发（ET）可能会漏掉一些连接。

**为什么ET模式下一定要设置非阻塞？**

在ET模式下，一般会设置非阻塞io, 也就是说，在当前没有可读取数据的情况下:
- 如果是阻塞io，recv()会阻塞
- 如果是非阻塞io，recv()会返回-1
因为ET模式下是无限循环读，直到出现错误为EAGAIN或者EWOULDBLOCK，这两个错误表示socket为空，不用再读了，然后就停止循环了，如果不是非阻塞，循环读在socket为空的时候就会阻塞到那里.