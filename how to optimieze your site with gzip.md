# 如何使用GZIP来优化你的网站
如果你想节省带宽提高网站速度，压缩是一种简单有效的方法。当我打算提高JavaScript的传输速率来开启GZIP压缩的时候，我犹豫了因为有旧版本浏览器的存在（IE6）。

然而在二十一世纪，我们大部分的流量来自于现代浏览器，坦白的讲，我们大部分的用户都是很懂技术的。我们不想让任何一个人在访问我们网站的时候卡顿，哪怕是他在用IE4.0和Wdinows95.谷歌和雅虎都开启了gzip压缩。一个现代的浏览器要想不仅要享受到现代网络信息还要享受到现代互联网的速度，就必须开启gzip压缩。以下是如何设置。

## 等等，为什么我们要开启gzip压缩
在此之前，我有必要解释一下什么编码。当你在互联网上想请求一个文件时,比如http://www.yahoo.com/index.html，你的浏览器会和服务器有一个会话，大概如下如所示。
![](https://betterexplained.com/wp-content/uploads/compression/HTTP_request.png)

1. 浏览器：嘿，给我来一个 index.html文件
2. 服务器：好的，让我去找找它是不是在~
3. 服务器：找到它了，我会返回一个成功的状态码（200 ok），我正在发送文件……
4. 浏览器：100kb？ 我滴天……等啊……等啊，好的，下载下来了

当然，实际的请求头和协议会更加正规一点。

但是，它生效了，我拿到了index.html文件。

## 那现在问题在哪呢？
好吧，这系统是正常的，但是太低效了，坦白讲100kb是一大段的文字，HTML是冗余的，每一个<html>,<table>,<div>都有一个几乎相同的闭合标签。虽然通篇文字都有重复，但是只要你砍掉任何的内容，html（以及它的一奶同胞xml）都不会正常显示。

当文件太大的时候有什么好办法呢，就是gzip压缩它。

如果我们传输一个替代原始大文件的zip的压缩文件给浏览器，就会节省带宽和下载时间。当浏览器可以下载zip文件，解压，并且渲染给用户。下载很快，页面加载也很快，用户心情就会very good。这个浏览器--服务器的会话大概是酱紫的：
![](https://betterexplained.com/wp-content/uploads/compression/HTTP_request_compressed.png)
1. 浏览器：嘿，给我来一个index.html，如果要有，给我来一个压缩版的可以吗
2. 服务器：容我找找……好，满足你，如果找到了给你压缩以下，gzip格式的哦
3. 服务器：yep，找到了，正在压缩，马上传给你。
4. 浏览器：太棒了，只有10kb，我来解压，并且渲染给用户。

情况很简单：文件越小，下载更快，用户感受更好。

不相信我？雅虎主页的html部分通过压缩从101kb直接压到了15kb：
![](https://betterexplained.com/wp-content/uploads/compression/yahoo.png)

## 残（淡）酷（定）的现实
变化的部分在于浏览器和服务器，它成功的发送过去一个压缩文件。对于gzip压缩的要点有两点：

- 浏览器发送一个请求头，告诉服务器接受压缩版本的文件（gzip和deflate是两种压缩算法）Accept-Encoding:gzip,deflate
- 如果文件压缩了,服务器返回一个头信息:Content-Encoding:gzip

如果服务器没有返回Content-Encoding的头信息，意味着这文件是没压缩的（浏览器可以直接解析的）。请求头Accept-Encoding只是浏览器的一个请求，而不是命令。如果服务器不返回压缩文件，浏览器就不得不处理那庞大的源文件。

## 服务器设置
“好消息”是我们没办法控制浏览器。他发“Accept-encoding: gzip, deflat”或者不发，请求头就在那里，不增不减。

我们需要做的只是给服务器配置，让它返回压缩版本。如果浏览器控制，我们可以为每一个人节省带宽。

对于IIS服务器，启动压缩在“设置”里。

如果你像我一样用nodejs来搭建服务器，那你中奖了，nodejs开启gzip非常的简单，只需增加两行代码搞定。


```
const express = require('express');
const app = express();

//express框架，前边肯定都是必要的，也就是只需安装compression组件，然后添加一下两句代码就好
const compression = require('compression');
app.use(compression());
```


对于Apache服务器，我们可以启动压缩输出也很简单直接。在你的.htaccess文件里增加如下代码：

```
# compress text, html, javascript, css, xml:
AddOutputFilterByType DEFLATE text/plain
AddOutputFilterByType DEFLATE text/html
AddOutputFilterByType DEFLATE text/xml
AddOutputFilterByType DEFLATE text/css
AddOutputFilterByType DEFLATE application/xml
AddOutputFilterByType DEFLATE application/xhtml+xml
AddOutputFilterByType DEFLATE application/rss+xml
AddOutputFilterByType DEFLATE application/javascript
AddOutputFilterByType DEFLATE application/x-javascript

# Or, compress certain file types by extension:
<files *.html>
SetOutputFilter DEFLATE
</files>
```
Apache服务器有两个压缩选择：

- mod_deflate 更简单设置更加标准。
- mod_gzip 看起来更加强大，可以预预缩文件。

Deflate更快，所以我用的它；当然如果想更爽，用mod_gzip。无论你选那种，Apache会检查浏览器是否发送“Accept-encoding”请求头来判断是返回压缩文件还是源文件。然而，一些旧版本浏览器会带来麻烦，需要一些特别的指令来纠正它。

如果你不能改你的.htaccess 文件，你可以用PHP来返回压缩的内容，给你的HTML文件一个.php 文件，把下边文件加在文件的最上边。
IN PHP :
```
<?php if (substr_count($_SERVER[‘HTTP_ACCEPT_ENCODING’], ‘gzip’)) ob_start(“ob_gzhandler”); else ob_start(); ?>
```
我们会检查“Accept-encoding”头，返回gzip压缩版本的文件（不然就返回源文件）。这简直像是在建设你自己的web服务器。如果你确实在搭建服务器，请用Apache来压缩你的文件。你肯定不想下载你的文件 just like bearing。

*这有点给apache打广告的意思啊*

## 验证压缩的效果
一旦你配置好了你的服务器，检查他是不是生效了。
- 在线查看：用[online gzip test](http://www.gidnetwork.com/tools/gzip-test.php)来检查你的网页是不是确实压缩了。
- 浏览器查看：在chrome谷歌浏览器，F12打开 开发者工具--> network页签（火狐和IE浏览器也是相似的）。刷新你的页面，点击这network航信息来查看。这“Content-encoding: gzip” 的头信息意味着内容压缩了传过来的。

![image](https://betterexplained.com/wp-content/uploads/2007/04/chrome-gzip-header.png)

点击“use large rows”表情来查看更多信息。包含了压缩以后的大小和源文件的大小。

![image](https://betterexplained.com/wp-content/uploads/2007/04/request-size.png)

奇迹般的，主页从187kb压缩到了59kb。

## 试试几个小栗子
我做了个几个页和一个下载demo：
- [index.html](https://betterexplained.com/examples/compressed/index.html) —— 默认压缩
- [index.htm](https://betterexplained.com/examples/compressed/index.htm) —— 通过在Apache上的.htaccess文件 增加 *.htm规则来压缩
- [index.php](https://betterexplained.com/examples/compressed/index.php) —— 通过php的头信息来压缩

随意下载这些文件，放到你的服务器，调整你的服务器设置。

## 警告
gzip压缩的出现如此的令人振奋，但是还有以下三个注意点：
- 低版本浏览器：一些浏览器接受压缩文件还是有问题（他们说他们可以但是他们并不行），如果你的站点必须在window95的网景1.0浏览器上，你可能不想要压缩文件。Apache mod_deflate设置了一些忽略规则来专门为旧浏览器。
- 已经压缩过的文件:大多数的图片，音乐和视频都已经压缩过了，不要浪费时间来压缩他们了。事实上，你可以只压缩那三巨头（HTML,CSS AND JAVARSCRIPT)。
- CPU负载：在传输过程中压缩文件耗费CPU但是节省带宽（用空间换时间）。通常压缩速率的选择需要权衡利弊。也存在一些预压缩静态文件的方法，但这要求更多的资源。考虑了cpu的耗费，压缩文件也是利大于弊。通过压缩实现更好的用户体验，更短的留白时间，值！


---

开启gzip压缩是优化网站最快的方法之一。大胆的用吧，让你的用户体验更棒。


*本文为翻译文章，欲了解原文请点击[原文链接](https://betterexplained.com/articles/how-to-optimize-your-site-with-gzip-compression/)*
