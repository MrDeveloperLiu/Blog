## 浅谈`autoreleasepool`的原理

注意：以下代码经过简化，有兴趣可自行下载libobjc源码阅读

```
As一名iOS开发工程师本文需要你带着问题去阅读: 
1、你了解ARC下，对象是如何自动释放的？
2、对象是何时自动释放的？
3、哨兵对象的是什么？
4、苹果工程师应用了什么数据结构来保存自动释放池，以及自动释放池的嵌套问题
```

#### 写在开头

​	我们回忆一下，在`MRC`时代，我们写代码的姿势：

```
- (id)createModel:(NSDictionary *json) cls:(Class)cls{
	id some = [[cls alloc] init];
	//prase
	return [some autorelease];
}
```

Oh mother fucker，每当我们看到一个`alloc`字眼，免不了的都是这种诸如`release`、`autorelease`的关键字跟着，更有甚者，当我们对某个对象发送`retain`、`copy`等关键字时，我们也需要显示的调用`释放`命令。

这其实很不友好：

其一，`代码量大` 无论我们做什么，我们需要的一个对象，对象就会有生命周期，初始化/拷贝/持有等动作，都会使得对象的引用计数加一，那么如果我们想要不造成内存泄漏，就要手动管理其生命周期，而这种`release`的操作无疑给代码量带来了巨大的贡献

```
ps: 如果我们是代码量为kpi评定的话，那老子确实愿意去写mrc下的代码
```

其二，`容易忘记` 很简单，当我们写某个方法时，对应的变量忘记`release`



#### 进入正题

首先在我们进入正题之前，我们先来看一下原理，并认识一个数据结构

1、每一个自动释放池其实对应的是一个`AutoreleasePoolPage`，每一页的大小为`4096字节`，为`4MB`存储空间，`objc`通过在主线程`Runloop`中注册了观察者，监听状态变更事件
2、释放池创建时机：`kCFRunLoopEntry`调用了`objc_autoreleasePoolPush`用来创建一个表
3、释放池循环时机：`kCFRunLoopBeforeWaiting`分别调用`objc_autoreleasePoolPop`与`objc_autoreleasePoolPush`用来释放旧的释放池，并创建新的释放池
4、释放池销毁时机：`kCFRunLoopExit`调用了`objc_autoreleasePoolPop`，用来释放最后一次创建的释放池



```
自动释放池的声明如下：
class AutoreleasePoolPage : private AutoreleasePoolPageData {.....}
struct AutoreleasePoolPageData
{
	__unsafe_unretained id *next; * 栈顶对象指针 *
	pthread_t const thread; * 存放的线程数据 *
	AutoreleasePoolPage * const parent; * 双链表的节点 *
	AutoreleasePoolPage *child; * 双链表的节点 *
	.....
}

释放池的创建于销毁源码如下：
void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

到这里，我们对`AutoreleasePool`的工作原理就有了一个初步的认识



#### 哨兵对象

```
#   define POOL_BOUNDARY nil
```

那么我们通过阅读源码中`AutoreleasePoolPage::push()`的实现可以看到，每当我们需要创建一张新的`AutoreleasePoolPage`时，我们都会传入一个对象为`POOL_BOUNDARY`，其值为`nil`但是是有实际的内存地址的一个对象，我们称之为`哨兵对象`

```
static inline void *push() 
{
    id *dest;
    if (slowpath(DebugPoolAllocation)) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    ASSERT(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

那么根据`push`的实现，相信一大波开发者都会卡在什么是`slowpath`上。其实吧，我们可以不用纠结于它，只当它是个一个判断的分支条件即刻，从声明上看来，实际上可能就是一个实现的快速路径与慢速路径



##### 插曲之slowpath & fastpath

其实关于`slowpath`还有一个与之对应的`fastpath`，定义如下

```
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
```

实际上他俩就有由编译器优化的代码，目的是为了告诉编译器帮助开发者优化一下代码逻辑，其具体作用为告诉编译器，**`最有可能的是什么`**，比如：

```
void testing(BOOL needed)
{
	if (needed) {	
		case 1
	}else{	
		case 2
	}
}
```

那么如果`needed大概率为NO`，则需要走到`case 2`则需要经历一次判断流程，则增加了代码运行时间。那么如果我们将`if (needed)`更改为`if (fastpath(needed))`，那么代码将从编译器层面为我们优化代码，如果优化的结果如下

```
void testing(BOOL needed)
{
	if (!needed) {	
		case 2
	}else{	
		case 1
	}
}
```

那么，是不是从源码层面则就加速了进入了`case 2`的判断流程



##### `autoreleaseFast` 分析

OK，我们回到正题；那么从`void *push() `中源码得知，大概率我们会走到`autoreleaseFast`的分支中

```
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

那么由此可以猜出来，苹果工程师的做法是:

1、如果当前的page不满，则像当前的page中添加一个哨兵对象

2、如果当前page存在（此时必然是page已满的情况）：

会新创建出一个自动释放池来，并将新创建出来的释放池设置为当前的`hotpage`，并添加上去一个哨兵对象

```
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    ASSERT(page == hotPage());
    ASSERT(page->full()  ||  DebugPoolAllocation);
    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```

3、如果都不是（此时必然是当前没有存在一个自动释放池）：

a：并不会直接创建出一个释放池，而是首先会走到`setEmptyPoolPlaceholder`往线程数据中写入一个键值对`AUTORELEASE_POOL_KEY: EMPTY_POOL_PLACEHOLDER`

b：如果当前线程数据中存在`AUTORELEASE_POOL_KEY`的值为`EMPTY_POOL_PLACEHOLDER`则才会创建出一个新的释放池，并且插入一个哨兵对象后，再插入其他的对象

```
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
        pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        return setEmptyPoolPlaceholder();
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);

    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }

    // Push the requested object or pool.
    return page->add(obj);
}

```

到这里，ARC下`autoreleasepool`如何记录着每一个对象的地址我们就清楚了



#### `OKay`那么我又一个疑问

> 当我们写下如下代码时
>
> ```
> @autoreleasepool {
> 	@autoreleasepool {
> 		@autoreleasepool {
> 			//code
> 		}
> 	}
> }
> ```
>
> 苹果如何保证双向链表可以正确的释放到最外层的释放池的呢？

带着这个疑问，我们将慢慢的进入到本节讲解的收尾环节

> 首先，这种嵌套的问题解决方案是什么？
>
> 我们很容易联想到`树`结构，按照正常的思路来说，最起码这种带有嵌套结构的代码，起码应该是一个树形结构吧
>
> 树根 -> [子节点0，子节点1] 
>
> [子节点0] -> [子节点]
>
> [子节点1] -> [子节点]
>
> 
>
> 其次，我们提到了这么久的哨兵对象，好像没有什么特殊的意义吧，到现在为止，我们所知的哨兵对象实际上就是一个值为`nil`地址是真实值的对象



#### 释放对象

我们先来看代码

```
static inline void pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;
    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        page = hotPage();
        if (!page) {
            return setHotPage(nil);
        }
        page = coldPage();
        token = page->begin();
    } else {
        page = pageForPointer(token);
    }
    stop = (id *)token;
    return popPage<false>(token, page, stop);
}

static inline void setHotPage(AutoreleasePoolPage *page) 
{
    if (page) page->fastcheck();
    tls_set_direct(key, (void *)page);
}

static inline AutoreleasePoolPage *coldPage() 
{
    AutoreleasePoolPage *result = hotPage();
    if (result) {
        while (result->parent) {
            result = result->parent;
            result->fastcheck();
        }
    }
    return result;
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    ASSERT(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}
```

> pop到指定对象
> 若当前的token为EMPTY_POOL_PLACEHOLDER：则判断表有没有被初始化，没有则清空线程数据，有则取到最上层的表的起始位置
> 若当前的token为其他对象：则根据对象地址，找到对应的表

`popPage`实际上是一个模板方法

```
template<bool allowDebug> static void popPage(void *token, AutoreleasePoolPage *page, id *stop)
{
    page->releaseUntil(stop);
}

void releaseUntil(id *stop) 
{
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();
        // fixme I think this `while` can be `if`, but I can't prove it
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }
        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();
        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}
```

> 而后会调用 `page->releaseUntil(stop)` 则，该方法会一直后移`next`指针，直到`next`指针为`stop`，将非哨兵对象全部调用`objc_release`释放，同时将每个栈顶的对象指针地址设置为`static uint8_t const SCRIBBLE = 0xA3;`



#### 最后

我们来解答`autoreleasepool`的嵌套问题，apple正是通过双向链表+哨兵对象来解决释放池嵌套释放池的操作

每当调用`@autoreleasepool`时，系统并不一定会创建新的释放池，而是通过插入一个哨兵对象，或者占位对象来完成该操作

不得不赞扬apple工程师的智慧

#### Ths
