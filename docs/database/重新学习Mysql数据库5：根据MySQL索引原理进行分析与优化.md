本文转自互联网

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《重新学习MySQL数据库》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从sql基本的使用方法，从MySQL执行引擎再到索引、事务等知识，一步步地学习MySQL相关技术的实现原理，更好地了解如何基于这些知识来优化sql，减少SQL执行时间，通过执行计划对SQL性能进行分析，再到MySQL的主从复制、主备部署等内容，以便让你更完整地了解整个MySQL方面的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 一：Mysql原理与慢查询

MySQL凭借着出色的性能、低廉的成本、丰富的资源，已经成为绝大多数互联网公司的首选关系型数据库。虽然性能出色，但所谓“好马配好鞍”，如何能够更好的使用它，已经成为开发工程师的必修课，我们经常会从职位描述上看到诸如“精通MySQL”、“SQL语句优化”、“了解数据库原理”等要求。我们知道一般的应用系统，读写比例在10:1左右，而且插入操作和一般的更新操作很少出现性能问题，遇到最多的，也是最容易出问题的，还是一些复杂的查询操作，所以查询语句的优化显然是重中之重。

本人从13年7月份起，一直在美团核心业务系统部做慢查询的优化工作，共计十余个系统，累计解决和积累了上百个慢查询案例。随着业务的复杂性提升，遇到的问题千奇百怪，五花八门，匪夷所思。本文旨在以开发工程师的角度来解释数据库索引的原理和如何优化慢查询。

## 一个慢查询引发的思考

```
select   count(*) from   task where   status=2    and operator_id=20839    and operate_time>1371169729    and operate_time<1371174603    and type=2;
```

系统使用者反应有一个功能越来越慢，于是工程师找到了上面的SQL。
并且兴致冲冲的找到了我，“这个SQL需要优化，给我把每个字段都加上索引”
我很惊讶，问道“为什么需要每个字段都加上索引？”
“把查询的字段都加上索引会更快”工程师信心满满
“这种情况完全可以建一个联合索引，因为是最左前缀匹配，所以operate_time需要放到最后，而且还需要把其他相关的查询都拿来，需要做一个综合评估。”
“联合索引？最左前缀匹配？综合评估？”工程师不禁陷入了沉思。
多数情况下，我们知道索引能够提高查询效率，但应该如何建立索引？索引的顺序如何？许多人却只知道大概。其实理解这些概念并不难，而且索引的原理远没有想象的那么复杂。



## 二：索引建立

1\. 主键索引

primary key() 要求关键字不能重复，也不能为null,同时增加主键约束 
主键索引定义时，不能命名

2\. 唯一索引

unique index() 要求关键字不能重复，同时增加唯一约束

3\. 普通索引

index() 对关键字没有要求

4\. 全文索引

fulltext key() 关键字的来源不是所有字段的数据，而是字段中提取的特别关键字

**关键字：可以是某个字段或多个字段，多个字段称为复合索引**

```
建表：creat table student(    stu_id int unsigned not null auto_increment,    name varchar(32) not null default '',    phone char(11) not null default '',    stu_code varchar(32) not null default '',    stu_desc text,    primary key ('stu_id'),     //主键索引    unique index 'stu_code' ('stu_code'), //唯一索引    index 'name_phone' ('name','phone'),  //普通索引，复合索引    fulltext index 'stu_desc' ('stu_desc'), //全文索引) engine=myisam charset=utf8; 更新：alert table student    add primary key ('stu_id'),     //主键索引    add unique index 'stu_code' ('stu_code'), //唯一索引    add index 'name_phone' ('name','phone'),  //普通索引，复合索引    add fulltext index 'stu_desc' ('stu_desc'); //全文索引 删除：alert table sutdent    drop primary key,    drop index 'stu_code',    drop index 'name_phone',    drop index 'stu_desc';
```

## 三：浅析explain用法

### 有什么用？

在MySQL中，当数据量增长的特别大的时候就需要用到索引来优化SQL语句，而如何才能判断我们辛辛苦苦写出的SQL语句是否优良？这时候**explain**就派上了用场。

### 怎么使用？

```
explain + SQL语句即可 如：explain select * from table;

```

如下

![explain参数](https://user-gold-cdn.xitu.io/2018/5/17/1636ce849c800023?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

相信第一次使用explain参数的朋友一定会疑惑这一大堆参数究竟有什么用呢？笔者搜集了一些资料，在这儿做一个总结希望能够帮助大家理解。

* * *

## 参数介绍

### id

```
如果是子查询，id的序号会递增，id的值越大优先级越高，越先被执行

```

### select_type

查询的类型，主要用于区别普通查询、联合查询、子查询等的复杂查询 SIMPLE:简单的select查询，查询中不包含子查询或者UNION PRIMARY:查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY（最后加载的那一个 ） SUBQUERY:在SELECT或WHERE列表中包含了子查询 DERIVED:在FROM列表中包含的子查询被标记为DERIVED（衍生）Mysql会递归执行这些子查询，把结果放在临时表里。 UNION:若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM字句的查询中，外层SELECT将被标记为:DERIVED UNION RESULT:从UNION表获取结果的SELECT type

显示查询使用了何种类型

从最好到最差依次是System>const>eq_ref>range>index>All（**全表扫描**）	一般来说**至少达到range级别，最好达到ref**

System:表只有一行记录，这是const类型的特例，平时不会出现(忽略不计)const:表示通过索引一次就找到了,const用于比较primary key或者unique索引，因为只匹配一行数据，所以很快。如将主键置于where列表中，MySQL就能将该查询转换为一个常量。

eq_ref:唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描。

ref：非唯一索引扫描，返回匹配某个单独值的行，本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而它可能会找到多个符合条件的行，所以它应该属于查找和扫描的混合体range：只检索给定范围的行，使用一个索引来选择行。

key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询。这种范围扫描索引比全表扫描要好，因为它只需要开始于索引的某一点，而结束于另一点，不用扫描全部索引。index:FULL INDEX SCAN,index与all区别为index类型只遍历索引树。这通常比all快，因为索引文件通常比数据文件小。

### extra

包含不适合在其他列中显示但十分重要的额外信息 包含的信息： **（危险!）**Using

filesort:说明mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，MYSQL中无法利用索引完成的排序操作称为“文件排序” **（特别危险!）**Using

temporary:使用了临时表保存中间结果，MYSQL在对查询结果排序时使用临时表。常见于排序order by 和分组查询 group by Using

index:表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错。如果同时出现using

where，表明索引被用来执行索引键值的查找；如果没有同时出现using where，表明索引用来读取数据而非执行查找操作。

### possible_keys

显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出， 但不一定被查询实际使用

### key

实际使用的索引，如果为NULL，则没有使用索引。查询中若使用了覆盖索引，则该索引仅出现在key列表中，key参数可以作为使用了索引的判断标准

### key_len

:表示索引中使用的字节数，可通过该列计算查询中索引的长度，在不损失精确性的情况下，长度越短越好，key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。

### ref

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引上的值。

### rows

根据表统计信息及索引选用情况，大致估算出找到所需记录所需要读取的行数

## 四：慢查询优化

关于MySQL索引原理是比较枯燥的东西，大家只需要有一个感性的认识，并不需要理解得非常透彻和深入。我们回头来看看一开始我们说的慢查询，了解完索引原理之后，大家是不是有什么想法呢？先总结一下索引的几大基本原则

### 建索引的几大原则

1.最左前缀匹配原则，非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。

2.=和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式

3.尽量选择区分度高的列作为索引,区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录

4.索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);
5.尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可

### 回到开始的慢查询

根据最左匹配原则，最开始的sql语句的索引应该是status、operator_id、type、operate_time的联合索引；其中status、operator_id、type的顺序可以颠倒，所以我才会说，把这个表的所有相关查询都找到，会综合分析；

比如还有如下查询

```
select * from task where status = 0 and type = 12 limit 10;

```

```
select count(*) from task where status = 0 ;

```

那么索引建立成(status,type,operator_id,operate_time)就是非常正确的，因为可以覆盖到所有情况。这个就是利用了索引的最左匹配的原则

### 查询优化神器 - explain命令

关于explain命令相信大家并不陌生，具体用法和字段含义可以参考官网[explain-output](http://dev.mysql.com/doc/refman/5.5/en/explain-output.html)，这里需要强调rows是核心指标，绝大部分rows小的语句执行一定很快（有例外，下面会讲到）。所以优化语句基本上都是在优化rows。

### 慢查询优化基本步骤

0.先运行看看是否真的很慢，注意设置SQL_NO_CACHE
1.where条件单表查，锁定最小返回记录表。这句话的意思是把查询语句的where都应用到表中返回的记录数最小的表开始查起，单表每个字段分别查询，看哪个字段的区分度最高
2.explain查看执行计划，是否与1预期一致（从锁定记录较少的表开始查询）
3.order by limit 形式的sql语句让排序的表优先查
4.了解业务方使用场景
5.加索引时参照建索引的几大原则

6.观察结果，不符合预期继续从0分析

## 五：最左前缀原理与相关优化

高效使用索引的首要条件是知道什么样的查询会使用到索引，这个问题和B+Tree中的“最左前缀原理”有关，下面通过例子说明最左前缀原理。

这里先说一下联合索引的概念。在上文中，我们都是假设索引只引用了单个的列，实际上，MySQL中的索引可以以一定顺序引用多个列，这种索引叫做联合索引，一般的，一个联合索引是一个有序元组，其中各个元素均为数据表的一列，实际上要严格定义索引需要用到关系代数，但是这里我不想讨论太多关系代数的话题，因为那样会显得很枯燥，所以这里就不再做严格定义。另外，单列索引可以看成联合索引元素数为1的特例。

以employees.titles表为例，下面先查看其上都有哪些索引：

1.  SHOW INDEX FROM employees.titles;
2.  +--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+
3.  | Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Null | Index_type |
4.  +--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+
5.  | titles | 0 | PRIMARY | 1 | emp_no | A | NULL | | BTREE |
6.  | titles | 0 | PRIMARY | 2 | title | A | NULL | | BTREE |
7.  | titles | 0 | PRIMARY | 3 | from_date | A | 443308 | | BTREE |
8.  | titles | 1 | emp_no | 1 | emp_no | A | 443308 | | BTREE |
9.  +--------+------------+----------+--------------+-------------+-----------+-------------+------+------------+

从结果中可以到titles表的主索引为<emp_no, title, from_date>，还有一个辅助索引<emp_no>。为了避免多个索引使事情变复杂（MySQL的SQL优化器在多索引时行为比较复杂），这里我们将辅助索引drop掉：

1.  ALTER TABLE employees.titles DROP INDEX emp_no;

这样就可以专心分析索引PRIMARY的行为了。

### 情况一：全列匹配。

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND title='Senior Engineer' AND from_date='1986-06-26';
2.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
5.  | 1 | SIMPLE | titles | const | PRIMARY | PRIMARY | 59 | const,const,const | 1 | |
6.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+

很明显，当按照索引中所有列进行精确匹配（这里精确匹配指“=”或“IN”匹配）时，索引可以被用到。这里有一点需要注意，理论上索引对顺序是敏感的，但是由于MySQL的查询优化器会自动调整where子句的条件顺序以使用适合的索引，例如我们将where中的条件顺序颠倒：

1.  EXPLAIN SELECT * FROM employees.titles WHERE from_date='1986-06-26' AND emp_no='10001' AND title='Senior Engineer';
2.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+
5.  | 1 | SIMPLE | titles | const | PRIMARY | PRIMARY | 59 | const,const,const | 1 | |
6.  +----+-------------+--------+-------+---------------+---------+---------+-------------------+------+-------+

效果是一样的。

### 情况二：最左前缀匹配。

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001';
2.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------+
5.  | 1 | SIMPLE | titles | ref | PRIMARY | PRIMARY | 4 | const | 1 | |
6.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------+

当查询条件精确匹配索引的左边连续一个或几个列时，如<emp_no>或<emp_no, title>，所以可以被用到，但是只能用到一部分，即条件所组成的最左前缀。上面的查询从分析结果看用到了PRIMARY索引，但是key_len为4，说明只用到了索引的第一列前缀。

### 情况三：查询条件用到了索引中列的精确匹配，但是中间某个条件未提供。

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND from_date='1986-06-26';
2.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
5.  | 1 | SIMPLE | titles | ref | PRIMARY | PRIMARY | 4 | const | 1 | Using where |
6.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+

此时索引使用情况和情况二相同，因为title未提供，所以查询只用到了索引的第一列，而后面的from_date虽然也在索引中，但是由于title不存在而无法和左前缀连接，因此需要对结果进行扫描过滤from_date（这里由于emp_no唯一，所以不存在扫描）。

如果想让from_date也使用索引而不是where过滤，可以增加一个辅助索引<emp_no, from_date>，此时上面的查询会使用这个索引。除此之外，还可以使用一种称之为“隔离列”的优化方法，将emp_no与from_date之间的“坑”填上。

首先我们看下title一共有几种不同的值：

1.  SELECT DISTINCT(title) FROM employees.titles;
2.  +--------------------+
3.  | title |
4.  +--------------------+
5.  | Senior Engineer |
6.  | Staff |
7.  | Engineer |
8.  | Senior Staff |
9.  | Assistant Engineer |
10.  | Technique Leader |
11.  | Manager |
12.  +--------------------+

只有7种。在这种成为“坑”的列值比较少的情况下，可以考虑用“IN”来填补这个“坑”从而形成最左前缀：

1.  EXPLAIN SELECT * FROM employees.titles
2.  WHERE emp_no='10001'
3.  AND title IN ('Senior Engineer', 'Staff', 'Engineer', 'Senior Staff', 'Assistant Engineer', 'Technique Leader', 'Manager')
4.  AND from_date='1986-06-26';
5.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
6.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
7.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
8.  | 1 | SIMPLE | titles | range | PRIMARY | PRIMARY | 59 | NULL | 7 | Using where |
9.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+

这次key_len为59，说明索引被用全了，但是从type和rows看出IN实际上执行了一个range查询，这里检查了7个key。看下两种查询的性能比较：

1.  SHOW PROFILES;
2.  +----------+------------+-------------------------------------------------------------------------------+
3.  | Query_ID | Duration | Query |
4.  +----------+------------+-------------------------------------------------------------------------------+
5.  | 10 | 0.00058000 | SELECT * FROM employees.titles WHERE emp_no='10001' AND from_date='1986-06-26'|
6.  | 11 | 0.00052500 | SELECT * FROM employees.titles WHERE emp_no='10001' AND title IN ... |
7.  +----------+------------+-------------------------------------------------------------------------------+

“填坑”后性能提升了一点。如果经过emp_no筛选后余下很多数据，则后者性能优势会更加明显。当然，如果title的值很多，用填坑就不合适了，必须建立辅助索引。

### 情况四：查询条件没有指定索引第一列。

1.  EXPLAIN SELECT * FROM employees.titles WHERE from_date='1986-06-26';
2.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
5.  | 1 | SIMPLE | titles | ALL | NULL | NULL | NULL | NULL | 443308 | Using where |
6.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+

由于不是最左前缀，索引这样的查询显然用不到索引。

### 情况五：匹配某列的前缀字符串。

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND title LIKE 'Senior%';
2.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
5.  | 1 | SIMPLE | titles | range | PRIMARY | PRIMARY | 56 | NULL | 1 | Using where |
6.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+

此时可以用到索引，但是如果通配符不是只出现在末尾，则无法使用索引。（原文表述有误，如果通配符%不出现在开头，则可以用到索引，但根据具体情况不同可能只会用其中一个前缀）

### 情况六：范围查询。

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no < '10010' and title='Senior Engineer';
2.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
5.  | 1 | SIMPLE | titles | range | PRIMARY | PRIMARY | 4 | NULL | 16 | Using where |
6.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+

范围列可以用到索引（必须是最左前缀），但是范围列后面的列无法用到索引。同时，索引最多用于一个范围列，因此如果查询条件中有两个范围列则无法全用到索引。

1.  EXPLAIN SELECT * FROM employees.titles
2.  WHERE emp_no < '10010'
3.  AND title='Senior Engineer'
4.  AND from_date BETWEEN '1986-01-01' AND '1986-12-31';
5.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
6.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
7.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
8.  | 1 | SIMPLE | titles | range | PRIMARY | PRIMARY | 4 | NULL | 16 | Using where |
9.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+

可以看到索引对第二个范围索引无能为力。这里特别要说明MySQL一个有意思的地方，那就是仅用explain可能无法区分范围索引和多值匹配，因为在type中这两者都显示为range。同时，用了“between”并不意味着就是范围查询，例如下面的查询：

1.  EXPLAIN SELECT * FROM employees.titles
2.  WHERE emp_no BETWEEN '10001' AND '10010'
3.  AND title='Senior Engineer'
4.  AND from_date BETWEEN '1986-01-01' AND '1986-12-31';
5.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
6.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
7.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
8.  | 1 | SIMPLE | titles | range | PRIMARY | PRIMARY | 59 | NULL | 16 | Using where |
9.  +----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+

看起来是用了两个范围查询，但作用于emp_no上的“BETWEEN”实际上相当于“IN”，也就是说emp_no实际是多值精确匹配。可以看到这个查询用到了索引全部三个列。因此在MySQL中要谨慎地区分多值匹配和范围匹配，否则会对MySQL的行为产生困惑。

### 情况七：查询条件中含有函数或表达式。

很不幸，如果查询条件中含有函数或表达式，则MySQL不会为这列使用索引（虽然某些在数学意义上可以使用）。例如：

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND left(title, 6)='Senior';
2.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+
5.  | 1 | SIMPLE | titles | ref | PRIMARY | PRIMARY | 4 | const | 1 | Using where |
6.  +----+-------------+--------+------+---------------+---------+---------+-------+------+-------------+

虽然这个查询和情况五中功能相同，但是由于使用了函数left，则无法为title列应用索引，而情况五中用LIKE则可以。再如：

1.  EXPLAIN SELECT * FROM employees.titles WHERE emp_no - 1='10000';
2.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+
5.  | 1 | SIMPLE | titles | ALL | NULL | NULL | NULL | NULL | 443308 | Using where |
6.  +----+-------------+--------+------+---------------+------+---------+------+--------+-------------+

显然这个查询等价于查询emp_no为10001的函数，但是由于查询条件是一个表达式，MySQL无法为其使用索引。看来MySQL还没有智能到自动优化常量表达式的程度，因此在写查询语句时尽量避免表达式出现在查询中，而是先手工私下代数运算，转换为无表达式的查询语句。

## 索引选择性与前缀索引

既然索引可以加快查询速度，那么是不是只要是查询语句需要，就建上索引？答案是否定的。因为索引虽然加快了查询速度，但索引也是有代价的：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。一般两种情况下不建议建索引。

第一种情况是表记录比较少，例如一两千条甚至只有几百条记录的表，没必要建索引，让查询做全表扫描就好了。至于多少条记录才算多，这个个人有个人的看法，我个人的经验是以2000作为分界线，记录数不超过 2000可以考虑不建索引，超过2000条可以酌情考虑索引。

另一种不建议建索引的情况是索引的选择性较低。所谓索引的选择性（Selectivity），是指不重复的索引值（也叫基数，Cardinality）与表记录数（#T）的比值：

Index Selectivity = Cardinality / #T

显然选择性的取值范围为(0, 1]，选择性越高的索引价值越大，这是由B+Tree的性质决定的。例如，上文用到的employees.titles表，如果title字段经常被单独查询，是否需要建索引，我们看一下它的选择性：

1.  SELECT count(DISTINCT(title))/count(*) AS Selectivity FROM employees.titles;
2.  +-------------+
3.  | Selectivity |
4.  +-------------+
5.  | 0.0000 |
6.  +-------------+

title的选择性不足0.0001（精确值为0.00001579），所以实在没有什么必要为其单独建索引。

有一种与索引选择性有关的索引优化策略叫做前缀索引，就是用列的前缀代替整个列作为索引key，当前缀长度合适时，可以做到既使得前缀索引的选择性接近全列索引，同时因为索引key变短而减少了索引文件的大小和维护开销。下面以employees.employees表为例介绍前缀索引的选择和使用。

从图12可以看到employees表只有一个索引<emp_no>，那么如果我们想按名字搜索一个人，就只能全表扫描了：

1.  EXPLAIN SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido';
2.  +----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
3.  | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
4.  +----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
5.  | 1 | SIMPLE | employees | ALL | NULL | NULL | NULL | NULL | 300024 | Using where |
6.  +----+-------------+-----------+------+---------------+------+---------+------+--------+-------------+

如果频繁按名字搜索员工，这样显然效率很低，因此我们可以考虑建索引。有两种选择，建<first_name>或<first_name, last_name>，看下两个索引的选择性：

1.  SELECT count(DISTINCT(first_name))/count(*) AS Selectivity FROM employees.employees;
2.  +-------------+
3.  | Selectivity |
4.  +-------------+
5.  | 0.0042 |
6.  +-------------+
7.  SELECT count(DISTINCT(concat(first_name, last_name)))/count(*) AS Selectivity FROM employees.employees;
8.  +-------------+
9.  | Selectivity |
10.  +-------------+
11.  | 0.9313 |
12.  +-------------+

<first_name>显然选择性太低，<first_name, last_name>选择性很好，但是first_name和last_name加起来长度为30，有没有兼顾长度和选择性的办法？可以考虑用first_name和last_name的前几个字符建立索引，例如<first_name, left(last_name, 3)>，看看其选择性：

1.  SELECT count(DISTINCT(concat(first_name, left(last_name, 3))))/count(*) AS Selectivity FROM employees.employees;
2.  +-------------+
3.  | Selectivity |
4.  +-------------+
5.  | 0.7879 |
6.  +-------------+

选择性还不错，但离0.9313还是有点距离，那么把last_name前缀加到4：

1.  SELECT count(DISTINCT(concat(first_name, left(last_name, 4))))/count(*) AS Selectivity FROM employees.employees;
2.  +-------------+
3.  | Selectivity |
4.  +-------------+
5.  | 0.9007 |
6.  +-------------+

这时选择性已经很理想了，而这个索引的长度只有18，比<first_name, last_name>短了接近一半，我们把这个前缀索引 建上：

1.  ALTER TABLE employees.employees
2.  ADD INDEX `first_name_last_name4` (first_name, last_name(4));

此时再执行一遍按名字查询，比较分析一下与建索引前的结果：

1.  SHOW PROFILES;
2.  +----------+------------+---------------------------------------------------------------------------------+
3.  | Query_ID | Duration | Query |
4.  +----------+------------+---------------------------------------------------------------------------------+
5.  | 87 | 0.11941700 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
6.  | 90 | 0.00092400 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
7.  +----------+------------+---------------------------------------------------------------------------------+

性能的提升是显著的，查询速度提高了120多倍。

前缀索引兼顾索引大小和查询速度，但是其缺点是不能用于ORDER BY和GROUP BY操作，也不能用于Covering index（即当索引本身包含查询所需全部数据时，不再访问数据文件本身

## 六：InnoDB的主键选择与插入优化

在使用InnoDB存储引擎时，如果没有特别的需要，请永远使用一个与业务无关的自增字段作为主键。

经常看到有帖子或博客讨论主键选择问题，有人建议使用业务无关的自增主键，有人觉得没有必要，完全可以使用如学号或身份证号这种唯一字段作为主键。不论支持哪种论点，大多数论据都是业务层面的。如果从数据库索引优化角度看，使用InnoDB引擎而不使用自增主键绝对是一个糟糕的主意。

上文讨论过InnoDB的索引实现，InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B+Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB默认为15/16），则开辟一个新的页（节点）。

如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。如下图所示：

![](http://blog.codinglabs.org/uploads/pictures/theory-of-mysql-index/13.png)

图13

这样就会形成一个紧凑的索引结构，近似顺序填满。由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置：

![](http://blog.codinglabs.org/uploads/pictures/theory-of-mysql-index/14.png)

图14

此时MySQL不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过OPTIMIZE TABLE来重建表并优化填充页面。

因此，只要可以，请尽量在InnoDB上采用自增字段做主键。