## GCD（Grand Central Dispatch)介绍
GCD是一种异步执行任务的技术，开发者只需把任务添加到Dispatch queue中，GCD会生成必要的线程并计划执行任务。线程交由系统内核统一管理，提高GCD的效率。
特点如下：
* 可用于多核的并行运算
* 会自动利用更多的CPU内核（比如双核、四核）
* 会自动管理线程的生命周期（创建线程、调度任务、销毁线程）
* 程序员只需要告诉GCD想要执行的任务，不需写任何线程管理代码

## 预备知识
### 线程
线程：CPU核一次只能执行一个命令，不能执行某处分开的并列的两个命令，所以cpu核执行的命令不会出现分歧，就好比一条无分叉的路径，这就是一个线程。<br/>
多线程：一个物理cpu芯片有多个核，一个cpu核虚拟为两个cpu核工作，那么一个多核cpu就出现多条不分叉的命令路径，即多线程。<br/>
上下文切换：内核发生操作系统事件(如每一定时间会唤起系统调用)时就会切换执行路径，执行路径中的状态(即线程状态，如cpu寄存器等信息)保存到各自路径专用内存块中，切换路径时从目标路径专用的内存块中复原路径状态，进而继续执行路径的命令列<br/>
多线程引发的问题：多线程编程实际上很容易发生各种问题
1. 数据竞争：多个线程同时更新同一块数据导致的数据不一致
2. 死锁：线程中有停止等待事件，会出现多个线程互相停止等待
3. 太多线程会消耗大量内存资源
多线程的优点：可以保证程序的响应性能，防止卡顿(线程阻塞)，提升用户体验，所以还是要使用多线程编程。但这一个优点就使多线程利大于弊，所以多线程被广泛应用。

### 串行和并发--任务的分发方式
串行：在一个线程中任务按顺序执行，一个任务完之后再执行下一个任务，好比生活中的“一个窗口排一个队”<br/>
并行：多个任务同时开始，一个线程一会处理这个任务，一会处理那个任务，不是真正的同时进行，好比生活中的“一个窗口排多个队”<br/>
并发：多个任务同时被多个线程处理，好比生活中“多个窗口排多条队”<br/>

### 同步和异步--任务的执行方式
同步任务sync：不能开启新线程
异步任务async：有创建多个线程的能力

## GCD接口
### GCD主队列
```objectivec
//主队列，主线程中的唯一队列，一个串行队列
dispatch_queue_t mainQueue = dispatch_get_main_queue();
```
主队类，程序运行起来就会创建一个主线程，线程里面对应有唯一一个主队列，这个队列是串行队列。

### GCD主队列
```objectivec
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
//队列的优先级
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN

//或者新的方式
__QOS_ENUM(qos_class, unsigned int,
	QOS_CLASS_USER_INTERACTIVE,//表示任务需要被立即执行提供好的体验，用来更新UI，响应事件等。这个等级最好保持小规模。
	QOS_CLASS_USER_INITIATED,//表示任务由UI发起异步执行。适用场景是需要及时结果同时又可以继续交互的时候。
	QOS_CLASS_DEFAULT,
	QOS_CLASS_UTILITY,//表示需要长时间运行的任务，伴有用户可见进度指示器。经常会用来做计算，I/O，网络，持续的数据填充等任务。这个任务节能。
	QOS_CLASS_BACKGROUND,//表示用户不会察觉的任务，使用它来处理预加载，或者不需要用户交互和对时间不敏感的任务
	QOS_CLASS_UNSPECIFIED,
);
```
全局队列，`dispatch_get_global_queue`的第一个参数代表队列的优先级，第二个参数是保留参数，目前先传0。iOS8以后第一个参数还可以使用QOS_ENMUM的枚举类型

### GCD中自己创建队列
```objectivec
dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
```
第一个参数是队列名称，建议使用逆序全程域名，eg“com.knowledge.TestGCD.queueName”,方便后续debug；第二个参数指定queue的类型，可用的参数有`DISPATCH_QUEUE_CONCURRENT`和`DISPATCH_QUEUE_SERIAL`。<br/>
`DISPATCH_QUEUE_SERIAL`表示创建一个串行队列，一个SerialQueue就会创建一个线程，如果创建过多的SerialQueue，就会创建过多的线程，过多的线程会占用大量内存，也会引起大量的上下文切换，这样就会大大降低系统响应性能。因此在使用SerialQueue时要注意避免创建过多。<br/>
`DISPATCH_QUEUE_CONCURRENT`表示创建一个并发队列，并发处理的数量取决于系统的状态(DispatchQueue中处理数、cpu核数、cpu负荷等)，并发任务的完成顺序取决于任务的大小和系统状态不同而不同，在哪个线程也不确定

### 同步和异步任务
```objectivec
//添加同步任务
dispatch_sync(queue, ^{
        //do something
    });
    
//添加异步任务
dispatch_async(queue, ^{
        //do something
    });
```
`dispatch_sync`添加一个同步任务,`dispatch_async`添加一个同步任务。第一个参数是任务执行的队列，第二个参数是任务，以`dispatch_block_t`的类型

### 任务和队列
一般我们不会创建并发队列，而是直接使用GCD提供的全局并发队列，那么就会有三种形式的队列，创建的串行队列，全局队列，主队列；任务有两种，同步和异步任务，组合起来就有6中形式

| 任务 | 创建的串行队列 | 全局队列 | 主队列 |
| ------ | ------ | ------ | ------ |
| 同步 | 任务在同一个线程按顺序执行 | 任务在同一个线程按顺序执行 | 任务在同一个线程按顺序执行 |
| 异步 | 任务在多个线程按顺序开始执行(不是真正的并发) | 任务在不同线程同时执行(真正的并发) | 任务在同一个线程按顺序执行 |
总结不同队列用于执行的任务类型：
* 主队列（顺序）：其他队列中有任务完成需要更新UI。
* 并发队列：用来执行与UI无关的后台任务，dispatch barrier同步、dispatch groups、读取网络数据、大量数据的数据库读写等任务
* 自定义串行队列：顺序执行后台任务并追踪它时。这样做同时只有一个任务在执行可以防止资源竞争。dipatch barriers解决读写锁问题的放在这里处理。

### dispatch_after
```objectivec
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3*NSEC_PER_SEC);
    dispatch_after(time, dispatch_get_main_queue(), ^{
        NSLog(@"after 3 seconds");
    });
```
`dispatch_after`表示time时间后加入任务，而不是time时间后执行任务。
`dispatch_time`表示的相对时间，第一个参数就是参考时间点，第二个参数是时间，计算的结果是纳秒，x*NSEC_PER_SEC就是X秒，x*NSEC_PER_MSEC就是x毫秒。
这个时间不是十分准确

### group
dispatch_group用于监视多个异步任务的执行结果，dispatch_group_t用来追踪不同队列中的任务的容器，首先把要监视的任务添加到容器，当容器任务都执行完毕后执行同一处理。
往group容器添加任务方式有两种：1.`dispatch_group_async` 2.`dispatch_group_enter`与`dispatch_group_leave`成对使用。统一处理任务结果的方式有两种：1.`dispatch_group_notify`异步等待 2.`dispatch_group_wait`同步等待会阻塞线程
```objectivec
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
    dispatch_group_t group = dispatch_group_create();
    //添加任务
    dispatch_group_async(group, globalQueue, ^{
        [NSThread sleepForTimeInterval:2.f];
        NSLog(@"1");
    });
    dispatch_group_async(group, globalQueue, ^{
        NSLog(@"2");
    });
    //监听任务都完成时:同步等待
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"go on");
```
另外一种写法
```objectivec
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
    dispatch_group_t group = dispatch_group_create();
    //添加任务
    dispatch_group_enter(group);
    dispatch_async(globalQueue, ^{
        NSLog(@"3");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(globalQueue, ^{
        NSLog(@"4");
        dispatch_group_leave(group);
    });
    
    //监听任务都完成时:异步等待
    dispatch_group_notify(group, globalQueue, ^{
        NSLog(@"go on");
    });
```

### dispatch_barrier
`dispatch_barrier_async`确保指定的任务在指定的并发队列中执行此任务时唯一执行的任务。先于此任务加入的任务都执行结束之后才开始执行此任务，
在此任务执行期间保证只执行这一个任务，当此任务结束之后，再并发执行此任务后面的其他任务。
需要注意`dispatch_barrier_async`只适用于并发队列，串行队列没必要使用这个；只在自己创建并发队列上有这种作用，在全局并发队列上没有效果，在串行队列上和dispatch_sync一样。
```objectivec
    dispatch_queue_t globalQueue = dispatch_queue_create("com.knowledge.viewcontroller", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(globalQueue, ^{
        NSLog(@"1");
    });
    dispatch_async(globalQueue, ^{
        NSLog(@"2");
    });
    dispatch_async(globalQueue, ^{
        NSLog(@"3");
    });
    dispatch_barrier_async(globalQueue, ^{
        NSLog(@"barrier");
    });
    dispatch_async(globalQueue, ^{
        NSLog(@"4");
    });
    dispatch_async(globalQueue, ^{
        NSLog(@"5");
    });
```
尤其是针对多个地方同时读写数据库时会用到。解决多线程并发读写同一个资源发生死锁。

### dispatch_apply
重复添加同一个任务到队列中，可以指定添加的次数，这个任务可以用有入参。类似for循环，不同点是dispatch_apply添加到并发队列时任务时并发执行。
需要注意的是`dispatch_apply`会阻塞当前线程
```objectivec
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
    dispatch_apply(10, globalQueue, ^(size_t i) {
        [NSThread sleepForTimeInterval:1];
        NSLog(@"%zu",i);
    });
    NSLog(@"end");
```
0～9打印完之后在打印end，0~9并不是顺序打印，并发执行的任务。
```objectivec
    //可能会死锁，因为线程数量太多，占用大量内存而崩溃
    for (int i = 0; i < 999 ; i++) {
        dispatch_async(concurrentQueue, ^{
            NSLog(@"wrong %d",i);
            //do something
        });
   }
   
   //使用applay避免产生多个线程
   dispatch_apply(1000, concurrentQueue, ^(size_t i) {
        NSLog(@"%zu",i);
        //do something
    });
```
可以解决线程暴涨的问题。尤其是在for循环次数上千千次时。

### dispatch_once
在多线程下保证代码只执行一次，百分百保证。多用于单例模式。
### dispatch_suspend和dispatch_resume
`dispatch_suspend` 挂起任务队列，已经开始执行的任务不起作用，尚未开始的任务不在执行。<br/>
`dispatch_resume` 恢复队列状态，继续任务中为开始执行的任务

### dispatch_block_t
GCD中任务都是以block的方式加入到队列中。`dispatch_block_t`就是用来描述任务的。这个是在iOS8之后添加的新接口。
```objectivec
    //创建任务
    dispatch_block_t block = dispatch_block_create(0, ^{
        NSLog(@"new block");
    });
    //添加异步block任务到主队列
    dispatch_async(dispatch_get_main_queue(), block);
```
`dispatch_block_notify`用于监听某个任务执行结果，这个不同于`dispatch_group_notify`,blockNotify只用于监听某一个任务执行的结果，groupNotify可用于监听多个任务执行的结果。
```objectivec
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        NSLog(@"new block");
    });
    dispatch_async(dispatch_get_main_queue(), block1);
    
    dispatch_block_notify(block1, dispatch_get_main_queue(), ^{
        NSLog(@"block1 finished");
    });
```
`dispatch_block_notify`第一个参数要监听的任务，第二个参数监听任务执行的队列，第三个参数监听任务。<br/>

`dispatch_block_cancel`用于取消某个任务
```objectivec
dispatch_block_cancel(block1);
```

## 死锁
前面我们说到多线程编程会出现的问题，数据竞争、死锁、创建太多线程。对于数据竞争问题这个要结合具体业务分析；创建太多线程的问题比较简单；这里我说一说比较复杂的死锁问题。<br/>
下面看第一种情况：串行队列里添加一个同步任务且添加任务执行的队列就是当前这个串行队列。
```objectivec
    dispatch_queue_t serialQueue = dispatch_queue_create("com.knowledge.viewcontroller.serialqueue", DISPATCH_QUEUE_SERIAL);
    NSLog(@"1");
    dispatch_async(serialQueue, ^{
        NSLog(@"2");
        //串行队列里面同步一个串行队列就会死锁
        dispatch_sync(serialQueue, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
```
输出结果是152，34因为发生死锁而crash。<br/>
分析：打印完2时正处在串行队列中，添加的任务也要同样的队列，当前队列正在执行任务且还没有结束(可以把打印2，添加任务，打印4看作是一个大任务，serialQueue正在执行这个大任务)，在执行这个任务时又要添加一个其他同步任务，这样就会造成互相等待。
解决方法：任务3改成并发队列或其他串行队列;或者把任务3改成异步的任务。<br/>
下面看另外一种情况：主队列发生任务一直不结束，还要往主队列添加同步任务
```objectivec
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"1");
        //回到主线程发现死循环后面就没法执行了
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"2");
        });
        NSLog(@"3");
    });
    NSLog(@"4");
    //死循环
    while (1) {
        //
    }
```
总结出现死锁的情况：
* 主队列或其他串行队列容易出现死锁
* 添加同步任务dispatch_sync时要注意
出现阻塞线程的地方：
* dispatch_sync、dispatch_apply、dispatch_group_wait、dispatch_barrier这些CGD的接口


## 内存管理
在ARC环境，我们无需管理，GCD会像对待OC对象一样自动内存管理。
