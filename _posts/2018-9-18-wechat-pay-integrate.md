---
layout: post
title: 微信支付集成简介
categories: Wechat
description: 介绍微信各种支付方式集成相关的知识
keywords: wechat pay appid h5 jssdk app
---

# 微信支付集成简介

### 引言

在我们的项目中，微信支付广泛使用。目前微信支付的线上支付有以下几种方式:

* 公众号支付;
* 扫码支付;
* APP支付;
* H5支付;
* 小程序支付。

除了小程序支付，其它的支付我都集成过，小程序支付的方式和公众号支付非常相似。所以在这篇blog中简单介绍一下微信支付各种方式的集成方式，以及需要申请什么权限。

### 各种支付的使用场景

* 公众号支付：使用微信浏览器打开的网页支付时使用;
* 扫码支付：PC上的支付使用;
* APP支付：手机APP需要支付时使用，调用后会吊起微信进行支付，支付完后可返回APP;
* H5支付：手机上除微信以外的浏览器支付;
* 小程序支付：在小程序里进行支付。

### 基本支付接口

微信支付主要有以下几个接口:

* 统一下单;
* 查询订单;
* 关闭订单;
* 申请退款;
* 查询退款;
* 支付结果通知;
* 退款结果通知。

对于各种不同的方式，除了统一下单有区别外，其它的接口使用都是一样的，并不需要管是哪种支付方式下单的。

我们通过统一下单接口下单，用户支付后通过回调通知我们支付结果。另外我们可以通过查询接口查询订单状态，来应对回调失败的情况。对于超时的订单，可以调用关闭订单接口进行关闭，让用户无法再次支付。对于订单可以用退款接口进行退款，退款结果通过退款结果通知进行回调，也可以对退款结果进行查询，以防回调失败的情况。

所以本篇blog我们只会涉及到统一下单接口。

首先我们需要一个商户，拿到`MchID`和`AppSecret`两个参数，`MchID`是商户的ID，而`AppSecret`则是对请求进行签名的。

然后对于公众号、移动应用或者小程序等，都有一个`AppID`标识这个应用。然后需要在商户后台绑定这个`AppID`（如果是从公众号或者移动应用申请的商户，`AppID`默认已经绑定了）。只有绑定后，才能够下单。

绑定教程如下:

[https://pay.weixin.qq.com/static/pay_setting/AppID_protocol.shtml](https://pay.weixin.qq.com/static/pay_setting/AppID_protocol.shtml)

这样我们就有了`AppID`、`AppSecret`和`MchID`三个必须的参数。

### 扫码支付

微信叫做Native支付。这是一种适用于PC网页的支付方式。

这种方式只需要注意使用的`AppID`。因为这种方式不需要一个相应的应用，也就是不需要一个专门的`AppID`，任何和商户号绑定的`AppID`都可以使用。比如申请了微信支付能力的公众号的`AppID`，或者移动应用的`AppID`。

注意选择不同的`AppID`，最大的区别就是在微信支付对话框里ICON的不同，会使用应用中设置的ICON。

使用统一下单接口后会返回一个`code_url`，只需要使用`jquery.qrcode`生成二维码即可。

### H5支付

H5支付是在非微信的手机浏览器上使用的，点击支付后会跳转到微信，支付完成后可以跳转回来。

#### 权限申请

H5支付首先要申请权限，在商户平台-->产品中心-->开发配置中的H5支付里配置，添加调用H5支付的域名。

<img src="/images/posts/wechat/jssdk_permission.jpg" />

这种方式使用的`AppID`和扫码支付一样，不需要一个相应的应用，也就是不需要一个专门的`AppID`，任何和商户号绑定的`AppID`都可以使用。比如申请了微信支付能力的公众号的`AppID`，或者移动应用的`AppID`。

#### 集成注意点

H5支付有以下注意点:

在统一下单接口中，需要注意`spbill_create_ip`这个参数，必须是用户的`IP`。在打开H5支付时微信会检查用户的`IP`和下单中的IP是否相同，如若不同，会报错"网络环境未能通过安全验证，请稍后再试"。

调起H5支付时`referrer`不能为空，也就是不能直接访问H5支付的链接，需要在页面跳转访问，否则会返回"商家参数格式有误，请联系商家解决"。

还有就是上面说的，一定要去商户平台配置H5支付的域名，否则会报错"商家存在未配置的参数，请联系商家解决"。

H5支付的链接必须是在微信外打开的，在微信内打开，会报错"请在微信外打开订单，进行支付"。

#### 打开链接

有了上面注意点后，就可以用统一下单接口下单，返回的信息中的字段`MWEB_URL`就是H5支付需要打开的地址了。注意`MWEB_URL`的有效期是5分钟，超时会报错"支付请求已失效，请重新发起支付"。并且，同一个`MWEB_URL`只能被一个微信号调起。

在获取到`MWEB_URL`后可以拼接`redirect_url`来配置回调的`url`，这个`url`需要`urlencode`，并且域名需要和之前设置的域名一致。这个`url`的作用是微信那边支付完成后，跳回到浏览器后，浏览器重定向的页面。

另外由于设置`redirect_url`后，回跳指定页面的操作可能发生在：1,微信支付中间页调起微信收银台后超过5秒 2,用户点击“取消支付“或支付完成后点“完成”按钮。因此无法保证页面回跳时，支付流程已结束，所以商户设置的`redirect_url`地址不能自动执行查单操作，应让用户去点击按钮触发查单操作。

```
假设您通过统一下单接口获到的MWEB_URL= https://wx.tenpay.com/cgi-bin/mmpayweb-bin/checkmweb?prepay_id=wx20161110163838f231619da20804912345&package=1037687096

则拼接后的地址为MWEB_URL= https://wx.tenpay.com/cgi-bin/mmpayweb-bin/checkmweb?prepay_id=wx20161110163838f231619da20804912345&package=1037687096&redirect_url=https%3A%2F%2Fwww.wechatpay.com.cn
```

### 公众号支付

公众号支付是在微信浏览器内支付的方式。首先需要有一个公众号，有相应的`AppID`，然后在商户平台绑定这个`AppID`。

#### 权限申请

公众号支付需要设置两个权限，分别是支付目录和授权域名。

在商户平台-->产品中心-->开发配置中的公众号支付（参考H5支付里的配置），添加一个公众号的支付授权目录，也就是公众号里打开这个授权过的域名才能够调用支付。

开发公众号支付时，在统一下单接口中要求必传用户`openid`，而获取`openid`则需要您在公众平台设置获取`openid`的域名，只有被设置过的域名才是一个有效的获取`openid`的域名，否则将获取失败。具体页面如下:

<img src="/images/posts/wechat/h5_permission.jpg" />

#### 获取openid

公众号支付时，调用统一下单接口时，需要一个参数`openid`，也就是这个公众号对这个用户ID的标识。为了获取到`openid`，公众号支付的页面，不能直接打开。而是应该打开一个页面:

`https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirec`

这个页面的其实我们经常遇到，当我们在微信中打开一些页面的时候，会弹框让我们授权，这个页面就是授权用的。

参数	|是否必须|说明
- | :-: | -: 
appid	| 是	| 公众号的唯一标识
redirect_uri	| 是	| 授权后重定向的回调链接地址， 请使用 `urlEncode` 对链接进行处理，这个就是我们实际打开的页面
response_type	| 是	| 返回类型，请填写`code`
scope	| 是	| 应用授权作用域，`snsapi_base` （不弹出授权页面，直接跳转，只能获取用户`openid`），`snsapi_userinfo` （弹出授权页面，可通过`openid`拿到昵称、性别、所在地。并且， 即使在未关注的情况下，只要用户授权，也能获取其信息 ）。如果只需要支付，就填写`snsapi_base`，这个就只获取`openid`，但是不需要用户授权
state	| 否	| 重定向后会带上`state`参数，开发者可以填写`a-zA-Z0-9`的参数值，最多128字节
\#wechat_redirect	| 是	| 无论直接打开还是做页面302重定向时候，必须带此参数

在微信中打开这个页面后，微信将重定向到`redirect_uri/?code=CODE&state=STATE`。

在微信公众平台里面也有一个`AppSecret`，和支付的不同，这个可以用户获取用户的`openid`，在服务端通过请求

` https://api.weixin.qq.com/sns/oauth2/access_token?appid=APPID&secret=SECRET&code=CODE&grant_type=authorization_code`

参数	 | 是否必须	| 说明
- | :-: | -: 
appid	| 是	| 公众号的唯一标识
secret	| 是	| 公众号的`appsecret`
code	| 是	| 填写重定向带来的`code`
grant_type	| 是	| 填写为`authorization_code`

会返回

```
{ "access_token":"ACCESS_TOKEN",
"expires_in":7200,
"refresh_token":"REFRESH_TOKEN",
"openid":"OPENID",
"scope":"SCOPE" }
```

这样就可以获取这个用户的`openid`。

打开页面，获取到`openid`后，我们可以存储在`session`里，这样下次再打开时，不需要再次获取。

#### 开始支付

在上一步获取到`openid`后，我们可以将`openid`用到统一下单接口中进行下单。

调用统一下单后，可以获取到响应的参数，然后在页面里的js调用如下函数，就会调起微信支付，支付成功后，会回调最后一个参数的回调函数，可以在里面发起查询，看支付的状态:

```
WeixinJSBridge.invoke(
    'getBrandWCPayRequest', {
    "appId": res.appId,     //公众号名称，由商户传入
    "timeStamp": res.timeStamp,         //时间戳，自1970年以来的秒数
    "nonceStr": res.nonceStr, //随机串
    "package": res.package, // 下单接口中返回的prepay_id
    "signType": res.signType,         //微信签名方式：
    "paySign": res.sign //微信签名  
    },  
    function(res){
      if(res.err_msg == "get_brand_wcpay_request:ok" ){
      // 使用以上方式判断前端返回,微信团队郑重提示：
      //res.err_msg将在用户支付成功后返回ok，但并不保证它绝对可靠。 
      checkPay();
    }   
) 
```

tips:

* 使用微信原生的接口时，返回的`package`需要自己拼接前缀`prepay_id=`，然后再作为`package`，需要自己计算签名的值。

### APP支付

商户APP调用微信提供的SDK调用微信支付模块，商户APP会跳转到微信中完成支付，支付完后跳回到商户APP内，最后展示支付结果。

#### 权限申请

首先要申请微信开放平台的账号。

申请完开放平台后，在里面新建一个移动应用。这样我们就获取了一个`AppID`。


#### 下单

接下调用统一下单接口下单，获取到`prepay_id`。将下单信息返回到APP端，然后调起支付。

字段名   | 变量名 |   类型 |  必填 |  示例值 |  描述
应用ID |   appid |   String(32)  |  是  |  `wx8888888888888888`  |  微信开放平台审核通过的应用`AppID`
商户号  |  partnerid  |  String(32) |   是  |  1900000109  |  微信支付分配的商户号
预支付交易会话ID |   prepayid  |  String(32)  |  是  |  `WX1217752501201407033233368018`  |  微信返回的支付交易会话ID
扩展字段  |  package |   String(128)  |  是  |  `Sign=WXPay`  |  暂填写固定值`Sign=WXPay`
随机字符串 |   noncestr  |  String(32)  |  是  |  `5K8264ILTKCH16CQ2502SI8ZNMTM67VS`  |  随机字符串，不长于32位。推荐随机数生成算法
时间戳   | timestamp  |  String(10)  |  是   | 1412000000  |  时间戳，请见接口规则-参数规定
签名  |  sign  |  String(32)  |  是  |  `C380BEC2BFD727A4B6845133519F3AD6`  |  签名，详见签名生成算法注意：签名方式一定要与统一下单接口使用的一致

iOS和Android调用方式不同，这个客户端开发比较熟悉。

### 总结

* 以上简单介绍了几种微信支付的方式，同时提到了一些注意点，希望能在大家集成的时候有点用，少踩坑。
* 微信开发中，权限申请很耗时，这里介绍了权限的需求，可以减少申请的周期。
* 小程序支付因为没有用过，所以这里没有介绍。
