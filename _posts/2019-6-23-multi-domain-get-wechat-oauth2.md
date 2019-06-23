---
layout: post
title: 多个域名获取微信登录授权权限
categories: Wechat
description: 多个域名获取微信登录授权权限
keywords: Wechat openid oauth2 授权登录
---

# 多个域名获取微信登录授权权限

## 前言

经历过微信的各种开发，我们发现微信的授权非常复杂，各种地方都需要配置。比如针对公众号开发，有以下这些授权:

* JS接口安全域名，这些域名才能调用微信的JS-SDK;
* 网页授权域名，这些域名才能使用`OAuth2`协议获取到微信的`code`，进而获取到一些用户信息;
* 公众号IP白名单，只有这些IP才能获取到`access_token`，作为一些接口调用的凭证。

然而很多域名的配置有数量限制，比如网页授权域名，只有两个。一般我们一个测试环境和一个正式环境就占据了两个域名。有时候做一些活动可能要用到其他的域名，这时候我们就没法配置了。

但是我们可以通过一些技巧，让多个域名都获取到网页授权的权限。

## 微信网页授权简介

微信的网页授权，也就是一个`OAtuh2`协议。用户在微信内打开一个页面的时候，不是直接打开这个页面，而是打开一个
`https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect`的页面， 这时候如果获得用户的授权的话，微信会重定向到`REDIRECT_URI`，并且带上参数`code`和`state`。

`scope`有两种选择，`snsapi_base` （不弹出授权页面，直接跳转，只能获取用户openid），`snsapi_userinfo` （弹出授权页面，可通过`openid`拿到昵称、性别、所在地。并且， 即使在未关注的情况下，只要用户授权，也能获取其信息 ）。其中`snsapi_base` 对用户是无感知的，而`snsapi_userinfo`是弹框让用户授权一些权限，授权后才能重定向。下图为`scope`等于`snsapi_userinfo`时的授权页面：

![](http://mmbiz.qpic.cn/mmbiz/PiajxSqBRaEIQxibpLbyuSK39dMUJfWKTThhpgvErSzV7X5YHOZdnnhlVWHp5y8b4TXsBzKzjakPXichajlWls8Vg/0?wx_fmt=jpeg)

`state`重定向后会带上`state`参数，开发者可以填写`a-zA-Z0-9`的参数值，最多`128`字节。

所以这个授权的过程，主要是为了拿到`code`参数，拿到`code`参数后，可以去调用微信的接口获取到用户的`openid`和`access_token`。如果是`snsapi_userinfo`的授权，还可以使用`access_token`获取用户的个人信息以及`unionid`。

这里有什么作用呢？第一可以拿到用户的个人信息。第二，我们可以获取到用户的`openid`，如果用户在我们的网站中进行验证，比如注册了账号，我们可以将用户的账号的`openid`进行绑定。这样当用户下次打开这个网页的时候，无需进入我们的登录流程就可以自动登录上来了。

而这里的`REDIRECT_URI`是需要在微信后台配置的，否则就会报错。但是微信后台只有两个位置，所以只有两个域名可以得到授权。

## 通过中转获取授权

如果有的域名可以获取到授权，有的域名无法获取到授权，最基本的思路就是我们可以让能够获取到授权的域名获取到授权后，将`code`传给无法获取到授权的域名。

这是网上找到的一个解决方案[https://github.com/HADB/GetWeixinCode](https://github.com/HADB/GetWeixinCode)。

### 使用方法

只需如下步骤:

* 我们为`wechat.example.com`在微信后台设置一份授权域名;
* 在这个域名下放一个文件`get-wechat-code.html`;
* 假设域名`www.abc.com/hello`想获取到授权，我们只需要用地址`https://wechat.example.com/get-wechat-code.html?appid=XXXX&scope=snsapi_base&state=hello-world&redirect_uri=https%3A%2F%2Fwww.abc.com%2Fhello`;
* 最后我们会跳转到`https://www.abc.com/hello?code=XXXXXXXXXXXXXXXXX&state=hello-world`，从而拿到授权`code`。

### 实现原理

实现原理都在`get-wechat-code.html`里面，它其实做了两件事，会两次打开这个页面:

* 第一次打开这个页面的话，拼接出微信授权的地址，跳转到微信授权的地址，设置的`redirect_uri`是当前页面;
* 经过授权后，微信会将`code`和`state`附在当前页面的地址后，跳转回当前页面;
* 第二次打开这个页面的话，这个页面将`code`和`state`附在原来的`redirect_uri`后，重新跳转回原来的`redirect_uri`。

最主要的代码如下:

```
// code是当前页面的参数
// 第一次打开页面，是没有code的。
if (!code) {
    baseUrl = "https://open.weixin.qq.com/connect/oauth2/authorize#wechat_redirect";
    if (scope == 'snsapi_login' && !isMp) {
        baseUrl = "https://open.weixin.qq.com/connect/qrconnect";
    }   
    //第一步，没有拿到code，跳转至微信授权页面获取code，这里拼接出了获取微信授权的url
    redirectUri = GWC.appendParams(baseUrl, {
        'appid': appId,
        'redirect_uri': encodeURIComponent(location.href),  // 跳转回当前页面
        'response_type': 'code',
        'scope': scope,
        'state': encodeURIComponent(state),
    }); 
} else {
    //第二步，从微信授权页面跳转回来，已经获取到了code，再次跳转到实际所需页面
    redirectUri = GWC.appendParams(GWC.urlParams['redirect_uri'], {
        'code': code,
        'state': encodeURIComponent(state)
    }); 
}                                                                                                                                                                                            
location.href = redirectUri; // 跳转，有两次跳转
```

经过以上的方法，我们就让其它的域名也获取到了微信的授权登录。

## 更近一步

以上的方法有一个缺点，就是任何页面都可以获取到微信登录授权，虽然这不是一个太大的问题，因为拿到`code`还需要密钥才能获取到用户信息。但是我们可以做得更好，限制只有特定的域名才能获取授权。

方法也很简单，打开`https://wechat.example.com/get-wechat-code.html`的时候，我们检查`redirect_uri`是否是合法的地址。代码如下:

```
def get_wechat_code(form):
    host = urlparse(form['redirect_uri'].data).hostname  # 解析域名
    if not host.endswith('example.com') and host not in config.WECHAT_OAUTH.URL:  # 对于自己的域名和配置的特定域名
        return ErrorResponse(u'不合法的回调地址')
    return TempResponse('get_wechat_code.html')
```

## 小结

* 这里通过多一次的跳转，解决了微信限制回调域名只能设置一个的问题；
* 解决方式非常简单，但是也很实用；
* 这种方式并没有破坏权限的验证。
