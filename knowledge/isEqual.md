## 你真的了解`isEqual:`吗

> ​	面试的时候，常常被问道`isEqual:`与`==`有什么区别？
>
> ​	乍一看，很懵逼，不过这两个就是直接比较对象的内存地址的，在某些层面上来讲（比如`NSObject *`）则`isEqual:`与`==`是等价的
>
> ​	但是，系统给予了我们拓展实例方法`isEqual:`的权利，这就代表，我们可以使它变得与`==`不一样

Now，笔者给你贴出它的默认实现

```
** 摘自 libobjc4-781
- (BOOL)isEqual:(id)obj {
    return obj == self;
}
```



#### 那么有人就要问了：如何不一样呢？

```
现在我们来设想一个场景：
	在一个Key类型为对象的字典或者集合中，我们不单单的想要根据内存地址定义两个Key值相等。而是可能是`id`或者其他属性；又或者是两个合起来的属性，例如`id`与`orderNo`
```

那么，我们依据需求来编码。没错；首先你需要定义个对象，并重写它的`isEqual:`方法。你的模型类里的代码，大概率会是这样的：

```
@interface Model : NSObject <NSCopying>
@property (nonatomic, copy) NSString *Id;
@property (nonatomic, copy) NSString *orderNo;
+ (instancetype)modelWithId:(NSString *)Id orderNo:(NSString *)orderNo;
@end

@implementation Model

- (id)copyWithZone:(nullable NSZone *)zone{
    Model *m = [[Model allocWithZone:zone] init];
    m.Id = self.Id;
    m.orderNo = self.orderNo;
    return m;
}

+ (instancetype)modelWithId:(NSString *)Id orderNo:(NSString *)orderNo{
    Model *m = [[self alloc] init];
    m.Id = Id;
    m.orderNo = orderNo;
    return m;
}

- (BOOL)isEqual:(Model *)obj {
    if ([_Id isEqualToString:obj.Id] && [_orderNo isEqualToString:obj.orderNo])
        return YES;
    return [super isEqual:obj];
}

@end
```

好的，完成，那我们拿来试一下；

```
+ (void)test {
    NSMutableDictionary *dict = @{}.mutableCopy;
            
    Model *key1 = [Model modelWithId:@"id-1" orderNo:@"no-1"];
    Model *key2 = [Model modelWithId:@"id-2" orderNo:@"no-2"];
    dict[key1] = @"1";
    dict[key2] = @"2";
    
    
    Model *keySome = [Model modelWithId:@"id-1" orderNo:@"no-1"];
    
    id value = dict[keySome];
    
    NSLog(@"value is: %@", value);
}
```

那么不出意外的情况下，我们会得到打印为`value is: 1`；好的，我们cmd+r运行一下

```
2021-09-08 15:34:39.852541+0800 TestOC[79978:9867416] value is: (null)
```

%￥%！……！@！#！@！#；What the fucking going on?

竟然不是！~



#### 演示到这里，确实与我们想要的结果不一样

Why? 那么笔者在这里就不跟你们打哑谜了，公布结果：

```
- (NSUInteger)hash{
    return _Id.hash ^ _orderNo.hash;
}
```

那当我们`Model.m`中添加一段代码后，cmd+r

```
2021-09-08 15:39:18.229516+0800 TestOC[80032:9871272] value is: 1
```

我们得到了我们想要的结果



#### 为什么呢？接下里笔者就要带你们进入源码的世界了，没有兴趣者，可到这里就结束了

注意以下代码经过简化，感兴趣的话，可下载源码查看

> 我们来到`CF-1153.18`工程中找到
>
> ```
> const_any_pointer_t CFDictionaryGetValue(CFHashRef hc, const_any_pointer_t key) {
>     CFBasicHashBucket bkt = CFBasicHashFindBucket((CFBasicHashRef)hc, (uintptr_t)key);
>     return (0 < bkt.count ? (const_any_pointer_t)bkt.weak_value : 0);
> }
> ```
>
> 经过我们翻阅`CFDictionary`的源码，我们找到了根据key获取value的方法，该方法调用了`CFBasicHashFindBucket`，则我们接着找到他的实现

如下：`__CFBasicHashFindBucket`实现了多个不同的策略

```
CF_INLINE CFBasicHashBucket __CFBasicHashFindBucket(CFConstBasicHashRef ht, uintptr_t stack_key) {
    if (0 == ht->bits.num_buckets_idx) {
        CFBasicHashBucket result = {kCFNotFound, 0UL, 0UL, 0};
        return result;
    }
    if (ht->bits.indirect_keys) {
        switch (ht->bits.hash_style) {
        case __kCFBasicHashLinearHashingValue: return ___CFBasicHashFindBucket_Linear_Indirect(ht, stack_key);
        case __kCFBasicHashDoubleHashingValue: return ___CFBasicHashFindBucket_Double_Indirect(ht, stack_key);
        case __kCFBasicHashExponentialHashingValue: return ___CFBasicHashFindBucket_Exponential_Indirect(ht, stack_key);
        }
    } else {
        switch (ht->bits.hash_style) {
        case __kCFBasicHashLinearHashingValue: return ___CFBasicHashFindBucket_Linear(ht, stack_key);
        case __kCFBasicHashDoubleHashingValue: return ___CFBasicHashFindBucket_Double(ht, stack_key);
        case __kCFBasicHashExponentialHashingValue: return ___CFBasicHashFindBucket_Exponential(ht, stack_key);
        }
    }
    HALT;
    CFBasicHashBucket result = {kCFNotFound, 0UL, 0UL, 0};
    return result;
}
```

具体的策略会根据以下宏不同，而组装不同的代码逻辑

```
#define FIND_BUCKET_NAME		___CFBasicHashFindBucket_Linear
#define FIND_BUCKET_HASH_STYLE		1
#define FIND_BUCKET_FOR_REHASH		0
#define FIND_BUCKET_FOR_INDIRECT_KEY	0
#include "CFBasicHashFindBucket.m"
```

如下：截取了`___CFBasicHashFindBucket_Linear`策略下的代码

```
static
#if FIND_BUCKET_FOR_REHASH
CFIndex
#else
CFBasicHashBucket
#endif
FIND_BUCKET_NAME (CFConstBasicHashRef ht, uintptr_t stack_key
#if FIND_BUCKET_FOR_REHASH
, uintptr_t key_hash
#endif
) {
    uint8_t num_buckets_idx = ht->bits.num_buckets_idx;
    uintptr_t num_buckets = __CFBasicHashTableSizes[num_buckets_idx];
#if FIND_BUCKET_FOR_REHASH
    CFHashCode hash_code = key_hash ? key_hash : __CFBasicHashHashKey(ht, stack_key);
#else
    CFHashCode hash_code = __CFBasicHashHashKey(ht, stack_key);
#endif
    for (CFIndex idx = 0; idx < num_buckets; idx++) {
        uintptr_t curr_key = keys[probe].neutral;
        if (curr_key == 0UL) {
					////
        } else if (curr_key == ~0UL) {
					////
        } else {
						/** 遍历keys找到对应的key值*/
            if (curr_key == stack_key || ((!hashes || hashes[probe] == hash_code) && __CFBasicHashTestEqualKey(ht, curr_key, stack_key))) {
                COCOA_HASHTABLE_PROBING_END(ht, idx + 1);
#if FIND_BUCKET_FOR_REHASH
                CFIndex result = probe;
#else
						/** 取出key对应的value*/
                CFBasicHashBucket result;
                result.idx = probe;
                result.weak_value = __CFBasicHashGetValue(ht, probe);
                result.weak_key = curr_key;
                result.count = (ht->bits.counts_offset) ? __CFBasicHashGetSlotCount(ht, probe) : 1;
#endif
                return result;
            }
#endif
        }
    return result; // all buckets full or deleted, return first deleted element which was found
}

#undef FIND_BUCKET_NAME
#undef FIND_BUCKET_HASH_STYLE
#undef FIND_BUCKET_FOR_REHASH
#undef FIND_BUCKET_FOR_INDIRECT_KEY

```

则，可以看见，内部寻找的逻辑是通过`__CFBasicHashHashKey`找到的所谓了桶`buckets`

```
CF_INLINE CFHashCode __CFBasicHashHashKey(CFConstBasicHashRef ht, uintptr_t stack_key) {
    CFHashCode (*func)(uintptr_t) = (CFHashCode (*)(uintptr_t))CFBasicHashCallBackPtrs[ht->bits.__khas];
    CFHashCode hash_code = func ? func(stack_key) : stack_key;
    COCOA_HASHTABLE_HASH_KEY(ht, stack_key, hash_code);
    return hash_code;
}
```

而`    CFHashCode (*func)(uintptr_t) = (CFHashCode (*)(uintptr_t))CFBasicHashCallBackPtrs[ht->bits.__khas];`这段代码，实际上调用的就是

```
CFHashCode CFHash(CFTypeRef cf) {
    CFHashCode (*hash)(CFTypeRef cf) = __CFRuntimeClassTable[__CFGenericTypeID_inline(cf)]->hash; 
    if (NULL != hash) {
	return hash(cf);
    }
    if (CF_IS_COLLECTABLE(cf)) return (CFHashCode)_object_getExternalHash((id)cf);
    return (CFHashCode)cf;
}
```

其次，再遍历桶，找到对应的key

```
CF_INLINE Boolean __CFBasicHashTestEqualKey(CFConstBasicHashRef ht, uintptr_t in_coll_key, uintptr_t stack_key) {
    COCOA_HASHTABLE_TEST_EQUAL(ht, in_coll_key, stack_key);
    Boolean (*func)(uintptr_t, uintptr_t) = (Boolean (*)(uintptr_t, uintptr_t))CFBasicHashCallBackPtrs[ht->bits.__kequ];
    if (!func) return (in_coll_key == stack_key);
    return func(in_coll_key, stack_key);
}
```

而`Boolean (*func)(uintptr_t, uintptr_t) = (Boolean (*)(uintptr_t, uintptr_t))CFBasicHashCallBackPtrs[ht->bits.__kequ];`则真实的调用到了它：

```
Boolean CFEqual(CFTypeRef cf1, CFTypeRef cf2) {
    if (NULL != __CFRuntimeClassTable[__CFGenericTypeID_inline(cf1)]->equal) {
	return __CFRuntimeClassTable[__CFGenericTypeID_inline(cf1)]->equal(cf1, cf2);
    }
    return false;
}
```

则；我们总结出`iOS`编码中的`NSDictionary`的`-objectForKey:`的方法具体实现如下

>1、根据所传入`key`的hash值寻找存储的桶
>
>2、若hash冲突，则进而通过isEqual:这个方法，找到与之匹配的`key`值
>
>3、找到`key`后，取出`value`

