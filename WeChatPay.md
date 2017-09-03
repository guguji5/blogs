<h2 align='center'>微信支付（2017-08-28）</h2>

### 写在前边的话
baery的公众号，折腾了三次终于把微信支付申请下来了，喜不自胜，周末实现了一下微信支付。周末除了游泳，就一直在做它了，看了好多博客。全都说坑，小马过河，边搞边填咯

---
微信支付首先得微信认证的服务号，认证后不是就能直接用微信支付了，还得申请“微信支付”。
微信支付根据文档有两种调用方式，一种是jssdk里边的“chooseWXPay”，这种需要引入jssdk的资源包。还有一种是利用微信浏览的内部对象"WeixinJSBridge", WeixinJSBridge.invoke('getBrandWCPayRequest')。这里是通过第二种方法实现的。

微信的文档上有一个微信支付的[流程图](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=7_4)，很重要，解析一下以备忘。微信支付有以下几个步骤：
1. 在你的微信站创建订单。
2. 通过微信的[统一下单](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=9_1)功能，在微信服务器创建一个金额和单据号都如你所愿的订单。
3. 统一下单成功后会返回一个prepay_id（无论你通过上边的哪种调用微信支付的方式都需要这个prepay_id）调起微信支付。
4. 微信付款后，微信服务器会回调你在统一下单接口填入的notify_url，把你微信下单的信息和付款的信息都返回给你，你需要“签名”并确认商户号和金额，及收款成功的回调事件都应在此接口。

这里有一个很关键的动词叫“签名”（sign），它是微信用来保证信息安全的。具体的签名操作在[这里](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=4_3)。举个栗子：

假设你要传的参数如下：

```
{   appid：	wxd930ea5d5a258f4f
    mch_id：	10000100
    device_info：	1000
    body：	test
    nonce_str：	ibuaiVcKdpRxkhJA
}
```
第一步：对参数按照key=value的格式（这里的key在你申请下来微信支付以后会让你配置），并按照参数名ASCII字典序排序如下：

```
stringA="appid=wxd930ea5d5a258f4f&body=test&device_info=1000&mch_id=10000100&nonce_str=ibuaiVcKdpRxkhJA";
```
第二步：拼接API密钥：

```
stringSignTemp=stringA+"&key=192006250b4c09247ec02edce69f6a2d" //注：key为商户平台设置的密钥key
sign=MD5(stringSignTemp).toUpperCase()="9A0A8659F005D6984697E2CA0A9CF3B7" //注：MD5签名方式
```
最终你要发送的数据即为：

```
{   appid：	wxd930ea5d5a258f4f
    mch_id：	10000100
    device_info：	1000
    body：	test
    nonce_str：	ibuaiVcKdpRxkhJA
    sign:9A0A8659F005D6984697E2CA0A9CF3B7    //这类可能叫“sign”也可以能叫“paySign”
}
```
### 结合代码回顾一下上边的4步
一、在你的微信站创建订单
- 在我的订单表创建一条订单，并返回id
- 把id作为“统一下单”的out_trade_no字段
- 为统一下单接口准备好数据，openid，out_trade_no,total_fee,attach,body

二、调用“统一下单”接口，获取prepay_id

- 将第一步准备好的数据，结合成统一下单接口的数据核实，进行签名（方法就在上边咯）
- 因为"统一下单"接口需要post过去的是xml数据，所以[封装方法](https://github.com/guguji5/bakery/blob/master/bakeryApi/wechat/unifiedorder.js)，将json数据转为xml
- 将xml发送到统一下单的接口
https://api.mch.weixin.qq.com/pay/unifiedorder
如果成功就会返回一段xml，大概长这个样子

```
<xml>
   <return_code><![CDATA[SUCCESS]]></return_code>
   <return_msg><![CDATA[OK]]></return_msg>
   <appid><![CDATA[wx2421b1c4370ec43b]]></appid>
   <mch_id><![CDATA[10000100]]></mch_id>
   <nonce_str><![CDATA[IITRi8Iabbblz1Jc]]></nonce_str>
   <openid><![CDATA[oUpF8uMuAJO_M2pxb1Q9zNjWeS6o]]></openid>
   <sign><![CDATA[7921E432F65EB8ED0CE9755F0E86D72F]]></sign>
   <result_code><![CDATA[SUCCESS]]></result_code>
   <prepay_id><![CDATA[wx201411101639507cbf6ffd8b0779950874]]></prepay_id>
   <trade_type><![CDATA[JSAPI]]></trade_type>
</xml>
```
- 你需要解析出来prepay_id，或许可以通过[字符串解析的方法](https://github.com/guguji5/bakery/blob/master/bakeryApi/dbconf/xmlParse.js)

三、拿到prepay_id，将数据签名调起微信支付


- 根据微信支付的[api](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=7_7&index=6)可以看到，我们需要的数据格式如下
```
{
           "appId":"wx2421b1c4370ec43b",     //公众号名称，由商户传入     
           "timeStamp":"1395712654",         //时间戳，自1970年以来的秒数     
           "nonceStr":"e61463f8efa94090b1f366cccfbbb444", //随机串     
           "package":"prepay_id=u802345jgfjsdfgsdg888",     
           "signType":"MD5",         //微信签名方式：     
           "paySign":"70EA570631E4BB79628FBCA90534C63FF7FADD89" //微信签名 
       }
```
- appId是你自己公众号的id，时间戳，随机字符生成都是小case，可能小困难在于获取签名，还是跟上边一样啊，把如下"appId","timeStamp","nonceStr","package","signType"五个字段进行签名得到paySign。
- 至此，微信支付应该能成功调起。
- 
<div align=center>
<img src='http://ww1.sinaimg.cn/large/7ec3646fgy1fizfr7ybcij20f00qodg9.jpg' width=350>
</div>

四、微信支付成功以后，微信服务器调用notify_url

```
<xml>
  <appid><![CDATA[wx2421b1c4370ec43b]]></appid>
  <attach><![CDATA[支付测试]]></attach>
  <bank_type><![CDATA[CFT]]></bank_type>
  <fee_type><![CDATA[CNY]]></fee_type>
  <is_subscribe><![CDATA[Y]]></is_subscribe>
  <mch_id><![CDATA[10000100]]></mch_id>
  <nonce_str><![CDATA[5d2b6c2a8db53831f7eda20af46e531c]]></nonce_str>
  <openid><![CDATA[oUpF8uMEb4qRXf22hE3X68TekukE]]></openid>
  <out_trade_no><![CDATA[1409811653]]></out_trade_no>
  <result_code><![CDATA[SUCCESS]]></result_code>
  <return_code><![CDATA[SUCCESS]]></return_code>
  <sign><![CDATA[B552ED6B279343CB493C5DD0D78AB241]]></sign>
  <sub_mch_id><![CDATA[10000100]]></sub_mch_id>
  <time_end><![CDATA[20140903131540]]></time_end>
  <total_fee>1</total_fee>
  <trade_type><![CDATA[JSAPI]]></trade_type>
  <transaction_id><![CDATA[1004400740201409030005092168]]></transaction_id>
</xml>
```
- 支付成功以后，微信会往你的nofify_url地址发送如上图一样的一段xml
- 首先把xml里的信息（除sign字段外）进行签名，并与sign字段比较是否相等
- 我们需要解析的是需要根据里边的out_trade_no去自己的系统查询交易信息(主要是交易金额），是否正确
- mch_id是否是你自己的“微信支付商户号”
- 判断return_code和result_code字段是否SUCCESS
- 如果以上全ok那代表信息安全且有效，钱进入你的账户



<div align=center>
<img src ='imgs/qrcode.jpg'>
</div>

