# Mysql order by 长驱直入

###前请提要「EXPLAIN」常规基础
#####「EXPLAIN」结果如下：
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ |
| 1 | SIMPLE | test | null | index | index_test_name | index_test_name | 768 | const | 3 | 100.00 | Using index |
-----
##### 1.type ： 类型
###### 包括System,const,eq_ref,ref,range,index,all   
+ all : 即全表扫描
+ index : 按索引次序扫描，先读索引，再读实际的行，结果还是全表扫描，主要优点是避免了排序。因为索引是排好的。
+ range : 以范围的形式扫描。
+ ref : 非唯一索引访问(只有普通索引)
+ eq_ref : 使用唯一索引查找(主键或唯一索引)
+ const : 常量查询，在整个查询过程中这个表最多只会有一条匹配的行，比如主键 id=1 就肯定只有一行，只需读取一次表数据便能取得所需的结果，且表数据在分解执行计划时读取。
当结果不是一条时，就会变成index或range等其他类型
+ system : 系统查询
+ null : 优化过程中就已经得到结果，不在访问表或索引
##### 2.possible_keys ： 可能用到的索引
##### 3.key ： 实际用到的索引
##### 4.key_line ： 索引字段最大可能使用长度
##### 5.ref ： 指出对 key 列所选择的索引的查找方式，常见的值有 const, func, NULL, 具体字段名。当 key 列为 NULL ，即不使用索引时，此值也相应的为 NULL 。
##### 6.rows ： 估计需要扫描的行数
##### 7.Extra ： 显示以上信息之外的其他信息
+ Using index
此查询使用了覆盖索引(Covering Index)，即通过索引就能返回结果，无需访问表。
若没显示"Using index"表示读取了表数据。
+ Using where
表示 MySQL 服务器从存储引擎收到行后再进行“后过滤”（Post-filter）。所谓“后过滤”，就是先读取整行数据，再检查此行是否符合 where 句的条件，符合就留下，不符合便丢弃。因为检查是在读取行后才进行的，所以称为“后过滤”。
+ Using temporary
使用到临时表
建表及插入数据：
create table a(a_id int, b_id int);
insert into a values(1,1),(1,1),(2,1),(2,2),(3,1);
mysql> explain select distinct a_id from a\G
        Extra: Using temporary
MySQL 使用临时表来实现 distinct 操作。

+ Using filesort
若查询所需的排序与使用的索引的排序一致，因为索引是已排序的，因此按索引的顺序读取结果返回，否则，在取得结果后，还需要按查询所需的顺序对结果进行排序，这时就会出现 Using filesort 。
select * from a order by id;
对于没有索引的列进行order by 就会出现filesort
-----