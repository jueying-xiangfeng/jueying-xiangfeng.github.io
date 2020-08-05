---
title: block4-__block原理
date: 2020-07-03 11:25:51
tags:
---

#### __Block 原理

先来测试一个经典的案例：

```
int age = 10;
void (^block)(void) = ^{
	NSLog(@"--- age = %d", age);
}; 
age = 20;
block();

/// 输出结果 --- age = 10
```

这个案例想必不会陌生，原理可以根据前面变量的捕获分析，来看下 cpp 代码：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age; /// 被捕获的 age 值
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
age 被捕获到了 block 里面，此时 age 和被 block 捕获的 age 完全就是两个不相干的内存地址，所以修改外面的 age 对 block 里面的 age 是没有影响的，试下在 block 里面直接修改 age 的值呢？可以看到编译报错，提示不能修改 age 的值，我们来从 cpp 代码分析下为什么：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age;
}

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        int age = 10;
        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age, 570425344));

        age = 20;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}
```
如上，在 block 里面访问 age，最终是在函数 __main_block_func_0 访问的结构体 __main_block_impl_0 里面捕获的 age，而且 age 变量是在主函数声明的，现在想要在 __main_block_func_0 函数修改是不被允许的，因为这是两个函数，调用栈也不一样。
那么怎么才能在 block 里面修改这种 auto 类型的变量呢，答案就是使用 `__block`。

把上面的 int age 变量加上修饰符 __block，再来看 cpp 代码：

```
struct __Block_byref_age_0 {
  void *__isa;
__Block_byref_age_0 *__forwarding;
 int __flags;
 int __size;
 int age;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_age_0 *age; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_age_0 *age = __cself->age; // bound by ref


            NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_f5d64f_mi_0, (age->__forwarding->age));
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->age, (void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        __attribute__((__blocks__(byref))) __Block_byref_age_0 age = {(void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};

        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));

        (age.__forwarding->age) = 20;

        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}


/************* 来整理一下上面的代码 *************/

struct __Block_byref_age_0 {
  void *__isa;
__Block_byref_age_0 *__forwarding;
 int __flags;
 int __size;
 int age;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_age_0 *age; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_age_0 *_age, int flags=0) : age(_age->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_age_0 *age = __cself->age; // bound by ref
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
	_Block_object_assign((void*)&dst->age, (void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
	_Block_object_dispose((void*)src->age, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        __Block_byref_age_0 age = {
        0,
        (__Block_byref_age_0 *)&age,
        0,
        sizeof(__Block_byref_age_0), 
        10};

        void (*block)(void) = &__main_block_impl_0(
        __main_block_func_0,
         &__main_block_desc_0_DATA,
         &age,
         570425344);

        (age.__forwarding->age) = 20;

        block->FuncPtr(block);
    }
    return 0;
}
```

加上 __block 后可以看到在 block 里面捕获的 age 的位置变成了 __Block_byref_age_0，而在 __Block_byref_age_0 里面有一个 int age 变量，在 main 函数里面声明的 int age 被包装成了 __Block_byref_age_0 的类型，在修改 age = 20 时实际代码是通过 __Block_byref_age_0 来访问 int age 的，也就是说不管在 main 函数还是在 block 内部，修改 age 的话都是修改的包装类型里面的 age。

**__block 原理总结**

* __block 修饰符可以用以解决 block 内部无法修改 auto 变量值的问题
* __block 不能修改全局变量、static 变量
* 编译器会将 __block 变量包装成一个对象

具体流程如下

> 1、将 __block 变量包装成 __Block_byref_age_0 类型的对象
> 2、访问 age 时都是访问的 __Block_byref_age_0 里面的 age
> 3、访问 age 时是通过 `__forwarding` 指针来访问的（初始化默认指向自己）

再来看下如果在 block 外面打印 age 的地址，这个地址到底是 __Block_byref_age_0 age 的地址还是 int age 的地址：

为了方便查看，将 block 生成的结构体拿过来用一下：

```
struct __Block_byref_age_0 {
    void *__isa;
    struct  __Block_byref_age_0 *__forwarding;
    int __flags;
    int __size;
    int age;
};

struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(void);
  void (*dispose)(void);
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  struct __Block_byref_age_0 *age; // by ref
};

/// main 函数代码
__block int age = 10;
        
void (^block)(void) = ^{
  NSLog(@"--- age = %d", age);
  age = 30;
  NSLog(@"--- age = %d", age);
};
struct __main_block_impl_0 * blockImpl = (__bridge struct __main_block_impl_0 *)block;
   
NSLog(@"-- age address : %p", &age);
```

在 NSLog(@"-- age address : %p", &age); 处打断点，来查看一下两个 age 变量的地址：

```
p/x &(blockImpl->age) : (__Block_byref_age_0 **) $2 = 0x000000010064a030

p/x &(blockImpl->age->age) : (int *) $3 = 0x000000010064a058

NSLog 打印地址： -- age address : 0x10064a058
```
可以看出来打印 age 的地址和生成的包装对象里面的 age 变量地址是一样的，这里可能类似 KVO，苹果不想让开发者看到实现的过程，直接屏蔽了，对于我们开发者来说，访问的其实就是 int age 本身。


最后再来看下面一个例子：

```
NSMutableArray * array = [NSMutableArray array];        
void (^block)(void) = ^{
  [array addObject:@"1"];
};
```
编译一下成功，执行完 block 后在打印 array，添加 object 成功，为什么呢？因为这里是使用指针 array，并不是修改指针，如果 array = nil 那么就会出现编译错误，因为要修改的话需要使用 __block 修饰符。

