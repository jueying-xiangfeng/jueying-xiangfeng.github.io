---
title: block5-block内存管理
date: 2020-07-06 09:09:58
tags:
---

### block 内存管理

<!-- more -->

测试代码：

```
NSObject * object = [[NSObject alloc] init];
__block NSObject * blockObject = [[NSObject alloc] init];
void (^block)(void) = ^{
  NSLog(@"--- %@", object);
  NSLog(@"--- %@", blockObject);
};
block(); 
NSLog(@"----");
```
查看 cpp 文件代码：

```
struct __Block_byref_blockObject_0 {
  void *__isa;
__Block_byref_blockObject_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 
 /// 最终被捕获到的 blockObject 变量
 NSObject *__strong blockObject;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  NSObject *__strong object;
  __Block_byref_blockObject_0 *blockObject; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSObject *__strong _object, __Block_byref_blockObject_0 *_blockObject, int flags=0) : object(_object), blockObject(_blockObject->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    
    __Block_byref_blockObject_0 *blockObject = __cself->blockObject; // bound by ref
    NSObject *__strong object = __cself->object; // bound by copy
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_3a341f_mi_0, object);
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_3a341f_mi_1, (blockObject->__forwarding->blockObject));
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    
    _Block_object_assign((void*)&dst->object, (void*)src->object, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_assign((void*)&dst->blockObject, (void*)src->blockObject, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    
    _Block_object_dispose((void*)src->object, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_dispose((void*)src->blockObject, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
        
        NSObject * object = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));

        __attribute__((__blocks__(byref))) __Block_byref_blockObject_0 blockObject = {
            0,
            (__Block_byref_blockObject_0 *)&blockObject,
            33554432, sizeof(__Block_byref_blockObject_0),
            __Block_byref_id_object_copy_131,
            __Block_byref_id_object_dispose_131,
            ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"))};
        
        
        void (*block)(void) = &__main_block_impl_0(__main_block_func_0,
                                                   &__main_block_desc_0_DATA,
                                                   object,
                                                   (__Block_byref_blockObject_0 *)&blockObject,
                                                   570425344));

        block->FuncPtr(block);

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_3a341f_mi_2);
    }
    return 0;
}
```

分析上面的代码可以看出，对于非 __block 修饰的 auto 变量 object 内存管理和我们 [前面](https://jueying-xiangfeng.github.io/2020/07/02/block3-%E7%B1%BB%E5%9E%8B%E3%80%81copy%E5%8E%9F%E7%90%86/) 讲过的 copy 过程一样：

持有：
> 1、block 内部调用 desp 里面的 copy 函数
> 2、copy 函数会调用 `_Block_object_assign` 
> 3、`_Block_object_assign` 函数会根据 auto 变量修饰符的类型（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用或弱引用

销毁：
> 1、block 内部调用 得搜 里面的 dispose 函数
> 2、dispose 函数会调用 `_Block_object_dispose`
> 3、`_Block_object_dispose` 函数会释放引用的 auto 变量

上面就是非 __block 修饰的对象类型的 auto 变量内存管理。

<br/>

下面来分析下 __block 修饰的对象类型的 auto 变量内存管理：
[上一节](https://jueying-xiangfeng.github.io/2020/07/03/block3-block%E5%8E%9F%E7%90%86/) 讲到了 \_\_block 的原理，可以想到 blockObject 被包装成了一个对象，而 `\_\_Block\_byref\_blockObject\_0` 里面的 `NSObject *\_\_strong blockObject;` 就是最终被捕获的变量，可以看到这里的修饰符是 \_\_strong，根据前面讲过的 copy 原理猜想一下这里的捕获结果可能和被修饰的 auto 变量的强弱有关，来做一个测试：

```
void (^block)(void);
{
  Animal * object = [[Animal alloc] init];
  __block __weak Animal * weakObject = object;
  block = ^{
      NSLog(@"--- %@", weakObject);
  };
}
```
查看打印结果，Animal 是在作用域结束后就释放了，而此时 block 还没有被释放，看下 cpp 文件的代码：

```
struct __Block_byref_weakObject_0 {
  void *__isa;
__Block_byref_weakObject_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 Animal *__weak weakObject;
};
```
说明捕获到的确实是 \_\_weak 的类型。

再来看一下上面 blockObject 被捕获对象初始化的地方，这里多了两个函数：`__Block_byref_id_object_copy_131`、`__Block_byref_id_object_dispose_131`，来看一下这两个函数的实现：

```
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```
这两个方法是不是很熟悉，跟我们前面讲 copy 的时候调用的方法是一样，看下 _Block_object_assign 函数的参数：`(char*)dst + 40`，这里的 dst 就是 `__Block_byref_blockObject_0`，那么 +40 代表什么呢？看下 `__Block_byref_blockObject_0` 的结构：

```
struct __Block_byref_blockObject_0 {
  void *__isa;	  8
__Block_byref_blockObject_0 *__forwarding; 8
 int __flags; 4
 int __size; 4
 void (*__Block_byref_id_object_copy)(void*, void*); 8
 void (*__Block_byref_id_object_dispose)(void*); 8
 NSObject *__strong blockObject;
};
```
+40 的地址正好是 blockObject 的地址，所以这两个函数是用来管理 blockObject 的。


总结下 \_\_block 修饰变量的内存管理：

持有：
> 1、blockObject 被包装成 `__Block_byref_blockObject_0` 对象，`__Block_byref_blockObject_0` 对象里面持有 blockObject（blockObject 的强弱是根据自身的修饰符决定的）
> 2、`__main_block_impl_0` 持有 `__Block_byref_blockObject_0`
> 3、`__Block_byref_blockObject_0` 的内存管理依赖 `__main_block_impl_0` 的 desp 的 copy 函数（`_Block_object_assign`）
> 4、blockObject 对象的内存管理依赖 `__Block_byref_blockObject_0` 的 copy 函数（`_Block_object_assign`）

销毁：
> 1、`__main_block_impl_0` 通过 desp 的 dispose 函数释放 `__Block_byref_blockObject_0`（`_Block_object_dispose`）
> 2、`__Block_byref_blockObject_0` 通过调用内部的 `__Block_byref_blockObject_0` 函数释放最终的 blockObject 对象

<br/>

#### 结论总结

* 当 block 在栈上时，对对象类型的 auto 变量、__block 变量不会产生强引用
* 当 block 被 copy 到堆上面时会通过 copy 函数来处理 (__block 变量：	_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/)， auto 变量：_Block_object_assign((void*)&dst->p, (void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/))
* 	\_Block\_object\_assign函数会根据所指向对象的修饰符（__strong、__weak、__unsafe_unretained）做出相应的操作，形成强引用（retain）或者弱引用（注意：这里仅限于ARC时会retain，MRC时不会retain）
*  当 block 从堆上移除时会调用 dispose 函数处理（__block 变量：_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/)， auto 变量：_Block_object_dispose((void*)src->p, 3/*BLOCK_FIELD_IS_OBJECT*/)）
* _Block_object_dispose函数会自动释放指向的对象（release）


### __block 的 __forwarding 指针

上面 blockObject 的访问其实是：blockObject->__forwarding->blockObject，为什么这里要多一步 \_\_forwarding 的操作呢，而且 \_\_forwarding 还是指向自己的。
因为当栈上的 block copy 到堆上时，block 实际访问的其实是堆上的内存，如果这时再访问栈内存是不正确的，所以此时栈上的 \_\_forwarding 指针会指向堆上面的内存，当访问 blockObject 的时候其实访问的是最终被 copy 到堆上面的内存，这样就不会出现访问错乱的问题。

可以来验证一下，使用 MRC 的环境：

```
NSObject * object = [[NSObject alloc] init];     
__block NSObject * blockObject = [[NSObject alloc] init];
void (^block)(void) = ^{
  NSLog(@"--- %@", object);
  NSLog(@"--- %@", blockObject);
};

struct __main_block_impl_0 * blockImpl = (struct __main_block_impl_0 *)block;   
[block copy];
NSLog(@"---- %@", blockImpl);
```
此时 block 是在栈上面，来打印下此时的地址：`(__Block_byref_blockObject_0 *) __forwarding = 0x00007ffeefbff468`
当调用完 copy 函数之后：`(__Block_byref_blockObject_0 *) __forwarding = 0x000000010053af80`，很明显，此时的地址从栈上指向了堆空间。


