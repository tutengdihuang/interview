## MyISAM索引与InnoDB索引的区别
- InnoDB支持事务，MyISAM不支持
- InnoDB支持外键，而MyISAM不支持
- InnoDB索引是聚簇索引，MyISAM索引是非聚簇索引。
  - 聚簇索引：数据和索引在一起，找到了索引就找到了数据
    - 聚簇索引默认是主键，如果表中没有定义主键，InnoDB 会选择一个唯一的非空索引代替，InnoDB 会隐式定义一个主键来作为聚簇索引
      - 如果没有非空主键，innodb隐士定义
    - 如果你已经设置了主键为聚簇索引，必须先删除主键，然后添加我们想要的聚簇索引，最后恢复设置主键即可
    - InnoDB 只聚集在同一个页面中的记录。包含相邻键值的页面可能相距甚远
    - 一般要根据这个表最常用的SQL查询方式来进行选择，某个字段作为聚簇索引，或组合聚簇索引
    - 最终目的就是在相同结果集情况下，尽可能减少逻辑IO
  - 非聚簇索引：将数据存储于索引分开结构，索引结构的叶子节点指向了数据的对应行，myisam通过key_buffer把索引先缓存到内存中，
    - 当需要访问数据时（通过索引访问数据），在内存中直接搜索索引，然后通过索引找到磁盘相应数据，这也就是为什么索引不在key buffer命中时，速度慢的原因
  - [参考文档](https://cloud.tencent.com/developer/article/1541265)
- InnoDB不保存表的具体行数，执行select count(*) from table时需要全表扫描。而MyISAM用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快（注意不能加有任何WHERE条件）
- Innodb不支持全文索引，而MyISAM支持全文索引，在涉及全文索引领域的查询效率上MyISAM速度更快高
- MyISAM表格可以被压缩后进行查询操作
- InnoDB支持表、行(默认)级锁，而MyISAM支持表级锁
- InnoDB表必须有唯一索引（如主键）（用户没有指定的话会自己找/生产一个隐藏列Row_id来充当默认主键），而Myisam可以没有
- Innodb存储文件有frm、ibd，而Myisam是frm、MYD、MYI
- InnoDB的主键索引的叶子节点存储着行数据，因此主键索引非常高效。

- MyISAM索引的叶子节点存储的是行数据地址，需要再寻址一次才能得到数据。

- InnoDB非主键索引的叶子节点存储的是主键和其他带索引的列数据，因此查询时做到覆盖索引会非常高效
  - 当索引包含了所有查询的数据时，这个索引就称之为覆盖索引
- innodb不支持全文索引，而MyISAM支持全文索引，查询效率上MyISAM要高
- 如何选择存储引擎
    - 是否要支持事务，如果要请选择innodb，如果不需要可以考虑MyISAM；
    - 如果表中绝大多数都只是读查询，可以考虑MyISAM，如果既有读写也挺频繁，请使用InnoDB。
    - 系统奔溃后，MyISAM恢复起来更困难，能否接受；

## InnoDB引擎的4大特性
- 1、插入缓冲（insert buffer)
  Insert Buffer 只对于非聚集索引（非唯一）的插入和更新有效，对于每一次的插入不是写到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，如果在则直接插入；
  若不在，则先放到Insert Buffer 中，再按照一定的频率进行合并操作，再写回disk。这样通常能将多个插入合并到一个操作中，目的还是减少了随机IO带来性能损耗

  避免校验唯一性，排除聚簇索引，唯一索引，其他索引生效
- 2、二次写(double write)

- 3、自适应哈希索引(ahi)

- 4、预读(read ahead)
- [refer](https://blog.csdn.net/weixin_45320660/article/details/115326483?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164576404416781683952707%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164576404416781683952707&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_click~default-1-115326483.pc_search_result_positive&utm_term=InnoDB%E5%BC%95%E6%93%8E%E7%9A%844%E5%A4%A7%E7%89%B9%E6%80%A7&spm=1018.2226.3001.4187)
## 数据库相关
- 事物的隔离级别
    - 读未提交
    - 读提交 解决脏读
    - 可重复读 解决不可重复读
    - 串行化    幻读
- 如何解决不可重复读？mysql MVCC底层原理
    - [查看mvcc的原理](https://www.processon.com/view/link/620b01c807912979960d3ac1)
    - [查看mvcc的原理](https://www.processon.com/view/link/620b01d46376897c8c712c57)

- 数据库什么情况下走索引，什么情况下不走索引
    - mysql认为全局遍历比走索引快的时候就会放弃索引，
    - 索引一般都是重复率低，或者不重复
    - 如果对性别只有男女进行索引，这种b+树只有一层，没必要走索引
- 第一类丢失更新，第二类丢失更新
    - 第一类丢失更新(回滚丢失，Lost update)，
        - A事务撤销时，把已经提交的B事务的更新数据覆盖了。
        - A事务在撤销时，“不小心”将B事务已经转入账户的金额给抹去了
    - 第二类丢失更新(覆盖丢失/两次更新问题，Second lost update)
        - A事务覆盖B事务已经提交的数据，造成B事务所做操作丢失

## 百万级别或以上的数据如何删除
- 先删除索引
- 删除无用的数据
- 删除完成后重新创建索引

## 什么是最左前缀原则？什么是最左匹配原则
- mysql 会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如 a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d 是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d 的顺序可以任意调整
    ```
    表 table (a , b ,c, d )
    索引 index01 (a,b,c)

    如果 select a,b,c from table where a=1 and b=1 and c=1;
    那么索引 a b c 都会用到，且不用回表，会最快返回；

    如果 select a,b,c from table where a=1 and  c=1;
    那么索引只会用到 a 过滤,但仍然不用回表，较快返回；

    如果 select a,b,c from table where a=1 and b>1 and b<1 and c=1;
    那么索引只会用到 a , b  ，但是仍然不用回表，较快返回；

    如果 select a,b,c ,d from table where  b=1 and c=1;
    那么索引会无法使用  ，返回时间长；
  
   如果 select a,b,c ,d from table where  a=1 or c=1;
   过滤出所有满足a=1或者c=1条件的数据行，然后再根据索引中的数据行指针，去表中读取这些数据行的完整数据，
   并返回给查询结果。因此，在这种情况下需要回表（即需要读取表中的数据行）
    ```
- MySQL联合索引原理
    - [refer](https://blog.csdn.net/cherry93925/article/details/100719559?spm=1001.2101.3001.6650.5&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-5.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-5.pc_relevant_default&utm_relevant_index=8)

## 什么是聚簇索引？何时使用聚簇索引与非聚簇索引
    - 聚簇索引：将数据存储与索引放到了一块，找到索引也就找到了数据
    - 非聚簇索引：将数据存储于索引分开结构，索引结构的叶子节点指向了数据的对应行

## 什么是临时表，何时删除临时表？
```mysql
在MySQL数据库中，临时表（Temporary Table）是一种特殊的表，用于在一个会话中暂存数据，只存在于当前数据库连接中。它的定义方式和普通表一样，但表名前需要添加 # 或 tmp 等前缀来表示是临时表，如 #temp_table 或 tmp_table。

临时表可以帮助我们在执行复杂查询时存储临时数据，以提高查询性能。临时表的使用通常有以下几个场景：

当查询需要排序、分组、合并等聚合操作时，可以将查询结果存储到临时表中，然后再对临时表进行操作，避免多次扫描原始表而影响性能。

当查询需要使用大量临时数据时，临时表可以避免将所有数据存储在内存中，从而节省内存资源。

当多个查询需要共享相同的临时数据时，可以使用临时表来存储这些数据，以避免在每个查询中重复计算。

在MySQL中，临时表会在以下情况下自动删除：

当前数据库连接关闭时，所有临时表会被自动删除。

当使用 DROP TEMPORARY TABLE 命令手动删除临时表时，相应的临时表会被删除。

当当前连接执行完成后，MySQL会自动清除由该连接创建的所有临时表。

需要注意的是，临时表只在当前数据库连接中可用，如果想在其他连接中使用同样的临时表，需要重新创建一遍。
```
## 结果集做去重(distinct)

## 建立索引的原则
    - 1、表的主键、外键必须有索引；
    - 2、数据量超过300的表应该有索引；
    - 3、经常与其他表进行连接的表，在连接字段上应该建立索引
    - 4、经常出现在Where子句中的字段，特别是大表的字段，应该建立索引；
    - 5、索引应该建在选择性高的字段上；
    - 6、索引应该建在小字段上，对于大的文本字段甚至超长字段，不要建索引；
    - 7、复合索引的建立需要进行仔细分析；尽量考虑用单字段索引代替：
        - A、正确选择复合索引中的主列字段，一般是选择性较好的字段；
        - B、复合索引的几个字段是否经常同时以AND方式出现在Where子句中？单字段查询是否极少甚至没有？如果是，则可以建立复合索引；否则考虑单字段索引；
        - C、如果复合索引中包含的字段经常单独出现在Where子句中，则分解为多个单字段索引；
        - D、如果复合索引所包含的字段超过3个，那么仔细考虑其必要性，考虑减少复合的字段；
        - E、如果既有单字段索引，又有这几个字段上的复合索引，一般可以删除复合索引;
    - 8、频繁进行数据操作的表，不要建立太多的索引；
    - 9、删除无用的索引，避免对执行计划造成负面影响； 以上是一些普遍的建立索引时的判断依据

## 创建索引的规则
- 取值离散大的字段
- 索引字段越小越好
- 主键外键必须有索引
- 数据量超过300的表应该有索引
- 经常与其他表进行连接的表，在连接字段上应该建立索引；
- 经常出现在Where子句中的字段，特别是大表的字段，应该建立索引
- 索引应该建在选择性高的字段上
- 索引应该建在小字段上，对于大的文本字段甚至超长字段，不要建索引
- 复合索引的建立需要进行仔细分析；尽量考虑用单字段索引代替
  - A、正确选择复合索引中的主列字段，一般是选择性较好的字段；
  - B、复合索引的几个字段是否经常同时以AND方式出现在Where子句中？单字段查询是否极少甚至没有？如果是，则可以建立复合索引；否则考虑单字段索引；
  - C、如果复合索引中包含的字段经常单独出现在Where子句中，则分解为多个单字段索引；
  - E、如果既有单字段索引，又有这几个字段上的复合索引，一般可以删除复合索引；
- 频繁进行数据操作的表，不要建立太多的索引；
- 删除无用的索引，避免对执行计划造成负面影响
  - 索引对于插入、删除、更新操作也会增加处理上的开销
  - 比如性别可能就只有两个值，建索引不仅没什么优势，还会影响到更新速度，这被称为过度索引
  - 只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的


## mysql的change buffer的作用？唯一索引，普通索引中用到？

~~~markdown
mysql8.0 change buffer ： [源码说明](https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html)
	解释：change buffer 顾名思义是 ‘更改’缓存，就是对数据库 更改 动作的一个缓存，但是缓存的是那些不在buffer pool的二级索引页的一些MDL（insert update delete）操作的，
    随后当遇到一些相关其他的读操作，mysql会‘一起’将他们merge到buffer pool里，然后buffer pool里的内容会purge（清洗或者理解成flush）到disk存储中，
    当然也有定时任务会对change buffer 和 buffer pool的内容进行合并。和聚簇索引不一样，普通索引一般都不唯一，在业务中插入二级索引比较常见且顺序随机，
    删除和更新等操作很可能会影响那些相邻的二级索引页，稍后合并缓存的更改，当受影响的页面被其他操作读入缓冲池时，避免了将二级索引页面从磁盘读入缓冲池所需的大量随机访问 I/O。
    在系统大部分空闲或缓慢关闭期间运行的清除操作会定期将更新的索引页写入磁盘。与将每个值立即写入磁盘相比，purge操作可以更有效地为一系列索引值写入磁盘块。
	
	注意：官网原文：Change buffering is not supported for a secondary index if the index contains a descending index column or if the primary key 
    includes a descending index column. 如果索引包含了降序索引列或者主键包含降序索引列，那就不支持使用change buffer了。
    这可能是排查问题的一个好点子，具体可查FAQ：https://dev.mysql.com/doc/refman/8.0/en/faqs-innodb-change-buffer.html
	
	特点（注意点）：
		1. 虽然叫change buffer，实际上是可持久化的数据。即change buffer在内存中有拷贝，也会被写进磁盘（change buffer的操作也记录到redo log）
		2. change buffer 只支持二级索引。聚簇，全文，空间索引都不支持，特别是全文索引，有他自己的缓存机制
		3. change buffer是 buffer pool 的部分 5.6版本change buffer最多可以使用30%，5.6之后最多50%，默认是25%。change buffer不会一直存在，LRU算法会进行淘汰
		4. 简单来说为了减少随机IO 的发生，change buffer适合应用在写多读少，该类业务模型常见为账单、日志类的系统。假设一业务的更新模式是写后马上查询，那么即使满足条件，
            将更新先记录在change buffer，但之后由于马上要访问该数据页，立即触发merge。这样随机访问IO的次数不会减少，反而增加change buffer维护代价。
            所以，对于这种业务模式，change buffer起反作用。
		5. redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。
		
	作用：当操作更新一个数据页时
		- 若数据页在内存（buffer pool），直接更新
		- 若该数据页不在内存，在不影响数据一致性前提下，InooDB会将这些更新操作缓存在change buffer，无需从磁盘读入该数据页，在下次查询需要访问该数据页	    	   时，将数据页读入内存，然后执行change buffer中与这个页有关的操作。通过该方式就能保证这个数据逻辑的正确性。

change buffer 和 二级索引、唯一索引有什么关系呢？
	刚才上面提到了，change buffer本质还是为了提高性能的，基本都是涉及到了二级索引页的变更。那如果涉及到唯一索引呢？
	对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束！因此，这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了【没想到吧>._.<】。，唯一索引的更新就不能使用 change buffer，
    实际上也只有普通索引可以使用。


~~~





## 查询很慢怎么排查和优化？
- [refer](https://www.cnblogs.com/qmfsun/p/4844472.html)
## EXPLAIN有什么用途？有哪些字段？
- 模拟Mysql优化器是如何执行SQL查询语句的，从而知道Mysql是如何处理你的SQL语句的。分析你的查询语句或是表结构的性能瓶颈
- (一)id列：(1)、id 相同执行顺序由上到下
- (二)select_type列：数据读取操作的操作类型
- (三)table列：该行数据是关于哪张表
- (四)type列：访问类型由好到差system > const > eq_ref > ref > range > index > ALL
- (五)possible_keys列：显示可能应用在这张表的索引，一个或者多个。查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用
- (六)keys列：实际使用到的索引
- (七)ken_len列：表示索引中使用的字节数，可通过该列计算查询中使用的索引长度
- (八)ref列：显示索引的哪一列被使用了，如果可能的话，是一个常数
- (九)rows列(每张表有多少行被优化器查询)
- (十)Extra列：扩展属性，但是很重要的信息
## InnoDB索引底层实现？为什么使用b+树不适用b树？
- [refer](https://blog.csdn.net/weixin_38054045/article/details/114024721?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164551017416780265482140%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=164551017416780265482140&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-2-114024721.pc_search_result_positive&utm_term=%E4%B8%BA%E4%BB%80%E4%B9%88b%2B%E6%A0%91%E5%87%8F%E5%B0%8F%E4%BA%86io&spm=1018.2226.3001.4187)
- B+ 树的非叶子节点上只储存键值，而 B 树的非叶子节点上不仅储存键值还储存数据。
- 在 MySQL 数据库中数据页的大小是固定的，Innodb 引擎数据页默认大小为 16 KB。
- B+ 树这种做法是为了让树的阶数更大，让树更矮胖。
- 进行查询的时候，磁盘 IO 次数就会减少，查询效率也会更快。
  - B+树的中间节点没有卫星数据的。所以同样大小的磁盘页可以容纳更多的节点元素。(这就意味着B+会更加矮胖，查询的IO次数会更少)
  - InnoDB存储引擎的最小存储单元是页，页可以用于存放数据也可以用于存放键值+指针，在B+树中叶子节点存放数据，非叶子节点存放键值+指针
- B+ 树的所有数据均储存在叶子节点中，并且是按键值有序排列。
- 但是 B 树的数据分散在各个节点。进行范围查询，排序查询的时候，B 树的效率肯定不如 B+ 树
- b+树的优势
  - 1.单一节点存储更多的元素，使得查询的IO次数更少。
  - 2.所有查询都要查找到叶子节点，查询性能稳定。
  - 3.所有叶子节点形成有序链表，便于范围查询
## 为什么b+树减少了io
- 见上一题
## 什么是索引？什么是回表？
- https://baijiahao.baidu.com/s?id=1669796110955401759&wfr=spider&for=pc
## MySQL的ACID怎么实现？
```mysql

原子性（Atomicity）：MySQL使用undo log实现原子性，即将事务中的所有操作作为一个整体，要么全部执行，要么全部回滚，不会出现部分执行的情况。如果一个事务的操作失败，MySQL会使用undo log回滚所有操作，保证数据一致性。

一致性（Consistency）：MySQL使用redo log实现一致性，即事务的执行不会破坏数据库的完整性和约束条件。MySQL通过redo log来记录数据的修改操作，当系统宕机或崩溃时，通过redo log进行恢复，保证数据的一致性。

隔离性（Isolation）：MySQL使用锁机制实现隔离性，即多个事务之间的执行互不干扰，每个事务之间的执行是隔离的，相互不可见。MySQL提供了多种隔离级别，包括读未提交、读已提交、可重复读和串行化。

持久性（Durability）：MySQL使用redo log和binlog实现持久性，即事务提交后，数据会被持久地保存在磁盘上，即使系统宕机或崩溃，也能够通过redo log和binlog来恢复数据。
```
## 有哪些隔离级别？实现原理？
```mysql
READ UNCOMMITTED
READ UNCOMMITTED 是最低的隔离级别，事务不需要获得任何锁定就可以访问未提交的数据。它的实现非常简单，由于不需要获取任何锁定，所以它的性能最好，但它也是最不安全的隔离级别，因为它可能会读取到其他事务正在修改的数据，从而出现脏读的现象。

READ COMMITTED
READ COMMITTED 隔离级别要求事务只能读取已提交的数据，即当一个事务读取某个数据时，如果另一个事务正在修改该数据，则需要等待该事务提交之后才能读取该数据。它可以通过共享锁来实现，即当一个事务读取某个数据时，需要对该数据加共享锁，如果有其他事务正在修改该数据，则需要等待该事务提交之后才能获得该数据的共享锁。

REPEATABLE READ
REPEATABLE READ 隔离级别要求事务读取数据后，无论其他事务如何修改，它读取到的数据始终保持一致。为了实现这种隔离级别，MySQL 采用了多版本并发控制（MVCC）机制。MVCC 机制会为每个修改操作生成一个版本号，然后在读取数据时，会根据事务开始的时间戳和数据的版本号来确定该事务可以读取的数据。由于需要保留历史版本的数据，因此需要占用更多的空间。

SERIALIZABLE
SERIALIZABLE 是最严格的隔离级别，它要求事务之间的操作串行执行，即一个事务要等到另一个事务执行完毕之后才能开始执行。为了实现这种隔离级别，MySQL 会对所有涉及到的数据加锁，防止其他事务修改或读取这些数据。

需要注意的是，隔离级别越高，对性能的影响也越大。因此，在选择隔离级别时，需要根据实际情况进行权衡。
```
## 脏读、幻读概念？怎么解决幻读的问题？间隙锁是什么？可重复读怎么实现？
- 脏读：事务A向表中插入了一条数据，此时事务A还没有提交，此时查询语句能把这条数据查询出来，这种现现象称为脏读；脏读比较好理解
- 不可重复读：一个事务A第一次读取的结果之后，  另外一个事务B更新了A事务读取的数据，A事务在第二次读取的结果和第一次读取的结果不一样这种现象称为不可重复读
- 幻读：事务A更新表里面的所有数据，这时事务B向表中插入了一条数据，这 时事务A第一次的查询结果和第二次的查询结果不一致，这种现象我称为幻读
- 间隙锁：当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但不存在的记录，叫做“间隙(GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁(NEXT-KEY)锁
  - 键值在条件范围内但不存在的记录，叫做“间隙(GAP)”，InnoDB也会对这个“间隙”加锁
- 间隙锁的作用：1.防止幻读, 2.防止数据误删/改

#
## MySQL怎么实现高可用？
```mysql
主从复制：通过将一个MySQL实例作为主服务器，将数据同步到一个或多个从服务器，从服务器上的数据始终保持与主服务器同步。如果主服务器发生故障，可以通过将从服务器提升为新的主服务器来实现自动故障转移。主从复制需要配置主服务器和从服务器之间的复制关系，可以使用基于GTID的复制来保证数据一致性。

MySQL Cluster：MySQL Cluster是一个基于共享存储和网络互连的高可用集群解决方案，可以提供高可用性和容错性。它包括MySQL Cluster Manager（NDB Cluster Manager）和MySQL Cluster（NDB Cluster）两个组件，其中MySQL Cluster Manager用于管理MySQL Cluster的配置和部署，MySQL Cluster则是提供高可用性的分布式数据库。

MySQL Group Replication：MySQL Group Replication是MySQL 5.7中新增的高可用性解决方案，它基于半同步复制实现多主复制，可以保证在一个节点出现故障时不会丢失数据。

MySQL InnoDB Cluster：MySQL InnoDB Cluster是一个集成了MySQL Group Replication、MySQL Router和MySQL Shell的高可用性解决方案，可以提供自动故障转移和负载均衡等功能。
```
## 自增id到了最大，再insert一条数据会发生什么？
- 产生主键冲突错误
- 解决方法
  - int-biging
  - 分表，有效避免这个问题
  - int类型设置为无符号的可以扩大一倍
## 从分组的结果中选出最大的5个数
- select a.* from tb a where val in (select top 2 val from tb where name=a.name order by val desc) order by a.name,a.val
## group和having的区别 
- [refer](https://blog.csdn.net/u012106306/article/details/115009698?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164430198116780357225944%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164430198116780357225944&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-115009698.pc_search_insert_ulrmf&utm_term=group%E5%92%8Chaving%E7%9A%84%E5%8C%BA%E5%88%AB&spm=1018.2226.3001.4187)

#
## live环境千万条数据如何迁移
```mysql
数据迁移的具体方案和步骤需要根据实际情况综合考虑，以下是一般情况下可能采取的迁移方式和步骤：

数据库备份：在源数据库中使用 mysqldump 或其他备份工具进行备份，将备份文件传输到目标服务器，然后在目标服务器中使用 mysql 命令或其他导入工具进行导入。备份和导入的过程需要保证数据的一致性和完整性。

数据库复制：在源数据库中开启二进制日志（binary log），并启用基于二进制日志的主从复制（replication），将主库的更新操作同步到从库中。在完成数据同步后，可以将从库提升为新的主库。主从复制可以在迁移过程中保证数据的一致性和高可用性，但需要保证主库和从库之间的网络稳定和延迟较小。

数据库迁移工具：使用第三方数据库迁移工具，例如 Percona XtraBackup，mydumper 等，它们可以在不停机的情况下迁移数据，但需要考虑数据库版本和迁移工具的兼容性，并进行测试以保证数据的一致性和完整性。

数据库分批迁移：将数据分批迁移到目标服务器，每次迁移一部分数据并进行测试，验证数据的正确性和完整性后再进行下一批数据的迁移。分批迁移可以降低数据迁移的风险和影响，但需要协调好迁移和测试的时间和资源，避免过长的迁移时间对业务产生影响。

无论采用哪种迁移方式，都需要在迁移前制定详细的计划和测试方案，以避免数据丢失和业务中断等风险，并在迁移后进行数据一致性验证和性能测试，以保证数据迁移的顺利完成和新环境的稳定运行。
```
#
## live环境数据备份
```mysql
在live环境中备份数据通常需要使用以下步骤：

关闭所有写入操作：在备份数据之前，需要确保没有正在进行的写入操作。可以通过暂停应用程序或禁用写入操作的方式实现。

使用备份命令备份数据：MySQL提供了多种备份命令，包括mysqldump、mysqlhotcopy和mysqlbackup等。其中，mysqldump是备份MySQL数据最常用的命令，它能够将整个数据库或单个表的数据导出为SQL脚本文件。

例如，使用mysqldump备份数据库可以执行以下命令：

css
Copy code
mysqldump -u username -p database_name > backup.sql
其中，username是MySQL数据库的用户名，database_name是要备份的数据库名称，backup.sql是备份的SQL脚本文件。

将备份数据复制到安全的位置：备份数据需要复制到一个安全的位置，例如另一台服务器或云存储。

启动写入操作：在备份完成后，需要恢复正常的写入操作。此时需要重新启动应用程序或允许写入操作。

注意，在备份数据时需要注意以下事项：

需要定期备份数据，以保证数据的可靠性和完整性。
在备份数据时需要确保备份数据的安全性，避免数据被未经授权的人访问。
备份数据需要考虑存储空间的问题，如果备份数据量很大，需要使用压缩技术或分片备份来降低存储空间的需求。
```

#
## live环境表格结构修改
  - 执行时间
    - 对于数据量较大的表，需要修改表结构，或者做一些耗时比较久的锁表操作，建议在晚上（业务闲时）执行
  - 第一种方案
    - 可以在变更表结构的命令中添加一个超时时间
    - alter table practice.Student wait 100 add column Sheight int(4) not null default 0 comment "身高"
  - 第二种方案
    - http://blog.sina.com.cn/s/blog_4cb992270101ke0z.html
## live环境批量数据修改
## live环境mysql主从同步 数据流失怎么办
```mysql
如果出现MySQL主从同步数据流失的情况，可以采取以下措施：

检查主从同步是否正常：可以查看主从同步的状态信息，包括两个服务器的binlog名称和位置是否相同，以及主服务器和从服务器之间是否有延迟。

检查网络连接：网络连接是主从同步的基础，如果网络连接不稳定或中断，会导致数据流失。可以通过检查网络连接是否正常，以及检查主从服务器之间的网络带宽是否足够等方式解决。

重新启动从服务器：可以尝试重新启动从服务器，重新连接主服务器，并从当前的binlog位置开始同步数据。

使用备份进行数据恢复：如果数据流失较多或者无法通过同步方式进行恢复，可以使用备份进行数据恢复，可以通过备份文件将数据还原到某一时刻。

使用工具进行数据恢复：如果以上措施都无法解决，可以尝试使用MySQL数据恢复工具，如MySQL备份工具、InnoDB recovery工具等进行数据恢复。

注意：在进行数据恢复之前，必须先对数据进行备份，以免数据恢复过程中出现问题导致数据丢失。此外，应该尽量避免出现数据流失的情况，可以采取双主模式、多从服务器等方式提高MySQL的高可用性，确保数据的安全性和可靠性。
```
## 什么是mysql
## 什么是b+树
  - [refer](https://blog.csdn.net/xl_1803/article/details/113327698?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164430231516780265484124%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164430231516780265484124&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_ulrmf~default-1-113327698.pc_search_insert_ulrmf&utm_term=%E4%BB%80%E4%B9%88%E6%98%AFb%2B%E6%A0%91&spm=1018.2226.3001.4187)
  - b树的缺点
    - 当进行范围查找时，存在回旋查找的问题
    - 排序的时候，需要进行一次中序遍历（order by）
## 2022-01-19收录
- 你碰到过的数据库优化最难的问题，及如何解决
- mysql 索引覆盖，回表 （滴滴）
  - SQL只需要通过索引就可以返回查询所需要的数据，而不必通过二级索引查到主键之后再去查询数据
- 忘了加唯一索引有啥补救措施吗
  - 只是参考答案：新增别出现相同数据就好
  - 添加唯一索引，检查重复数据，重新倒入数据，做好数据备份
- 在唯一索引的约束下，如何优雅地软删除
- 需求: 一张表中有个字段appid，同一个appid只允许存在一行正常记录，但是可以存在多条软删除记录
  - 答案：额外加一个status字段，0为正常，非零为已删除。和appid作为复合唯一索引，软删除的时候将status改为当前时间戳
- mysql索引的类型，各自的特点，还有索引失效的情况
- 腾讯外包公司题：
  - mysql唯一索引是否可以为null？为什么？、
    - 允许存在的多个NULL值数据
  - select for update是表锁还是行锁？（仔细查找答案，有坑）
    - 如果查询条件用了索引/主键，那么select ..... for update就会进行行锁。
    - 如果是普通字段(没有索引/主键)，那么select ..... for update就会进行锁表。
  - 乐观锁和悲观锁数据库层面如何实现？
    - [refer](https://blog.csdn.net/just_learing/article/details/124898579?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166779193316782417037044%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=166779193316782417037044&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-3-124898579-null-null.142^v63^control,201^v3^control_1,213^v1^control&utm_term=%E4%B9%90%E8%A7%82%E9%94%81%E5%92%8C%E6%82%B2%E8%A7%82%E9%94%81%E6%95%B0%E6%8D%AE%E5%BA%93%E5%B1%82%E9%9D%A2%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0&spm=1018.2226.3001.4187)
  - 缓存数据和数据库数据如何实现一致性？


- 使用索引查询一定能提高查询的性能吗？为什么
  - 不一定
  - 索引需要额外的存储空间和处理,那些不必要的索引反而会使查询反应时间变慢.使用索引查询不一定能提高查询性能
- 事务还没提交的时候，redolog 能不能被持久化到磁盘呢(字节一面)
  - [解释参考](https://mp.weixin.qq.com/s/kdPb4v5nOu0LMCj8s1ETNg)

- mysql 联合查询用法
  - INNER JOIN(等值连接) 只返回两个表中联结字段相等的行
  - LEFT JOIN(左联接) 返回包括左表中的所有记录和右表中联结字段相等的记录
  - RIGHT JOIN(右联接) 返回包括右表中的所有记录和左表中联结字段相等的记录
  - SELECT * FROM 表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号
  - SELECT * FROM (表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号) INNER JOIN 表3 ON 表1.字段号=表3.字段号
  - SELECT * FROM ((表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号) INNER JOIN 表3 ON 表1.字段号=表3.字段号) INNER JOIN 表4 ON Member.字段号=表4.字段号
- mysql group用法
  - group by语法可以根据给定数据列的每个成员对查询结果进行分组统计，最终得到一个分组汇总表
    - 查询每个部门的总的薪水数
    - SELECT DEPT, MAX(SALARY) AS MAXIMUM
      FROM STAFF
      GROUP BY DEPT
  - 将 WHERE 子句与 GROUP BY 子句一起使用
    - 查询公司2010年入职的各个部门每个级别里的最高薪水
    - SELECT DEPT, EDLEVEL, MAX( SALARY ) AS MAXIMUM
      FROM staff
      WHERE HIREDATE > '2010-01-01'
      GROUP BY DEPT, EDLEVEL
      ORDER BY DEPT, EDLEVEL
  - 在GROUP BY子句之后使用HAVING子句
    - 寻找雇员数超过2个的部门的最高和最低薪水
    - SELECT DEPT, MAX( SALARY ) AS MAXIMUM, MIN( SALARY ) AS MINIMUM
      FROM staff
      GROUP BY DEPT
      HAVING COUNT( * ) >2
      ORDER BY DEPT
- mysql group by  having 和 where 执行顺序
- mysql 索引有哪些
  - 1.普通索引
  - 2.唯一索引
  - 3.主键索引
  - 4.组合索引
  - 5.全文索引
    - fulltext索引
- mysql 主键索引和二级索引有什么区别
  - mysql中每个表都有一个聚簇索引（clustered index ），除此之外的表上的每个非聚簇索引都是二级索引，又叫辅助索引（secondary indexes）
  - 
- mysql 做过哪些优化
  - SQL语句的优化
    - 尽量避免使用子查询
    - 避免函数索引
    - 用IN来替换OR
    - LIKE前缀%号、双百分号、_下划线查询非索引列或*无法使用到索引，如果查询的是索引列则可以
    - 读取适当的记录LIMIT M,N，而不要读多余的记录
    - 避免数据类型不一致
    - 分组统计可以禁止排序sort，总和查询可以禁止排重用union all
      - union和union all的差异主要是前者需要将结果集合并后再进行唯一性过滤操作，这就会涉及到排序，增加大量的CPU运算，加大资源消耗及延迟
      - union all的前提条件是两个结果集没有重复数据
      - 一般是我们明确知道不会出现重复数据的时候才建议使用 union all 提高速度
      - 如果排序字段没有用到索引，就尽量少排序
    - 避免随机取记录
    - 禁止不必要的ORDER BY排序
    - 批量INSERT插入
    - 不要使用NOT等负向查询条件
    - 尽量不用select *
      - SELECT *增加很多不必要的消耗（cpu、io、内存、网络带宽）
    - 区分in和exists
      - 区分in和exists主要是造成了驱动顺序的改变
      - 如果是exists，那么以外层表为驱动表，先被访问
      - 如果是IN，那么先执行子查询
      - IN适合于外表大而内表小的情况；EXISTS适合于外表小而内表大的情况
  - 索引的优化
    - Join语句的优化
      - 尽可能减少Join语句中的NestedLoop的循环次数：“永远用小结果集驱动大的结果集
      - 优先优化Nested Loop的内层循环（也就是最外层的Join连接），因为内层循环是循环中执行次数最多的，每次循环提升很小的性能都能在整个循环中提升很大的性能
      - 对被驱动表的join字段上建立索引
      - 当被驱动表的join字段上无法建立索引的时候，设置足够的Join Buffer Size
      - 尽量用inner join(因为其会自动选择小表去驱动大表).避免 LEFT JOIN (一般我们使用Left Join的场景是大表驱动小表)和NULL，那么如何优化Left Join呢
        - 条件中尽量能够过滤一些行将驱动表变得小一点，用小表去驱动大表
        - 右表的条件列一定要加上索引（主键、唯一索引、前缀索引等），最好能够使type达到range及以上
      - 适当地在表里面添加冗余信息来减少join的次数
      - 使用更快的固态硬盘
    - 避免索引失效（mysql什么情况下索引会失效）
      - 最佳左前缀法则
      - 不在索引列上做任何操作
        - (计算、函数、(自动or手动)类型转换)，会导致索引失效而转向全表扫描
      - 存储引擎不能使用索引中范围条件右边的列
      - 尽量使用覆盖索引(只访问索引的查询(索引列和查询列一致))
      - mysql在使用不等于(!= 或者 <>)的时候无法使用索引会导致全表扫描。
      - is null, is not null 也无法使用索引，在实际中尽量不要使用null
      - like 以通配符开头(‘%abc..’)mysql索引失效会变成全表扫描的操作
      - 字符串不加单引号索引失效
      - 少用or，用它来连接时会索引失效
      - 尽量避免子查询，而用join
      - 在组合索引中，将有区分度的索引放在前面
      - 避免在 where 子句中对字段进行 null 值判断
      - [失效场景](https://mp.weixin.qq.com/s/nnwXug8EaLiIl4UIAElvMQ)
- 相对B树，B+树做索引的优势
  - B+树的磁盘读写代价更低
    - B+树的内部节点并没有指向关键字具体信息的指针，因此其内部节点相对B树更小
    - 把所有同一内部节点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多
    - 读取相同的数据量，io次数相对减少
  - B+树的查询效率更加稳定
  - B+树只需要去遍历叶子节点就可以实现整棵树的遍历，遍历效率高
- Mysql 默认的隔离级别是什么？在 Innodb 的可重复读的情况下可以解决幻读的情况吗？（字节）
  - 可重复读
  - MySQL可重复读的隔离级别中并不是完全解决了幻读的问题，而是解决了读数据情况下的幻读问题。而对于修改的操作依旧存在幻读问题。
- 如何解决幻读？
  - 第一种方式 使用串行化读的隔离级别
  - 第二种方式 MVCC+next-key locks：next-key locks由record locks(索引加锁) 和 gap locks(间隙锁，每次锁住的不光是需要使用的数据，还会锁住这些数据附近的数据)
- 如何对数据库进行分库分表，不允许停止服务
  - 第一阶段： 编写代理层和DAO层，代理层动态开关，决定写的是新表还是旧表，此时流量仍然是访问旧表
    - <img src="https://user-images.githubusercontent.com/31843331/152716838-883162bd-cdaf-4d01-b2ef-74b34d8099e4.png" width = "300" height = "300" alt="图片名称" />
  - 第二阶段： 开启双写，增量数据同时在旧表和新表进行新增和修改，日志或者临时表写入新表id的起始值，旧表中小于这个id值的数据就是存量数据
    - <img src="https://user-images.githubusercontent.com/31843331/152717372-ddb80d3e-3746-4a67-960f-3ecf1c86d497.png" width = "300" height = "300" alt="图片名称" />
  - 第三阶段：进行增量数据同步，通过脚本通过分页将存量数据同步到新库
    - <img src="https://user-images.githubusercontent.com/31843331/152717509-b4a82a9d-789a-4774-b3ea-0ac5cbb2929c.png" width = "300" height = "300" alt="图片名称" />

  - 第四阶段： 停读旧表，改读新表，此时新表已经承担了所有读写业务，但是不能停止写旧表，需要双写一段时间
    - <img src="https://user-images.githubusercontent.com/31843331/152717888-d86e480a-b254-4461-bdf9-711f570d1e90.png" width = "300" height = "300" alt="图片名称" />

  - 第五阶段：读写一段时间新表后，没有发生问题，可以停止写旧表
    - <img src="https://user-images.githubusercontent.com/31843331/152718010-7916171b-432c-4a42-96c7-d25324c4b76f.png" width = "300" height = "300" alt="图片名称" />
  - [reference](https://developer.aliyun.com/article/782236)


- mysql update 语句执行流程
  - [refer](https://processon.com/mindmap/60f6547e079129546fe40268)
  - redo undo,binlog  介绍下,    
    - redo log
      - 将哪个数据页哪里发生了修改写入到redo log当中,而不需要将修改过的整个数据页刷到磁盘当中去
      - 写redo log同样也是一次磁盘的写操作,凭什么说它的性能就更高一点呢
        - 数据顺序写入redo log当中,这里其实就是一次顺序写磁盘的操作,
        - 对于binlog来说一个修改操作可能会同时修改多个数据页,这些数据页又不是连续的,此时就意味着随机写磁盘
        - 写redo log和刷数据页,写redo log是磁盘的顺序写,小数据量,而刷数据页到磁盘可能就意味着随机写,而且还是 大数据量的,两者一比较,写redo log的性能可能比刷数据页的性能高100倍
        - 所以redo log 既能保证数据不丢失，也能保证了性能
    - binlog
      - Mysql binlog是二进制日志文件，用于记录mysql的数据更新或者潜在更新
      - Row level
      - Statement level（默认）
      - Mixed（混合模式）
    - undo log 
      - undo log 是逻辑日志，用来提供回滚操作
      - 指针链表，头指向最近的旧版本，尾部指向最早的版本
    - relay log
      ```mysql
      MySQL的Relay Log是一种二进制日志文件，用于记录从MySQL主服务器上复制的事务事件。
      在MySQL主从复制架构中，主服务器将所有的写操作都记录在二进制日志文件中，从服务器通过将主服务器的二进制日志复制到本地，来保持与主服务器的数据同步。
      在这个过程中，从服务器上的Relay Log就记录了从主服务器复制过来的事务事件。

      当从服务器连接到主服务器时，主服务器会将当前二进制日志的名称和位置发送给从服务器。
      从服务器接收到这个信息后，就会从主服务器复制相应的二进制日志，并将其写入本地的Relay Log。
      从服务器读取Relay Log中的事务事件，并在本地执行这些事务，从而保持与主服务器的数据同步。

      Relay Log文件的命名规则通常是"relay-bin.NNNNNN"，其中"NNNNNN"表示一个递增的数字，用于标识不同的Relay Log文件。
      在MySQL复制过程中，Relay Log文件可能会不断增长，因此需要定期清理和删除旧的Relay Log文件，以释放磁盘空间
      ```
- mysql 半同步介绍下
  - 主库只需要等待至少一个从节点，收到并且flush binlog到relay log文件即可，
  - 主库不需要等待所有从库给主库反馈，这里只是一个收到的反馈，而并不是从库已经完成并提交的反馈，
  - 即从库只应用完成io_thread内容即可无需等到sql_thread的执行完成
- mysql 事务原子性怎么实现的
- 分库分表策略
- 分表键和查询条件不一致咋整
- 缓存和数据库的一致性怎么保证,当不一致的时候如何解决
  - [refer](https://www.processon.com/view/link/620af9b50e3e7429dd02be21)

- innodb是如何存储数据的
  - [refer](https://mp.weixin.qq.com/s/665zAn_PuTqAl_rJa_5Ilg)
  
- redolog 落盘
  - https://blog.csdn.net/weixin_40471676/article/details/119732738

- mysql主备如何保证数据同步 
  - https://www.processon.com/view/link/620f131c1efad406e7332e92

- show processlist 命令详解 
  - https://blog.csdn.net/dhfzhishi/article/details/81263084

- 如何判断数据库是否出问题了 
  - https://www.processon.com/view/link/620f17ce6376897c8c7bce60

- mysql group by having 和 where 执行顺序 
  - 先执行where 再执行 group by 再执行 having


- clickhouse的分布式是怎么工作的

- 如何优化mysql, mysql慢查询如何优化，有哪些手段，
  - [如何定位慢查询](https://blog.csdn.net/qq_27276045/article/details/110020421)
  - [refer优化](https://blog.csdn.net/weixin_38805083/article/details/123061693)

- mysql事物回滚过程说一下，越详细越好
```shell
在数据修改的时候，不仅记录了redo，还记录了相对应的undo，如果因为某些原因导致事务失败或回滚了，可以借助该undo进行回滚。当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。

当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚

insert undo log
代表事务在insert新记录时产生的undo log, 只在事务回滚时需要，并且在事务提交后可以被立即丢弃

update undo log
事务在进行update或delete时产生的undo log; 不仅在事务回滚时需要，在快照读时也需要；所以不能随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被purge线程统一清除
```
- mysql假如每天有100万数据放进来，又要支持查询功能，有什么优化方式吗？
  - limit
  - 分表
  - 读写分离
- 分表通过哪些指标分表？
```shell
1、hash取模
2、range范围方案
hash取模和range范围方案；分库分表方案最主要就是路由算法，把路由的key按照指定的算法进行路由存放。
hash取模方案：没有热点问题，但扩容迁移数据痛苦
range方案：不需要迁移数据，但有热点问题。
```
- innodb宕机恢复时候的配置问题
  - 详细见 mysql 技术内幕 innodb存储引擎
- mysqlpage 了解吗？作用是什么？innodb怎么使用？做了哪些优化（结合innodb的特性）
  - [refer](https://blog.csdn.net/duanxiaobin2010/article/details/80896257?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164560172516780271948098%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164560172516780271948098&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-80896257.pc_search_result_positive&utm_term=mysql+page&spm=1018.2226.3001.4187)
- b+树是如何减少io的相对于b树
- 大表如何修改字段
  - [refer](https://www.cnblogs.com/tujia/p/13164389.html)

- 创建订单是一个数据库，创建库存是一个数据库，你怎么保证他们的数据一致性呢?其中一个消费失败 怎么处理呢?说一下
- 页分裂伪代码，b+树的倒数底层层可以页分裂么
  - [InnoDB中的页合并与分裂](https://zhuanlan.zhihu.com/p/98818611)

- redolog和binlog 怎么保证一致性的


#
- mysql服务崩溃后,redoLog恢复后,数据会写到对应的文件上吗?
#
- 数据写入redoLog的过程？ 在xx过程后,会写到redoLog文件中

#
- 缓存和mysql的数据不一致问题,怎么解决

#
- 数据更新时并发问题怎么解决,怎么解决

# 