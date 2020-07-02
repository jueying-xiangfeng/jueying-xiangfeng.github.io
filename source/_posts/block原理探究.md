---
title: block原理探究
date: 2020-07-01 19:36:19
tags:
---

<br/>

> 最近在做业务时，可谓一步一个坑，所以回过头来补充下最基础的原理知识。
> PS：基础好才是真的好！！！

<br/>

### Block 原理探究

将 block 相关的知识点归纳如下：

```
1. block 本质
2. block 变量捕获
3. block 类型
4. block copy 原理
5. __block __weak 类型原理
6. block 内存管理
7. block 循环引用
```

### Block 本质

测试代码：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) = ^{  
            NSLog(@"block --");
        };
        block();
    }
    return 0;
}
```

请出 `clang` 命令看一下 block 被编译成了什么：

```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));

        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    }
    return 0;
}
```

将上面的类型转换代码删除之后可以得到如下结构：

```
void (*block)(void) = &__main_block_impl_0(__main_block_func_0,
                                           &__main_block_desc_0_DATA));

block->FuncPtr(block);
```

可以看到调用 block() 其实就是调用 block->FuncPtr(block)。

再来看看 block 被编译后的具体结构：

```
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


            NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_395a2c_mi_0);
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

由 `void (*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA));` 参照上面 block 被编译的具体结构可以看到，block 被编译成了结构体 `struct __main_block_impl_0`，而 `struct __main_block_impl_0` 由结构体 `struct __block_impl impl` 和 `struct __main_block_desc_0* Desc` 构成，再来看下 impl 的结构：

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```
是不是很熟悉，第一个就是 isa 指针，而最后一个 FuncPtr 指针就是我们调用 block 时所调用的：`block->FuncPtr(block)`。

由上分析可以得出 block 调用的流程：

1. block 被编译成了结构体 `struct __main_block_impl_0`
2. 结构体初始化时传递了两个参数：`__main_block_func_0` 和 `&__main_block_desc_0_DATA`
3. `__main_block_func_0` 就是我们要实现的 block 内部的代码(block 内部代码被包装到了这个函数里面)
4. `&__main_block_desc_0_DATA` 的结构里面有一个 size，看结构体的初始化赋值：sizeof(struct __main_block_impl_0) 可以得出，size 存储的就是整体 block 的大小
5. `__main_block_impl_0` 结构体初始化时通过内部的同名构造函数将包装好的 block 实现函数 `__main_block_func_0` 指针和结构体 `__main_block_desc_0_DATA` 的地址传给了 `__main_block_impl_0` 相关的值
6. `__block_impl` 里面的 FuncPtr 存储的就是 block 内存代码被包装的函数指针
7. block 调用最终就是通过 `void (*block)(void)` block 指针调用 FuncPtr 函数：`block->FuncPtr(block)`

通过 struct __block_impl 里面的 isa 指针可以想到 block 可能是一个 OC 对象，下面我们来验证一下：
```
NSLog(@"%@ - %@ - %@ - %@",
              
              [block class],
              [[block superclass] class],
              [[[block superclass] superclass] class],
              [[[[block superclass] superclass] superclass] class]);
```

输出结果

```
__NSGlobalBlock__ - __NSGlobalBlock - NSBlock - NSObject
```

可以看到，block 最终的 superclass 就是继承自 NSObject，所以 block 就是一个 OC 对象。


#### 综合上得出：

> block 就是封装了函数调用和函数调用环境的 OC 对象


