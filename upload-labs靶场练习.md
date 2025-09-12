<font style="color:rgb(25, 27, 31);">upload-labs靶场有较全的文件上传漏洞，借此来练习通过文件上传进行getshell的不同姿势。此次练习配合</font>[《通过文件上传进行Getshell的不同骚姿势》](https://forum.butian.net/share/381)进行食用，相互补充。

# Pass-01
查看源码，发现是通过客户端（JavaScript检测）进行文件类型的校验，只允许上传`.jpg|.png|.gif`这三种类型的文件：

```javascript
function checkFile() {
  var file = document.getElementsByName('upload_file')[0].value;
  if (file == null || file == "") {
    alert("请选择要上传的文件!");
    return false;
  }
  //定义允许上传的文件类型
  var allow_ext = ".jpg|.png|.gif";
  //提取上传文件的类型
  var ext_name = file.substring(file.lastIndexOf("."));
  //判断上传文件类型是否允许上传
  if (allow_ext.indexOf(ext_name + "|") == -1) {
    var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件,当前文件类型为：" + ext_name;
    alert(errMsg);
    return false;
  }
}
```

代码中可以看到是直接检查最后一个`.`后的类型名，改名`shell.jpg.php`行不通。

由于是由JS进行校验，所以一个思路是直接禁止浏览器使用JavaScript：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757592823978-11e525bb-d0a8-411a-b943-4c63a868357b.png)

上传`shell.php`:

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757639246795-3a2717a6-779f-4abb-bebf-795074a9e864.png)

上传成功。也可以将`shell.php`改名为`shell.jpg`，再通过抓包来改文件类型：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757639343778-5c5d05dd-c975-4510-8443-2223ff3789d2.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757639361578-6845e980-f647-41b9-99f8-82d268e70072.png)

上传成功。看到目标目录下有两个文件：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757639482457-b7c732e8-2d91-4e85-8079-280df3a62632.png)

webshell也能连上：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757639621963-910dba77-0cad-4dee-bc84-a7aba0d58713.png)

# Pass-02
查看源码，可以看到这段代码使用`$_FILES['upload_file']['type']`来获取上传文件的MIME类型（服务端检测），确保只有JPEG、PNG和GIF格式的文件可以被上传。

> MIME是一种标准的、用来表示文档、文件、字节流等数据的格式和性质的一种流媒体类型。浏览器通常采用MIME类型而不是文件扩展名来确定如何处理URL和文件内容。所以在响应头需要添加标准的mimetype，否则浏览器将不能以正常的方式来处理解析文件内容。
>
> MIME类型大全可参考：《[MIME类型](https://blog.csdn.net/qq_42482078/article/details/130886050?ops_request_misc=%257B%2522request%255Fid%2522%253A%252208fc1ce8bcd1ef8c49f658e3908a8770%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=08fc1ce8bcd1ef8c49f658e3908a8770&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-130886050-null-null.142^v102^pc_search_result_base8&utm_term=MIME&spm=1018.2226.3001.4187)》
>

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
  if (file_exists(UPLOAD_PATH)) {
    if (($_FILES['upload_file']['type'] == 'image/jpeg') || ($_FILES['upload_file']['type'] == 'image/png') || ($_FILES['upload_file']['type'] == 'image/gif')) {
      $temp_file = $_FILES['upload_file']['tmp_name'];
      $img_path = UPLOAD_PATH . '/' . $_FILES['upload_file']['name']            
        if (move_uploaded_file($temp_file, $img_path)) {
        $is_upload = true;
      } else {
        $msg = '上传出错！';
      }
    } else {
      $msg = '文件类型不正确，请重新上传！';
    }
  } else {
    $msg = UPLOAD_PATH.'文件夹不存在,请手工创建！';
  }
}

```

那么思路也明确了，就是上传`shell.php`，抓包修改`ContentType`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757640661960-04baf0a0-1625-4189-8404-b41ecf976077.png)

上传成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757640682344-c53fe92c-c177-4137-89f7-9b2707a35e33.png)

> 这里有个想法，代码里并没有显式删除上传的错误类型文件，而是在临时文件目录中，那么如果这个临时文件目录有执行权限是不是就可以getshell了？不过`upload_tmp_dir`显示的是`no value`，上传了一个内容为`<?php echo sys_get_temp_dir(); ?>`的`tempdir.php`文件，访问显示`/tmp`，但是这个目录下只有一些PHP会话文件，实在是没有找到这个临时目录在哪，也有可能存在别的删除规则。不过一方面`/tmp`目录本身应该没有执行权限，而且大多数Web服务器默认情况下应该会禁止访问`/tmp`目录中的文件，所以攻击应该不能成功。
>

# Pass-03
查看源码，发现是通过一种黑名单的方式对文件类型进行校验，`.asp`、`.aspx`、`.php`、`.jsp`这几种类型会被禁止上传。黑名单最大的弊端就是容易存在绕过，所以这也是本题的思路。

> 文件上传的绕过可以参考：《[总结一些文件上传的绕过方法](https://www.freebuf.com/articles/web/433766.html)》
>

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array('.asp','.aspx','.php','.jsp');
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //收尾去空

        if(!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;            
            if (move_uploaded_file($temp_file,$img_path)) {
                 $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '不允许上传.asp,.aspx,.php,.jsp后缀文件！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

从源码中可以看到，大小写绕过也不好使，所以决定找等价后缀名。

网上找到这几种文件类型的等价后缀名如下：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757648335774-afc54e7b-e12c-4f0e-969d-93a61ccd38a8.png)

不妨将`shell.php`重命名为`shell.php3`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757648416797-d3ab3b7e-454c-4ef6-978d-b9058e20a8ae.png)

上传成功。

> 这里本来尝试查看`httpd.conf`文件，但是Apache2没有，只找到了`/etc/mime.types`，而且其中关于`application/x-httpd-php`的部分都被注释掉了，这是因为现代的Apache配置通常不再直接依赖`mime.types`来处理PHP文件的解析，而是通过`mod_php`或`mod_proxy_fcgi`来处理PHP请求。  
>
> 除此之外，本题如果改成用白名单校验，<font style="color:rgb(36, 41, 47);">可以在满足要求的文件后插入木马脚本语句来绕过，对此，要确保上传目录的文件不可执行。  </font>
>

# Pass-04
从源码中可以看到，黑名单与上一题相比非常完善，而且细数代码逻辑，等价后缀名绕过、大小写绕过、空格绕过、点绕过、`::$DATA`绕过全被处理了，只能考虑下`.htaccess`文件绕过和`.user.ini`文件绕过。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2","php1",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2","pHp1",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //收尾去空

        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件不允许上传!';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}

```

由于`.user.ini`文件绕过步骤多一点，这里选择`.htaccess`文件绕过。

修改`/etc/apache2/apache2.conf`，把`AllowOverride None`改成`AllowOverride ALL`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757658089705-3c695506-88fa-443f-9e08-e5b98bc2d49a.png)

> <font style="color:rgb(51, 51, 51);">除了这个条件之外，其实还需要</font>`<font style="color:rgb(51, 51, 51);">mod_rewrite</font>`<font style="color:rgb(51, 51, 51);">模块开启，中间件得是Apache，不支持nginx。</font>
>

<font style="color:rgb(51, 51, 51);"> 接下来构建</font>`<font style="color:rgb(51, 51, 51);">.htaccess</font>`<font style="color:rgb(51, 51, 51);">：</font>

```plain
<FilesMatch "shell.jpg">
SetHandler application/x-httpd-php
</FilesMatch>
```

意思是`.htaccess`文件可以将`shell.jpg`文件当作`php`文件解析执行。由于文本文件不在黑名单内，是可以上传成功的：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757658668053-f1c11817-91e6-4625-b54d-29f516530f45.png)

上传`shell.jpg`，成功后直接getshell成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757658963050-26225782-972b-4047-8791-e58d81e193c4.png)

# Pass-05
这题一看源码，发现连`.htaccess`也在黑名单上，所以决定用`.user.ini`文件绕过。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空

        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件类型不允许上传！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

> 这种方法也有要求：
>
> 1. PHP版本为5.3以上
> 2. 服务器模式为CGI或FASTCGI模式下（一般指nginx中间件）
> 3. 上传的文件夹有PHP文件，因为这样你上传的文件内容才可以加入PHP文件中执行
>

~~首先~~（首先要以上帝视角在上传的文件目录放一个PHP文件`god.php`）构建`.user.ini`文件：

```plain
auto_prepend_file=shell.jpg
```

意思是在每个PHP文件请求执行前加载指定的文件`shell.jpg`，也就是让所有PHP文件都“自动”包含这个指定的文件。构建好后上传。

然后上传提前准备好的`shell.jpg`，尝试getshell发现失败了，在浏览器新标签页打开刚刚上传的`shell.jpg`，发现被重命名：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757661480690-c5d09a53-535a-4b99-a43c-fe12d8855f80.png)

这意味着刚刚的`.user.ini`也被重命名了，回看源码发现确实有一行重命名的代码`$img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;`，而且从Pass-03开始就有了，检查了下Pass-03上传的文件也会被重命名，但是为什么Pass-04上传的`shell.jpg`没有被重命名呢？而且通过Pass-04上传的`.user.ini`也没有被重命名。怀疑页面展示的源码不是真的源码。。。

进入Pass-04的`index.php`一看，果不其然，弄虚作假：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757663596451-e11f8db1-e654-4a03-8def-8605487c1c57.png)

不管了，先直接把Pass-05的代码也给改了，让这题完结先。

继续回来上传`.user.ini`和`shell.jpg`。还是连不上shell，怎么还有坑。。。又站在上帝视角检查了下，发现Server API是Apache 2.0 Handler，不支持`.user.ini`文件的处理。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757664489546-c3f7c976-8794-4fcd-a50c-167e7600e3dc.png)

尝试改成CGI/Fast CGI太麻烦，要掉进新坑，先跑了。

正常来说连接`http://192.168.159.128:8000/upload/god.php`即可getshell。

回顾源码，发现其实也有别的思路，比如如果在Windows环境，可以用点空格点绕过，上传`shell.php. .`，最后在上传目录中的是`shell.php.`，而在Windows中<font style="color:rgb(51, 51, 51);">文件后缀名最后一个点会被自动去除。</font>

<font style="color:rgb(51, 51, 51);">除此之外，黑名单依旧不完善，还是有一些</font>









