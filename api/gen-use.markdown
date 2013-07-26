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

<a name="auth-request"></a>

## 用户请求的验证

<a name="upload-token"></a>

## Upload Token

<a name="download-token"></a>

## Download Token

<a name="access-token"></a>

## Access Token

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
