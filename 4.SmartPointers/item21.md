## 条款二十一：优先考虑使用`std::make_unique`和`std::make_shared`而非`new`

**Item 21: Prefer `std::make_unique` and `std::make_shared` to direct use of `new`**

让我们先对`std::make_unique`和`std::make_shared`做个铺垫。`std::make_shared`是C++11标准的一部分，但很可惜的是，`std::make_unique`不是。它从C++14开始加入标准库。如果你在使用C++11，不用担心，一个基础版本的`std::make_unique`是很容易自己写出的，如下：

```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

正如你看到的，`make_unique`只是将它的参数完美转发到所要创建的对象的构造函数，从`new`产生的原始指针里面构造出`std::unique_ptr`，并返回这个`std::unique_ptr`。这种形式的函数不支持数组和自定义析构（见[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)），但它给出了一个示范：只需一点努力就能写出你想要的`make_unique`函数。（要用可能的最小的努力创建一个有所有特性的`make_unique`，去找提供这个的标准化文件吧，然后拷贝你在那看到的那个实现。你想要的这个文件是N3656，是Stephan T. Lavavej写于2013-04-18的文档。）需要记住的是，不要把它放到`std`命名空间中，因为你可能并不希望在升级厂家编译器到符合C++14标准的时候产生冲突。

`std::make_unique`和`std::make_shared`是三个**`make`函数**中的两个：接收任意的多参数集合，完美转发到构造函数去动态分配一个对象，然后返回这个指向这个对象的指针。第三个`make`函数是`std::allocate_shared`。它行为和`std::make_shared`一样，只不过第一个参数是用来动态分配内存的*allocator*对象。

即使是对使用和不使用`make`函数创建智能指针的最简单比较，也揭示了为什么最好使用`make`函数的第一个原因。例如：

```c++
auto upw1(std::make_unique<Widget>());      //使用make函数
std::unique_ptr<Widget> upw2(new Widget);   //不使用make函数
auto spw1(std::make_shared<Widget>());      //使用make函数
std::shared_ptr<Widget> spw2(new Widget);   //不使用make函数
```

我高亮了区别：使用`new`的版本重复了类型，但是`make`函数的版本没有。（译者注：这里高亮的是`Widget`，用`new`的声明语句需要写2遍`Widget`，`make`函数只需要写一次。）重复写类型和软件工程里面一个关键原则相冲突：应该避免重复代码。源代码中的重复增加了编译的时间，会导致目标代码冗余，并且通常会让代码库使用更加困难。它经常演变成不一致的代码，而代码库中的不一致常常导致bug。此外，打两次字比一次更费力，而且谁不喜欢减少打字负担？

第二个使用`make`函数的原因和异常安全有关。假设我们有个函数按照某种优先级处理`Widget`：

```c++
void processWidget(std::shared_ptr<Widget> spw, int priority);
```

值传递`std::shared_ptr`可能看起来很可疑，但是[Item41](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item41.md)解释了，如果`processWidget`总是复制`std::shared_ptr`（例如，通过将其存储在已处理的`Widget`的一个数据结构中），那么这可能是一个合理的设计选择。

现在假设我们有一个函数来计算相关的优先级，

```c++
int computePriority();
```

并且我们在调用`processWidget`时使用了`new`而不是`std::make_shared`：

```c++
processWidget(std::shared_ptr<Widget>(new Widget),  //潜在的资源泄漏！
              computePriority());
```

如注释所说，这段代码可能在`new`一个`Widget`时发生泄漏。为何？调用的代码和被调用的函数都用`std::shared_ptr`s，且`std::shared_ptr`s就是设计出来防止泄漏的。它们会在最后一个`std::shared_ptr`销毁时自动释放所指向的内存。如果每个人在每个地方都用`std::shared_ptr`s，这段代码怎么会泄漏呢？

答案和编译器将源码转换为目标代码有关。在运行时，一个函数的实参必须先被计算，这个函数再被调用，所以在调用`processWidget`之前，必须执行以下操作，`processWidget`才开始执行：

- 表达式“`new Widget`”必须计算，例如，一个`Widget`对象必须在堆上被创建
- 负责管理`new`出来指针的`std::shared_ptr<Widget>`构造函数必须被执行
-  `computePriority`必须运行

编译器不需要按照执行顺序生成代码。“`new Widget`”必须在`std::shared_ptr`的构造函数被调用前执行，因为`new`出来的结果作为构造函数的实参，但`computePriority`可能在这之前，之后，或者**之间**执行。也就是说，编译器可能按照这个执行顺序生成代码：

1. 执行“`new Widget`”
2. 执行`computePriority`
3. 运行`std::shared_ptr`构造函数

如果按照这样生成代码，并且在运行时`computePriority`产生了异常，那么第一步动态分配的`Widget`就会泄漏。因为它永远都不会被第三步的`std::shared_ptr`所管理了。

使用`std::make_shared`可以防止这种问题。调用代码看起来像是这样：

```c++
processWidget(std::make_shared<Widget>(),   //没有潜在的资源泄漏
              computePriority());
```

在运行时，`std::make_shared`和`computePriority`其中一个会先被调用。如果是`std::make_shared`先被调用，在`computePriority`调用前，动态分配`Widget`的原始指针会安全的保存在作为返回值的`std::shared_ptr`中。如果`computePriority`产生一个异常，那么`std::shared_ptr`析构函数将确保管理的`Widget`被销毁。如果首先调用`computePriority`并产生一个异常，那么`std::make_shared`将不会被调用，因此也就不需要担心动态分配`Widget`（会泄漏）。

如果我们将`std::shared_ptr`，`std::make_shared`替换成`std::unique_ptr`，`std::make_unique`，同样的道理也适用。因此，在编写异常安全代码时，使用`std::make_unique`而不是`new`与使用`std::make_shared`（而不是`new`）同样重要。

`std::make_shared`的一个特性（与直接使用`new`相比）是效率提升。使用`std::make_shared`允许编译器生成更小，更快的代码，并使用更简洁的数据结构。考虑以下对new的直接使用：

```c++
std::shared_ptr<Widget> spw(new Widget);
```
显然，这段代码需要进行内存分配，但它实际上执行了两次。[Item19](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item19.md)解释了每个`std::shared_ptr`指向一个控制块，其中包含被指向对象的引用计数，还有其他东西。这个控制块的内存在`std::shared_ptr`构造函数中分配。因此，直接使用`new`需要为`Widget`进行一次内存分配，为控制块再进行一次内存分配。

如果使用`std::make_shared`代替：

```c++
auto spw = std::make_shared<Widget>();
```

一次分配足矣。这是因为`std::make_shared`分配一块内存，同时容纳了`Widget`对象和控制块。这种优化减少了程序的静态大小，因为代码只包含一个内存分配调用，并且它提高了可执行代码的速度，因为内存只分配一次。此外，使用`std::make_shared`避免了对控制块中的某些簿记信息的需要，潜在地减少了程序的总内存占用。

对于`std::make_shared`的效率分析同样适用于`std::allocate_shared`，因此`std::make_shared`的性能优势也扩展到了该函数。

更倾向于使用`make`函数而不是直接使用`new`的争论非常激烈。尽管它们在软件工程、异常安全和效率方面具有优势，但本条款的意见是，更**倾向于**使用`make`函数，而不是完全依赖于它们。这是因为有些情况下它们不能或不应该被使用。

例如，`make`函数都不允许指定自定义删除器（见[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)和[19](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item19.md)），但是`std::unique_ptr`和`std::shared_ptr`有构造函数这么做。有个`Widget`的自定义删除器：
```cpp
auto widgetDeleter = [](Widget* pw) { … };
```
创建一个使用它的智能指针只能直接使用`new`：
```cpp
std::unique_ptr<Widget, decltype(widgetDeleter)>
    upw(new Widget, widgetDeleter);

std::shared_ptr<Widget> spw(new Widget, widgetDeleter);
```
对于`make`函数，没有办法做同样的事情。

`make`函数第二个限制来自于其实现中的句法细节。[Item7](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item7.md)解释了，当构造函数重载，有使用`std::initializer_list`作为参数的和不用其作为参数的时，用花括号创建对象更倾向于使用`std::initializer_list`作为形参的构造函数，而用小括号创建对象调用不用`std::initializer_list`作为参数的构造函数。`make`函数会将它们的参数完美转发给对象构造函数，但是它们是使用小括号还是花括号？对某些类型，问题的答案会很不相同。例如，在这些调用中，

```cpp
auto upv = std::make_unique<std::vector<int>>(10, 20);
auto spv = std::make_shared<std::vector<int>>(10, 20);
```
生成的智能指针指向带有10个元素的`std::vector`，每个元素值为20，还是指向带有两个元素的`std::vector`，其中一个元素值10，另一个为20？或者结果是不确定的？

好消息是这并非不确定：两种调用都创建了10个元素，每个值为20的`std::vector`。这意味着在`make`函数中，完美转发使用小括号，而不是花括号。坏消息是如果你想用花括号初始化指向的对象，你必须直接使用`new`。使用`make`函数会需要能够完美转发花括号初始化的能力，但是，正如[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)所说，花括号初始化无法完美转发。但是，[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)介绍了一个变通的方法：使用`auto`类型推导从花括号初始化创建`std::initializer_list`对象（见[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)），然后将`auto`创建的对象传递给`make`函数。

```cpp
//创建std::initializer_list
auto initList = { 10, 20 };
//使用std::initializer_list为形参的构造函数创建std::vector
auto spv = std::make_shared<std::vector<int>>(initList);
```

对于`std::unique_ptr`，只有这两种情景（自定义删除器和花括号初始化）使用`make`函数有点问题。对于`std::shared_ptr`和它的`make`函数，还有2个问题。都属于边界问题，但是一些开发者常碰到，你也可能是其中之一。

一些类重载了`operator new`和`operator delete`。这些函数的存在意味着对这些类型的对象的全局内存分配和释放是不合常规的。设计这种定制操作往往只会精确的分配、释放对象大小的内存。例如，`Widget`类的`operator new`和`operator delete`只会处理`sizeof(Widget)`大小的内存块的分配和释放。这种系列行为不太适用于`std::shared_ptr`对自定义分配（通过`std::allocate_shared`）和释放（通过自定义删除器）的支持，因为`std::allocate_shared`需要的内存总大小不等于动态分配的对象大小，还需要**再加上**控制块大小。因此，使用`make`函数去创建重载了`operator new`和`operator delete`类的对象是个典型的糟糕想法。

与直接使用`new`相比，`std::make_shared`在大小和速度上的优势源于`std::shared_ptr`的控制块与指向的对象放在同一块内存中。当对象的引用计数降为0，对象被销毁（即析构函数被调用）。但是，因为控制块和对象被放在同一块分配的内存块中，直到控制块的内存也被销毁，对象占用的内存才被释放。

正如我说，控制块除了引用计数，还包含簿记信息。引用计数追踪有多少`std::shared_ptr`s指向控制块，但控制块还有第二个计数，记录多少个`std::weak_ptr`s指向控制块。第二个引用计数就是*weak count*。（实际上，*weak count*的值不总是等于指向控制块的`std::weak_ptr`的数目，因为库的实现者找到一些方法在*weak count*中添加附加信息，促进更好的代码产生。为了本条款的目的，我们会忽略这一点，假定*weak count*的值等于指向控制块的`std::weak_ptr`的数目。）当一个`std::weak_ptr`检测它是否失效时（见[Item19](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item19.md)），它会检测指向的控制块中的引用计数（而不是*weak count*）。如果引用计数是0（即对象没有`std::shared_ptr`再指向它，已经被销毁了），`std::weak_ptr`就已经失效。否则就没失效。

只要`std::weak_ptr`s引用一个控制块（即*weak count*大于零），该控制块必须继续存在。只要控制块存在，包含它的内存就必须保持分配。通过`std::shared_ptr`的`make`函数分配的内存，直到最后一个`std::shared_ptr`和最后一个指向它的`std::weak_ptr`已被销毁，才会释放。

如果对象类型非常大，而且销毁最后一个`std::shared_ptr`和销毁最后一个`std::weak_ptr`之间的时间很长，那么在销毁对象和释放它所占用的内存之间可能会出现延迟。

```c++
class ReallyBigType { … };

auto pBigObj =                          //通过std::make_shared
    std::make_shared<ReallyBigType>();  //创建一个大对象
                    
…           //创建std::shared_ptrs和std::weak_ptrs
            //指向这个对象，使用它们

…           //最后一个std::shared_ptr在这销毁，
            //但std::weak_ptrs还在

…           //在这个阶段，原来分配给大对象的内存还分配着

…           //最后一个std::weak_ptr在这里销毁；
            //控制块和对象的内存被释放
```

直接只用`new`，一旦最后一个`std::shared_ptr`被销毁，`ReallyBigType`对象的内存就会被释放：

```c++
class ReallyBigType { … };              //和之前一样

std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType);
                                        //通过new创建大对象

…           //像之前一样，创建std::shared_ptrs和std::weak_ptrs
            //指向这个对象，使用它们
            
…           //最后一个std::shared_ptr在这销毁,
            //但std::weak_ptrs还在；
            //对象的内存被释放

…           //在这阶段，只有控制块的内存仍然保持分配

…           //最后一个std::weak_ptr在这里销毁；
            //控制块内存被释放
```

如果你发现自己处于不可能或不合适使用`std::make_shared`的情况下，你将想要保证自己不受我们之前看到的异常安全问题的影响。最好的方法是确保在直接使用`new`时，在**一个不做其他事情的语句中**，立即将结果传递到智能指针构造函数。这可以防止编译器生成的代码在使用`new`和调用管理`new`出来对象的智能指针的构造函数之间发生异常。

例如，考虑我们前面讨论过的`processWidget`函数，对其非异常安全调用的一个小修改。这一次，我们将指定一个自定义删除器:
```c++
void processWidget(std::shared_ptr<Widget> spw,     //和之前一样
                   int priority);
void cusDel(Widget *ptr);                           //自定义删除器
```

这是非异常安全调用:
```c++
processWidget( 									    //和之前一样，
    std::shared_ptr<Widget>(new Widget, cusDel),    //潜在的内存泄漏！
    computePriority() 
);
```
回想一下：如果`computePriority`在“`new Widget`”之后，而在`std::shared_ptr`构造函数之前调用，并且如果`computePriority`产生一个异常，那么动态分配的`Widget`将会泄漏。

这里使用自定义删除排除了对`std::make_shared`的使用，因此避免出现问题的方法是将`Widget`的分配和`std::shared_ptr`的构造放入它们自己的语句中，然后使用得到的`std::shared_ptr`调用`processWidget`。这是该技术的本质，不过，正如我们稍后将看到的，我们可以对其进行调整以提高其性能：
```c++
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computePriority());  // 正确，但是没优化，见下
```
这是可行的，因为`std::shared_ptr`获取了传递给它的构造函数的原始指针的所有权，即使构造函数产生了一个异常。此例中，如果`spw`的构造函数抛出异常（比如无法为控制块动态分配内存），仍然能够保证`cusDel`会在“`new Widget`”产生的指针上调用。

一个小小的性能问题是，在非异常安全调用中，我们将一个右值传递给`processWidget`：
```c++
processWidget(
    std::shared_ptr<Widget>(new Widget, cusDel),    //实参是一个右值
    computePriority()
);
```
但是在异常安全调用中，我们传递了左值：
```c++
processWidget(spw, computePriority());              //实参是左值
```
因为`processWidget`的`std::shared_ptr`形参是传值，从右值构造只需要移动，而传递左值构造需要拷贝。对`std::shared_ptr`而言，这种区别是有意义的，因为拷贝`std::shared_ptr`需要对引用计数原子递增，移动则不需要对引用计数有操作。为了使异常安全代码达到非异常安全代码的性能水平，我们需要用`std::move`将`spw`转换为右值（见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)）：
```c++
processWidget(std::move(spw), computePriority());   //高效且异常安全
```
这很有趣，也值得了解，但通常是无关紧要的，因为您很少有理由不使用`make`函数。除非你有令人信服的理由这样做，否则你应该使用`make`函数。

**请记住：**
- 和直接使用`new`相比，`make`函数消除了代码重复，提高了异常安全性。对于`std::make_shared`和`std::allocate_shared`，生成的代码更小更快。
- 不适合使用`make`函数的情况包括需要指定自定义删除器和希望用花括号初始化
- 对于`std::shared_ptr`s，`make`函数可能不被建议的其他情况包括(1)有自定义内存管理的类；(2)特别关注内存的系统，非常大的对象，以及`std::weak_ptr`s比对应的`std::shared_ptr`s活得更久
