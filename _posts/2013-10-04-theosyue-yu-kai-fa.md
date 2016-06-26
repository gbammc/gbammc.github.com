---
layout: post
title: "TheOS越狱开发"
date: 2013-10-04 22:45
comments: true
categories: iOS 越狱开发
keywords: ios,theos,越狱开发,octopress
description: iOS 越狱开发入门。
---

因为最近对越狱开发感兴趣起来，所以果断利用Google去找资料，结果发现相关的东西却是少之又少，于是想如果我能在这个摸爬过程中学到一些东西的话，就都把它整理post出来，让更多有兴趣的人也能参与当中。

而对于越狱开发，我是这么认为的：为了利用iOS系统私有的API，你不得不去挖掘系统底层的东西，在这个过程中，你又会加深对整个系统的了解，从而将自己的开发水平提升一个档次。虽然苹果不会鼓励这样做，不过也是一条有乐趣的学习之道吧。

---

### 1.TheOS环境搭建

[http://brandontreb.com/beginning-jailbroken-ios-development-your-first-tweak](http://brandontreb.com/beginning-jailbroken-ios-development-your-first-tweak)
这是国外很全面的TheOS环境搭建和打包工具安装以及一个简单TheOS程序示例，英语比较好的同学可以参照这一篇。

安装theos主要步骤如下：


``` bash
$ export THEOS=/opt/theos
$ git clone git://github.com/DHowett/theos.git $THEOS
$ sudo chmod -Rf 777 $THEOS
$ curl -s http://dl.dropbox.com/u/3157793/ldid > $THEOS/bin/ldid
$ chmod 777 $THEOS/bin/ldid
$ cd $THEOS/include/IOSurface
$ sudo curl -O -k https://raw.github.com/javacom/toolchain4/master/Projects/IOSurfaceAPI.h
```

这样就完成TheOS的安装了。

另外还需要下载一个class-dump工具包，用于导出苹果库中的私有头文件:
[https://github.com/nygard/class-dump](https://github.com/nygard/class-dump)
，或者也可以直接使用rpetrich大神的[头文件库](https://github.com/rpetrich/iphoneheaders)。把得到的头文件解压并放到$THEOS/include中即可。

---

### 2.打包工具

#### 1)安装Macports

如果已经安装了那么可以忽略这一步，查看是否已经安装可以输入如下指令：

``` bash
$ port version
```

官方下载传送门：[http://www.macports.org/install.php](http://www.macports.org/install.php)

#### 2)安装dpkg
安装dpkg的命令

``` bash
$ sudo port install dpkg
```

---

### 3.TheOS生成的Makefile文件剖析

``` bash
include theos/makefiles/common.mk
```

告诉TheOS在编译脚本仲包括共同的make命令，避免做重复的make编译工作

``` bash
TWEAK_NAME = HelloWorld
```

我们要变异的应用程序的名称。Makefile将会用这个常量在内部做一些事情。除非你的应用程序改名称，否则不要修改这个值。

``` bash
[TWEAK_NAME]_FILES = Tweak.xm
```

这里是需要编译的文件列表。注意：不要把头文件添加到这里。如果你要添加一个新的.m或者.mm文件到项目中，确保在这里添加新的文件名称，否则将不会建立编译连接。

``` bash
[TWEAK_NAME]_FRAMEWORKS = UIKit Foundation
```

这里包括你想用到框架的名称

``` bash
include $(THEOS_MAKE_PATH)/application.mk
```

更多默认的用于帮助TheOS建立项目

---


### 4.第一个HelloWorld后台程序

在终端中输入：

``` bash
$ export THEOS=/opt/theos/
$ export SDKVERSION=5.1			// sdk版本号
$ $THEOS/bin/nic.pl				// 执行TheOS
NIC 2.0 - New Instance Creator
------------------------------
	[1.] iphone/application
	[2.] iphone/cydget
	[3.] iphone/dashboardx_widget
	[4.] iphone/framework
	[5.] iphone/library
	[6.] iphone/notification_center_widget
	[7.] iphone/preference_bundle
	[8.] iphone/sbsettingstoggle
	[9.] iphone/tool
	[10.] iphone/tweak
Choose a Template (required): 10
Project Name (required): HelloWorld
Package Name [com.yourcompany.helloworld]:
Author/Maintainer Name [Alvin]:
[iphone/tweak] MobileSubstrate Bundle filter 	[com.apple.springboard]:
Instantiating iphone/tweak in helloworld/...
Done.
```

我们创建一个后台程序，要做的是当系统开机时候弹出HelloWorld字样的Alert框。

打开Tweak.xm文件，清空后添加以下代码

``` objc
#import <UIKit/UIKit.h>

%hook SpringBoard

-(void)applicationDidFinishLaunching:(id)application {
	%orig;

	UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Welcome"
    	message:@"Welcome to your iPhone Brandon!"
    	delegate:nil
    	cancelButtonTitle:@"Thanks"
    	otherButtonTitles:nil];
	[alert show];
	[alert release];
}

%end

```

因为我们要用到hook（钩子）钩取系统开机时候调用的其中一个函数，在那个函数中插入我们的Alert。%orig;功能是执行这个函数原来的动作。如果你想完完全全禁止某个函数的功能，不使用 %orig;即可。

然后打开Makefile文件添加

``` bash
HelloWorld_FRAMEWORKS = UIKit
```

---

### 5.编译和安装

至此，便到了编译和安装的步骤：

__1.利用ssh安装__

首先export iPhone的ip：

```	bash
$ export THEOS_DEVICE_IP=192.168.1.115
```

然后输入：

```	 bash
$ make install
```

在运行过程中可能会多次要求输入密码，默认是alpine。安装完成后，你的设备就会自动重启，并且会显示你自定义的信息。

__2.自行打包手动安装__

打开控制台，进入到你的这个工程文件夹，使用命令

``` bash
make
make package
```

然后会生成一个com.yourcompany.fooproject_0.0.1-1_iphoneos-arm.deb这样的包在你的目录中，

然后利用iFunbox，iTools等工具将改包放到/private/var/root/Media/Cydia/AutoInstall/中，那么就会自动安装。

或者也可以使用iFiles手动安装。


---


### PS：
在输入make install后可能会出现一下几种情况：

##### 1.ssh Connection refused

``` bash
ssh: connect to host 192.168.1.115 port 22: Connection refused
lost connection
```

这是因为你还没有打开ssh，在SBSettings里的开关中你应该见到对应的开关。

##### 2.Connection closed by remote host

从Cydia里安装SBSettings和OpenSSH后，如果在终端里输入

``` bash
ssh root@192.168.1.115			// iPhone和Mac处于同一网络下
```

后，出现

``` bash
ssh_exchange_identification: Connection closed by remote host
```

那么应该立刻使用iFunBox等软件进入iPhone的文件系统，并删除以下内容

```
/System/Library/LaunchDaemons/com.ikey.bbot.plist
/bin/poc-bbot
```

然后重新安装OpenSSH。

出现以上问题，说明你的iPhone已经中了名叫ikee的病毒，所以如果装了OpenSSH后，应该立刻改掉你的root密码，默认是alpine。例如在Cydia安装Mobile Terminal，然后输入

``` bash
$ su
// 输入密码如：alpine
$ passwd root	// 输入新密码
```

参考资料：[[iPhone]OpenSSH连不上：ssh_exchange_identification: Connection closed by remote host](http://hi.baidu.com/celavi/item/83c5c68e241b5cd55f0ec1ad)

##### 3.collect2: ld terminated with signal 6 [Abort trap: 6]

出现如下错误：

``` bash
Making all for tweak PreferenceLoader...
Preprocessing Tweak.xm...
Compiling Tweak.xm...
Linking tweak PreferenceLoader...
collect2: ld terminated with signal 6 [Abort trap: 6]
ld(8724,0x7fff78fd2960) malloc: *** error for object 0x7f89b35003f0: pointer being freed was not allocated
*** set a breakpoint in mallocerror_break to debug
make[2]: *** [obj/PreferenceLoader.dylib] Error 1
make[1]: *** [internal-library-all] Error 2
make: *** [PreferenceLoader.all.tweak.variables] Error 2
```

那是因为Xcode4.5附带的两个不同版本的链接器。一个用于gcc来编译armv6架构，另一个clang则不会产生armv6的输出。现在没有理由用6.0的SDK去targetiOS4.3以下的系统，或其它基于armv6的平台。我们可以在Makefile最顶部加上如下两句：

```
export ARCHS=armv7
export TARGET=iphone:latest:4.3
```

另外，如果要兼容低版本那么可以替换为：

```
export ARCHS=armv6 armv7
export TARGET=iphone:<sdkversion-lower-than-6.0>:<deployment target-higher-than-3.0>
```

