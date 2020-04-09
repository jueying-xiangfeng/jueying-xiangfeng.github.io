---
title: category原理探究-3
date: 2018-07-22 18:40:53
tags:
---
### 关联对象的使用

```
@interface Animal (Eat)
@property (nonatomic, copy) NSString * name;
@end

@implementation Animal (Eat)
- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, _cmd);
}
@end
```
`key值几种使用方法`
> 1.static void *AnimalKey = &AnimalKey;
> 2.static char AnimalKey;
> 3.使用属性名作为key;
> 4.使用get方法的@selector作为key;

### 关联对象的源码解读


```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    /**
    * retain the new value (if any) outside the lock.
    * 用来记录原有关联值的变量
    */ 
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        /**
        * 全局的manager
        */    
        AssociationsManager manager;
        /**
        * 获取 AssociationsHashMap
        */
        AssociationsHashMap &associations(manager.associations());
        /**
        * 根据 object 获取disguised_ptr_t
        */
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            /**
            * break any existing association.
            * 根据 disguised_ptr_t 找到对应的AssociationsHashMap
            */ 
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            /**
            * 如果找到了AssociationsHashMap 则取出里面对应的 ObjectAssociationMaps
            * 根据 key 取出ObjectAssociationMaps 里面的ObjectAssociationMap
            */
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                /**
                * 如果存在key对应的ObjectAssociationMap,则取出里面的数据:
                * ObjcAssociation, 否则新建ObjcAssociation 并将ObjcAssociation以key为键存入ObjectAssociationMap中.
                */
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                /**
                * create the new association (first time).
                * 没有找到disguised_object对应的AssociationsHashMap,则创建一个并存入
                */
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            /**
            * setting the association to nil breaks the association.
            * 当要存入的 new_value为空时,查找对应的数据空间,如果找到用old_association记录存值的内存,并将ObjectAssociationMap里面的数据擦除
            */ 
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    /**
    * release the old value (outside of the lock).
    * 如果old_association.hasValue的值不为空的话则释放这块内存的数据
    */ 
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
```
上面是关联对象时数据的结构.

看一下AssociationsManager的数据结构:

```
<!-- AssociationsManager -->
class AssociationsManager {
    // associative references: object pointer -> PtrPtrHashMap.
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};

<!-- AssociationsHashMap -->
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap *, DisguisedPointerHash, DisguisedPointerEqual, AssociationsHashMapAllocator> {};

<!-- ObjectAssociationMap -->
class ObjectAssociationMap : public std::map<void *, ObjcAssociation, ObjectPointerLess, ObjectAssociationMapAllocator> {};

<!-- ObjcAssociation -->
class ObjcAssociation {
	uintptr_t _policy;
	id _value;
}
```
可以看到 ObjcAssociation 里面存储的就是我们存贮相关的策略和值: _policy : _value.

void objc_setAssociateObject(id object, const void *key, id value, Objc_AssociationPolicy policy);

### 总结

> 关联对象并不是存储在被关联对象本身内存中
> 关联对象存储在全局的统一的一个AssociatesManager中
> 设置关联对象为nil,就相当于是移除关联对象

