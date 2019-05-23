## ARC简述
我们知道，由于手机资源有限，程序运行过程中要占用内存资源，为了在有限的资源下程序能流程运行，就需要对内存进行管理来：
1. 及时回收不再使用的资源
2. 怎样判断资源不再被使用
3. 解决资源不会被释放的问题--循环引用
4. 解决资源回收后野指针问题--野指针

### 引用计数原理
苹果采用`引用计数`原理区分资源是否还再被使用，来控制资源释放回收的时机
* 苹果采用`retainCount`来表示资源被使用的地方个数，其值为0表示没有地方再使用，就应该释放并回收资源；其值>0表示资源还在被使用不应该回收。
* 每有一处要使用资源`retainCount`就+1，每有一处停止使用资源`retainCount`就-1。
* 资源都是因为需要而分配创建的，所以资源创建的同时`retainCount`值置为1
* `retainCount`的加减操作，是xcode在编译是自动插入的，我们只需要告诉编译器对象之间的引用关系即可。这样我们可以把更多精力放在逻辑上。
 
### 循环引用问题
出现循环应用的场景：当A使用了B，B里的处理逻辑也使用A，导致二者互相持有，互相等待被释放，最后retainCount都不会减为0，内存资源就不会被释放回收。
为了解决这个问题，引入了`弱引用`的概念。`弱引用`不会让`retainCount`值+1。A强引用B，B弱引用A，当除了B以外的其他地方都不使用A是，A的retainCount值就会为0，打破了相互等待的状态，这样AB就可以释放回收了。
这个需要我们在声明对象之间的持有关系时注意的地方。也是最容易发生错误的地方，尤其是在block、NSTimer时注意。

### 野指针问题
因为弱引用的引入，如果A被释放了，B还再被其他地方使用，当B在处理逻辑时需要访问A的值时，因为A已经被释放了，这就会出现野指针。
为了解决这个问题，苹果内部引入了`weak_table_t`和`weak_entry_t`的内部实现逻辑，保存弱引用指针和引用对象之间的关系，房对象被释放时，自动将弱引用该对象的指针置为nil。
这个过程是runtime自动实现的，我们在编码过程中也无需考虑。

### ARC的规则
* 不能手动发送retain、release、autoRelease、retainCount、dealloc消息，编译器会帮我们自动插入这些代码；可以实现dealloc方法，实现时不用在掉super，会自动调用
* OC中不能使用NSZone、NSAllocateObject和NSDeallocateObject
* ARC使内存延迟释放，可以使用自动释放池代码块，来加快释放。
* ARC只适用于OC对象，对于C、coreFoundation对像和指针还需我们自己手动管理

## ARC实现原理
### 预备知识
#### tagged pointer
在runtime的源码中我们总能看到，`isTaggedPointer`的判断。简单来说`TaggedPointer`是决定内存地址结构的一个标记位。<br/>
CPU的长度决定了数据类型的长度。在64位系统下，基本数据类型使用4个字节就能存储很大的数组，如果使用64个字节存储一个数值，就浪费空间。因此苹果引入了Tagger Point的概念：将对象指针分成两部分，一部分用于存储数据值，另一部分标记这个指针的特殊性。这样指针的值不再是指向另一块内存的地址，而是真正存储的值。最终在这个宏中赋值
```objectivec
#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif
```

* taggedPointer用于存储小对象，如NSDate、NSNumber
* taggedpointer的值不再指向另一块内存空间，而存储真正的值
* 读取速度就快很多

#### isa并不总直接指向metaClass和class
在![Runtime浅析]()中已经介绍了，`isa_t`的结构

#### RefcountMap中的size_t
在![上一篇](weak原理浅析.md#weak的数据结构)中已经介绍了，存储retainCount的地方在sideTable中的`RefcountMap`。

### 自动改变引用计数值


## 总结



