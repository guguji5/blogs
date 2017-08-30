<h2 align='center'>性能优化</h2>

香满苑蛋糕屋的微信站bakery运行在了亚马逊云（1G内存+1核+8G硬盘），俄亥俄洲的主机上，平均有300ms的延迟，因为是试运行所以暂时也没有换的打算。在“时间就是金钱的”21世纪，性能优化就显得更为重要，主页load时间过长会让用户失去耐心，前端后台以及mongodb数据库的优化变得不可或缺。

这里分前端、后台和数据库三部分来记录一下我说做的工作。

### web前端的优化
1. 代码的合并、压缩、增加md5的版本都是webpack做的，以及把文件打包成几个bundle文件。减少请求次数。
2. 小图标根据需要缩小，不使用大图。1024*1024的图标直减为20*20。
2. 小图标的合并，css雪碧图的使用，减少请求次数。
3. 大量图片（包括雪碧图）从服务器拉取比较慢，上传到新浪微博的图床。
4. code split。

<div align=center>
<img src ='http://ww1.sinaimg.cn/large/7ec3646fgy1fj1kf77vfzj211v0jhwha.jpg'  width=350>
</div>

   从webpack的analysis中可以看出，资源文件打包成两个包。vendor较大，230k左右，app很小几十K。手机端浏览器大部分会同时拉取4个资源，根据木桶原理，最终完毕时间肯定由最慢的一个资源决定。所以将mint-ui的内容从vendor中转移到app中，两个大小都在150k左右。

<div align=center>
<img src ='http://ww1.sinaimg.cn/large/7ec3646fgy1fj1kpdm5djj21130hq0v1.jpg' width=80%>
</div>

### node后台的优化

1. 设置一个设置favicon.ico，不然每次都返回404。
2. 后台开启gzip超级有效，大概压缩率打到75%，从上边code split中可以看出159k的可以直接压缩成54k传过来。（有关gzip还专门翻译了一篇文章：[how to optimieze your site with gzip](https://github.com/guguji5/blogs/blob/master/how%20to%20optimieze%20your%20site%20with%20gzip.md)）
3. 占据渲染时间大半壁江山的都是图片，图片缓存很重要，从新浪图床上下载下来的，新浪服务器缓存的很好。少量从我服务器上的图通过IF-Modified-since/last-modified last-modified和if-none-match/etags来设置好缓存。未过期的值或者未变化的值浏览器端直接读取缓存。
![设置过期时间](http://ww1.sinaimg.cn/large/7ec3646fgy1fj1lfdimitj20ja03wdft.jpg)

### 数据库和微信（菜鸟备忘）

1. 指定需要返回的键

    db.users.find({},{"username":1,"email":1})   比db.users.find() 然后再解析数据肯定是要快。
2. mongodb启动时候要加用户名密码dbpath -auth，裸奔很危险。
3. 用户表（user ）的收货人数组的id通过生成随机数的方式来使其保证唯一，可以换成$ne或者$addToSet来将数组作为数据集使用，保证数组内的元素不会重复。
4. 评论放在产品详情（productDetail）表，然后把name和头像url都存进去。
db.blog.update({"title":"a blog post"},
{$push:{"commit":
{"name":"jone","headimgurl":"www.example.com/1.jpg",
"content":"nice post"}}})
5. 微信端jsapi_ticket,access_token等数据都有每日最大请求次数和过期时间，需要缓存在数据库，通过[TTL](https://docs.mongodb.com/manual/tutorial/expire-data/)的设置来实现过期expires。

    

