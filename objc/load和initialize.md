
## +load方法

### +load的执行时机
运行时系统初始化之后，每次加载镜像文件时，镜像文件中的类已经加载到内存中时，就会检查是否有load符号，如果有就会调用load。这些都是在main函数之前执行。<br/>
因为镜像文件分多次加载的，如果在A类的load的实现中用到B类，那么就要先执行B类的load，同样执行B类的load方法要先确定B类的继承父类，可能B类在其他框架里还没加载，这样就会出现问题。所以尽量避免在laod方法中依赖其他的类，只做与本类相关的事情。<br/>
程序在加载我们自定义的类之前，必定已经加载了系统的framework，所以实现load方法时使用系统类没有关系，但是自定义的其他类，第三方库的类最好不要依赖。

### +load的执行顺序
* 先父后子--父类执行完后再执行子类
* 先主后分--主类执行完后再执行分类
* 不同类之间调用顺序不确定
* 实现load方法时不用显示的调用父类方法

### +load的特点
* 运行期间只调用一次
* 主类和分类的+load方法都执行，这不同于其他方法--分类会覆盖主类方法
* 日常开发中接触到的调用最靠前的方法
* 不会沿用父类的实现 `load方法的执行没有通过消息转发机制，而是直接取IMP地址执行` 

### 准备+load--确保先父类后子类
load方法执行分为两部分：准备load和执行load，这里先说第一部分<br/>
前面已经说到触发load是在镜像文件加载且所有类已经加载到内存时，会触发load_images方法
```objectivec
void load_images (const char *path __unused, const struct mach_header *mh) {
     //如果没有发现load符号直接结束
     if ( !hasLoadMethods( mh ) ) return;

     //加锁
     recursive_mutex_locker_t lock(loadMethodLock);
     {
         mutex_locker_t lock2(runtimeLock);
          //1准备load方法
          prepare_load_methods(mh);
     }

     //2调用load
     call_load_methods();
}
```
从这段代码可以看出`load`方法分为prepare和call两部分

```objectivec  
void prepare_load_methods (const headerType *mhdr) {
     size_t count, i;
     //获取实现load方法的主类
     classref_t *classlist = _getObjc2NonlazyClassList( mdr, &count);
     for (i=0; i < count; i++) {
          schedule_class_load( remapClass(classless[i]) );
     }

     //获取实现load方法的分类
     category_t **categorylist = _getObjc2NonlazyCategoryList( mhdr, &count );
     for (i=0; i < count; i++) {
          category_t *cat = category list[i];
          Class cls = remapClass(cat->cls);
          if (!cls) continue;
          realizeClass(cls);
          assert( cls->ISA()->isRealized() );//断言，保证主类已经加载了
          add_category_to_loadable_list(cat);
     }
}

static void schedule_class_load(Class cls) {
     if (!cls) return;
     assert( cls->isRealized() );
     if (cls->data()->flags & RW_LOADED) return;

     //递归调用，确保父类放在前面
     schedule_class_load( cls->superclass );

     //添加当前类
     add_class_to_loadable_list( cls );
     cls_setInfo( RW_LOADED );
}

```
准备阶段把实现load方法的主类保存在全局数组`loadable_classes`中，实现load的分类保存在全局数组`loadable_categories`中<br/>
源码中全局搜索`_getObjc2NonlazyClassList`，看到了对次方法的解释 `// Realize non-lazy classes (for +load methods and static instances)`，这个方法是从所有的类中挑出实现load方法的类,同理可以预测到 `_getObjc2NonlazyCategoryList`是从所有类中挑出实现load方法的分类<br/>
`schedule_class_load( cls->superclass )` 递归调用确保实现父类提前加入`loadable_classes`中，父类在前，子类在数组后<br/>
`add_category_to_loadable_list` 的实现比较简单这里就不在贴代码了

### 执行+load--确保先执行主类在执行分类

```objectivec
void call_load_methods(void){
     ...
     //添加自动释放池
     void *pool = objc_autoreleasePoolPush();
     do {
           //1调用主类的load方法

          while (loadable_calsses_used > 0) {
               call_calss_loads();
          }
          //2调用分类的load方法
          more_categories = call_category_loads();
     } while (loadable_calsses_used > 0 || more_categories);
     //倾倒自动释放池
     objc_autoreleasePoolPop(pool);
     ...
}

static void call_class_loads(void){
     //拷贝所有实现load方法类的列表
     struct loadable_calss *classes = loadable_calsses;
     int used = loadable_classes_used;
     //清空load列表
     loadable_classes = nil;
     loadable_classes_allocated = 0;//列表分配空间大小

     loadable_classes_used = 0;//列表已使用空间大小

     //调用所有主类的load方法
     for (int i=0; i < used; i++) {
          Class cls = classes[i].cls;
          load_method_t load_method = (load_method_t)classes.method;
          if (!cls) continue;
          //执行对应cls的load方法
          (*load_method)(cls, SEL_load);
     }

     //释放空间
     if (classes) free(classes);
}
```
`loadable_calsses_used` 标记 `loadable_classes`已使用空间大小<br/>
`(*load_method)(cls, SEL_load);` 执行load方式是直接找到方法地址直接执行，没有使用消息转发机制<br/>
先调用主类的+load，在调用分类的+load，分类没有覆盖主类的方法，并且都执行了，这点不同于其他分类覆盖<br/>
执行load方法前runtime添加了自动释放池，所以我们在实现load方法是不必再添加自动释放池了；

## +initialize方法

### +initialize的执行时机
* 简单来说就是当第一次给类发送消息时会触发+initialize方法。
* 详细说来，当libobjc等动态库加载后，runtime初始化(`_objc_init`调用)完成，`map_images`和`load_images`方法执行之后,第一次向类发送消息时执行。+load方法的执行是直接取函数地址执行，没有通过消息转发机制，所以+load方法不会触发+initialize方法，load要先于initialize执行。
* 在main函数之前会有其他的类已经初始化了，所以initialize方法在main函数前和后都会执行，一般我们自己定义类的initialize方法是在main函数之后执行。
* initialize方法有可能永远不会执行到。

### +initialize的内部实现
在+initialize方法处打断点看到，通过`_class_initialize`调用了initialize方法
```objectivec
void _class_initialize (Class cls) {
     assert( !cls->isMetaClass() );

     //递归调用，确保super的initialize先执行
     Class supercls = cls->superclass;
     if (supercls && !supercls->isInitialized() ) {
          _class_initialize(supercls);
     }

     ….线程加锁等操作
     callInitialize(cls);//调用initialize方法
     lockAndFinishInitializing (cls, supercls);
     ...
}
```
这里通过递归调用，保证父类先调用+initialize方法

```objectivec
void callInitialize (Class cls) {
     ( ( void(*)(Class, SEL) ) )objc_msgSend )( cls, SEL_initialize);
}
```
这里使用了`objc_msgSend`消息传递机制，这样就会造成父类+initialize方法执行多次

```objectivec
static void lockAndFinishInitializing (Class cls, Class supercls) {
     monitor_locker_t lock(classInitLock);//加锁
     if (!supercls || supercls->isInitialized() ) {
          _finishInitialzing( class, supercls);
     } else {
          _finishInitiallizingAfter(cls, supercls);//在pendingInitializeMap中添加依赖关系
     }
}

static void _finishInitializing (Class cls, Class supercls) {
      ...
     cls.setInitialized();//设置标记位为YES
     ...
     //移除PendingInitializeMap中相关项
     if ( !pendingInitializeMap ) return;
     //找到以cls为父类需要执行initialize方法的子类链表
     PendingInitialize *pending = (PendingInitialize *)NXMapGet( pendingInitializeMap, cls );
     if ( !pending )  return;
     NXMapRemove (pendingInitializeMap, cls);
     …pendingInitializeMap的重置清空操作
     //依次调用cls子类的_finishInitializing方法

     while (pending) {
          PendingInitialize *next = pending->next;
          if ( pending->subclass )  _finishInitializing (pending->subclass, cls);
          free ( pending );
          pending = next;
     }
}
```
通过 `setInitialized()` 标记当前类是否执行了+initialize方法<br/>
当初始化B时发现另一个线程正在初始化B的父类A,为了保证在A完成初始化后再标记B，防止等待阻塞线程，需要有数据结构来存储这种等待依赖关系，`pendingInitializeMap`就是存储这种依赖等待关系的数据结构<br/>
`pendingInitializeMap`是一个全局变量，用于存储等待标记initialized的类，以保证线程安全，防止阻塞线程<br/>
`pendingInitializeMap`以superClass为key，以继承自superClass的第一级子类链表为value<br/>
这里使用链表不是数组：数组需要连续的空间，链表不需要，这样更加节省内存空间

### +initialize的特点
* 内部实现已经保证了先调用super的initialize方法，再调用当前类的initialize方法，所以实现此方法时不需要显示的调用父类
* 在调用当前类的initialize方法时使用了消息转发机制，会导致父类的方法被调用多次，分类会覆盖主类的方法
* +initilaze是线程安全的

### +initialize的实现注意事项
* 这个方法同load一样不建议我们自己显示调用，runtime会自动调用。
* 为了防止我们自己调用执行多次发生错误，我们一般会在实现时使用`dispatch_onece`。同时这样也可以防止runtime系统多次调用而发生问题。
* 实现中只配置数据，不要依赖其他类，更不要调用本类的其他方法。防止其他类的数据没配置好，防止本类的其他方法的实现依赖的数据没有配置好。
* 无法在编译期设定的全局变量，可以方法initialize方法里初始化。

## +load和+initialize方法比较
项目|+load|+initialize
---|---|---
调用时机|被加载到runtime系统中|第一次收到消息
调用顺序|先父类后子类-先主类后分类|先父类后子类
调用次数|一次|多次
显示调用父类|否|否
沿用父类实现|否|是
分类实现|不覆盖，主类和分类都执行|分类会覆盖主类


* 调用顺序中，load不能保证父类的分类和子类的分类的顺序，这个取决于编译的顺序。<br/>
* 这里的调用次数是指runtime系统的调用次数，但是程序员仍然可以显示的调用这两个方法，但一般不建议这样做。所以为了安全我们一般在实现时也会使用dispatch_once。
