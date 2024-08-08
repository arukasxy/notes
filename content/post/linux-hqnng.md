---
title: Linux
slug: linux-hqnng
url: /post/linux-hqnng.html
date: '2024-07-11 20:32:54+08:00'
lastmod: '2024-08-08 13:29:30+08:00'
toc: true
tags:
  - Linux
categories:
  - 工具
keywords: Linux
isCJKLanguage: true
---

# Linux

# 进程

## 查看进程ps/top

1. ps 显示进程某个时间点的状态（PID、终端、使用的CPU时间）

    |选项|作用|
    | :------------------------------------------------: | --------------|
    |-aux<br />-ef|显示所有进程|
    |-e|显示环境变量|
    |--forest|进程树|
    |-eT|显示**线程**信息|
    |ps -ef \| grep defunct<br />ps aux \| grep Z<br />|查看僵尸进程|
2. top 实时显示进程信息

    |选项|作用|
    | ---------| ---------------------------------------------------------------|
    |-b|使用批处理模式输出。<br />和-n一起使用，把 top 命令重定向到文件中|
    |-H|查看线程|
    |-n|设定显示的总次数，完成后将会自动退出|
    |-p PID|仅查看指定 PID 的进程|
    |-d 秒数|指定 top 命令每隔几秒更新（默认3s）|
    |||

    > top -b -H -n 1 -p PID |grep PPID |awk '{print $9}' >> filename
    >
    > 筛选PPID，提取每行的第9列（CPU使用率），重定位到文件中
    >

## 进程信息解读

|指标|含义|
| --------------| ---------------------------------------------------------------|
|users|登录用户数|
|load average|系统平均负载（最近1min、5min、15min，值越大说明系统负载越高）|
|running|运行状态进程数|
|sleeping|休眠状态进程数|
|zombie|僵尸进程数|
|PR|进程优先级|
|NI|进程的谦让度值|
|VIRT/VSZ|占用虚拟内存量|
|RES/RSS|占用物理内存量|
|SHR|共享的内存总量|
|S/STAT|进程状态|
|TIME|进程占用的CPU时间|

## 结束进程kill

常见信号：信号

|命令|功能|
| ---------------------------------| ----------------------------|
|killall http*|kill指定进程名|
|kill -15 PID|kill默认，发送SIGTERM 信号|
|kill -2 PID<br />kill -s SIGINT PID|使用CTRL + C|
|kill -s SIGCHLD PID|kill僵尸进程|

## 进程优先级（+调度策略）

* 普通进程：renice

  -n 5 指定nice值为5

  ```shell
  sudo renice -n 5 PID
  ```

* 实时进程：chrt

  -p 指定PID，-f 修改调度策略为SCHED\_FIFO，修改优先级为10

  ```shell
  chrt -p -f 10 PID

  chrt -p PID
  # 查看进程优先级和调度策略
  ```

## 消息队列

```shell
# 查看消息队列, perms权限, messages消息数量
ipcs -q
```

# 内存

显示内存使用情况（物理内存、交换内存、内核缓冲区内存）

```shell
free -h
```

## 共享内存

```shell
ipcs -m
# 查看共享内存
ipcrm -m [shmid]
# 删除共享内存
```

# 用户与权限

## 创建用户

useradd会在/home目录下创建同名文件夹

```shell
useradd -m 用户名
# -s /bin/bash 指定终端
# -u 指定用户号
# -m 指定用户的登入目录
```

## 设置密码

```shell
passwd 用户名
```

## 切换用户

需要先为用户设置密码

```shell
su root
# 切换用户，默认为root
su - root
# 切换用户，并切换工作目录、环境变量
exit
# 返回root
```

> 1. su: Authentication failure
>
> 使用sudo passwd root，设置root密码后即可切换root

## 文件所有者

```shell
chown zhaosen:zhaosen /data/zhaosen -R
# user:[group] 设置文件所有者和文件关联组
# -c 显示更改部分的信息
# -f 忽略错误信息
# -h 修复符号链接
# -v 显示详细的处理信息
# -R 递归执行
```

## 文件权限

|模式|含义|
| -------| ------------------|
|d|目录|
|r = 4|read 读权限|
|w = 2|write 写权限|
|x = 1|execute 执行权限|
|u|拥有者|
|g|同组用户|
|o|其他用户|
|a|所有用户|

1. 查看文件权限

    ```shell
    ls -l filename
    ll filename
    ```

2. 修改文件权限

    -R：递归修改

    -v：显示修改的详细信息

    -c：仅显示修改的文件

    ```shell
    # 修改文件权限
    chmod -R 777 /opt
    // 777 分别表示文件拥有者/Group/O+ther
    # 添加权限
    chmod +x /opt
    # 移除权限
    chmod a-x filename
    ```

## 免密登录

1. 检测是否安装ssh

    ```shell
    ssh
    ```

2. 客户端生成密钥id_rsa.pub（公钥）

    ```shell
    cd /root/.ssh
    # root用户在/root下创建.ssh
    cd ~/.ssh
    # 普通用户在~下创建.ssh
    ssh-keygen
    ssh-keygen -t rsa -b 4096
    # 限制密钥长度
    ```

3. 复制文件到服务器端

    ```shell
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.1
    # 也可以手动下载 id_rsa.pub 到服务器的对应目录中
    cd .ssh
    # 如果没有authorized_keys文件, touch authorized_keys
    chmod 600 authorized_keys
    chmod 700 .ssh
    cat id_rsa.pub >> authorized_keys
    ```

4. 确定密钥登录功能开启

    ```shell
    sudo vim /etc/ssh/sshd_config
    # RSAAuthentication yes
    # PubkeyAuthentication yes
    # 禁用密码（可选）PasswordAuthentication no
    ```

5. 重启sshd

    ```shell
    sshd restart
    # 或使用 sudo service ssh restart
    ```

6. 检查本地私钥

    Windows端私钥应位于 C:\Users\User Name\.ssh\

    也可以在连接时指定私钥地址

    ```shell
    ssh -i id_rsa remote-username@remote-ip
    ```

     VScode中远程连接需指定私钥地址

# 环境变量

|命令|功能|
| ---------------------------------| --------------|
|export VARIABLE_NAME=value|设置**临时**环境变量|
|echo \$HOME<br />printenv HOME<br />|查看环境变量|
|env|查看**所有**环境变量|
|unset PYTHONHOME|**取消**环境变量|

## 修改位置及方式

```C++
export PATH="$PATH:/opt/au1200_rm/build_tools/bin"
# 推荐
source ~/.bashrc
```

|修改文件|场景|用户|
| ---------------------| --------------------| --------|
|/etc/profile|登录|所有|
|/etc/profile.d|登录时执行的sh脚本||
|/etc/bashrc|shell启动||
|$HOME/.bashrc|shell启动|单用户|
|$HOME/.bash_profile|登录||
|~/.bash_logout|shell退出||

## 常用环境变量

|环境变量|作用||
| -------------------------| --------------------| ----------------|
|PATH|可执行文件搜索路径||
|LIBRARY\_PATH|编译时动态库路径|建议都设置一遍|
|LD\_LIBRARY\_PATH|加载时动态库路径||

# 网络

## 端口开放信息

```shell
# 查看占用某个端口的进程
lsof -i :8080
	-nP 显示 ip 地址和端口号(默认显示别名)

# 列出开放端口及其服务
netstat -atnp | grep 8080
```

|选项|含义|
| ------| ----------------------------------------|
|-a|显示所有sockey, 默认不显示 LISTEN 相关|
|-t|仅显示 tcp 相关|
|-u|仅显示 udp 相关|
|-n|直接使用IP地址，而不通过域名服务器|
|-l|仅列出Listen (监听) 状态的socket|
|-p|显示使用socket的程序|
|-r|显示路由表|
|-e|显示扩展信息，如uid等|
|-s|按协议进行信息统计|
|-c|每隔一个固定时间，执行该 netstat 命令|

# 磁盘

## 挂载

```shell
# 查看挂载设备
df
mount

# 挂载设备
sudo mount /dev/sdb1 /mnt/mydrive
mount /dev/nvme0n1p1 /mnt/nvme0n1p1

# 卸载设备
umount [directory | device ]
```

|mount选项|含义|
| -----------| -------------------------------------------------|
|-t|文件系统类型，如ext4|
|-o|设置读写权限、用户访问，如ro只读，users所有用户|

## inode

```shell
# 查看磁盘inode总数和已使用的数量
df -i

# 查看每个inode大小
sudo dumpe2fs -h /dev/hda | grep "Inode size"

# 查看文件的inode号
ls -i example.txt
ls -i /etc

# 查看文件的inode信息（元信息）
stat file
```

# 文件和目录

## 创建

1. <span data-type="text">创建空文件</span>

    ```shell
    touch fileName
    ```

2. 创建大文件(4MB)

    if 输入文件,of 输出文件,bs 以字节为单位的块大小,count 被复制的块数

    ```shell
    dd if=/dev/zero of=test.data bs=2M count=2
    ```

3. 创建目录

    ```shell
    mkdir New_Dir
    mkdir -p New_Dir/Sub_Dir/Under_Dir
    # 递归创建子目录
    ```

4. 创建临时文件/目录

    /tmp中文件会被系统自动清理

    ```shell
    mktemp 临时文件
    mktemp -d 临时目录
    mktemp -u 仅生成文件名
    mktemp test.XXX 根据模板（至少三个X）创建临时文件
    ```

## 搜索

### 搜索文件find

如果传入绝对路径，find也返回绝对路径。

```shell
# 列出当前目录及子目录下的所有文件和文件夹
find base_path
# 搜索文件名中含有conf的文件
find . -name '*conf*'
# 搜索含有关键词的文件
find .|xargs grep -ri "IBM" -l|sort|uniq
# 最近7天内被访问过的所有文件
find . -type f -atime -7   

# 搜索etc目录下以sh开头的文件,-i忽略大小写
# 仅在/var/lib/slocate 数据库查找，速度更快，但需要系统维护数据库
updatedb  // 手动更新数据库
locate -i /etc/sh
```

|选项|含义|
| --------------------------| -----------------------------------------------------------------------|
|-type|类型，f 普通文件，d目录，l符号链接，c字符设置，b设备，s套接字，f FIFO|
|-name|文件名|
|-iname|忽略大小写的文件名|
|-maxdepth|最大搜索深度，1为在当前目录|
|-mindepth|开始搜索深度，1为从第二级目录开始搜索|
|!|否定参数|
|-atime|访问时间|
|-mtime|修改时间|
|-ctime|元数据修改时间（权限/所有权等）|
|-newer ""|比参考文件更新|
|-size|文件大小|
|-delete|删除匹配文件|
|-perm|文件权限，如644|
|-user|用户名或UID|
|-exec ./cmd.sh {} \\;|执行sh脚本|

### 搜索文本grep

```shell
# 搜索路径下的文件内容匹配关键词的行
grep -r -i 'concurrently' openGaussBase/testcase/
grep -i 'concurrently' file
```

|选项|含义|
| ---------------| ----------------------------|
|关键词+单引号|可以包含空格，不会替换变量|
|关键词+双引号|可以包含空格，替换变量|
|-r|递归|
|-i|不区分大小写|
|-w|匹配整个单词|
|-n|显示对应行号|
|-v|反向搜索|
|-A x|显示匹配行之后的 x 行内容|
|-B x|显示匹配行之前的 x 行内容|
|-C x|显示匹配行前后的 x 行内容|

## 查看

### 查看目录ls

```shell
# 查看当前绝对路径
pwd

# 查看链接文件
ll 链接

# 树状图显示目录
tree -a
```

|ls选项|含义|
| --------| -----------------|
|-a|显示隐藏文件|
|-R|递归|
|-l<br />ll|显示额外信息|
|-h|更友好|
|-F|区分目录、文件|
|-d|只列出目录|
|-i|文件的inode信息|

### 查看文件

* 统计文件中出现次数最多的前10个单词<sup>（每行一个单词）</sup>

  ```shell
  # 对单词进行排序 -> 去重,并在每行行首加上本行在文件中出现的次数
  # -> 按照第一个字段（出现次数），数值排序，且为逆序 -> 取前10行数据
  cat words.txt | sort | uniq -c | sort -k1,1nr | head -10
  ```
* <span data-type="text">查看文件中重复的行</span>

  ```shell
  sort file | uniq -d
  ```

```shell
# 查看开头10行,-f 实时文件（被其他进程使用）
head -n 10 -f log_file

# 查看最后10行,-f 实时文件（被其他进程使用）
tail -n 10 -f log_file

# 每次显示一页文本，空格输出下一页，回车显示下一行
more /etc/bash.bashrc
less /etc/bash.bashrc

# 查看文件类型,-i 查看文件编码方式
file my_file -i

# 查看文件信息
stat file

# 查看文件
cat file
	-n 添加行号
```

|cat选项|含义|
| ---------| ---------------------------------------------------|
|-s|删除连续的空白行|
|-n|增加行号|
|-b|只为非空行加行号|
|-T|将制表符标记为^，可用于排除python语言中的缩进错误|

## 删除

1. 删除文件或目录

    -i 需要确认, -f 强制, -r/R 递归

    ```shell
    rm -ri tmp
    find . -type f -name “*.txt” -print0 | xargs -0 rm -f
    # 将所有的txt文件删除

    uniq -z files.txt | xargs -0 rm
    # 删除files.txt里的文件
    ```

2. 删除空目录

    ```shell
    rmdir New_Dir
    ```

3. 清空文件内容

    ```shell
    truncate -s 0 my_access.log
    cat /dev/null > my_access.log
    echo "" > my_access.log
    ```

4. 删除指定文本(+删除重复行)

    ```shell
    sed -i '/xxx/d' filename
    # 删除包含xxx的行
    tr -d ‘0-9’
    # 删除字符集合（数字）
    tr -d -c ‘0-9 \n’
    # 将除数字、空格、换行符之外的字符删除
    sort | uniq
    # 删除重复的行, -s 跳过前n个字符, -w 用于比较的最大字符数
    ```

## 链接

特点：

* 软/硬链接不占用实际存储空间
* 修改任何一个链接都会改变文件内容（文件同步变化）
* 不能对目录创建硬链接（软链接可以）

1. 创建

    ```shell
    # 创建新的软链接
    ln -s 源文件或目录(绝对路径) 软链接
    cp -s source_file target_file

    # 修改软链接
    ln -snf 源文件或目录(绝对路径) 软链接
    // -f表示强制执行,-n把符号链接视为一般目录
    // 软链接如果不指定，默认为当前路径上创建同名软链接

    # 创建新的硬链接
    ln code_file hl_code_file
    cp -s source_file target_file
    ```

2. 删除

    ```shell
    rm –ri   ./软链接名称  # 删除软链接
    rm -ri    ./软链接名称/   # 删除软链接及其指向内容
    ```

3. 查看

    ```shell
    ls -l /usr/bin/python
    # 查看软链接指向
    ```

## 压缩

|后缀|压缩|解压缩|
| -----------------| ----------------------------------| -------------------------------------|
|.zip|zip mytxt.zip a.txt b.txt c.txt|unzip -d /home/abc/ mytxt.zip|
|.tar|tar -cvf test.tar test/ test2/|tar -xvf test.tar|
|.gz|gzip -c 源文件|gzip -d 压缩包名<br />gunzip 压缩包名<br />|
|.tar.gz<br />.tgz<br />|tar zcvf FileName.tar.gz DirName|tar -zxvf FileName.tar.gz|
|.lz4||lz4 -d|
|.tar.gz|tar -zcvf test.tar.gz file1 dir2|tar -zxvf test.tar.gz|
|.tar.bz2||tar -jxvf xx.tar.bz2|

**tar**：

|选项|含义|
| ------| ----------------------------|
|-z|使用 gzip 来压缩和解压文件|
|-v|--verbose 列出处理的文件|
|-f|必选，使用档案文件或设备|
|-c|创建新的归档（压缩）|
|-x|解压缩|

## 移动（重命名）

```shell
mv openGauss-third_party_binarylibs binarylibs
# 重命名,可以使用正则表达式,如*.log
mv file1.txt /path/to/directory/
# 移动文件

rename .htm .html *.htm
# 原字符串,目标字符串,文件列表

rsync -r source1 source2 destination
rsync -a source destination  # 形成destination/source
rsync -a source/ destination # 将source里的内容同步到destination中
```

mv：

|选项|含义|
| ------| ------------------------------------------------------------|
|-i|覆盖前提示|
|-f|强制覆盖|
|-u|只有当源文件比目标文件新，或者目标文件不存在时，才移动文件|
|-v|显示详细过程|

rsync：

|选项|含义|
| ------| --------------------------------------------------------------|
|-a|保留了所有人和所属组、时间戳、软链接、权限，并以递归模式运行|
|-z|同步时压缩数据|
|-nv|模拟执行，将结果输出到终端|

## 对比

```shell
diff file1 file2
# 对比文件差异
diff -r source destination
# 对比文件夹差异
```

## 复制

```shell
cp source destination -ri
// -i 覆盖文件前提示，-r 递归复制，-p保留文件属性（日期和权限等）,-f 强制覆盖
// -b 覆盖文件前备份原文件，-u source比dest新才执行复制
```

## 分割

```shell
# 以10kB 大小分割文件
split -b 10k file
// -l 10 以行数分割, -d 以数字为后缀, -a 4 指定后缀长度

# 根据文本分割
csplit server.log /SERVER/ -n 2 -s {*} -f server -b “%02d.log”
// /SERVER/ 以字符串为分界符，也可以行数
// -s 安静模式, -z 删除0B文件
// -n 文件名后缀数字个数, -f 指定文件名前缀, -b 指定文件名后缀
// {2} 分割次数（*表示直到文件不可分割为止）
```

## 统计

统计文件中出现次数最多的前10个单词<sup>（每行一个单词）</sup>参考查看文件

```shell
# 统计行数
wc -l 
	-w 显示字符数
	-c 显示bytes字节数

# 统计程序的行数
find path -type f -name “*.c” -print0 | xargs -0 wc -l

# 统计各行在文件中出现的次数
sort file | uniq -c
```

## 排序（去重）

```shell
# 按字符排序
sort file

# 去重
sort file | uniq -c
```

|sort选项|含义|
| ----------| ----------------------------------|
|-c|统计重复行次数|
|-n|按数字排序，默认以字符形式<br />|
|-M|按月排序（能识别三字符的月份名）|
|-k|指定排序的字段，从1开始<sup>（-k1表示以空格分割的第一列）</sup>|
|-t|指定字段分隔符，例如':'|
|-r|降序排列|

|uniq选项|含义|
| ----------| ----------------------------------------|
|-f|比较时跳过前n列|
|-c|显示行重复的次数|
|-u|只显示出现一次的行（重复行全部不显示）|
|-d|只显示重复的行|
|-s|跳过前n个字符|
|-w|用于比较的最大字符数|

# 软件

## ubuntu（apt安装）

* ```shell
  sudo apt install softName
  ```

### 设置镜像源

```shell
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo gedit /etc/apt/sources.list
sudo apt-get update
sudo apt-get upgrade

# 阿里源（任选一个）
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

## centos（yum安装）

### 设置镜像源

```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
# 阿里源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 网易源（任选一个）
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
yum clean all
yum makecache
yum update
```

## 下载文件wget/curl

```shell
sudo apt install wget
wget -c -q URL -O fileName
# -c 支持断点续传, -q 安静模式, -O 下载后改名
# 如果出现Connecting to xxxxxxxx:443，添加--no-cookie --no-check-certificate

curl [option] URL
```

## 软件下载位置

```shell
which 软件名
rpm -qa|grep redis
# 查看rpm包版本
dpkg -l | grep ruby
# 查看deb包版本
yum list installed | grep ruby
# 查看yum安装的包版本
rpm -ql redis-3.2.10-2.el7.x86_64
# 查看安装路径
```

# shell

## ssh远程连接

```shell
ssh 用户名@192.168.0.1 -p 22
```

## 后台运行

tmux重启后不会保存，可能出现 failed to connect to server: Connection refused：

[自动保存tmux会话 关机重启再也不怕 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/146544540)

```shell
# tmux
sudo yum install tmux
tmux new -s <session-name>
tmux detach 或 ctrl+B D
tmux attach -t <session-name>
tmux ls
tmux kill-session -t <session-name>
tmux switch -t <session-name>
tmux rename-session -t 0 <new-name>

# 后台运行，但控制台关闭（退出账户）时，会停止运行
cmd &

# 退出账号后，也会继续运行
nohup cmd &
nohup cmd >/dev/null 2>&1 &
```

‍
