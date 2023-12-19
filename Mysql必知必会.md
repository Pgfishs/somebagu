# 创建表
**-- 创建数据库**
`CREATE DATABASE demo；`
**-- 删除数据库**
`DROP DATABASE demo；
**-- 查看数据库**
`SHOW DATABASES;
**-- 创建数据表：**
`CREATE TABLE demo.test(barcode text,goodsname text,price int)`
**-- 查看表结构**
`DESCRIBE demo.test;
**-- 查看所有表**
`SHOW TABLES;`
**-- 添加主键**
`ALTER TABLE demo.test
`ADD COLUMN itemnumber int PRIMARY KEY AUTO_INCREMENT;
**-- 向表中添加数据**
`INSERT INTO demo.test
`(barcode,goodsname,price)
`VALUES ('0001','本',3);
# Mysql数据类型
在定义数据类型时，如果确定是整数，就用 INT；如果是小数，一定用定点数类型 DECIMAL；如果是字符串，只要不是主键，就用 TEXT；如果是日期与时间，就用 DATETIME
# Mysql增删查改
增
`INSERT INTO 表名 SELECT 字段 FROM 表名 WHERE 条件`
删
`DELETE FROM 表名 FROM`
改
`UPDATE 表名 SET 字段名=值 WHERE 条件`
查
`SELECT *|字段名 FROM 数据源 WHERE 条件
`GROUP BY 字段 HAVING 条件 ORDER BY字段 LIMIT 起始点、行数`
**GROUP BY**查询结果分组；**HAVING**查询搜索结果，和WHERE类似；**ORDER BY**查询结果如何排序，**ASC升序、DESC降序**；**LIMIT**显示部分查询结果
# Mysql如何正确设置主键
### 业务字段设置主键
容易因为需求而重复
### 自增字段设置主键
相同格式，内容不同的表之间容易出现冲突（会员卡和不同门店会员）
### 手动赋值主键
保证全局唯一性
# 外键和关联查询
### 如何创建外键
`CREATE TABLE 从表名 （ 字段名 类型...）CONSTRAINT 外键约束名 FOREIGN KEY（字段名） REFERENCE 主表名（字段名）
也可以修改表
`ALTER TABLE 从表名 ADD CONSTRAINT 约束名 FOREIGN KEY 字段名 REFERENCE 主表名（字段`
### 链接
#### 内连接
查询结果只返回符合链接条件的记录，JOIN、INNER JOIN、CROSS JOIN含义都是一样的，JOIN和ON配对使用，返回满足关联条件的所有内容
#### 外连接
还可以返回表中所有的记录，包括**左连接**和**右连接**
**左连接LEFT JOIN**，返回左边表中的所有记录和右表中符合连接条件的记录
右连接相反
### 关联查询误区
外键查询不是关联查询必要条件，有了外键约束，Mysql才会保证系统数据，避免出现误删
# where和having有什么区别
### where
WHERE 关键字的特点是，直接用表的字段对数据集进行筛选。如果需要通过关联查询从其他的表获取需要的信息，那么执行的时候，也是先通过 WHERE 条件进行筛选，用筛选后的比较小的数据集进行连接。这样一来，连接过程中占用的资源比较少，执行效率也比较高。
### having
having需要和group by组合使用
group by对数据按要求进行分组
### 如何正确使用where和having
- 如果需要通过连接从关联表中获取需要的数据，WHERE是先筛选后连接，而HAVING 是先连接后筛选。
- WHERE 可以直接使用表中的字段作为筛选条件，但不能使用分组中的计算函数作为筛选条件；HAVING 必须要与 GROUP BY 配合使用，可以把分组计算的函数和分组字段作为筛选条件
where先筛选数据再关联，执行效率高，不能使用分组中的计算函数进行筛选
havinf可以使用分组中的计算函数，但是在最后的结果集中筛选，执行效率低
# 聚合函数
### SUM
返回指定字段值的和；Left(str,n)返回str左边n个字符
求和函数获取的是分组中的合计数据，要知道按照什么字段进行分类的
### AVG、MAX、MIN
AVG返回平均值，通过计算分组内指定字段值的和以及分组内的记录数，算出指定字段的平均值
MAX表示获取指定字段在分组中的最大值，MIN表示获取指定字段在分组中的最小值。
### COUNT
了解数据集的大小，计算出符合条件的记录一共有多少，可以用于解决数据量过大的问题（进行分页）
# 时间类数据
![[Pasted image 20231217225931.png]]第一种方法是，可以利用 Windows 系统自带的网络同步的方式，来校准系统时间。
另一种办法就是，门店统一从总部 MySQL 服务器获取时间。由于总部的服务器的配置和运维状况一般要好于门店，所以系统时间出现误差的可能性也较小。如果采用云服务器，系统时间的可靠性会更高。
# 数学函数
![[Pasted image 20231219231556.png]]
# 索引相关
数据表字段越多，数据记录越多，索引速度提升越明显
### 单字段索引
1. 通过 CREATE 语句直接给已经存在的表创建索引
2. 可以在创建表的同时创建索引
3. 可以通过修改表来创建索引
#### 单字段索引原理
有了索引之后，MySQL 在执行 SQL 语句的时候多了一种优化的手段。也就是说，在查询的时候，可以先通过查询索引快速定位，然后再找到对应的数据进行读取，这样就大大提高了查询的速度。