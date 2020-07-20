---
title: Runtime原理探究
date: 2020-07-20 10:39:44
tags:

typora-copy-images-to: upload
---

```
原计划一周一个模块，但是看到 runTime 这里持续时间有点长，底层内容很多啊，原因还是之前基础太差，菜的抠脚。。。还有整理完 block 后再想 block hook 应该怎么实现，瞎搞了一通最后找到了开源的库 `blockHook`，看了几天的源码发现底层掌握的还是不牢，其中包括 runtime 部分，先整理完 runtime 部分然后再回头重新分析 blockHook。
```



### 1、isa 详解

在 arm64 结构之前，isa 就是一个普通指针，存储着 Class、Meta-Class 对象的内存地址。

从 arm64 结构开始，对 isa 进行了优化，变成了一个共用体（union）结构，使用位域来存储更多信息

更多信息是指 isa 除了存储对象内存地址外还存储了其他的信息：

```objective-c
union isa_t 
{
    Class cls;
    uintptr_t bits;
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL  
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33;
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
    };
};
```

- nonpointer：0 代表普通指针，存储着 Class、Meta-Class 对象的内存地址，1 代表优化过，使用位域存储更对信息
- has_assoc：是否有设置过关联对象，如果没有释放时更快
- has_cxx_dtor：是否有 C++ 的析构函数（.cxx_destruct），如果没有释放时更快
- shiftcls：存储着 Class、Meta-Class 对象的内存地址信息
- magic：用于在调试时分辨对相关是否完成初始化
- weakly_referenced：是否有被弱引用指向过，如果没有，释放更快
- deallocating： 对象是否正在释放
- has_sidetable_rc：引用计数器是否过大无法存储在isa中。1标识引用计数会存储在side table的类属性中
- extra_rc：里面存储的值是引用计数器减1

这里可以简单看一下实例对象 isa 指针里面存储的类，我们这里只研究 arm64 架构，可以看到上面有 ISA_MASK，这个是用来取 Class、Meta-Class 对象的内存地址的，来个小实验：

```
NSLog(@"--- %p", [ViewController class]);

结果:
--- 0x1000b1cf0

再来打印 self 的 isa：
(lldb) p/x self->isa
(Class) $0 = 0x000005a1000b1cf7 ViewController

可以看到上面的两个地址是不一样的，根据上面的结构，我们使用 ISA_MASK 来取一下 isa 中关于对象的内存地址：
(lldb) p/x 0x000005a1000b1cf7 & 0x0000000ffffffff8ULL
(unsigned long long) $1 = 0x00000001000b1cf0

由上可以看到，由 isa 取出来的地址和 ViewController class 类的真实地址是一样的。

```



关于上述标记位中更快释放，可以再 objc4 里面看到源码：

```objective-c
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?

    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}

id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}

void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
```

如上，在 dealloc 时会分别获取是否有关联对象、析构函数、弱指针等标记位，如果没有是直接释放的，否则会调用相关的方式释放对应的资源。

### 2、Class 结构

```objective-c
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

bits & FAST_DATA_MASK 可以拿到 class_rw_t

struct class_rw_t {
    uint32_t flags;
    uint32_t version;
    const class_ro_t *ro;
    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    Class firstSubclass;
    Class nextSiblingClass;
    char *demangledName;
}

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;
    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};

```

class_rw_t 里面的 methods、properties、protocols 是二维数组，是可读可写的，包含类的初始内容、分类的内容。

```
class method_array_t : 
	public list_array_tt<method_t, method_list_t> 
{
    typedef list_array_tt<method_t, method_list_t> Super;
};
```

Class_ro_t 里面的 baseMethodList、baseProtocols、ivars、baseProperties 是一维数组，是只读的，包含了类的初始信息。



下面来看下 method_t 的结构：

```objective-c
struct method_t {
    SEL name;	// 函数名
    const char *types;	// 编码（返回值类型、参数类型）
    IMP imp; // 指向函数的指针（函数地址）

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

IMP 代表函数的具体实现：

`typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);`

SEL 代表方法\函数名，一般称作选择器，底层结构和 char* 类型

- 可以通过 @selector() 和 sel_registerName() 获得
- 可以通过 sel_getName() 和 NSStringFromSelector() 转成字符串
- 不同类中相同名字的方法，所对应的方法选择器是相同的

`typedef struct objc_selector *SEL;`

types 是包含了函数返回值、参数编码的字符串



### 3、方法缓存

Class 内部结构用有个方法缓存（cache_t），用散列表（哈希表）来缓存曾经调用过的方法，可以提高方法的查找速度

```objective-c
struct bucket_t {
private:
    cache_key_t _key; // SEL 作为 key
    IMP _imp;	// 函数的内存地址

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
};

struct cache_t {
    struct bucket_t *_buckets; // 散列表
    mask_t _mask;	// 散列表长度 - 1
    mask_t _occupied; // 已经缓存的方法数量

public:
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();

    mask_t capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();

    static size_t bytesForCapacity(uint32_t cap);
    static struct bucket_t * endMarker(struct bucket_t *b, uint32_t cap);

    void expand();
    void reallocate(mask_t oldCapacity, mask_t newCapacity);
    struct bucket_t * find(cache_key_t key, id receiver);

    static void bad_cache(id receiver, SEL sel, Class isa) __attribute__((noreturn));
};
```

再来看下获取 cache 的实现：

```objective-c
// 如果 i为0则从mask开始查找（mask为表长度-1），如果i不为空则从i-1开始查找
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}

bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0);

    bucket_t *b = buckets();
    mask_t m = mask();
    mask_t begin = cache_hash(k, m);
    mask_t i = begin;
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}
```

散列表扩容，从源码可以看到，每次扩容都是原来容积的两倍，

```objective-c
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}
```



### 4、objc_msgSend 执行流程

#### 第一个阶段 - 消息发送

看一下下面的经典调用图片

![oc-class](https://raw.githubusercontent.com/jueying-xiangfeng/jueying-xiangfeng.github.io/hexo/assets/oc-class.png)

下面看下消息发送的流程：

![消息发送](https://raw.githubusercontent.com/jueying-xiangfeng/jueying-xiangfeng.github.io/hexo/assets/%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81.png)



#### 第二个阶段 - 动态方法解析

调用流程如下：

![动态方法解析](https://raw.githubusercontent.com/jueying-xiangfeng/jueying-xiangfeng.github.io/hexo/assets/%E5%8A%A8%E6%80%81%E6%96%B9%E6%B3%95%E8%A7%A3%E6%9E%90.png)

这里可以从 objc 的源码里面查到（lookUpImpOrForward 方法）：

```
if (resolver  &&  !triedResolver) {
    runtimeLock.unlockRead();
    _class_resolveMethod(cls, sel, inst);
    runtimeLock.read();
    // Don't cache the result; we don't hold the lock so it may have 
    // changed already. Re-do the search from scratch instead.
    triedResolver = YES;
    goto retry;
}

// No implementation found, and method resolver didn't help. 
// Use forwarding.

imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);
```

可以看到标记位 triedResolver 和我们流程图里面对应。

#### 第三个阶段 - 消息转发

看下流程图：

![消息转发](https://raw.githubusercontent.com/jueying-xiangfeng/jueying-xiangfeng.github.io/hexo/assets/%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91.png)

由于消息转发阶段没有开源，这里在网上扒出大神整理的伪代码：

```
// 伪代码
int __forwarding__(void *frameStackPointer, int isStret) {
    id receiver = *(id *)frameStackPointer;
    SEL sel = *(SEL *)(frameStackPointer + 8);
    const char *selName = sel_getName(sel);
    Class receiverClass = object_getClass(receiver);

    // 调用 forwardingTargetForSelector:
    if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
        id forwardingTarget = [receiver forwardingTargetForSelector:sel];
        if (forwardingTarget && forwardingTarget != receiver) {
            if (isStret == 1) {
                int ret;
                objc_msgSend_stret(&ret,forwardingTarget, sel, ...);
                return ret;
            }
            return objc_msgSend(forwardingTarget, sel, ...);
        }
    }

    // 僵尸对象
    const char *className = class_getName(receiverClass);
    const char *zombiePrefix = "_NSZombie_";
    size_t prefixLen = strlen(zombiePrefix); // 0xa
    if (strncmp(className, zombiePrefix, prefixLen) == 0) {
        CFLog(kCFLogLevelError,
              @"*** -[%s %s]: message sent to deallocated instance %p",
              className + prefixLen,
              selName,
              receiver);
        <breakpoint-interrupt>
    }

    // 调用 methodSignatureForSelector 获取方法签名后再调用 forwardInvocation
    if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
        NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
        if (methodSignature) {
            BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
            if (signatureIsStret != isStret) {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
                      selName,
                      signatureIsStret ? "" : not,
                      isStret ? "" : not);
            }
            if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {
                NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature frame:frameStackPointer];

                [receiver forwardInvocation:invocation];

                void *returnValue = NULL;
                [invocation getReturnValue:&value];
                return returnValue;
            } else {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
                      receiver,
                      className);
                return 0;
            }
        }
    }

    SEL *registeredSel = sel_getUid(selName);

    // selector 是否已经在 Runtime 注册过
    if (sel != registeredSel) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
              sel,
              selName,
              registeredSel);
    } // doesNotRecognizeSelector
    else if (class_respondsToSelector(receiverClass,@selector(doesNotRecognizeSelector:))) {
        [receiver doesNotRecognizeSelector:sel];
    }
    else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
              receiver,
              className);
    }

    // The point of no return.
    kill(getpid(), 9);
}
```



### 5、Super 本质

了解完 objc_msgSend 机制再来看下 super 的本质。

做个测试，重写 forwardInvocation 方法：

```objective-c

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [super forwardInvocation:anInvocation];
}

使用 clang 查看转换成的 c++ 代码

((void (*)(__rw_objc_super *, SEL, NSInvocation *))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("MJPerson"))}, sel_registerName("forwardInvocation:"), (NSInvocation *)anInvocation);

删除强制转换：
    objc_msgSendSuper(
                      {self, (id)class_getSuperclass(objc_getClass("MJPerson"))},
                      sel_registerName("forwardInvocation:"),
                      anInvocation);
```

如上，使用 super 调用方法的时候不再是 objc_msgSend，而是 objc_msgSendSuper，发送消息时有三个参数：

1. 结构体 struct __rw_objc_super
2. 方法名 SEL
3. 参数 anInvocation

这里主要看下第一个参数：

```
struct __rw_objc_super { 
	struct objc_object *object; 
	struct objc_object *superClass; 
	__rw_objc_super(struct objc_object *o, struct objc_object *s) : object(o), superClass(s) {} 
};
```

根据上面  objc_msgSendSuper 初始化的调用可以看到，super 的实质其实还是向 self 发送消息，receiver 依然是 self，只不过第二个参数是 superClass。

所以 super 的本质就是向 self（receiver）发送消息，不过查找方法时是从 super 开始查找。