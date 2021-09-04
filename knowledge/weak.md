## 浅谈`__weak`的原理

注意：以下代码经过简化，有兴趣可自行下载libobjc源码阅读

* 一、Objc通过调用以下方法
```
id objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}
void objc_destroyWeak(id *location)
{
    (void)storeWeak<DoHaveOld, DontHaveNew, DontCrashIfDeallocating>
        (location, nil);
}
```
用来实现存储或者清空`__weak`修饰的指针变量

* 二、那么`storeWeak `函数到底做了什么

```
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    // Clean up old value, if any.
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected
    }

    return (id)newObj;
}

```
> 2.1 那我们先看`weak_unregister_no_lock `的方法实现
```
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
    }
}

static void remove_referrer(weak_entry_t *entry, objc_object **old_referrer)
{
    size_t begin = w_hash_pointer(old_referrer) & (entry->mask);
    size_t index = begin;
    size_t hash_displacement = 0;
    while (entry->referrers[index] != old_referrer) {
        index = (index+1) & entry->mask;
    }
    entry->referrers[index] = nil;
    entry->num_refs--;
}

```
那么从这里就清晰的看到，实际上将`weak`修饰的指针变量清空时，objc底层所做的操作其实就是通过`w_hash_pointer`函数，查找到了对应的`weak_entry_t`存储结构，通过遍历取得存储`weak`指针的索引，最终清空它

> 2.2 其次我们再来看一下`weak_register_no_lock`的方法实现
```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }
    return referent_id;
}

static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
    size_t index = begin;
    while (entry->referrers[index] != nil) {
        index = (index+1) & entry->mask;
    }
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
```
OK到这里就很清晰了；同理`Objc`在存储`weak`类型的变量时，也是通过`w_hash_pointer`查找到了对应的`weak_entry_t`存储结构，找到一个空位置，塞进去。那么这么一来，`weak`的存储就变得特别清晰了


* 三、接下来我们来认识几个数据结构

> `SideTable` 它实际上存储了`weak_table` 定义如下： 
```
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;
};

```

> `StripedMap` 它实际上基于`SideTable` 建立的一个全局的`array`结构，内部使用哈希函数`indexForPointer`进行查找对应`SideTable`，其定义如下：
```
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

 public:
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
};

```

> `weak_table_t` 保存了具体`weak_entry_t`的一个引用表，其内部的`weak_entry_t`是个数组结构
```
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

> `weak_entry_t` 我们可以看到最终存储`weak`指针地址的地方则是它，实际上就是一个4容量的静态/动态数组
```
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};

```

