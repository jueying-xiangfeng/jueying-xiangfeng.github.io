---
title: block原理探究-1
date: 2018-07-25 13:26:21
tags:
---

### block 底层数据结构

测试代码:

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) = ^{
            NSLog(@"block1234");
        };
        block();
    }
    return 0;
}
```
用 `clang` 编译查看 .cpp 文件, 看block被编译成了什么:

```
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

去掉上面的强制类型转换如下:
void (*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA));

block->FuncPtr(block);
```
可以看到最终block的调用就是 `block->FuncPtr(block);` , 就是调用函数

看一下block的 `__main_block_impl_0` 的结构:

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

           NSLog((NSString *)&__NSConstantStringImpl__var_folders_y5_bv8t9pxx6jg830_kh4l6p_800000gn_T_main_fc5269_mi_0);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```
可以看到block其实就是一个结构体,再仔细看一下还有isa指针,是不是跟我们前面讲到的对象一样,所以block也是OC的对象,里面有 `struct __block_impl impl;` 和 `struct __main_block_desc_0* Desc;` 结构, 而 `__block_impl` 里面有isa指针和 `void *FuncPtr;` ,不过结构体里面有构造函数 `__main_block_impl_0` .

可以分析block在编译的时候是这样的: 根据block里面的实现先生成一个函数 `__main_block_func_0` ,然后接着生成一个 `__main_block_impl_0` 类型的结构体,将生成的函数指针和 `__main_block_desc_0` 传入并调用构造函数初始化block,这样block的结构就生成了.
当我们调用block时其实就是根据 `__block_impl` 查找函数指针 `void *FuncPtr` ,然后调用对应的函数的过程.

### block 变量的捕获

我们来针对三种方式来测试:

```
<!-- 1.全局变量 -->
int weight = 10;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        <!-- 2.局部变量 auto修饰(默认值) -->
        auto int age = 10;
        <!-- 3.局部变量 static修饰 -->
        static int height = 10;
        
        void (^block1)(void) = ^{
            NSLog(@"block1--- %d  %d  %d", age, height, weight);
        };
        age = 20;
        height = 20;
        weight = 20;
        
        block1();
    }
    return 0;
}
```
重新用clang编译查看结果:

```
int weight = 10;

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
  int *height;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int *_height, int flags=0) : age(_age), height(_height) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age; // bound by copy
  int *height = __cself->height; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_k8_chp8jt5d231d16l0s6w3c_gh0000gn_T_main_53994b_mi_0, age, (*height), weight);
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        auto int age = 10;
        static int height = 10;

        void (*block1)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age, &height));

        age = 20;
        height = 20;
        weight = 20;

        ((void (*)(__block_impl *))((__block_impl *)block1)->FuncPtr)((__block_impl *)block1);
    }
    return 0;
}
```
可以看到block里面只捕捉到了局部变量,而且捕捉到的局部变量的类型不一样, `auto` 类型的为值传递,而 `static` 类型的为指针传递.全局变量类型的为直接访问.

> block能捕获到局部变量,不能捕获到全局变量.

### block的类型

先来看一下测试:

```
void (^block1)(void) = ^{
	NSLog(@"block1--- %d  %d  %d", age, height, weight);
};
        
NSLog(@"%@  %@  %@  %@", [block1 class], [[block1 class] superclass], [[[block1 class] superclass] superclass], [[[[block1 class] superclass] superclass] superclass]);

打印结果:
__NSMallocBlock__  __NSMallocBlock  NSBlock  NSObject
```
block确实是对象,继承关系:
> \_\_NSMallocBlock__ : __NSMallocBlock : NSBlock : NSObject

block类型分为三种:
> \_\_NSGlobalBlock\__		(没有访问 auto 变量 -- 数据区 -- copy:无操作)
> \_\_NSStackBlock\__		(访问了 auto 变量 -- 栈 -- copy:由栈复制到堆)
> \_\_NSMallocBlock\__		(__NSStackBlock__调用了copy -- 堆 -- copy:引用计数增加)


