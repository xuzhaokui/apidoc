---
layout: default
title: "上传方式"
---

- [上传流程](#workflow)
    - [Local - 本地上传](#local-upload) 
    - [UGC - 终端用户加速直传](#ugc-upload) 
      - [上传模式1——普通上传](#upload-without-callback)
      - [上传模式2——高级上传（带回调）](#upload-with-callback)
- [上传文件](#upload)
    - [接口 - API](#upload-api)
    - [凭证 - uploadToken](#uploadToken)
        - [算法](#uploadToken-algorithm)
        - [参数](#uploadToken-args)
        - [使用上传模型1，App-Client 接收来自 Qiniu-Cloud-Storage 的 Response Body](#uploadToken-returnBody)
        - [使用上传模型2，App-Client 接收来自 App-Server 的 Response Body](#upload-with-callback-appserver)
        - [音视频上传预转 - asyncOps](#uploadToken-asyncOps)
        - [样例代码](#uploadToken-examples)
- [附录](#dictionary)
    - [魔法变量 - MagicVariables](#MagicVariables)
    - [自定义变量 - xVariables](#xVariables)
    - [错误码](#error-code)


- [资源上传基础](#upload-basic)
    - [Html Form Post](#html-form-post)
    - [Multipart](#multipart)
- [参数介绍](#parameters)
	- [UploadToken](#uploadtoken)
- [七牛的响应](#response)
	- [普通上传](#ordinary-upload)
	- [高级上传（带回调）](#advanced-upload)


<a name="upload-basic"></a>

# 资源上传基础

七牛云存储用于资源上传的域名是： `up.qiniu.com` 。七牛云存储特有的上传加速系统会将此域名解析到与上传客户端之间链路最好的数据中心，保证用户可以获得最佳的上传效果。

七牛云存储的资源上传基于HTTP协议，通过HTTP POST指令实现。上传指令参数采用 `multipart/form-data` 数据格式组织。采用该格式使得用户可以使用HTML Form向七牛云存储直接上传文件。

在资源上传的基础上，七牛云存储还提供一组与上传有关的附加功能，包括：

1. 用户定义返回值；
1. 客户端重定向；
1. 回调；
1. 异步云处理；

这些功能极大地方便了用户对上传资源的处理，使得用户可以在一次资源操作中完成较为复杂的业务处理。

<a name="upload-proto"></a>

## 上传协议

七牛云存储的资源上传使用[HTML Form POST](http://www.w3.org/TR/html4/interact/forms.html)，采用[multipart/form-data](http://www.w3.org/TR/html4/interact/forms.html#h-17.13.4.2)数据格式。

实际操作中，有两种方式。一种是在Web页面中上传文件，另一种是在非HTML页面客户端上传。

<a name="html-form-post"></a>

### 在HTML页面中上传资源

当用户的上传客户端是HTML页面时，可以直接使用 `form` 元素构造和发起资源上传请求。一个典型的上传HTML片段如下：

```
<form method="post" action="http://up.qiniu.com/" enctype="multipart/form-data">
  <input name="key" type="hidden" value="<resource key>">
  <input name="x:<custom_field_name>" type="hidden" value="<custom value>">
  <input name="token" type="hidden" value="<token>">
  <input name="file" type="file" />
  ...
</form>
```

资源上传所需的参数在 `input` 元素中设置。这些参数详见[资源上传参数](#parameters)小节中详细介绍。需要说明的是，名称为 `x:<custom_field_name>` （即[xVariable]()）的参数可以不止一个，用户可以根据需要填写任意多个，满足业务逻辑的需要。

当form的[submission](http://www.w3.org/TR/html4/interact/forms.html#h-17.13)被触发时，浏览器会将所有 `input` 的内容打包成 `multipart/form-data` 数据格式，发送至七牛云存储，完成一次上传操作。

<a name="multipart"></a>

### 非HTML客户端上传

如果上传客户端不是HTML页面，那么用户就需要模仿浏览器，将参数组织成 `multipart/form-data` 格式。该格式的大体形态如下：

```
POST http://up.qbox.me/
Content-Type: multipart/form-data; boundary=<Boundary>
<Boundary>

Content-Disposition: form-data; name="key"
<FileID>
<Boundary>

Content-Disposition: form-data; name="x:custom_field_name"
<SomeVal>
<Boundary>

Content-Disposition: form-data; name="token"
<UploadToken>
<Boundary>

...

Content-Disposition: form-data; name="file"; filename="<FileName>"
Content-Type: <MimeType>

<FileContent>
```

用户可以手工地构造出 `multipart/form-data` 数据。但更好地方式是使用七牛云存储提供的多种[SDK]()，简单快速地完成上传操作。此外，也有很多HTTP客户端组件、库和工具可以帮助用户快速构造 `multipart/form-data` 数据，在用户需要直接访问七牛云存储API的时候使用。

<a name="parameters"></a>

## 上传参数

上传参数包括两类，一类是服务参数，有三个，分别是：key、token和file。三者具体的含义参考以下表格。另一类是用户[自定义变量]()，即xVariable。用户可以通过 `x:<custom_field_name>` 参数将其传递到七牛云存储。七牛云存储根据[callbackUrl]()的设置，构造出回调结果。

名称        | 类型   | 必须 | 说明
------------|--------|------|-------------------------------------
token       | string | 是   | 上传授权凭证 - [UploadToken](#uploadtoken)
file        | file   | 是   | 文件本身
key         | string | 否   | 标识文件的索引，所在的存储空间内唯一。key可包含`/`，但不以`/`开头。若不指定 key，七牛云存储将使用文件的 etag（即上传成功后返回的hash值）作为key，并在返回结果中传递给客户端。
x:<custom_field_name> | string | 否 | 自定义变量，必须以 `x:` 开头命名，不限个数。里面的内容将在 `callbackBody` 参数中的 `$(x:custom_field_name)` 求值时使用。


<a name="put-policy"></a>

## 上传策略（PutPolicy）

上传策略是资源上传时的一组配置设定。通过这组配置信息，七牛云存储可以了解用户上传的需求：它将上传什么资源，上传到哪个空间，是否需要回调，还是使用重定向，是否需要设置反馈信息的内容，以及请求的失效时间等等。

上传策略同时还参与请求验证。实际上，[上传凭证（Upload Token）]()就是上传策略的加密结果。通过对PutPolicy的非可逆加密，可以确保用户对某个资源的上传请求是完全受到验证的。

上传策略的具体构成如下：

```
{
    scope: <Bucket string>,
    deadline: <UnixTimestamp int64>,
    endUser: <EndUserId string>,
    returnUrl: <RedirectURL string>,
    returnBody: <ResponseBodyForAppClient string>,
    callbackBody: <RequestBodyForAppServer string>
    callbackUrl: <RequestUrlForAppServer string>,
    asyncOps: <asyncProcessCmds string>
}
```

**参数详解** ：

 字段名       | 必须 | 说明
--------------|------|-----------------------------------------------------------------------
 scope        | 是   | 用于指定文件所上传的目标Bucket和Key。格式为：<bucket name>[:<key>]。若只指定Bucket名，表示文件上传至该Bucket。若同时指定了Bucket和Key（<bucket name>:<key>），表示上传文件限制在指定的Key上。两种形式的差别在于，前者是“新增”操作：如果所上传文件的Key在Bucket中已存在，上传操作将失败。而后者则是“新增或覆盖”操作：如果Key在Bucket中已经存在，将会被覆盖；如不存在，则将文件新增至Bucket中。
 deadline     | 是   | 定义上传请求的失效时间，[Unix时间戳](http://en.wikipedia.org/wiki/Unix_time)，单位为秒。
 endUser      | 否   | 给上传的文件添加唯一属主标识，特殊场景下非常有用，比如根据App-Client标识给图片或视频打水印
 returnUrl    | 否   | 设置用于浏览器端文件上传成功后，浏览器执行301跳转的URL，一般为`HTML Form`上传时使用。文件上传成功后会跳转到`returnUrl?query_string`, `query_string`会包含 `returnBody` 内容。
 returnBody   | 否   | 文件上传成功后，自定义从七牛云存储最终返回給终端 App-Client 的数据。支持 [魔法变量](#)。
 callbackBody | 否   | 文件上传成功后，七牛云存储向 App-Server 发送POST请求的数据。支持 [魔法变量](#) 和 [自定义变量](#)。
 callbackUrl  | 否   | 文件上传成功后，七牛云存储向 App-Server 发送POST请求的URL，必须是公网上可以正常进行POST请求并能响应 HTTP Status 200 OK 的有效 URL 
 asyncOps     | 否   | 指定文件（图片/音频/视频）上传成功后异步地执行指定的预转操作。每个预转指令是一个API规格字符串，多个预转指令可以使用分号`;`隔开。详细操作见[fop](#)

**注意**

- `callbackUrl` 与 `returnUrl` 不可同时指定，两者只可选其一。
- `callbackBody` 与 `returnBody` 不可同时指定，两者只可选其一。

<a name="upload-token"></a>

## 上传凭证（Upload Token）

上传凭证是七牛云存储用于验证上传请求合法性的机制。用户通过上传凭证授权客户端，使其具备访问指定资源的能力。

上传凭证算法如下：

1. 构造[上传策略](#put-policy)。用户根据业务需求，确定上传策略的要素，构造出具体的上传策略。比如，有用户需要向空间 `my-bucket` 上传一个名为 `sunflower.jpg` 的图片，有效期是到 `2015-12-31 00:00:00`，并且希望得到图片的名称、大小、宽、高和校验值。那么相应的上传策略的字段分别为：

```
    scope = "my-bucket:sunflower.jpg"
    deadline = 1451491200
    returnUrl = '{
      "name": $(fname),
      "size": $(fsize),
      "w": $(imageInfo.width),
      "h": $(imageInfo.height),
      "hash": $(etag),
    }'
```

1. 将上传策略序列化成为json格式。用于可以使用各种语言的json库，也可以手工地拼接字符串。序列化后，可以得到：

```
    put_policy = '{"scope":"my-bucket:sunflower.jpg","deadline":1451491200,"returnUrl":"{\"name\": $(fname),\"size\": $(fsize),\"w\": $(imageInfo.width),\"h\": $(imageInfo.height),\"hash\": $(etag),}"}'
```

1. 对json序列化后的上传策略进行[URL安全的Base64编码](http://en.wikipedia.org/wiki/Base64)：

```
    encoded = urlsafe_base64_encode(put_policy)
```

  得到

```
    "eyJzY29wZSI6Im15LWJ1Y2tldDpzdW5mbG93ZXIuanBnIiwiZGVhZGxpbmUiOjE0NTE0OTEyMDAsInJldHVyblVybCI6IntcIm5hbWVcIjogJChmbmFtZSksXCJzaXplXCI6ICQoZnNpemUpLFwid1wiOiAkKGltYWdlSW5mby53aWR0aCksXCJoXCI6ICQoaW1hZ2VJbmZvLmhlaWdodCksXCJoYXNoXCI6ICQoZXRhZyksfSJ9"
```

1. 用SecretKey对编码后的上传策略进行HMAC-SHA1加密，并且做URL安全的Base64编码：

```
    signature = hmac_sha1(SecretKey, encoded)
    encode_signed = urlsafe_base64_encode(signature)
```

  假设用户的 `SecretKey="Yx0hNBifQ5V5SqLUkzPkjyy0pbYJpav9CH1QzkG0"` 加密后的结果是：

```
    "5Cr3Nrw0qkyYKfQicd_ejAdIrfs="
```

1. 最后，将 `AccessKey`、`encode_signed` 和 `encoded` 用 “:” 连接起来：

```
    upload_token = AccessKey + ":" + encode_signed + ":" + encoded
```
  假设用户的 `AccessKey="j6XaEDm5DwWvn0H9TTJs9MugjunHK8Cwo3luCglo"` 。最后得到的上传凭证为：

```
    j6XaEDm5DwWvn0H9TTJs9MugjunHK8Cwo3luCglo:5Cr3Nrw0qkyYKfQicd_ejAdIrfs=:eyJzY29wZSI6Im15LWJ1Y2tldDpzdW5mbG93ZXIuanBnIiwiZGVhZGxpbmUiOjE0NTE0OTEyMDAsInJldHVyblVybCI6IntcIm5hbWVcIjogJChmbmFtZSksXCJzaXplXCI6ICQoZnNpemUpLFwid1wiOiAkKGltYWdlSW5mby53aWR0aCksXCJoXCI6ICQoaW1hZ2VJbmZvLmhlaWdodCksXCJoYXNoXCI6ICQoZXRhZyksfSJ9
```


<a name="response"></a>

## 上传请求的反馈

上传请求的反馈是标准的 [HTTP Response](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html)，遵循 HTTP/1.1 标准[Status Code](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html#sec6.1.1)。 Response Body 则包含七牛云存储的反馈信息。反馈信息以[json](http://www.json.org/)格式组织。

<a name="basic-resp"></a>

### 基本反馈

当用户的资源上传请求得到正确执行，七牛云存储会反馈成功，Status Code 200。Resopnse Body中携带两个值：

- `name`：已成功上传的资源名，即Key；
- `hash`：衣裳穿资源的校验码，供用户核对使用。

以下是一个典型的上传成功反馈：

```
  HTTP/1.1 200 OK
  Content-Type: application/json
  Cache-Control: no-store
    ...
  Response Body: {
    "name": "gogopher.jpg",
    "hash": "Fh8xVqod2MQ1mocfI4S4KpRL6D98",
  }
```

当用户的资源上传请求出现错误，七牛云存储会反馈相应的错误。比如，401代表验证失败。此时，Response Body中携带具体的错误信息。错误信息同样以 json 格式组织，基本形式为：{"error":"<reason>"}

以下是一个典型的上传失败反馈：

```
  HTTP/1.1 400 Bad Request
  Date: Mon, 05 Aug 2013 13:56:34 GMT
  Server: nginx/1.0.14
  Content-Type: application/json
  Access-Control-Allow-Origin: *
  Content-Length: 28
  X-Log: MC;SBD:10;RBD:11;BDT:12;FOPD/400;FOPG:63/400;IO:109/400
  X-Reqid: -RIAAIAI8UjcgRcT
  X-Via: 1.1 jssq179:8080 (Cdn Cache Server V2.0), 1.1 jsyc96:9080 (Cdn Cache Server V2.0)
  Connection: close
  Response Body: {
    "error":"invalid argument"
  }
```

通过这些错误信息，可以使用户了解问题的所在，以便做进一步处理。关于错误信息，详见[错误信息参考]()。

对于反馈信息，建议用户使用日志加以保存，以便分析和查找问题之用，也方便用户得到我们进一步的技术支持。

除错误信息外，七牛云存储的 Response Header 中也携带了一些有用的信息，有助于问题定位。其中主要有：

1. X-Reqid：用户请求的唯一id。通过这个id，七牛云存储可以快速查找到用户请求的相关记录。当用户在使用七牛云存储遇到问题时，可以通过该id，获得更多的信息；
1. X-Log：用户请求的日志缩略信息。通过这些信息，七牛云存储可以大致地获得用户请求的执行情况，帮助用户定位问题。

<a name="return-body"></a>

### Return Body

基本的上传反馈只会包含资源最基本的信息。很多情况下，用户希望得到更多有关资源的信息。用户可以通过七牛云存储提供[云处理]()操作获得这些扩展信息。但需要用户另外发起请求，查询这些资源。为了方便用户的使用，七牛云存储可以在上传请求中直接向用户反馈这些额外的信息。

用户可以通过 `returnBody` 参数指定需要返回的信息，比如资源的大小、类型，图片的尺寸等等。`returenBody` 实际上是一个用户定义的反馈信息模板。下面是一个returnBody的案例：

```
  {
    "foo": "bar",
    "name": $(fname),
    "size": $(fsize),
    "type": $(mimeType),
    "hash": $(etag),
    "w": $(imageInfo.width),
    "h": $(imageInfo.height),
    "color": $(exif.ColorSpace.val)
  }
```

`returnBody` 同真正的返回信息一样，也是json格式。在 `returnBody` 中，用户通过设定所谓[魔法变量（MagicVariable）]()，通知七牛云存储反馈那些信息。“魔法变量”采用 `$(<variable-name>)` 的形式，在反馈信息中占位。七牛云存储会根据变量名，将相应的数据替换“魔法变量”，反馈给用户：

```
  {
    "foo": "bar",
    "name": "gogopher.jpg",
    "size": 214513,
    "type": "image/jpg",
    "hash": "Fh8xVqod2MQ1mocfI4S4KpRL6D98",
    "w": 640,
    "h": 480,
    "color": "sRGB"
  }
```

`returnBody` 在普通客户端直传和重定向上传中都可以使用，但不能在回调上传时使用。回调上传时应该使用callbackBody。实际上，一旦设置了callbackUrl，启用回调上传，`returnBody` 也就不再起作用了。

<a name="callback-body"></a>

### Callback Body

同普通客户端直传和重定向上传一样，用户也可以控制回调中传递到客户回调URL的反馈信息。`callbackBody` 的格式同 `returnBody` 一样。但 `callbackBody` 除了可以使用“魔法变量”外，还可以使用[用户自定义变量]()。当客户端发送上传请求时，携带的 `x:<variable>` 参数将被七牛云存储提取出来，根据 `callbackBody` 中用户的设定，填充到回调反馈信息中。

在用户使用回调上传时，如果文件上传成功，但七牛云存储回调客户服务器失败，七牛云存储会将回调失败的信息连同 `callbackBody` 数据本身返回给App-Client, 用户可根据自己的策略进行相应的处理。


<a name="do-upload"></a>

# 上传资源

<a name="local-upload"></a>

## 本地上传

若您需要上传已存在您电脑或者是服务器上的文件到七牛云存储，可以直接使用七牛提供的上传工具：

| 名称                                         | 使用   | 适用平台                     | 说明                                 |
|----------------------------------------------|--------|------------------------------|--------------------------------------|
| [qrsync](/tools/qrsync.html)                 | 命令行 | Linux,Windows,MacOSX,FreeBSD | 手动同步文件/文件夹到七牛云存储      |
| [qiniu-autosync](/tools/qiniu-autosync.html) | 命令行 | Linux                        | 自动同步指定文件夹内的新增或改动文件 |

除了这类离线的备份以外，用户还可以在自己的程序中向七牛云存储上传文件。但是，进行本地上传的程序必须是运行在用户自己的服务器，或者桌面计算机上的程序。 **本地上传模式不能在客户端中使用。**

![本地上传](img/local-upload.png)

上图展示了本地上传的基本流程。本地上传只是用户程序和七牛云存储之间的交互，具体步骤说明如下：

1. 用户程序构造 `上传策略` 。对于本地上传而言，通常只需要设置 `scope` 和 `deadline` 。如果需要，可以设置 `returnBody` 设定返回信息的内容。如果用户需要对上传文件做进一步的云处理，也可以设置ansycOps参数；
1. 签名，生成Token。构造完成 `上传策略` 后，用户需要对 `上传策略` 加密，生成 `upload token` ；
1. 构造上传请求。用户将 `key` 、 `token` 等参数构造出 `HTTP Form Post` 请求；
1. 用户程序向七牛云存储发出资源上传请求。
1. 七牛云存储向用户程序反馈上传结果。如果上传成功，七牛云存储将反馈 `HTTP 200` ，以及用户设定的反馈信息。如果上传发生错误，七牛云存储将反馈相应的错误代码和信息。


<a name="direct-upload"></a>

## 普通客户端直传

当用户需要开发一个网络应用，那么基本上都会有客户端和服务端。客户端由七牛云存储的用户的用户，即最终用户使用。这些客户端可能是桌面程序、网站、手机应用、Pad应用等等。多数情况下，需要上传文件的是这些客户端。但是，对于七牛云存储而言，无法识别这些客户端的使用者，他们并非七牛云存储的用户。对于七牛云存储而言，那些应用开发者才是用户，是存储资源的所有者。于是，那些客户端不能直接向七牛云存储上传文件，必须要获得应用开发者（也就是七牛用户）的授权。

![普通客户端直传](img/normal-upload.png)

上图展示了普通客户端直传的基本流程。具体步骤说明如下：

1. 应用客户端向应用服务器请求上传文件。通常，应用客户端需要向应用服务器发送 `资源名` ， `空间名` 和 `deadline` 等参数由应用服务器的业务逻辑确定；
1. 应用服务器构造 `上传策略` ；
1. 应用服务器将 `上传策略` 序列化成json格式，对其实施签名算法，得到 `上传凭证` ；
1. 应用服务器将 `上传凭证` 返回给应用客户端；
1. 应用客户端构造完整的上传请求；
1. 应用客户端发送上传请求，启动上传；
1. 七牛云存储执行上传操作，保存资源。完成后反馈用户相应的信息。如果上传失败，七牛云存储将反馈用户具体的失败信息。


<a name="redirect-upload"></a>

## 重定向上传

重定向上传的基本流程同普通客户端直传类似，差异在于当应用服务器构造 `上传策略` 时，设定 `returnUrl` 。之后，七牛云存储一旦完成上传，便会向应用客户端反馈 `HTTP 301` 。应用客户端一旦收到301反馈，便可跳转至 `returnUrl` 所指的位置。

![重定向上传](img/redirect-upload.png)

上图展示了普通客户端直传的基本流程。具体步骤说明如下：

1. 应用客户端向应用服务器请求上传文件。通常，应用客户端需要向应用服务器发送 `资源名` ， `空间名` 和 `deadline` 等参数由应用服务器的业务逻辑确定；
1. 应用服务器构造 `上传策略` 。应用服务器的业务逻辑需要设置 `redirectUrl` 字段；
1. 应用服务器将 `上传策略` 序列化成json格式，对其实施签名算法，得到 `上传凭证` ；
1. 应用服务器将 `上传凭证` 返回给应用客户端；
1. 应用客户端构造完整的上传请求；
1. 应用客户端发送上传请求，启动上传；
1. 七牛云存储执行上传操作，保存资源。如果上传失败，七牛云存储将反馈用户具体的失败信息。如果上传成功，七牛云存储将向应用客户端反馈 `HTTP 301` 。应用客户端会据此执行跳转。

<a name="callback-upload"></a>

## 回调上传

回调上传也是在普通客户端直传上衍生出来的模型。相比普通客户端直传，回调上传增加了七牛云存储回调用户的回调服务器的步骤。

![重定向上传](img/redirect-upload.png)

上图展示了普通客户端直传的基本流程。具体步骤说明如下：

1. 应用客户端向应用服务器请求上传文件。通常，应用客户端需要向应用服务器发送 `资源名` ， `空间名` 和 `deadline` 等参数由应用服务器的业务逻辑确定；
1. 应用服务器构造 `上传策略` 。应用服务器的业务逻辑必须设置 `callbackUrl` 字段，如果需要，还可设置 `callbackBody` 字段，控制回调的反馈信息；
1. 应用服务器将 `上传策略` 序列化成json格式，对其实施签名算法，得到 `上传凭证` ；
1. 应用服务器将 `上传凭证` 返回给应用客户端；
1. 应用客户端构造完整的上传请求；
1. 应用客户端发送上传请求，启动上传；
1. 七牛云存储执行上传操作，保存资源。如果上传失败，七牛云存储将反馈用户具体的失败信息。如果上传成功，七牛云存储将向应用服务器发起回调；
1. 回调服务器反馈回调执行结果。回调服务器接收到回调数据后，可以执行后续的业务逻辑。而且，用户可以利用此机会，将一些处理结果反馈给七牛云存储，委托七牛云存储传递给应用客户端。如果回调执行失败，七牛云存储会将错误信息反馈应用客户端；
1. 七牛云存储完成回调后，将获得的回调返回信息，原封不动地反馈给应用客户端。


========================== 未完成分割线 ==========================


**注意**

- `Flags` 数据必须为标准的 [JSON](http://json.org/) 字符串
- `Flags` 各字段里边的尖括号“`<Variable Type>`”内容表示要被替换掉的“变量”，“变量”的数据类型已在括号内指定
- `urlsafe_base64_encode(string)` 函数按照标准的 [RFC 4648](http://www.ietf.org/rfc/rfc4648.txt) 实现，开发者可以参考 <https://github.com/qiniu> 上各SDK的样例代码。
- `AccessKey:EncodedSign:EncodedFlags` 这里的冒号是字符串，仅作为连接分隔符使用，最终连接组合的 UploadToken 也是一个字符串（String）。
- `callbackUrl` 与 `returnUrl` 不可同时指定，两者只可选其一。
- `callbackBody` 与 `returnBody` 不可同时指定，两者只可选其一。


<a name="uploadToken-args"></a>

### 参数

uploadToken 参数详解：

 字段名       | 必须 | 说明
--------------|------|-----------------------------------------------------------------------
 scope        | 是   | 一般指文件要上传到的目标存储空间（Bucket）。若为"Bucket"，表示限定只能传到该Bucket（仅限于新增文件）；若为"Bucket:Key"，表示限定特定的文件，可修改该文件。
 deadline     | 否   | 定义 uploadToken 的失效时间，Unix时间戳，精确到秒，缺省为 3600 秒
 endUser      | 否   | 给上传的文件添加唯一属主标识，特殊场景下非常有用，比如根据终端用户标识给图片或视频打水印
 returnUrl    | 否   | 设置用于浏览器端文件上传成功后，浏览器执行301跳转的URL，一般为 HTML Form 上传时使用。文件上传成功后会跳转到 returnUrl?query_string, query_string 会包含 returnBody 内容。returnUrl 不可与 callbackUrl 同时使用。
 returnBody   | 否   | 文件上传成功后，自定义从 Qiniu-Cloud-Server 最终返回給终端 App-Client 的数据。支持 [魔法变量](#MagicVariables)，不可与 callbackBody 同时使用。
 callbackBody | 否   | 文件上传成功后，Qiniu-Cloud-Server 向 App-Server 发送POST请求的数据。支持 [魔法变量](#MagicVariables) 和 [自定义变量](#xVariables)，不可与 returnBody 同时使用。
 callbackUrl  | 否   | 文件上传成功后，Qiniu-Cloud-Server 向 App-Server 发送POST请求的URL，必须是公网上可以正常进行POST请求并能响应 HTTP Status 200 OK 的有效 URL 
 asyncOps     | 否   | 指定文件（图片/音频/视频）上传成功后异步地执行指定的预转操作。每个预转指令是一个API规格字符串，多个预转指令可以使用分号“;”隔开


<a name="uploadToken-returnBody"></a>

### 使用上传模型1，App-Client 接收来自 Qiniu-Cloud-Storage 的 Response Body

如果开发者使用上传模型1，App-Client 上传一张图片到 Qiniu-Cloud-Storage 后，App-Client 想知道该图片的一些信息比如 Etag, EXIF 等信息，那么此时即可在 uploadToken 中使用 **`returnBody`** 参数。 

App-Client 想求值得到的这些 Etag, EXIF 等信息我们称之为魔法变量（[MagicVariables](#MagicVariables)）。

returnBody 赋值可以把 魔法变量（[MagicVariables](#MagicVariables)）的求值结果以 `Content-Type: application/json` 形式返回給 App-Client。

一个典型的包含 MagicVariables 的 returnBody 字段声明如下（returnBody 必须是一个JSON字符串）：

    Flags["returnBody"] = `{
        "foo": "bar",
        "name": $(fname),
        "size": $(fsize),
        "type": $(mimeType),
        "hash": $(etag),
        "w": $(imageInfo.width),
        "h": $(imageInfo.height),
        "color": $(exif.ColorSpace.val)
    }`

假使如上，当一个用户在 iOS 端用包含该 returnBody 字段的 uploadToken 成功上传一张图片，那么该 iOS 端程序将收到如下一段 HTTP Response 应答：

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: no-store
    Response Body: {
        "foo": "bar",
        "name": "gogopher.jpg",
        "size": 214513,
        "type": "image/jpg",
        "hash": "Fh8xVqod2MQ1mocfI4S4KpRL6D98",
        "w": 640,
        "h": 480,
        "color": "sRGB"
    }

可用的魔法变量列表参考：[MagicVariables](#MagicVariables)

### HTML Form 上传后跳转

若在 uploadToken 中指定了 `returnUrl` 和 `returnBody` 选项，且文件上传成功，Qiniu-Cloud-Storage 会返回如下应答：

    HTTP/1.1 301 Moved Permanently
    Location: returnUrl?upload_ret={returnBodyEncoded}

即跳转到指定的 `returnUrl` 并附带上经过 `urlsafe_base64_encode()` 编码过的 `returnBody` 。 `urlsafe_base64_encode(string)` 函数按照标准的 [RFC 4648](http://www.ietf.org/rfc/rfc4648.txt) 实现，开发者可以参考 <https://github.com/qiniu> 上各SDK的样例代码。可以通过逆向还原 `returnBody` 。

若上传失败，且上传失败的原因不是由于 uploadToken 无效造成的，Qiniu-Cloud-Storage 会返回如下应答：

    HTTP/1.1 301 Moved Permanently
    Location: returnUrl?code={error_code}&error={error_message}


<a name="upload-with-callback-appserver"></a>

### 使用上传模型2，App-Client 接收来自 App-Server 的 Response Body

如果开发者使用了上传模型2，在 uploadToken 中指定了 `callbackUrl` 和 `callbackBody` 选项。App-Client 使用该 uploadToken 将文件上传成功后，Qiniu-Cloud-Storage 会向指定的 `callbackUrl` 以 HTTP POST 方式将 `callbackBody` 的值以 `application/x-www-form-urlencoded` 的形式发送给 App-Server。App-Server 接收请求后，返回 `Content-Type: "application/json"` 形式的 Response Body, 该段 JSON 数据会原封不动地经由 Qiniu-Cloud-Storage 返回给 App-Client 。

**callbackUrl** 必须是公网上可以公开访问的 URL  

**callbackUrl** 若指定，**callbackBody** 也必须指定，且两者的值都不能为空  

**callbackBody** 必须是 `a=1&b=2&c=3` 这种形式的字符串。当包含 [魔法变量](#MagicVariables) 时，可以是这样一种形式：`a=1&key=$(etag)&size=$(fsize)&uid=$(endUser)` 。当包含 [自定义变量](#xVariables) 时，可以是这样一种形式：`test=$(x:test)&key=$(etag)&size=$(fsize)&uid=$(endUser)`，其中 `x:test` 是一个自定义变量。自定义变量命名必须以 `x:` 开头，且在 `multipart/form-data` 上传流中存在。

Qiniu-Cloud-Storage 回调 App-Server 成功后，App-Server 必须返回如下格式的 Response:

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: no-store
    Response Body: {
        "foo": "bar",
        "name": "gogopher.jpg",
        "size": 214513,
        "type": "image/jpg",
        "w": 640,
        "h": 480
    }

其中，Response Body 部分 App-Server 随意，只需为标准的 [JSON](http://json.org) 格式即可。

参考：

- [魔法变量 - MagicVariables](#MagicVariables)
- [自定义变量 - xVariables](#xVariables)

### callback 失败处理

如果文件上传成功，但是 Qiniu-Cloud-Storage 回调 App-Server 失败，Qiniu-Cloud-Storage 会将回调失败的信息连同 `callbackBody` 数据本身返回给 App-Client, App-Client 可选自定义策略进行相应的处理。

<a name="uploadToken-asyncOps"></a>

### 音视频上传预转 - asyncOps

七牛云存储的云处理API（图像/音视频在线处理）满足如下规格:

    url?<fop>

即基于文件的 URL 通过问号传参来实现即时云处理，`<fop>` 表示 API 指令及其所需要的参数，是 File Operation 概念的缩写。

例如,

- [http://apitest.b1.qiniudn.com/sample.wav?avthumb/mp3/ar/44100/aq/3](http://apitest.b1.qiniudn.com/sample.wav?avthumb/mp3/ar/44100/aq/3)

其中,

- `url = http://apitest.b1.qiniudn.com/sample.wav`
- `fop = avthumb/mp3/ar/44100/aq/3`

表示将原 wav 格式的音频转换为 mp3 格式，并指定动态码率（VBR）参数为3，采样频率为 44100 进行输出。

由于音视频文件一般都比较大，转换也是一个比较耗时的操作，故七牛云存储提供上传异步预转功能，即文件上传完毕后执行异步转换处理，这样在访问时即可获取到已经转换好了的目标文件。

可以在上传时候指定预转选项，只需在生成 uploadToken 时对 **asyncOps** 赋值相应的 `<fop>` 指令即可。可同时异步执行多个预转指令：

    asyncOps = <fop>[;<fop2>;<fop3>;…;<fopN>]

**asyncOps** 预转示例参见如下说明。

**上传**

1. 设定 `asyncOps = "avthumb/mp3/ar/44100/ab/32k;avthumb/mp3/aq/6/ar/16000"`
2. 以此生成带有预转功能的上传授权凭证（UploadToken）
3. 向七牛云存储上传一个 aac 格式的音频文件
4. 传成功后，服务器会对这个 aac 音频文件异步地做如下两个预转操作
    - `avthumb/mp3/ar/44100/ab/32k`
    - `avthumb/mp3/aq/6/ar/16000`

**下载**

可以通过 `http://<domain>/<key>` 的形式下载：

- `http://<bucket>.qiniudn.com/<key>?avthunm/mp3/ar/44100/ab/32k`
- `http://<bucket>.qiniudn.com/<key>?avthumb/mp3/aq/6/ar/16000`

如果有为 `<fop>` 定义 `<style>`, 那么也可以用友好URL风格进行访问。

我们先来熟悉 [qboxrsctl](/tools/qboxrsctl.html) 的两个命令行，

    // 定义 url 和 fop 之间的分隔符为 separator 
    qboxrsctl separator <bucket> <separator>

    // 定义 fop 的别名为 aliasName
    qboxrsctl style <bucket> <aliasName> <fop>

例如:

    qboxrsctl separator <bucket> "."
    qboxrsctl style <bucket> "mp3" "avthumb/mp3/aq/6/ar/16000"

那么，以下两个 URL 则等价:

- `http://<bucket>.qiniudn.com/<key>?avthumb/mp3/aq/6/ar/16000`
- `http://<bucket>.qiniudn.com/<key>.mp3`


访问以上链接，如果之前上传已经成功做完预转，那么此次请求就不需要再转换，将会直接下载预转后的结果文件。

图片、视频预转类似，开发者需要熟悉七牛云存储的更多 `<fop>` 指令，参考:

- [图像处理接口](/api/image-process.html)
- [音频/视频/流媒体处理](/api/audio-video-hls-process.html) 


<a name="uploadToken-examples"></a>

### 生成 uploadToken 的样例代码

生成 uploadToken 的样例代码可以参考: 

- C - <https://github.com/qiniu/c-sdk/blob/develop/qiniu/rs.c>
- Go - <https://github.com/qiniu/api/blob/develop/rs/token.go>


<a name="dictionary"></a>

## 附录

<a name="MagicVariables"></a>

### 魔法变量 - MagicVariables

文件上传成功后，Qiniu-Cloud-Storage 返回给 App-Client 的 Response Body 可以包含该文件的一些属性信息，这些属性信息通常需要上传成功后询问 Qiniu-Cloud-Storage 得知，然后作为 Response Body 的一部分返回给 App-Client。

例如 App-Client 成功上传一张图片到 Qiniu-Cloud-Storage，App-Client 想知道该图片的一些信息像是 Etag, EXIF 等信息，App-Client 想求值得到的这些 Etag, EXIF 等信息我们可以通过魔法变量（MagicVariables）的方式获取。 

MagicVariables 是一组规定的 API Call，可以使用 `$(APIName)` 或者是 `$(APIName.FieldName)` 的形式进行求值。主要用在 [uploadToken 的 returnBody 选项](#uploadToken-returnBody) 中。

可用 MagicVariables 列表:

API 名称  | 子项 | 说明
----------|------|-------------------------------------------
etag      | 无   | 文件上传成功后的 etag，上传前不指定 key 时，etag 等同于缺省的 key
fname     | 无   | 原始文件名
fsize     | 无   | 文件大小，单位: Byte
mimeType  | 无   | 文件的资源类型，比如 .jpg 图片的资源类型为 `image/jpg`
imageInfo | 有   | 获取所上传图片的基本信息，支持访问子字段
exif      | 有   | 获取所上传图片EXIF信息，支持访问子字段
endUser   | 无   | 获取 uploadToken 中指定的 endUser 选项的值，即终端用户ID

MagicVariables 支持同 [JSON](http://json.org/) 对象一样的 `{Object}.{Property}` 访问形式，比如：

- {API名称}
- {API名称}.{子项}
- {API名称}.{子项}.{下一级子项}

其中花括号部分（“`{…}`”）实际情况下需要用具体的 API 及其子项代替。

MagicVariables 求值示例：

- `$(etag)` - 获取当前上传文件的 etag
- `$(fname)` - 获取原始文件名
- `$(fsize)` - 获取当前上传文件的大小
- `$(mimeType)` - 获取当前上传文件的资源类型
- `$(imageInfo)` -  获取当前上传图片的基本属性信息
- `$(imageInfo.width)` - 获取当前上传图片的原始高度
- `$(imageInfo.height)` - 获取当前上传图片的原始高度
- `$(imageInfo.format)` -  获取当前上传图片的格式
- `$(endUser)` - 获取 uploadToken 中指定的 endUser 选项的值，即终端用户ID

imageInfo 接口返回的 JSON 数据可参考：<http://qiniuphotos.qiniudn.com/gogopher.jpg?imageInfo>

- `$(exif)` - 获取当前上传图片的 EXIF 信息
- `$(exif.ApertureValue)` - 获取当前上传图片所拍照时的光圈信息
- `$(exif.ApertureValue.val)` - 获取当前上传图片拍照时的具体光圈值

exif 接口返回的 JSON 数据可参考：<http://qiniuphotos.qiniudn.com/gogopher.jpg?exif>


<a name="xVariables"></a>

### 自定义变量 - xVariables

已知 [上传API](#upload-api) 结构如下

HTML Form API

    <form method="post" action="http://up.qiniu.com/" enctype="multipart/form-data">
      <input name="key" type="hidden" value="{FileID}">
      <input name="x:custom_field_name" type="hidden" value="{SomeVal}">
      <input name="token" type="hidden" value="{UploadToken}">
      <input name="file" type="file" />
    </form>

自定义变量即是其中的 `x:custom_field_name`，Qiniu-Cloud-Storage 允许在 form 或 `multipart/form-data` 流中添加任意以 `x:` 开头的自定义字段，不限个数，例如：

	<input name="x:uid" value="xxx">
	<input name="x:album_id" value="yyy">

这样，开发者在 uploadToken 的 `callbackBody` 选项里面就可以通过 `$(x:album_id)` 引用此自定义字段的值。例如此时 `callbackBody` 可以设置为 `key=$(etag)&album=$(x:album_id)&uid=$(x:uid)`，App-Server 通过此设置即可得到 App-Client 端文件上传成功后附带的自定义变量的值。


<a name="error-code"></a>

### 错误码

HTTP 状态码 | 错误说明
------------|-----------------------------------------------------------------
400         | 请求参数错误
401         | 认证授权失败，可能是 AccessKey/SecretKey 错误或 UploadToken 无效
405         | 请求方式错误，非预期的请求方式
579         | 文件上传成功，但是回调（callback app-server）失败
599         | 服务端操作失败
614         | 文件已存在
631         | 指定的存储空间（Bucket）不存在
701         | 上传数据块校验出错
