# MySQL高级开发

## 一、内置函数

### 1、概念

>在开发称之为“方法”，将一组逻辑语句放在方法体中，对外暴露的方法名。

### 2、作用

>1、隐藏代码实现细节
>2、提高代码的重用性

### 3、语法

```mysql
select 函数名(参数) [from 表]
```

### 4、分类

>单行函数，例如： concat()、length()等。
>分组函数,例如：sum()、count()等。
>其他函数 now()

### 5、单行函数

#### 1.字符函数

##### 1）length：获取字节长度

>获取字节量，收到字符集影响。

```mysql
select length('abc');

select length('xiaowu');

show variables like '%char%';

select length('😀');

# 判断表种某列的字节,帮助我们确认数据类型,判断索引是否需要前缀。
select length(name) as len from world.city order by len desc ;
desc world.city;
```

##### 2）concat：字符串拼接

```mysql
mysql> select concat(" mysqldump -uroot -p123 ",table_schema," ",table_name," >/bak/",table_schema,"_",table_name,".sql") as sqltext from information_schema.tables where table_schema='world';
+--------------------------------------------------------------------------------+
| sqltext                                                                        |
+--------------------------------------------------------------------------------+
| mysqldump -uroot -p123 world city >/bak/world_city.sql                         |
| mysqldump -uroot -p123 world country >/bak/world_country.sql                   |
| mysqldump -uroot -p123 world countrylanguage >/bak/world_countrylanguage.sql   |
+--------------------------------------------------------------------------------+
```

##### 3）upper&lower：大小写转换

```mysql
select upper('abc')
select lower('CDE')
```

##### 4）substr：截取字符串

```mysql
语法： substr(字符串,pos,[len]) substring(字符串,pos,[len])


案例：
select SUBSTR('小龙女被尹志平**了',5,3);
select SUBSTR('小龙女被尹志平**了',1,3);
select substr('李莫愁爱上了陆展元',1,3) as test;
select substr('李莫愁爱上了陆展元',7) as test;
select substr('李莫愁爱上了陆展元',4,3) as test
```

##### 5）instr：返回子集首次次出现的索引

```mysql
案例：
select INSTR('ABCDBCA','B') AS test;

select INSTR('杨不悔爱上了殷六侠','殷六侠') AS test;


实际需求：判断某个字符串是否在表中某行出现过。
select id, instr(name,'qingdao') as a from world.city where countrycode='CHN' having a>0 ;
select sum(instr(name,'qingdao')) as a from world.city where countrycode='CHN';
```

##### 6）trim：掐头去尾

```mysql
select TRIM(' 张翠山 ') AS test;
select TRIM('a' from 'aaaaa张翠山aaaa') AS test;
```

##### 7）lpad：左填充

```mysql
select LPAD('张柏芝',10,'*') AS test;
```

##### 8）rpad：右填充

```mysql
select RPAD('张柏芝',10,'*') AS test;
```

##### 9）replace：替换

```mysql
# 案例：
select REPLACE('谢霆锋***张柏芝','张柏芝','王菲') as test;

#例子： 替换-为空
select replace(uuid(),'-','') as uuid;
```

#### 2.数学函数

##### 1）round：四舍五入

```mysql
select ROUND(3.1415);
select ROUND(3.1415,3);
```

##### 2）ceil：向上取 >= 最小整

```mysql
select CEIL(3.14);
select CEIL(3.00);
select CEIL(-3.14);
```

##### 3）floor：向下<= 最大取整

```mysql
select floor(9.99)
select FLOOR(9.00)
```

##### 4）truncate：小数点保留截断

```mysql
select TRUNCATE(-3.15,1);
```

##### 5）mod：取模

```mysql
案例：
select MOD(10,3)
select MOD(10,-3)
select MOD(-10,3)
select MOD(-10,-3)
```

##### 6）rand：生成某个范围内的随机整数

```mysql
select floor(rand()*10)+1;

例子： 略。
# 需求从以下生成的字符串中，随机截取连续6个字符。
select replace(uuid(),'-','') as uuid;
select substr(replace(uuid(),'-','') ,1+floor(rand()*28),10) as ps

# 生成一个第一字母为大写，复杂度为（大写字母、小写字母、数字组合）的12位密码
mysql> select concat(substr('ABCDEFGHIJKLMNOPQRSTUVWXYZ',1+floor(rand()*26),1),substr(replace(uuid(),"-",""),1+floor(rand()*21),11));
```

##### 7）综合案例

```mysql
# 生成随机时间字符串
11:34:15
00-23
00-59
00-59

# 1. 生成1-23随机整数
select rand(); -- 生成0-1之间任意小数
select rand()*23;-- 生成0-23之间任意小数
select floor(rand()*23) --生成0-22之间的随机整数
select 1+floor(rand()*23) --生成1-23之间的随机整数
select floor(rand()*24) --生成0-23之间的随机整数

# 2. 拼接
select concat(
lpad(floor(rand()*24),2,'0'),":",
lpad(floor(rand()*60),2,'0'),":",
lpad(floor(rand()*60),2,'0')
) as t ;

练习： 生成一个随机的IP地址
SELECT CONCAT(
FLOOR(RAND()*255),".",
FLOOR(RAND()*256),".",
FLOOR(RAND()*256),".",
FLOOR(RAND()*256)
) AS IP ;
```

##### 8）进制换算

```mysql
conv(n,from_base,to_base)
mysql> select conv("a",16,2);

ascii(str)
bin(n)
oct(n)
hex(n)

char(n,...)
返回由参数n,...对应的ascii代码字符组成的一个字串(参数是n,...是数字序列,null值被跳过)
mysql> select char(77,121,83,81,'76');
'mysql'
mysql> select char(77,77.3,'77.3');
'mmm'
```

#### 3.日期函数

##### 1）当前时间、日期函数

```mysql
select NOW();
select CURDATE();
select CURTIME();
select current_timestamp();
```

##### 2）截取时间

```mysql
select MONTH(NOW());
select MONTHNAME(NOW());
select DAY(NOW());
select HOUR(NOW());
select MINUTE(NOW());
select SECOND(NOW());

案例：
select (year(NOW())-year('1992-02-14 06:01:30'));
select FROM_UNIXTIME(12344234)
select UNIX_TIMESTAMP('2020-05-23')
SELECT DATE(FROM_UNIXTIME(UNIX_TIMESTAMP('1970-01-01') +
FLOOR(RAND() * (UNIX_TIMESTAMP('2020-05-23') - UNIX_TIMESTAMP('1970-01-01') + 1)))) AS date;
```

##### 3）以指定格式识别日期

```mysql
select STR_TO_DATE()
例子：
select STR_TO_DATE('5-3 2020','%m-%d %Y') as test;
```

| 格式符 | 作用                       |
| ------ | -------------------------- |
| %Y     | 4位年份，例如：1998        |
| %y     | 2位年份，例如：98          |
| %m     | 月份，例如： 01，02,...,12 |
| %c     | 月份，例如： 1，2,...,12   |
| %d     | 日期，例如01，02,..,31     |
| %H     | 24小时制                   |
| %h     | 12小时制                   |
| %i     | 分钟，例如：00-59          |
| %s     | 秒,例如：00-59             |

##### 4）以指定字符串格式输出日期

```mysql
select DATE_FORMAT()
select DATE_FORMAT(NOW(),'%Y-%m-%d');
```

#### 4.其他函数

```mysql
version()
database()
user()
uuid()
BENCHMARK(count,expr)
```

#### 5.流程控制函数

##### 1）if函数== >if else

```mysql
select if(2>1,'yes','no')

案例：
select user,if(user='root',"管理员","普通用户") from mysql.user
```

##### 2）case函数

###### ①等值判断

```mysql
case 表达式
when 等值判断 then 值1
...
else 值N。
end

select
case num
when 110 then CONCAT(num,':抓小偷')
when 119 then CONCAT(num,':救火')
else CONCAT(num,':救人')
end
as test
from tt;
```

###### ②范围判断

```mysql
case
when 条件1 then 结果或语句
when 条件2 then 结果或语句
else
end

# 案例：
统计每门课程:优秀(85分以上),良好(70-85),一般(60-70),不及格(小于60)的学生列表(case)
select course.cname,
group_concat(case when ifnull(sc.score,0)>=85 then student.sname end) as '优秀',
group_concat(case when ifnull(sc.score,0) between 70 and 85 then student.sname end ) as '良好',
group_concat(case when ifnull(sc.score,0) between 60 and 70 then student.sname end) as '一般',
group_concat(case when ifnull(sc.score,0)< 60 then student.sname end) as '不及格'
from student
join sc
on sc.sno=student.sno
join course
on course.cno=sc.cno
group by course.cno;


课程 优秀 良好 一般 不及格
语文 zs,ls w5 m6 s
```

## 二、变量

### 1、系统变量

#### 1.全局变量

```mysql
show global variables like '';
set global read_only=1;
select @@global.read_only;
```

#### 2.回话变量

```mysql
show session variables like '';
set session read_only=1;
select @@session.read_only;
```

### 2、自定义变量--用户变量

#### 1.作用域

>针对会话有效，会话任意位置使用。单独设置或者在存储过程函数都可。

#### 2.声明并初始化

```mysql
set @var=值;
set @var1:=值;
select @var2:=值;
```

#### 3.赋值（更新）

>注意：只能单一赋值

```mysql
方式1：
set @var=值;
set @var1:=值;
select @var2:=值;

方式2：
select count(*) into @count from world.city
```

#### 4.使用

```mysql
select @count;
```

#### 5.混合案例

```mysql
select replace(uuid(),'-','') into @str;
select substring(@str,floor(rand()*21+1),11) into @str1;
select
concat(substring('ABCDEFGHIJKLMNOPQRSTUVWXYZ',floor(rand()*26+1),1),@str1);
```

### 3、自定义变量--局部变量

#### 1.作用域

>必须在存储过程内部使用，即：begin ....end中。

#### 2.声明

```mysql
DECLARE 变量名 类型;
DECLARE 变量名 类型 DEFAULT 值;
```

#### 3.赋值

```mysql
set var=值;
set var1:=值;
select @var2:=值;
```

```mysql
select count(*) into count
from world.city

select id ,name ,age from t1 into v_id,v_name,v_age
```

#### 4.使用

```mysql
select count;
```

## 三、存储过程基础应用

### 1、语法

```mysql
DELIMITER $$
CREATE
    PROCEDURE `world`.`test`(参数列表)
    BEGIN
        过程体（1组SQL语句）
    END$$
DELIMITER ;
```

### 2、语法说明

```mysql
DELIMITER $$ :
说明：
语句结束标记定义。
过程体中，每条SQL都应该使用“;”结束。并且所有语句作为一个过程体运行。
如果过程体只有一条SQL，可省略begin end。

参数列表：
参数模式 参数名 参数类型
参数模式 ：
    IN ： 输入参数，单独传参。
    OUT ： 输出参数，作为返回值的参数。
    INOUT： 既输入有输出，可做输入也可做输出。
```

### 3、调用

>CALL 存储过程名(实参列表);

### 4、应用实例

#### 1.空参数列表应用

```mysql
DELIMITER $$
USE `world`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p1`()
    COMMENT 'p1'
BEGIN
    select *
    from world.city
    where countrycode='CHN';
END$$
DELIMITER ;

call p1();
```

#### 2.IN 参数列表应用

```mysql
DELIMITER $$
USE `world`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p2`(in cd varchar(20))
    COMMENT 'p2'
BEGIN
    select *
    from world.city as wc
    where wc.countrycode='cd';
END$$
DELIMITER ;

call p2('USA');

DELIMITER $$
USE `world`$$
CREATE PROCEDURE `p3` (in cd varchar(20) ,in pop int)
BEGIN
select * from city as a
where a.countrycode=cd
and a.population < pop;
END$$
DELIMITER ;

call p3('USA',90000);
```

#### 3.OUT 参数列表应用

```mysql
DELIMITER $$
USE `world`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p4`(out cc varchar(20),out count int)
BEGIN
select c.countrycode,count(*) into cc,count
from city as c
where c.countrycode='USA';
select cc,count;
END$$
DELIMITER ;

call p4(@aa,@bb);


USE `world`;
DROP procedure IF EXISTS `p5`;
DELIMITER $$
USE `world`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p5`(in bb varchar(20) ,out cc
varchar(20),out count int)
BEGIN
select c.countrycode,count(*) into cc,count
from city as c
where c.countrycode=bb;
select cc,count;
END$$
DELIMITER ;


CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p6`(in c int ,out aname
varchar(20),out bname varchar(20))
BEGIN
select a.name,b.name into aname,bname
from city as a join country as b
on a.countrycode=b.code
where a.population < c ;
select aname,bname;
END
```

#### 4.INOUT 参数列表应用

```mysql
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p7`(inout a int, inout b int ,out c int)
BEGIN
select a*2 into a ;
select b*2 into b ;
select a*b into c;
select a,b,c ;
END
set @a=10;
set @b=20;
call p7(@a,@b,@c)
```

#### 5.变量在过程中应用

##### 1）练习1

>需求： 存储过程中实现，往指定表中插入1行随机值
>uname:6字符随机长度。
>pass：12位随机密码，第一位是大写，剩下的是随机数字字母组合。

```mysql
use test;
create table t1(
id int not null primary key auto_increment,
uname varchar(64) not null ,
pass varchar(20) not null
)engine=innodb charset=utf8mb4;

select substr('abcdefghijklmnopqrstuvwxyz',1+floor(rand()*20),6);

USE `test`;
DROP procedure IF EXISTS `test`.`p_var1`;

DELIMITER $$
USE `test`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_var1`()
BEGIN
declare v_u varchar(64) ;
declare v_p varchar(20);
declare str,str_11 varchar(64);
declare str_1 varchar(64) default 'abcdefghijklmnopqrstuvwxyz';
select substr(str_1,1+floor(rand()*20),6) into v_u;
select replace(uuid(),'-','') into str;
select substring(str,floor(rand()*21+1),11) into str_11;
select
concat(substring('ABCDEFGHIJKLMNOPQRSTUVWXYZ',floor(rand()*26+1),1),str_11) into v_p;

insert into t1 (uname,pass) values(v_u,v_p);

END$$
DELIMITER ;

-- 调用方法：
call p_var1();
select * from t1;
```

##### 2）练习2

>/*
>需求： 存储过程中实现，往指定表中插入1行随机值
>uname:6字符随机长度。
>pass：12位随机密码，第一位是大写，剩下的是随机数字字母组合。
>u_time: 随机出生日期(1980-2020)，例如：1996-01-02
>u_age ：根据出生日期算出来。
>u_tel ：随机手机号
>*/

```mysql
-- 提示： 随机日期
SELECT DATE(FROM_UNIXTIME(UNIX_TIMESTAMP('1970-01-01') +
FLOOR(RAND() * (UNIX_TIMESTAMP('2020-05-23') - UNIX_TIMESTAMP('1970-01-01') + 1))))
AS date;

/* 1 38 00001111
1 30-99 00000000-99999999
*/
-- 生成随机手机号:
select concat('1',30+floor(rand()*70),lpad(floor(rand()*100000000),8,'0'));

-- 存储过程体：
USE `test`;
DROP procedure IF EXISTS `test`.`p_var2`;

DELIMITER $$
USE `test`$$
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_var2`()
BEGIN
-- 定义变量:
declare v_u,v_p,v_t,v_tel varchar(64);
declare v_a int;
declare str,str_11 varchar(64);
declare str_1 varchar(64) default 'abcdefghijklmnopqrstuvwxyz';
-- 给变量赋值：
select substr(str_1,1+floor(rand()*20),6) into v_u;
select replace(uuid(),'-','') into str;
select substring(str,floor(rand()*21+1),11) into str_11;
select
concat(substring('ABCDEFGHIJKLMNOPQRSTUVWXYZ',floor(rand()*26+1),1),str_11) into v_p;
SELECT DATE(FROM_UNIXTIME(UNIX_TIMESTAMP('1980-01-01') +
FLOOR(RAND() * (UNIX_TIMESTAMP('2020-01-01') - UNIX_TIMESTAMP('1980-01-01') + 1)))) into v_t;
select year(now())-year(v_t) into v_a;
select concat('1',30+floor(rand()*70),lpad(floor(rand()*100000000),8,'0')) into v_tel;
-- 调用变量：
insert into t2(uname,pass,u_time,u_age,u_tel)
values(v_u,v_p,v_t,v_a,v_tel);
END$$
DELIMITER ;

call p_var2()
select * from t2;
```

## 四、存储过程高级应用-流程控制结构

### 1、顺序结构

>从上至下一次执行

### 2、分支结构

>多条路径中选择其中一条

#### 1.case

```mysql
方法1：
case 变量|表达式|字段
when 判断的值 then 结果或语句
when 判断的值 then 结果或语句
...
else 结果或语句
end case;

方法2：
case
when 条件1 then 语句;
when 条件2 then 语句;
else 语句
end case;
```

#### 2.if

```mysql
if 条件1 then 语句1;
elseif 条件2 then 语句2;
...;
else 语句n;
end if;
```

#### 3.分支判断语句应用

##### 1）判断胖瘦

```mysql
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_if`(in tz int )
BEGIN
declare result varchar(10);
if tz <100 then
set result='靓';
elseif tz between 100 and 130 then
set result='壮';
else
set result='胖';
end if;
select result;
END
```

##### 2）判断年龄范围

```mysql
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_case1`(age int )
BEGIN
declare result varchar(10) ;
case
when age between 0 and 10
then
    set result='黄口小儿';
when age between 11 and 20
then
    set result='青少年';
when age between 21 and 30
then
    set result='青年';
when age between 31 and 50
then
    set result='中年';
else
    set result='退休了';
end case;
select result;
END
```

##### 3）判断输入的用户密码

```mysql
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_login`(in u varchar(20),in p varchar(20))
BEGIN
declare result varchar(20);
declare count int default 0;
select count(*) into count
from t2 where t2.username=u and t2.pass=p;
case
when count>0
then set result='success!';
else set result='error!';
end case;
select result;
END
```

### 3、循环结构

>满足条件，重复执行一段代码

#### 1.类型

```mysql
while
loop
repeat
```

#### 2.循环控制

```mysql
iterate ----> continue
leave -----> break
```

#### 3.while 语法

```mysql
[标签：] while 条件 do
循环体；
end while [标签]；
```

#### 4. loop 语法（死循环)

```mysql
[标签：] loop
循环体；
end loop [标签]；
```

#### 5.repeat

```mysql
[标签：] repeat
循环体；
until 条件
end repeat [标签]；
```

#### 6.应用

```mysql
/*
案例: t4表
id name age gender
1 asdfss 23 M
2 erfghj 32 F
... xsrhnc 22 M
10000 ertyup 18 F
需求： 随机往t4表插入1w行数据
id: 1-10000
name: 随机6个字符
age :18-35
gender : M/F
*/

CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p1_kz`(in num int)
BEGIN
declare str1 varchar(64) default 'abcdefghijklmnpqrstuvwxyz';
4.3.7 循环控制应用
declare str2 varchar(10) default 'MF';
declare v_name varchar(64);
declare v_age int;
declare v_gender varchar(10);
declare i int default 0;
while i<=num
do
    select substr(str1,1+floor(rand()*20),6) into v_name;
    select substr(str2,1+floor(rand()*2),1) into v_gender;
    set v_age=18+floor(rand()*12);
    insert into tt values(i,v_name,v_age,v_gender);
    set i=i+1;
end while ;
END

CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `pt_repeat`(in num int)
BEGIN
declare str1 varchar(64) default 'abcdefghijklmnpqrstuvwxyz';
declare str2 varchar(10) default 'MF';
declare v_name varchar(64);
declare v_age int;
declare v_gender varchar(10);
declare i int default 1;
repeat
    select substr(str1,1+floor(rand()*20),6) into v_name;
    select substr(str2,1+floor(rand()*2),1) into v_gender;
    set v_age=18+floor(rand()*12);
    insert into tt values(i,v_name,v_age,v_gender);
    set i=i+1;
    until i>num
end repeat;
END

CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_loop`(in num int)
BEGIN
declare str1 varchar(64) default 'abcdefghijklmnpqrstuvwxyz';
declare str2 varchar(10) default 'MF';
declare v_name varchar(64);
declare v_age int;
declare v_gender varchar(10);
declare i int default 1;
lab1:loop
    select substr(str1,1+floor(rand()*20),6) into v_name;
    select substr(str2,1+floor(rand()*2),1) into v_gender;
    set v_age=18+floor(rand()*12);
    if i>num
    then leave lab1;
    else
    insert into tt values(i,v_name,v_age,v_gender);
    set i=i+1;
    end if;
end loop lab1;
END
```

### 4、游标

#### 1.什么是游标

>保存select语句的数据集，主要用于对数据集逐行进行处理。

#### 2.使用方法

```mysql
定义游标 ：
    DECLARE cur_1 CURSOR FOR SELECT id,name FROM city;
定义游标异常处理：
    declare done int default 1;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 0;
打开游标：
    open cur_1;
提取数据:
    fetch cur_1 into c_id,c_name;
关闭游标:
    close cur_1;
```

#### 3.案例

```mysql
CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_cur`()
BEGIN
declare c_id int ;
declare c_name varchar(64);
declare done int default 1;
DECLARE cur_1 CURSOR FOR SELECT id,name FROM tt;
open cur_1;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
fetch cur_1 into c_id,c_name;
select c_id,c_name;
fetch cur_1 into c_id,c_name;
close cur_1;
END
```

#### 4.异常处理

```mysql
ERROR 1329 (02000): No data - zero rows fetched, selected, or processed
++++++
declare done int default 1;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 0;
或者：
DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET done = 0;
++++++

CREATE DEFINER=`root`@`10.0.0.%` PROCEDURE `p_cur1`()
BEGIN
declare c_id int ;
declare c_name varchar(64);
declare done int default 1;
DECLARE cur_1 CURSOR FOR SELECT id,name FROM tt;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 0;
open cur_1;
fetch cur_1 into c_id,c_name;

while done=1 do
select c_id,c_name;
fetch cur_1 into c_id,c_name;
end while ;
END
```

#### 5.存储过程查询及删除

```mysql
select * from information_schema.ROUTINES\G
drop procedure p_iterate;
```

## 五、自定义函数

### 1、语法

```mysql
CREATE FUNCTION `f1`(参数)
RETURNS int(11)
DETERMINISTIC
BEGIN
    函数体SQL；
RETURN 变量;
END
```

### 2、调用方法

```mysql
select f1();
```

### 3、删除及查看

```mysql
select * from information_schema.ROUTINES\G
drop FUNCTION f1;
```

## 六、触发器

### 1、介绍

>触发器是与表有关的数据库对象，在满足定义条件时触发，并执行触发器中定义的语句集合。
>触发器的特性：
>    1、有begin end体，begin end;之间的语句可以写的简单或者复杂
>    2、什么条件会触发：I、D、U
>    3、什么时候触发：在增删改前或者后(before/after)
>    4、触发频率：针对每一行执行
>    5、触发器定义在表上，附着在表上。
>也就是由事件来触发某个操作，事件包括INSERT语句，UPDATE语句和DELETE语句；可以协助应用在数据库端确保数据的完整性。

### 2、语法

```mysql
CREATE TRIGGER trigger_name
trigger_time: { BEFORE | AFTER }
ON tbl_name FOR EACH ROW
trigger_body
trigger_event: { INSERT | UPDATE | DELETE }
```

| 类型   | NEW和OLD使用                                               |
| ------ | ---------------------------------------------------------- |
| INSERT | NEW变量，获取Insert后的数据。                              |
| update | NEW变量，获取update后的数据；OLD变量，获取update前的数据。 |
| delete | OLD变量，获取删除前数据。                                  |

### 3、触发器应用

```mysql
#日志表应用
create table t4_log(
id int not null primary key auto_increment,
act_user varchar(64),
act_type varchar(50) ,
act_time varchar(50),
act_id varchar(20),
act_comment varchar(100));

# 插入触发器
DELIMITER $$
CREATE trigger tr_insert_t4
after insert on t4 for each row
BEGIN
insert into
t4_log(act_user,act_type,act_time,act_id,act_comment)
values(user(),'insert',now(),new.id,
concat('insert into t4
values(',new.id,',',new.name,',',new.age,',',new.gender,');'));
END$$
DELIMITER;

# 删除类触发器
DELIMITER $$
CREATE trigger tr_del_t4
before delete on t4 for each row
BEGIN
insert into
t4_log(act_user,act_type,act_time,act_id,act_comment)
values(user(),'del',now(),old.id,
concat("delete from t4 where id=",old.id,";"));
END$$
DELIMITER;
```

```mysql
## 商品库存自动更新
# 商品信息
create table goods(
    id int primary key auto_increment,
    name varchar(20) not null,
    price decimal(10,2) default 1,
    inv int comment '库存数量'
) ;

insert into goods
values
(null,'华为',11999,1000),
(null,'苹果',15999,50),
(null,'惠普',5999,2000),
(null,'小米',10999,2500),
(null,'戴尔',6999,3000);

# 订单
create table orders(
    id int primary key auto_increment,
    o_id int not null comment '商品id',
    o_number int comment '商品数量'
) ;

create trigger after_order
after insert on orders
for each row
begin
    update goods set inv = inv - new.o_number where id = new.id;
end
insert into orders(o_id,o_number) values(1,1,3);
```

## 七、事件管理器

### 1、介绍

>类似于Linux中的计划任务。

### 2、开启时间调度器

```mysql
#通过命令行
SET GLOBAL event_scheduler = ON;
SET @@global.event_scheduler = ON;
SET GLOBAL event_scheduler = 1;
SET @@global.event_scheduler = 1;

#通过配置文件my.cnf
event_scheduler = 1 #或者ON
```

### 3、应用

```mysql
# 案例1（立即启动事件）
create table ev1
(
ev_name varchar(20) not null,
ev_started timestamp not null);

create event event_now
on schedule
at now()
do insert into ev1 values('ev_test', now());


# 案例2（每分钟启动事件）
create event ev2
on schedule
every 1 minute
do insert into ev1 values('ev_test1', now());

# 案例3（每秒钟启动事件）
CREATE event ev3
ON SCHEDULE
EVERY 1 SECOND
DO INSERT INTO ev3 VALUES(1);

# 案例4（每秒钟调用存储过程）
CREATE DEFINER=`root`@`localhost` EVENT `eventUpdateStatus`
ON SCHEDULE EVERY 1 SECOND
STARTS '2017-11-21 00:12:44'
ON COMPLETION PRESERVE
ENABLE
DO call updateStatus()

# 过程式创建events
DELIMITER $$
//事件的名称
CREATE EVENT `test`
//60秒循环一次
ON SCHEDULE EVERY 60 MINUTE_SECOND
// 开始时间,结束时间
STARTS '2017-11-01 00:00:00.000000' ENDS '2017-11-30 00:00:00.000000'
//过期后禁用事件而不删除
ON COMPLETION PRESERVE ENABLE

DO
BEGIN
//执行的内容
insert into ev1 values('event_now', now());
insert into ev1 values('event_now1', now());
ENDR $$
DELIMITE ;
```

## 八、视图

### 1、介绍

>自定义视图。
>系统视图。

### 2、作用

>保存Select 语句执行方法。不保存数据。

### 3、自定义应用

```mysql
create view v_city
as
select a.name as aname ,b.name as bname ,a.population ,b.surfacearea
from city as a
join country as b where a.population<100;

select * from v_city;
```

### 4、系统自带视图

```mysql
查询系统中所有视图对象。
mysql> select table_schema,table_name,table_type from information_schema.tables
where table_type like '%VIEW%';
系统视图
information_schema(I_S),sys
```

### 5、I_S基本应用

#### 1.作用

>提供了用来查询系统“元数据”的视图。封装了元数据查询的方法。
>我们可以通过I_S和show更加方便的查询元数据信息。

#### 2.常用视图应用

>tables : 提供数据库中所有表相关元数据
>TRIGGERS ：提供数据库中所有触发器相关元数据
>views ：提供数据库中所有视图相关元数据
>ROUTINES ：提供数据库中所有存储过程相关元数据
>COLUMNS ：提供数据库中所有表中列相关元数据
>processlist ：提供数据库连接方面的系统状态。

#### 3.tables应用

##### 1）结构介绍

```mysql
desc information_schema.tables;
TABLE_SCHEMA -- 表所在的库。
TABLE_NAME -- 表名
TABLE_TYPE -- 表类型
ENGINE -- 存储引擎类型
TABLE_ROWS -- 数据行（粗略的）
AVG_ROW_LENGTH -- 平均行长度（字节）
DATA_LENGTH -- 数据长度
INDEX_LENGTH -- 索引长度
DATA_FREE -- 碎片量
CREATE_TIME -- 创建时间
UPDATE_TIME -- 更新时间
TABLE_COMMENT -- 注释
```

##### 2）应用实例

```mysql
1. 统计当前实例中业务相关的库和表的信息（排除掉mysql sys information_schema performance_schema）
库名 表个数 表名列表
mysql> select table_schema,group_concat(table_name),count(*) from
information_schema.tables where table_schema not in
('sys','mysql','information_schema','performance_schema') group by table_schema;


2. 统计当前实例每个数据库的数据总量（排除掉mysql sys information_schema performance_schema）
select table_schema,sum(table_rows * avg_row_length + index_length)/1024/1024 as total_mb
from information_schema.tables
where table_schema not in
('sys','mysql','information_schema','performance_schema')
group by table_schema;


3. 统计当前实例非innodb的表（排除掉mysql sys information_schema performance_schema）
select table_schema,table_name ,engine
from information_schema.tables
where table_schema not in
('sys','mysql','information_schema','performance_schema') and engine <>
'INNODB';
alter table world.aaaaa engine=innodb;

4. 查询有碎片的表信息
select table_schema,table_name ,data_free
from information_schema.tables
where table_schema not in
('sys','mysql','information_schema','performance_schema')
and data_free >0;

5. 拼接SQL

a. 查询当前系统中所有非INNODB的表。
select table_schema,table_name ,engine
from information_schema.tables
where table_schema not in
('sys','mysql','information_schema','performance_schema') and engine <>
'INNODB';

b. 将这些非INNODB的表替换为INNODB
mysql> select concat("alter table ",table_schema,".",table_name,"
engine=innodb;") from information_schema.tables where table_schema not
in ('sys','mysql','information_schema','performance_schema') and engine <>
'INNODB' into outfile '/tmp/alter.sql';
source /tmp/alter.sql
```

#### 4.TRIGGERS、views、ROUTINES、EVENTS应用

```mysql
需求： 迁移备份前需要确认是否有特殊对象
mysql> select TRIGGER_SCHEMA,EVENT_OBJECT_SCHEMA,TRIGGER_NAME ,ACTION_STATEMENT
,DEFINER from information_schema.triggers where TRIGGER_SCHHEMA not in
('sys','mysql','information_schema','performance_schema');

mysql> select ROUTINE_SCHEMA , ROUTINE_NAME,ROUTINE_DEFINITION ,DEFINER from
information_schema.ROUTINES where ROUTINE_SCHEMA not in
('sys','mysql','information_schema','performance_schema')\G
```

#### 5.columns

```mysql
select
TABLE_SCHEMA,
TABLE_NAME ,
COLUMN_NAME ,
IS_NULLABLE,
DATA_TYPE ,
COLUMN_KEY ,
COLUMN_COMMENT
from information_schema.columns where table_schema not
in ('sys','mysql','information_schema','performance_schema') ;
```

#### 6.processlist

```mysql
需求： 维护性操作需要停业务。需要将所有外部连接进行释放。(或者使用pt-kill)
select concat("kill ",id,";") from PROCESSLIST where host not in ('localhost','127.0.0.1','db01')
```
