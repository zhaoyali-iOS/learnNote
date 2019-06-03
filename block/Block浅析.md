## Block概述
Block是C语言的扩充功能。Block是可以截获自动变量的匿名函数。可以通过简短代码来实现功能而不用额外定义OC类，既可以节省代码量，又可以使代码不分散。Block是通过C语言实现的，所以执行效率会快很多。
总之Block是一个可以捕获自动变量的匿名函数，能够短小而高效地实现功能。
### 匿名函数
所谓的匿名相对于程序员来说的。我们程序员无需给Block语法命名，就能实现功能。实质上编译器会给Block函数命名，[后面一篇文章](Block的实现.md)会详细分析Blcok实现时就明白了。
### 截获自动变量
当Block语法中要访问Block以外的自动变量时，Block会自动捕获这个自动变量的值，在Block块中访问。


## 语法
`Block`有两种不同的含义：1.定义匿名函数表达式的语法块叫做Block语法，简称为Block。2.把这个匿名函数当作一个变量看待处理，这个时候这个变量也可以叫做block型变量，简称block。下面我们分别说说他们。
### Block语法
`^` `返回值类型` (`参数列表`) {`表达式`}
```objectivec
^int (int para) {NSLog(@"hellow block!");}
```
这是一个完整的Block定义，`^`插入符号不能省；`返回值类型`可以省略，当省略时如果表达式没有返回值那么Block的返回值就是void，表达式中有返回值且所有返回值类型一致时，Block的返回值就是表达式中返回类型；`参数列表`可以省略；`表达式`不能省略，表达式中有多个返回值时类型要保持一致。
```objectivec
^ { NSLog(@"hellow block!"); }
^ (int par) { return par + 1; };
```
这个是我们最常见的简单block，第一个省略了返回值和参数。第二个省略了返回值类型，在Block语法中经常会省略返回值类型。

### block型变量
`返回值类型` (^`变量名`) (`参数列表`) <br/>
声明block变量时返回值、参数列表、拆入符号都不能省略。
```objectivec
int (^blk) (int);
```
这个就是定义了一个返回值是int参数是int的block变量。
```objectivec
int (^blk) (int) = ^int (int par) { return par + 1; };
```
这个就是把通过Block语法生成的匿名函数赋值给了block型变量，这个变量的名字是blk；blk变量的block类型具体是一种返回值int参数int的block。<br/>
block型变量跟其他变量一样可以作为自动变量、函数参数、静/动态全局/局部变量等使用，跟普通变量一样使用。Block真正是一个结构体对象，可以跟普通对象一样使用。
如：把block型变量作为函数参数使用
```objectivec
- (void)setBlock:( int(^)(int) )blc{
    int (^myBlock)(int) = blc;
}
```
下面这个是把block类型变量作为普通变量使用
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
这样看起来就比较容易理解，且书写也简单。我们还可以对相同签名的block定义不同的名字，这个尤其是在AF的返回block中看到应用。这样做也是为了更容易理解代码，和后续代码修改方便。

## 截获自动变量的瞬时值
我们使用的变量有：全局变量、自动变量(局部变量)、静态全局变量、静态变量(静态局部变量)、函数参数。<br/>
* 全局变量、静态全局变量的作用域大大，可以在多个函数间传递，Block语法不用捕获这类变量，可以直接读取和修改。
* 静态变量(静态局部变量)的作用域是当前函数内且数据不会随函数执行结束而释放；同一个函数多次调都用可以访问这个静态变量且内容是上一次执行时的内容。Block语法是捕获静态局部变量的指针，这样Block块就可以修改静态局部变量的值并保存修改的值。
* 自动变量作用域只有当前函数内且随函数结束而释放资源，所以Block会捕获自动变量，默认不可以修改，使用__block修饰符才可以修改。
捕获变量的机制：在Block语法执行时会创建一个新的变量并把用到的外部变量赋值给这个新变量，这个新变量保存在block中，所以捕获的时机是Block语法执行时，捕获的值是执行语法块时刻的自动变量的瞬间值。之后再修改外面的自动变量也不会改Bblock里捕获的值（除非使用__block修饰）。
```objectivec
    int a = 10;
    void (^blk)(void) = ^{
        NSLog(@"%d",a);
    };
    a = 20;
    blk();
```
上面代码打印的结果是10，不是20；执行Block语法时a的值是10，所以捕获的到的值是10；因为Block将捕获的值保存在另外一个新变量里，所以当修改a时不会更改Block内部创建的新变量。
```objectivec
    self.str = @"hellow";
    void (^blk)(void) = ^{
        NSLog(@"%@",self.str);
    };
    self.str = @"hi,girl";
    blk();
```
上面打印结果是hi,girl。这是因为：block语法捕获的是self，而不是self.str,在block里新的变量的值是self的地址，相当于Block持有self，只要self的地址不变，self里面的内容修改是可以同步到Block里面的，同时Block里面修改的内容可以同步到Block外面。
```objectivec
    self.testProperty = @"test property";    
    void (^blk)(void) = ^{
        //等效于self.testProperty = @"a new value of test property";
        _testProperty = @"a new value of test property";
    };
    blk();
    NSLog(@"%@",self.testProperty);
```
这里打印的结果是“a new value of test property”，block捕获的变量是self而不是_testProperty。所以block也可以修改实例变量的值。
还有一种特殊情况
```objectivec
char text[] = "hellow";
    void (^blk)(void) = ^{
        NSLog(@"%c",text[0]);
    };
    blk();
```
这样会编译错误，提示Block中不能使用C数组。因为block截获自动变量时不能截获C数组，我们可以使用指针实现同样的功能
```objectivec
    char *text = "hellow";
    void (^blk)(void) = ^{
        NSLog(@"%c",text[0]);
    };
    blk();
```
总结如下：
* 全局静/动态变量，Block不用截获，可以直接读取和修改。
* 局部静态变量，Block截获的是局部静态变量的指针，在Block中可以读取和修改。
* 局部变量（自动变量），Block需要截获，默认只能读取不能修改。
* Block截获对象型自动变量时，在Block里可以修改对象里面的内容；
* Block使用对象的实例变量时捕获的时对象而不是对象的实例变量。
* Block截获数值型变量时，Block中不可以修改变量值，除非使用__block修饰符。
* Block不支持C数组变量的截获。


## __block修饰符
Block外的变量使用`__block`修饰符之后，在Block内外都可以修改变量并互相同步。可以理解成Block内外的变量引用的是同一个变量，不使用`__block`修饰符时Block内外其实是两个变量，只不过两个变量的值相同。

## Block类型
既然Block也是一个结构体对象，那么也就会涉及到内存管理。尤其是在block捕获到对象型变量时，就更加需要内存管理了。Block一共有三种类型
`__NSStackBlock__`、`__NSMallocBlock__`、`__NSGlobalBlock__`，根据名字知道他们内存分别在栈区、堆区、全局区。
```objectivec
    NSNumber *a = [NSNumber numberWithInt:10];
    //__NSStackBlock__
    NSLog(@"%@",^{
        NSLog(@"hellow:%@",a);
    });
    //__NSMallocBlock__
    void (^blk)(void) = ^{
        NSLog(@"hellow");
    };
    NSLog(@"%@",blk);
```
这段代码是打印不同类型Block：第一个是__NSStackBlock__类型，第二个是__NSMallocBlock__类型，这是因为Block的内存分配在栈区，进行`=`操作时把栈上的对象copy到了堆上。也就是说Block类型的`=`操作都是copy。
 ```objectivec
    //__NSGlobalBlock__
    NSLog(@"%@",^{
        NSLog(@"hellow");
    });
    //__NSGlobalBlock__
    void (^blk2)(void) = ^{
        NSLog(@"hellow");
    };
    NSLog(@"%@",blk2);
    //__NSGlobalBlock__,dyAall是一个全局变量
    NSLog(@"%@",^{
        NSLog(@"hellow，%d",dyAall);
    });
 ```
 这里打印出来的都是__NSGlobalBlock__类型。这里先说全局类型的Block是指在全局语义内定义或者表达式没有捕获任何自动变量的块。全局Block的等号copy是一个空操作。这也算是对Block的一种优化，避免多余copy。也是为什么会有__NSGlobalBlock__类型的原因。
```objectivec
- ( void(^)(void) )getBlok {
    int a = 10;
    return ^{
        NSLog(@"girl:%d",a);
    };
}

NSLog(@"%@",[self getBlok]);
```
当Block作为函数返回值时是__NSMallocBlock__类型，这里是因为，返回值在函数结束时其作用域也就结束，理论上应该释放的，这样的话这个方法就会永远返回nil，跟ARC一样要延长Block的声明周期，所以XCode也把Block先copy到堆上，然后放到自动释放池中，相当于这样：
```objectivec
- ( void(^)(void) )getBlok {
    int a = 10;
    void(^blk)(void) = ^{
        NSLog(@"girl:%d",a);
    };
    blk = [ [blk copy] autorelease];
    return blk;
}
```
block作为函数参数时不会copyBlock，block的类型不变，引用计数不会+1或-1
```objectivec
- (void)testBlock:( void(^)(void) ) blk {
    NSLog(@"%@",blk);
}

    //__NSStackBlock__，因为函数参数不会自动copy
    [self testBlock:^{
        NSLog(@"testBlock:%@",a);
    }];
```
小结：
* `__NSStackBlock__`:Block最开始分类的类型，存储在栈上，超出作用域就会释放。
* `__NSMallocBlock__`：分配在堆上，一般是stack类型copy过去的。
* `__NSGlobalBlock__`：表达式中没有捕获任何自动变量，或者只有全局变量、静态变量，或者没有然后变量的块。他的copy是空操作。
* `=`：把stack类型Blockcopy到堆上，把Malloc类型Block的引用计数+1；
* Block作为函数返回值时会自动copy到堆上，并自动添加到autoreleasePool中；Block作为函数参数类型不变，引用计数也不变。
* Block中捕获对象型自动变量是会使捕获的变量引用计数+1，当Block释放时也会使捕获的变量引用计数-1。





