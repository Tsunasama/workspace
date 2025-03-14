## INNODB

> innodb 核心竞争力: bufferpool,mvvc,事务,行锁,崩溃恢复(redolog和 undolog)

##### * 资料

* [庖丁解INNODB](http://catkang.github.io/2020/02/27/mysql-redo.html)
* [mysql日志篇](https://zhuanlan.zhihu.com/p/652252941)
* [深入浅出Mysql InnoDb核心竞争力 BufferPool (1)](https://zhuanlan.zhihu.com/p/691785266)

##### * 数据存储格式![1728716775884](image/mysql/1728716775884.png)

##### * Page存储格式![1728717954701](image/mysql/1728717954701.png)


##### * Buffer Pool 刷脏时机

> 引发数据库flush脏页的四种典型场景各是什么?对mysql性能的影响各是怎样的? 缓存池中的内存页有哪三种状态? 哪两种刷脏页的情况会比较影响性能?

答：四种场景如下:

* 第一种场景是，粉板满了，记不下了。这时候如果再有人来赊账，掌柜就只得放下手里的活儿，将粉板上的记录擦掉一些，留出空位以便继续记账。当然在擦掉之前，他必须先将正确的账目记录到账本中才行。

　　　　 **这个场景，对应的就是 InnoDB 的 redo log 写满了。这时候系统会停止所有更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写。** 画了一个 redo log 的示意图，这里我改成环形，便于大家理解。

　　![1729266698668](image/mysql/1729266698668.png)

* 第二种场景是，这一天生意太好，要记住的事情太多，掌柜发现自己快记不住了，赶紧找出账本把孔乙己这笔账先加进去。

　　**这种场景，对应的就是系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。**

你一定会说，这时候难道不能直接把内存淘汰掉，下次需要请求的时候，从磁盘读入数据页，然后拿 redo log 出来应用不就行了？这里其实是从性能考虑的。如果刷脏页一定会写盘，就保证了每个数据页有两种状态：

* * 一种是内存里存在，内存里就肯定是正确的结果，直接返回；
  * 另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高。
* 第三种场景是，生意不忙的时候，或者打烊之后。这时候柜台没事，掌柜闲着也是闲着，不如更新账本。

**这种场景，对应的就是 MySQL 认为系统“空闲”的时候。当然，MySQL“这家酒店”的生意好起来可是会很快就能把粉板记满的，所以“掌柜”要合理地安排时间，即使是“生意好”的时候，也要见缝插针地找时间，只要有机会就刷一点“脏页”。**

* 第四种场景是，年底了咸亨酒店要关门几天，需要把账结清一下。这时候掌柜要把所有账都记到账本上，这样过完年重新开张的时候，就能就着账本明确账目情况了。

**这种场景，对应的就是 MySQL 正常关闭的情况。这时候，MySQL 会把内存的脏页都 flush 到磁盘上，这样下次 MySQL 启动的时候，就可以直接从磁盘上读数据，启动速度会很快。**

上面四种场景对性能的影响如下：

　　其中，第三种情况是属于 MySQL 空闲时的操作，这时系统没什么压力，而第四种场景是数据库本来就要关闭了。这两种情况下，你不会太关注“性能”问题。所以这里，我们主要来分析一下前两种场景下的性能问题。

　　第一种是“redo log 写满了，要 flush 脏页”，这种情况是 InnoDB 要尽量避免的。因为出现这种情况的时候，整个系统就不能再接受更新了，所有的更新都必须堵住。如果你从监控上看，这时候更新数会跌为 0。

　　第二种是“内存不够用了，要先将脏页写到磁盘”，这种情况其实是常态。 **InnoDB 用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态** ：

* * 第一种是，还没有使用的；
  * 第二种是，使用了并且是干净页；
  * 第三种是，使用了并且是脏页。

InnoDB 的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少。

而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：

* 如果要淘汰的是一个干净页，就直接释放出来复用；
* 但如果是脏页呢，就必须将脏页先刷到磁盘，变成干净页后才能复用。

所以，刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
2. 日志写满，更新全部堵住，写性能跌为 0，这种情况对敏感业务来说，是不能接受的。

所以，InnoDB 需要有控制脏页比例的机制，来尽量避免上面的这两种情况。


## MyISAM

* MyISAM虽然也使用B+树作为存储结构，但是索引和数据是分开的，数据按照插入顺序存储在.myd文件中
* MyISAM的索引结构叶子节点存储的数据项是数据记录地址![Alt](image/mysql/1728647968325.png)

## Mysql架构

* 架构图![1728568163911](image/mysql/1728568163911.png)

## 数据引擎

* InnoDb 支持外键、事务、行级锁，文件格式.ibd .frm
* MyISAM	用于小表、查询插入操作，有数据丢失风险，文件格式.myd .myi .frm
* ARCHIVE	归档，存储时压缩数据，支持行级锁，仅支持插入和查询（查询速度高，插入速度慢），适合存储大量的独立的作为历史记录的数据
* BLACKHOLE 丢弃所有存储的数据
* CSV 将普通CSV作为表处理，文件格式.csm .csv
* Memory 存储在内存，支持hash索引，数据易丢失
* NDB

## 索引

* B+树
  * 图示![1728640982868](image/mysql/1728640982868.png)
  * B+树根节点所在page万年不动
  * 二级索引中，B+树内节点目录项的索引项目，除了指定的索引项还会额外包含主键共同构成联合索引，防止二级索引项相同，增加索引插入效率![1728646543047](image/mysql/1728646543047.png)
  * 
* 聚簇索引
  * 缺点
    * 插入速度依赖于插入顺序，否则出现页分裂影响性能
    * 更新主键代价高
    * 二级索引访问需要两次索引查找

## 事务

## 日志

## SQL性能调优

1. 调优流程![1728733731749](image/mysql/1728733731749.png)![1728733912540](image/mysql/1728733912540.png)![1728733998407](image/mysql/1728733998407.png)
2. 调优手段
   * 查看系统性能参数 show status like 'slow_queries'
   * 查看查询成本 show status like 'last_query_cost'
   * 慢查询日志
     ```
       开启配置
       set global slow_query_log=on;
       慢查询配置(and关系)
       set global long_query_time=1;
       set global min_examined_row_limit=0
     ```
   * profiling
     * show profiles
     * show profile cpu,block io for query 1
   * explain![1728823748153](image/mysql/1728823748153.png)
3. JOIN调优![1728880933531](image/mysql/1728880933531.png)
4. SORT优化![1728885868033](image/mysql/1728885868033.png)
   * 尝试提高sort_buffer_size（排序缓存 进程级）
   * 尝试提高max_length_for_sort_data（双路 单路选择）
   * 避免select *
5. GROUP BY![1728886570169](image/mysql/1728886570169.png)
6. 分页查询![1728886985720](image/mysql/1728886985720.png)
7. 索引下推（ICP index condition pushdown)![1728889607756](image/mysql/1728889607756.png)
8. EXISTS与IN的区分![1728890186008](image/mysql/1728890186008.png)
9. COUNT效率对比![1728890581150](image/mysql/1728890581150.png)

## 事务隔离级别

   ![1728916400552](image/mysql/1728916400552.png)

## 日志

    1. redo log:
       * 是存储引擎层生成的日志，记录的是物理级别上的页修改操作，比如页号、偏移量写入了xxx数据，主要为了保证数据可靠性
       * InnoDB采用了WAL技术（Write-Ahead Logging)，即先写日志，再写磁盘，只有日志写入成功，才算事务提交成功
       * 优点：降低刷盘频率、占用空间小
       * 特点：顺序写入磁盘，事务执行过程中不断记录
       * 刷盘策略：
         * Innodb_flush_log_at_trx_commit=1![1728972480333](image/mysql/1728972480333.png)
         * Innodb_flush_log_at_trx_commit=2![1728972560810](image/mysql/1728972560810.png)
         * Innodb_flush_log_at_trx_commit=0![1728972668928](image/mysql/1728972668928.png)
    2. undo log
       * 是存储引擎层生成的日志，记录的是逻辑操作日志，比如对某一行数据进行了INSERT语句操作，那么undo log就记录一条与之相反的DELETE操作。主要用于事物的回滚和一致性非锁定读（undo log回滚行记录到某种特定的版本--MVCC）
       * 在事务更新数据的前置操作先写入undo log
    3. 更新过程日志记录![1728978687580](image/mysql/1728978687580.png)

## 锁

   ![1728987551614](image/mysql/1728987551614.png)

    1. 乐观锁悲观锁对比：乐观锁适合读多写少的场景，悲观锁适合写多读少场景

## MVCC原理

    1. 相关字段
       1. trx_id:每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的事务id赋值给trx_id隐藏列
       2. roll_pointer:每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息![1729083164255](image/mysql/1729083164255.png)![1729083170300](image/mysql/1729083170300.png)
    2. READ VIEW
       1. 定义：ReadView就是事务在使用MVCC机制进行快照读操作时产生的读视图。事务启动时，会生成数据库系统当前的一个快照，InnoDB为每个事务构造了一个数组，用来记录并维护系统当前活跃事务的ID
       2. 设计思路
          * RU隔离级别直接读取记录最新版本
          * SER级别使用加锁访问
          * RC、RR使用READ VIEW
       3. 字段设计![1729084136210](image/mysql/1729084136210.png)
       4. 快照访问规则![1729084830326](image/mysql/1729084830326.png)
       5. MVCC整体操作流程![1729084873984](image/mysql/1729084873984.png)
       6. RC VS RR
          1. RC级别下，一个事物的每一次SELECT查询都会重新获取一次READ VIEW
          2. RR级别下，一个事务只在第一次SELECT时获取一次READ VIEW，后面所有SELECT都会复用这个READ VIEW

## 日志

    1. 日志类型![1729139284640](image/mysql/1729139284640.png)
    2. 二进制日志
       1. 应用场景![1729139362609](image/mysql/1729139362609.png)
       2. binlog刷盘![1729141820679](image/mysql/1729141820679.png)
          1. 参数sync_binlog=0（默认是0），每次提交只write，有系统自行判断fsync执行时机
          2. sync_binlog=1，每次提交事务都执行fsyc
          3. sync_binlog=N(N>1)，每次提交都write，累计N个之后fsync
       3. binlog对比redolog![1729142004318](image/mysql/1729142004318.png)
       4. 两阶段提交![1729144959894](image/mysql/1729144959894.png)![1729145275755](image/mysql/1729145275755.png)![1729145278091](image/mysql/1729145278091.png)
       5.
主从复制![1729150601989](image/mysql/1729150601989.png)

    1. binlog日志格式及优缺点![1729151431701](image/mysql/1729151431701.png)![1729151460442](image/mysql/1729151460442.png)![1729151520695](image/mysql/1729151520695.png)![1729151594725](image/mysql/1729151594725.png)
    2. 主从一致性问题解决方案
       1. 异步复制![1729153099808](image/mysql/1729153099808.png)
       2. 半同步复制![1729153136522](image/mysql/1729153136522.png)
       3. 组复制![1729153235281](image/mysql/1729153235281.png)![1729153261350](image/mysql/1729153261350.png)
