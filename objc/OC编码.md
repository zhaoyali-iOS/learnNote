## 使用类型常量代替宏定义
使用预处理宏定义的常量，编译器只是简单的查找和替换，即使有其他地方定义了一模一样名字的宏也不会报错。这就导致常量不一致。而编译器能保证常量的唯一性(只有一个地方定义，可以被修改)；
在实现文件中定义的静态常量，总用范围只在文件内，不会出现在全局符号表中，所以不用担心重名问题，一般以k开头命名。
在头文件中使用extern声明全局常量时，防止重名，命名时要加前缀，


## 属性和存取方法
`@property` 告诉编译器，自动生成属性的实例变量(以_+属性名)、getter、setter方法
```objectivec
@property (nonatomic, strong) NSObject * propertyName;
```
分类声明属性不会报错，也可以编译通过，但是运行调用属性的getter和setter方法就会报错，因为分类不会自动生成getter和setter方法。
分类一般只用于扩展功能，而非封装数据，所以在分类中尽量不要定义属性。

`@synthesize` 告诉编译器，为属性生成指定的实例变量名，即属性名=实例变量名
```objectivec
@synthesize propertyName=_pName;
```
.h文件声明的属性propertyName，编译器为属性自动生成的实例变量是_pName。<br/>

`@dynamic` 告诉编译器不要创建属性的实例变量、不生成getter和setter方法，这种请情况一般是自己实现setter和getter方法。<br/>

strong方式的setter方法实现时，要先retain新值再release旧值，最后修改属性指针的指向；防止新值和旧值指向同一个内存区，先release可能会导致新值被释放变成一个悬挂指针。<br/>

使用存取方法和实例变量的时机：
* 类外部都使用属性（点语法）
* 类内部一般读取时使用实例变量，设置时使用属性
* 在init和dealloc方法中使用实例变量，防止子类重写存取方法而出错
* 类内部实现了懒加载方式，就是用属性。


## 全能初始化方法
在类中要提供一个全能初始化方法，其他初始化方法都应该调用此全能初始化方法。全能初始化方法中要调用其父类的初始化方法。<br/>
父类的全能初始化方法不适用子类时，子类要定义新的全能初始化方法，并且要重写父类的全能初始化方法


## 代理委托
若有必要，可实现含有位段的结构体，将委托对象是否能响应某协议方法信息缓存这个结构体中，减少responseToSelector的判断。
```objectivec
//首先定义协议方法的是否实现的枚举
typedef NS_ENUM(NSInteger, DelegateFlags) {
    DelegateFlagsUnknow = 0,
    DelegateFlagsDidReviceData = 1<<0,
    DelegateFlagsDidFail = 1<<1,
    DelegateFlagsDidSuccess = 1<<2
};

//重写delegate的setter方法，给枚举赋值
- (void)setDelegate:(id<UITabBarDelegate>)delegate {
    _delegate = delegate;
    if ([delegate respondsToSelector:@selector(xxx)]) {
        self.delegateFlags = self.delegateFlags |DelegateFlagsDidReviceData;
    }
    
}

//在需要回调代理方法的地方使用枚举判断
  if ( self.delegateFlags&DelegateFlagsDidReviceData ) {
        [self.delegate performSelector:@selector(xxx)];
    }
```
并不是所有时候都有必要这么做，只有在相关方法要调用很多次时才值得优化。是否要优化还要根据具体代码做分析，找出瓶颈。


## 异常处理中的内存泄漏
* 在OC中只有发生严重错误时才会抛出异常然后终止程序，这个时候即便有内存泄漏因为程序马上就要终止了，所以没必要管理内存。
* 在错误不严重的时候、令方法返回nil/0、使用NSError来处理错误，从而避免因错误而发生的内存泄漏
* 所以默认情况下OC不开启‘异常安全码’
* 在C++模式下就需要处理异常来避免内存泄漏。


## NSTimer
* timer会强引用target
* 一次性timer执行完之后会释放target，一般不会引起循环引用
* 重复的timer就会造成循环引用，执行invalidate会释放terget，打破循环引用，但invalidate不一定会执行
* 解决方法：timer分类用block实现，block中如果使用target要使用weak引用

## performSelector
performSelector方法在执行时因为无法识别将要执行函数时哪一个，就无法插入内存管理的代码，所以容易出现没存问题。<br/>
performSelector方法所能处理的方法的局限性大，参数个数类型，返回值都要局限性，不够灵活。<br/>
所以最好使用GCD。




