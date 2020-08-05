---
title: category原理探究-2
date: 2018-07-22 12:55:46
tags:
---
### +load 方法解析

<!-- more -->

分析load方法前先来做一个小测验：

```
// Animal 类
@interface Animal : NSObject
@end
@implementation Animal
+ (void)load {
    NSLog(@"Animal -- load");
}
@end

// Dog 类 (->superclass animal)
@interface Dog : Animal
@end
@implementation Dog
+ (void)load {
    NSLog(@"Dog -- load");
}
@end

// Cat 类 (->superclass animal)
@interface Cat : Animal
@end
@implementation Cat
+ (void)load {
    NSLog(@"Cat -- load");
}

// Animal 分类
@interface Animal (Eat)
@end
@implementation Animal (Eat)
+ (void)load {
    NSLog(@"Animal (Eat) -- load");
}
@end

// Dog 分类
@interface Dog (Eat)
@end
@implementation Dog (Eat)
+ (void)load {
    NSLog(@"Dog (Eat) -- load");
}
@end

// Cat 分类
@interface Cat (Eat)
@end
@implementation Cat (Eat)
+ (void)load {
    NSLog(@"Cat (Eat) -- load");
}
@end
```
我们来运行一下程序，发现没有调用这些类但是load方法都加载了，加载的结果如下：

```
2018-07-22 15:36:46.369503+0800 debug-objc[11354:638134] Animal -- load
2018-07-22 15:36:46.369954+0800 debug-objc[11354:638134] Cat -- load
2018-07-22 15:36:46.369965+0800 debug-objc[11354:638134] Dog -- load
2018-07-22 15:36:46.369981+0800 debug-objc[11354:638134] Cat (Eat) -- load
2018-07-22 15:36:46.369996+0800 debug-objc[11354:638134] Dog (Eat) -- load
2018-07-22 15:36:46.370009+0800 debug-objc[11354:638134] Animal (Eat) -- load
```
[上一篇](https://jueying-xiangfeng.github.io/2018/07/21/category%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6-1/)中我们讲到了category的原理，猜想这里的结果应该是只会调用分类的方法，可是为什么这里都调用了呢？还有看一下 `PROJECT->TARGETS->Build Phases->Compole Source` 的编译顺序，分类的编译顺序和调用顺序能对上，但是cat和dog都在animal的前面，为什么会是先调用的animal的load方法呢?

我们来分析一下load方法在runtime的源码，老规矩，先下载runtime源码，找到入口 `_objc_init` 的 `load_images` 方法，看一下上面的官方注解：`Process +load in the given images which are being mapped in by dyld.` ，可以发现load方法就是在这里面加载的，看源码：

```
load_images(const char *path __unused, const struct mach_header *mh) {
    /**
    * 准备 load methods
    */
    prepare_load_methods((const headerType *)mh);
    /**
    * 调用 load methods
    */
    call_load_methods();
}
```
这两个方法我们拿出来单独看一下：

**a>** `prepare_load_methods`：

```
void prepare_load_methods(const headerType *mhdr) {
    /**
    * 加载所有类
    */
    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        /**
        * 调用 class 的 load 方法
        */    
        schedule_class_load(remapClass(classlist[i]));
    }
    /**
    * 加载所有分类
    */
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0； i < count；i++) {
        category_t *cat = categorylist[i]；
        /**
        * 将category中的load方法加载到loadable的列表中
        */
        add_category_to_loadable_list(cat);
    }
}
```
可以看到，这里永远都是先加载类的load方法然后在加载分类的load方法，正好解释我们上面的小测试的结果，类的load方法会先与分类被调用。再把 `schedule_class_load` 方法拿出来看一下：

```
static void schedule_class_load(Class cls) {
    /**
    * 判断如果已经加载过load方法则直接返回
    */
    if (cls->data()->flags & RW_LOADED) return;
    /**
    * Ensure superclass-first ordering
    * 这里是递归调用，优先加载 class的superclass的load方法，直到superclass为空
    */
    schedule_class_load(cls->superclass);
    /**
    * 将类的load方法加载到loadable的类表中
    */
    add_class_to_loadable_list(cls);
    /**
    * 设置此类的load方法标志位，表示已经加载过load方法
    */
    cls->setInfo(RW_LOADED); 
}
```
这里的调用非常巧妙，用递归调用优先加载superclass的load方法，这里就明白了原来在加载类的load方法时会优先加载superclass的load方法，到这里明白了上面的小测验为什么先回调 animal 的load方法了吗？
再看一下 `add_class_to_loadable_list` 方法：

```
void add_class_to_loadable_list(Class cls) {
    IMP method;
    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    /**
    * 这里是将cls的method方法存放到 loadable_classes 数组中，
    * loadable_classes_used++ 可以看出这里是按编译时的顺序加载的
    */
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```
注意：上面获取 method 的时候是获取的load method 的 `IMP` 。

`prepare_load_methods` 的分类加载的时候调用的就是 `add_class_to_loadable_list` 方法，所以分类的加载顺序就是按编译的顺序加载的。

到这里所有的类和分类的数据load完毕，并且加载顺序也已经确定。

**b>** `call_load_methods`：

```
void call_load_methods(void) {
    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);
}
/**
* call_class_loads 方法
*/
static void call_class_loads(void) {
    int i;
    /**
    * loadable_classes 就是我们上面 prepare 的加载完成的数据
    */
    struct loadable_class *classes = loadable_classes;
    /**
    * i ++  可以看到这里是按着顺序加载我们在 prepare 时准备的数据的
    */
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 
        /**
        * 调用 load 方法
        */	
        (*load_method)(cls, SEL_load);
    }
}
```
看上面的call_load_methods方法，先加载类在加载分类，在加载类的方法 `call_class_loads` 中是按着我们在 `prepare_load_methods` 方法中准备好的数据顺序执行的，所有prepare时加载的顺序就是load调用的顺序，可以查看分类的 `call_category_loads` 方法，跟加载类的方式是一样的。

到这里类和分类的所有load方法调用完毕。

`关于load方法总结`：
> 1. 调用方式：根据函数的 IMP 直接调用。
> 2. 调用时机：在runtime加载类、分类时调用（只调用一次）
> 3. 调用顺序："1、a> 先调用类的load(优先编译的优先调用)"  "b> 调用子类的load方法之前会先调用父类的load方法"  "1、a> 再调用分类的load方法(优先编译的分类优先调用)"

### initialize 方法


`我们知道 + initialize 方法会在类的第一次接收消息时调用。`

将上面的小测试的所有load替换为initialize，然后分别调用如下：

```
第一次调用：
[Animal class];

打印结果：
2018-07-22 16:52:15.039883+0800 debug-objc[12499:710327] Animal (Eat) -- initialize

第二次调用：
[Dog class];

打印结果：
2018-07-22 16:53:35.175441+0800 debug-objc[12534:711843] Animal (Eat) -- initialize
2018-07-22 16:53:35.175637+0800 debug-objc[12534:711843] Dog (Eat) -- initialize

第三次调用：
[Animal class];
[Dog class];

打印结果：
2018-07-22 16:54:04.664353+0800 debug-objc[12553:712521] Animal (Eat) -- initialize
2018-07-22 16:54:04.664578+0800 debug-objc[12553:712521] Dog (Eat) -- initialize
```
看到三次的打印结果可以推测 `initialize` 的方法的调用符合我们上一篇文章中讲解，最后加载的分类会优先类调用，并且类的方法不会再调用，第二次的调用可以推测在调用 `initialize` 时会先调用父类的 `initialize` 方法。

下面我们来把子类Dog和dog分类的 `initialize` 方式删除，再来打印看看结果：

```
调用：
[Dog class];

打印结果：
2018-07-22 17:08:29.497283+0800 debug-objc[12830:725499] Animal (Eat) -- initialize
2018-07-22 17:08:29.497558+0800 debug-objc[12830:725499] Animal (Eat) -- initialize
```
为什么 Animnal 的 `initialize` 会出现两次的调用呢？ `initialize` 不是只在第一次接收消息时调用吗？

initialize 是在发送消息时调用的，所以我们找到 `objc_msgSend` 方法，最终找到 `class_getInstanceMethod` 方法，根据调用：`->class_getInstanceMethod -> lookUpImpOrNil ->lookUpImpOrForward` 下面我们来分析一下runtime的源码：

```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    /**
    * 如果cls还没有 Initialized 调用 _class_initialize
    */
    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
    }
}

void _class_initialize(Class cls) {
    Class supercls;
    /**
    * 递归调用 cls父类的 callInitialize
    */
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    /**
    * 没有 initialize 时设置 reallyInitialize为true
    */
    if (!cls->isInitialized() && !cls->isInitializing()) {
        cls->setInitializing();
        reallyInitialize = YES;
    }
    if (reallyInitialize) {
        /**
        * 调用返送 SEL_initialize 消息方法
        */
        callInitialize(cls);
    }
}
/**
* 发送 SEL_initialize 消息
*/
void callInitialize(Class cls) {
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
}
```
分析源码可以看到，`initialize` 方法会先去查找父类，如果父类 `initialize` 没有调用过，会先向父类发送 `SEL_initialize` 消息，这里可以解释一下我们上面的小测试为什么 animal 的方法调用了两次，因为在 `_class_initialize` 方法中第一次由父类 animal 发送 `SEL_initialize` 消息，第二次由 dog 类发送 `SEL_initialize` 消息，而由于 dog 类没有 `initialize` 方法，所以会去调用父类 animal 的方法。

`关于initialize方法总结`：
> 1. 调用方式：initialize是通过objc_msgSend调用。
> 2. 调用时机：initialize时类在第一次接收到消息时调用，每个类只会initialize一次（父类的initialize方法可能会被多次调用）。
> 3. 调用顺序：先初始化父类 再初始化子类（可能最终调用的是父类的initialize方法）


