---
title: API安全实践探索
date: 2024-01-12 17:10:00 +0800
categories: [服务器]
tags: []
pin: false
---

在`原生APP`（客户端）与`服务器`数据交互过程怎样实现一套安全的机制？本文探索 `Web API` 在设计上可以选择的一些常见安全措施！

## 1. 防止非法请求: 使用token鉴权

鉴权指只有经过合法授权的用户才能调用我们的接口，常规的鉴权流程通常包含一些步骤：

- 用户首先需要通过 `OAuth平台`、`手机短信验证`、`账号密码` 等方式进行登录;
- 服务端校验账号，校验成功返回一个唯一`token`作为用户身份凭证，该 `token` 包含用户的一些基本信息;
- APP将`token`缓存，同时登录成功;
- 用户使用APP浏览数据，APP每次向服务端请求数据时须同时带上缓存的`token`，根据JWT规范，将`token`放在http报文头部的`Authorization`字段;
- 服务端收到请求，首先会校验`token`的合法性，校验成功正常返回数据，校验失败直接返回错误。

基于`token`机制，服务端可以实现基本的鉴权逻辑。我们可以采用的是`JWT 即 Json Web Token`（<https://jwt.io>）验证机制来实现以上过程。

## 2. 防抓包取敏感信息: 加密传输

### 2.1 为什么需要加密传输？

token验证机制并不能防止APP被抓包，恶意调用者只需要带上`token`再请求我们的API接口同样还是能获取到数据。因为APP与服务端都是明文通信，一抓包就能看到请求参数，我们必须要对 `敏感数据`进行加密处理。

通常，`手机号`、`姓名`、`地址`等可以被认定为`敏感数据`，而一些与查询数据相关的参数可以不认定为`敏感数据`，例如 `社区帖子ID`、`日期` 等！但也不绝对，在实际开发中，哪些是敏感数据需要根据项目实际情况进行判定!

### 2.1 加密和解密过程

假设我们认为设备信息是`敏感数据`，我们可以将用户的`设备信息`。将用户的设备信息，使用`JSON`数据格式进行拼装，包含以下字段

```json
{
  "openid": "设备唯一标识(设备号)",
  "channel": "渠道号",
  "version": "应用版本(versionName)",
  "source": "来源",
  "model": "型号",
  "os_version": "系统版本"
}
```

将以上数据使用约定的加密算法得到的加密字符串作为`query`参数的`params`字段中，请求接口时携带该`query`参数，如下

```
https://demo.com/user?params=hAWDCJ3LhVi3Ik2kY1Q/VA==
```

服务端得到加密串进行解密，得到用户的设备信息

### 3.1 加密方式

以上的加加密过程，推荐使用非对称加密，加密与解密过程为：生成一对密钥，包括公钥和私钥。公钥可以给APP客户端，而私钥必须保密（存放在服务器）。使用公钥加密的数据只能使用相应的私钥进行解密。这种加密方式提供了安全性和身份验证的功能，因为只有持有私钥的人才能解密数据。

## 3 防重放攻击: 签名验证

### 3.1 为什么需要签名验证？

数据加密之后虽然抓包之后看不到明文数据了，但是这并不能阻止不怀好意之人发起重放攻击。拦截到请求之后只需再原样发送该请求到服务端就可以发起重放攻击，如果接口存在比较耗性能的逻辑，那么在短时间内发起大量重放攻击的可能导致服务端瘫痪。因此推荐实现“签名验证”机制，同时整个过程推荐使用 “非对称加密”。

### 3.2 客户端加密生成签名

- 取所有请求参数（包括query参数、body的表单参数、body的json参数）
- 根据参数键名首字母，使用对应 `ASCII码值` 的大小进行升序排列，如果首字母一样比第2个字母，以此类推，必传参数如下
- 排序后的参数使用`&`符号拼接，再加上`appsecret`，如`a=1&b=2&appsecret=secret`，得到拼接之后的参数
- 把拼接之后的参数进行`MD5`加密，将`MD5`加密得到的结果转换为大写，最终得到签名字符串`sign`，最终请求任意接口时必带的query参数如下

| 参数      |  描述          |
| ------------- |:-------------:|
| appkey      | 后端分配的app_key |
| t     | 进行接口调用时的时间戳，即秒级时间戳 |
| sign     | 输入参数计算后的签名结果 |
| params     | 加密之后的设备信息参数 |

### 3.3 服务端验证签名

服务端收到请求后进行签名验证，如果签名校验成功，并且在该请求的是在规定时间发起的请求（例如签名之后的5分钟之内），则进行数据处理，否则拦截该请求。
