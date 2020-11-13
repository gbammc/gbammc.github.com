---
layout: post
title: 使用 Cocoapods 集成 XCFramework 时的 “Multiple commands produce” 错误分析
keywords: ios,xcframework,cocoapods
description: 有坑不可怕，最怕是不知道为什么掉坑里
date: 2020-11-13 23:19 +0800
---
## 问题

最近把我们组负责的模块改成了通过 Cocoapods 集成到主工程，竟然在运行单元测试的 Target 时出现了类似下面的错误：

```bash
Showing All Messages
Multiple commands produce '/Users/Alvin/Library/Developer/Xcode/DerivedData/XCFrameworkDemo-fjmdgsvymhuemlebgdzvccvivdtu/Build/Products/Debug-iphonesimulator/cocoapods-artifacts-Debug.txt':

1. That command depends on command in Target 'XCFrameworkDemo' (project 'XCFrameworkDemo'): script phase “[CP] Prepare Artifacts”
2. That command depends on command in Target 'XCFrameworkDemoTests' (project 'XCFrameworkDemo'): script phase “[CP] Prepare Artifacts”
```

经过一番调查后才发现这是 Cocoapods 在集成 XCFramework 时的一个 bug。

## **XCFramework**

XCFramework 是苹果在 WWDC2019 新推出的代码框架格式，主要是为了解决代码库在不同平台、不同架构的分发问题。现在苹果生态下大致可分为 4 个不同的 OS（iOS，macOS，watchOS，tvOS），各个 OS 又支持不同的架构：

![各个 OS 支持的架构](https://i.loli.net/2020/11/10/Xya4pmxg7efG3bl.png)

以往为了支持多个平台，代码库的使用者需要配置不同的库，有时还需要设置一些搜索路径、编译选项等等。但如果代码库支持 XCFramework 后，就只需要把新的 .xcframework 文件直接拖到 Xcode，Xcode 将会自动为我们处理好一切。

XCFramework 的本质是一个类似文件夹的容器，它把多个平台和架构的文件都按指定的格式放置在一起。典型的目录结构类似下面这样，本文以 [BugfenderSDK](https://github.com/bugfender/BugfenderSDK-iOS) 为例：

![xcframework 的目录结构](https://i.loli.net/2020/11/10/qXIyamJZCWr7vO9.png)

Info.plist 跟其它地方的用途一样，记录了这个 .xcframework 的属性，只不过 `CFBundlePackageType` 这里是 `XFWK`，其余部分定义了这个 SDK 所支持的平台、架构和对应的库文件路径。

## 通过 Cocoapods 集成 XCFramework

作为苹果生态最受欢迎的依赖管理工具，Cocoapods 从 1.9 Beta 版本开始支持 XCFramework。为了处理 .xcframework 文件， `pod install` 后会在 Target 的构建阶段（Build Phases）新加入了名为 `[CP] Prepare Artifacts` 的步骤。

!["Prepare Artifacts" Build Phases](https://i.loli.net/2020/11/10/EtUquCA7M38cFrk.png)

可以看到，在这个步骤中执行了一个脚本文件，并且设定了输入输出文件列表。.xcfilelist 文件是 WWDC18 推出的新特性，用于帮助 Xcode 更好地处理 Targets 之间的依赖，可用于加快编译速度：

> In [WWDC'18 Session #408 ("Building Faster in Xcode")](https://developer.apple.com/videos/play/wwdc2018/408/), they present a new feature of Xcode 10 where you can provide .xcfilelist files to specify the list of input and output files for the build phase, which will allow Xcode to better determine the dependency graph between targets, files having to be rebuilt, etc.

> Note: Input & Output files for build phases were already present in previous versions of Xcode to determine when to run the build phases and optimise the build dependency resolution; what's new in Xcode 10 is the ability to put the list of those input/output files in xcfilelist — that can then be generated/updated by external tools without having to modify the xcodeproj for that

如果输入输出文件列表里的文件都已经存在，那么 Xcode 就会跳过对应步骤的重建。换句话说，开发者可以自定义构建步骤，跳过一些不需要重复执行的操作，减少编译时间。本文 Demo 里的输入输出文件列表内容如下。

Pods-XCFrameworkDemoTests-artifacts-Debug-input-files.xcfilelist：
```
${PODS_ROOT}/Target Support Files/Pods-XCFrameworkDemoTests/Pods-XCFrameworkDemoTests-artifacts.sh
${PODS_ROOT}/BugfenderSDK/BugfenderSDK.xcframework
```

Pods-XCFrameworkDemoTests-artifacts-Debug-output-files.xcfilelist：
```
${BUILT_PRODUCTS_DIR}/cocoapods-artifacts-${CONFIGURATION}.txt
```

而脚本所做的事可以概括为以下两点：

1. 根据编译目标的平台和架构找到 .xcframework 里对应的库，然后复制到 DerivedData 目录
2. 生成 `cocoapods-artifacts-${CONFIGURATION}.txt` 文件，里面的内容是第一步复制后的库路径，用于最终产品的编译链接。

![Prepare Artifacts](https://i.loli.net/2020/11/12/xXpnYreL5Ty8zFi.png)

本文的问题就出现在 XCFrameworkDemo 和 XCFrameworkDemoTests 两个 Target 都依赖了 BugfenderSDK 这个带 XCFramework 的第三方库，导致在 `[CP] Prepare Artifacts` 这一步都去生成 `cocoapods-artifacts-${CONFIGURATION}.txt` 文件，造成冲突。

## 解决方法

解决本文的问题有三个方法：

1. 升级到 1.10.0 或以上。从这个版本开始 Cocoapods 会重新移除 `[CP] Prepare Artifacts` 步骤，彻底解决问题。不过这需要整个项目组的人员及工具链都进行升级，成员较少的团队可以采用。
2. 避免主工程 Target 和单元测试 Target 同时直接依赖带有 XCFramework 的第三方库。比如将所有依赖都打包到另一个子 Target。要做到这点有可能比较复杂，比如把我这边的模块改为集成到子 Target 就需要很大的工作量。
3. 改为源码编译或者回归到原来的 .framework 或 .a 集成方式。
