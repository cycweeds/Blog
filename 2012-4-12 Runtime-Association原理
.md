# **Runtime-Association原理**
## **前言**
如何在iOS开发中，向一个类动态添加的实例变量？  
类在运行时就内存布局就已经分布好了，不能添加 **Ivar(Instance variable)** 。
## **使用**
我们可以通过关联对象方式，对已经存在的类在扩展中添加自定义的属性。

```
- (void)setAssociatedObject:(id)object {
     objc_setAssociatedObject(self, @selector(associatedObject), object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)associatedObject {
    return objc_getAssociatedObject(self, @selector(associatedObject));
}
```
## **原理**

先来看关联对象的三个调用方法：

+ **objc_setAssociatedObject**
+ **objc_getAssociatedObject**
+ **objc_removeAssociatedObjects**

### **objc_setAssociatedObject**

```
void
objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{
    SetAssocHook.get()(object, key, value, policy); // 最后会调用 _base_objc_setAssociatedObject 方法
}

static void
_base_objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{
  _object_set_associative_reference(object, key, value, policy);
}

// ChainedHookFunction 为一个类    为了线程安全的链接钩子函数的存储。
// 目的是get()和set()使用适当的屏障，以便旧值在新值调用来使用之前被安全地写入变量（ChainedHookFunction 类的注释 翻译）


static ChainedHookFunction<objc_hook_setAssociatedObject> SetAssocHook{_base_objc_setAssociatedObject};

void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    // This code used to work when nil was passed for object and key. Some code
    // probably relies on that to not crash. Check and handle it explicitly.
    // rdar://problem/44094390
    if (!object && !value) return;

    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
    // 伪装对象 不被类似leaks这些调式工具发现具体的对象
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value};

    // retain the new value (if any) outside the lock.
    association.acquireValue();

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());

        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            if (refs_result.second) {
                /* it's the first association we make */
                object->setHasAssociatedObjects();
            }

            /* establish or replace the association */
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);

                    }
                }
            }
        }
    }

    // release the old value (outside of the lock).
    association.releaseHeldValue();
}

```
### **objc_getAssociatedObject**
```
id
objc_getAssociatedObject(id object, const void *key)
{
    return _object_get_associative_reference(object, key);
}

id
_object_get_associative_reference(id object, const void *key)
{
    ObjcAssociation association{};

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            ObjectAssociationMap &refs = i->second;
            ObjectAssociationMap::iterator j = refs.find(key);
            if (j != refs.end()) {
                association = j->second;
                association.retainReturnedValue();
            }
        }
    }

    return association.autoreleaseReturnedValue();
}
```

### **objc_removeAssociatedObjects**
```
void objc_removeAssociatedObjects(id object)
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}

void
_object_remove_assocations(id object)
{
    ObjectAssociationMap refs{};

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            refs.swap(i->second);
            associations.erase(i);
        }
    }

    // release everything (outside of the lock).
    for (auto &i: refs) {
        i.second.releaseHeldValue();
    }
}
```
> + 在项目中我们应该避免使用这个函数，会导致所有的关联对象都被释放。我们可以调用 setAssociatedObject(nil, key)的方式。
>
> + 在 [对象的初始化以及释放.md](https://github.com/cycweeds/blog/blob/main/2021-4-9%20%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BB%A5%E5%8F%8A%E9%87%8A%E6%94%BE.md)一文中已经讲解到，在对象释放的时候会自动释放 **Associatedobjects**。业务中也少有会调用此方法。




下面我们来详细讲解上述代码中的各个类的作用：
方便理解，先上图(引用他人https://blog.csdn.net/tanxianbo/article/details/104619987)

![图](https://img-blog.csdnimg.cn/20200303233255682.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RhbnhpYW5ibw==,size_16,color_FFFFFF,t_70)

```
// 他是一个单例  存放着所有的对象的association
class AssociationsManager {
    using Storage = ExplicitInitDenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap>;
    static Storage _mapStorage;

public:
     Manager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }

    AssociationsHashMap &get() {
        return _mapStorage.get();
    }

    // 在runtime 初始化的时候调用
    static void init() {
        _mapStorage.init();
    }
};
```

```
// 理解为 [key: (value, policy)]
typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
// 理解为 [object: ObjectAssociationMap]
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;

class ObjcAssociation {
    uintptr_t _policy;
    id _value;
    // 下面代码省略
  }
```

_mapStorage简化成字典格式的话  
  **[DisguisedPtr(object): ObjectAssociationMap([key: ObjcAssociation(value, policy)])]**
