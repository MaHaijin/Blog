---
date: 2015-01-20 21:46
status: public
author: Haijin Ma
title: '【原创】MySQL服务器性能优化'
categories: [数据库]
tags: [MySQL]
---
##服务器性能剖析

**第一原则：性能即响应时间。**

**第二原则：无法测量就无法有效的优化**
1. 时间花在哪？
2. 时间为什么花在那？

完成任务的时间分为：执行时间和等待时间。

性能剖析分为两步：
1. 测量任务所花费的时间；
2. 对结果进行统计和排序，将耗费昂贵的任务排在前边。

性能剖析分类：
1. 基于执行时间的分析；
   研究什么任务的执行时间最长？
2. 基于等待时间的分析。
   判断任务在什么地方被阻塞的时间最长？

优化查询注意两点：
1. 一些只占总响应时间5%的查询是不值得优化的；
   阿姆达尔定律：对一个占总响应时间不超过5%的查询进行优化，无论如何努力，收益也不会超过5%。
2. 如果优化的成本大于收益，就应当停止优化。

优化查询分两种场景：
1. 查询次数很多，但是每次查询耗时较少；
   优先考虑降低查询次数。
2. 查询次数很少，但是每次查询耗时很多。
   优化查询

性能剖析遵循自上而下的原则：应用程序-数据库-操作系统。

剖析工具：New Relic、xhprof（For PHP）

MySQL慢查询日志，如果设置long_query_time=0表示捕获所有查询，时间单位可以精确到微秒级别。一般只在收集负载样本时开启慢查询日志，如果一直开启，日志量会很大。
MyQL慢查询日志分析工具：pt-query-digest(分析tcpdump)

在MySQL5.6，profile已经过期了，取而代之的是performance_schema。

诊断间歇性问题：
1. 确认是单条查询问题还是服务器问题？
   - 使用Show Global Status
   - 使用shwo processlist
   - 使用查询日志
2. 捕获诊断数据


解决他人提出的技术问题的步骤：
1. 弄清楚问题是什么？一定要能清晰的描述出来。
2. 为了解决这个问题，已经做过什么操作了。

---
##Schema与数据类型优化
数据库设计数据：Beginning Databse Design

数据库设计原则：
1. 更小的通常更好
   尽量使用能正确存储数据的最小数据类型。
2. 简单就好
   整形比字符型简单。
3. 尽量避免Null
   通常最好指定列为非Null。

Timestamp只使用DateTime一半的存储空间，并且是带时区的，允许的时间范围要小得多。

Varchar类型是变长字符串，其需要用1或2个字节存储字符串长度，如果字符串长度小于或等于255，则使用1个字节表示长度；否则用2个字节。如：varchar(10)占用11个字节；varchar(1000)占用1002个字节。
使用变长是为了节省空间。但是变长字符串在做更新操作时，可能会遇到比原来长度更长的情况，这个时候需要做些额外处理，Innodb是分裂页。

Varchar适用场景：字符串的最大长度比平均长度要大得多；列的更新很少；使用了像UTF-8这种复杂的字符集，每个字符都是用不同的字节进行存储。

Char类型是定长字符串，非常适合存储MD5值这种定长的字段。

Blob类型家族是二进制存储，Text类型家族是字符串存储。

max_heap_table_size和tmp_table_size控制内存临时表的大小，超过这个大小则转为磁盘临时表。

DateTime与时区无关，表示范围：1001-9999年，精度为秒；使用8个字节。
TimeStatmp和时区相关，表示范围：1970-2038年，精度为秒；使用4个字节存储。
Mysql默认会设置第一个TimeStamp为当前时间，更新的时候也会默认更新第一个Timestamp的值为当前时间。

Schema设计陷阱：
1. 太多的列
   因为Mysql服务器转换来自存储引擎的行缓冲为行数据格式的CPU开销很大，而转换的代价依赖于列数。

2. 关联太多的表
   Mysql支持一个查询最多关联61张表，但根据经验，如果考虑并发性和查询速度，最好不要超过12张表。

范式化的schema设计的一个缺点是会导致一个查询会有很多关联。
如果不需要关联表，则对大部分最坏的情况：表没有使用索引或者是全表扫描，当数据比内存大的时候，全表扫描可能比关联很多表要更快，因为避免了随机IO。

全表扫扫描一般是顺序IO，依赖于具体的存储引擎。

提升读性能的手段：缓存表、汇总表、影子表、物化视图、计数器表

---
## 创建高性能索引
复合索引字段的顺序很重要，因为MySQL只能高效的使用索引最左前缀列。
索引类型：
1. B-Tree索引
   实际上大部分存储引擎使用的是B+Tree数据结构，比如InnoDB。
2. 哈希索引
   SHA1和MD5是强加密函数，设计目标是最大限度消除冲突。
3. R-Tree索引
4. 全文索引

索引的有点：
1. 索引大大减少了服务器需要扫描的数据量；
2. 索引可以帮助服务器避免排序和临时表；
3. 索引可以将随机IO变为顺序IO。

对于B-Tree索引，需要注意索引的顺序。索引列排序是从左到右进行排序。一般将选择性最高的列放于最左边。

如果数据都是放在内存中，那么聚族索引并没有优势。


按索引顺序读取数据比顺序的全表扫描慢。

Limit会限制mysql查询的数据集大小。

##查询性能优化
缓存很重要
衡量mysql性能的最简单的三个指标：
响应时间
扫描的行数
返回的行数

###使用where的三种方式好坏依次为：
1. 在索引中使用where条件来过滤不匹配的行，这时在存储引擎层来完成的。
2. 使用索引覆盖扫描（在执行计划的Extra中出现using index）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这时在MySQL服务器层来完成的，无须再回表查询记录。
3. 从数据表中返回数据，然后过滤不满足条件的记录（在执行计划的Extra中出现Using where）。这时在MySQL服务器层来完成的，MySQL需要从数据表读取记录然后过滤。

MySQL的性能指标参考：
一个通用服务器可能也能运行每秒10W的查询。
一个千兆网卡能轻松满足每秒2000次查询。
MySQL每秒能够扫描内存中上百万行数据。
一次删除1W行数据一般来说是一个比较高效的的做法。

###重构查询的方式
1. 将复杂查询分为多个简单查询。
2. 切分为小查询，每个查询只负责一部分数据的查询。
3. 分解关联查询，将合并动作放在应用程序中去做。
   好处：
   1. 让缓存更高效；
   2. 减少锁竞争；
   3. 在应用层做关联，更容易对数据库进行拆分和扩展；
   4. 查询本身效率也会有所提升；
   5. 相当于用哈希关联来替代MySQL的嵌套循环关联。


###MySQL查询原理基础
1. MySQL客户端和服务端的通信是半双工的。这意味着，当客户端发送数据给服务端时，服务端只能等待，不能干别的，并且客户端发送给服务端的一般是一个查询SQL的包；同理，服务端发送结果给客户端，一般是由多个包组成，客户端必须等待接收完全部数据，除非主动断开连接。这就是Limit存在的意义所在。
2. 对于一个MySQL连接，或者说一个线程来说，任何时刻都有一个状态，该状态表明了MySQL当前正在干什么。状态如下：
   1. Sleep
   2. Query
   3. Locked
   4. Analyzing and statistics
   5. Coping to tmp table[on disk]
   6. Sorting result
   7. Seneing data


#MySQL分区表
