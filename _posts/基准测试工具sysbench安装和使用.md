---
date: 2015-05-17 12:32
status: public
author: Haijin Ma
title: 【原创】基准测试工具sysbench安装和使用
categories: [基准测试]
tags: [sysbench,MySQL]
---

## sysbench介绍
SysBench是一个模块化的、跨平台、多线程基准测试工具，主要用于评估测试各种不同系统参数下的数据库负载情况。它主要包括以下几种方式的测试：
1. cpu性能
2. 磁盘io性能
3. 线程调度性能
4. 互斥锁性能
5. 数据库性能(OLTP基准测试)
6. 内存性能

目前sysbench主要支持 MySQL、pgsq、Oracle 这3种数据库。
  sysbench在github的版本分为两个分支：0.4和0.5。0.4的最新版本是0.4.12，也是我使用的版本。
> github：https://github.com/akopytov/sysbench

## 安装环境
操作系统：VMware虚拟CentOS 7
数据库：MySQL 5.6.24

## 安装步骤
1. github下载sysbench的源码zip包：sysbench-0.4.zip到本地目录，如：/home/mahj/software目录下；
2. 在/home/mahj/software目录下运行命令：unzip sysbench-0.4.zip，解压zip包，会生成sysbench-0.4目录；
3. 进入sysbench-0.4目录，运行命令：./autogen.sh。这一步可能会报错：automake 1.10.x (aclocal) wasn't found, exiting。这说明你的操作系统没有安装automake，运行命令：yum install automake.noarch，即可安装。然后再运行./autogen.sh命令，又报错：libtoolize 1.4+ wasn't found, exiting。说明你的操作系统没有安装libtool，运行命令：yum install libtool，即可安装。继续运行
4. 运行make命令，可能会报错：drv_mysql.c:35:19: fatal error: mysql.h: No such file or directory。这是因为，你本机没有安装mysql的开发lib库导致的，运行命令：yum install mysql-community-devel.x86_64，即可安装。然后运行make命令，即可正确。
5. 运行make install；
6. 运行sysbench --help测试安装是否正常。

命令运行过程如下：
```SHELL
[root@bogon sysbench-0.4]# sh autogen.sh
automake 1.10.x (aclocal) wasn't found, exiting
[root@bogon sysbench-0.4]# yum install automake.noarch
[root@bogon sysbench-0.4]# sh autogen.sh
libtoolize 1.4+ wasn't found, exiting
[root@bogon sysbench-0.4]# yum install libtool
[root@bogon sysbench-0.4]# sh autogen.sh
[root@bogon sysbench-0.4]# make
Making all in doc
make[1]: Entering directory `/home/mahj/software/sysbench-0.4/doc'
Making all in xsl
make[2]: Entering directory `/home/mahj/software/sysbench-0.4/doc/xsl'
make[2]: Nothing to be done for `all'.
make[2]: Leaving directory `/home/mahj/software/sysbench-0.4/doc/xsl'
make[2]: Entering directory `/home/mahj/software/sysbench-0.4/doc'
touch manual.html
make[2]: Leaving directory `/home/mahj/software/sysbench-0.4/doc'
make[1]: Leaving directory `/home/mahj/software/sysbench-0.4/doc'
Making all in sysbench
make[1]: Entering directory `/home/mahj/software/sysbench-0.4/sysbench'
Making all in drivers
make[2]: Entering directory `/home/mahj/software/sysbench-0.4/sysbench/drivers'
Making all in mysql
make[3]: Entering directory `/home/mahj/software/sysbench-0.4/sysbench/drivers/mysql'
gcc -DHAVE_CONFIG_H -I. -I../../../config  -I/usr/include/mysql -g -pipe -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches   -m64 -fPIC  -g -fabi-version=2 -fno-omit-frame-pointer -fno-strict-aliasing -DMY_PTHREAD_FASTMUTEX=1 -I../../../sysbench -D_XOPEN_SOURCE=500 -D_GNU_SOURCE  -W -Wall -Wextra -Wpointer-arith -Wbad-function-cast   -Wstrict-prototypes -Wnested-externs -Winline   -funroll-loops  -Wundef -Wstrict-prototypes -Wmissing-prototypes -Wmissing-declarations -Wredundant-decls -Wcast-align        -pthread -O2 -ggdb3  -MT libsbmysql_a-drv_mysql.o -MD -MP -MF .deps/libsbmysql_a-drv_mysql.Tpo -c -o libsbmysql_a-drv_mysql.o `test -f 'drv_mysql.c' || echo './'`drv_mysql.c
drv_mysql.c:35:19: fatal error: mysql.h: No such file or directory
 #include <mysql.h>
                   ^
compilation terminated.
make[3]: *** [libsbmysql_a-drv_mysql.o] Error 1
make[3]: Leaving directory `/home/mahj/software/sysbench-0.4/sysbench/drivers/mysql'
make[2]: *** [all-recursive] Error 1
make[2]: Leaving directory `/home/mahj/software/sysbench-0.4/sysbench/drivers'
make[1]: *** [all-recursive] Error 1
make[1]: Leaving directory `/home/mahj/software/sysbench-0.4/sysbench'
make: *** [all-recursive] Error 1
[root@bogon sysbench-0.4]# yum install mysql-community-devel.x86_64
[root@bogon sysbench-0.4]# make
[root@bogon sysbench-0.4]# make install
[root@bogon sysbench-0.4]# sysbench --help
```

## 常用测试命令
1. 帮助信息
        sysbench --help
2. CPU测试
         sysbench --test=cpu --cpu-max-prime=2000 run
         说明：通过计算2000以内的素数来测试CPU性能。
3. 文件IO测试
  - 准备文件数据
         sysbench --test=fileio --num-threads=16 --file-total-size=10G prepare
  - 运行测试
         sysbench --test=fileio --num-threads=16 --init-rng=on --file-total-size=5G --file-test-mode=rndrw run
  - 读写模式（file-test-mode）包括：
         seqwr 顺序写入
         seqrewr 顺序重写
         seqrd 顺序读取
         rndrd 随机读取
         rndwr 随机写入
         rndrw 混合随机读/写
  - 清理测试准备的文件
         sysbench --test=fileio --num-threads=16 --file-total-size=10G cleanup
4. 线程调度测试
       sysbench --test=threads --num-threads=64 --thread-yields=100 --thread-locks=2 run
5. 互斥锁测试
       sysbench --test=mutex --num-threads=16 --mutex-num=1024 --mutex-locks=10000 --mutex-loops=5000 run
       说明：模拟所有线程在同一时刻并发运行，并都短暂请求互斥锁。
6. 内存测试
       sysbench --test=memory --num-threads=16 --memory-block-size=8192 --memory-total-size=1G run
       说明：测试内存的连续读写性能。
7. MySQL数据库测试
- 准备表
      sysbench --test=oltp --oltp-table-size=1000000 --mysql-db=app-db --mysql-user=appuser prepare
      说明：用appuser登陆，并在app-db数据库创建sbtest表，其中包含100000万条记录。
- 运行测试
      sysbench --test=oltp --oltp-table-size=1000000 --mysql-db=app-db --mysql-user=appuser --max-time=60 --oltp-read-only=on --num-threads=8 run
      说明：起8个线程，开启只读模式，执行60秒的测试。

