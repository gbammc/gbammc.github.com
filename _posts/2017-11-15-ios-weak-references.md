---
layout: post
title: OC 和 Swift 的弱引用源码分析
date: 2017-11-15 17:02:00 +0800
comments: true
categories:
keywords: ios,swift,weak,reference,objective-c,oc,
description: 通过源码来学习 iOS 的设计
---

用引用计数进行内存管理，必然会发生“循环引用”的问题，为了正确打破对象间相互引用的关系，我们一般的方法都是使用 ```weak``` 作为工具。通过 ```weak``` 修饰符表示的弱引用除了不会增加对象的引用计数外，另一个好处是，当引用的对象被释放后，这个弱引用会自动失效并且处于 nil 的状态（zeroing）。

以下就来尝试分析苹果对 Objective-C 和 Swift 分别的实现原理。

## OC 时代

OC 的 ```__weak``` 关键字是随着 iOS5 新增的 ARC 特性而来。最早的实现出现在苹果开源的 [objc4-493.9](https://opensource.apple.com/tarballs/objc4/)。
而文中的源码来自最新的 [objc4-723](https://opensource.apple.com/tarballs/objc4/) 版本。关于弱引用的实现主要在 ```objc-weak.h```、```objc-weak.mm``` 和 ```NSObject.mm``` 这三个文件中。

### 初始化

当我们像下面那样初始化一个弱引用时：

``` objc
// NSObject *o = ...;
__weak id weakPtr = o;
```

编译器会转换为下面的实现：

``` objc
// NSObject *o = ...;
objc_initWeak(&weakPtr, o);
```

对于 ```objc_initWeak()``` 的实现：

``` c
id objc_initWeak(id *location, id newObj)
{
	// 查看对象是否有效
	// 无效对象立刻使指针置空
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```

可以看到，这个函数最后会调用 ```storeWeak()```，而且传入的三个[非类型模板参数](https://msdn.microsoft.com/zh-cn/library/x5w1yety.aspx#非类型模板参数)的名字很好地解释了它们的意义：弱引用不存在已有指向对象（```DontHaveOld```），但需要指向新的对象（```DoHaveNew```），如果目标对象正在释放，那就崩溃吧（```DoCrashIfDeallocating```）。

再来看一下 ```storeWeak()``` 的实现：

``` c
template <HaveOld haveOld, HaveNew haveNew, CrashIfDeallocating crashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    // 用于标记已经初始化的类
    Class previouslyInitializedClass = nil;
    id oldObj;
    // 声明新旧辅助表
    SideTable *oldTable;
    SideTable *newTable;

    // 获取新旧值（存在的话）的辅助表，并且加锁，
    // 如果新旧值辅助表同时存在时，以锁的地址大小排序，防止锁的顺序问题
 retry:
    if (haveOld) {
        // 如果有旧值的话，通过指针获取目标对象，
        // 再以目标对象的地址为索引，取得旧值对应的辅助表
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        // 如果有新值，以新值的地址为索引，取得新值对应的辅助表
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    // 加锁
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
   		// 线程冲突处理，
   		// 如果有旧值，但 location 指向的对象不为 oldObj，那很可能被其它线程修改过，
   		// 解锁并重试
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // 确保新值的 isa 类已经调用 +initialize 初始化，
    // 避免弱引用机制和 +initialize 机制间的死锁
    if (haveNew  &&  newObj) {
        // 获得新值的 isa 类
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&
            !((objc_class *)cls)->isInitialized())
        {
            // 新值 isa 非空，并且未初始化，
            // 解锁
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            // 初始化 isa
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // 如果这个 isa 类正在当前线程运行 +initialize
            //（例如在 +initialize 方法里对自己的实例调用了 storeWeak ），
            // 很显然会处于一个正在初始化，但未初始化完的状态，
            // 所以设置 previouslyInitializedClass 为这个类进行标记
            previouslyInitializedClass = cls;

            // 重试
            goto retry;
        }
    }

    // 清除旧值
    if (haveOld) {
        // 从 oldObj 的弱引用条目删除弱引用的地址
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // 设置新值
    if (haveNew) {
        // 把弱引用的地址注册到 newOjb 的弱引用条目
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location,
                                  crashIfDeallocating);

        // 如果 weakStore 操作应该被拒绝，weak_register_no_lock 会返回 nil，否则
        // 对被引用对象设置弱引用标记位（is-weakly-referenced bit）
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        *location = (id)newObj;
    }
    else {
        // 没有新值，不用更改
    }

    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

可以看到，很多操作都需要对 ```SideTable``` 的实例进行操作。实际上 ```SideTable``` 也的确是作为全局对象用于管理所有对象的引用计数和 weak 表，在 Runtime 启动时就和主线程的 AutoreleasePool 一同创建。

### SideTable

SideTable 的定义如下：

``` c
struct SideTable {
    spinlock_t slock;			// 用于原子操作的自旋锁
    RefcountMap refcnts;		// 引用计数哈希表
    weak_table_t weak_table;	// weak 表

    // ...
};
```

从 ```storeWeak()``` 可以看到，Runtime 是通过以下方式获取对象的 ```SideTable```：

```
objSideTable = &SideTables()[obj];
```

这个 ```SideTables()``` 方法返回的就是一个 ```StripedMap``` 的哈希表，以对象的地址作为键值返回对应的 ```SideTable```。

```
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

```reinterpret_cast``` 是C++标准转换运算符，用来处理无关类型之间的转换，它会产生一个新的值，这个值会有与原始参数（expressoin）有完全相同的比特位。

``` c
reinterpret_cast <new_type> (expression)
```

```StripedMap``` 是一个模板类，定义于 ```objc-private.h``` 文件中，提供了一个以地址为键值的哈希结构。

```
template<typename T>
class StripedMap {
	// ...

	// 嵌入式系统的 StripeCount 为 8，iOS 上为 64
	enum { StripeCount = 64 };

	static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);

        // 哈希操作
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
	}

public:
    T& operator[] (const void *p) {
        return array[indexForPointer(p)].value;
    }
    const T& operator[] (const void *p) const {
        return const_cast<StripedMap<T>>(this)[p];
    }

	// ...
}
```

在实现中，```StripedMap``` 重定义了数组运算符，传入对象的地址即可通过哈希算法获得对应内容。从原有的注释可以看到，在 Runtime 初始化后，iOS 系统就生成了 64 个 ```SideTable``` 留作以后的使用。

```SideTable``` 里与弱引用有直接关系的就是 weak 表。weak 表也是作为哈希表实现，将目标对象的地址作为键值进行检索以获取对应的弱引用变量地址。另外，由于一个对象可同时赋值给多个弱引用变量，所以对于一个键值，可注册多个弱引用变量的地址。

``` c
struct weak_table_t {
    // 弱引用条目列表
    weak_entry_t *weak_entries;
    // 条目数量
    size_t    num_entries;
    // 条目列表大小
    uintptr_t mask;
    // 最大哈希偏移值
    uintptr_t max_hash_displacement;
};
```

```weak_entries``` 和上面的 ```StripedMap``` 不同，```StripedMap``` 并不需要处理冲突，但因为 ```weak_entries``` 需要对应到具体的内容，所以出现冲突后还需要再处理，苹果使用的是[开放地址法](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8#.E5.A4.84.E7.90.86.E5.86.B2.E7.AA.81)，```max_hash_displacement``` 就是用于出现冲突后辅助检查查找的内容是否存在。

``` c
typedef DisguisedPtr<objc_object *> weak_referrer_t;
#define WEAK_INLINE_COUNT 4

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    // ...
};
```

以上是 ```weak_entry_t``` 的定义，目标对象和弱引用变量的指针都被封装在 ```DisguisedPtr``` 里。```DisguisedPtr``` 用于隐藏封装对象的类型，避免内存分析工具可以轻松看到其中的类型信息。可以看到苹果为了节省内存空间和效率，特意使用了联合结构。当目标对象的弱引用数少于等于 ```WEAK_INLINE_COUNT``` 时，将会使用內联静态数组的形式来存取弱引用指针地址，否则就会以和 ```weak_table_t``` 相同的结构来存储（同样以地址作为键值的哈希表）。

### zeroing

OC 的弱引用变量 zeroing 发生在目标对象的释放时候。在对象的 ```dealloc``` 过程中会调用 ```weak_clear_no_lock``` 函数：

```  c
/**
 * Called by dealloc; nils out all weak pointers that point to the
 * provided object so that they can no longer be used.
 */
void
weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
{
    objc_object *referent = (objc_object *)referent_id;

    // 获取弱引用条目
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        return;
    }

    weak_referrer_t *referrers;
    size_t count;

    // 获取弱引用变量地址数组和数目
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    }
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }

    // 把它们全置为 nil
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n",
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }

    // 移除整个条目
    weak_entry_remove(weak_table, entry);
}
```

### 小结

可以看到，不论是存放或是加载一个弱应用变量都需要：

1. ```SideTable``` 哈希搜索一次；
2. ```weak_table_t``` 开放地址法哈希搜索一次；
3. ```weak_entry_t``` 同样是开放地址法哈希再搜索一次。

整套操作下来也不简单，但考虑到 iOS 5 那时的可用内存还是挺少的，估计为了能立刻回收释放的内存，苹果就选择这种时间换空间的方式来实现了。

## Swift4 之前的实现

接着我们来看 Swift 里的实现。在 Swift 的运行时里，被分配到堆上的对象都是 ```HeapObject``` 类型：

``` swift
/// The Swift heap-object header.
struct HeapObject {
  /// This is always a valid pointer to a metadata object.
  HeapMetadata const *metadata;

  SWIFT_HEAPOBJECT_NON_OBJC_MEMBERS;
  // FIXME: allocate two words of metadata on 32-bit platforms

#ifdef __cplusplus
  HeapObject() = default;

  // Initialize a HeapObject header as appropriate for a newly-allocated object.
  constexpr HeapObject(HeapMetadata const *newMetadata)
    : metadata(newMetadata)
    , refCounts(InlineRefCounts::Initialized)
  { }
#endif
};
```

```HeapMetadata``` 相当于 Objective-C 的 ```isa``` 字段，实际上两者也确实是可以互换的。随后的 ```SWIFT_HEAPOBJECT_NON_OBJC_MEMBERS``` 宏就是我们要找的：

``` swift
#define SWIFT_HEAPOBJECT_NON_OBJC_MEMBERS       \
      StrongRefCount refCount;                  \
      WeakRefCount weakRefCount
```

强弱引用计数都是这么直接的定义在里面了！可能是 Swift 的开发团队也意识到 OC 的实现方法已经相当过时和低效，从而重新设计了整套机制。新的实现里弱引用变量就是一个只存有目标对象地址的结构体。

``` c
struct WeakReference {
  uintptr_t Value;
};
```

初始化一个弱引用变得很直接，只是单纯地把目标对象的地址记录起来。

``` c
void swift::swift_weakInit(WeakReference *ref, HeapObject *value) {
  ref->Value = (uintptr_t)value | WR_NATIVE;
  SWIFT_RT_ENTRY_CALL(swift_unownedRetain)(value);
}
```

所以当需要把目标对象加载出来也很简单。下面的方法是 2015 年的[实现](https://github.com/apple/swift/blob/swift-2.2-SNAPSHOT-2015-12-01-b/stdlib/public/runtime/HeapObject.cpp#L636)（升级到 Swift4 之前只是为了处理多线程的情况而把一些存取操作换成了原子操作，基本意思还是一样）。

``` c
HeapObject *swift::swift_weakLoadStrong(WeakReference *ref) {
  auto object = ref->Value;
  if (object == nullptr) return nullptr;
  if (object->refCount.isDeallocating()) {
    swift_weakRelease(object);
    ref->Value = nullptr;
    return nullptr;
  }
  return swift_tryRetain(object);
}
```

Swift 的 zeroing 就是发生在访问弱引用的时候，如果目标对象正在被释放（已经被析构，但还没释放），那就置空这个引用，否则就尝试 retain 并且返回这对象。

再来看 ```swift_weakRelease``` 函数：

``` c
void swift::swift_weakRelease(HeapObject *object) {
  if (!object) return;

  if (object->weakRefCount.decrementShouldDeallocate()) {
    // Only class objects can be weak-retained and weak-released.
    auto metadata = object->metadata;
    assert(metadata->isClassObject());
    auto classMetadata = static_cast<const ClassMetadata*>(metadata);
    assert(classMetadata->isTypeMetadata());
    swift_slowDealloc(object, classMetadata->getInstanceSize(),
                      classMetadata->getInstanceAlignMask());
  }
}
```

只有当对象的弱引用数减少到为零时，这才把对象的内存真正给释放出来。

### 小结

1. 在 Swift 里苹果把弱引用的实现简化了，弱引用变量只保存指向目标对象的地址，并通过对象內的弱引用计数进行内存管理。
2. Swift 把对象的析构和释放时机进行了解耦，析构发生在强引用数为零时，只有强弱引用数都为零才会释放。
3. 使用弱引用时，runtime 会检查目标对象的状态，如果已经析构了就会执行 zeroing 操作。

## Swift4 后

虽然上面的方法简化了弱引用的实现和提高了存取效率，但却有一个很大的问题。如果对象的弱引用数一直不为零，那么对象占用的剩余内存就不会完全释放。这些死而不僵的对象还占用很多空间的话，累积起来也是对内存造成浪费。所以在 Swift4 以后，苹果再次加入 ```SideTable``` 的机制。

不过此 ```SideTable``` 跟 OC 的 ```SideTable``` 不一样，系统不再是把它作为全局对象使用。

新的 ```SideTable``` 是针对有需要的对象而创建，系统会为目标对象分配一块新的内存来保存该对象额外的信息。 因为这不是对象必须的内容，所以这个 ```SideTable``` 可有可无。对象会有一个指向 ```SideTable``` 的指针，同时 ```SideTable``` 也有一个指回原对象的指针。在实现上为了不额外多占用内存，目前只有在创建弱引用时，会先把对象的引用计数放到新创建的 ```SideTable``` 去，再把空出来的空间存放 ```SideTable``` 的地址，而 runtime 会通过一个标志位来区分对象是否有 ```SideTable```。在 [RefCount.h](https://github.com/apple/swift/blob/c262440e70896299118a0a050c8a834e1270b606/stdlib/public/SwiftShims/RefCount.h#L87-L107) 文件的注释里，Swift 开发团队也已经清晰地写了：

```
  Storage layout:

  HeapObject {
    isa
    InlineRefCounts {
      atomic<InlineRefCountBits> {
        strong RC + unowned RC + flags
        OR
        HeapObjectSideTableEntry*
      }
    }
  }

  HeapObjectSideTableEntry {
    SideTableRefCounts {
      object pointer
      atomic<SideTableRefCountBits> {
        strong RC + unowned RC + weak RC + flags
      }
    }
  }
```

虽然这些 ```SideTable``` 还是得等到最后一个弱引用被访问时才会释放，不过这样就大大缓解了内存浪费的问题，而且还能继续沿用 Swift 的弱引用机制，只不过现在弱引用变量指向的是对象的 ```SideTable```。最后，```SideTable``` 的实现也为以后加入更多特性提供了方便。

## 总结

Swift 弱引用机制的改进在提高效率的同时使得实现更加优雅。通过开源代码我们也能看到苹果在这方面是如何思考和设计的，怎样在时间和空间上进行取舍来实现需求。

## References

[weak 弱引用的实现方式](http://www.desgard.com/weak/)

[Swift Weak References](https://mikeash.com/pyblog/friday-qa-2015-12-11-swift-weak-references.html)

[Swift 4 Weak References](https://mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)

