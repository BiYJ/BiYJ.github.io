---
title: Sql
categories : 数据存储
---

## 一、[SQL](https://baike.baidu.com/item/%E7%BB%93%E6%9E%84%E5%8C%96%E6%9F%A5%E8%AF%A2%E8%AF%AD%E8%A8%80/10450182?fr=aladdin&fromid=86007&fromtitle=sql) 语句

1. 什么是 SQL

	> SQL 全称 structured query language 结构化查询语言，是用于访问和处理数据库的标准的计算机语言。

	
	- SQL 指结构化查询语言
	- SQL 使我们有能力访问数据库
	- SQL 是一种 [ANSI](https://www.ansi.org/) 的标准计算机语言

2. SQL 能做什么？

	SQL 可与数据库程序协同工作，比如 MS Access、DB2、Informix、MS SQL Server、Oracle、Sybase 以及其他数据库系统。
	
	- SQL 面向数据库执行查询
	- SQL 可从数据库取回数据
	- SQL 可在数据库中插入新的记录
	- SQL 可更新数据库中的数据
	- SQL 可从数据库删除记录
	- SQL 可创建新数据库
	- SQL 可在数据库中创建新表
	- SQL 可在数据库中创建存储过程
	- SQL 可在数据库中创建视图
	- SQL 可以设置表、存储过程和视图的权限

3. RDBMS

	RDBMS 指的是关系型数据库管理系统。
	
	<font color=#cc0000>RDBMS 是 SQL 的基础</font>，同样也是所有现代数据库系统的基础。
	
	RDBMS 中的数据存储在被称为表（tables）的数据库对象中。表是相关的数据项的集合，它由列和行组成。

4. 什么是 SQL 语句

	使用 SQL 语言编写出来的句子/代码，就是 SQL 语句。在程序运行过程中，要想操作数据库中的数据，必须使用 SQL 语句。

5. SQL 语句的特点

	* 不区分大小写（比如数据库认为 user 和UsEr 是一样的）
	* 每条语句都必须以分号 ';' 结尾。分号是在数据库系统中分隔每条 SQL 语句的标准方法，这样就可以在对服务器的相同请求中执行一条以上的语句。如果使用的是 MS Access 和 SQL Server 2000，则不必在每条 SQL 语句之后使用分号，不过某些数据库软件要求必须使用分号。

6. SQL 中的常用关键字

	select、insert、update、delete、from、create、where、desc、order、by、group、table、alter、view、index 等等。数据库中不可以使用关键字来命名表、字段。

## 二、SQL 语句的种类

1. 数据定义语句（DDL:Data Definition Language）可以用于对数据库中的表进行操作。包括 create、drop、alter。
	
	- CREATE DATABASE - 创建新数据库
	- ALTER DATABASE - 修改数据库
	- CREATE TABLE - 创建新表
	- ALTER TABLE - 变更（改变）数据库表
	- DROP TABLE - 删除表
	- CREATE INDEX - 创建索引（搜索键）
	- DROP INDEX - 删除索引

2. 数据操作语句（DML:Data Manipulation Language）可以用于对数据库中的数据进行操作。包括insert、update、delete。

	- UPDATE - 更新数据库表中的数据
	- DELETE - 从数据库表中删除数据
	- INSERT INTO - 向数据库表中插入数据

3. 数据查询语句（DQL:Data Query Language）可以用于查询获得表中的数据。关键字 select 是DQL(也是所有 SQL)用得最多的操作。其他 DQL 常用的关键字有 where、order by、group by 和 having。

## 三、基本操作

1. 创建表

	create table               表名 (字段名1 字段类型1, 字段名2 字段类型2, ...);
	
	create table if not exists 表名 (字段名1 字段类型1, 字段名2 字段类型2, ...);

	```
	create table t_st (id integer, name text, age integer, score real);
	```

2. 字段类型

	SQLite 将数据划分为以下几种存储类型：

	*    text      文本字符串
	*    real      浮点值
	*    integer   整型值
	*    blob      二进制数据(比如文件)

	注意：实际上 SQLite 是无类型的，就算声明为 integer 类型，还是能存储<font color=#cc0000>字符串文本(主键除外)</font>。建表时声明任何类型或者不声明类型都可以，也就意味着创表语句可以这么写：
	
	```
	create table t_st(name, age);
	```
	
	提示：为了保持良好的编程规范、方便程序员之间的交流，编写建表语句的时候最好加上每个字段的具体类型。

3. 删表

	drop table           表名;
	
	drop table if exists 表名;

	示例：
	
	```
	drop table t_st;
	```

4. 插入数据

	insert into 表名 (字段1, 字段2, ...) values (字段1的值, 字段2的值, ...); 

	示例：
	
	```
	insert into t_st (name, age) values ('mj', 10);
	```

	注意：数据库中的字符串内容应该用<font color=#cc0000>单引号</font>括住。

5. 更新数据

	update 表名 set 字段1 = 字段1的值, 字段2 = 字段2的值, ...; 

	示例：
	
	```
	update t_st set name = 'jack', age = 20; 
	```
	
	注意：上面的示例会将 t_st 表中所有记录的 name 都改为 jack，age 都改为 20

6. 删除数据

	delete from 表名 ;

	示例：

	```
	delete from t_st ;
	```
	
	注意：上面的示例会将 t_st 表中所有记录都删掉。

7. 条件语句

	如果只想更新或者删除某些固定的记录，那就必须在 DML 语句后加上一些条件。
	
	条件语句的常见格式：
	
	```
	where 字段 = 某个值 ;       // 不能用 ==
	where 字段 is 某个值 ;      // is 相当于 =
	where 字段 != 某个值 ; 
	where 字段 is not 某个值 ;  // is not 相当于!= 
	where 字段 > 某个值 ; 
	where 字段1 = 某个值 and 字段2 > 某个值 ;  // and 相当于 C 语言中的 &&
	where 字段1 = 某个值  or 字段2 = 某个值 ;  //  or 相当于 C 语言中的 ||
	```
	
	示例：
	
	```
	①、将 t_st 表中年龄大于 10 并且 姓名不是 jack 的记录，年龄都改为 5
	
	update t_st set age = 5 where age > 10 and name != 'jack' ;
	
	②、删除 t_st 表中年龄小于等于 10 或者年龄大于 30 的记录
	
	delete from t_st where age <= 10 or age > 30 ;
	
	③、将 t_st 表中名字等于 jack 的记录，score 字段的值都改为 age 字段的值
	
	update t_st set score = age where name = 'jack' ;
	```

8. DQL 语句

	```
	select 字段1, 字段2, ... from 表名 ;
	select * from 表名;  // 查询所有的字段
	```
	
	示例：
	
	```
	select name, age from t_st where age > 10 ; 
	```

9. 起别名(字段和表都可以起别名)

	```
	select 字段1 别名, 字段2 别名, ... from 表名 别名 ;
	select 别名.字段1, 别名.字段2, ... from 表名 别名 ;
	select 字段1 别名, 字段2 as 别名, ... from 表名 as 别名 ;
	```

	示例：
	
	```
	select name myname, s.age from t_st as s ;
	```

10. 计算记录的数量

	```
	select count (字段) from 表名 ;
	select count ( * ) from 表名 ;
	```
	
	示例：
	
	```
	select count (age) from t_st where score >= 60;
	```

11. 排序

	select * from t_st order by 字段 ;
	
	默认是按照升序排序(由小到大)，也可以变为降序(由大到小)
	
	select * from t_st order by age desc ;  // 降序
	
	select * from t_st order by age asc ;   // 升序(默认)
	
	也可以用多个字段进行排序。
	
	// 先按照年龄升序，相同年龄再按照身高降序
	
	select * from t_st order by age asc, height desc ;

12. limit

	使用 limit 可以精确地控制查询结果的数量，比如每次只查询 10 条数据
	
	select * from 表名 limit 数值1, 数值2 ;
	
	示例：
	
	```
	select * from t_st limit 4, 8 ;  // 跳过最前面 4 条语句，然后取8 条记录
	```
	
	limit 常用来做分页查询，比如每页固定显示 5 条数据，那么应该这样取数据：
	
	```
	第 n 页：limit 5*(n-1), 5    // limit 5 相当于limit 0, 5 取最前面的5 条
	```

## 四、约束

1. 简单约束

	建表时可以给特定的字段设置一些约束条件，常见的约束有
	
	*    not null     规定字段的值不能为 null
	*    unique       规定字段的值必须唯一
	*    default      指定字段的默认值
	
	建议：尽量给字段设定严格的约束，以保证数据的规范性。
	
	示例：
	
	```
	create table st (id integer, name text not null unique, age integer not null default 1) ;    // name 字段不能为null，并且唯一；age 字段不能为null，并且默认为 1
	```

2. 主键约束

	①、简单说明
	
	如果 t_st 表中就 name 和 age 两个字段，而且有些记录的 name 和 age 字段的值都一样时，那么就没法区分这些数据，造成数据库的记录不唯一，这样就不方便管理数据。
	
	良好的数据库编程规范应该要保证每条记录的唯一性。为此，增加了主键约束，也就是说，每张表都必须有一个主键，用来标识记录的唯一性
	
	②、什么是主键？
	
	主键(Primary Key，简称 PK)用来唯一地标识某一条记录。例如：t_st 可以增加一个 id 字段作为主键，相当于人的身份证。主键可以是一个字段或多个字段。
	
	③、主键的设计原则
	
	*    主键应当是对用户没有意义的 
	*    永远也不要更新主键
	*    主键不应包含动态变化的数据
	*    主键应当由计算机自动生成
	
	④、主键的声明
	
	在创表的时候用 primary key 声明一个主键
	
	```
	// integer 类型的id 作为t_st 表的主键
	
	create table t_st (id integer primary key, name text, age integer);
	```
	
	只要声明为 primary key，就说明是一个主键字段。主键字段默认就包含了 not null 和 unique 两个约束。
	
	如果想要让主键自动增长(必须是 integer 类型)，应该增加 autoincrement
	
	```
	create table t_st (id integer primary key autoincrement, ...) ;
	```

3. 外键约束

	利用外键约束可以用来建立表与表之间的联系。外键的一般情况是：一张表的某个字段，引用着另一张表的主键字段。
	
	新建一个外键：
	
	```
	// 以下外键的作用是用 t_st 表中的 class_id 字段引用 t_class 表的 id 字段
	
	create table t_st (id integer primary key autoincrement, name text, age integer, class_id integer, constraint fk_student_class foreign key (class_id) references t_class (id));    // t_st 表中有一个叫做 fk_t_student_class_id_t_class_id 的外键
	```

4. 表连接查询

	需要联合多张表才能查到想要的数据
	
	表连接的类型：
	
	*    内连接：inner join 或者join(显示的是左右表都有完整字段值的记录)
	*    左外连接：left outer join(保证左表数据的完整性)
	
	示例：
	
	```
	// 查询 0316iOS 班的所有学生
	
	select s.name, s.age from t_st s, t_class c where s.class_id = c.id and c.name = '0316iOS';
	```
	
## 五、文章

[W3school](https://www.w3school.com.cn/sql/index.asp)
[菜鸟教程](https://www.runoob.com/sql/sql-summary.html)