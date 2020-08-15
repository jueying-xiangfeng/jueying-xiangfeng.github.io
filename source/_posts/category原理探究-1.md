---
title: category原理探究-1
date: 2018-07-21 14:40:13
tag: category 原理
toc: true
description: 
---

### category探究准备
先来创建我们测试需要的类：

```
<!-- Animal类 -->
@interface Animal : NSObject
- (void)animal;
@end
<!-- Animal+Eat分类 -->
@interface Animal (Eat) <NSCopying, NSCoding>
@property (nonatomic, assign) int age;
- (void)eat;
@end
<!-- Animal+Play分类 -->
@interface Animal (Play)
- (void)play;
@end
```
以Eat分类为例，请出 `clang` 命令：`clang -rewrite-objc Animal+Eat.m` ，生成.cpp文件。

<!-- more -->

### category的真面目
在.cpp文件最下面可以找到category被编译后的结构：

```
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};
```
+ `name` 这里的name表示的是 `类名` 而不是category的名字。
+ `cls` 要扩展的类对象，编译期间值为空，在被runtime加载时根据name对应到类对象。
+ `instance_methods` category所有的实例方法。
+ `class_methods` category所有的类方法。
+ `protocols` category实现的所有协议。
+ `properties` category的所有属性。

再来看看我们的Animal+Eat被编译成了什么：

```
static struct _category_t _OBJC_$_CATEGORY_Animal_$_Eat __attribute__ ((used, section ("__DATA,__objc_const"))) =  {
	"Animal",
	0, // &OBJC_CLASS_$_Animal,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Animal_$_Eat,
	0,
	(const struct _protocol_list_t *)&_OBJC_CATEGORY_PROTOCOLS_$_Animal_$_Eat,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Animal_$_Eat,
};
```
看一下结构体的名称：`_OBJC_$_CATEGORY_Animal_$_Eat`，最后面的Eat就是我们分类的名称，前面有表示CATEGORY和类名Animal，这也就是为什么同一个类的category不能重名的原因了。
再对应一下其他的结构，例如instance_methods：

```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_Animal_$_Eat __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"eat", "v16@0:8", (void *)_I_Animal_Eat_eat}}
};
```
我们里面只有一个 `eat` 方法，被编译后为 `_I_Animal_Eat_eat`。

最后可以看到所有的category被放到了一个数组中，存在了 `__DATA` 段下的 `__objc_catlist section` 里了：

```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_Animal_$_Eat,
};
```
这里编译期间的工作就做完了，接下来进入runtime。

### runtime加载category

先下载一下苹果官方runtime的源码 [**这里**](https://opensource.apple.com/tarballs/objc4/)，当然官方的编译是失败，要想调试runtime的请看 [**这里**](https://github.com/RetVal/objc-runtime)。

大致加载的流程如下：

+ 找到runtime的入口：`objc-os.mm` 的 `_objc_init` 方法，在library加载前由libSystem dyld调用，进行初始化操作。
+ 调用map_images方法将文件中的image map到内存。
+ 调用_read_images方法初始化map后的image。
+ 找到 `Discover categories` 可以看到 `category_t` 是通过 `_getObjc2CategoryList` 方法初始化的，这个方法拿出来看看：

```
#define GETSECT(name, type, sectname)                                   \
    type *name(const headerType *mhdr, size_t *outCount) {              \
        return getDataSection<type>(mhdr, sectname, nil, outCount);     \
    }                                                                   \
    type *name(const header_info *hi, size_t *outCount) {               \
        return getDataSection<type>(hi->mhdr(), sectname, nil, outCount); \
    }
    
GETSECT(_getObjc2CategoryList, category_t *, "__objc_catlist");
```
看到这里有没有很熟悉，在这里加载的 `__objc_catlist` 就是在编译期间存放的数据。

来看一下加载的源码：

```
for (EACH_HEADER) {

   /**
   * 取出 category 数据 此处为数组代表一个类所有的分类
   */
   category_t **catlist = _getObjc2CategoryList(hi, &count);

   for (i = 0; i < count; i++) {
	   
	  /**
	  * 按顺序取出 category_t 
	  */
	  category_t *cat = catlist[i];
	  /**
	  * remapClass：加载category_t的class指针
	  */
	  Class cls = remapClass(cat->cls);
	  
	  if (cat->instanceMethods ||  cat->protocols  ||  cat->instanceProperties) 
	  {
	      addUnattachedCategoryForClass(cat, cls, hi);
	      if (cls->isRealized()) {
	      	remethodizeClass(cls);
	      }
	  }
	
	  if (cat->classMethods  ||  cat->protocols ||  (hasClassProperties && cat->_classProperties)) 
	  {
	      addUnattachedCategoryForClass(cat, cls->ISA(), hi);
	      if (cls->ISA()->isRealized()) {
	      	remethodizeClass(cls->ISA());
	      }
	  }
   }
}
```
可以看到每次循环中 `category_t` 的加载 `addUnattachedCategoryForClass` 方法有两个调用，对比一下参数可以发现第二个参数不同 cls 和 cls->ISA()，再结合判断条件的 cat->instanceMethods 和 cat->classMethods，这两次的加载是将category中的信息分别加载到类和元类中，然后再调用 `remethodizeClass` 重新组织结构。接下来调用附加信息的方法 `attachCategories` ，将分类的信息附加到类中：

```
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    /**
    * 是否为元类
    */ 	
    bool isMeta = cls->isMetaClass();

    /**
    * 方法数组
    */
    method_list_t **mlists = (method_list_t **) 
        malloc(cats->count * sizeof(*mlists));
    /**
    * 属性数组
    */
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
   /**
    * 协议数组
    */
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));
    /**
    ********** 注意 ：这里是倒序循环 **********
    */
    while (i--) {
        /**
        * 取出某个分类
        */
        auto& entry = cats->list[i];
        /**
        * 取出某个分类的方法列表 (根据 isMeta 来判断取实例方法还是类方法)
        */
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            /**
            * 将分类方法列表正序添加到 mlists
            */        
            mlists[mcount++] = mlist;
        }
        /**
        * 取出某个分类的属性列表
        */
        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            /**
            * 将分类属性列表正序添加到 proplists
            */  
            proplists[propcount++] = proplist;
        }
        /**
        * 取出某个分类的协议列表
        */
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            /**
            * 将分类协议列表正序添加到 protolists
            */  
            protolists[protocount++] = protolist;
        }
    }
    /**
    * 取出类的信息数据  class_rw_t
    */
    auto rw = cls->data();
    /**
    * 初始化方法的一些信息，比如有没有实现retain、release、allocWithZone等方法。
    */
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    /**
    * 将所有分类的方法、属性、协议列表附加到类的方法、属性、协议列表中。
    */
    rw->methods.attachLists(mlists, mcount);
    rw->properties.attachLists(proplists, propcount);
    rw->protocols.attachLists(protolists, protocount);
}
```
由上面的while循环可以看到加载方法、协议、属性的时候是 `倒序` 加载的，是不是想到了什么？如果Animal类和两个分类都有一个 `-(void)run` 方法，那么最终会调用哪个里面的run方法呢？答案当然是最后加载的那个run方法，不过没有被调用的run方法并没有被 `覆盖` ，方法还在那里只是按顺序没有被调用。

最后看一下 `methods.attachLists` 方法：

```
/**
* 将分类的 方法、协议、属性等信息附加到类中
*/
void attachLists(List* const * addedLists, uint32_t addedCount) {
	 
	uint32_t oldCount = array()->count;
	uint32_t newCount = oldCount + addedCount;
	/**
	* 重新分配内存（大小为： oldCount addedCount 原有count和要添加的count总和）
	*/ 
	setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
	array()->count = newCount;
	/**
	* 重新布局
	*/
	memmove(array()->lists + addedCount, array()->lists,
		 oldCount * sizeof(array()->lists[0]));
	memcpy(array()->lists, addedLists,
		 addedCount * sizeof(array()->lists[0]));
}
```
重新布局的时候有两个方法：
+ `memmove`：`void *memmove(void *__dst, const void *__src, size_t __len);` 可以看到是将src变量的数据移动到dst，所以最终是将 array()->lists 的数据移动到了 array()->lists + addedCount 的位置。
+ `memcpy`：`void	*memcpy(void *__dst, const void *__src, size_t __n);` 可以看到是将src变量的数据copy到dst，所以最终是将分类中的信息 `addedLists` copy 到 array()->lists 的位置。

正如我们上面说的run方法，Animal类中的run方法是被最后加载的，因为Animal类中的方法列表被移动到了分类的后面，加载的时候会先调用分类中的方法，而且可以看到Animal中的run方法确实没有被覆盖，只是调用的时候发现分类中有不会再调用Animal的run方法而已。

### class extention与category

上面知道了category，我们再来看看class extention，class extention算是一种特殊的分类（匿名分类），那么我们可以思考平时在 .m 文件的匿名分类中写的私有属性、方法等在加载的时候会不会和分类一样呢？我们来验证一下，在Animal的 .m 文件里添加属性 height 和方法 test：

```
@interface Animal ()
@property (nonatomic, assign) int height;
- (void)test;
@end

@implementation Animal
- (void)animal {
    
}

- (void)test {
    
}
@end
```
用clang命令来编译 Animal：`clang -rewrite-objc Animal.m`

```
/**
* 元类结构
*/
static struct _class_ro_t _OBJC_METACLASS_RO_$_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1, sizeof(struct _class_t), sizeof(struct _class_t), 
	(unsigned int)0, 
	0, 
	"Animal",
	0, 
	0, 
	0, 
	0, 
	0, 
};
/**
* 类结构
*/
static struct _class_ro_t _OBJC_CLASS_RO_$_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, __OFFSETOFIVAR__(struct Animal, _height), sizeof(struct Animal_IMPL), 
	(unsigned int)0, 
	0, 
	"Animal",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Animal,
	0, 
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Animal,
	0, 
	0, 
};
/***************************************/
/**
* 可以看到类中方法列表 ‘_INSTANCE_METHODS_Animal’对应下面的结构
* animal、test、height、setHeight 方法都在类结构中
*/
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[4];
} _OBJC_$_INSTANCE_METHODS_Animal __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	4,
	{{(struct objc_selector *)"animal", "v16@0:8", (void *)_I_Animal_animal},
	{(struct objc_selector *)"test", "v16@0:8", (void *)_I_Animal_test},
	{(struct objc_selector *)"height", "i16@0:8", (void *)_I_Animal_height},
	{(struct objc_selector *)"setHeight:", "v20@0:8i16", (void *)_I_Animal_setHeight_}}
};
```
编译的结果如上，可以看到匿名类别的编译结果并不是 `category_t` 的类型在 runtime 时加载的，而是直接在编译期间将相关的属性方法等加载到了类中，匿名分类声明的属性方法相当于在类的 .h 文件的声明。


