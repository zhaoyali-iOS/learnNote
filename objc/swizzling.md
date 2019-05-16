##
#### 
OC中的Method swizzling是一个强大的黑魔法。它允许我们动态地替换方法的实现。比如利用切片思想可以十分高效地处理埋点逻辑,使业务逻辑和埋点逻辑分开，大大改善项目的耦合性。<br/>

## 使用到的方法
```objectivec
OBJC_EXPORT Method _Nullable
class_getInstanceMethod(Class _Nullable cls, SEL _Nonnull name)
```
`class_getInstanceMethod`获取对应类的实例方法，它的实现中会从当前类一直向上查找，直到NSObject，所以返回值可能是当前类的实现，也可能是父类的实现，或者nil
<br/>
<br/>

```objectivec
OBJC_EXPORT IMP _Nonnull
method_getImplementation(Method _Nonnull m)
```
获取`Method`对应的`IMP`实现，Method为空就返回nil
<br/>
<br/>

```objectivec
OBJC_EXPORT const char * _Nullable
method_getTypeEncoding(Method _Nonnull m) 
```
获取`Method`对应的参数和返回值类型描述的字符串，[字符编码规范](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)可以从这里查看
<br/>
<br/>

```objectivec
OBJC_EXPORT BOOL
class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, 
                const char * _Nullable types) 
```
`class_addMethod` 会调用到`addMethod`，传入replace为NO
```objectivec
static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;
    ...
    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists，通过replace标记为来替换或返回
        if (!replace) {
            result = m->imp;
        } else {
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
      //添加方法
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdupIfMutable(types);
        newlist->first.imp = imp;

        prepareMethodLists(cls, &newlist, 1, NO, NO);
        cls->data()->methods.attachLists(&newlist, 1);
        flushCaches(cls);

        result = nil;
    }

    return result;
}

//替换method的IMP
static IMP 
_method_setImplementation(Class cls, method_t *m, IMP imp)
{
    ...
    if (!m) return nil;
    if (!imp) return nil;

    IMP old = m->imp;
    m->imp = imp;
    
    flushCaches(cls);

    updateCustomRR_AWZ(cls, m);

    return old;
}
```
`class_addMethod`实现的功能是：给类动态添加方法和实现。方法特点如下：
* `getMethodNoSuper_nolock`说明只检查当前类中是否实现，不往上找
* 当前类已经实现方法就不添加返回NO，也不会替换方法实现(replace参数为NO)
* 当前类没有实现方法就添加并返回YES,添加时没有判断SEl、IMP是否为空，所以这种情况下有可能添加的Method的IMP指正为nil,当程序调用此SEL时就会crash
<br/>
<br/>

```objectivec
//替换SEL的实现
IMP 
class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;

    mutex_locker_t lock(runtimeLock);
    return addMethod(cls, name, imp, types ?: "", YES);
}
```
`class_replaceMethod`也会调用到`addMethod`,传入的replace为YES,特点如下：
* `getMethodNoSuper_nolock`说明只检查当前类中是否实现，不往上找
* 当前类已经实现方法就替换方法实现(replace参数为YES)，替换时检查入参，IMP有值就替换，否则保持原来实现不变
* 当前类没有实现方法就添加,同样没有判断SEl、IMP是否为空，当程序在调用此SEL时也会crash
<br/>
<br/>

```objectivec
//直接替换两个method的IMP
void method_exchangeImplementations(Method m1, Method m2)
{
    if (!m1  ||  !m2) return;

    mutex_locker_t lock(runtimeLock);

    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;

    flushCaches(nil);
    ...
}
```
替换时会检查入参，但没有进一步检查IMP是否有值，同样也存在因为IMP为空而crash的风险
<br/>
<br/>

## 应用实例
```objectivec
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
    
        Class aClass = [self class];
    
        SEL originalSelector = @selector(cc);
        SEL swizzledSelector = @selector(xxx_cc);
        
        //class_getInstanceMethod实现中会从当前类开始往上找，返回的Method有可能是父类的实现
        //这里尤其要保证swizzled方法是当前类已经实现的方法
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector);
        
        BOOL didAddMethod =
        class_addMethod(aClass,
                        originalSelector,
                        method_getImplementation(swizzledMethod),
                        method_getTypeEncoding(swizzledMethod));
                        
        if (didAddMethod) {
            class_replaceMethod(aClass,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}
```
这里要注意的点：
* 使用dispatch是为了避免程序员手动调用此方法，导致多次交换方法
* 注意操作的是类方法还是实例方法，类方法要在metaClass中找IMP(objc_getClass([self class]) )，实例方法在Class中找IMP([self class])
* `class_getInstanceMethod`方法的特点要注意，会在父类中查找，为了避免错误，最好保证两个方法在当前类中都已经实现了

## 总结
swizzing是把双刃剑，给我们带来便利的同时也极容易出错，因此在使用是要注意细节
* swizzing的实现时机是load而不是initialize，因为initialize会执行多次且分类会覆盖主类和其他分类的实现
* swizzing的影响范围是全局的，所以要考虑交换之后是否影响正常逻辑，其他库是否受牵连
* swizzing对类簇不起作用，类簇的内部实现是工厂模式
* swizzing的方法最好保证当前类中都已经实现，而不是只在父类中实现，进而避免未知的问题（循环调用，SEL未实现，命名冲突等问题）
* 分类中load方法的调用顺序不确定，所以swizzing方法的实现逻辑中最好不要出现依赖主类或父类成员变量的逻辑
* swizzing会修改method对应的IMP，所以方法实现中的_cmd值已经不在是我们编写代码时预期的值了，尤其在关联对象的setter和getter方法中一般都会应用到_cmd
* 出于代码安全考虑，方法交换时要调用原方法
