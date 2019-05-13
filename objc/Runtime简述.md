### Runtime概述
OC是面向对象的语言，具有面向对象语言特性：封装（setter和getter，方法、成员变量）、继承、多态<br/>
OC语言有面向对象特性、消息传递机制的特点，而这些都是通过RunTime实现的。<br/>
OC还是一门动态性语言，通过运行时系统(Runtime机制)可以实现动态地创建类和对象、动态添加属性、消息传递和转发等功能。编译的时候决定发送什么样的消息，运行时决定reciver怎样接受这个消息。<br/>
Runtime系统由一系列的函数和数据结构组成。<br/>
利用Runtime的动态性和AOP思想(切片思想Aspect Oriented Programming)可以把琐碎的事务逻辑从主逻辑中分离出来作为单独的模块开发。(比如无埋点，Logging)

### Runtime的调用方式
* 通过OC源代码调用，大多数情况下程序员只编写OC代码，Runtime系统会在幕后辛勤劳动
* NSObject类的方法，NSObject的抽象接口方法,如class、isKindOfClass、description等
* 直接调用Runtime函数，一般不会用到，当在写与其他语言桥接或底层bug工作时用到

### 数据结构
#### objc_object
```objectivec
typedef struct objc_object *id;
```
objc_object代表id类型的实例对象

```objectivec
struct objc_object {
     isa_t isa;
     ...
}

union isa_t {
     isa_t() {  }
     ...
     Class cls;
     uintptr_t bits;
     ...
}
```
`isa_t` 是一个[union联合体](https://blog.csdn.net/engerled/article/details/6205584)，也就是说isa_t、cls、bits公用同一块地址空间,不同时间存储不同的变量

#### objc_class
```objectivec
typedef struct objc_class *Class;
struct objc_class ：objc_object {
     Class superclass;
     cache_t cache;//缓存指针和vtable，加速方法调用
     class_data_bits_t bits; // class_rw_t* 加上flags，存储类实例的方法、属性、协议
     ....
}
```
可以看出OC的类也是一个结构体对象<br/>
`objc_class`继承自`objc_object`,自然也有isa成员变量<br/>
`cache` 缓存当前类已经加载过的方法，是一个bucket_t元素的散列表<br/>
`class_data_bits_t`存储类的实例方法、属性、协议，是`class_rw_t` 加上flags

```objectivec
struct class_rw_t {
    ...
    uint32_t flags;//各种标记位：meta、initializing、initialized、loaded、root等的标记位

    const class_ro_t *ro;

    method_array_t methods;//是method_t的数组
    property_array_t properties;
    protocol_array_t protocols;
    ...
};

struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;
    ...
};
```
`flags` 是一个32字节的数值，用于各种标记位，如是否是meta、是否是root类、是否已经loaded、是否正在initializing，是否已经initialized等<br/>
`class_ro_t`存储编译时期确定的方法、属性、协议<br/>
`method_array_t` 存储运行时加载的所有方法，主类和分类都有，分类方法在前，消息传递中在这个数组中找到相同方法名就结束，不会继续往后找，所以分类会覆盖主类的方法<br/>

```objectivec
struct cache_t {
    struct bucket_t *_buckets;//散列表,元素是bucket_t
    mask_t _mask;//散列表总容量
    mask_t _occupied;//散列表已使用容量
    ...
};

struct bucket_t {
     ....
    cache_key_t _key;//方法名@selector
    MethodCacheIMP _imp;//方法的实现地址IMP
     ....
};
```
`cache_t` 缓存的是方法名和IMP，实现是一个hash表，通过f(key)获取对应的IMP<br/>

### isa的理解
* OC中所有的实例对象、类在Runtime中都理解成是一个结构体对象
* 从objc_object和objc_class结构体可以看出，对象和类都有一个isa_t结构体
* objc_object结构体中的isa指针指向objc_class，objc_class中的isa指针指向metaClass
* 引入元类是为了保证类方法和实例方法的调用机制相同
     1.实例方法调用时，通过对象的isa在类中查找方法
     2.类方法调用时，通过类的isa在元类中查找方法

![instance-calss-meta](image/instance-class-metaClass.png)

### 遗留问题
```objectivec
struct bucket_t {
private:
    // IMP-first is better for arm64e ptrauth and no worse for arm64.
    // SEL-first is better for armv7* and i386 and x86_64.
#if __arm64__
    MethodCacheIMP _imp;
    cache_key_t _key;
#else
    cache_key_t _key;
    MethodCacheIMP _imp;
#endif
    ...
};
```
这里的注释说明，不同架构下结构体成员变量的先后顺序会影响性能，这个的依据是什么？



