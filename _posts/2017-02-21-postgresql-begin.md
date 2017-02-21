---
layout: post
title: PostgreSql内核分析入门介绍
date: 2017-02-21
category: Storage
tags: postgresql
---


近期分析PostgreSql 9.2 (PG)的内核， 记录一些常用的系统命令、系统表和一些函数工具等方便后续分析，工欲善其事，必先利其器。

一、常用的基本命令
登入PG：  psql -U pguser  -d testdb -h ip地址 -p 5432
常用命令：
\?： 会列出所有命令；
\l： 显示所有的数据库；
\c 数据库名:  连接某个数据库；
\dt 表名(可含通配符)： 列出所有满足条件的表信息；
\di 索引名(可含通配符)： 列出所有满足条件的索引信息；
\d+ 表名或索引名(可含通配符)： 显示表或索引的定义；
\q： 退出；

二、常用的术语
tuple: 元祖，等同于行；
releation: 等同于表；
filenode: 是个id，标识了唯一的一个表或索引；
block/page: 块/页，默认为8K大小，数据是分块存储的；
heap/heap page:存放用户数据的文件，包含了多个页，不同于操作系统里heap；
ctid: 标识一个位置信息，包含了页号和页内偏移等信息；
oid:  object identifier， 对象标识符；
    
三、用户表的基本信息存在哪里？
    用户表的基本信息保存在系统表里，涉及到的系统表很多，此处列举一些比较常用的。现创建了一个名为t4的用户表，表结构和索引如下：

后续的举例都基于这个表。

1. pg_database
pg_database存储了各个数据库的信息，其中比较常用的字段有oid，datname（数据库名）， encoding(编码方式）等。

例如可以查看整个PG中有哪些数据库，oid是隐藏的列，可以直接通过select找出来。后面分析物理文件存储在哪里的时候需要用到这个系统表。关于这些字段的具体含义可参考 https://www.postgresql.org/docs/9.2/static/catalog-pg-database.html 


2. pg_class
pg_class存储了各个表、视图、和索引的信息。

常用的列有：
relname: 表名、视图名、索引名等;
relfilenode:  表或索引所在的oid；
relpages:已分配了多少物理页，该值不是确切值；
reltupes: 当前有多少行数据，该值也不是确切值；
relnatts:表有多少列（不包含系统添加的隐藏列）；
relkind:表的类型，r表示普通的表， i表示是索引，v表示是视图，t表示是toast表；
relhashpkey：是否有主键, t表示true。
其它字段的含义可参考官方文档：https://www.postgresql.org/docs/9.2/static/catalog-pg-class.html
可以看下t4和它的索引在pg_class中的信息：


3. pg_attribute
    存储了各个表或索引中各个列的信息。

常用字段及其含义如下：
attrelid: 该列所属哪个表，同pg_class中的relfilenode;
attname: 列名；
attlen：该列占用的长度，如果是定长的话，会有一个正整数的值，如果是varchar等变长的，用-1表示；
attnum: 该列在表中的顺序，如果是系统添加的隐藏列，用负数表示。
其它字段的信息可参考：https://www.postgresql.org/docs/9.2/static/catalog-pg-attribute.html

可以看到t4表的各个字段的信息，一共有10列，6个隐藏列，这些隐藏列不一定会真正占用物理空间，比如cmin和cmax就是占用的同一块物理空间。

下图是t4的一个组合索引的情况，该组合素有一共有两个列组成。attrelid表示对应索引的filenode。


4. pg_index
    存储了索引相关的信息。

常用字段及其含义如下：
indexrelid: 索引的filenode；
indrelid: 该索引所属表的filenode；
indnatts: 该索引包含了多少列；
indisunique: 该索引是否为唯一索引；
indisprimary: 该索引是否为主索引；
indkey: 同pg_attribute中的attnum值，表示是表中的第几个字段。
其它字段的具体含义参考：https://www.postgresql.org/docs/9.2/static/catalog-pg-index.html
可以看下t4表的索引信息：


5. 统计视图
    上面介绍的都是表，现在介绍一个统计相关的视图pg_stat_user_tables. PG中有很多监控统计的视图，具体可以参考https://www.postgresql.org/docs/9.2/static/monitoring-stats.html。

该表主要包含了当前有多少行数据，增、删、改了多少行， vacuum相关的时间等。各个字段的含义比较好理解，看个例子：


四、内核分析的相关函数
pageinspect模块中的一些方法可以用来分析各种页中的内容，查看各种数据结构的取值等。要使用该模块时，需要先 CREATE EXTENSION  pageinspect;然后再使用。
get_raw_page（表名，页号）函数：返回每个页的内容，一般和下面的函数结合使用；
page_header 函数： 返回各个页中PageHeader结构体中的内容;
heap_page_items 函数：返回heap page中各行数据的信息；可看下图示例：


bt_metap（索引名） 函数： 查看索引页中meta page中的内容。关于索引名，如果是primary key，没有设置索引名的话，默认的名字为 表名_pkey；如：

bt_page_stats（索引名、页号） 函数： 查看索引页中的一些概要信息；如：

bt_page_items（索引名、页号） 函数：查看索引页中各个数据的信息；如：


fsm_page_contents 函数： 查看fsm page页中各个字节的取值；
关于这些函数的具体信息，可参考https://www.postgresql.org/docs/9.2/static/pageinspect.html 。
此外， 查看fsm的信息，也可以通过freespacemap模块来查看(该模块需要额外安装)，使用前先 CREATE EXTENSION pg_freespacemap; 即可。如：


五、用户数据存在哪里？
PG中数据文件存放的默认路径为/var/lib/pgsql/data， 可以通过postgresql.conf来配置。该目录下面包含以下文件和目录。其中：

base: 包含了每个数据库的数据；
global：包含了集群范围的表的数据；
pg_clog: 事务提交的状态数据；
pg_xlog: WAL（write ahead log），用户增、删、改前都需要记录日志，这部分日志就存储在这个目录里；
pg_log: 数据库运行日志，和pg_clog, pg_xlog完全没关系；
pg_subtrans： 包含了子事务的状态数据；
postgresql.con： 包含了PG的配置。
其它文件和目录的含义见 https://www.postgresql.org/docs/9.2/static/storage-file-layout.html。

用户数据存储在base目录下，base目录下包含了多个名字为数字的目录。这些数字表示的是数据库的oid，可以通过pg_database来查看分别对应的是哪个数据库。进入到某个数据库后，就是该库所有的表文件，这些文件名也都是数字，通过pg_class的filenode可以知道各个数字对应的是哪个表、哪个索引等。如下示意图是表t4所对应的数据文件，每个表除了表本身有个文件外，还有一个fsm和vm文件与之对应。索引文件没有fsm和vm，关于fsm和vm文件后续再介绍。

由于通过pg_database和pg_class 去查数据库、表对应的oid不是很方便，因此PG又提供了一个命令oid2name, 用于将oid转成名字，如下图所示。


六、内核相关的资源整理
Bruce monjian:  Postgres Internals Presentations
https://momjian.us/main/presentations/internals.html

A Tour of PostgreSQL Internals
https://www.postgresql.org/files/developer/tour.pdf

Following a Select Statement Through Postgres Internals
http://patshaughnessy.net/2014/10/13/following-a-select-statement-through-postgres-internals

http://patshaughnessy.net/2014/11/11/discovering-the-computer-science-behind-postgres-indexes

Introduction to PostgreSQL physical storage
http://rachbelaid.com/introduction-to-postgres-physical-storage/

New Free Space Map and  Visibility Map 
https://wiki.postgresql.org/images/8/81/FSM_and_Visibility_Map.pdf

PGConf2016: Index Internals (PConf EU,US,硅谷, 下文是Russia开的,heikki)
https://www.pgcon.org/2016/schedule/attachments/434_Index-internals-PGCon2016.pdf


最后
本文介绍了一些内核分析会涉及到的系统表、函数等，为后续分析PG内核做些准备，下一步将会分析PG存储中涉及到的数据结构。
