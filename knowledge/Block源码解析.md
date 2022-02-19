### Block_copy

#### 调用Block_copy时发生了什么

对block进行copy时，实际上会调用到如下源码：

1、若Block的标记位flag含有BLOCK_NEEDS_FREE时，调用copy操作，仅仅返回自身，并且改变标记位的值

2、若Block的标记位中判断该BLOCK_IS_GLOBAL（表示该Block为全局的block），仅仅返回自身

3、以上情况都不是的话，则认为所传入的Block即为栈上的Block，则对其进行复制到堆区操作，同时更改Block对象的isa指针为_NSConcreteMallocBlock

```
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

那么我们对第三步（将Block由栈区赋值到堆区的操作）进行展开：

在Block被复制到堆区的同时，则会对Block内部的copy指针进行一次调用，用来将捕获的变量复制一份

```
static void _Block_call_copy_helper(void *result, struct Block_layout *aBlock)
{
    if (auto *pFn = _Block_get_copy_function(aBlock))
        pFn(result, aBlock);
}
```

#### Copy指针在何时调用

那么我们摘自[iOS Block](http://events.jianshu.io/p/ad81b2df2354)中的一段编译后的cpp文件来看

```
int main(int argc, const char *argv[])
{
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;

        MJPerson *person = ((MJPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((MJPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("MJPerson"), sel_registerName("alloc")), sel_registerName("init"));
        ((void (*)(id, SEL, int))(void *)objc_msgSend)((id)person, sel_registerName("setAge:"), 2);

        MJBlock myBlock = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, person, 570425344));
    }
    return 0;
}

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  MJPerson *__strong person;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, MJPerson *__strong _person, int flags=0) : person(_person) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->person, (void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);}
```

我们看到在main函数中，作者应该是初始化MJPerson类的对象后，使用Block捕获了person对象，其中`__main_block_desc_0 `中的`copy`指针则被编译器指向了`static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src)`这个函数

那么，仔细观察一下，该函数在做什么：`_Block_object_assign((void*)&dst->person, (void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/) `

```
void _Block_object_assign(void *destArg, const void *object, const int flags) {
    const void **dest = (const void **)destArg;
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_OBJECT:
        /*******
        id object = ...;
        [^{ object; } copy];
        ********/

        _Block_retain_object(object);
        *dest = object;
        break;

      case BLOCK_FIELD_IS_BLOCK:
        /*******
        void (^object)(void) = ...;
        [^{ object; } copy];
        ********/

        *dest = _Block_copy(object);
        break;
    
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        /*******
         // copy the onstack __block container to the heap
         // Note this __weak is old GC-weak/MRC-unretained.
         // ARC-style __weak is handled by the copy helper directly.
         __block ... x;
         __weak __block ... x;
         [^{ x; } copy];
         ********/

        *dest = _Block_byref_copy(object);
        break;
        
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
        /*******
         // copy the actual field held in the __block container
         // Note this is MRC unretained __block only. 
         // ARC retained __block is handled by the copy helper directly.
         __block id object;
         __block void (^object)(void);
         [^{ object; } copy];
         ********/

        *dest = object;
        break;

      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        /*******
         // copy the actual field held in the __block container
         // Note this __weak is old GC-weak/MRC-unretained.
         // ARC-style __weak is handled by the copy helper directly.
         __weak __block id object;
         __weak __block void (^object)(void);
         [^{ object; } copy];
         ********/

        *dest = object;
        break;

      default:
        break;
    }
}
```

这一长串又是什么... （其实不难）；我们先来熟悉一下一些枚举值以及一些静态函数

```
enum {
    // see function implementation for a more complete description of these fields and combinations
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...  // 对象类型
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable //block类型
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable //__block 修饰的变量
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers //__weak 修饰的变量
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines. //__block 修饰的block类型
};

enum {
    BLOCK_ALL_COPY_DISPOSE_FLAGS = 
        BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_BLOCK | BLOCK_FIELD_IS_BYREF |
        BLOCK_FIELD_IS_WEAK | BLOCK_BYREF_CALLER
};


static void _Block_retain_object_default(const void *ptr __unused) { }

static void _Block_release_object_default(const void *ptr __unused) { }

static void _Block_destructInstance_default(const void *aBlock __unused) {}

static void (*_Block_retain_object)(const void *ptr) = _Block_retain_object_default;
static void (*_Block_release_object)(const void *ptr) = _Block_release_object_default;
static void (*_Block_destructInstance) (const void *aBlock) = _Block_destructInstance_default;

```

其实源码注释以及很全了：以下情况是对变量捕获的情况做出了分类展开

case 1、当Block内部捕获的是一个对象类型的时，会调用`_Block_retain_object(object)`那么不难看出，实际上在Block内部捕获到对象类型的同时，Block中指向copy函数指针中，会对对象进行一次retain操作；那么实际上是什么含义呢？？？没错，此处则是造成Block循环引用的`罪魁祸首`

case 2、当Block内部捕获的是一个Block时，则会调用`_Block_copy(object);`，对Block进行一次copy操作

case 3、当Block内部捕获的是一个被`__weak __block`修饰的变量时，则会调用`_Block_byref_copy(object)`；在此方法中，会对`__block`修饰的变量，根据标记位的参数进行复制到堆区的操作

case 4、当Block内部捕获的是一个被`__block`修饰的Block时，do nothing，并不会copy任何值

case 5、当Block内部捕获的是一个被`__weak __block`修饰的Block时，do nothing，并不会copy任何值

那么我们依然摘自上文提及的作者中的代码段来看

```
int main(int argc, char *argv[])
{
    NSString *appDelegateClassName;
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
        appDelegateClassName = NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class")));
    //右边是在给__Block_byref_person_0结构体初始化，相较于用"__block"来修饰"普通数据类型"的"auto变量",这里的结构体里多了copy和dispose函数，而在自赋值初始化过程中，__Block_byref_id_object_copy_131和__Block_byref_id_object_dispose_131函数内部调用的也是_Block_object_assign函数，这不过这个结构体里的copy和dispose用来管理__Block变量，而这整个结构体因为包装成对象所以当被拷贝到堆上时是强引用给block。
        __attribute__((__blocks__(byref))) __Block_byref_person_0 person = {(void*)0,(__Block_byref_person_0 *)&person, 33554432, sizeof(__Block_byref_person_0), __Block_byref_id_object_copy_131, __Block_byref_id_object_dispose_131, ((MJPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((MJPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("MJPerson"), sel_registerName("alloc")), sel_registerName("init"))};
        void(*myBlobk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_person_0 *)&person, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)myBlobk)->FuncPtr)((__block_impl *)myBlobk);
    }
    return UIApplicationMain(argc, argv, __null, appDelegateClassName);
}

其中__main_block_impl_0如下
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_person_0 *person; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_person_0 *_person, int flags=0) : person(_person->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

其中__block_impl如下
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

其中__main_block_desc_0如下
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

其中__Block_byref_person_0如下
struct __Block_byref_person_0 {
  void *__isa;
__Block_byref_person_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);//唯一不同点就在这儿，desc里面的copy和dispose管理的是外界捕获的值的内存也就是__Block_byref_person_0结构体的内存。而__Block_byref_person_0结构体里的copy和dispose管理的是__Block_byref_person_0结构体里捕获的真正用到的对象的内存。
    //如果是普通类型auto，即便是加了__Block，__Block_byref_person_0结构体里也不会有copy和dispos的，只有desc里会有。
 void (*__Block_byref_id_object_dispose)(void*);//唯一不同点就在这儿，desc里面的copy和dispose管理的是外界捕获的值的内存也就是__Block_byref_person_0结构体的内存。而__Block_byref_person_0结构体里的copy和dispose管理的是__Block_byref_person_0结构体里的对象的内存。
 MJPerson *__strong person; //可以看到person是强引用
};

其中__main_block_func_0如下
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
     __Block_byref_person_0 *person = __cself->person; // bound by ref

    (person->__forwarding->person) = __null;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_8k_pf6sbvtj611_l7bnqctfq_km0000gn_T_main_315fcb_mi_0,(person->__forwarding->person));
 }

//结构体里的copy和dispose函数调用方法
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```

则看一下 `_Block_byref_copy`的实现

```
static struct Block_byref *_Block_byref_copy(const void *arg) {
    struct Block_byref *src = (struct Block_byref *)arg;

    if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) {
        // src points to stack
        struct Block_byref *copy = (struct Block_byref *)malloc(src->size);
        copy->isa = NULL;
        // byref value 4 is logical refcount of 2: one for caller, one for stack
        copy->flags = src->flags | BLOCK_BYREF_NEEDS_FREE | 4;
        copy->forwarding = copy; // patch heap copy to point to itself
        src->forwarding = copy;  // patch stack to point to heap copy
        copy->size = src->size;

        if (src->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) {
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            struct Block_byref_2 *src2 = (struct Block_byref_2 *)(src+1);
            struct Block_byref_2 *copy2 = (struct Block_byref_2 *)(copy+1);
            copy2->byref_keep = src2->byref_keep;
            copy2->byref_destroy = src2->byref_destroy;

            if (src->flags & BLOCK_BYREF_LAYOUT_EXTENDED) {
                struct Block_byref_3 *src3 = (struct Block_byref_3 *)(src2+1);
                struct Block_byref_3 *copy3 = (struct Block_byref_3*)(copy2+1);
                copy3->layout = src3->layout;
            }

            (*src2->byref_keep)(copy, src);
        }
        else {
            // Bitwise copy.
            // This copy includes Block_byref_3, if any.
            memmove(copy+1, src+1, src->size - sizeof(*src));
        }
    }
    // already copied to heap
    else if ((src->forwarding->flags & BLOCK_BYREF_NEEDS_FREE) == BLOCK_BYREF_NEEDS_FREE) {
        latching_incr_int(&src->forwarding->flags);
    }
    
    return src->forwarding;
}
```

若 标记位 flag 中含有 BLOCK_REFCOUNT_MASK 标记，则执行对原对象的内存拷贝，拷贝其所有信息

#### 释放操作

上一节讲到的拷贝操作，那么：释放也是同理，Block内部都会根据标记位不同，进行必要的释放，逻辑较为简单，不进行展开

```
void _Block_object_dispose(const void *object, const int flags) {
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        // get rid of the __block data structure held in a Block
        _Block_byref_release(object);
        break;
      case BLOCK_FIELD_IS_BLOCK:
        _Block_release(object);
        break;
      case BLOCK_FIELD_IS_OBJECT:
        _Block_release_object(object);
        break;
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        break;
      default:
        break;
    }
}

void _Block_release(const void *arg) {
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    if (!aBlock) return;
    if (aBlock->flags & BLOCK_IS_GLOBAL) return;
    if (! (aBlock->flags & BLOCK_NEEDS_FREE)) return;

    if (latching_decr_int_should_deallocate(&aBlock->flags)) {
        _Block_call_dispose_helper(aBlock);
        _Block_destructInstance(aBlock);
        free(aBlock);
    }
}

static void _Block_byref_release(const void *arg) {
    struct Block_byref *byref = (struct Block_byref *)arg;

    // dereference the forwarding pointer since the compiler isn't doing this anymore (ever?)
    byref = byref->forwarding;
    
    if (byref->flags & BLOCK_BYREF_NEEDS_FREE) {
        int32_t refcount = byref->flags & BLOCK_REFCOUNT_MASK;
        os_assert(refcount);
        if (latching_decr_int_should_deallocate(&byref->flags)) {
            if (byref->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) {
                struct Block_byref_2 *byref2 = (struct Block_byref_2 *)(byref+1);
                (*byref2->byref_destroy)(byref);
            }
            free(byref);
        }
    }
}

```
