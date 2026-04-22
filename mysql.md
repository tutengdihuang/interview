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

**Extra 列常见值说明**：

| 值 | 含义 | 优化建议 |
|---|---|---|
| **Using index** | 使用覆盖索引，直接从索引返回数据，无需回表 | ✅ 好，性能优 |
| **Using index condition** | 使用索引条件下推（ICP），将 WHERE 条件下推到存储引擎层过滤 | ✅ 好，MySQL 5.6+ 优化 |
| **Using index for group-by** | 使用索引完成 GROUP BY 或 DISTINCT 操作 | ✅ 好，索引覆盖 |
| **Using index for skip scan** | 使用索引跳跃扫描 | ⚠️ 一般 |
| **Using MRR** | 使用 Multi-Range Read 优化，减少随机 IO | ✅ 正常优化 |
| **Using union** | Index Merge 使用 union 算法合并多个索引 | ✅ 正常 |
| **Using sort_union** | Index Merge 使用 sort_union 算法合并索引 | ✅ 正常 |
| **Using intersect** | Index Merge 使用 intersect 算法取交集 | ✅ 正常 |
| **Distinct** | 优化 DISTINCT，找到匹配行后立即停止搜索 | ✅ 正常 |
| **Not exists** | 优化 NOT EXISTS 或 LEFT JOIN，找到匹配后不再搜索 | ✅ 正常 |
| **No matching rows after const** | const 表没有匹配行 | ✅ 正常 |
| **No tables used** | 查询没有 FROM 子句 | ✅ 正常 |
| **Using where** | 使用 WHERE 条件过滤，需要注意是否伴随 ALL/index 类型 | ⚠️ 需结合 type 判断 |
| **Using temporary** | 需要创建临时表存储结果，通常发生在 GROUP BY/ORDER BY 列无索引时 | ⚠️ 需优化 |
| **Using filesort** | 需要额外排序操作，无法利用索引排序 | ⚠️ 需优化 |
| **Range checked for each Record** | 未找到理想索引，每个记录都需要检查可用索引 | ⚠️ 性能差 |
| **Using filesort,Using temporary** | 同时出现，查询需要临时表且需要排序 | ❌ 差，必须优化 |

**Extra 效率判断**：
- ✅ **好**：Using index、Using index condition、Using index for group-by、Using MRR、Using union/sort_union/intersect、Distinct、Not exists
- ⚠️ **警告**：Using where、Using temporary、Using filesort、Range checked for each Record
- ❌ **差**：Using filesort + Using temporary 组合，必须优化

**type 列连接类型效率排序（从高到低）**：
```
system > const > eq_ref > ref > range > index > ALL
```
| 类型 | 含义 |
|------|------|
| **system** | 系统表，只有单行记录 |
| **const** | 主键或唯一索引的等值查询，只匹配一行 |
| **eq_ref** | 连接中使用主键或唯一索引，只匹配一行 |
| **ref** | 非唯一索引的等值查询 |
| **range** | 索引范围扫描（>, <, BETWEEN, IN 等） |
| **index** | 全索引扫描，比 ALL 快 |
| **ALL** | 全表扫描，性能最差，应尽量避免 |
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
#### GROUP BY 与 HAVING 的区别

#### 基本定义

**GROUP BY** 是分组子句，将结果集按指定列的值合并成若干组，通常配合聚合函数（`COUNT`、`SUM`、`AVG` 等）使用。

**HAVING** 是分组后的过滤子句，对 `GROUP BY` 产生的每一个分组进行条件筛选，保留满足条件的分组。

---

#### 核心区别

| | WHERE | GROUP BY | HAVING |
|---|---|---|---|
| 作用对象 | 原始行 | 分组依据 | 分组结果 |
| 执行时机 | 分组**之前** | 过滤后分组 | 分组**之后** |
| 能否用聚合函数 | ❌ 不能 | — | ✅ 可以 |

---

#### 执行顺序

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

> `WHERE` 先过滤原始数据，再交给 `GROUP BY` 分组，`HAVING` 最后对分组结果再筛选。

---

#### 示例对比

**场景：统计每个部门的员工数，只保留人数超过5人的部门**

```sql
-- ❌ 错误写法：WHERE 不能用聚合函数
SELECT dept_id, COUNT(*) AS cnt
FROM employees
WHERE COUNT(*) > 5       -- 报错！此时还没分组，聚合函数无意义
GROUP BY dept_id;

-- ✅ 正确写法：用 HAVING 过滤分组结果
SELECT dept_id, COUNT(*) AS cnt
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 5;
```

**场景：WHERE 和 HAVING 配合使用**

```sql
-- 先用 WHERE 排除离职员工，再分组，再过滤人数
SELECT dept_id, COUNT(*) AS cnt
FROM employees
WHERE status = 'active'       -- ① 先过滤原始行
GROUP BY dept_id              -- ② 再分组
HAVING COUNT(*) > 5;          -- ③ 最后过滤分组
```

---

#### 记忆口诀

> **WHERE 过滤行，HAVING 过滤组；聚合函数只能交给 HAVING 来管。**
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
#### Live 环境表结构修改

---

#### 一、核心原则

> **Live 环境改表的最大风险：锁表 → 阻塞业务请求 → 服务不可用**

MySQL DDL 操作默认会加 **MDL 元数据锁（Metadata Lock）**，大表执行 `ALTER TABLE` 期间，所有读写请求全部阻塞排队。

---

#### 二、不同操作的风险等级

| 操作 | 风险 | 说明 |
|------|------|------|
| 加索引 | 🔴 高 | 大表重建，锁时间长 |
| 加列（末尾） | 🟡 中 | MySQL 8.0+ 支持 Instant，旧版本重建表 |
| 修改列类型 | 🔴 高 | 全表重建，必须用工具 |
| 删除列 | 🔴 高 | 全表重建 |
| 修改列名 | 🔴 高 | 全表重建 |
| 加默认值 | 🟢 低 | 仅改元数据，瞬间完成 |
| 删除索引 | 🟢 低 | 仅改元数据 |
| 改列长度（varchar增大）| 🟢 低 | 部分场景仅改元数据 |

---

#### 三、推荐工具：pt-online-schema-change & gh-ost

##### 1. pt-osc（Percona Toolkit）

**原理：**
```
① 创建影子表 _new（新结构）
② 在原表加触发器，同步增删改到影子表
③ 批量将原表数据 COPY 到影子表
④ 原子性 RENAME 交换两张表
⑤ 删除旧表和触发器
```

```bash
pt-online-schema-change \
  --host=127.0.0.1 \
  --user=root \
  --password=xxx \
  --alter "ADD COLUMN remark VARCHAR(255) DEFAULT NULL" \
  D=mydb,t=orders \
  --execute
```

**关键参数：**
```bash
--chunk-size=1000          # 每批 COPY 行数（按业务负载调整）
--max-load=Threads_running=50   # 超过阈值自动暂停
--critical-load=Threads_running=100  # 超过直接退出
--no-drop-old-table        # 保留旧表，自己手动确认删除
--dry-run                  # 预演，不真实执行
```

**限制：**
- 原表必须有主键或唯一索引
- 触发器期间有小量额外写入开销

---

##### 2. gh-ost（GitHub 出品）⭐ 更推荐

**原理：** 不用触发器，直接解析 **binlog** 实时同步变更，对原表侵入更小。

```bash
gh-ost \
  --host=127.0.0.1 \
  --user=root \
  --password=xxx \
  --database=mydb \
  --table=orders \
  --alter="ADD COLUMN remark VARCHAR(255) DEFAULT NULL" \
  --allow-on-master \
  --execute
```

**关键参数：**
```bash
--chunk-size=1000           # 每批复制行数
--max-load=Threads_running=30   # 自动限速
--throttle-control-replicas     # 根据从库延迟限速
--postpone-cut-over-flag-file=/tmp/gh-ost.postpone  # 手动控制最终切换时机
--panic-flag-file=/tmp/gh-ost.panic    # 写入该文件立即中止
```

**优势对比 pt-osc：**

| | pt-osc | gh-ost |
|---|---|---|
| 同步方式 | 触发器 | binlog 解析 |
| 原表写入影响 | 有（触发器开销）| 极小 |
| 可暂停/恢复 | ❌ | ✅ |
| 手动控制切换 | ❌ | ✅ |
| 需要从库 | 否 | 推荐有从库 |

---

#### 四、MySQL 8.0 Instant DDL

部分操作直接支持 **ALGORITHM=INSTANT**，毫秒级完成，无需借助工具：

```sql
-- 末尾加列（8.0.29+ 支持任意位置）
ALTER TABLE orders
  ADD COLUMN remark VARCHAR(255) DEFAULT NULL,
  ALGORITHM=INSTANT;

-- 修改列默认值
ALTER TABLE orders
  ALTER COLUMN status SET DEFAULT 0,
  ALGORITHM=INSTANT;

-- 如果不支持 INSTANT，立即报错，不会降级为锁表操作
```

---

#### 五、操作 SOP（标准流程）

```
1. 评估表大小
   SELECT table_rows, data_length/1024/1024 AS size_mb
   FROM information_schema.TABLES
   WHERE table_name = 'orders';

2. 低峰期执行（凌晨 2~4 点）

3. 先在从库验证一遍

4. 执行前监控好以下指标：
   - Threads_running（活跃线程数）
   - 主从复制延迟
   - 慢查询数量

5. 使用 gh-ost / pt-osc 执行，保留 --no-drop-old-table

6. 观察 15~30 分钟，确认无异常后删除旧表

7. 回滚预案：
   - gh-ost 写入 panic-flag 文件立即中止
   - 旧表仍在 → RENAME 回来即可
```

---

#### 六、PostgreSQL Live 改表

PostgreSQL 对部分 DDL 更友好，但 `ACCESS EXCLUSIVE LOCK` 同样危险：

```sql
-- ✅ 加列有默认值（PG 11+ 瞬间完成，不重写表）
ALTER TABLE orders ADD COLUMN remark TEXT DEFAULT '';

-- ✅ 并发建索引，不锁写操作
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- ⚠️ 设置超时保护，避免长时间等锁
SET lock_timeout = '3s';    -- 超3秒获取不到锁直接失败
SET statement_timeout = '0'; -- DDL 本身不限制执行时长
```

> **总结：** Live 改表 = **gh-ost/pt-osc + 低峰期 + 监控 + 回滚预案**，绝不裸跑 `ALTER TABLE`。
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
### 你碰到过的数据库优化最难的问题，及如何解决
#### 碰到过的数据库优化最难的问题及解决过程

---

#### 问题一：千万级订单表分页查询，越翻越慢

##### 背景

订单列表页支持按时间倒序翻页，前几页响应正常，翻到第 500 页之后接口直接超时。

##### 根因分析

```sql
-- 原始写法
SELECT * FROM orders ORDER BY created_at DESC LIMIT 500000, 20;
```

`LIMIT 500000, 20` 的本质是：**扫描并丢弃前 50 万行，再返回 20 行**，即使有索引也逃不掉这个代价：

```
第    1 页：LIMIT 0,      20  → 扫描 20 行
第    2 页：LIMIT 20,     20  → 扫描 40 行
第  100 页：LIMIT 1980,   20  → 扫描 2000 行
第 5000 页：LIMIT 100000, 20  → 扫描 100020 行  ← 超时
```

##### 解决过程

###### 方案一：主键游标分页（最优解）

不用 OFFSET 跳过数据，改为记录**上一页最后一条 id**，下次从这个位置继续往后取：

```sql
-- 第1页（无游标，正常查）
SELECT id, user_id, amount, status, created_at
FROM orders
ORDER BY id DESC
LIMIT 20;
-- 返回：id = 100, 99, 98 ... 81，最后一条 id = 81

-- 第2页：从 id=81 之前继续取（书签式翻页）
SELECT id, user_id, amount, status, created_at
FROM orders
WHERE id < 81          -- 游标：上一页最后一条的 id
ORDER BY id DESC
LIMIT 20;
-- 返回：id = 80, 79, 78 ... 61，最后一条 id = 61

-- 第3页：从 id=61 之前继续
SELECT id, user_id, amount, status, created_at
FROM orders
WHERE id < 61
ORDER BY id DESC
LIMIT 20;
```

**直观理解：**

```
id：100, 99, 98 ... 81 | 80, 79, 78 ... 61 | 60, 59 ...
     ←────第1页──────→    ←────第2页──────→
                        ↑
                   WHERE id < 81
                   书签，从这里继续
```

**为什么主键游标不会回表：**

```
InnoDB 聚簇索引结构：
  叶子节点 = 主键id + 完整行数据

WHERE id < 81 → 直接走聚簇索引 → 叶子节点已含全部字段 → 无需回表 ✅

对比普通索引（如 created_at）：
  辅助索引叶子节点 = created_at + id（只有主键）
  → 还需拿 id 回聚簇索引再查一次完整行  ← 回表 ❌
```

**翻页性能对比：**

| | OFFSET 分页 | 主键游标分页 |
|---|---|---|
| 第 1 页 | 扫描 20 行 | 扫描 20 行 |
| 第 100 页 | 扫描 2000 行 | 扫描 20 行 |
| 第 5000 页 | 扫描 10 万行 | 扫描 20 行 |
| 耗时趋势 | 越翻越慢 📈 | 始终恒定 ✅ |

**效果：** 从超时 → 稳定 8ms，无论翻到第几页耗时一致。

**代价：** 不支持跳页，产品侧改为"加载更多"交互模式。

---

###### 方案二：延迟关联（需要支持跳页时）

```sql
-- ❌ 原始：直接 OFFSET，回表 50 万次
SELECT * FROM orders ORDER BY id DESC LIMIT 500000, 20;

-- ✅ 延迟关联：子查询只扫覆盖索引，找到目标 id 后只回表 20 次
SELECT o.*
FROM orders o
JOIN (
    SELECT id FROM orders
    ORDER BY id DESC
    LIMIT 500000, 20    -- 覆盖索引，不回表
) tmp ON o.id = tmp.id;
```

---

###### 方案三：有筛选条件时配合联合索引

```sql
-- 查某个用户的订单，按 id 倒序翻页
CREATE INDEX idx_user_id ON orders(user_id, id DESC);

SELECT id, amount, status, created_at
FROM orders
WHERE user_id = 123 AND id < 8888888
ORDER BY id DESC
LIMIT 20;
-- 命中联合索引，精准定位，只扫 20 行 ✅
```

---

###### 接口设计示例

```
请求：GET /orders?size=20&last_id=81

后端 SQL：
  WHERE id < 81（last_id 为空时不加，表示第一页）
  ORDER BY id DESC LIMIT 20

响应：
  { "data": [...], "next_cursor": 61, "has_more": true }
```

---

###### 三种方案对比

| 方案 | 支持跳页 | 性能 | 改造成本 |
|------|---------|------|---------|
| 主键游标分页 | ❌ | 🟢 最优，始终 O(1) | 中（前端记录游标） |
| 延迟关联 | ✅ | 🟡 较好，深分页仍有代价 | 低（只改 SQL） |
| Elasticsearch | ✅ | 🟢 最优 | 高 |

---

#### 问题二：读写分离后，数据不一致导致 Bug

##### 背景

引入主从读写分离后，用户下单后立刻跳转订单详情页，概率性出现"订单不存在"。

##### 根因分析

```
用户下单 → 写主库 → 立刻读从库
                         ↑
              主从复制有 50~200ms 延迟
              从库还没同步到这条数据 → 查不到 → "订单不存在"
```

##### 解决过程

```
① 强一致读：写操作完成后的下一次读，强制走主库
  → ORM 层标记：刚写过的请求，本次会话内读主库

② 半同步复制：开启 semi-sync，至少 1 个从库确认收到 binlog
  才返回写成功，从根本上将延迟从 200ms 压到 20ms 以内
```

最终方案：**① + ② 结合**，核心写操作后强制读主库 + 半同步复制兜底。

---

#### 问题三：EXPLAIN 全是索引，但查询还是慢

##### 背景

一条 SQL 的 EXPLAIN 显示走了索引，rows 也不大，但线上 P99 延迟高达 3 秒。

##### 根因分析

```sql
-- EXPLAIN 看起来正常
EXPLAIN SELECT ...
-- type=ref, key=idx_user_id, rows=200  ← 一切正常？

-- 用 EXPLAIN ANALYZE 看实际耗时
EXPLAIN ANALYZE SELECT ...
-- actual time=2800ms  ← 和预估完全不符！

-- 查看锁状态
SHOW ENGINE INNODB STATUS\G
-- 发现大量 lock wait，有事务持锁未释放

-- 找到罪魁祸首
SELECT * FROM information_schema.innodb_trx\G
-- 有一个事务开启了 40 分钟，持有大量行锁！
```

发现代码里：**事务内部做了外部 HTTP 调用**，HTTP 超时导致事务迟迟不提交，行锁一直持有。

##### 解决过程

```
① 立即 kill 掉超长事务（应急）

② 修改代码：
   事务内只做纯数据库操作
   HTTP 调用移到事务提交之后执行

③ 加监控告警：
   事务执行超过 5s 自动告警
   SET innodb_lock_wait_timeout = 10;  -- 等锁超过10s直接报错
```

---

#### 问题四：加了索引反而更慢

##### 背景

给 `status` 字段加了索引，EXPLAIN 显示命中，但整体查询反而比全表扫描慢。

##### 根因分析

```
status 字段只有 4 个值，数据分布极度不均：
  paid       → 占 85% 的数据
  pending    → 占 10%
  cancelled  → 占  4%
  refunded   → 占  1%
```

查 `status = 'paid'` 时命中 85% 的行，MySQL 优化器判断：**回表代价 > 全表扫描**，但有时仍强行走索引，反而更慢。

##### 解决过程

```sql
-- 方案一：低基数字段改为联合索引，加上时间范围缩小扫描
CREATE INDEX idx_status_created ON orders(status, created_at DESC);

SELECT * FROM orders
WHERE status = 'paid' AND created_at > '2024-01-01'
ORDER BY created_at DESC LIMIT 20;

-- 方案二：应急强制忽略该索引
SELECT * FROM orders IGNORE INDEX(idx_status) WHERE status = 'paid';

-- 方案三：归档历史数据，主表只保留近3个月数据，总量大幅下降
```

---

#### 四个问题的共同规律

```
问题一（深分页）   → 索引设计不适配业务访问模式
问题二（主从延迟） → 架构引入了一致性窗口，业务代码没有感知
问题三（锁等待）   → 慢不等于 SQL 本身慢，要看等待链路全貌
问题四（索引失效） → 数据分布决定索引价值，不是加了索引就一定快
```

> **核心经验：数据库优化最难的地方不是技术，是"表象和根因之间的距离"——EXPLAIN 正常、索引命中，不代表查询就快，要深入到锁、事务、数据分布、架构链路去找真正的瓶颈。**

### mysql 索引覆盖，回表 （滴滴）
  - SQL只需要通过索引就可以返回查询所需要的数据，而不必通过二级索引查到主键之后再去查询数据
### 忘了加唯一索引有啥补救措施吗
  - 只是参考答案：新增别出现相同数据就好
  - 添加唯一索引，检查重复数据，重新倒入数据，做好数据备份
### 在唯一索引的约束下，如何优雅地软删除
### 需求: 一张表中有个字段appid，同一个appid只允许存在一行正常记录，但是可以存在多条软删除记录
  - 答案：额外加一个status字段，0为正常，非零为已删除。和appid作为复合唯一索引，软删除的时候将status改为当前时间戳
### mysql索引的类型，各自的特点，还有索引失效的情况
### 腾讯外包公司题：

###  mysql唯一索引是否可以为null？为什么？、
#### MySQL唯一索引是否可以为NULL
**可以，且允许存在多个 NULL 值。**

##### 核心原因
1. **唯一索引约束的是“非NULL值”唯一**
   - 唯一索引只限制**非NULL字段值不能重复**
   - 对 NULL 值不做唯一性校验
2. **NULL 在数据库中代表“未知值”**
   - SQL 标准规定：**NULL 不等于任何值，也不等于自身**
   - 多个 NULL 之间不判定为重复，因此可以共存
3. **业务与底层实现**
   - 唯一索引允许字段为 NULL，且能插入多条 NULL 记录
   - 主键索引**不允许 NULL**，且只能有一条记录

##### 一句话背诵版（面试直接说）
**MySQL 唯一索引允许为 NULL，且可以存在多个 NULL。因为 NULL 代表未知值，不参与唯一性判断，唯一约束只限制非 NULL 值不能重复。**

### select for update是表锁还是行锁？（仔细查找答案，有坑）
    - 如果查询条件用了索引/主键，那么select ..... for update就会进行行锁。
    - 如果是普通字段(没有索引/主键)，那么select ..... for update就会进行锁表。
### 乐观锁和悲观锁数据库层面如何实现？
    - [refer](https://blog.csdn.net/just_learing/article/details/124898579?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166779193316782417037044%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=166779193316782417037044&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-3-124898579-null-null.142^v63^control,201^v3^control_1,213^v1^control&utm_term=%E4%B9%90%E8%A7%82%E9%94%81%E5%92%8C%E6%82%B2%E8%A7%82%E9%94%81%E6%95%B0%E6%8D%AE%E5%BA%93%E5%B1%82%E9%9D%A2%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0&spm=1018.2226.3001.4187)
  - 缓存数据和数据库数据如何实现一致性？


### 使用索引查询一定能提高查询的性能吗？为什么
  - 不一定
  - 索引需要额外的存储空间和处理,那些不必要的索引反而会使查询反应时间变慢.使用索引查询不一定能提高查询性能
### 事务还没提交的时候，redolog 能不能被持久化到磁盘呢(字节一面)
  - [解释参考](https://mp.weixin.qq.com/s/kdPb4v5nOu0LMCj8s1ETNg)
#### 事务未提交时，redo log 能否持久化到磁盘
**可以，而且一定会发生。**
这是 MySQL 最经典的面试题之一，答案非常明确：

##### 核心结论
**事务未提交时，redo log 完全可以、并且经常会被持久化（刷盘）到磁盘。**

##### 详细原理（面试标准答案）
1. **redo log 的刷盘时机，不依赖事务是否提交**
   - InnoDB 有一个**后台线程**，每隔 **1秒** 就会把 redo log buffer 里的日志**强制刷到磁盘**。
   - 哪怕事务还在执行、还没执行 `commit`，只要时间到了，就会刷盘。

2. **刷盘的触发条件**
   - 每隔 1 秒，后台线程自动刷盘
   - redo log buffer 占用达到 **50%**，自动刷盘
   - 事务执行 `commit` 时，**强制刷盘**（保证持久性）

3. **为什么未提交也要刷盘？**
   - 防止数据库崩溃时，**已经执行但未提交的操作也能恢复**
   - 崩溃恢复时，未提交的事务会通过 undo log 回滚，已提交的通过 redo log 重做

##### 一句话背诵版（字节面试直接说）
**事务没提交，redo log 也能持久化到磁盘。因为 InnoDB 有后台线程每隔1秒、或日志缓冲区满了就会刷盘，不依赖 commit。**
### mysql 联合查询用法
  - INNER JOIN(等值连接) 只返回两个表中联结字段相等的行
  - LEFT JOIN(左联接) 返回包括左表中的所有记录和右表中联结字段相等的记录
  - RIGHT JOIN(右联接) 返回包括右表中的所有记录和左表中联结字段相等的记录
  - SELECT * FROM 表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号
  - SELECT * FROM (表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号) INNER JOIN 表3 ON 表1.字段号=表3.字段号
  - SELECT * FROM ((表1 INNER JOIN 表2 ON 表1.字段号=表2.字段号) INNER JOIN 表3 ON 表1.字段号=表3.字段号) INNER JOIN 表4 ON Member.字段号=表4.字段号
### mysql group用法
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
### mysql group by  having 和 where 执行顺序
### mysql 索引有哪些
  - 1.普通索引
  - 2.唯一索引
  - 3.主键索引
  - 4.组合索引
  - 5.全文索引
    - fulltext索引
### mysql 主键索引和二级索引有什么区别
  - mysql中每个表都有一个聚簇索引（clustered index ），除此之外的表上的每个非聚簇索引都是二级索引，又叫辅助索引（secondary indexes）
  - 
### mysql 做过哪些优化

【有道云笔记】04.SQL优化.md
https://share.note.youdao.com/s/TnWDz50Q

#### 1. SQL 语句优化
##### 1.1 查询语句优化
###### 1.1.1 禁止使用 SELECT *
增加 CPU、IO、内存、网络带宽消耗，无法使用覆盖索引，必须回表查询。
###### 1.1.2 使用 LIMIT 控制返回条数
读取适当记录，避免查询多余数据，减少网络传输和内存占用。
###### 1.1.3 避免数据类型不一致
字符串不加引号、数字与字符串比较，会导致索引失效、全表扫描。
###### 1.1.4 避免负向查询条件
NOT、!=、<>、IS NULL、IS NOT NULL 无法使用索引。
###### 1.1.5 避免随机取记录
ORDER BY RAND() 性能极差，禁止使用。
###### 1.1.6 禁止不必要的 ORDER BY 排序
排序字段无索引且数据量大时，消耗大量 CPU 和 IO。

##### 1.2 子查询与连接优化
###### 1.2.1 尽量避免子查询，改用 JOIN
子查询会产生临时表，效率远低于 JOIN。
###### 1.2.2 用 IN 替换 OR
OR 连接多条件容易导致索引失效，IN 列表值不宜过多。
###### 1.2.3 合理使用 IN 和 EXISTS
- IN：先执行子查询，再匹配外表，适用于外表大、内表小。
- EXISTS：先遍历外表，再子查询校验，适用于外表小，内表大。

##### 1.3 聚合与合并结果优化
###### 1.3.1 分组统计禁止额外排序
GROUP BY 默认排序，可加 ORDER BY NULL 禁止排序。
###### 1.3.2 优先使用 UNION ALL
- UNION：合并、去重、排序，资源消耗高。
- UNION ALL：直接合并，无去重无排序，效率更高。
- 使用前提：明确结果集无重复数据。

##### 1.4 插入与写入优化
###### 1.4.1 批量 INSERT 插入
多行合并插入，减少事务和连接开销。

#### 2. 索引优化
##### 2.1 索引设计原则
###### 2.1.1 遵循最佳左前缀法则
组合索引必须满足最左前缀才能命中。
###### 2.1.2 高区分度字段放前面
提升索引筛选效率。
###### 2.1.3 使用覆盖索引
查询列与索引列一致，避免回表。
###### 2.1.4 不在索引列上做操作
计算、函数、类型转换会导致索引失效。

##### 2.2 JOIN 语句优化
###### 2.2.1 小结果集驱动大结果集
减少 NestedLoop 循环次数。
###### 2.2.2 被驱动表关联字段建索引
###### 2.2.3 无法建索引时调大 join_buffer_size
###### 2.2.4 优先使用 INNER JOIN
优化器自动选择小表驱动大表。
###### 2.2.5 LEFT JOIN 优化
驱动表提前过滤变小，右表关联字段建索引。
###### 2.2.6 适当冗余字段
减少 JOIN 次数。

#### 3. 索引失效场景
##### 3.1 查询条件导致失效
###### 3.1.1 LIKE 以通配符开头
%abc、_abc 索引失效，abc% 有效。
###### 3.1.2 OR 连接且有字段无索引
整体索引失效。
###### 3.1.3 使用负向条件
!=、<>、IS NULL、IS NOT NULL、NOT IN。
###### 3.1.4 WHERE 判断字段 NULL
无法使用索引。

##### 3.2 索引列操作导致失效
###### 3.2.1 索引列使用函数
###### 3.2.2 索引列进行计算
###### 3.2.3 索引列发生类型转换
字符串不加单引号等。

##### 3.3 组合索引使用不当
###### 3.3.1 违反最佳左前缀法则
###### 3.3.2 范围条件右边列失效
范围条件后索引列无法命中。

##### 3.4 优化器选择导致失效
优化器判断全表扫描更快，或数据区分度极低、分布不均。

#### 4. 补充优化点
##### 4.1 避免函数索引
非必要不建立。
##### 4.2 控制 JOIN 表数量
##### 4.3 无索引尽量不排序
##### 4.4 使用 SSD 提升 IO 性能

---

#### 最终精简版（面试背诵）
##### 一、SQL 优化
不用 SELECT *，用 LIMIT，少排序；
避免子查询，改用 JOIN；
优先 UNION ALL；
批量 INSERT，避免负向查询。

##### 二、索引优化
遵循左前缀，高区分度在前；
使用覆盖索引，禁止索引列运算；
JOIN 小表驱动大表，被驱动表建索引。

##### 三、索引失效场景
左模糊 %abc；
OR 无索引、类型不一致、函数运算；
负向查询、判断 NULL；
组合索引跨列使用。


### 相对B树，B+树做索引的优势
  - B+树的磁盘读写代价更低
    - B+树的内部节点并没有指向关键字具体信息的指针，因此其内部节点相对B树更小
    - 把所有同一内部节点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多
    - 读取相同的数据量，io次数相对减少
  - B+树的查询效率更加稳定
  - B+树只需要去遍历叶子节点就可以实现整棵树的遍历，遍历效率高
### Mysql 默认的隔离级别是什么？在 Innodb 的可重复读的情况下可以解决幻读的情况吗？（字节）
  - 可重复读
  - MySQL可重复读的隔离级别中并不是完全解决了幻读的问题，而是解决了读数据情况下的幻读问题。而对于修改的操作依旧存在幻读问题。
### 如何解决幻读？
  - 第一种方式 使用串行化读的隔离级别
  - 第二种方式 MVCC+next-key locks：next-key locks由record locks(索引加锁) 和 gap locks(间隙锁，每次锁住的不光是需要使用的数据，还会锁住这些数据附近的数据)
### 如何对数据库进行分库分表，不允许停止服务
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

#
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
        - redo log是循环写的，空间固定会用完；
        - binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志
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
    - 执行流程 两阶段提交（https://blog.csdn.net/qq_33591903/article/details/122030252）
      ```
#      
- mysql 半同步介绍下
  - 主库只需要等待至少一个从节点，收到并且flush binlog到relay log文件即可，
  - 主库不需要等待所有从库给主库反馈，这里只是一个收到的反馈，而并不是从库已经完成并提交的反馈，
  - 即从库只应用完成io_thread内容即可无需等到sql_thread的执行完成
- mysql 事务原子性怎么实现的
- 分库分表策略
### 分表键和查询条件不一致咋整
#### 分表键和查询条件不一致怎么办

---

#### 一、先理解问题场景

```
假设订单表按 user_id 分片：
  orders_0 → user_id % 4 = 0
  orders_1 → user_id % 4 = 1
  orders_2 → user_id % 4 = 2
  orders_3 → user_id % 4 = 3

正常查询（带分表键）：
  SELECT * FROM orders WHERE user_id = 123;
  → 直接路由到 orders_3 ✅ 只查一张表

非正常查询（不带分表键）：
  SELECT * FROM orders WHERE order_no = 'SN202401010001';
  SELECT * FROM orders WHERE status = 'paid';
  → 不知道去哪张表找 ❌ 只能全部扫描
```

```
全分片扫描的代价：
  4 张表还好
  分了 64 张表 → 同时查 64 张表
  分了 1024 张表 → 同时查 1024 张表
  每次查询都是全量扫描，性能极差
```

---

#### 二、解决方案一：建立映射表（索引表）

用一张单独的表记录**其他查询条件 → 分表键**的映射关系：

```sql
-- 映射表（不分片，单独存储）
CREATE TABLE order_index (
    order_no  VARCHAR(64),    -- 其他查询条件
    user_id   BIGINT,         -- 分表键
    PRIMARY KEY (order_no),
    INDEX idx_user_id (user_id)
);
```

##### 查询流程

```
业务方用 order_no 查订单：

第一步：查映射表，拿到 user_id
  SELECT user_id FROM order_index WHERE order_no = 'SN202401010001';
  → user_id = 123

第二步：用 user_id 路由到正确分片
  123 % 4 = 3 → orders_3

第三步：查目标分片
  SELECT * FROM orders_3 WHERE order_no = 'SN202401010001';
  → 返回结果 ✅

总共两次查询，精准定位，无全表扫描
```

##### 写入时同步维护映射表

```java
@Transactional
public void createOrder(Order order) {
    // 1. 写入分片表
    orderMapper.insert(order);  // 路由到 orders_3

    // 2. 同步写入映射表
    orderIndexMapper.insert(new OrderIndex(
        order.getOrderNo(),
        order.getUserId()
    ));
}
```

##### 映射表的问题

```
问题：映射表本身成为单点瓶颈
  → 所有查询都要先查映射表，高并发下压力大

解决：
  映射表加 Redis 缓存
  order_no → user_id 缓存到 Redis
  缓存命中直接拿 user_id，不查 DB
```

---

#### 三、解决方案二：基因法（冗余分表键信息）

在生成 order_no 时，把 user_id 的路由信息**编码进去**，查询时从 order_no 解析出分片信息：

```
设计 order_no 格式：

  SN + 时间戳 + user_id后4位 + 随机数

  例：user_id = 123456
      order_no = SN20240101_6789_1234 56_0023
                                        ↑
                                  user_id 后两位 = 56
                                  56 % 4 = 0 → 路由到 orders_0
```

##### 查询时直接解析路由

```java
public Order getByOrderNo(String orderNo) {
    // 从 order_no 中提取路由基因
    String gene = orderNo.substring(15, 17);  // 提取 user_id 后两位
    int shardIndex = Integer.parseInt(gene) % 4;

    // 直接路由到对应分片，无需查映射表
    return orderMapper.getByOrderNo(shardIndex, orderNo);
}
```

```
优点：
  不需要映射表，少一次查询
  order_no 本身携带路由信息

缺点：
  order_no 生成规则复杂，需要提前设计好
  后期分片数变化，老数据解析规则会对不上
  对业务有侵入性
```

---

#### 四、解决方案三：冗余写入多份

同一条数据**按不同维度写入多张表**：

```
场景：订单既要按 user_id 查，又要按 merchant_id 查

写入时冗余两份：
  orders_by_user     → 按 user_id 分片（用户端查自己的订单）
  orders_by_merchant → 按 merchant_id 分片（商家端查自己店铺的订单）
```

```java
@Transactional
public void createOrder(Order order) {
    // 按 user_id 写一份
    orderByUserMapper.insert(order);      // 路由到 user 分片

    // 按 merchant_id 再写一份
    orderByMerchantMapper.insert(order);  // 路由到 merchant 分片
}
```

```
查询时直接路由：
  用户查自己的订单   → orders_by_user     WHERE user_id = ?
  商家查店铺订单     → orders_by_merchant WHERE merchant_id = ?

优点：查询时直接路由，无需二次查询
缺点：存储翻倍，写入需要保证两份数据一致性
```

---

#### 五、解决方案四：ES 承接非分表键查询

把数据同步到 Elasticsearch，复杂条件查询走 ES：

```
架构：

写入数据
  ↓
MySQL 分片表（按 user_id 分片）
  ↓ binlog 同步（Canal/Flink）
Elasticsearch（全量索引，支持任意字段查询）

查询路由：
  带 user_id   → 直接走 MySQL 分片（精准快速）
  不带 user_id → 走 ES 查询（灵活支持任意条件）
               → ES 返回 order_id 列表
               → 再用 order_id 回 MySQL 查完整数据
```

```java
public List<Order> searchOrders(OrderQuery query) {
    if (query.getUserId() != null) {
        // 有分表键，直接路由 MySQL
        return orderMapper.getByUserId(query.getUserId());
    } else {
        // 无分表键，走 ES
        List<Long> orderIds = esOrderService.search(query);
        return orderMapper.getByIds(orderIds);
    }
}
```

```
优点：
  ES 天然支持全文检索、多条件组合查询
  MySQL 只负责精准查询，各司其职

缺点：
  引入 ES，架构复杂度增加
  数据同步有延迟（秒级）
  需要保证 MySQL 和 ES 数据一致性
```

---

#### 六、四种方案对比

| 方案 | 额外查询 | 存储开销 | 实时性 | 复杂度 | 适合场景 |
|------|---------|---------|--------|--------|---------|
| 映射表 | 多一次 DB 查询 | 小 | 实时 | 低 | 查询维度少（1~2个） |
| 基因法 | 无 | 无 | 实时 | 中 | 提前规划，查询维度固定 |
| 冗余写入 | 无 | 翻倍 | 实时 | 中 | 写少读多，维度固定 |
| ES 承接 | 多一次 ES 查询 | 大 | 秒级延迟 | 高 | 查询条件复杂多变 |

---

#### 七、选型建议

```
查询维度固定（1~2个）且数据量不大
  → 映射表 + Redis 缓存

提前规划好，order_no 能编码路由信息
  → 基因法（最优雅，无额外查询）

写入不频繁，查询维度固定（如买家/卖家双视角）
  → 冗余写入

查询条件复杂多变（后台管理、报表、搜索）
  → ES 承接非分表键查询
```

> **核心：分表键选好是前提，尽量让 80% 的查询都带分表键。剩下 20% 不带分表键的查询，用映射表或 ES 兜底，避免全分片扫描。**

### 缓存和数据库的一致性怎么保证,当不一致的时候如何解决
  - [refer](https://www.processon.com/view/link/620af9b50e3e7429dd02be21)

### innodb是如何存储数据的
  - [refer](https://mp.weixin.qq.com/s/665zAn_PuTqAl_rJa_5Ilg)
  
### redolog 落盘
  - https://blog.csdn.net/weixin_40471676/article/details/119732738

### mysql主备如何保证数据同步 
  - https://www.processon.com/view/link/620f131c1efad406e7332e92

### show processlist 命令详解 
  - https://blog.csdn.net/dhfzhishi/article/details/81263084
#### SHOW PROCESSLIST 命令详解

---

#### 一、是什么

```sql
-- 查看当前 MySQL 所有连接线程的状态
SHOW PROCESSLIST;

-- 显示完整 SQL（不截断）
SHOW FULL PROCESSLIST;
```

```
作用：
  实时查看哪些连接在执行什么操作
  排查慢查询、锁等待、连接堆积等问题
  是数据库问题排查的第一现场
```

---

#### 二、输出字段详解

```sql
SHOW FULL PROCESSLIST;

-- 输出示例：
+------+------+-----------+--------+---------+------+------------------------+------------------+
| Id   | User | Host      | db     | Command | Time | State                  | Info             |
+------+------+-----------+--------+---------+------+------------------------+------------------+
| 101  | app  | 10.0.0.1  | mydb   | Query   | 0    | executing              | SELECT * FROM... |
| 102  | app  | 10.0.0.2  | mydb   | Query   | 30   | Waiting for lock       | UPDATE orders... |
| 103  | app  | 10.0.0.3  | mydb   | Sleep   | 600  |                        | NULL             |
| 104  | root | localhost | mydb   | Query   | 120  | Copying to tmp table   | SELECT * FROM... |
+------+------+-----------+--------+---------+------+------------------------+------------------+
```

##### Id（线程 ID）

```
每个连接的唯一标识
排查到问题线程后，用这个 ID 执行 KILL

KILL 102;        -- 终止 102 号线程的查询，保留连接
KILL QUERY 102;  -- 只终止查询，不断开连接
```

##### User（用户名）

```
发起连接的数据库用户
可以判断是哪个应用或服务发来的请求

system user  → MySQL 内部线程（主从复制 IO/SQL 线程）
root         → 通常是 DBA 或运维操作
app          → 业务应用
```

##### Host（来源地址）

```
格式：IP:端口

10.0.0.1:52345  → 来自应用服务器
localhost       → 本机连接

通过 IP 可以定位是哪台应用服务器发来的请求
```

##### db（当前数据库）

```
当前连接使用的数据库
NULL 表示没有选择数据库
```

##### Command（命令类型）

```
Query    → 正在执行 SQL 查询      ← 重点关注
Sleep    → 空闲连接，等待客户端发命令
Connect  → 正在建立连接
Execute  → 正在执行预处理语句
Quit     → 正在断开连接

Sleep 时间过长（几百秒）→ 连接池没有回收，浪费连接资源
```

##### Time（持续时间，单位秒）

```
当前 Command 已持续的秒数

Query  + Time 很大  → 慢查询，需要排查
Sleep  + Time 很大  → 僵尸连接，可以 KILL
Lock   + Time 很大  → 锁等待，需要找持锁者

Time > 30s  → 需要关注
Time > 300s → 必须处理
```

##### State（线程状态）

```
最关键的字段，反映线程当前在做什么
```

常见 State 值及含义：

```
State                          含义                    处理
──────────────────────────────────────────────────────────────────
executing                    正在执行查询               正常
Waiting for lock             等待表锁                  找持锁者 KILL
Waiting for row lock         等待行锁                  找持锁者 KILL
Copying to tmp table         结果集写入临时表            SQL 需优化（GROUP BY/ORDER BY 无索引）
Sorting result               对结果排序                SQL 需优化（无索引排序）
Sending data                 向客户端发送数据            结果集过大，考虑分页
Opening tables               正在打开表                 表数量过多或表锁竞争
Locked                       被锁住                    找持锁者 KILL
Creating sort index          创建排序索引               filesort，SQL 需优化
init                         初始化查询                 正常，很快结束
statistics                   计算执行计划统计信息         正常
NULL / 空                    Sleep 状态，无操作          正常
```

##### Info（正在执行的 SQL）

```
SHOW PROCESSLIST    → SQL 超过 100 字符会截断
SHOW FULL PROCESSLIST → 显示完整 SQL

NULL → Sleep 状态，没有执行 SQL
```

---

#### 三、实战排查场景

##### 场景一：排查慢查询

```sql
-- 找出执行超过 10 秒的查询
SELECT id, user, host, db, time, state, info
FROM information_schema.processlist
WHERE command = 'Query'
  AND time > 10
ORDER BY time DESC;

-- 输出：
-- id=104, time=120, state='Copying to tmp table'
-- info='SELECT * FROM orders GROUP BY status ORDER BY created_at'

-- 分析：state 是 Copying to tmp table，说明有大量数据写临时表
-- 优化：给 status 和 created_at 加联合索引
```

##### 场景二：排查锁等待

```sql
-- 找出等锁的线程
SELECT id, user, time, state, info
FROM information_schema.processlist
WHERE state LIKE '%lock%'
ORDER BY time DESC;

-- 输出：
-- id=102, time=30, state='Waiting for row lock'
-- info='UPDATE orders SET status=? WHERE id=888'

-- 说明：102 号线程在等待 id=888 的行锁
-- 需要找到持有这把锁的线程
```

```sql
-- 找持锁者
SELECT
    r.trx_id             AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    b.trx_id             AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_started        AS blocking_started,
    TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_seconds
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;

-- 找到持锁线程 id=101，持锁已 300 秒
-- KILL 101;  ← kill 掉，释放锁
```

##### 场景三：排查连接堆积

```sql
-- 统计各状态连接数
SELECT command, state, COUNT(*) AS cnt
FROM information_schema.processlist
GROUP BY command, state
ORDER BY cnt DESC;

-- 输出：
-- Sleep   NULL                    180   ← 大量空闲连接
-- Query   Waiting for row lock     45   ← 大量锁等待
-- Query   Copying to tmp table     12   ← 大量临时表操作

-- 180 个 Sleep 连接 → 连接池配置过大，或连接没有及时释放
-- 45 个锁等待     → 有长事务持锁，找到持锁者 KILL
```

##### 场景四：批量 KILL 问题连接

```sql
-- 生成 KILL 语句（Sleep 超过 600 秒的僵尸连接）
SELECT CONCAT('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep'
  AND time > 600;

-- 输出：
-- KILL 103;
-- KILL 107;
-- KILL 115;

-- 复制输出批量执行
```

---

#### 四、information_schema.processlist 更灵活

```sql
-- SHOW PROCESSLIST 不方便加条件筛选
-- 用 information_schema.processlist 代替，支持 WHERE/GROUP BY

-- 查找执行时间最长的前10个查询
SELECT id, user, host, db, time, state,
       SUBSTR(info, 1, 100) AS sql_preview
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC
LIMIT 10;
```

---

#### 五、performance_schema 更详细

```sql
-- MySQL 5.7+ 推荐用 performance_schema，信息更全
SELECT
    t.processlist_id      AS thread_id,
    t.processlist_user    AS user,
    t.processlist_host    AS host,
    t.processlist_time    AS time,
    t.processlist_state   AS state,
    t.processlist_command AS command,
    s.sql_text            AS full_sql      -- 完整 SQL，不截断
FROM performance_schema.threads t
LEFT JOIN performance_schema.events_statements_current s
       ON t.thread_id = s.thread_id
WHERE t.processlist_command != 'Sleep'
ORDER BY t.processlist_time DESC;
```

---

#### 六、监控告警建议

```sql
-- 以下情况需要立即告警：

-- 1. 等锁线程数 > 10
SELECT COUNT(*) FROM information_schema.processlist
WHERE state LIKE '%lock%';

-- 2. 执行超过 30 秒的查询
SELECT COUNT(*) FROM information_schema.processlist
WHERE command = 'Query' AND time > 30;

-- 3. 总连接数接近上限
SHOW STATUS LIKE 'Threads_connected';  -- 当前连接数
SHOW VARIABLES LIKE 'max_connections'; -- 最大连接数
-- 超过 80% 需要告警
```

---

#### 七、总结

```
SHOW PROCESSLIST 排查步骤：

发现问题
  ↓
SHOW FULL PROCESSLIST
  ↓
重点看：Command=Query + Time 较大的线程
  ↓
看 State：
  Waiting for lock    → 找持锁者，KILL 持锁线程
  Copying to tmp table → SQL 需要优化，加索引
  Sorting result      → 排序无索引，加索引
  Sending data        → 结果集太大，加分页
  ↓
定位 SQL（Info 字段）→ EXPLAIN 分析 → 优化
```

> **核心：`SHOW PROCESSLIST` 是数据库问题排查的第一步，Time 字段告诉你"慢了多久"，State 字段告诉你"卡在哪里"，Info 字段告诉你"是哪条 SQL"，三个字段结合就能快速定位大多数数据库问题。**


### 如何判断数据库是否出问题了 
  - https://www.processon.com/view/link/620f17ce6376897c8c7bce60

### mysql group by having 和 where 执行顺序 
  - 先执行where 再执行 group by 再执行 having
#### MySQL 故障判断完整指南

---

#### 一、快速连通性判断

##### 方法1：select 1（最简单）

```sql
select 1;
```

**能判断：**
- 数据库进程是否存活
- 网络连接是否正常

**不能判断：**
- 并发线程是否打满
- 磁盘是否异常
- 实际业务表是否可读写

> ⚠️ select 1 返回正常 ≠ 数据库完全健康，只是最基础的存活检测。

---

#### 二、并发线程过多的判断

##### 核心参数

```sql
-- 查看当前并发线程上限
SHOW VARIABLES LIKE 'innodb_thread_concurrency';

-- 建议设置范围
SET GLOBAL innodb_thread_concurrency = 64; -- 推荐 64~128
```

##### 概念区分（重要）

| 概念 | 含义 | 数量级 |
|---|---|---|
| **并发连接** | 已建立的连接数（show processlist） | 可达数千 |
| **并发线程** | 真正在 CPU 上执行的线程 | 受 innodb_thread_concurrency 限制 |

> 看到 `show processlist` 有 3000 个连接 ≠ 有 3000 个线程在跑，行锁/间隙锁等待的线程不计入并发线程数，也不消耗 CPU。

##### 方法2：查表检测

```sql
-- 建立专用健康检测表
CREATE TABLE mysql.health_check (
    id  INT PRIMARY KEY,
    t_modified TIMESTAMP
);

-- 执行查表检测
SELECT * FROM mysql.health_check;
```

**比 select 1 多检测了什么：**
- 当并发线程数已满，新请求会排队
- `select 1` 是内部操作不进线程池，所以仍会成功
- `select * from mysql.health_check` 需要真正进入 InnoDB 引擎执行，能检测出线程打满的问题

##### 模拟并发线程打满（测试验证）

```sql
-- session1
SELECT sleep(100) FROM t;

-- session2
SELECT sleep(100) FROM t;

-- session3（此时若 innodb_thread_concurrency=2，这条会被阻塞）
SELECT sleep(100) FROM t;
```

---

#### 三、数据库更新能力的判断

##### 方法3：更新检测（最精准）

```sql
-- 建表
CREATE TABLE mysql.health_check (
    id         INT            PRIMARY KEY,
    t_modified TIMESTAMP      DEFAULT NOW()
);

-- 健康检测 SQL
INSERT INTO mysql.health_check (id, t_modified)
VALUES (@@server_id, NOW())
ON DUPLICATE KEY UPDATE t_modified = NOW();
```

##### 为什么用 @@server_id？

```
主库  server_id = 1  → 插入 id=1
备库  server_id = 2  → 插入 id=2

如果两个库都用固定 id=1
→ 备库同步主库检测语句时会发生冲突
→ @@server_id 保证主备库各自独立，互不干扰
```

##### 这条语句能检测出什么问题？

| 故障类型 | select 1 | 查表检测 | 更新检测 |
|---|---|---|---|
| 进程挂了 | ✅ 能检测 | ✅ | ✅ |
| 并发线程打满 | ❌ 检测不到 | ✅ 能检测 | ✅ |
| binlog 磁盘满 | ❌ | ❌ | ✅ 能检测 |
| 数据写入异常 | ❌ | ❌ | ✅ 能检测 |

---

#### 四、磁盘故障的判断

##### binlog 磁盘满的特征

```
现象：
  ✅ SELECT 查询正常返回
  ❌ UPDATE / INSERT / DELETE 全部阻塞
  ❌ 事务 COMMIT 被阻塞

原因：
  写 binlog 是事务提交的必要步骤
  磁盘满 → binlog 写不进去 → 事务无法提交
```

##### 判断命令

```bash
# 查看磁盘使用率
df -h

# 查看 binlog 所在目录大小
du -sh /var/lib/mysql/

# 查看 binlog 文件列表
SHOW BINARY LOGS;
```

##### IO 利用率 100% 不等于数据库"慢"

```
IO 利用率 = 100%
    ↓ 代表的含义
IO 资源满负荷，每个请求仍有机会获取 IO 执行

正确判断逻辑：
  IO 100% + 健康检测 update 超时 → 数据库确实有问题
  IO 100% + 健康检测 update 正常返回 → 数据库仍在正常服务
```

---

#### 五、performance_schema 精细化 IO 监控（MySQL 5.6+）

##### 开启监控

```sql
-- 开启 file 级别 IO 统计
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/io/file/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE '%events_waits%';
```

##### 核心监控表

```sql
SELECT * FROM performance_schema.file_summary_by_event_name
WHERE event_name IN (
    'wait/io/file/innodb/innodb_log_file',  -- redo log
    'wait/io/file/sql/binlog'               -- binlog
);
```

##### 统计维度详解

| 字段 | 含义 | 用途 |
|---|---|---|
| `COUNT_READ` | 读请求次数 | 判断读 IO 压力 |
| `SUM_TIMER_READ` | 读请求总耗时（皮秒） | 判断读是否慢 |
| `COUNT_WRITE` | 写请求次数 | 判断写 IO 压力 |
| `SUM_TIMER_WRITE` | 写请求总耗时 | 判断写是否慢 |
| `SUM_TIMER_WAIT` | 总 IO 等待时间 | 综合 IO 健康度 |
| `MAX_TIMER_WAIT` | 单次最大 IO 等待 | 发现毛刺和抖动 |

##### 实战：判断 redo log 是否有 IO 瓶颈

```sql
SELECT
    event_name,
    COUNT_WRITE,
    SUM_TIMER_WRITE / 1000000000000 AS write_sec,   -- 换算成秒
    MAX_TIMER_WRITE / 1000000000000 AS max_write_sec
FROM performance_schema.file_summary_by_event_name
WHERE event_name = 'wait/io/file/innodb/innodb_log_file';
```

##### 异常基线检测（自动化监控思路）

```sql
-- 记录上一次的统计值，与当前值做 diff
-- 若两次采集间隔内 SUM_TIMER_WAIT 增长过快 → IO 有问题

SELECT
    event_name,
    SUM_TIMER_WAIT - @last_wait AS delta_wait  -- 与上次对比
FROM performance_schema.file_summary_by_event_name
WHERE event_name LIKE 'wait/io/file/%';

-- 采集完后清零，方便下次对比
TRUNCATE TABLE performance_schema.file_summary_by_event_name;
```

---

#### 六、综合判断流程

```
数据库出现问题？
        ↓
Step1: select 1
  ❌ 超时 → 进程挂了/网络故障 → 重启或检查网络
  ✅ 正常 → 继续往下

Step2: select * from mysql.health_check
  ❌ 超时 → 并发线程打满
           → 检查 show processlist / 是否有大查询 / 调整 innodb_thread_concurrency
  ✅ 正常 → 继续往下

Step3: insert ... health_check (更新检测)
  ❌ 超时 → 写入异常
           → 检查 binlog 磁盘是否满 / redo log 是否异常
  ✅ 正常 → 继续往下

Step4: performance_schema IO 监控
  SUM_TIMER_WAIT 异常高 → 具体哪个文件 IO 慢 → 定向排查
  MAX_TIMER_WAIT 有毛刺 → 偶发性 IO 抖动 → 检查存储设备
```

---

#### 七、监控告警核心指标汇总

| 监控项 | 告警阈值参考 | 说明 |
|---|---|---|
| binlog 磁盘使用率 | > 80% | 满了会阻塞所有写操作 |
| 并发线程数 | > innodb_thread_concurrency × 80% | 接近上限需预警 |
| 健康检测 update 响应时间 | > 1s | 超时说明写入有问题 |
| redo log IO 等待 | MAX_TIMER_WAIT > 200ms | 存在 IO 瓶颈 |
| 慢查询数量 | 突增 | 有锁或全表扫描 |
| show processlist 中 waiting 数 | 大量 waiting for lock | 锁竞争严重 |

### clickhouse的分布式是怎么工作的

### 如何优化mysql, mysql慢查询如何优化，有哪些手段，
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
  - 如果是web页面展示，可以用前1000条为真，并保证性能。
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
- select count(*) from stu offset 10 limit 10, 如果表中有一百条数据，能查出多少条数据。
```mysql
这个查询语句无法按预期工作，因为"count(*)"函数返回表中的总行数，将"offset"和"limit"应用于"count"函数是没有意义的。
```

# 如何让> <走索引
- 让优化器自己分析认为会走索引，可以考虑range函数界限
- 加上等于号
# 分库分表的界限是什么，如何做

在实际数据库操作中，并没有严格意义上规定数据量达到多少就必须进行分表。因为是否分表需要综合考量多方面因素，以下为你详细分析不同场景下的大致参考数据量以及相关因素：

### 参考数据量
- **百万级别**：如果数据库表的数据量达到百万级别，并且查询性能出现明显下降，尤其是在进行复杂查询（如多表关联、排序、聚合等操作）时，查询响应时间显著变长，那么可以考虑分表。例如，一个包含百万条记录的电商订单表，在统计某段时间内的销售数据时，查询可能会变得很慢。
- **千万级别**：当数据量达到千万级别时，分表往往是必要的。因为此时单表的索引维护成本会变得很高，磁盘I/O压力增大，查询性能会急剧下降。即使是简单的查询，也可能需要较长的时间才能返回结果。

### 需考虑的其他因素
- **业务类型**：不同的业务对数据库性能的要求不同。对于实时性要求较高的业务，如金融交易系统、电商秒杀系统等，数据量可能在几十万甚至更低时就需要考虑分表，以保证系统的响应速度。而对于一些对实时性要求不高的业务，如日志记录系统，可以容忍较大的数据量，分表的时机可以适当推迟。
- **硬件资源**：如果数据库服务器的硬件配置较高，如拥有大量的内存、高速的磁盘阵列等，那么在一定程度上可以缓解数据量增长带来的性能问题。相反，如果硬件资源有限，数据量增长到一定程度就会很快出现性能瓶颈，此时可能需要更早地进行分表。
- **查询模式**：如果表的查询主要集中在少量数据上，并且通过合理的索引可以满足查询需求，那么即使数据量较大，也不一定需要分表。但如果查询经常涉及到全量数据或者大量数据的扫描，那么分表可以有效提高查询性能。

总之，是否进行分表不能仅仅依据数据量的大小来判断，需要综合考虑业务需求、硬件资源、查询模式等多方面因素，通过性能测试和评估来确定最佳的分表时机。 

分库分表分为垂直和水平两种方式，下面详细介绍这两种方式的具体操作：

### 垂直分库分表
#### 垂直分表
- **原理**：把一个表按字段分成多个表，每个表存储一部分字段，常用于表字段较多的情况，这样可以减少IO压力，提高查询性能。
- **操作步骤**：
    1. **分析字段相关性**：确定哪些字段经常一起使用，将其划分为一组。例如，在一个用户表中，用户的基本信息（如姓名、性别、出生日期）和用户的详细信息（如地址、联系方式）可以分开存储。
    2. **创建新表**：根据划分好的字段组创建新的表。
    3. **数据迁移**：将原表中的数据按照字段组迁移到新表中。
    4. **修改应用代码**：修改应用程序中对原表的操作，使其指向新的表。

#### 垂直分库
- **原理**：按照业务功能将不同的表拆分到不同的数据库中，每个数据库专注于特定的业务，以提高系统的可维护性和扩展性。
- **操作步骤**：
    1. **业务模块划分**：根据业务功能将表划分为不同的组，例如，将用户管理、订单管理、商品管理等业务的表分别划分到不同的数据库中。
    2. **创建新数据库**：为每个业务模块创建独立的数据库。
    3. **数据迁移**：将原数据库中的表按照业务模块迁移到新的数据库中。
    4. **修改应用代码**：修改应用程序中对数据库的连接配置，使其指向新的数据库。

### 水平分库分表
#### 水平分表
- **原理**：在同一个数据库内，将表的数据按照一定的规则划分到多个表中，每个表的结构相同，但存储的数据不同，以此降低单表的数据量，提高查询性能。
- **操作步骤**：
    1. **选择分片规则**：常见的分片规则有哈希取模、范围分片、时间分片等。例如，使用哈希取模规则，根据用户ID对表的数量取模，将用户数据分散到不同的表中。
    2. **创建新表**：根据分片规则创建多个结构相同的表。
    3. **数据迁移**：将原表中的数据按照分片规则迁移到新的表中。
    4. **修改应用代码**：修改应用程序中对原表的操作，根据分片规则定位到相应的表。

#### 水平分库
- **原理**：把表的数据按照一定规则划分到多个数据库中，每个数据库存储一部分数据，从而提升系统的并发处理能力和数据存储容量。
- **操作步骤**：
    1. **选择分片规则**：与水平分表类似，选择合适的分片规则，如哈希取模、范围分片等。
    2. **创建新数据库和表**：在多个数据库中创建结构相同的表。
    3. **数据迁移**：将原数据库中的数据按照分片规则迁移到新的数据库和表中。
    4. **修改应用代码**：修改应用程序中对数据库的连接和操作，根据分片规则定位到相应的数据库和表。

### 借助中间件实现分库分表
如果手动实现分库分表的过程较为复杂，你还可以借助中间件来简化操作，例如 MyCAT 和 ShardingSphere。以下是使用它们的基本步骤：
1. **安装和配置中间件**：按照官方文档的指导，安装和配置 MyCAT 或 ShardingSphere。
2. **定义分片规则**：在中间件的配置文件中定义分库分表的规则，如分片键、分片算法等。
3. **修改应用代码**：修改应用程序中对数据库的连接，使其连接到中间件，而不是直接连接到数据库。
4. **测试和优化**：对分库分表后的系统进行测试，根据测试结果进行优化。

以下是一个简单的Python示例，展示如何使用哈希取模规则进行水平分表：
```python
# 假设我们有一个用户表，需要根据用户ID进行水平分表
def get_table_name(user_id, table_count):
    """
    根据用户ID和表的数量，使用哈希取模规则计算表名
    :param user_id: 用户ID
    :param table_count: 表的数量
    :return: 表名
    """
    mod = user_id % table_count
    return f"user_table_{mod}"

# 示例调用
user_id = 123
table_count = 4
table_name = get_table_name(user_id, table_count)
print(f"用户ID {user_id} 对应的表名为: {table_name}")
```
这个示例定义了一个函数 `get_table_name`，它接收用户ID和表的数量作为参数，通过哈希取模规则计算出用户数据应该存储的表名。 