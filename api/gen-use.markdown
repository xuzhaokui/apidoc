---
layout: default
title: "访问概述"
---

- [准备工作](#prepare)
- [验证和Token](#auth-token)
  - [用户请求的验证](#auth-request)
  - [Upload Token](#upload-token)
  - [Download Token](#download-token)
  - [Access Token](#access-token)
- [上传模型](#upload-models)
  - [本地上传](#local-upload)
  - [客户端直传](#client-direct)
  - [回调上传](#callback-upload)
  - [重定向](#redirect-upload)
- [下载模型](#download-models)
  - [公有资源下载](#public-download)
  - [私有资源下载](#private-download)
- [资源管理](#rs-manage)
- [云处理](#fop)


<a name="prepare"></a>

## 准备工作

在开始使用七牛云存储之前，需要一些准备工作。

首先，需要获得AccessKey和SecretKey。这两个密钥用于用户请求的身份验证，可以从七牛云存储的“开发者平台”获得。用户如果还没有七牛云存储的帐号，需要先[注册](https://portal.qiniu.com/signup)。获得帐号后，登录进入“开发者平台”，而后进入[密钥管理](https://portal.qiniu.com/setting/key)，获取密钥。

一个用户最多可以拥有两对密钥，并且可以同时使用。通常情况下，用户只需使用一对密钥。但当用户需要切换密钥的时候，可以同时使用两对密钥，作为过渡。完成切换后，再删除原来的那个密钥。

AccessKey用来标识用户身份，会随同用户发起的请求传递到七牛云存储。七牛云存储通过AccessKey识别用户的身份。

SecretKey同AccessKey一一对应，用户使用ScreteKey对请求签名，而七牛云存储则使用SecretKey对请求作验证，以确认请求的合法性。

**注意：SecretKey用于签名请求，因此用户必须严格保密，不能泄露给第三方。亦不可将SecretKey至于客户端，分发给最终用户。一旦发生SecretKey泄露，请立刻更换密钥。**

<a name="auth-token"></a>

## 验证和Token

为了保护用户的资源，七牛云存储要求对每个上传请求、文件管理请求，以及私有资源的下载和云处理请求，都需要进行验证，确保访问都是用户授权的。

<a name="auth-request"></a>

### 用户请求的验证

验证方式并不复杂，大致有这样几个步骤：

1. 按照既定规则，将请求中的关键部分整合成待签名数据；
1. 以SecretKey为key，对带签名数据进行hmac_sha1()计算；
1. 对hmac所得的结果进行URL安全的Base64编码；
1. 将编码结果同AccessKey等数据合并起来，构成最终的token。

用户在自己的业务服务器中执行上述验证算法，获得token。然后将请求和token一同交付给最终客户端，客户端据此向七牛云存储发起请求，执行所需的操作。

如果用户仅仅是从自己的服务器，或者桌面计算机发起请求，那么执行验证算法后，可随即发请求。

但是，上述验证过程**绝对不可在最终客户端执行**，该过程涉及SecretKey，如在客户端执行，会不可避免地分发SecretKey，造成泄露，危及用户资源的安全。

七牛云存储的服务中用到三种token：

1. 用于上传的UploadToken；
1. 用于下载的DownloadToken；
1. 用于资源管理的AccessToken。

这三种token验证的过程基本一致，差别在于签名数据的合成，以及最后token的附加成分不同。

下面逐一介绍：

<a name="upload-token"></a>

### Upload Token

Upload Token用于资源上传请求的验证。由上传请求（使用[mulit-part格式的POST方法]()）的token字段携带，发送至七牛云存储。

一个Upload Token由三个部分组成：`<AccessKey>:<SignedData>:<UnsignedData>`。

其中：

1. `<AccessKey>`：用户的AccessKey，用于向七牛云存储表明请求者的身份；
1. `<SignedData>`：签名后的结果，即`urlsafe_base64_encode(hmac_sha1(<SecretKey>, <UnsignedData>))`；
1. `<UnsignedData>`：待签名数据块。将一组JSON格式的上传参数进行URL安全的Base64编码，获得的字符串即为`UnsignedData`。

关于上传参数，详见[Upload Token参考]()小节。

下面是一个Upload Token的示例：

`j6XaEDm5DwWvn0H9TTJs9MugjunHK8Cwo3luCglo:PDpKklPEog5x3bpcY5Jkgh0YsPY=:eyJzY29wZSI6IndvbGZnYW5nIiwiZGVhZGxpbmUiOjEzNzMxMDExOTN9`。

<a name="download-token"></a>

### Download Token

Download Token用于[私有资源]()的下载，以及对私有资源作云处理时的请求验证。由下载URL的[token参数]()携带，发送至七牛云存储。

一个Download Token由两个部分组成：`<AccessKey>:<SignedData>`

其中：

1. `<AccessKey>`：用户的AccessKey，用于向七牛云存储表明请求者的身份；
1. `<SignedData>`：签名后的结果，即`urlsafe_base64_encode(hmac_sh1(<SecretKey>, <UnsignedData>))`；

这里的`<UnsignedData>`是下载资源的URL（包含请求的过期时间）：`http://my-bucket.qiniu.com/the-key?e=1373013163`。详见[Download Token参考]()

下面是一个Download Token的示例：

`iN7NgwM31j4-BZacMjPrOQBs34UG1maYCAQmhdCV:vT1lXEttzzPLP4i5T8YVz0AEjCg=`

<a name="access-token"></a>

### Access Token

Access Token用于[资源管理]()的请求验证。由资源管理请求的HTTP头中的`Authorization`字段携带，发送至七牛云存储。

一个Access Token由三个部分组成：`QBox <AccessKey>:<SignedData>`

其中：

1. `QBox `：固定的前缀，后面紧跟一个空格；
1. `<AccessKey>`：用户的AccessKey，用于向七牛云存储表明请求者的身份；
1. `<SignedData>`：签名后的结果，即`urlsafe_base64_encode(hmac_sh1(<SecretKey>, <UnsignedData>))`；

`<UnsignedData>`是对整个请求的核心要素的整合，遵循以下规则：

- 如果请求不包含Body（HTTP请求没有携带Body），则`<UnsignedData>`为`<path>?<query>\n`。`<path>`是URL请求的[路径部分]()，`<query>`是URL的[查询部分]()。注意：最后的换行符`\n`是必须的。
- 如果请求的Content-Type不是`application/x-www-form-urlencoded`，那么即使请求包含Body，也将被忽略。
- 除上述情况外，请求的Body都将被包含在被签名数据中，即`<UnsignedData>`为`<path>?<query>\n<body>`。

下面是一个Access Token的示例：

`QBox j6XaEDm5DwWvn0H9TTJs9MugjunHK8Cwo3luCglo:qJLVPzrkTycA7pKb0_lSw7DYAjg=`

<a name="upload-models"></a>

## 上传模型

<a name="local-upload"></a>

### 本地上传

<a name="client-direct"></a>

### 客户端直传

<a name="callback-upload"></a>

### 回调上传

<a name="redirect-upload"></a>

### 重定向

<a name="download-models"></a>

### 下载模型

<a name="public-download"></a>

### 公有资源下载

<a name="private-download"></a>

### 私有资源下载

<a name="rs-manage"></a>

### 资源管理

<a name="fop"></a>

### 云处理
