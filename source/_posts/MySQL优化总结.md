---
title: MySQL常见优化汇总
date: 2016-2-03 11:23:26
categories: [编程艺术]
tags: [MySQL]
cover: https://cdn.jsdelivr.net/gh/huanshen1104/blog-imgs/202112031614620.jpg
---

## 查询优化

### MySQL查询过程
1. 客户端发送一条查询给服务器；
2. 服务器先会检查查询缓存，如果命中了缓存，则立即返回存储在缓存中的结果；
3. 服务器端进行SQL解析、预处理，再由优化器生成对应的执行计划；
4. MySQL根据优化器生成的执行计划，调用存储引擎的API来执行查询；
5. 将结果返回给客户端。

### 慢查询
1. 是否向数据库请求了不必要的数据；
2. 是否扫描了过多的数据，衡量查询开销三个指标：响应时间、扫描行数、返回行数；
3. 没有索引或者没有用到索引；

### 开启慢查询日志
- 方法一：全局变量设置

将 `slow_query_log` 全局变量设置为“ON”状态
> mysql> set global slow_query_log='ON'; 

设置慢查询日志存放的位置
> mysql> set global slow_query_log_file='/usr/local/mysql/data/slow.log';

查询超过1秒就记录
>mysql> set global long_query_time=1;

- 方法二：配置文件设置

修改配置文件 `my.cnf` ，在 `[mysqld]` 下的下方加入
````
[mysqld] 
slow_query_log = ON 
slow_query_log_file = /usr/local/mysql/data/slow.log 
long_query_time = 1
````
重启MySQL服务
````
service mysqld restart
````

### 查询优化神器explain
语法：`EXPLAIN SELECT * from user_info WHERE id < 300;`

EXPLAIN 命令的输出内容大致如下:

> mysql> explain select * from user_info where id = 2;
 id: 1 
select_type: SIMPLE 
table: user_info 
type: const 
possible_keys: PRIMARY 
key: PRIMARY 
key_len: 8 
ref: const 
rows: 1 filtered: 100.00 
Extra: NULL 
> 1 row in set, 1 warning (0.00 sec)

各列的含义如下:

- id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符
- select_type: SELECT 查询的类型.
- table: 查询的是哪个表
- **type: 显示连接使用的类型**
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引
- key_len:使用的索引的长度
- **ref:显示索引的哪一列被使用了**
- **rows: 显示此查询一共扫描了多少行. 这个是一个估计值**
- filtered: 表示此查询条件所过滤的数据的百分比
- extra: 额外的信息

`type` 参数可以帮助我们判断此次查询是全表扫描还是索引扫描，它的值有以下几种：

- system

表中只有一条数据，这个类型是特殊的 const 类型。

- const

针对主键或唯一索引的等值查询扫描，最多只返回一行数据，const 查询速度非常快。
> 实例：explain select * from user_info where id = 2

- eq_ref

此类型通常出现在多表的 join 查询，表示对于前表的每一个结果，都只能匹配到后表的一行结果，并且查询的比较操作通常是 =，查询效率较高。索引是UNIQUE或PRIMARY KEY。

> 实例： explain select * from user_info, order_info where user_info.id = order_info.user_id

- ref

此类型通常出现在多表的 join 查询，针对于非唯一或非主键索引, 或者是使用了最左前缀规则索引的查询。

- range

表示使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中。
> 实例： EXPLAIN SELECT  *  FROM user_info WHERE id BETWEEN 2 AND 8

- index

表示全索引扫描和ALL类型类似，只不过ALL类型是全表扫描，而index类型则仅仅扫描所有的索引而不扫描数据。

index类型通常出现在所要查询的数据直接在索引树中可以获取到而不需要扫描数据这种场景下，这个时候，Extra字段显示为Using index。

- all

表示全表扫描，这个类型的查询是性能最差的查询之一，通常来说, 我们的查询不应该出现ALL类型的查询，因为这样的查询在数据量大的情况下，对数据库的性能来说是巨大的灾难，如果一个查询是ALL类型查询，通常情况下可以对相应的字段添加索引来避免。

`rows` 显示MYSQL执行查询的行数，简单且重要，数值越大越不好，说明没有用好索引。

### 查询优化注意点
1. 不要使用 select * from table ，用具体的字段列表代替“*”，不要返回用不到的任何字段。
2. 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。
3. 当只要一行数据时使用LIMIT1。
4. 尽量不用负向查询（!=、NOT IN…）。
5. 尽量用in查询代替or条件。


## 索引优化
### 索引简介
1. **单列索引** 
由关键字KEY或INDEX定义的索引，其唯一任务是加快对数据的访问速度， 它的sql格式是 
`CREATE INDEX IndexName ON TableName(字段名(length))` 
或者
`ALTER TABLE TableName ADD INDEX IndexName(字段名(length))`

2. **唯一索引**
与普通索引类似，不同的是唯一索引要求所有的列的值是唯一的，这一点和主键索引一样，区别是它允许有空值，其sql格式是
`CREATE UNIQUE INDEX IndexName ON TableName(字段名(length))` 
或者 
`ALTER TABLE TableName ADD UNIQUE (columnList) `

3. **组合索引**
一个表中含有多个单列索引不代表是组合索引，通俗一点讲，组合索引是:**包含多个字段但是只有一个索引名称**，其sql格式是
 `CREATE INDEX IndexName On TableName(字段名(length), 字段名(length),...)`，如果建立了组合索引`(nickname,account,createTime)` ，那么他实际包含的是3个索引: 
`(nickname)`、`(nickname,account)`、`(nickname,account,createTime)`。

### 索引使用需要注意的点
1. **负向条件查询不能使用索引**
`select * from order where status!=0 and stauts!=1`
`not in/not exists` 都不是好习惯，可以优化为in查询：
`select * from order where status in(2,3)`

2. **前导模糊查询无法使用索引**
   `select * from order where desc like '%XX'` 无法使用索引
   `select * from order where desc like 'XX%'` 使用到了索引
    
3. **在属性上进行计算不能命中索引**
`select * from order where YEAR(date) < = '2017'`
即使date上建立了索引，也会全表扫描，可优化为值计算：
`select * from order where date < = CURDATE()`
或者：
`select * from order where date < = '2017-01-01'`

4. **复合索引最左前缀，并不是指SQL语句的where顺序要和复合索引一致**
用户表中建立了(login_name, passwd)的复合索引
`select * from user where login_name=? and passwd=?`
`select * from user where passwd=? and login_name=?`
都能够命中索引
`select * from user where login_name=?`
也能命中索引，满足复合索引最左前缀
`select * from user where passwd=?`
不能命中索引，不满足复合索引最左前缀

5. **数据区分度不大的字段不宜使用索引**
`select * from user where sex=1`
原因：SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引。

6. **强制类型转换会全表扫描**
`select * from user where telphone=13416348194`
如果telphone为varchar类型，数据库会进行强制类型转换而导致索引失效。

## 表结构优化
### 表的设计规范
1. **永远为每张表设置一个主键**      
- 主键递增，数据行写入可以提高插入性能
- 主键要选择较短的数据类型，因为较短的数据类型可以有效的减少索引的磁盘空间，提高索引的缓存效率

2. **尽可能的使用 `NOT NULL`**     
- `null` 的列使索引更加复杂，对MySQL来说更难优化
- `null` 值需要更多的存储空间，无论是表还是索引中每行中的null的列都需要额外的空间来标识
- 对 `null` 做处理的时候，只能采用 `is null` 或 `is not null`，而不能采用`=`、`in`、`<`、`<>`、`!=`、`not in`这些操作符号

3. **必须使用InnoDB存储引擎**
支持事务、行级锁、并发性能更好、CPU及内存缓存页优化使得资源利用率更高

4. **必须使用UTF8字符集**
无需转码，无乱码风险，节省空间

5. **字段类型尽量使用最小、最简单的数据类型**
如用 `TINYINT` 代替 `INT`，`VARCHAR` 的长度只分配真正需要的空间
占用更少的磁盘、内容、CPU缓存，大大减少IO开销


