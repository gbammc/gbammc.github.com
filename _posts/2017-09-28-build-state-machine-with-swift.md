---
layout: post
title: 用 Swift 实现一个简单的状态机
date: 2017-09-28 13:49:00 +0800
comments: true
categories:
keywords: ios,swift,statemachine,state,machine,状态机,mach-o,dyld
description: 使用状态机对于复杂的状态转移有很好的理解和简化作用
---

使用（有限）状态机对于复杂的状态转移有很好的理解和简化作用。一个状态机一般有以下特征：

* 状态（state）总数是有限的；
* 任何时刻只在一种状态之中；
* 接收到某个事件（event）触发后，会从一种状态转移（transition）到另一种状态。

下面就以[番茄工作法](https://zh.wikipedia.org/wiki/%E7%95%AA%E8%8C%84%E5%B7%A5%E4%BD%9C%E6%B3%95)的流程作为例子实现一个状态机。

![番茄工作法](http://ww1.sinaimg.cn/large/4ccba622gy1fjz8trgayhj21fm0uitc2.jpg)

上图是番茄工作法的一个状态转移图。可以看到一共有 4 个状态：闲时、工作、短休息和长休息。还有这些状态之间的 5 个转移。

首先来定义各个状态和触发各个状态转移的事件：

```swift
protocol StateType: Hashable {}
enum State: StateType {
    case idle
    case work
    case shortBreak
    case longBreak   
}

protocol EventType: Hashable {}
enum Event: EventType {
    case startWork
    case startShortBreak
    case startLongBreak
    case backToIdle
}
```

这样，我们的状态机可以先定义为：

```swift
class StateMachine<S: StateType, E: EventType> {
    private(set) var currentState: S
    
    init(_ initialState: S) {
        self.currentState = initialState
    }
}

let stateMachine = StateMachine<State, Event>(.idle)
```

在初始化的时候除了要指定状态和事件的具体类型外，还需要提供一个初始状态作为当前状态。

接着为了处理事件发生，添加一个记录状态转移的方法：

```swift
func listen(_ event: E, transit fromState: S, to toState: S, callback: @escaping () -> Void) {
}
```

我们监听事件 event 发生时，如果当前状态为 fromState，那么转移状态到 toState，并且执行回调方法。

为了方便回调时获取整个转移过程，我们再定义一个 Transition 结构体用于封装这些内容：

```swift
struct Transition<S: StateType, E: EventType> {
    let event: E
    let fromState: S
    let toState: S
    
    init(event: E, fromState: S, toState: S) {
        self.event = event
        self.fromState = fromState
        self.toState = toState
    }
}
```

在 ```StateMachine``` 里再定义一个 ```Operation``` 的对象，把对应的回调和转移绑定一起：

```swift
private struct Operation<S: StateType, E: EventType> {
    let transition: Transition<S, E>
    let triggerCallback: (Transition<S, E>) -> Void
}
   
private var routes = [S: [E: Operation<S, E>]]()
```

记录状态转移的方法再改为：

```swift
func listen(_ event: E, transit fromState: S, to toState: S, callback: @escaping (Transition<S, E>) -> Void) {
    var route = routes[fromState] ?? [:]
    let transition = Transition(event: event, fromState: fromState, toState: toState)
    let operation = Operation(transition: transition, triggerCallback: callback)
    route[event] = operation
    routes[fromState] = route
}
```

最后添加触发事件的函数：

```swift
func trigger(_ event: E) {
    guard let route = routes[currentState]?[event] else { return }
    
    route.triggerCallback(route.transition)
    currentState = route.transition.toState
}
```

这样我们简易的状态机就能开始工作了。示例用法如下：

```swift
let stateMachine = StateMachine<State, Event>(.idle)

stateMachine.listen(.startWork, transit: .idle, to: .work) { (transition) in
    ...
}

stateMachine.trigger(.startWork)
```

可以看到，有限状态机的写法，逻辑清晰，表达力强，有利于封装事件。一个对象的状态越多、发生的事件越多，就越适合采用有限状态机的写法。

总的行数不超过 60 行，完整的代码在[这里](https://gist.github.com/gbammc/3546ae3b52037756bc3065d9f1fc8f29)

## Reference

[JavaScript与有限状态机](http://www.ruanyifeng.com/blog/2013/09/finite-state_machine_for_javascript.html)

