---
layout: post
title: "Cocoa开发 -- 事件监听"
date: 2013-11-05 21:45
comments: true
categories: Mac
keywords: cocoa,mac,keyboard,event,listen,按键,事件,监听,carbon,全局,快捷键
description: Thor 是我开发的第一个 Mac 应用,它实现的功能是通过按下全局的快捷键来切换应用,以此达到提高效率的目的。
---

# 前言

[Thor](https://github.com/gbammc/Thor) 是我开发的第一个Mac应用,它实现的功能是通过按下全局的快捷键来切换应用,以此达到提高效率的目的。开发的原动力是因为最近发现我的手指在触摸板的使用过程中越来越痛了,而且不停的control+tab来切换应用觉得实在很烦很低效,但我经常需要切换的应用其实也就那么三四个,所以 Thor 就应运而生了。而在开发过程中所学到的Cocoa开发知识我想记录下来,所以这是第一篇 —— 事件监听。

---

# 全局快捷键

全局快捷键是 Thor 中最重要的部分了,通过底层的Carbon.framework提供的API, Thor 向系统注册全局的快捷键,当相应的按键被按下时,系统就可以立刻切换到对应的程序。当然我们可以它来做更多的事,下面就介绍如何注册一个全局快捷键。

## 注册快捷键
需要用Carbon框架,当然少不了添加依赖链接,然后在对应文件上加上引用 ```#import <Carbon/Carbon.h>```

在方法中添加注册快捷键需要的变量:

``` objc

EventHotKeyRef gMyHotKeyRef;
EventHotKeyID gMyHotKeyID;
EventTypeSpec eventType;
eventType.eventClass = kEventClassKeyboard;
eventType.eventKind = kEventHotKeyPressed;

```

这些变量将会保存我们快捷键的基本信息。通过对EventTypeSpec的eventClass和eventKind分别赋予kEventClassKeyboard和kEventHotKeyPressed两个值,来声明这个快捷键响应的是键盘按键事件。

接着我们注册一个实际的快捷键:

``` objc

// 快捷键的签名,实际类型为UInt32,所以用4个字符最好
gMyHotKeyID.signature='hkid';
// 快捷键的id，处理多个全局快捷键时最好的标识
gMyHotKeyID.id=1;
// 通过这个函数向系统注册快捷键
RegisterEventHotKey(49, cmdKey + optionKey, gMyHotKeyID, GetApplicationEventTarget(), 0, &gMyHotKeyRef);

```

因为Carbon是用基于C语言写成的,所以RegisterEventHotKey()调用形式是C类型,第一个参数的数字,代表最终响应的键值,49代表空格键。第二个参数是控制键参数,可以选cmdKey, shiftKey, optionKey, controlKey或者是它们的组合,需要注意的是它们的连接使用‘+’号。当快捷键注册成功后,最后一个参数将会返回这个快捷键在系统中的引用。我们可以保存这个引用以便在以后再取消该快捷键。

## 实现和注册回调函数

回调函数原型为

``` objc

OSStatus hotKeyHandler(EventHandlerCallRef nextHandler, EventRef anEvent, void *userData);

```

同样也是一个C类型函数，函数签名必须是上面的形式。示例实现代码如下：

``` objc

OSStatus hotKeyHandler(EventHandlerCallRef nextHandler, EventRef anEvent, void *userData) {
    if (!((AZAppDelegate *)[NSApplication sharedApplication]。delegate)。enableHotKey) return noErr;

    EventHotKeyID hotKeyRef;

    GetEventParameter(anEvent, kEventParamDirectObject, typeEventHotKeyID, NULL, sizeof(hotKeyRef), NULL, &hotKeyRef);

    unsigned int hotKeyId = hotKeyRef.id;

    switch (hotKeyId) {
        case 1:
            // do something
            break;
        case 2:
            // do other thing
            break;
        default:
            break;
    }

    return noErr;
}

```

处理多个全局快捷键的方式正如上面所示，通过判断快捷键的id值即可。最后我们在EventTypeSpec的赋值后面注册这个回调函数

``` objc

EventHotKeyRef gMyHotKeyRef;
EventHotKeyID gMyHotKeyID;
EventTypeSpec eventType;
eventType.eventClass = kEventClassKeyboard;
eventType.eventKind = kEventHotKeyPressed;
InstallApplicationEventHandler(&hotKeyHandler, 1, &eventType, NULL, NULL);

```

至此，Mac系统就可以响应我们的快捷键了。

----

# 监听系统事件

全局快捷键的响应必须由控制键参数和最终响应的键同时按下才能触发。如果我们要监听任意按键事件或鼠标事件呢？那么就需要```NSEvent```下的这两个方法了：

``` objc

// 监听全局事件
+ (id)addGlobalMonitorForEventsMatchingMask:(NSEventMask)mask handler:(void (^)(NSEvent*))block;
// 监听本地事件
+ (id)addLocalMonitorForEventsMatchingMask:(NSEventMask)mask handler:(NSEvent* (^)(NSEvent*))block;

```

这两个方法分别监听的是全局事件和本地事件。只要我们指定监听的```NSEventMask```类型，当对应事件触发时，我们就能够对其作进一步的处理。```NSEventMask```类型有十多种，这里就不全列出来了，不过基本覆盖了用户与系统所有的交互事件，在开发时选择合适的组合就好。

全局事件和本地事件的不同之处，在于全局事件我们不可以修改或阻止它继续向其它应用传送。但对于本地事件来说我们还是有能力这么做的，所以本地事件回调的block里我们需要返回一个NSEvent的变量。示例代码如下：

``` objc

// 全局
[NSEvent addGlobalMonitorForEventsMatchingMask:NSFlagsChangedMask handler:^(NSEvent *event){
    NSUInteger flags = [event modifierFlags] & NSDeviceIndependentModifierFlagsMask;
    if (flags == NSCommandKeyMask) {
        // handle it
    }
}];
// 本地
[NSEvent addLocalMonitorForEventsMatchingMask:NSFlagsChangedMask handler:^(NSEvent *event){
    NSUInteger flags = [event modifierFlags] & NSDeviceIndependentModifierFlagsMask;
    if (flags == NSCommandKeyMask) {
        // handle it
    }
    return event;
}];

```
----

# 让控件响应特定事件

上面的方法已经很好很强大了，但如果我们要令特定控件响应某些事件呢？比如让NSTextField在按下esc键后清除所有内容，当鼠标移动到图片上时让其高亮，这样我们就不能简单地使用以上的方法了。但对于这种情况我们可以用__NSResponder__的方法处理。

__NSResponder__中定义了很多事件响应方法，如:


``` objc

- (void)rightMouseDown:(NSEvent *)theEvent;
- (void)otherMouseDown:(NSEvent *)theEvent;
- (void)mouseUp:(NSEvent *)theEvent;
- (void)rightMouseUp:(NSEvent *)theEvent;
- (void)otherMouseUp:(NSEvent *)theEvent;
- (void)mouseMoved:(NSEvent *)theEvent;
- (void)mouseDragged:(NSEvent *)theEvent;
- (void)scrollWheel:(NSEvent *)theEvent;
- (void)rightMouseDragged:(NSEvent *)theEvent;
- (void)otherMouseDragged:(NSEvent *)theEvent;
- (void)mouseEntered:(NSEvent *)theEvent;
- (void)mouseExited:(NSEvent *)theEvent;
- (void)keyDown:(NSEvent *)theEvent;
- (void)keyUp:(NSEvent *)theEvent;
- (void)flagsChanged:(NSEvent *)theEvent;
- (void)tabletPoint:(NSEvent *)theEvent;
- (void)tabletProximity:(NSEvent *)theEvent;
- (void)cursorUpdate:(NSEvent *)event;

```

通过定制这些方法，就可以让控件的表现得更独特或让用户感到更便捷，下面是按下```cmd + W```关闭窗口事件的示例代码:

``` objc

- (void)keyDown:(NSEvent *)theEvent {
    NSString *key = [theEvent charactersIgnoringModifiers];
    if (([theEvent modifierFlags] & NSCommandKeyMask) && [key isEqualToString:@"w"]) {
        [self close];
    } else {
        [super keyDown:theEvent];
    }
}

```

----

# PS：

经常在论坛上看到有人说Mac开发很难，我想可能是因为在Mac开发方面的中文资料太少了，在开发 Thor 的过程中我也碰到了不少问题，不过基本上都是靠苹果的英文文档，StackOverflow或Google搜到的国外博客的帮助下解决的。现在OS X免费了，加上Mac产品优秀的用户体验(可能有些装X，但谁用谁知道)，我想以后Mac的用户会越来越多，市场也会越来越大，真心希望Mac开发的阵营也能更大~

