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

但是，上述验证过程 **绝对不可在最终客户端执行** ，该过程涉及SecretKey，如在客户端执行，会不可避免地分发SecretKey，造成泄露，危及用户资源的安全。

七牛云存储的服务中用到三种token：

1. 用于上传的UploadToken；
1. 用于下载的DownloadToken；
1. 用于资源管理的AccessToken。

这三种token验证的过程基本一致，差别在于签名数据的合成，以及最后token的附加成分不同。

下面逐一介绍：

<a name="upload-token"></a>

#### Upload Token

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

#### Download Token

Download Token用于[私有资源]()的下载，以及对私有资源作云处理时的请求验证。由下载URL的[token参数]()携带，发送至七牛云存储。

一个Download Token由两个部分组成：`<AccessKey>:<SignedData>`

其中：

1. `<AccessKey>`：用户的AccessKey，用于向七牛云存储表明请求者的身份；
1. `<SignedData>`：签名后的结果，即`urlsafe_base64_encode(hmac_sh1(<SecretKey>, <UnsignedData>))`；

这里的`<UnsignedData>`是下载资源的URL（包含请求的过期时间）：`http://my-bucket.qiniu.com/the-key?e=1373013163`。详见[Download Token参考]()

下面是一个Download Token的示例：

`iN7NgwM31j4-BZacMjPrOQBs34UG1maYCAQmhdCV:vT1lXEttzzPLP4i5T8YVz0AEjCg=`

<a name="access-token"></a>

#### Access Token

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

所谓本地上传，即一个将您的本地资源同步到七牛云端的过程。

如果您需要将手头现有的一批数据同步到七牛云端，那么这种上传方式就是为您准备的。我们已经为您准备好了各种平台上的文件上传工具，不管您的数据量有多么庞大，别担心，这都将会是一个简单而又愉快的过程。

我们提供了下面两种同步工具，可以帮助您轻松地将文件同步到七牛云端：

| 名称 | 使用   | 适用平台   | 说明 |
|-----|--------|----------------|----|
| [qrsync](/tools/qrsync.html) | 命令行 | Linux,Windows,MacOSX,FreeBSD | 手动同步文件/文件夹到七牛云存储 |
| [qiniu-autosync](/tools/qiniu-autosync.html) | 命令行 | Linux | 自动同步指定文件夹内的新增或改动文件 |

当然，本地上传工具只是我们提供给您所有工具的一部分，更多工具，您可以到[这里](tools/index.html)查看。

### 客户端直传

<a name="callback-upload"></a>
如果您的业务需要将资源通过您的网站(Web)或者移动应用(App)等终端上传到七牛云端，那么这种上传方式就是为您设计的。

我们精心设计了两种上传模型，分别适用于不同的业务场景。

#### 普通上传

第一种为普通上传模型，典型的上传流程为：

![普通上传](api/img/normal_upload.png)

流程简述：

1. App-Client 向 App-Server 请求上传文件
2. App-Server 使用 Qiniu-SDK 生成上传授权凭证（UploadToken），并颁发给 App-Client
3. App-Client 取得上传授权许可（UploadToken）后，使用 Qiniu-Client-SDK 直传文件到最近的存储节点
4. 文件上传成功后，Qiniu 返回给 App-Client 上传结果（可包含相应的文件信息）
5. App-Client 将文件上传结果及相关信息汇报给 App-Server，App-Server 可写表做记录等操作

在普通上传模型中，一个完整的上传动作除了实际传输文件到七牛云端之外，所有的业务逻辑都由业务方自由实现，因此这种模型适用于业务终端(App-Client)和业务服务器(App-Server)之间有着较为复杂的自定义业务逻辑的场景中。

如果您的业务逻辑符合这种模型，请参考[这里](api/put.html)查看更多信息。


#### 高级上传

除了第一种普通上传模型之外，我们还提供另一种相对高级的上传模型，在这种模型中，我们七牛云端的服务器不单会以一个文件传输的接受者，还会参与到其余的业务逻辑中去，以此来帮助您简化业务逻辑的实现。

高级上传中主要分为两种模型：

1. 一类是重定向模型，可以让我们的服务器在文件上传结束后通知业务客户端（App-Client）进行301跳转操作。
2. 另一类是回调模型，可以让我们的服务器在文件上传结束后回调一个指定的URL以通知业务服务器（App-Server）。

##### 重定向模型（Redirect）

<a name="download-models"></a>

重定向模型和普通上传模型很相似，只是七牛服务器在上传成功后会返回给上传者一个301跳转，使其跳转到指定的页面。

一个重定向模型典型的上传流程为：

![高级上传(带重定向)](api/img/redirect_upload.png)

流程简述：

1. App-Client 向 App-Server 请求上传文件
2. App-Server 使用 Qiniu-SDK 生成上传授权凭证（UploadToken），并颁发给 App-Client
3. App-Client 取得上传授权许可（UploadToken）后，使用 Qiniu-Client-SDK 直传文件到最近的存储节点
4. 文件上传成功后，Qiniu 将返回状态码为 301 的重定向HTTP Response 给上传者App-Client（可包含相应的文件信息）
5. App-Client 访问跳转到重定向页面。


##### 回调模型（Callback）

<a name="redirect-upload"></a>

一个回调模型典型的上传流程为：

![高级上传(带回调)](api/img/callback_upload.png)

流程简述：

1. App-Client 向 App-Server 请求上传文件
2. App-Server 使用 Qiniu-SDK 生成上传授权凭证（UploadToken），并颁发给 App-Client
3. App-Client 取得上传授权许可（UploadToken）后，使用 Qiniu-Client-SDK 直传文件到最近的存储节点
4. 文件上传成功后，Qiniu 以 HTTP POST 方式告知 App-Server 上传结果（可包含相应的文件信息）
5. App-Server 可写表做记录等操作，然后经 Qiniu 中转返回给 App-Client 它想要的信息
6. Qiniu 作为代理，原封不动地将回调 App-Server 的返回结果回传给 App-Client


回调模型相对于普通上传更为高级，体现在以下几方面:

1. App-Client 无需向 App-Server 发送通知，全部统一由 Qiniu 发送 Callback，当存在多种终端（比如Web/iOS/Android）都需要上传文件时，每个终端不需要各自处理 Callback 业务逻辑。
2. Callback 环节加速，七牛云存储的就近节点能以比 App-Client 更优异的网络回调 App-Server。
3. 只要文件上传成功，App-Server 必然知情。即使 App-Server 回调失败，App-Client 还是会得到完整的回调数据，可自定义策略进行异步处理。

在这种上传模型中，七牛云端的服务器不只是作为文件传输的接受和存储者，同时也参与到了其余的业务逻辑中，为您的业务服务器(App-Server)和业务客户端(App-Client)简化了业务逻辑的实现。同时，利用我们服务器端的网络优势，可以缩短整个流程的完成时间，并大大提高一次上传流程的成功率。

如果您的业务逻辑符合这种模型，请参考[这里](api/put.html)查看更多信息。


### 下载模型

<a name="public-download"></a>

### 公有资源下载

公有资源是放在`公有bucket`上面的资源，可以在[portal.qiniu.com]()上面对应的`bucket`里设置。  
当客户的资源可以在互联网上面公开，例如分享的图片，提供下载的资源等，可以将它们放在`公有的bucket`内。   

公有资源下载时，客户仅需要访问资源对应的`url`即可。`url`的格式为

	http://<domain>/<key>

其中 `<domain>` 为 `<bucket>.qiniudn.com` 或者`自定义绑定的域名`。比如：

	http://shars.qiniudn.com/pub_download.png

**流程**

![pub_download.png](http://shars.qiniudn.com/pub_download.png)

流程简述：

1. App-Client 访问文件 URL 请求下载资源
2. Qiniu-Cloud-Storage 响应 App-Client, 命令距离 App-Client 物理距离最近的 IO 节点输出文件内容

<a name="private-download"></a>

### 私有资源下载

公私有资源是放在`私有bucket`上面的资源，可以在[portal.qiniu.com]()上面对应的`bucket`里设置。  
当客户的资源有一定私密性，只有特定用户才可以访问。例如网盘用户不分享的文件，个人笔记本里的日志等，可以将它们放在`私有的bucket`内。

首先 App-Client 向 App-Server 申请访问资源。App-Server 根据[download token加密算法](#)生成 `download token`颁发给 App-Client。
App-Client 根据得到的授权就可以访问对应的资源。

**流程**

![src_download.png](http://shars.qiniudn.com/src_download.png)

流程简述：

1. App-Client 向 App-Server 请求下载授权
2. App-Server 根据 [downloadToken 签名算法](#) 生成 `downloadToken`, 并颁发给 App-Client
3. App-Client 拿到 `downloadToken` 后，向 Qiniu-Cloud-Storage 请求下载文件
4. Qiniu-Cloud-Storage 在校验 `downloadToken` 成功后，输出文件内容。如果校验失败，返错误信息（401 bad token）

<a name="rs-manage"></a>

### 资源管理

<a name="fop"></a>

### 云处理
