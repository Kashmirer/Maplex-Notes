1、Select：

int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout); 
void FD_ZERO(fd_set *fdset);	//清空一个fdset
void FD_SET(int fd, fd_set *fdset);	//添加一个fd进入fdset
void FD_CTR(int fd,fd_set *fdset);	//从fdset删除一个fd
int FD_ISSET(int fd, fd_set *fdset);	//判断 fdset里面是否有某个fd

select函数监听的文件描述符分3类，分别是writefds、readfds和exceptfds。调用后select函数会阻塞，知道有描述符就绪，或者超时，函数返回。当select函数返回后，可以通过遍历fdset来找到就绪的描述符。select目前在几乎所有平台上支持，其良好的跨平台支持也是它的一个优点。缺点就是单个进程能够监听的描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义来提升这一限制，但是会降低效率。

2、Poll：

int poll (struct pollfd *fds, unsigned int nfds, int timeout); 

不同于select使用3个位图来表示3个fdset的方式，poll使用一个pollfd的指针实现。

struct pollfd{
	int fd;
	short events;
   short revents;
}
pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递方式。同时，pollfd并没有最大数量限制（但是数量大了性能变差）。和select一样，poll返回后需要轮询pollfd来获取就绪的描述符。从上面的例子看，select和poll都需要在返回后同遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监听的描述符数量增长，其效率线性下降。

3、Epoll：
epoll是select和poll的增强版本。epoll更加灵活，没有数量限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放在内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。epoll操作过程需要3个接口，分别如下所示：

int epoll_create(int size);
int epoll_ctl(int epfd, int op, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
epoll_create()创建一个epoll句柄，size用来告诉内核这个监听的数目一共多大，这个参数无法限制epoll监听的最大数目，而是给出一个建议。 epoll句柄会占用一个fd值。
epoll_ctl()对指定的描述符fd执行op操作，可以添加（EPOLL_CTL_ADD）/删除（EPOLL_CTL_DEL）/修改（EPOLL_CTL_MOD）对fd的监听事件。event是告诉我们内核需要监听数目事件。struct epoll_event结构如下所示：

struct epoll_event {
	_uint32_t events;	/* Epoll events */
   epoll_data_t data;  /* User data variable */
};

//events可以表示为以下几个宏的集合
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里 
epoll_wait()等待epfd上的I/O事件，最多返回maxevents个事件。参数events用来从内核得到事件的集合，maxevents告知内核这个events有多大。
Epoll在内核维护了两个数据结构，一个红黑树用来存储监听IO事件的fd，另一个用来存储IO事件的双链表。红黑树能够提供高效的fd查找、修改、删除操作（epoll_ctl()）。一旦某个fd发生了IO事件，就会触发fd的回调函数，将事件添加到双链表中。而epoll_wait()只需要检查链表是否为空即可，不为空等待返回，否则等待IO事件发生触发回调函数。
