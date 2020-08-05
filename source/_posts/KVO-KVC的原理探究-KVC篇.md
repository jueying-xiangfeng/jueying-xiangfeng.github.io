---
title: KVO-KVC的原理探究 - KVC篇
date: 2018-07-19 15:12:40
tags:
---
### 关于KVC的探究
#### 基本介绍和使用
KVC全称Key-Value Coding 键值编码，可以通过Key来访问某个属性，常见的API：
```
- (nullable id)valueForKeyPath:(NSString *)keyPath;
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;

- (nullable id)valueForKey:(NSString *)key;
- (void)setValue:(nullable id)value forKey:(NSString *)key;
```
创建Person类、Animal类，添加属性如下：

```
@interface Animal : NSObject
@property (nonatomic, assign) NSInteger height;
@end

@interface Person : NSObject
@property (nonatomic, assign) NSInteger age;
@property (nonatomic, strong) Animal * dog;
@end
```
如上，使用KVC赋值的方式为：
```
Animal * dog = [[Animal alloc] init];
Person * person = [[Person alloc] init];
person.dog = dog;
// Key        
[person setValue:@11 forKey:@"age"];
// KeyPath
[person setValue:@123 forKeyPath:@"dog.height"];
        
NSLog(@"---- %@ -- %@", @(person.age), @(person.dog.height));

打印结果：
2018-07-19 23:00:48.546719+0800 KVC[53839:3059207] ---- 11 -- 123
```
可以看到直接赋值的话两中方式都可以，但是类似上面的dog.height嵌套的方式必须通过KeyPath的方式赋值。
#### KVC原理
进入Foundation里面查看 `- (void)setValue:(nullable id)value forKey:(NSString *)key;` 方法的注解可以了解到，**`KVC的赋值步骤`** 如下图：

```flow
st=>start: 调用setValue：forKey
e_findSucceed=>end: 传递参数，调用方法
op1=>operation: Operation1
sub1=>subroutine: My Subroutine
cond_findMethod=>condition: 按照 setKey：
_setKey：
顺序查找方法
cond_instanceVariables=>condition: 查看
accessInstance
VariablesDirectly
（默认返回YES）
方法的返回值
cond_findIvar=>condition: 按照
_key、_isKey
key、isKey
顺序查找
成员变量
e_findIvar=>end: 直接赋值
e_exception=>end: 调用
setValue：forUndefinedKey：
并且抛出
NSUnKnownKeyException

st->cond_findMethod
cond_findMethod(yes, bottom)->e_findSucceed
cond_findMethod(no, right)->cond_instanceVariables
cond_instanceVariables(yes, bottom)->cond_findIvar
cond_findIvar(yes, bottom)->e_findIvar
cond_instanceVariables(no, right)->e_exception
cond_findIvar(no, right)->e_exception
```
以上为markdown语法，用的 MWebLite编写的，奈何blog不支持，所以直接将结果导图截图贴在下面：
![](http://ww1.sinaimg.cn/large/005O0Zogly1ftgaf779vdj315m10a7c9.jpg)

进入Foundation里面查看 `- (nullable id)valueForKey:(NSString *)key;` 方法的注解可以了解到，**`KVC的取值步骤`** 如下图：

```flow
st=>start: 调用valueForKey：
e_findSucceed=>end: 调用方法
op1=>operation: Operation1
sub1=>subroutine: My Subroutine
cond_findMethod=>condition: 按照 getKey
key、isKey、
_key、_getKey
顺序查找方法
cond_instanceVariables=>condition: 查看
accessInstance
VariablesDirectly
（默认返回YES）
方法的返回值
cond_findIvar=>condition: 按照
_key、_isKey
key、isKey
顺序查找
成员变量
e_findIvar=>end: 直接取值
e_exception=>end: 调用
valueForUndefinedKey：
并且抛出
NSUnKnownKeyException

st->cond_findMethod
cond_findMethod(yes, bottom)->e_findSucceed
cond_findMethod(no, right)->cond_instanceVariables
cond_instanceVariables(yes, bottom)->cond_findIvar
cond_findIvar(yes, bottom)->e_findIvar
cond_instanceVariables(no, right)->e_exception
cond_findIvar(no, right)->e_exception
```
![](http://ww1.sinaimg.cn/large/005O0Zogly1ftgcd9stvjj30lx0i7mzu.jpg)

#### 接下来我们来验证一下：
**setValue：forKey：赋值**
使用上面的Person类，将属性全部删除，添加以下成员变量，重写两个set方法：
```
@interface Person : NSObject {
    @public
    int _age;
    int _isAge;
    int age;
    int isAge;
}
@end

- (void)setAge:(NSInteger)age {
    NSLog(@"--- setAge");
}

- (void)_setAge:(NSInteger)age {
    NSLog(@"--- _setAge");
}
```
然后调用KVC给age赋值，可以看打印结果：
```
2018-07-20 14:41:18.679275+0800 KVC[65810:3455316] --- setAge
```
此时调用的是 `setAge:` 方法，注释掉 `setAge:` 方法再次运行：
```
2018-07-20 14:41:43.561291+0800 KVC[65834:3456053] --- _setAge
```
当这两个方法都没有实现的时候就会调用 `accessInstanceVariablesDirectly` 方法，若返回YES，则直接给成员变量赋值，如没有或返回值为NO则会调用 `setValue：forUndefinedKey：` 并且抛出NSUnKnownKeyException异常。
**valueForKey：取值**
重写流程图中的get方法：
```
- (int)getAge {
    return 11;
}

- (int)age {
    return 12;
}

- (int)isAge {
    return 13;
}

- (int)_age {
    return 14;
}

- (int)_getAge {
    return 15;
}

打印结果：
NSLog(@"---- %@", [person valueForKey:@"age"]);
```
如上我们可以查看当调用过的方法后就注释，直到执行完成所有的方法。我们可以控制 `accessInstanceVariablesDirectly` 方法的返回值来证明我们想要的结果。

#### 思考一下KVC能触发KVO吗？
我们来测试一下，添加Observer类来监听并输出Log：
```
@implementation Observer
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey, id> *)change context:(void *)context {
    NSLog(@"observeValueForKeyPath - %@", change);
}
@end

测试代码：
Observer * ob = [Observer new];
Person * person = [[Person alloc] init];
[person addObserver:ob forKeyPath:@"age" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
        
[person setValue:@1 forKey:@"age"];
[person removeObserver:ob forKeyPath:@"age"];

打印结果为：
2018-07-20 15:19:30.526730+0800 KVC[67145:3507833] observeValueForKeyPath - {
    kind = 1;
    new = 1;
    old = 0;
}
```
可以看到KVC确实调用了KVO，[上一篇文章](https://jueying-xiangfeng.github.io/2018/07/17/KVO-KVC%E7%9A%84%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6-KVO%E7%AF%87/)中我们了解到了KVO的实现，接下来可以大概验证一下：
```
在Person类中实现如下方法：
- (void)willChangeValueForKey:(NSString *)key {
    NSLog(@"willChangeValueForKey");
    [super willChangeValueForKey:key];
}

- (void)didChangeValueForKey:(NSString *)key {
    NSLog(@"didChangeValueForKey-- begin");
    [super didChangeValueForKey:key];
    NSLog(@"didChangeValueForKey-- end");
}

打印结果：
2018-07-20 15:23:34.761496+0800 KVC[67256:3513685] willChangeValueForKey
2018-07-20 15:23:34.761836+0800 KVC[67256:3513685] didChangeValueForKey-- begin
2018-07-20 15:23:34.762151+0800 KVC[67256:3513685] observeValueForKeyPath - {
    kind = 1;
    new = 1;
    old = 0;
}
2018-07-20 15:23:34.762202+0800 KVC[67256:3513685] didChangeValueForKey-- end
```
可以看到打印结果跟KVO是一样的，这里可以猜测苹果大大在KVC内部的实现：
```
[person willChangeValueForKey:@"age"];
person->_age = 1;
[person didChangeValueForKey:@"age"];
```
KVC相当于手动调用了KVO。


