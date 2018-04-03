## SQL优化



#### 1.定位sql 

* 查看各类sql的执行频率

  ```sql
  show [session|global] status like 'Com_%';  # 默认是session
  ```

  此命令显示当前session中所有统计参数的值，我们主要关心下面几个：

  - Com_select：执行select 操作的次数，一次查询只累加1。
  - Com_insert：执行INSERT 操作的次数，对于批量插入的INSERT 操作，只累加一次。
  - Com_update：执行UPDATE 操作的次数。
  - Com_delete：执行DELETE 操作的次数。

* 定位低效率sql语句

  * 开启慢日志记录

    mysql默认没有开启慢日志记录，需要我们手动开启，修改my.cnf文件，增加或修改参数slow_query_log 和slow_query_log_file后，然后重启MySQL服务器。

    ```shell
    slow_query_log =1
    slow_query_log_file=/tmp/mysql_slow.log
    ```

    slow_query_log属性表示大于此值的慢sql会被记录到$slow_query_log_file文件中

    重启后可以执行下面的命令，查看开启是否成功

    ```shell
    show variables like 'slow_query%';
    ```

  * 查看正在执行的sql

    慢日志只能记录已经查询完的sql，因此如果我们需要查看当前执行sql是否存在问题时，就需要**show processlist**帮忙了。此命令可以查看当前MySQL 在进行的线程， 包括线程的状态、是否锁表等，可以实时地查看SQL 的执行情况。具体可参考[MySql使用show processlist查看正在执行的Sql语句](http://www.cnblogs.com/jasondan/p/3491258.html)

#### 2.分析sql

使用**explain**或者**desc**查看sql执行计划，如下图：

![](https://i.loli.net/2018/04/02/5ac1f8a0d5914.png)

* select_type：表示SELECT 的类型，常见的取值有SIMPLE（简单表，即不使用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或者后面的查询语句）、SUBQUERY（子查询中的第一个SELECT）等。
* table：输出结果集的表。
* type：表示表的连接类型，性能由好到差的连接类型为：
  * system（表中仅有一行，即常量表）。
  * const（单表中最多有一个匹配行，例如primary key 或者unique index）。
  * eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接中使用primary key或者      unique index）。
  * ref （与eq_ref类似，区别在于不是使用primary key 或者unique index，而是使用普通的索引）。
  * ref_or_null（与ref 类似，区别在于条件中包含对NULL 的查询）、index_merge(索引合并优化)。
  * unique_subquery（in的后面是一个查询主键字段的子查询）。
  * index_subquery（与unique_subquery 类似，区别在于in 的后面是查询非唯一索引字段的子查询）  。
  * range（单表中的范围查询）。
  * index（对于前面的每一行，都通过查询索引来得到数据）。
  * all（对于前面的每一行，都通过全表扫描来得到数据）。
* possible_keys：表示查询时，可能使用的索引。
* key：表示实际使用的索引。
* key_len：索引字段的长度。
* rows：扫描行的数量。
* Extra：执行情况的说明和描述


我们可以根据结果找出问题出现在哪里，比如说type为all说明是在全表扫描，这时你就需要寻找为什么没有使用索引的原因了。

#### 3.索引注意点

开启索引后，并不意味着在接下来的每次查询中都会使用索引，有以下需要注意的几点：

* 如果条件中有or，即使其中有条件带索引也不会使用(这也是为什么尽量少用or的原因)，注意：要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引。
* 对于多列索引，不是使用的第一部分、like查询是以%开头、查询条件使用函数在索引列上，或者对索引列进行运算，运算包括(+，-，*，/，! 等) 、在 where 子句中使用 != 或 <> 操作符，不会使用索引
* 如果列类型是字符串，那一定要在条件中将数据使用单引号引用起来，否则不使用索引。所以不管是什么类型，都加上单引号。
* 在 where 子句中使用 IN 或 NOT IN，用exists 或 not exists来代替。
* 查询出来的表上的数据行超出表总记录数30%，变成全表扫描。

#### 4.其他注意点

* group by 

  - 默认情况下，MySQL 对所有GROUP BY col1，col2....的字段进行排序。这与在查询中指定ORDER BY col1，col2...类似。因此，如果显式包括一个包含相同的列的ORDER BY 子句，则对MySQL 的实际执行性能没有什么影响。
  - 如果查询包括GROUP BY 但用户想要避免排序结果的消耗，则可以指定ORDER BY NULL禁止排序。禁止后用explain查看的Extra不会显示Using filesort。

* or

  对于含有OR 的查询子句，如果要利用索引，则OR 之间的每个条件列都必须用到索引；如果没有索引，则应该考虑增加索引。

* order by

  以下几种情况下不使用索引

  ```sql
  SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC；
  # order by 的字段混合ASC 和DESC
  SELECT * FROM t1 WHERE key2=constant ORDER BY key1；
  # 用于查询行的关键字与ORDER BY 中所使用的不相同
  ```






参考文档：

* [mysql学习笔记——4.sql优化](https://www.jianshu.com/p/add70a5168bd)
* [MySQL Explain详解](http://www.cnblogs.com/xuanzhi201111/p/4175635.html)
* [MySQL 性能调优之查询优化](https://www.cnblogs.com/markjiao/p/5665775.html)