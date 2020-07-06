---
title: block6-循环引用
date: 2020-07-06 11:15:01
tags:
---

### 循环引用

下面是 block 循环引用的经典图片：

![retaincircly.png](http://ww1.sinaimg.cn/large/005O0Zogly1ggh999v50lj30pa0h6n31.jpg)

看一下测试代码：

```
@interface Animal : NSObject
@property (nonatomic, copy) void (^block)(void);
@property (nonatomic, assign) int age;
@end


Animal * animal = [[Animal alloc] init];
animal.block = ^{
  NSLog(@"--- %d", animal.age);
};
NSLog(@"---- ");
```

分析下各个部分指针指向：

![retaincircle1.png](http://ww1.sinaimg.cn/large/005O0Zogly1ggh3caddxqj317w0t4acx.jpg)

解决循环引用的办法也很简单，这里只需要将其中的一个强指针改为弱指针就可以了，根据前面 block 捕获的知识可以得到：使用 \_\_weak 可以解决：

```
Animal * animal = [[Animal alloc] init];   
__weak typeof(animal) weakAnimal = animal;
animal.block = ^{
  NSLog(@"--- %d", weakAnimal.age);
};
```

![retaincircly2.png](http://ww1.sinaimg.cn/large/005O0Zogly1ggh3fzscb9j317g0po0vh.jpg)

还可以使用 `__unsafe_unretained` 来解决，不过和 `__weak` 的区别是：`__unsafe_unretained` 不会将释放的指针置为 nil。


#### __block 解决循环引用

\_\_block 也可以解决循环引用，不过方法有点特殊：

```
__block Animal * animal = [[Animal alloc] init];   
animal.block = ^{
  NSLog(@"--- %d", animal.age);
};
```
分析下指针的指向：

![retaincircly3.png](http://ww1.sinaimg.cn/large/005O0Zogly1ggh3nyy6hbj31hw102n2w.jpg)

现在是三角的闭环，这里可以使用如下方法来解决循环引用的问题：

![retaincircly4.png](http://ww1.sinaimg.cn/large/005O0Zogly1ggh41xn5gcj31hy0ymjx3.jpg)

具体代码如下：

```
__block Animal * animal = [[Animal alloc] init];
   
animal.block = ^{
  NSLog(@"--- %d", animal.age);
  animal = nil;
};
animal.block();
```
此种方发必须调用 block，而且在调用完 block 后需要设置 \_\_block 变量为 nil。


### MRC 的循环引用

1、使用 \_\_unsafe\_unretained 解决。

```
Animal * animal = [[Animal alloc] init];   
__unsafe_unretained Animal * weakAnimal = animal;
animal.block = ^{
  NSLog(@"--- %d", weakAnimal.age);
};
```

2、使用 \_\_block，前面讲到过在 MRC 环境下 \_\_block 变量对 animal 一直都是弱引用，所以这里使用 \_\_block 就能解决。

```
__block Animal * animal = [[Animal alloc] init];   
animal.block = ^{
  NSLog(@"--- %d", animal.age);
};
```


