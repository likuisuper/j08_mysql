# 视图View

数据的基本对象:table,index,view,***函数,存储过程***

视图的本质是一张**"虚拟表"** , 实际上就是一条简单的或者复杂的sql.视图就是对查询语句的封装.

视图的分类:

* 简单视图 - 视图来自于单张表的查询 - 支持DML操作
* 复杂视图 - 视图来自于多张表的关联查询 - 不支持DML操作

视图的好处:

* 保证一些数据的安全性.
* 保证数据的清晰的结构,去除一些冗余的不需要的一些数据.
* 封装一些复杂的关联查询 - 多次进行调用.



## 创建视图的语法

~~~mysql
create view 视图名称
as select语句;

删除视图
drop view 视图的名称

查询视图
select * from 视图名称;
~~~

~~~mysql
create view emp_view
as select id,name,start_date from emp;

select * from emp_view;

-- 更新原表emp,看是否对视图emp_view产生影响
update emp set name = 'aaa' where id=1;
-- 查询视图("镜像")
select * from emp_view;

-- 结论1 - 修改原表,对视图造成影响

-- 更新视图 - 简单视图支持DML操作
update emp_view set name = '小小' where id = 1;
-- 查看原来的表
select * from emp;

-- 结论2: 更新视图,也会对原表产生影响.

-- 关于视图的查询,DML操作和对table的操作一模一样.
~~~

## with check option

禁止更新视图的时候,去update它的来源的原表的条件列.

~~~mysql
-- 删除视图
drop view emp_view;
-- 创建视图
create view emp_view
as select id,name,start_date from emp where id=1;
-- 查询视图
mysql> select * from emp_view;
+----+--------+------------+
| id | name   | start_date |
+----+--------+------------+
|  1 | 小小   | 2020-08-24 |
+----+--------+------------+
1 row in set (0.00 sec)

-- 更新视图
update emp_view set id = 10 where id=1;

mysql> select * from emp_view;
Empty set (0.00 sec)
~~~

~~~mysql
with check option一定是配合where语句使用.
如果创建视图的时候没有出现where语句,就没有必要出现它.

-- 删除视图
drop view emp_view;
-- 创建视图
create view emp_view
as select id,name,start_date from emp where id=1 with check option;

-- 更新视图 - 禁止更新视图中的视图来源的select语句的条件列
update emp_view set id = 10 where id=1;
ERROR 1369 (HY000): CHECK OPTION failed 'j08.emp_view'
~~~



## 复杂视图

复杂视图:视图来源于多表查询,***不支持DML操作的***

~~~mysql
create view emp_dept_region
as select d.id,d.name,r.name 区域名 from s_dept d join s_region r on d.region_id = r.id where r.name='Asia';

mysql> select * from emp_dept_region;
+----+------------+-----------+
| id | name       | 区域名    |
+----+------------+-----------+
| 44 | Operations | Asia      |
| 34 | Sales      | Asia      |
+----+------------+-----------+
2 rows in set (0.00 sec)
~~~



# limit语句

项目中只要涉及到查询的业务 - 涉及到分页的业务.

~~~mysql
select * from s_emp limit 5;//前5行的数据

limit m,n  m从0开始,第一行,n取多少条出来.
select * from s_emp limit 0,2;//第一行和第二行
~~~

~~~java
public List<User> findAll(Integer pageNow,Integer pageSize){
  //伪代码
  
  //1,3
  
  //第一行~第三行数据, limit 0,3
  
  //pageNow = 2, pageSize=3
  //第一页 limit 0,3
  //第二页 limit 3,3
  
  //limit (pageNow-1)*pageSize,pageSize;
}
~~~

## limit语句的性能问题.

sql优化操作.

~~~mysql
select * from xx where name = 'xxx';//全表扫描.
如果name是一个唯一性的值.

select * from xx where name = 'xxx' limit 1;//避免全表扫描了.
~~~

~~~mysql
偏移量如果过大,会导致limit语句性能极其低下
select * from xxx limit 100000,5;


select * from xxx where id>100000 limit 5;
~~~

