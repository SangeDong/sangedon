---
title: Mysql中的索引类型
date: 2019-05-21 20:32:24
tags: [读书笔记, 高性能Mysql]
categories: 数据库
---



索引是存储引擎中用于快速找到记录的一种数据结构，因此索引优化是对于查询性能优化最有效的手段，尤其是当表中数据越来越大时。然而，不恰当的索引在数据量逐渐增大的情况下会导致性能的急剧下降，而历史经验告诉我们，**最优**的索引有时比一个**好的**索引性能要好两个数量级，因此，创建一个真正的最优索引对于提升查询性能非常有利。

- 小实例

如果在`user`表的`user_id`字段上创建了索引，Mysql如何执行下面这条语句？

```sql
select username from user where user_id = 10;
```

就像我们查字典一样，Mysql首先在索引中查找值为`10`的索引，然后根据索引找到`user_id`为`10`的所有行并返回

## 索引的类型

索引是在存储引擎层而非服务器层实现的，不同的存储引擎对于索引均有各自的实现，且同一种索引在不同的存储引擎中实现方式也可能不同

### B-Tree索引

一般我们所谈论的索引在没有特别指明时多为B-Tree索引

### Hash索引

基于哈希表实现，只有精确匹配的查询才有效，对于每行数据，存储引擎都会对所有的索引列计算一个哈希码，不同的键值计算的哈希码不一致，而哈希索引将所有的哈希码存储在索引中，同时保存指向每行记录的指针。

如果多行的索引列计算的哈希值相同，索引会以链表的方式存放多个记录的指针到同一个哈希码条目中

- 查询过程

```sql
select * from user where username = 'sange';
```

如以上查询语句，若使用的是哈希索引，且索引列为`username`，则Mysql先根据where子句中的username的值计算哈希值，如 hash('sange'') = 8080，Mysql在索引中查询值为`8080`的索引，则可以查到到指向`username`为`sange`的行，最后比较第三行的值是否和 要查找的值相同

#### 特点

- 只包含哈希值和行指针，不存储字段值，因此不能避免读取行。由于访问内存中的行速度非常快，因此，多数情况下对于性能影响并不明显
- 哈希索引数据并非按照顺序存储，因此无法用于排序
- 由于是以所有的索引列的值来计算哈希值，因此也不支持部分索引匹配查找；如索引列为A，B，查询时只有索引列`A | B`，此时无法使用索引
- 哈希索引仅支持等值比较查询，如 `=`，`IN()`，`<=>`等，不支持范围查询
- 若无哈希冲突时，访问索引非常快，有冲突时，存储引擎将遍历链表上的所有指针，逐个比较直到符合
- 当哈希冲突较多时，索引维护操作会有很高的代价，因此好的哈希函数非常重要

#### 应用

当我们需要经常需要当做条件进行查询的某个列非常长的时候，可以在B-Tree索引之上创建一个伪哈希索引，例如表中某列数据为`URL`，我们经常需要根据`URL`进行查询，如下

```sql
select * from tb_urls where url = 'https://sangedon.cn';
```

当表中数据较多且类似的查询语句较多时，相对于在原`url`列上创建索引，我们可以在表中加入一列`hash_url`，并删除原来的`url`列上的索引，在`hash_url`列上创建索引，我们的查询语句就可以写成这样

```sql
select * from tb_urls where url = 'https;//sangedon.cn' and hash_url = hash('https://sangedon.cn')
```

- 因为要避免哈希冲突问题，所以where子句中必须携带常量值的条件，如`url = 'https;//sangedon.cn'`

### 空间数据索引(R-Tree)

MyISAM支持空间索引，可作为地理数据存储。和B-Tree索引不同，此类索引无需前缀查询。空间索引会从所有维度来索引数据，查询时可以有效的使用任意维度来组合查询。因为MySQL的GIS支持并不完善，而维护索引必须使用MySQL的GIS相关函数，如MBRCONTAINS()，因此该索引类型较少被使用。开源关系数据库中对GIS的解决方案做得较好的是PostgreSQL和PostGIS。

### 全文索引

全文索引是一种特殊的索引类型，是根据文本中的关键词而非比较索引中得值进行查找。全文索引和其他积累索引的匹配方式完全不一样，有许多需要注意的细节，如停用词、词干和复数、布尔搜索等，更类似于搜索引擎(Baidu/Google)，而非简单的where条件匹配。

在相同的列上同时创建全文索引和基于值得B-Tree索引不会引发冲突，全文索引需要用`match against`而非普通的`where`操作。

### 其他索引类型

还有很多第三方存储引擎使用其他的数据结构作为索引，如ToKuDB使用`分形树`索引，这种较新的数据结构既有B-Tree的很多优点，也避免了一些B-Tree中存在的缺点。另外还有ScaleDB使用`Patricia tries`，其他一些存储引擎技术如 InfiniDB 和Infobright 则使用了一些特殊的数据结构来优化某些特殊的查询。

## 为什么要使用索引

根据前面的介绍我么已经知道了索引可以让服务器快速的定位到表的指定位置，我们常见的B-Tree索引根据其顺序存储数据的特性让我们可以进行诸如`order by`和`group by`等与数据有序性相关的操作，因为数据是有序的，所以B-Tree索引会将相关的列值存储在一起。索引中存储了实际的列值也让我们可以只通过索引就完成全部的查询，据此，我们使用索引的理由大概便有：

- 索引大大减少了查询过程中服务器需要扫描的数据量
- 索引可以帮助服务器避免排序和临时表
- 索引可以将随机`I/O`变为顺序`I/O`

然而，索引总是最好的选择吗？很显然，并不那么绝对。回头去看看索引的定义 `索引是存储引擎中用于快速找到记录的一种数据结构`，修饰短句为`用于快速找到记录的`，因此，使用索引的条件便很清晰了，即只有当索引帮助存储引擎快速查找到记录所带来的好处大于其额外工作成本时，索引才是有效的。例如，对于较小的表，通常全表扫描会更高效，而中大型表，索引就非常有效。对于超大型表，建立和使用索引的代价也会逐步升高，此时就需要使用类似于分区的技术来区分查询所需要的一组数据，而不是一条记录一条记录的匹配。

### 参考文章

《高性能MySQL》- 第五章 ：创建高性能的索引