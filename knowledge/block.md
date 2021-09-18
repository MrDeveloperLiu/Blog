### ObjC之`Block`篇


> 本文部分代码摘自`libclosure`，感兴趣的同学请自行下载查看
>
> 探究之来源：
>
> ​	在`ObjC`中，我们经常会用到`Block`，但我们其实很少去探索它。正如大家都知道，这个一个可回调代码块，但其具体是什么，很多人回答不清。笔者在没有阅读源码之前，也是停留在知道`Block`实际上也可以说成是一个对象
>
> 具体分为 `NSStackBlock` 和 `NSGloablBlock` 和 `NSMallocBlock`

#### 在介绍之前呢，我想给大家分享一下，一段代码以及运行结果

```
extern void * _NSConcreteStackBlock[32];
extern void * _NSConcreteMallocBlock[32];
extern void * _NSConcreteAutoBlock[32];
extern void * _NSConcreteFinalizingBlock[32];
extern void * _NSConcreteGlobalBlock[32];

extern void * _NSConcreteWeakBlockVariable[32];


int main(int argc, const char * argv[]) {
    @autoreleasepool {

        NSLog(@"_NSConcreteStackBlock: %@", _NSConcreteStackBlock);
        NSLog(@"_NSConcreteMallocBlock: %@", _NSConcreteMallocBlock);
        NSLog(@"_NSConcreteGlobalBlock: %@", _NSConcreteGlobalBlock);
        
        NSLog(@"_NSConcreteAutoBlock: %@", _NSConcreteAutoBlock);
        NSLog(@"_NSConcreteFinalizingBlock: %@", _NSConcreteFinalizingBlock);
        
        NSLog(@"_NSConcreteWeakBlockVariable: %@", _NSConcreteWeakBlockVariable);
    }
    return 0;
}
```

> ```
> 2021-09-18 11:58:44.420565+0800 Testing[68577:1699520] _NSConcreteStackBlock: __NSStackBlock__
> 2021-09-18 11:58:44.421184+0800 Testing[68577:1699520] _NSConcreteMallocBlock: __NSMallocBlock__
> 2021-09-18 11:58:44.421254+0800 Testing[68577:1699520] _NSConcreteGlobalBlock: __NSGlobalBlock__
> 2021-09-18 11:58:44.421313+0800 Testing[68577:1699520] _NSConcreteAutoBlock: __NSAutoBlock__
> 2021-09-18 11:58:44.421371+0800 Testing[68577:1699520] _NSConcreteFinalizingBlock: __NSFinalizingBlock__
> 2021-09-18 11:58:44.421403+0800 Testing[68577:1699520] _NSConcreteWeakBlockVariable: __NSBlockVariable__
> ```

**Emmmmmmmmmmmm.... ** `这是什么` *????*

别急别急，莫慌，待我给大家贴出来源码

>```
>// the class for stack instances created by block expressions
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSStackBlock__ : NSObject
>@end
>```
>
>```
>// the class for Blocks copied under retain/release (non-GC) scenarios
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSMallocBlock__ : NSObject
>@end
>```
>
>```
>// the class for Blocks copied under GC scenarios
>// Note: even though we no longer support garbage collection, we leave this in for compatibility reasons.
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSAutoBlock__ : NSObject
>@end
>```
>
>```
>// inherit copy methods
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSFinalizingBlock__ : __NSAutoBlock__
>@end
>```
>
>```
>// the class for blocks in global scope
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSGlobalBlock__ : NSObject
>@end
>```
>
>```
>// The following class is not a Block but rather a wrapper object around a __block variable.
>//
>__attribute__((objc_nonlazy_class, visibility("hidden")))
>@interface __NSBlockVariable__ : NSObject
>{
>    //struct Block_byref, minus the duplicated isa
>    __strong struct Block_byref *forwarding;    // Compaction:  made strong so pointer can be updated.
>    int flags;//refcount;
>    int size;
>    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
>    void (*byref_destroy)(struct Block_byref *);
>    // The all important field; this forces a layout map that GC can peruse.
>    // The compiler, of course, doesn't see this and emits code using its own layout info.
>    /* __weak */ id containedObject;
>}
>
>@end
>```



#### Block实质

其实在以前，笔者所知的`Block`也是停留在`结构体指针`的范畴中，但这就很明显的从上面代码中得到结论：`Block`其实是`Objc`内建的隐藏性的类的**对象**。

那么`Block`总结一下：

* `__NSStackBlock__`：创建在栈区的
* `__NSMallocBlock__`：被拷贝到堆区的
* `__NSGlobalBlock__`：创建在全局区的代码段上的
* `__NSAutoBlock__`：在GC场景下复制的
* `__NSFinalizingBlock__`：继承自`__NSAutoBlock__`，实现了`-finalize`，同样GC场景下应用的
* 非Block类型-- `__NSBlockVariable__`：`__block`包装的对象

那么后两者**Block类型**不在本文的讨论范围之内 

ps: 很多朋友们仅仅知道Objc这门语言使用的是**引用计数**来管理内存，但其实其管理内存方式也有**GC(垃圾回收机制)**，只不过，**GC**机制不支持`iOS`开发



#### 创建Block

话不多数，上代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {

        int var = 1;
        
        id block1 = ^{};
        NSLog(@"block1: %@", block1);
        
        id block2 = ^{ var; };
        NSLog(@"block2: %@", block2);
        
        id block3 = [block2 copy];
        NSLog(@"block3: %@", block3);
    }
    return 0;
}
```

> ```
> 2021-09-18 14:02:25.235005+0800 Testing[68915:1729667] block1: <__NSGlobalBlock__: 0x100004038>
> 2021-09-18 14:02:25.235292+0800 Testing[68915:1729667] block2: <__NSStackBlock__: 0x7ffeefbff420>
> 2021-09-18 14:02:25.235340+0800 Testing[68915:1729667] block3: <__NSMallocBlock__: 0x10061a6e0>
> ```

本次测试程序，运行在`MRC`环境下，请忽略内存泄漏，我们主要讨论的问题是**Block**被创建在什么区

>分析：
>
>1、`block1` 就是在某个方法作用域中，创建的一个临时变量，我们可以看到控制台打印：则这种方式创建出来的Block，默认被创建于**全局区**
>
>2、`block2` 与 `block1`的不同之处是，`block2`内部捕获了一个**外部变量**，则它被放置到了**栈区**
>
>3、`block3`实际上就是对`block2`进行了一次`copy`操作，则他被放置到了内存的**堆区**

那我们将代码运行环境调整成`ARC`环境，则看到不同的打印

>```
>2021-09-18 14:10:19.226308+0800 Testing[68940:1733751] block1: <__NSGlobalBlock__: 0x100004038>
>2021-09-18 14:10:19.226651+0800 Testing[68940:1733751] block2: <__NSMallocBlock__: 0x100566c20>
>2021-09-18 14:10:19.226702+0800 Testing[68940:1733751] block3: <__NSMallocBlock__: 0x100566c20>
>```

> 则：	我们能够看到异同点在于`block2`上，笔者猜测，`ARC`环境下，编译器对**Block**赋值时，自动进行了**copy**操作



#### 那么面试中，大家可能会遇到一个问题

问：**Block**用什么修饰？

答：`ARC`环境下用 **copy** 与 **strong** 均可，但是官方提醒开发者，**最好**用**copy**



我们来探究一下`MRC`下，对**Block**用`retain`会有什么后果；

则我们对代码做如下改动：

```
id block3 = [block2 retain];
NSLog(@"block3: %@", block3);

2021-09-18 14:15:43.125541+0800 Testing[68968:1737070] block3: <__NSStackBlock__: 0x7ffeefbff420>
```

Oops，我们可以看到`MRC`环境下，**retain**操作竟然什么都没有做，那这么导致的后果就是，出了创建**Block**的方法作用域后，则属性/实例变量可能指向一块**僵尸对象**，为代码埋下了崩溃的隐患

为了证明我们的控制台打印结果与猜想，我们在源码中找到如下代码

```
@implementation __NSStackBlock__
- (id)retain {
    return self;
}
@end
```

证实了我们的猜想，实际上`__NSStackBlock__`的**retain**方法，什么都没有做，仅仅返回了自己



#### Block的Copy操作

其实我们在对**Block**进行**copy**时，则是调用到了`void *_Block_copy(const void *arg)`

那么我们探究其实现：

```
// Copy, or bump refcount, of a block.  If really copying, call the copy helper if present.
void *_Block_copy(const void *arg) {
    struct Block_layout *aBlock;

    if (!arg) return NULL;
    
    // The following would be better done as a switch statement
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }
    else {
        // Its a stack block.  Make a copy.
        size_t size = Block_size(aBlock);
        struct Block_layout *result = (struct Block_layout *)malloc(size);
        if (!result) return NULL;
        memmove(result, aBlock, size); // bitcopy first
#if __has_feature(ptrauth_calls)
        // Resign the invoke pointer as it uses address authentication.
        result->invoke = aBlock->invoke;

#if __has_feature(ptrauth_signed_block_descriptors)
        if (aBlock->flags & BLOCK_SMALL_DESCRIPTOR) {
            uintptr_t oldDesc = ptrauth_blend_discriminator(
                    &aBlock->descriptor,
                    _Block_descriptor_ptrauth_discriminator);
            uintptr_t newDesc = ptrauth_blend_discriminator(
                    &result->descriptor,
                    _Block_descriptor_ptrauth_discriminator);

            result->descriptor =
                    ptrauth_auth_and_resign(aBlock->descriptor,
                                            ptrauth_key_asda, oldDesc,
                                            ptrauth_key_asda, newDesc);
        }
#endif
#endif
        // reset refcount
        result->flags &= ~(BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING);    // XXX not needed
        result->flags |= BLOCK_NEEDS_FREE | 2;  // logical refcount 1
        _Block_call_copy_helper(result, aBlock);
        // Set isa last so memory analysis tools see a fully-initialized object.
        result->isa = _NSConcreteMallocBlock;
        return result;
    }
}
```

整个方法即为`[block copy]`

> 分析：
>
> 1、首先的判断分支block会判断其flags标志位，如果含有BLOCK_NEEDS_FREE 或者 BLOCK_IS_GLOBAL 时，最终返回的都是所传的入参
>
> 2、其次，则断定，该block实际上就是栈block。则对其进行下一步操作
>
> 3、计算Block占用的内存大小，初始化一个结构体指针，将原Block的所有参数，拷贝到新的内存地址中
>
> 4、拷贝调用函数
>
> 5、拷贝Block的描述descriptor
>
> 6、更改其isa指针为_NSConcreteMallocBlock

则这也就解释了，为什么我们对`__NSStackBlock__`进行`copy`操作后，就变成了`__NSMallocBlock__`



#### ARC环境下的编译器优化Block暂不做调研，有兴趣者可自行查阅资料
