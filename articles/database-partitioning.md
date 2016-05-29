3月是个多事之秋，原本计划上周完成的分享被生生拖到了现在，以上纯属吐槽（笑）。最近我的主要工作在于为系统中的大数据量表做切分，过程中考虑使用MySQL分区表特性，做了一些探索和分析，整理如下。

### 什么是分区表

将逻辑上的数据表在物理层面按指定的规则拆分成多个部分存储。

### 分区表解决的场景

当某张表的数据到达一定数量级，单表操作会出现性能下降。标准的SQL并不支持物理分区查询。这时，你可能会这样做：

- 在物理层将表拆分，在业务的数据层增加分表策略。
- 使用MySQL提供的分区表特性，业务层对物理分区无感。

在分表规则简单（复杂的规则分区表无法支持），对查询性能要求不是那么高（分区表的主键查询性能不高）的场景下，我们选择MySQL的分区表是简单高效的一种做法。比如用户行为日志，数据量大，分块规则简单（通常按日期切分），对实时查询性能要求不高（通常定时异步分析）。

### 如何创建MySQL分区表

以日志表按时间分区举例，说明如何创建并维护分区表。

首先，创建日志表（在创建之初就需要分区信息）。

```
# 按天分区，LESS THAN表示小于不包含等于的数据
CREATE TABLE logs (
    date date NOT NULL,
    name varchar(20) NOT NULL,
    value int(10) NOT NULL,
    KEY date (date)
) engine=innodb DEFAULT CHARSET=utf8 
PARTITION BY RANGE COLUMNS (date) (
    PARTITION logs_20160314 VALUES LESS THAN ('2016-03-15'),
    PARTITION logs_20160315 VALUES LESS THAN ('2016-03-16')
);
```

然后，每日需要创建新的分区项，以插入日期为当日的日志数据。

```
# 如果数据不满足任何分区条件无法插入
ALTER TABLE logs ADD PARTITION ( PARTITION logs_20160402 VALUES LESS THAN ('2016-04-03'));
```

最后，定期清理不需要的旧数据。

```
# 删除分区表会同时清除分区表所包含的数据！
ALTER TABLE logs DROP PARTITION logs_20160313;
```


### MySQL分区表的特性

在对日志表做分区表的过程中，我总结了MySQL分区表的一些特性：

- 支持四种分区规则 [MySQL分区表分区规则](http://dev.mysql.com/doc/refman/5.5/en/partitioning-types.html)
	- RANGE partitioning 基于指定字段的数值范围分区。 MIN < VALUE < MAX
	- LIST partitioning 基于指定字段的值的集合分区。 VALUE IN (V1, V2, V3...)
	- HASH partitioning 基于指定字段HASH算法的结果分区。 HASH(VALUE)
	- KEY partitioning 类似于HASH分区，由MySQL内部提供HASH算法，并且没有HASH字段为整型的限制。
- 在创建数据表时必须定义分区，非分区表无法添加分区。
- 分区表和唯一索引之间的关系
	- 每个唯一索引都必须使用分区表表达式中所涉及的到所有字段。
	- 主键被认为是特殊唯一索引。

```
# 创建失败，col1/col2没有在分区表达式中
CREATE TABLE t1 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col2)
)
PARTITION BY HASH(col3)
PARTITIONS 4;

# 创建失败，唯一索引没有包含所有的分区表达式所涉及的字段
CREATE TABLE t2 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1),
    UNIQUE KEY (col3)
)
PARTITION BY HASH(col1 + col3)
PARTITIONS 4;

# 创建成功
CREATE TABLE t1 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col2, col3)
)
PARTITION BY HASH(col3)
PARTITIONS 4;

# 创建成功
CREATE TABLE t2 (
    col1 INT NOT NULL,
    col2 DATE NOT NULL,
    col3 INT NOT NULL,
    col4 INT NOT NULL,
    UNIQUE KEY (col1, col3)
)
PARTITION BY HASH(col1 + col3)
PARTITIONS 4;
```

### 总结

- 是否选择分区表需要根据实际场景分析，一般实时性不高的日志系统推荐使用分区表。
- MySQL分区表支持四种分区方式，常用的一般是按范围或者HASH分区。
- 分区表一般不使用主键，如果需要使用则分区条件必须包含所有主键。



### 问题
一致性HASH算法？

http://dev.mysql.com/doc/refman/5.5/en/partitioning-linear-hash.html