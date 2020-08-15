---
title: block原理探究-2
date: 2019-01-14 19:25:32
tags:
description: 
---

<!-- more -->

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


应用程序的内存分配关系为：

`程序区` --> `数据区`	 --> `堆区` --> `栈区`
---------------------------------------------------------->

block类型分为三种:
> \_\_NSGlobalBlock\__	 (没有访问 auto 变量 -- `数据区` -- copy:无操作)
> \_\_NSStackBlock\__  (访问了 auto 变量 -- `栈区` -- copy:由栈复制到堆)
> \_\_NSMallocBlock\__	 (__NSStackBlock__调用了copy -- `堆区` -- copy:引用计数增加)

### 总结下block的copy

#### 1、ARC环境下
在ARC环境下编译器会根据情况自动将栈上的block复制到堆上：

* block作为函数返回值
* 将block赋值给__strong指针
* block作为Cocoa API中的方法名含有usingBlock的方法参数时
* block作为GCD API的方法参数时 

#### 2、block作为属性的写法 (修饰)

* ARC ：strong/copy
* MRC ：copy

### 补充：对象类型的AUTO变量
当block内部访问了对象类型的auto变量时：

1、 如果block在栈上，不会对auto变量产生强引用
2、 如果block被拷贝到堆上会调用block的copy函数

* copy函数调用_Block_object_assign
* _Block_object_assign函数会根据auto变量的修饰符 (\__strong、__weak、__unsafe_unretained)做出相应的操作，形成强引用或弱引用

3、如果block从堆上移除会调用dispose函数

* dispose函数会调用_Block_object_dispose函数
* _Block_object_dispose会自动释放引用的auto变量(release)

#### clang命令无法支持__weak的问题
在使用clang转换OC为C++时，遇到以下问题：
cannot create __weak refrence in file using manual refrence

解决：支持ARC、指定运行时系统版本：
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-8.0.0 main.m

测试代码：

```
typedef void (^Block)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Block block;
        
        {
            NSObject * obj = [[NSObject alloc] init];
            
            __weak typeof(obj) weakObj = obj;
            
            block = [^{
                NSLog(@"---- %@", weakObj);
            } copy];
        }
        
        block();
        
        NSLog(@"------");
        
    }
    return 0;
}
```

转换C++：

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  NSObject *__weak weakObj;	// 为捕获到的对象auto变量
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSObject *__weak _weakObj, int flags=0) : weakObj(_weakObj) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  NSObject *__weak weakObj = __cself->weakObj; // bound by copy

                NSLog((NSString *)&__NSConstantStringImpl__var_folders_lw_6z_mkyd178q8n6w9m1vqbk9w0000gn_T_main_5e91db_mi_0, weakObj);
            }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
    
    _Block_object_assign((void*)&dst->weakObj, (void*)src->weakObj, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    
    _Block_object_dispose((void*)src->weakObj, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  
  // 多出以下两个函数  copy、dispose
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
}
```


