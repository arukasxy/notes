---
title: 网络通信
slug: telecommunication-ot0ya
url: /post/telecommunication-ot0ya.html
date: '2024-07-09 10:21:15+08:00'
lastmod: '2024-08-08 13:08:51+08:00'
toc: true
tags:
  - 网络
categories:
  - 网络
keywords: 网络
isCJKLanguage: true
---

# 网络通信

# TCP/IP模型(+OSI模型 + 常用协议)

|OSI模型|TCP/IP模型|功能|数据封装|常用协议（设备）|
| ------------| ----------------------------------------------| --------------| ----------------------------------------| ----------------------------------------------------------------------------|
|应用层|应用层<br />||数据message|HTTPS、HTTP、Telenet、FTP、TFTP、DNS、SMTP、SSH|
|表示层|||||
|会话层|||||
|传输层|传输层|决定传输方式|报文段<sup>（UDP datagram/TCP segment）</sup>|TCP、UDP|
|网络层|网络层|决定传输路径|数据包<sup>（packet/datagram）</sup>、分组<br />|路由器、IP、ICMP、RIP、IGMP、OSPF、BGP|
|数据链路层|数据链路层|建立物理连接|帧frame|网桥、交换机、以太网、令牌环、PPP、PPTP、L2TP、ARP、ATMP、ARQ|
|物理层|物理层||比特流|物理线路、光纤、无线电、网线、集线器|

# IO多路复用

其他IO模型参考IO模型

## select

* 优点：兼容性好
* 缺点：

  * 每次调用select函数时向OS传递监视对象信息
  * 调用select函数后对所有文件描述符进行循环
  * 监听的文件描述符数量 < FD\_SIZE（1024）
* 场景：服务器端接入者少；程序应具有兼容性

```C++
int select(int maxfd, fd_set *readset , fd_set *exceptset, const struct timeval *timeout);
# maxfd 监视描述符数量，设置为最大的文件描述符值+1
# readset 将所有关注“是否存在待读取数据”的文件描述符注册到fd_set型变量
# writeset 将所有关注“是否可传输无阻塞数据”的文件描述符注册到fd_set型变量
# exceptset 将所有关注“是否发生异常”的文件描述符注册到fd_set型变量
# timeout为以1/1000秒为单位的等待时间，传递-1表示等待到事件发生，传递0表示执行一次非阻塞式的检查
# 返回：发生错误为-1，超时返回0，事件返回描述符数量

struct timeval
{
    long tv_sec;  // seconds
    long tv_usec;  // mircroseconds
}
```

```C++
#include <sys/select.h>
#include <sys/time.h>

# 1. 设置文件描述符
fd_set reads, cpy_reads;  
FD_ZERO(&set); // [0,0,0...]
FD_SET(listenfd, &set);
fd_max = listenfd;

# 2. 设置监视范围
while(true) // 此时进入处理循环
# 将初始值复制到temps,以temps使用select(因为select将除发生变化的文件描述符对应位外,其余位初始化为0)
cpy_reads = reads; 

# 3. 设置超时
# 当监视的文件描述符发生变化时/超时,select才返回。不设置超时,可将timeout设置为NULL
# 每次调用select前都需要设置超时(因为select会将timeval替换为超时前剩余时间)
struct timeval timeout;
timeout.tv_sec = 5;
timeout.tv_usec = 0;

# 4. 调用select函数
int result = select(fd_max+1, &cpy_reads, 0, 0, &timeout);

# 5. 查看调用结果
if(result == -1) {  
    # 错误处理
}
else if(result == 0) {
    # 超时处理 
}
else { 
    for(int i=0; i<fd_max+1; i++)  // 遍历文件描述符
    {
        if(FD_ISSET(i, &cpy_reads)) {  // 判断对应文件描述符i是否变化
            if(i == listenfd){
                # connfd = accept接收请求
                FD_SET(connfd, &reads);
                if(fd_max < connfd) fd_max = connfd;
            }else{ // 其他connfd有数据接收
                # 数据读取和处理
                str_len = read(i, buf, BUF_SIZE);
                if(str_len == 0){ // 接收的数据为EOF，需要断开连接
                    FD_CLR(i, &reads); // 断开连接
                    close(i);
                }else{ ... }
            }
        } 
    }
}
```

## poll

* 优点：相比于select，无最大文件描述符数量限制
* 缺点：等同于select

```C++
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
# timeout为以1/1000秒为单位的等待时间，传递-1表示等待到事件发生，传递0表示执行一次非阻塞式的检查

struct pollfd{
	int fd;			//文件描述符，如果暂时不关心任何事件，设为-1（忽略标准输入）
	short events;	//等待的事件，参考epoll
	short revents;	//实际发生的事件，初始化为0
};
```

```C++
typedef std::vector<struct pollfd> PollFdList;
PollFdList pollfds_;
int numEvents = ::poll(&*pollfds_.begin(), pollfds_.size(), timeoutMs);
for(auto iter = pollfds_.begin(); iter != pollfds_.end() && numEvents > 0; iter++){
        if(iter->revents > 0){
            --numEvents;
            // iter->fd is the file descriptor，处理参考epoll
        }
}
```

## epoll（IOCP）

* 优点：

  仅向OS传递一次监视对象

  监视内容发生变化时只通知发生变化的事项
* 缺点：需要OS支持，且支持程度和方式存在差异

```C++
struct epoll_event{
	__uint32_t events;
	epoll_data_t data;
}
typedef union epoll_data{
	void *ptr;
	int fd;
	__uint32_t u32;
	__uint64_t u64;
}epoll_data_t;

# 返回epoll文件描述符epfd; size为epoll例程的大小(仅供OS参考)
int epoll_create(int size);

# 返回发生事件的文件描述符数
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
# events保存发生事件的文件描述符集合
# maxevents 为可用保存的最大事件数
# timeout为以1/1000秒为单位的等待时间，传递-1表示等待到事件发生，传递0表示执行一次非阻塞式的检查

# 对文件描述符fd进行op操作
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
# op常量
EPOLL_CTL_ADD 将文件描述符注册到epoll例程
EPOLL_CTL_DEL 从epoll例程中删除文件描述符，event传入NULL
EPOLL_STL_MOD 更改注册的文件描述符的关注事件发生情况
# event常量（可用|同时设置）
EPOLLIN 需要读取数据的情况
EPOLLOUT 输出缓冲为空，可以立即发送数据的情况
EPOLLPRI 收到OOB数据
EPOLLRDHUP 断开连接或半关闭的情况
EPOLLERR 发生错误的情况
EPOLLET 以边缘触发的方式得到事件通知
EPOLLONESHOT 发生一次事件后，相应文件描述符不再收到事件通知，需要使用op=EPOLL_STL_MOD再次设置事件
EPOLLNVAL fd未打开
```

```C++
#include <sys/epoll.h>
#define EPOLL_SIZE 50  // 足够大，Linux2.6.8后忽略size参数
struct epoll_event *ep_events;
struct epoll_event event;
int epfd, eventcnt;

epfd = epoll_create(EPOLL_SIZE);
ep_events = malloc(sizeof(struct epoll_event)*EPOLL_SIZE);

event.events = EPOLLIN;
event.data.fd = listenfd; //listen(sockfd, ...)
epoll_stl(epfd, EPOLL_CTL_ADD, listenfd, &event);

# 以下可循环
event_cnt = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
for(int i=0; i<event_cnt; i++){
	if(ep_events[i].data.fd == listenfd){ // 连接事件
		# connfd = accept(...) 
		event.events = EPOLLIN;
		event.data.fd = connfd;
		epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &event);
	}else if(ep_events[i].events & EPOLLIN){ // 数据传输事件
		str_len = read(ep_events[i].data.fd, buf, BUF_SIZE);
		if(str_len == 0){ // close请求
			epoll_ctl(epfd, EPOLL_DEL, ep_events[i].data.fd, NULL);
			close(ep_events[i].data.fd);
		}else{
			write(ep_events[i].data.fd, buf, str_len);
		}
	}
}
close(sockfd);
close(epfd);
```

### 条件触发（+边缘触发）

# Socket API

|API|作用|使用|
| ------------------------------------------------------------------------------| --------------------------| --------------------------------------------------------------------------------------------------------------------------------|
|int socket(int family<sup>(IPV4为AF_INET; IPV6为AF_INET6; 本地通信为AF_LOCAL)</sup>, int type<sup>(UDP为SOCK_DGRAM; TCP为SOCK_STREAM)</sup>, int protocol<sup>(0,除非同一协议族中存在多个数据传输方式相同的协议)</sup>);|创建socket|listenfd = **socket**(AF_INET, SOCK_STREAM, 0);|
|int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);|将地址绑定到socket上|**bind**(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));|
|int listen(int sockfd, int backlog<sup>(最多允许连接数，可设置为SOMAXCONN(listen队列最大长度))</sup>);|监听socket|**listen**(listenfd, 20);|
|int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);|申请建立连接|**connect**(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));|
|int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);|接收连接，返回连接socket|socklen_t addrlen = static_cast<socklen_t>(sizeof cliaddr);<br />connfd = **accept**(listenfd, (struct sockaddr *)&cliaddr, &cliaddr_len);<br />|
|int shutdown(int sock, int howto<sup>(SHUT_RD断开输入流（无法接收）、SHUT_WR断开输出流（无法发送）、SHUT_RDWWR同时断开I/O流)</sup>);|断开单条通道|**shutdown**(sockfd, SHUT_WR)|
|int close(int fd);|关闭socket|**close**(sockfd)|

## socket地址

## 读写

可以使用文件IO函数。

|API|功能|使用|
| ---------------------------------------------------------------------------------------------------------------------| -------------| ---------------------------------------------------------------------------------|
|ssize_t sendto(int fd, const void *buf, size_t len, int flags,const struct sockaddr *dest_addr, socklen_t addrlen);|UDP发送数据|**sendto**(sendfd, buf, strlen(buf), 0, (struct sockaddr *)&recv_addr, sizeof(recv_addr))|
|ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);|UDP接收数据|**recvfrom**(recvfd, buf, MAXLINE, 0, (struct sockaddr *)&send_addr, &send_addr_len)|
|int s<sup>(对端关闭连接或发送0字节数据，send才会返回0)</sup>​en<sup>(对端关闭连接或本端发送0字节数据，send才会返回0)</sup>​d<sup>(对端关闭连接或发送0字节数据，send才会返回0)</sup>(int s, const void *msg, size_t len, int flags);|TCP发送数据|**send**(sendfd, buf, strlen(buf))|
|int recv<sup>(只有在对端关闭连接时才会返回 0，如果对端发送0字节数据，recv不会收到)</sup>(int sockfd,void *buf,int len,int flags);|TCP接收数据|**recv**(recvfd, buf, MAXLINE)|

‍

# 应用层

## HTTP（+RPC）

||HTTP|RPC|
| ----------| :-----------------------------------------: | --------------------------|
||超文本传输协议|远程过程调用|
|作用|定义解析规则<br />||
|服务发现<sup>（找到服务对应的IP和端口）</sup>|DNS|中间服务<sup>（consul/etcd/CoreDNS）</sup>|
|底层连接|长连接keep alive|长连接 + 连接池|
|传输内容|JSON序列化<sup>（消息头header + 消息体body）</sup>|protobuf等其他序列化协议|
|性能||更好|

### 请求报文（+请求方法）

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240717125557-dlb99ci.png)​

> **请求方法**：
>
> * GET（获取资源）
> * POST（上传数据）
> * DELETE（删除资源）
> * HEAD（获取资源元信息）
> * OPTIONS（列出可对资源实行的方法）
> * PUT（类似于POST，替换整个资源）
> * PATCH（替换部分资源）
> * TRACE（追踪请求 - 响应的传输路径）
> * CONNECT（建立特殊的连接隧道）

||GET|POST|
| ----------| ------------------------------| ------------------------------------------------------------|
|参数位置|URL|报文体|
|数据长度|URL最大长度<sup>（2KB（2048个字符））</sup>|无长度消毒|
|缓存<sup>（GET通常进行查询操作，POST通常将数据发送给服务器，修改和更新数据）</sup>|浏览器有缓存|无缓存|
|数据类型|ASCII字符|更多字符类型|
|安全<sup>（只读，请求方法不会「破坏」服务器上的资源）</sup>|安全|不安全|
|幂等<sup>（多次执行相同的操作，结果都是「相同」的）</sup>|幂等|不幂等|
|安全||安全性高，不会被缓存，<br />不会保存在服务器日志和浏览记录中<br />|
|功能|搜索、排序和筛选<br />等查询操作|修改和写入数据|

### 响应报文（+状态码）

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240717125655-wykyz61.png)​

|状态码|含义|
| --------| ----------------------------------------------------------------------------------------------|
|1xx|请求已经收到了，正在处理|
|2xx|处理成功|
|206|Partial Content客户端表明自己只需要目标URL上的部分资源的时候返回的，可实现断点续传、带宽遏流|
|3xx|重定向到其它地方<br />|
|302|暂时性转移Temporarily Moved|
|4xx<br />|处理发生错误，责任在客户端<br />|
|5xx|处理发生错误，责任在服务端|

### 强缓存和协商缓存

加载资源时，先检查**强缓存**是否命中，如果资源过期，进行**协商缓存**，确定资源是否有效

||强缓存|协商缓存|
| ------| ------------------------------------------| ---------------------------------------------------------------|
||缓存请求过的资源|向服务器确认缓存资源是否有效|
|功能|节省带宽<br />提高访问速度<br />降低服务器压力<br />|确保获取到最新资源|
|场景|静态资源<sup>（图片、CSS文件等）</sup>|频繁更新的资源|
|实现|Expires过期时间<sup>（HTTP1.0）</sup><br />Cache-Control缓存有效时间<sup>（HTTP1.1,提供更灵活的缓存控制机制）</sup>|ETag资源唯一标识符<br />Last-Modified资源最后修改时间<br />返回状态码|

### HTTP版本

||HTTP1.0|HTTP1.1|HTTP2.0|HTTP3.0|
| ------------| ------------------------------------------------------------------------------------------------------------| -----------------------------| ----------------------------| -------------------------------|
|本质|<br />||基于HTTPS|基于**UDP**的QUIC可靠协议|
|长连接<sup>（TCP 三次握手时会有 1.5 RTT 的延迟，以及建立连接后慢启动（slow-start），重复建立连接浪费时间和带宽）</sup>|默认短连接|默认keep-alive<br />|||
|请求的管道处理<sup>（多路复用，即一个TCP连接上可以传送多个HTTP请求和响应）</sup>|×|队头请求阻塞后，请求队列均阻塞<sup>（请求和响应之间没有序号标识）</sup>|用流和<sup>（请求和响应消息被拆分为帧，为每个帧分配序号，独立传输）</sup>​分帧解决了<sup>（请求对应一个流，被拆分为帧，为每个帧分配序号，独立传输）</sup>​HTTP的队头阻塞<sup>（请求和响应消息被拆分为帧，为每个帧分配序号，独立传输）</sup><br />存在TCP队头阻塞<sup>（TCP分节丢失后，后序分节需保存直到重传完成）</sup><br />|解决了TCP的队头阻塞<sup>（放弃了TCP）</sup>|
|Header压缩|||HPAK算法<sup>（在客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，就不用重复发送同样字段了，只发送索引号，减少数据量提高速度）</sup>||
|二进制传输|纯文本<br />||二进制||
|主动通知|||服务器主动向客户端发送数据||
|连接时间|||TCP三次握手 + TLS握手|不需要握手|
|重传策略|初始包和重传包使用相同的sequence number，必须有序确认，加大之后重传计算的耗时<br />|||包使用unique packet number<sup>(唯一且单调递增)</sup>，支持乱序确认|
|连接迁移|四元组（源IP、源端口、目的IP、目的端口)任一发生变化，需重新建立连接<br />|||使用Connection ID维持原有连接|
||||||

**HTTPS**：

* 添加**SSL/TLS**安全协议，报文可以加密传输
* 除TCP三次握手外，需进行SSL/TLS握手
* 端口号为443（HTTP为80）
* HTTPS协议需要向CA(证书权威机构)申请数字证书，来保证服务器的身份是可信的

### Cookie与Session

||Cookie|Session|
| ---------------------| -------------------------------------------------------| ----------------------------------|
|实现|cookie存储和发送session id||
|功能|跟踪用户身份<br />||
|位置|浏览器|服务器|
|安全|可以伪造|不能伪造<sup>（因为在服务器上）</sup>|

## CDN内容分发网络

特点：

* 用户就近访问服务器，提升响应速度

# 传输层

||UDP|TCP|
| ------------------| -----------------------------------------------------------| --------------------------------------------------|
||面向消息的不可靠协议|面向字节流的可靠协议<br />|
|端口号重复|TCP和UDP套接字可以共用端口号<br />||
|连接|无连接|建立连接|
|使用|SOCK_DGRAM|SOCK_STREAM|
|大小|512字节以内||
|报文|仅一个UDP报文|将大的报文按**序列号**拆分与合并|
|传播|支持单播、多播、广播|仅支持单播|
|头部|仅8字节|最小20字节，最大60字节|
|数据边界|数据有边界<sup>（接收数据的次数应和传输次数相同）</sup>|数据无边界|
|保持连接|每次需要添加目标地址|keep-alive|
|功能<br />（可靠性）|无|传递给应用层的数据按序交付<sup>（序列号）</sup>、不丢失<sup>（确认号、超时重传）</sup>、不重复<br />流量控制、拥塞控制|
|实现|无发送缓冲区、数据包无序号||
|场景|性能，如实时视频|可靠性，如传输文件|
|协议|DNS、TFTP、SNMP、NTP|HTTP、FTP、SMTP、TELNET、SSH|

## UDP

### 实现可靠传输

参考QUIC、KCP、UDT、RTP、RUDP

要求：数据不丢失、数据有序、流量控制<sup>（控制向应用层交付数据的速率, 避免应用层来不及消费数据而丢包）</sup>、拥塞控制

1. 自动重传请求：发送缓冲区 + 超时时间

    **发送缓冲区<sup>（原UDP没有发送缓冲区）</sup>** 中暂存已经发送但没有收到确认的数据包，当超过指定时间后，自动重传该数据包
2. 数据包添加序号
3. 流量控制和拥塞控制参考TCP

## TCP

​![TCP](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/TCP-20240718110304-kwr7q9h.jpg)​

### 报文解析

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240718094104-9hhevqp.png)​

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240718094132-8j1jkb8.png)​

|标志位|含义|
| --------| ---------------------------------------------|
|URG|紧急指针|
|ACK|确认序号是否有效，置1|
|PSH|提示接收端应用程序立即从TCP缓冲区把数据读走|
|RST|重新建立连接|
|SYN|请求建立连接，置1|
|FIN|断开连接|

#### TIME_WAIT

* 主动关闭的一方，即先发送FIN消息，进入TIME_WAIT状态
* **持续时间**：2MSL（最大报文段生存时间）
* **原理**：socket对应的四元组（源目IP、源目端口）处于冻结状态，不会释放端口
* **作用**：

  * 避免第四次挥手报文丢失造成**连接关闭异常**​：处于TIME_WAIT状态，则可以重新发送ACK，之后重新计时2MSL时间才会进入CLOSED状态。如果没有TIME_WAIT状态，会响应RST，导致被动关闭端异常关闭TCP连接。
  * 避免乱序到来的报文在**新的socket连接**中引发混乱：假设在关闭前有TCP报文由于中间网络传输原因导致在CLOSED完成之后才到达，如果没有TIME\_WAIT状态而A和B又使用同样的4元组新建了一个新的socket，那么迷路的数据包就会进入到新的socket中进行处理，可能导致业务异常。
* 大量的TIME_WAIT：

  * 原因：高并发且持续的短连接请求，例如爬虫服务器、HTTP1.0
  * 后果：端口被占用2MSL时间，占用大量CPU、内存、文件描述符、端口数量，导致新的连接无法建立
  * 措施：

    * socket选项设置SO\_LINGER / SO\_REUSEADDR
    * 业务代码：将短连接变为长连接，如HTTP2.0

### 流量控制

* 功能：控制发送端的发送速度，使其按照接收端的数据处理速度来发送数据，避免接收端处理不过来，产生网络拥塞或丢包。
* 实现：使用可变大小的**滑动窗口**来交换双方**缓冲区空闲空间**​信息
* 可发送数据大小 \= min(拥塞窗口， 接收窗口) - 未确认数据大小

#### 零窗口

* 接收端处理数据太慢，接收窗口为0

如何通知发送端可用？

	使用**ZWP（Zero Window Probe，零窗口探针）** 技术，发送端收到零窗口的应答后，启动计时器，每隔一段时间问接收端。若持续返回零窗口，可能会发送RST断开连接。

#### Nagle算法

* 原理：**累计**小数据，直到包长度大于MSS<sup>(Max Segment Size，TCP 报文段一次可传输的最大分段大小)</sup> 或者 收到前一数据的ACK，才发送下一数据
* 优点：避免数据报过多而发生网络过载；减少网络流量，提高网络传输效率
* 缺点：速度慢
* 适用场景：小数据块
* 不适用场景：传输大文件数据

### 拥塞控制

* 功能：控制发送端的发送速度，避免网络拥塞
* 实现：维护**拥塞窗口**​
* 可发送数据大小 \= min(拥塞窗口， 接收窗口) - 未确认数据大小

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240803233035-shay5nw.png)​

#### 慢开始

1. 连接建立时，初始化 cwnd（拥塞窗口大小）\= 1，表示可以传一个 MSS 大小的数据
2. 每收到一个 ACK 包，cwnd++
3. 每经过一个 RTT（往返时延），cwnd 会翻倍（指数增长）
4. 当 cwnd \>\= ssthresh (slow start threshold) 时，进入拥塞避免阶段

#### 拥塞避免

1. 每收到一个 ACK 包，cwnd \= cwnd + 1/cwnd
2. 每经过一个 RTT，cwnd \= cwnd + 1（加法增大）
3. 如果触发超时重传或快速重传，则网络出现了拥塞

#### 超时重传

1. 把 sshthresh 设为当前拥塞窗口的一半（乘法减小）
2. cwnd 重置为 1，重新开始慢启动过程

#### 快速重传（快速恢复）

**触发时机**：接收端收到乱序包时，会发送 duplicate ACK 通知发送端。当发送端收到 3 个 duplicate ACK 时，就立刻开始重传，而不必继续等待到计时器超时。

**原理**：如果网络出现拥塞，是不会收到多个重复的 ACK 的，所以现在网络可能没有出现拥塞。

1. 把 sshthresh 设为当前拥塞窗口的一半（乘法减小）
2. cwnd 重置为 sshthresh，重新开始拥塞避免过程

### 粘包

* 定义：连续给对端发送两个或者两个以上的数据包，对端在一次收取中收到的数据包数量可能大于 1 个，当大于 1 个时，可能是几个（包括一个）包加上某个包的部分，或者干脆就是几个完整的包在一起。
* 解决：区分包与包之间的边界

  * 固定包长的数据包：灵活性差，太少要填充，太长要分包
  * 以指定字符（串）为包的结束标志：可能需要转义
  * 包头 + 包体格式

    包头是固定大小的，且包头中必须含有一个字段来说明接下来的包体有多大。

    对端先收取包头大小字节数目（如果不够还是先缓存起来，直到收够为止），然后解析包头，根据包头中指定的包体大小来收取包体，等包体收够了，就组装成一个完整的包来处理。

    ​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240804014525-9s6466d.png)​

# 网络层

## IPv4和IPv6

||IPv4|IPv6|
| ------| ------| -------|
|大小|32位|128位|
||||

## NAT协议

网络地址转换协议

* 目的：解决IPV4地址资源短缺
* 原理：数据包发送时将局域网IP转为公网IP；数据包接收是将公网IP转为局域网IP
* 作用：局域网中的设备共享一个公网IP

# 数据链路层

## ARP协议

地址解析协议

* 作用：IP地址 -> MAC地址
* 主机 A 发送 IP 数据报给主机 B 途中经过了 5 个路由器。试问在 IP 数据报的发送过程中总共使用了几次 ARP？

  在发送过程中，需要依次获得5个路由器的MAC地址和主机B的MAC地址。所以一共要获得6个MAC地址，即要使用6次ARP协议。

# 流程解读

## ping

1. 获取IP地址：

    * 域名缓存
    * 向DNS服务器发送DNS请求报文<sup>（发送流程和第2步相同，目的IP为DNS服务器地址，目的MAC为网关MAC）</sup>（通过静态配置或DHCP学习到DNS服务器地址）
2. 发送ICMP请求报文

    * 构造ICMP请求报文<sup>（目的IP是ping的主机，目的MAC为网关，源IP为本机IP，源MAC为本机MAC）</sup>
    * 计算网段号，查询路由表，获取下一跳地址
    * 查询ARP缓存表获取下一跳的MAC地址，如果没有就发送ARP广播

## 输入URL到显示页面

1. 请求资源是否有缓存且缓存有效
2. 获取IP地址
3. 和服务器建立TCP连接
4. 发送HTTP请求报文
5. 接收响应报文后，解析资源，渲染页面

## DNS查询过程

1. 浏览器的DNS缓存
2. 操作系统的DNS缓存
3. 路由器的DNS缓存
4. **本地域名服务器**的DNS缓存
5. 本地域名服务器向 **根域名服务器** 发出请求，返回 **顶级域名服务器** 的地址
6. 本地域名服务器向 顶级域名服务器 发出请求，返回 **权限域名服务器** 的地址
7. 本地域名服务器向 权限域名服务器 发出请求，返回 对应IP地址

# 字节序

输出变量的各个字节参考输出变量的各个字节

||小端|大端（网络字节序）|
| ----------| ----------------| --------------------|
|数据低位|内存的低地址中|内存的高地址中|
||||

```C++
int main() {
  int number = 1;
  if (*(char *)&number)
    std::cout << "Little-endian!\n";
  else
    std::cout << "Big-endian!\n";
  return 0;
}

int main() {
  int a = 0x1234;
  char *p = (char *)&a;
  printf("%02x\n", *p);
  printf("%02x\n", *(p + 1));
  printf("%02x\n", *(p + 2));
  return 0;
}
# 小端输出 34 12 00
# 大端输出 00 00 12
```

## 转换

* 只有在填socket地址`sockaddr_in`​时，才需要考虑字节序

```C++
#include <arpa/inet.h>
# 将32位的 IP地址 从 主机字节序 转为 网络字节序
uint32_t htonl(uint32_t hostlong);
# 将16位的 端口号 从 主机字节序 转为 网络字节序
uint16_t htons(uint16_t hostshort);

# 将32位的 IP地址 从 网络字节序 转为 主机字节序
uint32_t ntohl(uint32_t netlong);
# 将16位的 端口号 从 网络字节序 转为 主机字节序
uint16_t ntohs(uint16_t netshort);
```

```C++
struct sockaddr_in cliaddr;
ntohs(cliaddr.sin_port)
```
