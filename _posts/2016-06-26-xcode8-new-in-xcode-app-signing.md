---
layout: post
title: "Xcode8 新特性 -- 新应用签名机制"
date: 2016-06-26 12:30:00 +0800
comments: true
categories: iOS
keywords: xcode8 signing 应用签名 wwdc wwdc2016 iOS
description: Xcode8 提供新的签名管理功能，更加方便开发者管理各种证书问题了。
---

这是我 WWDC2016 笔记中的一篇，本文仅作为个人记录使用，也欢迎在[许可协议](http://creativecommons.org/licenses/by-nc/3.0/deed.zh)范围内转载或使用，但是烦请保留原文链接，谢谢您的理解合作。

## 签名

Xcode8 提供新的签名管理功能，带来了以下新特性：

* 新的基础架构
* 新的工作流程和 UI
* 可操作消息提醒
* 状态报告

以前处理应用签名对于开发者来说是非常烦人的事，多人开发或者多设备开发经常遇到需要导出证书或 revoke / recreate 证书的问题，但 Xocde8 经过重新设计后在这方面更加方便易用，烦人的 `Fix Issue` 按钮不再出现，取而代之的是更加友好的自动操作或者是错误提示。首先简要地回顾下签名机制。

### 签名机制

代码签名可以让用户验证我们的身份（macOS 上使用没有签名的应用将会出现提示或不允许使用，而在 iOS 上更是不能安装），防止签名后的应用被修改，以及允许访问系统的服务。整个签名机制主要由以下三个文件组成：签名证书（Signing certificates），配置文件（Provisioning profiles）和授权文件（Entitlements）。

#### 签名证书

![签名证书](http://ww1.sinaimg.cn/large/4ccba622gw1f58i478caaj20hm07pt9q.jpg)

* 由苹果签发
* 分成开发证书和分发证书
* 开发设备必须有对应证书的密钥
* 苹果不会保存该密钥

#### 配置文件

![配置文件](http://ww3.sinaimg.cn/large/4ccba622gw1f58i52g96vj20hk08dwf6.jpg)

* 同样由苹果签发
* 应用专用
* 决定应用能否在一个特定设备上运行
* 联系可使用的授权文件

#### 授权文件

![授权文件](http://ww1.sinaimg.cn/large/4ccba622jw1f58i602znkj20hi089js4.jpg)

* 声明可使用的系统服务
* 各 target 有对应的授权文件
* 可使用 `Capabilities` 面板管理

总的来说，在签名时配置文件和可选的授权文件会和我们的程序一同进行打包，而证书则用于生成代码封条（code seal）。

以下就是 Xcode8 提供的新特性如何帮助我们更专注于开发上：

## 每台 Mac 设备将拥有独立的开发证书

![独立证书](http://ww1.sinaimg.cn/large/4ccba622jw1f58i6l1pi2j20ov0aldhs.jpg)

在 Xcode8 以前，我们如果想用同一个开发者账号在不同的 Mac 上开发，就必须从证书生成的 Mac 上把证书和密钥导出到另一台 Mac 上安装。但 Xcode8 后，每台 Mac 都会根据开发者账号生成一个设备独立的开发证书，这样我们就免去证书导出或者是 revoke / recreate 证书的烦恼。不过分发证书还是只能由一台设备存有。

## 支持自动管理证书和自定义管理证书

### 自动管理证书

![自动管理证书](http://ww1.sinaimg.cn/large/4ccba622jw1f58i72qxqzj20om0dpmyk.jpg)

自动管理证书将会根据你所选择的开发者账号自动为你处理配置文件和签名证书的设置，并且只在必要时进行提醒。

如果你好奇 Xcode 为你做了什么，在`报告`面板里还列出了每一个步骤和原因。例如，我刚在 Capabilities 把 CloudKit 打开，重新 build 一次后出现以下消息：

![build 报告](http://ww2.sinaimg.cn/large/4ccba622jw1f58i7oqfgjj20m805zmyz.jpg)

另外，自动管理证书仅对 Xcode 生成的配置文件有效，所以不用担心手动生成的会被修改。

### 自定义管理证书

![自定义证书](http://ww2.sinaimg.cn/large/4ccba622gw1f58i95k3jtj21130lx78l.jpg)

在自定义管理证书模式下，Xcode8 允许你对各 Configurations 进行配置文件和签名证书的设置。不过在面板上将会提示你出现什么错误。

![新引用配置文件的方式](http://ww3.sinaimg.cn/large/4ccba622jw1f58iafxocuj20nc059t9k.jpg)

新的 `PROVISIONING_PROFILE_SPECIFIER` 编译参数将废弃以前通过 unique id 引用配置文件的方式，改为通过“TeamID + 配置文件名称”的方式去引用（在工程文件中保存为：PROVISIONING_PROFILE_SPECIFIER = "XXXXXXXXXX/ProvisionFileName"），这样做的好处是以前当我们添加一个小伙伴到团队或一个新设备后配置文件就需要重新生成，但以后这些都交给 Xcode 自己处理吧。

## 最佳实践

* 不要设置 `CODE_SIGN_IDENTITY`
* 使用新的 `General` 和 `Capabilities` 面板操作
* 使用自动签名



