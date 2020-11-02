### select/poll/epoll

这三个都是 I/O 多路复用的具体实现，select 出现的最早，之后是 poll，再是 epoll。

#### 1. select

```c
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

- fd_set 表示描述符集合；
- readset、writeset 和 exceptset 这三个参数指定让操作系统内核测试读、写和异常条件的描述符；
- timeout 参数告知内核等待所指定描述符中的任何一个就绪可花多少时间；
- 成功调用返回结果大于 0；出错返回结果为 -1；超时返回结果为 0。

返回结果中内核并没有声明哪些 fd_set 已经准备好了，所以如果返回值大于 0 时，程序需要遍历所有的 fd_set 判断哪个 I/O 已经准备好。

在 Linux 中 select 最多支持 1024 个 fd_set 同时轮询，其中 1024 由 Linux 内核的 FD_SETSIZE 决定。如果需要打破该限制可以修改 FD_SETSIZE，然后重新编译内核。

### 2. poll


### 3. epoll

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```


### select 和 poll 比较

#### 1. 功能

它们提供了几乎相同的功能，但是在一些细节上有所不同：

- select 会修改 fd_set 参数，而 poll 不会；
- select 默认只能监听 1024 个描述符，如果要监听更多的话，需要修改 FD_SETSIZE 之后重新编译；
- poll 提供了更多的事件类型。

#### 2. 速度

poll 和 select 在速度上都很慢。

- 它们都采取轮询的方式来找到 I/O 完成的描述符，如果描述符很多，那么速度就会很慢；
- select 只使用每个描述符的 3 位，而 poll 通常需要使用 64 位，因此 poll 需要复制更多的内核空间。

### 3. 可移植性

几乎所有的系统都支持 select，但是只有比较新的系统支持 poll。

### eopll 工作模式

epoll_event 有两种触发模式：LT（level trigger）和 ET（edge trigger）。

#### 1. LT 模式

当 epoll_wait() 检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用 epoll_wait() 时，会再次响应应用程序并通知此事件。是默认的一种模式，并且同时支持 Blocking 和 No-Blocking。

#### 2. ET 模式

当 epoll_wait() 检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用 epoll_wait() 时，不会再次响应应用程序并通知此事件。很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。只支持 No-Blocking，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

### select poll epoll 应用场景

很容易产生一种错觉认为只要用 epoll 就可以了，select poll 都是历史遗留问题，并没有什么应用场景，其实并不是这样的。

#### 1. select 应用场景

select() poll() epoll_wait() 都有一个 timeout 参数，在 select() 中 timeout 的精确度为 1ns，而 poll() 和 epoll_wait() 中则为 1ms。所以 select 更加适用于实时要求更高的场景，比如核反应堆的控制。

select 历史更加悠久，它的可移植性更好，几乎被所有主流平台所支持。

#### 2. poll 应用场景

poll 没有最大描述符数量的限制，如果平台支持应该采用 poll 且对实时性要求并不是十分严格，而不是 select。

需要同时监控小于 1000 个描述符。那么也没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

需要监控的描述符状态变化多，而且都是非常短暂的。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。epoll 的描述符存储在内核，不容易调试。

### 3. epoll 应用场景

程序只需要运行在 Linux 平台上，有非常大量的描述符需要同时轮询，而且这些连接最好是长连接。

### 4. 性能对比

> [epoll Scalability Web Page](http://lse.sourceforge.net/epoll/index.html)
