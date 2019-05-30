## Block概述
block是C语言的扩充功能。block是可以截获自动变量值的匿名函数。可以通过简短代码的实现功能而不用额外定义OC类等，可以节省代码量。block是通过C语言实现的，所以执行效率会快很多。
总之block是一个可以捕获自动变量的匿名函数，能够短小而高效实现功能。
### 匿名函数
所谓的匿名相对于程序员来说的。我们程序员无需给函数命名，但实质上编译器会给函数命名的。后面查看blcok编译后代码就明白了。
### 截获自动变量
当block快中要访问block以外的自动变量时，block会自动捕获这个自动变量的值，在block块中访问。


## 语法
`block`代表两种不同的含义：1.定义匿名函数表达式可以叫做block。2.也可以把这个匿名函数当作一个变量看待处理，这个时候这个变量也可以叫做block。下面我们分别说说他们的语法。
### block表达式语法
`^` `返回值类型` (`参数列表`) {`表达式`}
```objectivec
^int (int para) {NSLog(@"hellow block!");}
```
这是一个完整的block定义，`^`插入符号不能省；`返回值类型`可以省略，当省略时如果表达式没有返回值那么block的返回值就是void，表达式有返回值且所有返回值类型一致时，block的返回值就是表达式中类型；
`参数列表`可以省略；`表达式`不能省略，表达式中有多个返回值时类型要保持一致。
```objectivec
^ { NSLog(@"hellow block!"); }
^ (int par) { return par + 1; };
```
这个是我们最常见的简单block，省略了返回值和参数。尤其是返回值类型，在block语法中经常省略。

### block型变量
`返回值类型` (^`变量名`) (`参数列表`) <br/>
声明block变量时返回值、参数列表都不能省略。
```objectivec
int (^blk) (int);
```
这个就是定义了一个返回值是int参数是int的block变量。
```objectivec
int (^blk) (int) = ^int (int par) { return par + 1; };
```
这个就是把通过block语法生成的匿名函数赋值给了blk变量；blk是变量的名字，这个变量的类型是一种返回值int参数int的block。<br/>
block型变量跟其他变量一样可以作为自动变量、函数参数、静/动态全局/局部变量等使用，跟普通变量一样使用。如：把block型变量作为函数参数使用
```objectivec
- (void)setBlock:( int(^)(int) )blc{
    int (^myBlock)(int) = blc;
}
```
下面这个是把block类型变量作为自定义类型的变量使用
```objectivec
@property (nonatomic, copy) void (^blk)(void);
```
上面我们看到当block变量作为函数参数时书写比较复杂，我们可以使用`typedef`来简化写法。

```objectivec
//全局定义一个block类型
typedef void (^finishBlock)(void);
//1作为自定义类型使用时
@property (nonatomic, copy) void (^blk)(void);
finishBlock blc = ^{ NSLog(@"hi,girl") };
//2.作为函数参数使用时
- (void)setBlock:( finishBlock )blc{
    //....
}
//3.做给函数返回值使用时
- (finishBlock )getBlock{
    //....
}
```
这样看起来就比较容易理解，且书写也简单

## 截获自动变量瞬时值
我们使用的变量有：全局变量、自动变量(局部变量)、静态全局变量、静态变量(静态局部变量)、函数参数。<br/>
* 全局变量、静态全局变量的作用域大大，可以在多个函数间传递，所以这些变量block不用捕获，可以直接读取和修改。
* 静态变量(静态局部变量)的作用域是当前函数内且数据不会随函数执行结束而释放。同一个函数多次调都用可以访问这个静态变量且内容是上一次执行时的内容；block是通过捕获静态局部变量的指针，这样block里面也可以修改静态局部变量的值并保存修改的值。
* 自动变量作用域只有当前函数内且随函数执行结束而释放资源，所以block会捕获自动变量，默认不可以修改，使用__block修饰符才可以修改。
block表达式内使用到外面的自动变量时，block会自动捕获用到的自动变量。实现机制是在block语法执行时将自动变量赋值另外一个新变量，并把这个新变量保存在block中，所以捕获的时机是block语法执行时，捕获的值是自动变量的瞬间值。之后再修改外面的自动变量也不会改变block里捕获的值。
```objectivec
    int a = 10;
    void (^blk)(void) = ^{
        NSLog(@"%d",a);
    };
    a = 20;
    blk();
```
上面代码打印的结果是10，不是20；执行block语法时a的值是10，所以捕获的到的值是10；因为block将捕获的值保存在另外一个新变量里，所以当修改a时不会更改block里捕获到的值。
```objectivec
    self.str = @"hellow";
    void (^blk)(void) = ^{
        NSLog(@"%@",self.str);
    };
    self.str = @"hi,girl";
    blk();
```
上面打印结果是hi,girl。这是因为：block语法捕获的是self，而不是self.str,在block里保存的新的变量是self的地址，相当于block持有self，只要self的地址不变，self里面的内容修改是可以同步到block里面的。
还有一种特殊情况
```objectivec
char text[] = "hellow";
    void (^blk)(void) = ^{
        NSLog(@"%c",text[0]);
    };
    blk();
```
这样会编译错误，提示block中不能使用C数组。因为block截获自动变量时不能截获C数组，我们可以使用指针实现同样的功能
```objectivec
    char *text = "hellow";
    void (^blk)(void) = ^{
        NSLog(@"%c",text[0]);
    };
    blk();
```
总结如下：
* 全局静/动态变量，block不用截获，可以直接读取和修改。
* 局部静态变量，block截获的是局部静态变量的指针，在block中可以读取和修改。
* 局部变量（自动变量），block需要截获，默认只能读取不能修改。
* block截获对象型自动变量时，在block里可以修改对象里面的内容；
* block截获数值型变量时，block中不可以修改变量值，除非使用__block修饰符。
* block不支持C数组变量的截获。


## __block修饰符


## block类型
### NSConcreteStackBlock
### NSConcreteGlobalBlock
### NSConcreteMallocBlock

## block与内存

