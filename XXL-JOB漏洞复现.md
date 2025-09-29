[XXL-JOB](http://www.xuxueli.com/xxl-job/)是一个知名的开源分布式任务调度平台，支持定时任务和分布式任务。开发中遇到过定时任务需求的小伙伴对它一定不陌生。同类的`@Scheduled`主要用于单机环境，Quartz又没有管理界面，这种时候可能就会选择XXL-JOB。

本文主要分析XXL-JOB的未授权访问导致RCE和CVE-2022-43183这两个漏洞，其他的后台弱口令、默认AccessToken等就不在此复现，可以阅读《[浅析xxl-obj分布式任务调度平台RCE漏洞](https://blog.csdn.net/weixin_39190897/article/details/135232151)》。

# 未授权访问致远程命令执行
该漏洞环境基于Vulhub快速搭建：

```bash
git clone https://github.com/vulhub/vulhub.git
cd vulhub/xxl-job/unacc
docker-compose up -d
```

XXL-JOB分为调度中心admin（8080端口）和执行器executor（9999端口）两侧。

阿里云、腾讯云等厂商对该漏洞的描述如下：默认情况下XXL-JOB的API接口没有配置认证措施，未授权的攻击者可以构造恶意请求，造成远程命令执行，直接控制服务器。漏洞利用无需登录，实际风险较高。2.2.0及以下的版本会被影响。

有意思的是XXL-JOB的作者并不认为这是一个漏洞，底层通讯是安全的，官网版本提供了鉴权组件，只是用户没有开启：《[XXL JOB 未授权访问致远程命令执行 "漏洞" 声明](https://mp.weixin.qq.com/s/jzXIVrEl0vbjZxI4xlUm-g)》。而问题出在XXL-JOB提供的GLUE模式，用户在线通过Web IDE开发的任务代码，远程推送至执行器，实时加载执行，但是如果执行器未开启访问令牌，会导致无法识别并拦截非法的调度请求。

下面开始复现。

访问`http://127.0.0.1:8080/`，发现Vulhub并没有提供调度中心的管理界面：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759130194365-7b9d70f3-ff6a-4709-9634-cc4806f193a8.png)

只能直接调用执行器的`/run`接口直接触发任务。请求格式参考官方文档的《6.2 执行器 RESTful API》：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759130409666-b073e909-077f-4b1a-b732-c14c58c5a803.png)

这里的`glueType`可以选择以下类型：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759130496920-971d085f-3bee-453f-acff-1acc8ecab169.png)

由于靶场部署在Ubuntu上，这里我选择`GLUE_SHELL`，具体请求包如下：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759130768583-0bb85774-6571-4e64-b906-e9d46a6d02b0.png)

此时攻击机也拿到了反弹Shell：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759130819337-c62aa86f-2c49-4b6d-a1d4-d75bf29aa882.png)

 针对该漏洞，作者提供了几种安全防护策略。

1. 开启XXL-JOB自带的鉴权组件：官方文档中搜索`xxl.job.accessToken`，按照文档说明启用即可。
2. 端口防护：及时更换默认的执行器端口，不建议直接将默认的9999端口开放到公网。
3. 端口访问限制：通过配置安全组限制只允许指定IP才能访问执行器9999端口。

在2.3.0版本的Release Notes中也可以看到，作者针对该漏洞进行了一些初步的校验，后续的安全强化也在规划中：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759131242776-2dc0268b-0940-4589-8c60-8bcfa075c43e.png)

# CVE-2022-43183
2.4.0版本的Release Notes中提到该版本修复了CVE-2022-43183这个SSRF漏洞（另一个CVE-2022-36157是一个很简单的垂直越权漏洞，不在此赘述）：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759131405657-f63fa910-040b-4b60-8510-b0f579c8ebad.png)

根据CVE的描述，`/admin/controller/JobLogController.java`中存在SSRF漏洞，2.3.1及之前版本都会受影响。具体的漏洞细节由该[issue](https://github.com/xuxueli/xxl-job/issues/3002)提出，根因是`/logDetailCat`直接向`executorAddress`指定的地址发送查询日志请求，而没有判断`executorAddress`参数是否为有效的执行器地址。查询请求会带有 `XXL-JOB-ACCESS-TOKEN`，导致`XXL-JOB-ACCESS-TOKEN`泄露，进而使攻击者获取`XXL-JOB-ACCESS-TOKEN`并调用任意执行器，导致执行器任意命令执行。`/logDetailCat`接口调用只需要平台低权限用户即可。

这里为了方便，仍然在上一个未授权访问漏洞环境（2.2.0版本）上进行分析。

看一下`/logDetailCat`接口：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759132123229-a7836f2e-3de9-4a50-be1d-0b20e6b70cf3.png)

执行器的执行地址`executorAddress`是可以自定义传入的，然后通过`XxlJobScheduler`的`getExecutorBiz`方法包装成执行器客户端，并且把`accessToken`传了进去：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759132191251-5e0c8b43-66d3-4bce-8e54-7fbaa7aff462.png)

调用执行器客户端`log`方法查询日志时会把`accessToken`明文带过去，而且并不会校验`address`的合法性:

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759133142138-39c79350-ce68-42d0-88a2-42828391b90b.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759133169384-58489745-65ed-46e2-83e7-4a49e2e1e121.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759133221486-a2000466-727c-46ca-a4e6-86b79663b43d.png)

所以复现的时候可以在攻击机启动`http`服务模拟执行器，就能收到调度中心的请求，并且从`header`中取到`accessToken`。

首先要创建一个普通用户（没有任何执行器权限）。~~由于Vulhub并没有提供调度中心的管理界面，所以直接调用~~`~~/xxl-job-admin/user/add~~`~~接口创建~~ 其实Vulhub是提供了调度中心的管理界面，访问`http://127.0.0.1:8080/xxl-job-admin`即可跳转到登录页面，默认登录账号密码是`admin/123456`。进入用户管理新建用户：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759135154016-84f3007d-b2b4-4a53-b867-a6803e81c9ca.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759135197263-cf81c7b9-f356-4a93-838b-0ec01ac24a32.png)

接着在本地启动一个`http`服务器，模拟执行器，并打印`http`请求头日志。这里用了滨伟哥的脚本：

```python
"""
@File：httpdemo.py
@Time：2024/1/2 22:25
@Auth：Tr0e
@Github：https://github.com/Tr0e
@Description：搭建http server并打印远程发送过来的http请求完整信息，用于SSRF漏洞窃取token
"""
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
from colorama import Fore, init

init(autoreset=True)  # 配置colorama颜色自动重置，否则得手动设置Style.RESET_ALL

data = {'result': 'HTTP SERVER OK'}
host = ('192.168.159.130', 6666)


class MyServer(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        # 发给请求客户端的响应数据
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps(data).encode())

    def do_POST(self):
        self.send_response(200)
        datas = self.rfile.read(int(self.headers['content-length']))
        print(Fore.RED + "[+]POST:", self.path, self.client_address)
        print(Fore.GREEN + str(self.headers))
        print(Fore.GREEN + str(datas))
        # 发给请求客户端的响应数据
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps(data).encode())


if __name__ == '__main__':
    server = HTTPServer(host, MyServer)
    print(Fore.YELLOW + "[*]Server已启动，地址信息: %s:%s" % host)
    server.serve_forever()

```

在攻击机运行该脚本：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759135515927-ba57be3b-0ccd-4d2f-a04a-439d22602561.png)

现在退出当前admin用户，以刚刚创建的普通用户身份登录后，调用`/xxl-job-admin/joblog/logDetailCat`接口，并将参数`executorAddress`设置为攻击机（即`http`服务器）的地址：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1759136858109-a8956492-e256-43ab-92d3-3cfe16ecdf02.png)



没有打印XXL-JOB

