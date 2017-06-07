---
layout: post
title: "xgboost 在 macOS 上的通用安装方法"
date: 2017-06-07 11:30:00 +0800
comments: true
categories:
keywords: xgboost,macOS,安装
description: 授人以鱼不如授人以渔
---

根据[官方文档](https://xgboost.readthedocs.io/en/latest/build.html)，使用系统自带的 clang 编译出来的 xgboost 并不支持多线程。要使用多线程特性就需要用带有 [OpenMP](https://zh.wikipedia.org/zh-hans/OpenMP) 的 gcc 来编译。没安装过 gcc 的还得先安装一个：

``` shell
brew install gcc --without-multilib
```

然后就是下载 xgboost 的源码：

``` shell
git clone --recursive https://github.com/dmlc/xgboost
```

在编译前，macOS 10.10 及以后的系统还需要修改 make 文件来指定使用的 gcc 版本，目前网络上大多数博客都是说要改为 __gcc-6/g++-6__ ，然而因为 gcc 的版本也会随着时间变化，我们也要根据系统安装的版本来作对应修改，例如在写这篇博客时我安装的是第七版，就应该把 __Makefile__ 的 47 和 50 行以及 __make/config.mk__ 的 20 和 21 行改为：

``` shell
export CC = gcc-7
export CXX = g++-7
```

然后运行：

``` shell
cd xgboost; cp make/config.mk ./config.mk; make -j4
```

等大约半小时后再安装编译后的 python 包：

``` shell
cd python-package; sudo python setup.py install
```
