---
title: block3-类型、copy原理
date: 2020-07-02 16:15:21
tags: block 类型、copy
toc: true
description: 
---

### block 类型

block 的类型有3中，可以通过调用 class 方法或者 isa 指针查看具体类型，最终都是集成子 NSBlock 类型

1. 数据区：__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）
2. 栈区：__NSStackBlock__ （ _NSConcreteStackBlock ）
3. 堆区：__NSMallocBlock__ （ _NSConcreteMallocBlock ）

> block 类型的具体类型区分：

block 类型 | 环境
--------- | -----
\_\_NSGlobalBlock\_\_ | 没有访问 auto 变量
\_\_NSStackBlock\_\_ | 访问了 auto 变量
\_\_NSMallocBlock\_\_ | \_\_NSStackBlock\_\_ 调用了 copy

<!-- more -->

看一下测试代码：

```
void (^block1)(void) = ^{
	NSLog(@"block1");
};
        
int age = 1;
void (^block2)(void) = ^{
	NSLog(@"block2 -- %d", age);
};
        
NSLog(@"%@  %@  %@", [block1 class], [block2 class], [^{
            NSLog(@"block3 -- %d", age);
        } class]);
        
ARC 环境打印结果：
__NSGlobalBlock__  __NSMallocBlock__  __NSStackBlock__     
```
根据上面 block 的类型区分分析测试代码里面的 block2 应该是在栈空间的，可是打印出的类型却是在堆空间，这是为什么呢？我看来看下生成的 cpp 文件，三种类型的 block isa 指针类型都是：_NSConcreteStackBlock，感觉也不对。
到这里就不能只看静态编译的 cpp 文件代码来分析了，这里要以运行时的结果为准，就是我们上面打印的结果，因为目前的环境是 ARC 环境，ARC 帮我们做了一些处理，下面我们使用 MRC 环境在走一遍上面的打印：
```
/// MRC 环境打印结果
__NSGlobalBlock__  __NSStackBlock__  __NSStackBlock__
```
ARC 结果与我们分析的不正确是因为在 ARC 环境下会对 stack 类型的 block 做一次 copy 操作。

<br/>

### block copy

不同类型的 block copy 结果

Block 类型 | 副本源的配置存储域 | copy 效果
--------- | ---------------- | --------
\_\_NSGlobalBlock\_\_ | 程序的数据区域 | 什么也不做
\_\_NSStackBlock\_\_ | 栈  | 从栈赋值到堆
\_\_NSMallocBlock\_\_ | 堆 | 引用计数增加

再来看下 ARC 环境下栈空间 block 自动 copy 的情况：

> block 作为参数返回值
> 将 block 赋值给 __strong 指针时
> block 作为 cocoa api 中方法名含有 usingBlock 的方法参数时
> block 作为 GCD api 的方法参数时

栈空间的 block 是捕获了 auto 的类型变量，而 auto 类型是自动销毁类型，在作用域消失后会回收内存，如果栈空间的 block 不做 copy 的话内存访问可能会出现错乱的问题，看下测试代码：

```
/// MRC 环境
typedef void (^Block)(void);
Block block;
void test() {
    int age = 10;
    block = ^{
        NSLog(@"---- block -- %d", age);
    };
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
       test();
       block();
    }
    return 0;
}
```
如上打印的结果就是随机值，因为上面 block 的内存分配在栈空间，当作用域结束后内存会回收，所以捕获的age的内存空间会回收，当我们再次访问时就会成为随机值。

<br/>

### 对象类型的 auto 变量

特殊的结论：
> 当 block 内存访问了对象类型的 auto 变量时，如果 block 是在栈上，将不会对 auto 变量产生强引用。

因为栈空间的 block 自己都可能随时销毁，这里不会对对象类型的 auto 变量产生强引用。

<br/>

block 被拷贝到堆上面的情况：
看下测试代码：

```
// 1、访问 animal
Block testBlock;
{
	Animal * animal = [[Animal alloc] init];
	animal.age = 10;
	
	testBlock = ^{
		NSLog(@"--- %d", animal.age);
	};
}

// 2、wealAnimal
Block testBlock;
{
	Animal * animal = [[Animal alloc] init];
	animal.age = 10;
	__weak Animal * weakAnimal = animal;
	testBlock = ^{
		NSLog(@"--- %d", weakAnimal.age);
	};
}
```

如上，第一种情况 animal 的释放是在 testBlock 销毁时才会释放，第二种是在 animal 作用域结束也就是大括号结束时就释放了，说明这两种情况下的捕获有区别，来 cpp 文件看下捕获的情况：

```
// 1、访问 animal
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  
  // 捕获的是 __strong 类型
  Animal *__strong animal;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, Animal *__strong _animal, int flags=0) : animal(_animal) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// 2、wealAnimal
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;   
  
  // 捕获的是 __weak 类型
  Animal *__weak weakAnimal;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, Animal *__weak _weakAnimal, int flags=0) : weakAnimal(_weakAnimal) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
可以看到在堆空间 block 对象类型的 auto 变量捕获是根据捕获的指针的强弱进行的。
再看 cpp 代码，生成的 __main_block_desc_0 结构有一点点的差别：

```
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->weakAnimal, (void*)src->weakAnimal, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->weakAnimal, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```
在生成的  __main_block_desc_0 中多了两个函数： `copy` 和 `dispose`。这两个函数就是用来管理捕获到的对象指针的。

总结调用流程：

**如果 block 被拷贝到堆上**
> 1、调用 block 内部的 copy 函数
> 2、copy 函数会调用 `_Block_object_assign` 函数
> 3、`_Block_object_assign` 函数会根据 auto 变量修饰符的类型（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用或弱引用

**block 从堆上面移除**
> 1、调用 block 内部的 dispose 函数
> 2、dispose 函数会调用 `_Block_object_dispose` 函数
> 3、`_Block_object_dispose` 函数会自动释放引用的 auto 变量

函数 | 调用时机
--- | ---
copy | 栈上的 block 赋值到堆上
dispose | 栈上的 block 被废弃


<br/>

#### clang 命令：
普通：`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`

需要编译 `__weak` 修饰符：`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-13.0.0 main.m`







