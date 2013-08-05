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
  - [普通客户端直传](#normal-upload)
  - [重定向上传](#redirect-upload)
  - [回调上传](#callback-upload)
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
1. `<UnsignedData>`：待签名数据块。将一组JSON格式的上传参数进行URL安全的Base64编码，获得的字符串即为`<UnsignedData>`。

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

所谓本地上传，是将用户的本地资源同步到七牛云存储的过程。存放在用户服务器中的数据文件，或者用户桌面计算机中的存档文件等等，都可以通过这种方式，方便地同步到七牛云存储

七牛云存储提供了下面两种同步工具，可以帮助您轻松地将文件同步到七牛云存储：

| 名称 | 使用   | 适用平台   | 说明 |
|-----|--------|----------------|----|
| [qrsync](/tools/qrsync.html) | 命令行 | Linux,Windows,MacOSX,FreeBSD | 手动同步文件/文件夹到七牛云存储 |
| [qiniu-autosync](/tools/qiniu-autosync.html) | 命令行 | Linux | 自动同步指定文件夹内的新增或改动文件 |

当然，本地上传工具只是七牛云存储提供给您所有工具的一部分，更多工具，您可以到[这里](tools/index.html)查看。

<a name="normal-upload"></a>

### 普通客户端直传

更多的用户从事网络应用、网站、移动应用等产品的开发，需要从App-Client上传文件。传统的方式是从App-Client将文件上传至App-Server，再由App-Server将文件传送到云存储服务。这种模式会增加用户App-Server的带宽压力，和服务器压力。同时，还使得用户不得不承担文件的入和出两份流量，增加用户的成本。

为消除这些问题，七牛云存储提供了从App-Client直接上传文件的模式。出于用户资源安全的考略，访问的签名需要在用户所控制的App-Server中进行。因此，需要由App-Server向App-Client颁发相应的访问授权。App-Client获得访问请求和授权后，向七牛云存储上传文件。

典型的上传流程如下图所示：

![普通上传](api/img/normal_upload.png)

流程简述：

1. App-Client向App-Server 请求上传文件
2. App-Server使用相应的[算法](#upload-token)或者使用七牛提供的 SDK 生成上传凭证（UploadToken），并颁发给App-Client
3. App-Client 取得上传许可（UploadToken）后，使用七牛提供的 SDK 或者直接使用[上传 API](api/put.html) 直传文件到最近的存储节点
4. 文件上传成功后，Qiniu-Cloud-Storage 返回给 App-Client 上传结果（可包含相应的文件信息）
5. App-Client 将文件上传结果及相关信息汇报给 App-Server，App-Server 可据此执行后续野无逻辑。

普通上传模型用途广泛。几乎所有的网络应用、网站、在线服务等等，都可以使用普通上传提升服务品质，优化资源配置。


<a name="redirect-upload"></a>

### 重定向上传

重定向上传是普通客户端直传的一个扩展。允许用户在上传请求中设定一个url（[returnUrl]()），七牛云存储完成上传之后，将反馈[HTTP 301]()，引导客户端执行自动跳转。这种上传模型主要适用于建立网站的用户。

一个重定向模型典型的上传流程如下图所示：

![高级上传(带重定向)](api/img/redirect_upload.png)

流程简述：

1. App-Client 向 App-Server 请求上传文件；
2. App-Server 使用相应的[算法](#upload-token)或者使用七牛提供的 SDK 生成上传凭证（UploadToken），并颁发给 App-Client；
3. App-Client 取得上传许可（UploadToken）后，使用七牛提供的 SDK 或者直接使用[上传 API](api/put.html) 直传文件到最近的存储节点；
4. 文件上传成功后，Qiniu-Cloud-Storage 将返回状态码为 301 的重定向HTTP Response 给上传者App-Client（可包含相应的文件信息）；
5. App-Client 访问跳转到重定向页面。


<a name="callback-upload"></a>

### 回调上传

回调上传是普通客户端直传的另一种扩展。用户在上传请求中指定回调URL（[callbackUrl]()），七牛云存储在上传完成后以特定的方式调用用户所提供的URL，将上传的结果传送至用户的App-Server。用户可以在收到回调结果后执行相关的业务逻辑。此后，用户可以将一些数据安置在响应回调的HTTP Response中，反回给七牛云存储。七牛云存储收到响应后，将其中的数据放在HTTP Response中，传递给App-
Client。

这种上传模型对于那些需要在App-Client和App-Server之间进行复杂交互的用户非常有用。比如，移动应用的客户终端往往没有很好的网络环境，频繁地进行客户端-服务器之间的交互严重影响使用体验。而使用回调上传模型，客户终端在上传过程中只需一次HTTP访问，即可完成包括服务端通知在内的多个操作。

一个回调上传模型典型的上传流程为：

![高级上传(带回调)](api/img/callback_upload.png)

流程简述：

1. App-Client 向 App-Server 请求上传文件
2. App-Server 使用相应的[算法](#upload-token)或者使用七牛提供的 SDK 生成上传凭证（UploadToken），并颁发给 App-Client
3. App-Client 取得上传许可（UploadToken）后，使用七牛提供的 SDK 或者直接使用[上传 API](api/put.html) 直传文件到最近的存储节点
4. 文件上传成功后，Qiniu-Cloud-Storage 以 HTTP POST 方式告知 App-Server 上传结果（可包含相应的文件信息）
5. App-Server 处理完回调的请求后返回相应的结果信息，经 Qiniu-Cloud-Storage 中转返回给 App-Client
6. Qiniu-Cloud-Storage 作为代理，原封不动地将回调 App-Server 的返回结果回传给 App-Client


回调模型相对于普通上传更为高级，体现在以下几方面:

1. App-Client 无需向 App-Server 发送通知，全部统一由 Qiniu 发送 Callback，当存在多种终端（比如Web/iOS/Android）都需要上传文件时，每个终端不需要各自处理 Callback 业务逻辑。
2. Callback 环节加速，七牛云存储的就近节点能以比 App-Client 更优异的网络回调 App-Server。
3. 只要文件上传成功，App-Server 必然知情。即使 App-Server 回调失败，App-Client 还是会得到完整的回调数据，可自定义策略进行异步处理。

在这种上传模型中，七牛云存储的服务器不只是作为文件传输的接受和存储者，同时也参与到了其余的业务逻辑中，为您的业务服务器(App-Server)和业务客户端(App-Client)简化了业务逻辑的实现。同时，利用七牛云存储服务器端的网络优势，可以缩短整个流程的完成时间，并大大提高一次上传流程的成功率。


<a name="download-models"></a>

## 下载模型

七牛云存储的用户可以将一个空间设置为[公开]()和[私有]()两种保护级别。当空间被设置为公有，任何人都可以无需授权访问此空间的内容。而私有空间的内容则无法在没有用户授权的情况下访问。于是便形成两种不同的下载模式：公开资源下载和私有资源下载。

**注意：** 对于设置[原图保护]()的空间，尽管同时设置成公开，空间内的原图也必须使用私有资源下载的方式访问。

<a name="public-download"></a>

### 公开资源下载

公开资源资源下载，非常简单，就如同下载互联网上任何公开文件和数据那样，客户仅需要构造出资源的`URL`，然后发出HTTP请求即可。

七牛云存储资源的`URL`的格式为：

```
	http://<domain>/<key>
```

其中 `<domain>` 是资源下载的域名，有两种形式：

1. 七牛二级域名： `<bucket>.qiniudn.com`。以用户的空间名为二级域名，构成资源下载的域名。比如：my-bucket.qiniudn.com；
1. 自定义域名： 用户自有的域名，通过[域名绑定]()方式绑定到用户空间。比如：files.my-domain.com。

`<key>` 是需要下载的[资源名(Key)]()。

构造出资源下载的URL，便可以通过任何HTTP客户端，向七牛云存储发出请求，进行下载。

下图展示了公有资源下载的基本流程：

![pub_download.png](http://shars.qiniudn.com/pub_download.png)

流程简述：

1. App-Client 访问文件 URL 请求下载资源
2. Qiniu-Cloud-Storage 响应 App-Client, 指令距离 App-Client 物理距离最近的 IO 节点输出文件内容。


<a name="private-download"></a>

### 私有资源下载

私有资源的下载需要资源的拥有者（七牛云存储用户）向资源的需求者授权，因此需要对下载请求进行验证签名。

私有资源下载所用的URL是在公开资源下载使用的URL基础上增加[请求凭证(Token)]()部分：

```
  http://<domain>/<key>?e=<deadline>&token=<token>
```

其中：

- `<deadline>` 是URL的过期时间。通过设置这个参数，确保URL在一定时间以后失效，以防资源无限访问。
- `<token>` 是[下载凭证](#download-token)。对整个URL进行HMAC-SHA1签名，获得的摘要。七牛云存储将采用相同的签名算法验证该URL是否具备合法的资源访问权限。

下图是司有资源下载的基本流程：

![src_download.png](http://shars.qiniudn.com/src_download.png)

流程简述：

1. App-Client 向 App-Server 请求下载授权
2. App-Server 根据 [下载凭证签名算法](#download-token) 生成 `downloadToken`, 并颁发给 App-Client
3. App-Client 拿到 `downloadToken` 后，向 Qiniu-Cloud-Storage 请求下载文件
4. Qiniu-Cloud-Storage 在校验 `downloadToken` 成功后，输出文件内容。如果校验失败，返错误信息（401 bad token）


<a name="rs-manage"></a>

## 资源管理


<a name="fop"></a>

## 云处理
