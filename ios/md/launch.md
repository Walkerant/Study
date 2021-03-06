项目代码越来越多，APP启动速度也因此越来越慢，针对如何优化APP启动速度做一些总结。

# APP启动时做了哪些事情？
一般而言，APP的启动过程是指从用户点击APP图标，到用户看见第一个界面之间发生的一切，总结来说，可分为三个阶段：

1. `main()`函数执行前；
2. `main()`函数执行后；
3. 首屏渲染完成后。

## `main()`函数执行前

从程序的源代码来看，所有程序的入口处都是`main()`函数。但是，当程序运行到这里时，已经做了很多事情了，比如加载Mach-o可执行文件，在`runtime`运行时注册相关类、category等。具体而言，需要做的事有：

```Objc
int main(int argc, char * argv[]) { 
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

* **加载可执行文件（Mach-o文件，Apple操作系统的可执行文件都是此格式）**。 `dyld`会首先读取`mach-o`文件的Header和load commands，接着就知道了这个可执行文件依赖的动态库，例如加载动态库A到内存，接着检查A所依赖的动态库，就这样的递归加载，直到所有的动态库加载完毕。通常一个App所依赖的动态库在100-400个左右，其中大多数都是系统的动态库，它们会被缓存到dyld shared cache，这样会提高读取效率。
* **加载动态链接库，进行Rebase指针调整和Bind符号绑定以解决指针引用问题**。 App启动时，程序被映射到逻辑地址空间，这个地址一般会通过ASLR（Address space layout randomization，地址空间布局随机化）技术随机化了，这样就可以避免黑客用开始地址+`offset`得到函数地址。而Rebase指的就是Mach-o文件内部的指针指向。另外，外部的符号（其他库中的函数，例如`printf`）则是用Bind解决。
* **Objc runtime运行时初始化处理，包括Objc相关类的注册，Category的注册，Selector唯一性检查等**。 Objective-C是动态语言，在执行`main()`函数之前，需要把类的相关信息注册到一个全局表中。同时，由于OC支持Category，所以也会把Category中的方法注册到类中，还要保证Selector的唯一性等工作。
* **初始化**，包括执行`+load()`方法，C/C++静态初始化对象和标记为__attribute__(constructor)的方法。

**针对这一阶段，其可优化的地方有：**

* **减少动态库的加载**。每个库本身又有依赖库，Apple建议使用更少的动态库，并且建议在使用动态库较多时，尽量将多个动态库进行合并。Apple最多支持将6个非系统动态库进行合并为一个。
* **尽量少加载非必要类和方法**。
* **`+load()`方法中的内容放到首屏渲染完成后执行，或者使用`+initialize()`方法替换**。因为在`+load()`方法中，进行运行时方法替换会导致4毫秒的消耗，不要小看这4毫秒，随着项目代码越来越多，`+load()`方法对启动速度的影响会越来越大。
* **控制C++全局变量的数量**。

## `main()`函数执行后

在这一阶段，指的是从`main()`函数执行开始，到appDelegate的`application:didFinishLaunchingWithOptions`方法里首屏渲染相关方法执行完成，首页的业务代码也是在这一阶段执行的，主要包括：

* 首屏初始化所需配置文件的读写操作；
* 首屏列表大量数据的读取；
* 首屏所有View对象的渲染计算。

很多时候，我们为了工作方便，常常喜欢将所有初始化操作都放在这个阶段执行，而不管这个操作的最恰当时机是不是此时，这就导致APP可能会提早执行一些操作，这些操作如果非常耗时，那么就会有明显的启动变慢现象。

所以，针对这一阶段，优化方法就是：**从功能上梳理出哪些是首屏渲染必要的初始化功能，哪些是App启动必要的初始化功能，而哪些是只需要在对应功能开始使用时才需要初始化的。然后，只执行所有首屏必要的初始化工作。**

## 首屏渲染完成后

这一阶段，是从首屏渲染完成后，一直到`application:didFinishLaunchingWithOptions`方法作用域结束时为止。在这个时间段，主要会进行非首屏其他业务模块的初始化、监听的注册、配置文件的读取等。

这个阶段用户已经能够看到 App 的首页信息了，所以优化的优先级排在最后。但是，那些会卡住主线程的方法还是需要最优先处理的，不然还是会影响到用户后面的交互操作。

## 其他优化
### 功能级别的优化

功能级别的启动优化，就是要从`main()`函数执行后这个阶段下手。优化的思路是： `main()` 函数开始执行后到首屏渲染完成前，只处理首屏相关的业务，其他非首屏业务的初始化、监听注册、配置文件读取等都放到首屏渲染完成后去做。

### 方法级别的优化

在功能级别的优化之后，我们需要进一步做的是，需要检查在首屏渲染完成之后，还有哪些方法是耗时操作，将没必要的执行延迟执行或者异步执行。

通常，耗时较多的方法都是因为在主线程上执行了大量耗时计算，具体的表现就是**加载、编辑、存储图片或者文件**。然而，并不是只优化对资源的操作就好了，还需要对其他耗时操作进行优化，比如`+load`方法会耗费4毫秒，用ReactiveCocoa每创建一个信号Signal，都需要6毫秒。这样，稍不注意，耗时操作积少成多，就会对APP的启动速度产生严重的影响。

# 知识点
## `+load` VS `+initialize`

### 相同点
`+load()` 和 `+initialize()`方法都是Objective-C中类的初始化操作，它们都会被自动调用一次，不需要调用`[super load];`和`[super initialize];`，因为一个类只需要初始化一次就足够了，不需要额外的初始化操作。

### 不同点
#### `+load()`
**当类被引用的时候，就会执行，而不会管是不是被使用了**。所以，在优化APP加载速度时，一定要减少非必要类和对象的引入。具体而言：

1. 当父类和子类都实现`load`函数时,父类的`load`方法执行顺序要优先于子类；
2. 当一个类未实现`load`方法时,不会调用父类`load`方法；
3. 类中的`load`方法执行顺序要优先于Category；
4. 当有多个Category都实现了load方法,这几个load方法都会执行,但执行**顺序不确定**。

#### `+initialize()`
**当类或者子类的被第一次使用时，才会执行类的`+initialize()`方法**。也就是说，如果某个类只是被引用进来，但是并没有使用，那么就只执行`+load()`方法，而不执行`+initialize()`方法。另外，和`+load()`一样，由于是系统自动调用，所以不需要显式调用`[super initialize];`。其他特点如下：

1. 父类的`+initialize()`方法会比子类先执行；
2. 当子类不实现`+initialize()`方法，会把父类的实现继承过来调用一遍。在此之前，父类的方法会被优先调用一次；
3. 当多个Category都实现了`initialize`方法时，会覆盖类中的方法，并且只执行一个(会执行Compile Sources 列表中最后一个Category 的)`+initialize()`方法。
