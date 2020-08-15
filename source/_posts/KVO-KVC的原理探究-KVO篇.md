---
title: KVO-KVC的原理探究 - KVO篇
date: 2018-07-17 11:38:17
tag: KVO
toc: true
description: 

---


### 关于KVO的探究

#### KVO的基本使用

<!-- more -->

创建Person类，添加属性age：

```
@interface Person : NSObject
@property (nonatomic, assign) NSInteger age;
@end
```
在ViewController中添加属性`@property (nonatomic, strong) Person * person1;`
实例化并添加KVO观察age属性：
```
self.person1 = [[Person alloc] init];    
self.person1.age = 1;

NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
[self.person1 addObserver:self forKeyPath:@"age" options:options context:nil];
```
添加观察监听回调并打印：
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    NSLog(@"被监听的 %@ 的值 %@ 改变为 %@", object, keyPath, change);
}
```
此时准备工作完成，当点击view时就会修改age的值，并且回调打印出监听的结果，这里在ViewController的touchedBegan中修改值：
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.person.age = 11;
}
```
记得在最后移除键值观察
```
- (void)dealloc {
    [self.person1 removeObserver:self forKeyPath:@"age"];
}
```
以上为KVO的基本使用。

#### 关于KVO的疑问和分析
再次添加属性 `@property (nonatomic, strong) Person * person2;`
实例化person2，在touchedBegan方法中修改值但是不添加KVO：
```
self.person2 = [[Person alloc] init];
self.person2.age = 2;

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.person.age = 11;
    self.person1.age = 22;
}

```
点击view可以看到打印台的日志为：

```
2018-07-17 14:09:26.944619+0800 KVO-KVC[36344:935709] 被监听的 <Person: 0x6040000106d0> 的值 age 改变为 {
    kind = 1;
    new = 11;
    old = 1;
}
```
这时就可以思考都是修改age属性值，为什么person1会有回调而person2没有，修改的本质都是调用age的set方法。猜想person1和person2的set方法实现可能不一样，但是实例方法都是存放在class中的，set方法应该是一样的才对，在`touchesBegan处打断点`，然后直接查看person1和person2的isa指针，看看person1和person2的class是否一样：

```
(lldb) p self.person1.isa
	(Class) $0 = NSKVONotifying_Person
  		Fix-it applied, fixed expression was: 
    	self.person1->isa
(lldb) p self.person2.isa
	(Class) $1 = Person
  		Fix-it applied, fixed expression was: 
    	self.person2->isa
```

可以看到person1的class为 `NSKVONotifying_Person` person2的class为 `Person` ，isa指针指向的就是instance的class，但是为什么person1和person2会不一样呢？我们在添加键值观察之前和之后分别打印person的类型：
```
NSLog(@"添加前 person1 : %@    person2 : %@", object_getClass(self.person1), object_getClass(self.person2));

NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person1 addObserver:self forKeyPath:@"age" options:options context:nil];
    
NSLog(@"添加后 person1 : %@    person2 : %@", object_getClass(self.person1), object_getClass(self.person2));
```
打印的结果为

```
2018-07-17 14:40:59.918227+0800 KVO-KVC[37038:970983] 添加前 person1 : Person    person2 : Person
2018-07-17 14:40:59.918636+0800 KVO-KVC[37038:970983] 添加后 person1 : NSKVONotifying_Person    person2 : Person
```

可以看到添加键值观察之后person1的isa指针指向确实被修改了，指向了 `NSKVONotifying_Person` 类，结合上面的猜想，会不会是 `NSKVONotifying_Person` 这个类重新实现了person1的 `setAge:` ，否则怎么会和person2不一样呢？
我们来验证一下，通过 `methodForSelector:` 来获取 `setAge:` 的实现：
```
NSLog(@"添加前 person1 : %p    person2 : %p",
          [self.person1 methodForSelector:@selector(setAge:)],
          [self.person2 methodForSelector:@selector(setAge:)]);
          
NSLog(@"添加后 person1 : %p    person2 : %p",
          [self.person1 methodForSelector:@selector(setAge:)],
          [self.person2 methodForSelector:@selector(setAge:)]);
```
打印的结果为

```
2018-07-17 14:46:56.489956+0800 KVO-KVC[37183:978368] 添加前 person1 : 0x102493570    person2 : 0x102493570
2018-07-17 14:46:56.490699+0800 KVO-KVC[37183:978368] 添加后 person1 : 0x1027d9bf4    person2 : 0x102493570
```

我们知道instance的方法、属性、协议等信息都存在与class中，所以当person1和person2调用 `setAge:` 时得到的地址应该是一样的，但是在添加键值观察之后person1的调用方法地址改变了，为什么会改变呢？让我们来看看这两个地址的IMP，在添加键值观察之后断点，直接查看两个地址的IMP：

```
(lldb) p (IMP)0x100a43570
(IMP) $0 = 0x0000000100a43570 (KVO-KVC -[Person setAge:] at Person.m:13)
(lldb) p (IMP)0x100d89bf4
(IMP) $1 = 0x0000000100d89bf4 (Foundation _NSSetLongLongValueAndNotify)
```

可以看到添加键值观察之后调用 `setAge:` 方法其实就是调用了 `Foundation _NSSetLongLongValueAndNotify` 

由此可以猜测在添加键值观察之后person1的isa指向了新生成的类 `NSKVONotifying_Person` ，`NSKVONotifying_Person` 可能继承自 `Person` 类，并且重写了 `setAge:` 方法，伪代码如下：
```
- (void)setAge:(NSInteger)age {
    _NSSetLongLongValueAndNotify();
}

void _NSSetLongLongValueAndNotify() {
    [self willChangeValueForKey:@"age"];
    [super setAge:age];
    [self didChangeValueForKey:@"age"];
}

- (void)didChangeValueForKey:(NSString *)key {
    [observer observeValueForKeyPath:key ofObject:self change:opetions context:nil];
}
```

**综上我们的猜想KVO的实现：instance添加键值观察之后isa指针会被修改为指向 `NSKVONotifying_Person` ，`NSKVONotifying_Person` 继承自 `Person` 并且重写了 `setAge:` 方法，方法实现如上。**
在这里就有了那道最经典的面试题：**如何手动实现KVO**，我们只需要在修改值的时候替换 `_NSSetLongLongValueAndNotify` 方法里面的 `[super setAge:age];` 就好了。

#### KVO内部实现窥探
由上我们猜测出了KVO的实现原理，下面我们来继续探索一下KVO内部的实现。
我们分别在添加KVO前后打印person1和person2的class，这次我们用两种方式：
```
NSLog(@"添加前 person1 : %@ -- %@   person2 : %@ -- %@", [self.person1 class], object_getClass(self.person1), [self.person2 class], object_getClass(self.person2));

NSLog(@"添加后 person1 : %@ -- %@   person2 : %@ -- %@", [self.person1 class], object_getClass(self.person1), [self.person2 class], object_getClass(self.person2));
```
打印出的结果为：
```
2018-07-19 11:05:50.553735+0800 KVO-KVC[40616:2560144] 添加前 person1 : Person -- Person   person2 : Person -- Person
2018-07-19 11:05:52.772905+0800 KVO-KVC[40616:2560144] 添加后 person1 : Person -- NSKVONotifying_Person   person2 : Person -- Person
```
可以看到我们通常用来获取class的方法在添加前后结果都是 `Person` ，通过runtime API获取到的class不相同，怎么回事呢？我们先来看一下苹果官方runtime的源码 [**这里**](https://opensource.apple.com/tarballs/objc4/)，当然官方的编译是失败，要想调试runtime的请看 [**这里**](https://github.com/RetVal/objc-runtime)。
我们来分析一下源码：
```
class方法：

+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}

runtime object_getClass方法：

Class object_getClass(id obj) {
    if (obj) return obj->getIsa();
    else return Nil;
}

```
`class` 的类方法或者实例方法最终返回的都是class的self，而 `object_getClass` 方法返回的是obj的isa指针，所以通过 `object_getClass` 获取的才是当前obj的真正class，所以在添加KVO之后person1的isa指针确确实实是被修改了。
我们再来看一下捕捉到的 `NSKVONotifying_Person` 到底是个什么鬼？
先来看一下 `NSKVONotifying_Person` 的meta-class：
```
NSLog(@"元类对象 person : %@    person1 : %@",
          object_getClass(object_getClass(self.person1)),
          object_getClass(object_getClass(self.person2)));
打印结果：
2018-07-19 11:39:30.210378+0800 KVO-KVC[41164:2599225] 元类对象 person : NSKVONotifying_Person    person1 : Person
```
`NSKVONotifying_Person` 的meta-class为 `NSKVONotifying_Person`。

在添加KVO之后打住断点，借用 **[DLIntrospection](https://github.com/delebedev/DLIntrospection)** 再来查看一下此时class里面方法都有什么：
```
(lldb) po [[self.person1 class] instanceMethods]
	<__NSArrayI 0x60400023daa0>(
		- (void)setAge:(q)arg0 ,
		- (q)age
	)
(lldb) po [object_getClass(self.person1) instanceMethods]
	<__NSArrayI 0x60400025fb30>(
		- (void)setAge:(q)arg0 ,
		- (class)class,
		- (void)dealloc,
		- (BOOL)_isKVOA
	)
```
结果可以看到 `NSKVONotifying_Person` 重写了 `setAge:` 方法，并且还有其他的三个方法，可证上面的猜想确实没错，`NSKVONotifying_Person`重写了 `setAge:` 方法，但是还有一个上面的猜想没有验证，那就是 `NSKVONotifying_Person` 的superClass到底是谁？
类似isa指针的方式，我们断点直接打印：
```
(lldb) po self.person1.superclass
NSObject

(lldb) po self.person2.superclass
NSObject
```
咦~~~ 等等，这跟我们猜测的不一样啊，怎么superclass都是NSObject呢？那我们的猜测是不是都错了？
为了看看superClass里面到底是什么下面我们请出 `clang` 大神：
`clang -rewrite-objc Person.m`
可以看出编译完成后Person类被编译成了这样：
```
struct NSObject_IMPL {
    Class isa;
};

struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    NSInteger _age;
};
```
结合runtime源码分析，Class为 `typedef struct objc_class *Class;` 类型的结构体，再看下结构体里面的结构：
```
struct objc_object {
private:
    isa_t isa;
    ···
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;
    class_data_bits_t bits;
    ···
}
```
里面确实有superclass，仿照runtime的结构我们自己来创建一个类似的结构体：
```
struct XFPerson_IMPL {
    Class isa;
    Class super_Class;
    NSInteger _age;
};
```
用我们自己创建的结构体来接收 `NSKVONotifying_Person` ，看看他的superclass到底是什么类型：
```
struct XFPerson_IMPL * xfPerson1 = (__bridge struct XFPerson_IMPL *)(object_getClass(self.person1));
struct XFPerson_IMPL * xfPerson2 = (__bridge struct XFPerson_IMPL *)(object_getClass(self.person2));

NSLog(@"person1--- %@", xfPerson1->super_Class);
NSLog(@"person2--- %@", xfPerson2->super_Class);

打印结果：
2018-07-19 14:05:38.855549+0800 KVO-KVC[43578:2717734] person1--- Person
2018-07-19 14:05:38.855658+0800 KVO-KVC[43578:2717734] person2--- NSObject
```
结果可见是符合我们的猜想的，`NSKVONotifying_Person` 确实是Person的子类，但是为什么上面直接打印instance的superclass却都是NSObject呢？
回过头来看一下上面我们找到的 `NSKVONotifying_Person` 除了 `setAge:` 还有三个方法，其中就有class方法，我们已经知道runtime的class的实现，class返回的就是self，而通过 `[self.person1 class]` 得到的是 `Person` ，这就证明了 `NSKVONotifying_Person` 重写了class方法，并且返回的是 `Person` 类，通过源码查看runtime的superclass方法的实现：
```
+ (Class)superclass {
    return self->superclass;
}

- (Class)superclass {
    return [self class]->superclass;
}
```
就是先通过class方法找到class，然后在根据class找到superclass，所以前面直接通过 `self.person1.superclass` 找到的是 `Person`，因为此时的class方法返回已经被修改了。

苹果大大可能是因为整个事件中 `NSKVONotifying_Person` 是个人畜无害的东西，对于开发者使用KVO是可以不用知道的，所以用这种方式来骗骗开发者，真不容易，还好最近看 **白夜追凶** 看的整个人都比较有耐心了就是要找到真相，哈(不)哈(要)哈(脸)😁。
再看看看其他的两个方法，`dealloc` 方法可能就是做一些销毁现场的事情，毕竟中间动态创建了 `NSKVONotifying_Person` ，不用了一定要销毁，而   `_isKVOA` 返回的一定是 YES ，表示当前确实是在用KVO，到此关于KVO的黑科技已经探究明白了，好了，打完收工，接着去看两集 **白夜追凶**， 哈哈哈。


