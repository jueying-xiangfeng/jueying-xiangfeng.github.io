---
title: block2-变量的捕获
date: 2020-07-02 14:14:51
tags:
---

<br/>

### 变量的捕获

为了保证 block 内部能够正常访问外部变量，block 有个变量捕获机制：

变量类型 | 捕获到 block 内部 | 访问方式
------- | ---------------- | -------
局部变量 - auto | √ | 值传递
局部变量 - static | √ | 指针传递
全局变量 | x | 直接访问

<br/>

下面我们来验证一下，测试代码如下：

```
/// 全局变量
int a_ = 10;
static int sa_ = 10;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        /// 局部 auto 变量
        int b = 10;
        /// 局部 static 变量
        static int sb = 10;
        
        void (^block)(void) = ^{
            NSLog(@"log -- %d, %d, %d, %d", a_, sa_, b, sb);
        };
        block();
    }
    return 0;
}
```
通过 clang 查看上面代码的编译结果：

```
int a_ = 10;
static int sa_ = 10;

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int b;
  int *sb;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _b, int *_sb, int flags=0) : b(_b), sb(_sb) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int b = __cself->b; // bound by copy
  int *sb = __cself->sb; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_1x_bw2dcypj2dng06lb8qw1h8gh0000gn_T_main_3ee42d_mi_0, a_, sa_, b, (*sb));
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        int b = 10;
        static int sb = 10;

        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, b, &sb));

        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
        
        /// 这里是删除类型转换后的结果
        void (*block)(void) = &__main_block_impl_0(__main_block_func_0,
                                                   &__main_block_desc_0_DATA,
                                                   b,
                                                   &sb));

        block->FuncPtr(block);
    }
    return 0;
}
```

如上可以看到 block 结构体初始化时除了传递 FuncPtr 和 desp 两个参数外，多了 b 和 &sb 参数，再看结构体 `__main_block_impl_0` 里面确实多了这两个参数，注意这里并没有和 a 相关的捕获。
我们在 block 里面访问了四个变量，来看 `__main_block_func_0` 函数里面是如何访问的：

1. 全局变量 a_ 和 sa_ 是直接访问的
2. 局部变量 b 和 sb 是访问的 `__main_block_impl_0` 里面捕获的值
3. b 是捕获的值，sb 是 int* 类型，标识的是捕获的地址

由上可以验证我们变量捕获的结论。

<br/>

#### 测试

下面我们来做一个测试，新建一个 `Animal` 的类，声明一个 age 属性和 test 方法：

```
@interface Animal : NSObject
@property (nonatomic, assign) int age;
@end

@implementation Animal
- (void)test {
    void (^block)(void) = ^{
        NSLog(@"--- %d", self.age);
    };
    block();
}
@end
```

如上测试代码，block 有捕获 self 吗？

来看下 clang 编译的结果：

```
struct __Animal__test_block_impl_0 {
  struct __block_impl impl;
  struct __Animal__test_block_desc_0* Desc;
  Animal *const __strong self;
  __Animal__test_block_impl_0(void *fp, struct __Animal__test_block_desc_0 *desc, Animal *const __strong _self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

看结果是有捕获的，这里有没有疑问，self 难道不是全局变量吗？为什么这里会有捕获呢？根据 block 捕获的机制，只有 auto 变量才会捕获，这里既然有捕获 self，就证明 self 不是全局变量，下面我们看下 test 方法，验证一下：

```
static void _I_Animal_test(Animal * self, SEL _cmd) {

    void (*block)(void) = ((void (*)())&__Animal__test_block_impl_0((void *)__Animal__test_block_func_0, &__Animal__test_block_desc_0_DATA, self, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
    
   /// 简化一下
    void (*block)(void) = &__Animal__test_block_impl_0(__Animal__test_block_func_0, 
        &__Animal__test_block_desc_0_DATA,
         self,
          570425344));
          
    block->FuncPtr(block);
}
```
由上可以看到 __Animal__test_block_impl_0 在初始化时确实传递了 self 参数，再看 test 方法的两个参数：`self`、`_cmd`，是不是很熟悉，这是每个方法里面默认自带的参数，所以我们才能在方法里面使用 self 和 _cmd。
到这里也就明白了，self 是作为形参被捕获的，完全符合我们总结的 block 捕获机制。

##### 总结

> 变量有没有被 block 捕获，就看变量的类型是全局变量还是局部变量。


