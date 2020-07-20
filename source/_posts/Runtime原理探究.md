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

