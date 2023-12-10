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