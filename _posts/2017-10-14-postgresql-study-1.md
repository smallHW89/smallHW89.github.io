---
date: 2017-10-14 13:42
status: public
title:  postgresql 学习-1 
---

## 版本、环境
ubunntu 16.04
postgresql  9.6.2


##  源码编译安装、运行、创建DB

参考文章: http://www.cnblogs.com/zhoulf/p/4040768.html

注意：用户名不一定必须是postgresql, 

启动postgresql后，用客户端psql连接 本地服务器进行操作

如果./psql 提示找不到对应的socket域文件，可以查看下postgresql.conf下面socket域的目录，用./psql --host=xxx来指定socket域文件路径

psql 的命令与mysql不一样，

例如：\d  显示当前DataBase的所有table, \l 显示所有的DataBase， 
参考: http://blog.csdn.net/smstong/article/details/17138355


SQL语法和mysql差时一样的，按标准,

例如 ： select * from  xxx_table ;

## postgresql 调试 

参考文章： http://www.cnblogs.com/flying-tiger/p/5866660.html

主要是编译时启用--debug_enbale ,修改Makefile文件 去掉-O2等优化。

```
(gdb) bt
#0  0x00007f786ab9a9b3 in __epoll_wait_nocancel () at ../sysdeps/unix/syscall-template.S:84
#1  0x00000000006a8526 in WaitEventSetWait ()
#2  0x00000000005ec35e in secure_read ()
#3  0x00000000005f4328 in pq_recvbuf ()
#4  0x00000000005f50f5 in pq_getbyte ()
#5  0x00000000006c7e45 in PostgresMain ()
#6  0x000000000046cede in ServerLoop ()
#7  0x000000000066c5c1 in PostmasterMain ()
#8  0x000000000046dee0 in main ()
```

## postgresql  源码结构

参考文章： http://blog.csdn.net/jpzhu16/article/details/52421294

这里比较核心的是backend,bin,interface这几个目录。Backend是对应于后端，bin和interface对应于前端。

bin里面有pgsql,initdb,pg_dump等各种工具的代码

interface里面有PostgreSQL的C语言的库libpq,另外可以在C里嵌入SQL的ECPG命令的相关代码。

backend 目录结构如下：

access:   各种存储访问方法(在各个子目录下) common(共同函数)、gin (Generalized Inverted Index通用逆向索引), gist (Generalized  Search Tree通用索引)、 hash (哈希索引)、heap (heap的访问方法)、index (通用索引函数)、 nbtree (Btree函数)、transam (事务处理)

bootstrap:  数据库的初始化处理(initdb的时候)

catalog: 系统目录

commands: SELECT/INSERT/UPDATE/DELETE以为的SQL文的处理

executor: 执行器
  
foreign:  FDW(Foreign Data Wrapper)处理

ibpq:  前端/后端通信处理

main:  postgresql的主函数
    
nodes:  构文树节点相关的处理函数

optimizer: 优化器

parser: SQL构文解析器

port:       平台相关的代码

postmaster: postmaster的主函数 (常驻postgres)

replication: streaming replication

regex:     正则处理

## postgresql 的系统结构

参考文章: 
https://wiki.postgresql.org/wiki/Pgsrcstructure  \
http://www.cnblogs.com/songyuejie/p/3911059.html \
http://www.cnblogs.com/songyuejie/p/3910970.html



