<p align="center">
  <a href="https://github.com/uncleyeung">
   <img alt="Uncle-Yeung-Logo" src="https://raw.githubusercontent.com/uncleyeung/uncleyeung.github.io/master/web/img/logo1.jpg">
  </a>
</p>

<p align="center">
  为简化开发工作、提高生产率而生
</p>

<p align="center">
  
  <a href="https://github.com/996icu/996.ICU/blob/master/LICENSE">
    <img alt="996icu" src="https://img.shields.io/badge/license-NPL%20(The%20996%20Prohibited%20License)-blue.svg">
  </a>

  <a href="https://www.apache.org/licenses/LICENSE-2.0">
    <img alt="code style" src="https://img.shields.io/badge/license-Apache%202-4EB1BA.svg?style=flat-square">
  </a>
</p>

# Mysql order by 长驱直入

### 前情提要「EXPLAIN」常规基础
##### 「EXPLAIN」结果如下：
 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | index | index_test_name | index_test_name | 768 | const | 3 | 100.00 | Using index 

-----
##### 1.type
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
##### 2.possible_keys
+ 可能用到的索引
##### 3.key
+ 实际用到的索引
##### 4.key_line
+ 索引字段最大可能使用长度
##### 5.ref
+ 指出对 key 列所选择的索引的查找方式，常见的值有 const, func, NULL, 具体字段名。当 key 列为 NULL ，即不使用索引时，此值也相应的为 NULL 。
##### 6.rows
+ 估计需要扫描的行数
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

### 准备一些基础数据
```sql
/*
 Navicat Premium Data Transfer

 Source Server         : xx
 Source Server Type    : MySQL
 Source Server Version : 50722
 Source Host           : xx:3306
 Source Schema         : test

 Target Server Type    : MySQL
 Target Server Version : 50722
 File Encoding         : 65001

 Date: 04/09/2019 15:48:50
*/

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for test
-- ----------------------------
DROP TABLE IF EXISTS `test`;
CREATE TABLE `test`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `login_time` datetime(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0),
  `age` int(11) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `index_test_name`(`name`) USING BTREE,
  INDEX `index_test_name_id`(`name`, `id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 15 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of test
-- ----------------------------
INSERT INTO `test` VALUES (1, 'aaa', '2019-09-04 12:45:49', 12);
INSERT INTO `test` VALUES (2, 'bbbb', '2019-09-04 12:45:50', 13);
INSERT INTO `test` VALUES (4, 'dddd', '2019-09-04 12:45:52', 14);
INSERT INTO `test` VALUES (5, 'eeee', '2019-09-04 12:45:54', 13);
INSERT INTO `test` VALUES (6, 'fff', '2019-09-04 12:45:56', 12);
INSERT INTO `test` VALUES (7, 'gggg', '2019-09-04 12:45:58', 11);
INSERT INTO `test` VALUES (8, 'gggg', '2019-09-04 12:46:00', 15);
INSERT INTO `test` VALUES (10, 'gggg', '2019-09-04 12:46:02', 20);
INSERT INTO `test` VALUES (11, 'aaa', '2019-09-04 12:46:04', 21);
INSERT INTO `test` VALUES (12, 'abcdfr', '2019-09-04 12:46:06', 19);
INSERT INTO `test` VALUES (13, 'abcdfr', '2019-09-04 12:46:11', 18);
INSERT INTO `test` VALUES (14, 'dsdafd', '2019-09-04 12:46:13', 15);
INSERT INTO `test` VALUES (18, 'test', '2019-09-04 12:46:17', 16);

SET FOREIGN_KEY_CHECKS = 1;
```
----
### 普通索引实验
```sql
--索引实验-查询条件[*]和[具体参数]的区别
--1
EXPLAIN SELECT * FROM test WHERE name ="gggg";
--2
EXPLAIN SELECT name FROM test WHERE name ="gggg";
--3
EXPLAIN SELECT age FROM test WHERE name ="gggg";
```
##### EXPLAIN 结果
+  1 命中了索引 index_test_name，Extra为null

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | ref | index_test_name,index_test_name_id | index_test_name | 768 | const | 3 | 100.00 | null
+  2 命中了索引 index_test_name，Extra为Using index，使用了覆盖索引，直接可以从索引中取出数据不需要访问表

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | ref | index_test_name,index_test_name_id | index_test_name | 768 | const | 3 | 100.00 | Using index

+  3 命中了索引 index_test_name，Extra为null

  id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
  :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
 1 | SIMPLE | test | null | ref | index_test_name,index_test_name_id | index_test_name | 768 | const | 3 | 100.00 | null

##### 综上所述
+ 查询最好避免使用[*]作为查询列
+ 当查询命中索引列，查询列也为索引列，查询效率最高，不需要访问表，推荐使用
+ 当命中索引时type=ref，非唯一索引访问(只有普通索引)
 ----
### 排序基础版实验 
```sql
---排序实验：无查询条件 查询列* 命中索引&非命中索引
EXPLAIN SELECT * FROM test ORDER BY id desc;
EXPLAIN SELECT * FROM test ORDER BY age desc;
```

##### EXPLAIN 结果
+  1 命中了索引 PRIMARY，Extra为null

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | index | null | PRIMARY | 4 | null | 13 | 100.00 | null
+  2 未命中了索引全表扫描，Extra为Using filesort，

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | ALL | null | null | null | null | 13 | 100.00 | Using filesort

##### 综上所述
+ 论证type为index和ref的区别：
+ index：按索引次序扫描，先读索引，再读实际的行，结果还是全表扫描，主要优点是避免了排序。因为索引是排好的。
+ ref：当命中索引时type=ref，非唯一索引访问(只有普通索引)
+ 从sql1取出全表数据{全表扫描}，但是排序却使用了主键id，由于id已经是排好顺序，所以先读了主键索引，但还是拿了全表的数据，只是没有进行排序。
+ 从sql2中可以看出type：all Extra为Using filesort 说明全表扫描了数据，并进行了针对age的二次排序所以产生了Using filesort
 ----
### 排序进阶版实验 
```sql
---排序实验：无查询条件 查询列*/命中索引列/非命中索引列
EXPLAIN SELECT * FROM test ORDER BY name desc;
EXPLAIN SELECT age FROM test ORDER BY name desc;
EXPLAIN SELECT id FROM test ORDER BY name desc;
EXPLAIN SELECT name FROM test ORDER BY name desc;

```

##### EXPLAIN 结果
+  1 全表扫描type：all，Extra为Using filesort

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | ALL | null | null | null | null | 13 | 100.00 | Using filesort
+  2 全表扫描type：all，Extra为Using filesort

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | ALL | null | null | null | null | 13 | 100.00 | Using filesort
+  3 全表扫描type：index，Extra为Using index

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | index | null | index_test_name | 768 | null | 13 | 100.00 | Using index
+  4 全表扫描type：index，Extra为Using index

 id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra 
 :------| :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ | :------ 
1 | SIMPLE | test | null | index | null | index_test_name | 768 | null | 13 | 100.00 | Using index

##### 综上所述
+ sql1&sql2由于查询列使用[*] 根据第一次论证已解释（回表就是索引上获取不到列的数据 需要从表里面把数据查出来）所以造成了全表扫描，后又根据name排序造成了二次排序，产生了Extra为Using index
+ sql3&sql4查询列中只存在索引列，所以使用了覆盖索引Extra为Using index，
+ 至于本小段的sql1和上一小段sql1都是索引，只不过一个是主键索引一个是普通索引，结果确是order by id：tyep为index & order by name：tyep为all [请点击这里见解答](https://segmentfault.com/q/1010000004197413)
 ----
 **排序搞鸡版实验 待续。。。**