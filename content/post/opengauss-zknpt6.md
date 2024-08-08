---
title: openGauss
slug: opengauss-zknpt6
url: /post/opengauss-zknpt6.html
date: '2024-07-14 21:30:43+08:00'
lastmod: '2024-08-08 13:29:59+08:00'
toc: true
tags:
  - openGauss
  - 数据库
categories:
  - 工具
keywords: openGauss,数据库
isCJKLanguage: true
---

# openGauss

# 编译安装

## 环境变量

```sh
#!/bin/bash
export CODE_BASE="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
export BINARYLIBS=/mnt/nvme0n1p1/gcc10/binarylibs
export GAUSSHOME=$CODE_BASE/dest/
export GCC_PATH=$BINARYLIBS/buildtools/gcc10.3/
export CC=$GCC_PATH/gcc/bin/gcc
export CXX=$GCC_PATH/gcc/bin/g++
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$GCC_PATH/gcc/lib64:$GCC_PATH/isl/lib:$GCC_PATH/mpc/lib/:$GCC_PATH/mpfr/lib/:$GCC_PATH/gmp/lib/:$LD_LIBRARY_PATH
export C_INCLUDE_PATH=$GAUSSHOME/include:$GAUSSHOME/include/postgresql/server:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$GAUSSHOME/include:$GAUSSHOME/include/postgresql/server:$CPLUS_INCLUDE_PATH
export PATH=$GAUSSHOME/bin:$GCC_PATH/gcc/bin:$PATH
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$GAUSSHOME/install/geos/lib:$GAUSSHOME/install/proj4/lib:$GAUSSHOME/install/gdal/lib:$GAUSSHOME/install/libxml2/lib/:$LD_LIBRARY_PATH
export PGTEMPPORT=6993
export PGPORT=6992
export L_PGPORT=7993
export R_PGPORT=60032
export LPORT=50036
export LHPORT=50034
export LSERVICE=50035
export RPORT=60036
export RHPORT=60034
export RSERVICE=60035
export DATA_SOURCE_NAME="host=127.0.0.1 user=sysbench password=12345abc! port=6992 dbname=sysbench sslmode=disable"
```

## 依赖项目

```sh
```

## 编译命令

```shell
source env.sh

# release版本（gcc也可为7.3.0）
./configure --gcc-version=10.3.0 CC=g++ CFLAGS="-O2 -g3" --prefix=$GAUSSHOME --3rd=$BINARYLIBS --enable-thread-safety --with-readline --without-zlib

# debug版本（gcc也可为7.3.0）
./configure --gcc-version=10.3.0 CC=g++ CFLAGS='-O0' --prefix=$GAUSSHOME --3rd=$BINARYLIBS --enable-debug --enable-cassert --enable-thread-safety --with-readline --without-zlib

make uninstall
make clean
make -sj 2 2>&1|grep rror
make install -sj 2
```

## 创建数据库

```shell
rm -rf $GAUSSHOME/data
mkdir $GAUSSHOME/data
gs_initdb -D $GAUSSHOME/data --nodename=datanode1 -wQwer1234
```

postgresql.conf中设置

```shell
enable_default_ustore_table=on
shared_buffers = 内存的30%~40%
undo_zone_count=16384
password_encryption_type = 0
```

pg_hba.conf中设置

```shell
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             tpcc            10.19.0.120/32            md5
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
```

# 启动

```shell
source env.sh
gs_ctl start -D $GAUSSHOME/data
gsql -d postgres -p $PGPORT -r
```

# YAT测试

> 编译安装需要网络支持

1. 创建调度文件
2. 开始测试

```c++
yat suite run -s ./schedule/testschedule.schd
```

# 创建online索引

```c++
create table test_idx(id serial primary key, note text) with (storage_type=astore);
create table test_idx(id serial primary key, note text) with (storage_type=ustore);
insert into test_idx(note) select generate_series(1,500000);
create index concurrently idx_test_idx2 on test_idx(note);
select oid from pg_class where relname='idx_test_idx2';
select pg_size_pretty(pg_indexes_size('test_idx'));
drop index idx_test_idx2;
REINDEX INDEX CONCURRENTLY idx_test_idx2;
drop table test_idx;
```

## 验证online索引创建

> 只有当执行SQL才会分配事务ID，不会在begin时分配

|Session1|Session2||
| ------------------| ---------------| --|
||Begin开启事务||
|创建索引|||
||增加insert||
||删除delete||
||查询select||
||End结束事务||
|等待索引创建完成|||
||查看查询计划||
||查询select||

```c++
begin;
copy test_idx from stdin delimiter ',' csv;
500001,500001
\.
delete from test_idx where id = 5;
explain select note from test_idx where note='5';
```

# 查看oid

```SQL
# 表或索引的oid
select oid from pg_class where relname='foo';

# 数据库的oid
select oid,datname from pg_database where datname='syd';
```

# 长事务（+数据导入）

```SQL
begin;
copy test_idx from stdin delimiter ',' csv;
50001,50001
\.
```

# 查看索引状态

[Pg Index (osinfra.cn)](https://docs-opengauss.osinfra.cn/zh/docs/2.0.0/docs/Developerguide/PG_INDEX.html)

```SQL
# 查看索引状态
select T.* from pg_index T where indrelid=(select oid from pg_class where relname='test_idx');
```

# 页面解析pagehack

```shell
# 编译
cd $CODE_BASE/contrib/pagehack
make

# 查找oid
select oid from pg_class where relname='foo';
find . -name 16502

# 解析表页面
$CODE_BASE/contrib/pagehack/pagehack -f 16502 -t uheap -n 4
# 解析索引页面
$CODE_BASE/contrib/pagehack/pagehack -f ./dest/data/base/15702/49210 -t btree_index -n 2
```
