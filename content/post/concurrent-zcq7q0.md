---
title: 并发
slug: concurrent-zcq7q0
url: /post/concurrent-zcq7q0.html
date: '2024-07-09 10:20:43+08:00'
lastmod: '2024-08-08 13:18:11+08:00'
toc: true
tags:
  - 多线程
categories:
  - 并发
keywords: 多线程
isCJKLanguage: true
---

# 并发

||进程|线程|协程|
| ----------| -------------------------------| ---------------------------------------------------------| ---------------------------------------|
|本质|资源分配|CPU调度的最小单位|用户态线程<br />在堆上模拟栈 + 实现调度器|
|通信|进程通信IPC|共享内存||
|切换开销|CPU的上下文切换<sup>（保存和恢复相关寄存器的内容）</sup><br />加载页表<br />可能刷新<sup>（当ASID分配完后，flush所有TLB，重新分配ASID）</sup>快表TLB<br />系统调用|CPU的上下文切换<br />系统调用<br />|CPU的上下文切换|
|共享||内存空间<br />文件描述符<br />信号处理器<br />进程 ID / 进程组 ID<br />||
|不共享||栈<br />线程 ID<br />寄存器<br />错误返回码<sup>（系统调用或库函数发生错误时，会设置全局变量 errno，各个线程的错误返回码应该是独立的）</sup><br />信号屏蔽码<sup>（每个线程所感兴趣的信号不同，所以线程的信号屏蔽码应该由线程自己管理；但每个线程都共享本进程的信号处理器）</sup><br />||
|优点||开销小<br />通信方便|不需要用户态转换到内核态的切换成本|
|缺点||线程发生错误后，OS会终止整个进程||

||优点|缺点|场景|
| ----------------------| ---------------------------| ---------------------------------------------------------| ----------------------------------------------------------------------------------------------------------|
|一个单线程的进程|简单<br /><br />|不能发挥多核能力<br />EventLoop非抢占，可能发生优先级反转<br />|限制程序的CPU占用率<br />辅助性程序和主进程在同一台机器上时<sup>（如日志文件压缩备份服务应使用单线程压缩工具（gzip/bzip））</sup><br />程序可能会fork(2)<br />看门狗程序（启动其他进程）<br />IO很快达到瓶颈<sup>（如静态Web服务器/FTP服务器，很少的CPU负载能让IO跑满）</sup><br />任务执行时间>>进程创建和销毁时间|
|一个多线程的进程<br />|减少CPU Cache换入换出<br />可scale up<sup>(纵向拓展，即机器性能提升)</sup>||工作集<sup>（服务程序响应一次请求所访问的内存大小）</sup>较大<br />共享数据可修改<br />区分事件优先级<br />不是IO或CPU密集型<sup>（latency和throughput同样重要，不是逻辑简单的IO密集型或CPU密集型程序）</sup>|
|多个单线程的进程<sup>（将一个单线程的进程 运行多份）</sup>|简单||工作集<sup>（服务程序响应一次请求所访问的内存大小）</sup>较小<br />CPU密集型<br />无共享数据或共享数据只读|
|主进程 +<br />Worker进程||||

# 进程

## 进程状态

三态模型：就绪态、运行态、阻塞态

五态模型：新建态、就绪态、运行态、阻塞态、终止态

进程控制原语：进程创建、进程阻塞、唤醒进程、进程终止

|S/STAT|含义|事件|
| --------| -------------------------------------------------| --------------------------------------------------|
|R|TASK\_RUNNING <br />已就绪/正在执行<br />|放入CPU的执行队列中|
|S|TASK\_INTERRUPTIBLE <br />可中断的睡眠状态<br />|等待事件发生|
|D|TASK\_UNINTERRUPTIBLE <br />不可中断的睡眠状态<br />|内核的某些处理流程不能被打断，即不能响应异步信号|
|T|TASK\_STOPPED 暂停<br />TASK\_TRACED 跟踪<br />|发送SIGSTOP信号 <br />gdb调试<br />|
|Z|EXIT\_ZOMBIE 僵尸进程||
|X|EXIT\_DEAD 退出|进程即将销毁|

## 进程描述符（PCB）

包括：

* 进程标识符
* CPU状态：寄存器
* 进程调度信息：进程状态、优先级、其他信息（进程已等待CPU的时间总和、已执行的时间总和等）、阻塞事件
* 进程控制信息：程序和数据的首地址、 进程同步和通信机制、资源清单、进程所在队列 PCB 的链接指针

## 僵尸进程/孤儿进程

||僵尸进程|孤儿进程|
| ----------| ----------------------------------------------------------------------------------------------------------------------------------| ------------------------|
|产生原因|子进程退出后，父进程没有发起函数请求来读取它的结束状态|父进程先退出|
|后果|子进程的PCB内存无法释放，造成内存泄漏；占用进程号|由1号进程领养，无危害|
|处理方式|父进程等待wait子进程结束<br />父进程创建信号处理函数SIGCHLD<sup>(调用wait)</sup><br />父进程忽略SIGCHLD信号，让内核进行回收<br />kill 父进程，由Init进程领养<br />kill -s SIGCHLD PPID<br />|kill -9 pid|
|识别方式|查看进程ps/top|ps -ef中ppid 为1的进程|

## 进程调度

```C++
#include <sched.h>
int sched_setscheduler(struct task_struct *p, int policy,
		       const struct sched_param *param);
struct sched_param my_param;
my_param.sched_priority =20;
if(sched_setscheduler(getpid(),SCHED_FIFO,&my_param)){
	printf("set schedller fail....\n");
	return -1;
}
```

|适用进程|调度器|调度策略|含义|优点|缺点|
| ---------------------------------------| ---------------------------------| -----------------| ------------------------------------------------------------------------------| -----------------------| ------|
|普通进程<br />|CFS<sup>(参考右边的SCHED_NORMAL的CFS算法)</sup><br />|SCHED_NORMAL<br />(SCHED_OTHER<sup>(和SCHED_NORMAL相同)</sup>)|CFS算法：完全公平调度<sup>（vruntime = （实际运行时间*nice值为0的进程权重）/该进程的权重 vruntime跑的越慢（越小）越会被调度）</sup>，根据优先级权重占比<sup>（优先级权重占 所有进程优先级权重之和 的比例）</sup>，决定不同的时间片长度<br />动态时间片，根据系统负载自调整|||
|||SCHED_BATCH|适合批量任务|||
|||SCHED_IDLE|最低优先级运行|||
|实时进程<br />|RT<br />|SCHED_FIFO|先进先出<br />无时间片|||
|||SCHED_RR|Roound-Robin<br />时间片轮转<br />|||
|限期进程<sup>（必须在规定时间内完成）</sup>|DL<br />(Deadline)<br />|SCHED_DEADLINE|最早截止时间优先EDF<sup>(Earliest Deadline First)</sup>算法<br />使用红黑树，按进程绝对截止时间排序，选择最小进程运行<br />|||
|||SJF|短作业优先|平均等待/周转时间最少||
|||HRRN<sup>(High Response Ratio Next)</sup>|高响应比<sup>（响应比Rp  = (等待时间+要求服务时间)/要求服务时间  = 响应时间/要求服务时长）</sup>优先调度|有利于短作业||

## 进程优先级PRI

### 普通进程

特点：

* 任何 实时进程 优先级都高于 普通进程
* 静态优先级不会被内核修改
* 通过nice值调整优先级
* 优先级越小，分配的基时间量就越少

$static\_priority = nice+20+MAX\_RT\_PRIO \in[100,139]$

$PR=static_priority-100\in[0,39]$  

$nice\in[-20,19]$

MAX_RT_PRIO默认为100

### 实时进程

特点：

* 使用chrt调整优先级

$real\_time\_priority\in[0,99]$  

$PR=-1-real\_time\_priority$

### 优先级更新

1. 用户调整优先级
2. 根据进入等待状态的频繁程度提升或降低优先级（IO密集型的优先级高）
3. 长时间得不到执行而被提升优先级

## 进程地址空间

* 内核空间是所有进程**共享**的

​![进程地址空间](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/%E8%BF%9B%E7%A8%8B%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4-20240725111232-6bl89ry.jpg)​

### 数据段和代码段分离

优点：

1. 可重入
2. 可共享数据
3. 可保护代码为只读
4. 更好支持内存回收策略
5. 可共享代码段

缺点：不方便编程

## 创建进程fork

原理：

1. 复制父进程的页表；将父进程的所有内存页设置为只读
2. 为子进程创建新的进程描述符
3. 使用写时复制COW共享地址空间：复制耗时且浪费物理内存；执行exec加载新程序后，复制就没有用了
4. 将内存页的复制延迟到第一次写入时，触发缺页异常，进入内核态复制页面，将新页面设置为可写，原页面的引用计数 - 1
5. 如果页面只有一个引用，可以直接修改

|共享|不共享|
| ----------------------------------------------| ----------------|
|文件描述符，包括socket套接字|变量，包括全局变量<sup>（如果只做读操作，则共享全局变量）</sup>|
|物理页面/物理地址空间（使用COW共享只读页面）|独立的地址空间|

```C++
#include <unistd.h> 
pid_t fork(void)
# 子进程返回值为0，父进程返回值为子进程的PID(>0)
```

### 创建多个子进程

```C++
// 创建n个子进程
int i = 0; // 区分子进程
pid_t pid;
for(i=0;i<n;i++){
	pid = fork();
	if(pid == 0 || pid == -1){ // 子进程或发生错误
		break;
	}
}
if(pid > 0){ // 父进程
	while(wait(NULL)!=-1){} // 阻塞自己，等待子进程
}
```

## 进程通信

特点：

||管道|消息队列|共享内存|信号|Socket|
| ------------------------| ---------------------------------| ---------------------------------------| -----------------------------------| -------------------------| --------------------|
|本质|内核中的特殊文件|存放在内存中的消息链表|相同的物理空间区域|中断处理|TCP|
|速度|同步<sup>（管道写进程需要读进程存在）</sup>|异步|最快|||
|用户进行同步与互斥|×|×|√|×||
|数据格式|无格式字节流|有格式||||
|内核态和用户态数据拷贝|√|√|×|使用系统调用||

### 管道

* 本质：内核中的特殊**文件**，内核中维护一块缓存区，其于管道文件相关联
* 无格式数据传输：传输无格式的字节流，需要事先约定数据格式
* 半双工通信：允许信号在两个方向上传输，但某一时刻只允许信号在一个方向上单向传输
* **双向通信**需要2个管道，数据进入管道后成为无主数据，因此1个管道不可能进行双向通信
* 一次性读取操作：数据一旦被读取后就会被抛弃
* 阻塞，即没有数据可读后，读进程阻塞；管道已满后，写进程阻塞
* 如果写端尝试写，但读端不存在，进程被SIGPIPE信号终止

  如果读端尝试读，但写端不存在，且管道中没有数据，则返回0
* 通过内核进行数据传输，需进行多次用户态和内核态切换，数据传输经过内核的缓存区，相比共享内存慢
* 不需要用户进行同步与互斥，同一时刻只能有一个进程访问

#### 匿名管道pipe

特点：

1. 只能具有血缘关系的进程间通信

    > 因为要从公共祖先处继承 **管道文件描述符**
    >

2. 基于内存进行操作，读写速度更快
3. 关闭所有文件描述符后自动销毁
4. fd[0]读端，fd[1]写端
5. read返回0

```C++
#include <unistd.h>
int fd[2];
// int pipe(int filedes[2]);
if (pipe(fd) < 0) {
	perror("pipe");
	exit(1);
}
pid_t pid = fork();
// fork子进程
if(pid == 0){
	close(fd[0]); // 关闭读端
	write(fd[1], str1, sizeof(str1));
}else{
	close(fd[1]); // 关闭写端
	read(fd[0], line, MAXLINE);
	read(fd[0], buf, BUF_SIZE);
}
```

#### 命名管道FIFO

特点：

1. 可用于任何进程间通信，支持跨网络通信
2. 不同于匿名管道，以FIFO的文件形式存储于**文件系统**中
3. 基于磁盘上实际文件进行操作，进程退出后，管道依然存在
4. 需要显式删除
5. 可以使用文件的相关操作

```C++
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>

int main(){
	if(mkfifo("./mypipe",0666|S_IFIFO) < 0){ // 创建管道
		perror("mkfifo");
		exit(1);
	}
	int fd = open("./mypipe",O_RDONLY);
	if(fd < 0){
        perror("open");
        return 2;
    }
	read(fd, buf, BUF_SIZE);
	close(fd);
}

int main(){
	int fd = open("./mypipe",O_WRONLY);
	if(fd < 0){
        perror("open");
        return 2;
    }
	write(fd, str1, sizeof(str1));
	close(fd);
}
```

### 共享内存

* 原理：允许不相干的进程通过页表映射将**同一段物理内存**连接到它们各自的地址空间中
* 访问无需借助内核<sup>（不需要系统调用）</sup>，不需要来回拷贝数据
* 需要用户进行同步与互斥

使用：

1. 创建或获取共享内存

    int shmget(key_t key, size_t size, int shmflg);

    key：通过ftok获取，或者传入IPC_PRIVATE（用于**有亲缘关系**的进程间通信）由操作系统自动分配

    size：共享内存的大小，以页为单位分配<sup>（Linux系统中一页大小是4KB=4096B，小于4096B则分配一页，size传入4097则分配两页）</sup>，创建共享内存时size>0，访问已存在的共享内存时size = 0

    shmflg：IPC_CREAT（create）、 IPC_EXCL（确保创建，如果共享缓冲区已存在，会调用失败）

                  0666|IPC_CREAT (推荐) 中 0 表示8进制数，666表示读写权限

                  IPC_CREAT|IPC_EXCL 确保会创建共享内存

```C++
#include <sys/ipc.h> 
#include <sys/shm.h>
#include <sys/types.h>

#define PATHNAME "."   
#define PROJ_ID 0x6666
key_t key = ftok(PATHNAME,PROJ_ID);
if(key < 0)
{
	printf("ftok error\n");
}
int shmid = shmget(key, sizeof(UserData), 0666|IPC_CREAT);
if(shmid == -1){
	printf("shmget errno is: %s\n",strerror(errno));
}
```

2. 连接共享内存

    每个进程都需要 shmat，即使是创建共享内存的进程

    每次调用 shmat 都需要先调用 shmget 函数

    void *shmat(int shmid, const void *shmaddr, int shmflg);

    shmid：共享存储区的标识

    shmaddr：共享存储区的开始地址，设置为NULL<sup>(交给操作系统去做)</sup>

    shmflg：当前进程的读写权限，如 SEM_RDONLY，默认0为读写均可

```C++
ShmStruct* addr=(ShmStruct*)shmat(shmid,NULL,0);
if(addr== (void*)-1){
	printf("shmat errno is: %s\n",strerror(errno));
}else{
	printf("Attach shared-memory: %p\n",addr);
	printf("Attach shared memory status:\n");
	system("ipcs -m");
}
```

3. 断开连接共享内存

    int shmdt(const void *shmaddr);

    shmaddr：共享存储区的开始地址

4. 删除共享内存

    int shmctl(int shmid, int cmd, struct shmid_ds *buf);

    shmid：共享存储区的标识

    cmd：IPC_RMID 删除

    buf：设置NULL

### 消息队列

* 本质：**存放在内存中的消息链表**
* 可以实现消息的**随机查询**，如按消息类型读取
* 异步：写入消息时不需要其他​进程等待​消息到达<sup>（管道写进程需要读进程存在）</sup>；接收者必须**轮询**消息队列，才能收到最近的消息
* 需要将数据从内核态和用户态之间拷贝
* 无需同步与互斥，需要内核介入

查看消息队列参考消息队列

```C++
#include <stdio.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

#define PATHNAME "."   
#define PROJ_ID 0x6666
#define MAX_SIZE 1024
key_t key = ftok(PATHNAME,PROJ_ID);
# 返回消息队列标识符
int msgid = msgget(key, IPC_CREAT | 0666); // 参考共享内存

struct msgbuf buffer;
buffer.mtype = 1;
msgsnd(msgid, &buffer, sizeof(buffer)-1, 0);
msgrcv(msgid, &buffer, MAX_SIZE, 1, 0);
printf("%s\n", buffer.mtext);

# 设置消息队列属性
struct msqid_ds mqs;
memset(&mqs, 0, sizeof(mqs));
mqs.msg_perm.mode = 0444;
msgctl(msgid, IPC_SET, &mqs);

# 获取消息队列状态
msgctl(msgid, IPC_STAT, &mqs);

# 删除消息队列
msgctl(msgid, IPC_RMID, NULL);
```

### 信号

原理：

1. 操作系统提供发送信号的系统调用
2. 该系统调用会将信号放到目标进程的`信号队列`​中
3. 进程可以阻塞<sup>（该信号的传递被延迟，直到其阻塞被取消时才被传递给进程）</sup>​某个信号；接收信号后可以忽略信号、处理信号<sup>（定义信号处理函数）</sup>、执行默认操作。

* 不可处理、忽略、捕捉的信号：SIGSTOP、SIGKILL
* 命令行使用kill发送信号

|信号|含义||
| ---------| -----------------------------------------------| ---|
|SIGABRT|程序的异常终止，如调用 abort||
|SIGALRM|闹钟函数||
|SIGCHLD|子进程终止||
|SIGFPE|错误的算术运算，如除以零或导致溢出的操作||
|SIGILL|检测非法指令||
|SIGINT|程序终止信号，如CTRL+C||
|SIGSTOP|程序停止信号，如CTRL+Z||
|SIGKILL|程序KILL信号，如CTRL+\ 或 kill -s SIGKILL pid|9|
|SIGSEGV|非法访问内存||
|SIGTERM|发送到程序的终止请求||

1. 发送信号

```C++
#include <signal.h>
int kill(pid_t pid, int signo);	# 给指定进程发送信号
int raise(int signo);  # 给当前进程发送信号

#include <stdlib.h>
void abort(void); 
# 发送SIGABRT信号，使程序异常终止

#include <unistd.h>
unsigned int alarm(unsigned int seconds);
# 到达时间后发送SIGALRM信号
# 返回以前设定的闹钟时间还余下的秒数
# seconds: 0表示取消以前定的闹钟;如果在seconds秒内再次调用了alarm函数设置了新的闹钟，则后面定时器的设置将覆盖前面的设置
# 如果未指定SIGALRM信号的处理函数，则通过调用signal函数终止进程。
```

2. 设置信号处理函数

    signal函数在UNIX系列的不同OS中可能存在区别，但sigaction函数完全相同

```C++
# 声明
void (*signal(int sig, void (*func)(int)))(int)
struct sigaction{
	void (*sa_handler)(int);
	sigset_t sa_mask; // 初始化为0
	int sa_flags; // 初始化为0
}
int sigaction(int sig, const struct sigaction *act, struct sigaction *oldact);

# 方式1
	void signhandler(int sig){}
	signal(SIGINT, sighandler);
# 方式2
	struct sigaction act;
	act.sa_handler = func;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	sigaction(SIGALRM, &act, 0);
	alarm(2);
```

### Socket

优点：

* 可跨主机（伸缩性）
* OS自动回收、程序重启后容易恢复、快速failover(OS关闭连接)
* port独占防止程序重复启动
* 可记录、可重现（tcpdump和wireshark分析性能、解决争端）
* 压力测试（tcpcopy）
* 跨语言（服务器和客户端语言可不同）
* 任何一个进程可单独重启（TCP连接是可再生的，重建连接后可继续工作）

缺点：

* 会有marshal/unmarshal（序列化）的开销，需要选择合适的消息格式（wire format）,推荐google Protocol Buffers
* 可在TCP上构建RPC/HTTP/SOAP之类的上层通信协议，和应用级的广播协议（构建分布式系统）

## 进程相关函数

|功能|函数|
| --------| ------|
|进程ID|pid_t getpid()<sup>(<unistd.h>)</sup>|
|<br />|<br />|

## 等待与分离

# 线程

## 创建线程thread

传入可调用对象、std::packaged\_task（参考future获取异步结果）

传入参数：传值、传引用、传指针

```c++
std::jthread  C++20支持自动join的线程
# 传入可调用对象
std::thread worker(DoWork, param1, param2);

# 传入std::packaged_task
std::packaged_task<int(int)> task([](int x) { return x * 3; });
std::thread t(std::move(task), 5);

# 传入成员函数
class_name item;
std::thread worker(&class_name::func, &item, param1); // 传对象指针
std::thread worker(&class_name::func, item, param1); // 拷贝副本

# 传引用
std::thread worker(DoWork, std::ref(param1));
```

## 线程通信

全局变量、传参（传值、传引用、传指针）

### 队列

库：Disruptor、concurrentqueue、MPMCQueue.h

### 栈

```C++
# 基于CAS
template <typename T> class mt_stack {
 	std::deque<T> s_;
 	int cap_ = 0;
  	struct counts_t {
    	int p_ = 0; // 生产者索引，第一个空闲槽的索引
    	int c_ = 0; // 使用者索引，最近一个完全构造的元素的索引
    	bool equal(std::atomic<counts_t>& n) {
     		if (p_ == c_) return true;
     		*this = n.load(std::memory_order_relaxed);
     		return false;
   		}
 	};
 	mutable std::atomic<counts_t> n_;
public:
 	mt_stack(size_t n = 100000000) : s_(n), cap_(n) { }
 	void push(const T& v);
 	std::optional<T> pop();
}

void push(const T& v) {
  	counts_t n = n_.load(std::memory_order_relaxed);
 	if (n.p_ == cap_) abort();
 	while (!n.equal(n_) || 
   		!n_.compare_exchange_weak(n, {n.p_+1, n.c_},
      		std::memory_order_acquire,
     		std::memory_order_relaxed)) {
   		if(n.p_ == cap_) { ...allocate more memory ...}
 	};
 	++n.p_;
 	new (&s_[n.p_]) T(v);
 	assert(n_.compapa_exchange_strong(n, {n.p_, n.c_ + 1},
    std::memory_order_release, std::memory_order_relaxed));
}

std::optional<T> pop() {
 	counts_t n = n_.load(std::memory_order_relaxed);
 	if (n.c_ == 0) return std::optional<T>(std::nullopt);
	while (!n.equal(n_)) ||
   		!n_.compare_exchange_weak(n, {n.p_, n.c_ - 1},
    		std::memory_order_acquire,
   			std::memory_order_relaxed) {
 		if (n.c_ == 0) return std::optional<T>(std::nullopt);
 	};
 	--n.cc_;
 	std::optional<T> res(std::move(s_[n.p_]));
 	s_[n.pc_].~T();
 	assert(n_.compare_exchange_strong(n, {n.p_ - 1, n.c_},
	std::memory_order_release, std::memory_order_relaxed));
	return res;
}
```

```C++
# 基于锁
template <typename T> class mt_stack {
	std::stack<T> s_;
	mutable std::shared_mutex l_;
public:
std::optional<T> pop() {
	std::unique_lock g(l_);
	if(s_.empty()) {
		return std::optional<T>(std::nullopt);
	}else{
		std::optional<T> res(std::move(s_.top()));
		s_.pop();
		return res;
	}
}
std::optional<T> top() const {
	std::shared_lock g(l_);
	if(s_.empty()) {
		return std::optional<T>(std::nullopt);
	}else{
		std::optional<T> res(std::move(s_.top()));
		return res;
	}
}
void push(const T& v){
	std::unique_lock g(l_);
	s_.push(v);
}
}
```

## 线程相关函数

|功能|函数|
| -----------------| --------------------------------------------|
|线程ID|std::thread::id std::this_thread::get_id()|
|TLS线程本地存储|__thread<br />__declspec(thread)<sup>(Microsoft使用)</sup><br />thread_local<sup>(变量的生命周期延长为整个线程)</sup><br />|

## 等待与分离

每个子线程都需要且只能调用一次join()或detach()

分离子线程需要确保其访问的数据<sup>（变量的引用不会随主线程的退出而失效,如引用参数）</sup>是有效的

```c++
std::thread t(do_background_work);
if( worker.joinable() ){
	# 主线程等待子线程t完成
	t.join();
}
if( worker.joinable() ){
	# 分离子线程,子线程在后台运行,被C++运行时库接管
	t.detach();
}

# 等待线程组的每个线程
for (auto& thr : threads_)
{
	thr->join();
}
```

## 异步async

* 创建线程的新方式，传参参考创建线程thread

```c++
#include <future>
std::future<string> answer = std::async(hello,(string)"hello");
# 立刻执行线程
std::future<string> answer = std::async(std::launch::async, hello);
# 延迟执行线程(直到调用wait或get,注意wait_for不可以)
std::future<string> answer = std::async(std::launch::deferred, hello);
```

## 线程池

```c++
#pragma once
#include <vector>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <deque>
#include <thread>

class ThreadPool {
public:
    typedef std::function<void ()> Task;

    ThreadPool();
    ~ThreadPool();

    void setMaxQueueSize(int maxSize) { maxQueueSize_ = maxSize; }
    void setThreadInitCallback(const Task& cb)
    { threadInitCallback_ = cb; }
    void start(int numThreads);
    void stop();
    size_t queueSize() const;
    void run(Task task);  // 传入任务
private:
    bool isFull() const;
    void runInThread();  // 线程主循环函数
    Task take();

    mutable std::mutex mutex_;
    std::condition_variable notEmpty_ ;
    std::condition_variable notFull_ ;
    Task threadInitCallback_;
    std::vector<std::unique_ptr<std::thread>> threads_; // 线程组
    std::deque<Task> queue_; // 任务队列
    size_t maxQueueSize_;
    bool running_;
};
```

```c++
#include <cassert>

ThreadPool::ThreadPool()
    : maxQueueSize_(0),
      running_(false)
{
}

ThreadPool::~ThreadPool()
{
    if (running_)
    {
        stop();
    }
}

void ThreadPool::start(int numThreads)
{
    assert(threads_.empty());
    running_ = true;
    threads_.reserve(numThreads);
    for (int i = 0; i < numThreads; ++i)
    {
        threads_.emplace_back(new std::thread(std::bind(&ThreadPool::runInThread, this)));
    }
    if (numThreads == 0 && threadInitCallback_)
    {
        threadInitCallback_();
    }
}

void ThreadPool::stop()
{
    // 停止时可能有任务没有完成
    {
        std::unique_lock<std::mutex> lock(mutex_);
        running_ = false;
        notEmpty_.notify_all();
    }
    for (auto& thread : threads_)
    {
        thread->join();
    }
}

size_t ThreadPool::queueSize() const
{
    std::unique_lock<std::mutex> lock(mutex_);
    return queue_.size();
}

void ThreadPool::run(Task task){ // 传入任务
    if(threads_.empty()){
        task(); // 主线程完成任务
    }else{
        std::unique_lock<std::mutex> lock(mutex_);
        while(isFull()){
            notFull_.wait(lock);
        }
        assert(!isFull());
        queue_.push_back(std::move(task));
        notEmpty_.notify_one();
    }
}

ThreadPool::Task ThreadPool::take(){ // 获取任务
    std::unique_lock<std::mutex> lock(mutex_);
    while(queue_.empty() && running_){
        notEmpty_.wait(lock);
    }
    Task task;
    if(!queue_.empty()){
        task = queue_.front();
        queue_.pop_front();
        if(maxQueueSize_ > 0){
            notFull_.notify_one();
        }
    }
    return task;
}

bool ThreadPool::isFull() const{
    return maxQueueSize_ > 0 && queue_.size() >= maxQueueSize_;
}

void ThreadPool::runInThread(){ // 线程主循环函数
    try{
        if(threadInitCallback_){
            threadInitCallback_();
        }
        while(running_){
            Task task(take());
            if(task){
                task();
            }
        }
    }catch(const std::exception& ex){
        fprintf(stderr, "reason: %s\n", ex.what());
        exit(1);
    }catch(...){
        fprintf(stderr, "unknown error\n");
        throw;
    }
}
```

### 线程数量（阻抗匹配原则）

1. 密集计算所占事件比重为P（0~1），系统有C个CPU，线程池大小T=C/P（可上下浮动50%）
2. 如果P<0.2，则T可以取固定值，如5*C，C可以为“**分配给这项任务的CPU数目**”

```C++
# 获取硬件支持的并发线程数，如果系统信息无法获取，返回0
unsigned int in = std::thread::hardware_concurrency();
```

# 同步与互斥

特点：

* **进程**实现同步与互斥需要使用**共享内存** 或 **文件系统**

|同步方式|优先级翻转||
| ----------| ------------| ----------------|
|信号量|√||
|锁|×||
|原子操作||仅用于单个变量|

## 信号量

* 本质：低级通信，传输信号量
* 信号量大于 0 时表示当前可用资源的数量；小于 0 时表示等待使用该资源的进程个数
* P操作：**申请资源**

  将信号量S的值减1，即S\=S-1； 

  如果S\>\=0，则该进程继续执行；

  否则该进程置为等待状态，排入等待队列。
* V操作：释放资源

  将信号量S的值加1，即S\=S+1；

  如果S<=0，则从等待队列中唤醒1个进程
* 函数为原子操作
* 原理：硬件提供的原子指令，如CAS（Compare And Swap）

### 基于内存的信号量

1. 创建基于**内存的信号量**

    只能调用一次sem_init

    int sem_init(sem_t *sem, int pshared, unsigned int value);

    pshared：0为只在当前进程的所有线程中共享，非0为进程间共享（sem需要在共享内存中）

    value：信号量初始值

```C++
#include <semaphore.h>

sem_t bin_sem;
sem_init(&bin_sem, 0, 1);
```

2. 增加信号量，即释放资源

    int sem_post(sem_t *sem);
3. 减少信号量，即申请资源

    int sem_wait(sem_t *sem);

    int sem_trywait(sem_t *sem);  非阻塞
4. 销毁信号量

    int sem_destroy(sem_t *sem);

### 有名信号量

1. 创建或打开有名信号量

    有名信号量总能在进程间共享

    sem\_t \*sem\_open(const char \*name, int oflag);

    sem\_t \*sem\_open(const char \*name, int oflag,mode\_t mode, unsigned int value);

    name：与信号量关联的文件名

    oflag：0（打开已创建的）、O\_CREAT、O\_CREAT|O\_EXCL（如果没有指定的信号量就创建）

    mode：权限位

    val：信号量初始值

```C++
#include <fcntl.h>        
#include <sys/stat.h>      
#include <semaphore.h>

```

2. 增加信号量，即释放资源
3. 减少信号量，即申请资源
4. 关闭信号量

    int sem\_close(sem\_t \*sem);
5. 销毁信号量

    int sem\_unlink(const char \*name);

## 锁

* 优点：互斥锁解决了**优先级翻转<sup>（高优先级任务等待低优先级任务的锁时，中优先级任务抢占了低优先级任务的执行）</sup>**问题：优先级继承<sup>（高优先级的任务等待低优先级任务的锁时，低优先级继承高优先级，直到释放锁后还原其优先级）</sup>

  [二值信号量和互斥锁到底有什么区别？ - 代码螺丝钉 - 博客园 (cnblogs.com)](https://www.cnblogs.com/codescrew/p/8970514.html)
* 缺点：

  * 死锁
  * **活锁**：

    多线程互相谦让。

    解决：引入随机等待时长，增加重试次数
  * **锁护送**：当多个相同优先级的线程频繁地争抢同一个锁。

    导致线程频繁切换、调度粒度变小、分配的时间片大小不同。

    解决：在每个线程获取锁的时候先尝试（try），如果尝试多次仍不成功，再阻塞。
* 数据库中的锁：

  * 可能需要先获取常规锁，在实际访问对应页面时，获取LWLock
  * LWLock轻量级读写锁，只用于对共享内存变量的互斥访问，如Buffer（Clog Buffer、Shared Buffer、wal buffer）。

    等锁使用信号量、等待队列、原子操作实现。
  * 常规锁<sup>（https://blog.csdn.net/qq_43687755/article/details/106478057）</sup>具有锁协议（对应隔离级别）、锁的粒度、锁兼容矩阵。

    等锁使用epoll实现。
* 锁通常需要设置为**mutable**

### 互斥锁mutex

* 通过 优先级继承 解决了 优先级翻转 问题
* 如果函数返回保护数据的指针或引用，会破坏数据
* 在 **const函数** 中使用时，需要用 **mutable** 修饰锁
* **缺点**：系统调用、线程重新调度、上下文切换开销大

|锁定方式|含义|备注|
| -----------------| ------------------------------------------------------------------------------| -------------------------------------|
|lock()/unlock()||所有分支路径都要unlock()|
|lock_guard|创建加锁，离开作用域后自动解锁<br />不可移动，不可复制|C++11，使用 **{ }** 限定作用域，达到提前解锁|
|unique_lock|允许中途解锁，可以移动，不可复制<br />支持延迟锁定，支持锁的所有权转移<sup>（可以从函数中返回）</sup><br />支持限时尝试锁定、超时锁定|C++11|
|scoped_lock|锁定多个互斥量<sup>（也可以使用std::lock，参考加多个锁）</sup>|C++17|

```C++
#include <mutex>

std::mutex mtx;
mtx.lock();
mtx.unlock();
mtx.try_lock();

{
std::lock_guard<std::mutex> guard(mtx);
std::lock_guard<std::mutex> guard(mtx, std::adopt_lock);
# std::adopt_lock : mutex已经加锁，即领养锁
# std::try_to_lock：if(guard.owns_lock()) 即使不能获得锁，也会立刻返回，不会阻塞
}

std::unique_lock<std::mutex> guard(mtx);
std::mutex *ptx = guard.release();
# 释放锁，返回管理的mutex对象指针
std::unique_lock<std::mutex> guard2(std::move(guard))
# 可以移动move
```

#### 尝试锁定（+延迟锁定）

```C++
std::unique_lock<std::mutex> guard(mtx, std::defer_lock);
// std::defer_lock ：初始化一个不加锁的mutex，如果mutex已经加锁，会抛异常
# 尝试锁定（不阻塞）
if(!guard.try_lock()){return ;}

# 尝试200ms后返回
while (!guard.try_lock_for(std::chrono::milliseconds(200))) {
	...
}

# 尝试加锁直到某个时间点
try_lock_until()
```

#### 加多个锁

1. 使用lock，再分别使用lock_guard + std::adopt_lock
2. 使用unique_guard + std::defer\_lock，再使用lock

```C++
std::lock(mutex1,mutex2,....)
# 如果其中一个加锁失败，会将其他的锁释放
std::lock_guard<std::mutex> guard1(mutex1, std::adopt_lock);
std::lock_guard<std::mutex> guard2(mutex2, std::adopt_lock);
或
std::unique_lock<std::mutex> lock1(from.m, std::defer_lock);
std::unique_lock<std::mutex> lock2(to.m, std::defer_lock);
std::lock(lock1, lock2);
```

### 读写锁

特点：

* 使用 **shared_mutex、shared_timed_mutex**
* Boost库提供 upgrade_lock 升级锁，STL未提供
* 获取写锁时，可能不小心调用会修改状态的函数
* 需要更新当前reader的数目，影响性能
* 在 **const函数** 中使用时，需要用 **mutable** 修饰锁
* C++14支持

```C++
#include <shared_mutex>
mutable std::shared_mutex l_;
l_.lock();
if(l_.try_lock())
l_.unlock()
# 申请和释放写锁
l_.lock_shared();
if(l_.try_lock_shared())
l_.unlock_shared();
# 申请和释放读锁

std::unique_lock<std::shared_mutex> g(l_);
# 申请写锁（排他锁）
std::shared_lock<std::shared_mutex> g(l_);
# 申请读锁

int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const 
pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); 
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock); 
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock); 
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock); 
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

### 自旋锁

* 轮询忙等待，消耗CPU
* 场景：加锁时间非常短（几个周期）
* 优点：避免OS重新调度、上下文切换的开销
* 持有时间过长可能触发内核core dump
* 自旋锁在单核CPU<sup>(占有资源的进程无法得到CPU，因此无法释放锁)</sup>下不起作用

```C++
class Spinlock {
public:
    void lock(){
        for(int i=0; flag_.load(std::memory_order_relaxed) ||
            flag_.exchange(1, std::memory_order_acquire); ++i){
            if( i==8 ){
                lock_sleep();
                i=0;
            }
        }
    }
    void unlock(){
        flag_.store( 0, std::memory_order_acquire );
    }
    void lock_sleep(){
        static const timespec ns = {0,1};
        nanosleep(&ns, NULL);
    }
    Spinlock() = default;
    Spinlock(const Spinlock& other) = delete;
private:
    std::atomic<unsigned int> flag_;
}
```

### 超时锁

* 可以使用**unique_lock 或 timed_mutex**、shared_timed_mutex、recursive_timed_mutex

```C++
std::timed_mutex mtx;
if( mtx.try_lock_for(std::chrono::milliseconds(2000)) )
	mtx.unlock()
-----
std::unique_lock<std::mutex> guard(mtx, std::defer_lock);
// std::defer_lock ：初始化一个不加锁的mutex，如果mutex已经加锁，会抛异常
# 尝试锁定（不阻塞）
if(!guard.try_lock()){return ;}

# 尝试200ms后返回
while (!guard.try_lock_for(std::chrono::milliseconds(200))) {
	...
}

# 尝试加锁直到某个时间点
try_lock_until()
```

### 可重入锁

* 允许同一线程多次获取同一个锁
* 可使用 **recursive_mutex**、recursive_timed_mutex
* 场景：自调用函数、递归函数

### 文件锁

```c++
#include <unistd.h>
#include <fcntl.h>
int file = open("example.txt", O_RDWR);
# 获取文件锁
struct flock fl;
fl.l_type = F_WRLCK;  // 写锁
fl.l_whence = SEEK_SET;
fl.l_start = 0;
fl.l_len = 0;
fl.l_pid = getpid();
fcntl(file, F_SETLKW, &fl);  // 阻塞获取文件锁

# 释放文件锁
fl.l_type = F_UNLCK;
fcntl(file, F_SETLK, &fl);

close(file);
```

## 原子操作

* 优点：解决了死锁、活锁、锁护送
* 缺点：仅用于单个变量
* 需要**所有线程**都使用原子操作访问
* 原子操作是申请对**整个缓存行**的独占访问权限
* 原子变量不能移动或拷贝

### 创建

```C++
#inlcude <atomic>
std::atomic<int32_t> atm = 0;
std::atomic<bool>    atm{ false };
std::atomic<int>     atm(0);
std::atomic<unsigned long> atm[1024];
```

|原子变量类型|含义|
| --------------------------| ------------------------------|
|atomic\_bool|bool布尔型|
|atomic\_char|char字符型|
|atomic\_uchar|unsigned char|
|atomic\_schar|signed char|
|atomic\_short|short|
|atomic\_ushort|unsigned short|
|atomic\_int|int|
|atomic\_uint|unsigned int|
|atomic\_long|long|
|atomic\_ulong|unsigned long|
|atomic\_llong|long long|
|atomic\_ullong|unsigned long long|
|atomic\_wchar\_t|wchar\_t|
|atomic\_char16\_t|char16\_t|
|atomic\_char32\_t|char32\_t|
|atomic\_intmax\_t|intmax_t<sup>(<inttypes.h>)</sup>|
|atomic\_uintmax\_t|uintmax_t<sup>(<inttypes.h>)</sup>|
|atomic\_intptr\_t|intptr_t <sup>(<stddef.h>)</sup> 指针宽度整数|
|atomic\_uintptr\_t|uintptr_t<sup>(<stddef.h>)</sup> 指针宽度无符号整数|
|atomic\_size\_t|size\_t|
|atomic\_ptrdiff\_t|ptrdiff\_t 两个指针的距离|

### 成员函数

除store()，均返回操作前的值

```C++
a.store(true)
int b=a.load()
```

|函数|功能|
| -------------------------| ----------------------------------------|
|load()|读取|
|store()|写入，无返回值|
|exchange()|写入|
|fetch\_add()|+|
|fetch\_sub()|-|
|fetch\_and()|按位与|
|fetch\_or()|按位或|
|fetch\_xor()|按位异或|
|fetch\_min()|将原子变量和给定值的较小值存入原子变量|
|fetch\_max()|将原子变量和给定值的较大值存入原子变量|
|fetch\_mul()|*|
|fetch\_div()|/|
|fetch\_and\_not()|按位与非|
|fetch\_negate()|取反|

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240725140956-wg9y6dn.png)​

### CAS（比较 And 交换）

1. 读取变量的当前值
2. 检查必要条件，如果条件失败，则执行特定操作
3. 如果当前值仍然等于之前读取的值，则以原子方式将值替换为所需的结果
4. 如果步骤3失败，则当前值已经更新，再次检查，重复步骤3、4

```C++
# 即使当前值和预期值匹配，weak版本有时候也会返回false
bool compare_exchange_weak (T& expected, T val)
# 对于非循环算法，首选compare_exchange_strong，否则为compare_exchange_weak + 循环
bool compare_exchange_strong (T& expected, T val)

std::atomic_bool isRunning = false;
bool expected = false;
if(!isRunning.compare_exchange_weak(expected, true)){
	return ;
}
```

#### 有界原子操作

```C++
# 有界原子递增操作
std::atomic<int> n_ = 0;
int bounded_fetch_add(int dn, int maxn) {
	int n = n_load(std::memory_order_relaxed); // 1
	do {
		if( n+dn >= maxn || n+dn<0 ) return -1;  // 2
	}while (!n_.compare_exchange_weak(n, n+dn,
			std::memory_order_release,
			std::memory_order_relaxed)); // 3、4
	return n;
} 
int i = 0;
while( bounded_fetch_add( ... ) ){
	if( ++i == 8) {
		static constexpr timespec ns = {0,1};
		i=0;
		nanosleep(&ns, NULL);
	}
}
```

### 原子变量数组

> 由于atomic不能复制或移动，因此不能拥有一个vector<atomic>

```C++
std::vector<std::unique_ptr<std::atomic<int>>> examp;
examp.resize(64);   // 64 default unique_ptrs; they point to nothing

// init the vector with unique_ptrs that actually point to atomics
for (auto& p : examp) {
	p = std::make_unique<std::atomic<int>>(0);   // 初始化atomic变量
}

// use it
*examp[3] = 5;

for (auto& p : examp) {
	 cout << *p << ' ';
}
```

## 条件变量

* 本质：一个或多个线程等待某个布尔表达式为真，即等待别的线程“唤醒”它
* 场景：多个线程等待同一事件
* 为什么和锁一起使用：判断条件时需要加锁；**不满足条件** 和 进入等待唤醒队列 之间如果生产者修改了条件并通知了线程，可能丢失唤醒信号。
* cond.wait()将当前线程加入等待唤醒队列，然后解开锁

```c++
#include <condition_variable>
# 只能和std::mutex一起工作, 只能使用unique_lock
std::condition_variable cond;
# 可以和任何满足最低标准的互斥量工作
std::condition_variable_any cond;
std::mutex mtx;

# 事件生产者
1. 加锁 unique_lock
2. do something
3. 随机通知一个在等待唤醒队列中的线程（资源可用）
   cond.notify_one() == pthread_cond_signal(&cond)
   通知其他所有等待唤醒队列中的线程（状态转换）
   cond.notify_all() == pthread_cond_broadcast(&cond)

# 事件消费者
1. 加锁 unique_lock
2. while(判断条件不满足){ cond.wait(lk); }
3. do something
```

### 虚假唤醒

原因：

1. notify\_all() 中多个线程被唤醒
2. 可能被信号唤醒
3. Linux pthread实现不会出现EINTR错误导致的spurious wakeup

    当一个慢系统调用(read, write 等)被阻塞时，此时外部产生信号，然后捕获处理信号，处理函数返回，这个系统调用不再阻塞而是被中断，返回错误(EINTR)

方法：使用 **while** 而不是 if 来判断

## 事件通知eventfd

* 支持多路IO复用，epoll监听eventfd的读事件
* 可以在类中定义唤醒wakeup函数，类外仅需调用wakeup函数
* 进程可以先定义eventfd再fork；线程可以通过传值 eventfd
* eventfd不需要加锁

```C++
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);
uint64_t u = 1;
ssize_t n;
int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
switch(fork()){
case 0:
	n = write(evtfd, &u, sizeof(uint64_t));
default:
	n = read(evtfd, &u, sizeof(uint64_t)); 
}
```

# 让步与睡眠

**yield**：将当前线程所抢到的**时间片让给其他线程**，等到其他线程使用完时间片后，由操作系统调度，当前线程和其他线程一起抢时间片。

```C++
std::this_thread::yield()

#include <sched.h>
int sched_yield(void);
```

**sleep**：**阻塞线程**，让出CPU。当休眠时间结束后，重新参与调度。

> **发生信号时**可能唤醒由于sleep而阻塞的线程。

```C++
#include <chrono>
using namespace std::literals::chrono_literals
std::this_thread::sleep_for(1s);  // 可以使用24h,15min
std::this_thread::sleep_for( std::chrono::milliseconds(200) )
# 指定时间点
std::this_thread::sleep_until()  // 例如早上6点启动

#include <unistd.h>
unsigned sleep(unsigned seconds);  // 以秒为单位
void usleep(int micro_seconds);  // 以微秒为单位
#include <time.h>
int nanosleep(const struct timespec *req,struct timespec *rem); // 以纳秒为单位
```

# 获取异步结果

1. 进程通信 或 线程通信 

    * 条件变量：唤醒主线程
2. 使用future和promise

## future

1. 异步async返回值
2. 作为std::thread的传入参数
3. std::packaged_task 调用

```c++
# 获取future
std::promise<int> pr;
std::future<int> fut = pr.get_future();
void DoWork(std::promise<int>& pr){
	pr.set_value(5);
}
std::thread worker(DoWork, std::ref(pr));
或
std::future<string> fut = std::async(hello)
或
std::packaged_task<int(int)> task([](int x) { return x * 3; });
std::thread t(std::move(task), 5);
std::future<int> fut = task.get_future();

# 等待异步完成
cout<< fut.wait();
# 获取异步结果
cout<< fut.get();
# 超时等待,wait_for或wait_until
if (fut.wait_for(0) == std::future_status::deferred)  // 如果任务被推迟
{
    ...     // fut使用get或wait来同步调用f
} else {    // 任务没有被推迟
    while(fut.wait_for(100ms) != std::future_status::ready) { // 不可能无限循环
      ...    // 任务没有被推迟也没有就绪，所以做一些并发的事情直到任务就绪
    }
    ...        // fut就绪
}
```

# 仅执行一次的函数

一个线程执行时，**另外一个会被锁住**直到函数执行完毕

场景：延迟初始化

编译时需要添加-pthread

```C++
#include <mutex>
class X
{
public:
    data getData()const
    {
        std::call_once(initDataFlag, &X::initData, this);
        ...
    }
private:
    mutable std::once_flag initDataFlag;
    void initData()const;
};
```

# 退出

通常while( running_ )死循环作为函数

```c++
# 终止进程，不进行任何清理
# 生成coredump
#include <stdlib.h>
abort()

# 终止进程前，将文件缓存区写回文件，销毁全局和static对象，关闭IO通道
# 会执行退出函数atexit
# 不会调用局部对象的析构函数（需在exit前显式调用）
#include <stdlib.h>
exit(0) // 正常退出

# 终止进程, 不刷新IO缓冲区, 不执行退出函数atexit（和exit相比）
#include <unistd.h>
_exit(0)
```

## 设置退出函数

atexit在**return、exit**前执行，在_exit和abort前不执行atexit

调用退出函数顺序 与 **函数登记**相反

```C++
void func1() { printf("The process is done...\n"); }
atexit(func1);
```

# 协程Routine/纤程Fiber

||有栈（纤程）|无栈|
| ------| ----------------------------------| --------|
|本质|可以中断和恢复执行的函数|状态机|
|存储|函数栈|系统栈|
|优点|强大且灵活|效率高<sup>（使用系统栈，CPU cache缓存友好）</sup>|
|支持|libco<br />boost.context<br />|C++20|
|缺点|栈过多后，切换开销大，占用内存多||

# 死锁

## 必要条件

1. **互斥**：资源独占且排他使用
2. **不可剥夺**：只能由获得该资源的进程释放资源
3. **请求和保持**：在占有一部分资源的同时，申请新的资源
4. **循环等待**：A等待B占用的资源，B等待C占用的资源，C等待A占用的资源

## 预防死锁

1. 进程在申请新资源时不能得到满足而变为等待状态之前，必须释放已占有的资源
2. 一次性申请所需的所有资源：无法预知进程所需的所有资源；降低资源利用率，降低并发度
3. **资源有序分配**

    * 对资源编号，按顺序申请资源
    * 层次锁，每次从高到低上锁
    * 使用 std::lock 加多个锁
4. 在分配资源时，使用 银行家算法 检测

### 银行家算法

特点：

* 本质：要设法保证系统动态分配资源后不进入不安全状态，以避免可能产生的死锁
* **安全序列**：序列中每一个进程需要的资源（Need）<= 系统当前剩余的资源量 + 前面的进程所占用的资源之和<sup>（前面的进程执行完成之后会释放资源）</sup>
* 如果不存在一个安全序列，则系统处于**不安全**状态。

**构建安全序列**：

|进程顺序|系统剩余资源（before）|进程已占有资源（Alloc）|进程需要的资源（Need）<br />= 最大资源需求MAX - Alloc|系统剩余资源（after）|
| ----------| ------------------------| -------------------------| ---------------------------------------------------| -----------------------|
|P1|1 5 2 0<br />|1 3 5 4|1 0 0 2|1 8 7 4|
|P2|1 8 7 4||||

1. 寻找满足以下条件的进程

    * 请求资源 <= 进程需要的资源Need
    * 请求资源 <= 系统剩余资源Avail
2. 进程完成，系统剩余资源Avail += 进程占有资源

## 检测和解决死锁

特点：

* **本质**：允许死锁发生，监控死锁是否发生，采取措施并以最小代价解决死锁
* **检测时机**：定时检测、进程等待时（请求锁时写入开始等待的时间戳，请求成功后将时间戳置0<sup>(可以每隔一段时间检测，也可以设置定时器唤起死锁检测)</sup>）、资源利用率<sup>（CPU利用率、内存利用率）</sup>下降时等
* **检测方法**：进程-资源分配图、使用gdb或gstack查看调用栈、off-cpu火焰图
* **解决死锁**：

  1. 剥夺其他进程的资源给死锁进程
  2. 逐个撤销死锁进程
  3. 定期设置检查点，某个死锁进程回滚到取得资源的检查点，将释放的资源分配给其他死锁进程

### 进程-资源分配图

1. 先看系统还剩下多少资源没分配，再看有哪些进程是不阻塞<sup>（系统有足够的空闲资源分配给它）</sup>的
2. 把不阻塞的进程的所有边都去掉，形成一个孤立的点，再把系统分配给这个进程的资源回收回来
3. 看剩下的进程有哪些是不阻塞的，然后又把它们逐个变成孤立的点
4. 如果所有的资源和进程都变成孤立的点，则可完全简化，不会产生死锁
