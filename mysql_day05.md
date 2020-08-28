# 函数function

mysql8.x默认不允许创建函数

~~~mysql
set global log_bin_trust_function_creators=TRUE;
~~~



~~~mysql
-- delimiter是用来重新定义mysql的分隔符
delimiter $$
create function 函数名([形参列表]) returns 数据类型
begin
	函数体 - 最后一定要有return+返回的结果
	-- return 10;
end $$
delimiter ;
~~~

~~~mysql
-- 传入俩个变量,返回这俩个变量的和
delimiter $$
create function adds(a int,b int) returns int
begin
	return a+b;
end $$
delimiter ;
-- 调用函数
select add(10,20);
~~~

-- 查看自定义的函数的创建的语法

~~~mysql
show create function 函数名称;
~~~

-- 删除函数

~~~mysql
drop function 函数名;
~~~

## 全局变量

~~~mysql
-- 求1~x之间的总和
drop function my_sum;
delimiter $$
create function my_sum(x int) returns int
begin
	-- 定义一个全局变量,初始值1
	set @i = 1;
	set @sums = 0;
	while @i<=x do
		-- 循环体
		set @sums = @sums + @i;
		-- 变量自增1
		set @i = @i + 1;
	end while;
	return @sums;
end $$
delimiter ;

select my_sum(100);

-- 可以访问全局变量@sums
select @sums;
~~~



## 局部变量

~~~mysql
1-100之间的和,但是不包括5的倍数.

drop function my_adds;
delimiter //
create function my_adds() returns int
begin
	-- 定义一个局部变量
	declare i int default 1;
	declare sums int default 0;
	success: while i<=100 do
		-- 循环体
		if i%5 = 0 then
		  set i = i + 1;
		  -- 继续迭代success标签修饰的while循环 - 等同于continue语句
			iterate success;
		end if;
		set sums = sums + i;
		set i = i + 1;
	end while;
	return sums;
end //
delimiter ;

select my_adds();

-- 局部变量只能在函数体中进行访问
mysql> select sums;
ERROR 1054 (42S22): Unknown column 'sums' in 'field list'
~~~



# 存储过程

客户端(Java程序) - 发送SQL到mysql服务器 - mysql-server端对sql语句进行编译 - 解释得到sql的执行结果 - 响应给mysql-client.

传统方式 - 每次想要mysql-client端执行sql语句,都是需要经过**连接DB,**经过**编译**处理,得到结果的过程. 

存储过程(procedure)  -为了完成一些特定的功能,提前将***sql预编译好***,存储在mysql-server端系统的执行计划表中.

第一次去调用存储过程时候,对sql进行预编译并且进行保存,第二次调用的时候,省去了编译sql的过程了.也是可以做到***标准的组件编程(封装sql语句)***



## 面试题

函数和存储过程的区别.

* 存储过程的签名上面不需要使用returns语句,存储过程可以没有返回值.
* 存储过程体中可以不需要使用return语句
* 函数是使用select语句调用,存储过程是使用call调用.
* 存储过程的形参中可以使用in/out来表示参数是输入/输出

## 语法体验

~~~mysql
-- 删除存储过程
drop procedure 存储过程名;
-- 创建存储过程
delimiter $$
create procedure 存储过程名([out|in 变量 数据类型])
begin
	-- 过程体
end $$
delimiter ;
~~~

~~~mysql
-- 体验 - 把s_emp表中的员工的平均薪资的sql进行预编译好放在mysql-server端执行计划表.
drop procedure s_emp_avg_pd;
delimiter $$
create procedure s_emp_avg_pd()
begin
	select avg(salary) from s_emp;
end $$
delimiter ;

-- 第一次调用存储过程,mysql-server端就会对存储过程中过程体中的sql语句进行预编译并且保存
call s_emp_avg_pd();

-- 第二次调用的时候,省去了再次对sql编译的过程,从而提升了查询的效率,复用了sql语句.
call s_emp_avg_pd();
~~~



## 输入/输出

* in - 存储过程调用者向存储过程中传入的数据
* out - 存储过程需要向外返回的数据.
* inout - 输入/输出



### 输入in

~~~mysql
drop procedure my_in_pd;
delimiter //
create procedure my_in_pd(in p_in int)
begin
	-- p_in是一个局部变量
	-- 获取一下这个参数p_in
	-- p_in是使用in来修饰的,p_in可以用来存储调用者传入进来的数据
	select p_in;
	-- 对这个p_in重新赋值
	set p_in = 2;
	-- 输出一下
	select p_in;
	-- 设置全局变量
	set @p_ins = 2;
end //
delimiter ;

-- 如果使用in来修饰. 传入值
-- ①直接传入一个字面量
call my_in_pd(1);

-- ②直接传入全局变量
set @result = 1;
call my_in_pd(@result);

-- 结果仍然是1
select @result;
~~~

输入out

~~~mysql
drop procedure my_out_pd;
delimiter //
create procedure my_out_pd(out p_out int)
begin
  -- out 变量 - return 变量 - 返回存储过程执行之后的结果
  -- p_out是null - out修饰的 - 不会接受数据的,只会输出数据
	select p_out;
	-- 对这个p_in重新赋值
	set p_out = 2;
	-- 输出一下2
	select p_out;
end //
delimiter ;

-- 如果参数是out修饰的,那么是不能传入一个字面量的
call my_out_pd(10);//error

-- 必须传入一个全局变量
set @results = 1;
call my_out_pd(@results);

-- 2 - out会把@results对应的存储过程中的变量的结果向外输出.
selet @results;
~~~

### 练习

-- 创建一个存储过程,根据员工的id来得到员工的first_name和salary.

~~~mysql
drop procedure s_emp_pd;
delimiter //
-- 存储过程签名中的变量不要和表中的列的名称冲突
-- 结合真实操作的时候,类型一定要加上数据类型[长度]
create procedure s_emp_pd(in eid int(7),out fname varchar(25),out sal float(11,2))
begin
	select first_name into fname from s_emp where id=eid;
	select salary into sal from s_emp where id=eid;
end //
delimiter ;

-- 调用存储过程
call s_emp_pd(1,@fname,@sal);

select @fname;
select @sal;
~~~

~~~mysql
drop procedure s_emp_pd;
delimiter //
-- 存储过程签名中的变量不要和表中的列的名称冲突
-- 结合真实操作的时候,类型一定要加上数据类型[长度]
create procedure s_emp_pd(in eid int(7),out fname varchar(25),out sal float(11,2))
begin
	select first_name,salary into fname,sal from s_emp where id=eid;
end //
delimiter ;

-- 调用存储过程
call s_emp_pd(1,@fname,@sal);

select @fname;
select @sal;
~~~

## 处理返回结果集

-- 找出工资大于1200的员工的id,first_name,salary

~~~mysql
解决方案有两种
① 已经弃用 - 游标操作 - 性能很低
② 创建第三张表来进行存储返回的结果集

业务 - 查询出来的列和原来表中的列一致.
create table s_emp_result as select id,first_name,salary from s_emp where 1=2;
drop procedure s_emps_pd;
delimiter //
-- 存储过程签名中的变量不要和表中的列的名称冲突
-- 结合真实操作的时候,类型一定要加上数据类型[长度]
create procedure s_emps_pd(in sal float(11,2))
begin
	insert into s_emp_result(id,first_name,salary) select id,first_name,salary from s_emp
	where salary>sal;
end //
delimiter ;

call s_emps_pd(1200);
~~~



## 条件语句

* if ... then .. else .. end if;

  ~~~mysql
  drop procedure if_pd;
  delimiter //
  create procedure if_pd(in x int)
  begin
    -- if语句单独使用
  	if x=1 then
  		select 1;
  	end if;
  	
  	-- if..else.语句,没有else if语句
  	if x=2 then 
  		select 2;
  	else
  		select 4;
  	end if;
  end //
  delimiter ;
  call if_pd(1);
  ~~~

* case ... when ... then...  else .. end case

  ~~~mysql
  drop procedure case_pd;
  delimiter //
  create procedure case_pd(in x int,out res int)
  begin
    declare param int default 0;
    set param = x + 1;
    case param
    	when 2 then
    		set res = 20;
    	when 3 then
    		set res = 30;
    	else
    		set res=50;
    end case;
  end //
  delimiter ;
  call case_pd(10,@res);
  select @res;
  ~~~



## 循环语句

* while...do...end while

  ~~~mysql
  drop procedure while_pd;
  delimiter //
  create procedure while_pd(in x int)
  begin
   	declare sums int default 0;
   	declare i int default 1;
   	while i<=x do
   		if i%2 = 0 then
   			set sums = sums + i;
   			set i = i + 1;
   		end if;
   		set i = i + 1;
   	end while;
   	select sums;
  end //
  delimiter ;
  call while_pd(100);
  ~~~

* loop...end loop - 等同于while(true)

  ~~~mysql
  drop procedure loop_pd;
  delimiter //
  create procedure loop_pd(in x int)
  begin
   	declare sums int default 0;
   	declare i int default 1;
  	success: 
  	loop
  		set sums = sums+i;
  		set i = i + 1;
  		if i = 101 then
  			-- 打破循环
  			leave success;
  		end if;
  	end loop;
  	select sums;
  end //
  delimiter ;
  call loop_pd(100);
  ~~~

* repeat ... until ... end repeat 

  等同于do...while

  ~~~mysql
  drop procedure repeat_pd;
  delimiter //
  create procedure repeat_pd(in x int)
  begin
   	repeat
   		-- 先进入循环体中执行一次,然后再进行条件判断.
   		set x = x + 1;
   		select x;
   	until x>0
   	end repeat;
  end //
  delimiter ;
  call repeat_pd(0);
  ~~~

* iterater

  ~~~mysql
  函数 - 相同continue语句
  ~~~



## 带事务

~~~mysql
-- 存取钱
drop procedure acc_pd;
delimiter //
create procedure acc_pd(in sid int(7),in tid int(7),in money double(7,2),in st int(2),out msg varchar(25))
begin
  -- declare @msg varchar default '0';
  -- 手动开启一个事务
	start transaction;
	update account set balance = balance - money where id = sid;
  
	-- 模拟一个错误
	if st=1 then
		-- 失败的
	  set msg = '发生错误了';
		-- 事务的回滚操作.
		rollback;
	else
    update account set balance = balance + money where id = tid;
    set msg = '操作成功!';
    select msg;
    -- 手动提交一个事务
    commit;
  end if;
  select msg;
end //
delimiter ;

-- 能够commit
call acc_pd(1,2,100,0,@msg);

-- 失败 - rollback
call acc_pd(1,2,100,1,@msg);
~~~

# 触发器

在myql中,当我们执行一些操作的时候,比如DML操作(触发器触发的事件),一旦事件被触发,那么

就会执行一段程序.**触发器本质上就是一个特殊的存储过程.**

## 分类

* after触发器 - 在触发条件之后执行
* before触发器 - 在触发条件之前执行

## 语法

~~~mysql
-- 删除触发器
drop trigger 触发器名称;
delimiter $$
-- 创建触发器
create trigger 触发器名
触发时机(after,before) 触发事件(insert,delete,update) on 触发器事件所在的表名
for each row
-- 触发器需要执行的逻辑.
begin
end
$$
delimiter ;
~~~

## 练习

1. 删除account表中的任意一条数据的时候,并且把这条数据插入到acc_copy表中

   ~~~mysql
   create table acc_copy as select * from account where 1=2;
   drop trigger acc_tg;
   delimiter //
   create trigger acc_tg
   after delete on account
   for each row
   begin
   	insert into acc_copy values(old.id,old.balance);
   end //
   delimiter ;
   
   -- 删除account表中的任何一条数据就行了
   mysql>delete from account where id = 3;
   ~~~

2. ***级联删除*** - 删除一的一方的之前,把这个一的一方的多的子记录全部删除.

   比如:删除的区域的时候,级联删除这个区域对应的所有的部门的信息

   ~~~mysql
   create table dept_copy as select * from s_dept;
   create table region_copy as select * from s_region;
   drop trigger region_tg;
   delimiter //
   create trigger region_tg
   before delete on region_copy
   for each row
   begin
   	delete from dept_copy where region_id = old.id;
   end //
   delimiter ;
   
   delete from region_copy where id = 1;
   ~~~

3. oracle数据中存在一个check自检约束,mysql不存在,mysql可以使用触发器的方式来对添加到表中的数据进行一个验证.

   ~~~mysql
   drop table cks;
   create table cks(
   	id int(7) primary key,
     age int(3)
   );
   -- 添加age数据的时候,age数据必须要保证是大于0并且小于等于18数据,否则插入失败!
   drop trigger cks_tg;
   delimiter //
   create trigger cks_tg
   before insert on cks
   for each row
   begin
   	declare msg varchar(25);
   	-- 验证age,age是表cks中是不存在的,new.age
   	if new.age<=0 or new.age>18 then
   		-- 插入失败的 - 抛出一个异常
   		signal sqlstate 'HY000' set message_text = 'age必须大于0并且小于等于18数据';
   	else
   		set msg = 'success';
   	end if;
   end //
   delimiter ;
   
   insert into cks values(1,100);
   ERROR 1644 (HY000): age必须大于0并且小于等于18数据
   
   insert into cks values(1,17);
   ~~~

   

# 数据库优化

* 分表

  * 水平 - 列不多,记录多[行多] - 比如可以根据表的month,year,hours,type把一张再拆分.

  * 垂直分 - 列比较多,但是记录[行不算多] - 需要对列进行拆分.形成1:1关系

    a -id name  sss ssss sss2 sss3 ss4s sss5 sss6 ssss7 sss8 ....

    b id  aid sss4sss5 ss6 ...

* 主从复制
  * master 主数据库节点
  * slave 从数据库节点 - 实时性,效率要求不是特别高的请求 - 请求到从数据库.

* ***sql优化操作***
  * select查询列不要出现*
  * 不鼓励使用order by语句
  * 查询唯一的非索引列的值的时候,配合limit 1语句,避免全表扫描.
  * 不要让索引列参与计算.

## sql优化

1. MySQL如何执行区分大小写的字符串比较？

   ~~~mysql
   select * from s_emp where binary first_name = 'Carmen';
   ~~~

2. 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎进行全表扫描。

3. 对查询进行优化，应尽量避免全表扫描，首先应考虑在where及order by涉及的列上建立索引。

4. 应尽量避免在where子句中对字段进行not null值判断，否则将导致引擎放弃使用索引而进行全表扫描

5. 尽量避免在where子句中使用or来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

​    select id from t where num=10 or num=20
​	可以这样查询：
​	*sel*ect id from t where num=10*
​	*union* 
​    select id from t where num=20*

6. 下面的查询也将导致全表扫描：(不能前置百分号)

   select id from t where name like ‘c%’;//走索引.

  7. not in也要慎用，否则会导致全表扫描，如：

     ~~~mysql
     对于连续的数值，能用between就不要用in了：
     select id from t where num between 1 and 3
     ~~~

8. 尽量避免在where子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：
   select id from t where num/2=100

   ~~~mysql
   	应改为:
     	select id from t where num=100*2
   ~~~

9. 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

10. 不要在where子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

11. 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。
    - 遵循最左原则.

13. 很多时候用exists代替in[查询性能很低]是一个好的选择：

​        select num from a where num in(select num from b)
​     	用下面的语句替换：
​     	***select num from a where exists(select 1 from b where num=a.num)***

14. 并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用 - 索引有效 - 数据控制30%;

15. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert及update的效率，因为insert或update时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。***一个表的索引数最好不要超过6个***，若太多则应考虑一些不常使用到的列上建的索引是否有 必要。

16. 应尽可能的避免更新 clustered [聚簇]索引数据列，因为clustered索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新clustered索引数据列，那么需要考虑是否应将该索引建为clustered索引。

17. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

18. 尽可能的使用**varchar**/nvarchar代替**char**/nchar，因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

19. 任何地方都不要使用select * from t，用具体的字段列表代替“*”，不要返回用不到的任何字段。

20. 避免频繁创建和删除临时表，以减少系统表资源的消耗。

    ~~~mysql
    临时表和表变量 - 推荐表变量来代替临时表的用法.
    ~~~

21. 临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件，最好使用导出表。

22. 在新建临时表时，如果一次性插入数据量很大，那么可以使用select into代替create table，避免造成大量log，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

23. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先truncate table，然后drop table，这样可以避免系统表的较长时间锁定。

24. 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

25. 尽量避免大事务操作，提高系统并发能力。



# 笔试题

* DDL,DCL[linux],DML,DTL,DQL
* ***左连接,右连接***,内连接
* 约束
* ***事务(ACID) - 事务的隔离级别以及脏读,可不重复读,幻读.***
* 索引的底层原理 - 聚簇索引和非聚簇索引
* myisam和innodb区别
* 悲观锁和乐观锁
* SQL查询语句5~10题.
* 索引失效的场景
* SQL优化操作.
* drop delete truncate区别.

