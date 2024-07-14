# 第6章 *lambda*表达式

**CHAPTER 6 Lambda Expressions**

*lambda*表达式是C++编程中的游戏规则改变者。这有点令人惊讶，因为它没有给语言带来新的表达能力。*lambda*可以做的所有事情都可以通过其他方式完成。但是*lambda*是创建函数对象相当便捷的一种方法，对于日常的C++开发影响是巨大的。没有*lambda*时，STL中的“`_if`”算法（比如，`std::find_if`，`std::remove_if`，`std::count_if`等）通常需要繁琐的谓词，但是当有*lambda*可用时，这些算法使用起来就变得相当方便。用比较函数（比如，`std::sort`，`std::nth_element`，`std::lower_bound`等）来自定义算法也是同样方便的。在STL外，*lambda*可以快速创建`std::unique_ptr`和`std::shared_ptr`的自定义删除器（见[Item18](../4.SmartPointers/item18.md)和[19](../4.SmartPointers/item19.md)），并且使线程API中条件变量的谓词指定变得同样简单（参见[Item39](../7.TheConcurrencyAPI/item39.md)）。除了标准库，*lambda*有利于即时的回调函数，接口适配函数和特定上下文中的一次性函数。*lambda*确实使C++成为更令人愉快的编程语言。

与*lambda*相关的词汇可能会令人疑惑，这里做一下简单的回顾：

- ***lambda*表达式**（*lambda expression*）就是一个表达式。下面是部分源代码。在

  ```cpp
  std::find_if(container.begin(), container.end(),
               [](int val){ return 0 < val && val < 10; });   //译者注：本行高亮
  ```

  中，代码的高亮部分就是*lambda*。

- **闭包**（*enclosure*）是*lambda*创建的运行时对象。依赖捕获模式，闭包持有被捕获数据的副本或者引用。在上面的`std::find_if`调用中，闭包是作为第三个实参在运行时传递给`std::find_if`的对象。

- **闭包类**（*closure class*）是从中实例化闭包的类。每个*lambda*都会使编译器生成唯一的闭包类。*lambda*中的语句成为其闭包类的成员函数中的可执行指令。

*lambda*通常被用来创建闭包，该闭包仅用作函数的实参。上面对`std::find_if`的调用就是这种情况。然而，闭包通常可以拷贝，所以可能有多个闭包对应于一个*lambda*。比如下面的代码：

```cpp
{
    int x;                                  //x是局部对象
    …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    …
}
```

`c1`，`c2`，`c3`都是*lambda*产生的闭包的副本。

非正式的讲，模糊*lambda*，闭包和闭包类之间的界限是可以接受的。但是，在随后的Item中，区分什么存在于编译期（*lambdas* 和闭包类），什么存在于运行时（闭包）以及它们之间的相互关系是重要的。

## 条款三十一：避免使用默认捕获模式

**Item 31: Avoid default capture modes**

C++11中有两种默认的捕获模式：按引用捕获和按值捕获。但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

这就是本条款的一个总结。如果你偏向技术，渴望了解更多内容，就让我们从按引用捕获的危害谈起吧。

按引用捕获会导致闭包中包含了对某个局部变量或者形参的引用，变量或形参只在定义*lambda*的作用域中可用。如果该*lambda*创建的闭包生命周期超过了局部变量或者形参的生命周期，那么闭包中的引用将会变成悬空引用。举个例子，假如我们有元素是过滤函数（filtering function）的一个容器，该函数接受一个`int`，并返回一个`bool`，该`bool`的结果表示传入的值是否满足过滤条件：

```c++
using FilterContainer =                     //“using”参见条款9，
	std::vector<std::function<bool(int)>>;  //std::function参见条款2

FilterContainer filters;                    //过滤函数
```

我们可以添加一个过滤器，用来过滤掉5的倍数：

```c++
filters.emplace_back(                       //emplace_back的信息见条款42
	[](int value) { return value % 5 == 0; }
);
```

然而我们可能需要的是能够在运行期计算除数（divisor），即不能将5硬编码到*lambda*中。因此添加的过滤器逻辑将会是如下这样：

```c++
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
}
```

这个代码实现是一个定时炸弹。*lambda*对局部变量`divisor`进行了引用，但该变量的生命周期会在`addDivisorFilter`返回时结束，刚好就是在语句`filters.emplace_back`返回之后。因此添加到`filters`的函数添加完，该函数就死亡了。使用这个过滤器（译者注：就是那个添加进`filters`的函数）会导致未定义行为，这是由它被创建那一刻起就决定了的。

现在，同样的问题也会出现在`divisor`的显式按引用捕获。

```c++
filters.emplace_back(
    [&divisor](int value) 			    //危险！对divisor的引用将会悬空！
    { return value % divisor == 0; }
);
```

但通过显式的捕获，能更容易看到*lambda*的可行性依赖于变量`divisor`的生命周期。另外，写下“divisor”这个名字能够提醒我们要注意确保`divisor`的生命周期至少跟*lambda*闭包一样长。比起“`[&]`”传达的意思，显式捕获能让人更容易想起“确保没有悬空变量”。

如果你知道一个闭包将会被马上使用（例如被传入到一个STL算法中）并且不会被拷贝，那么在它的*lambda*被创建的环境中，将不会有闭包的引用比父函数的局部变量和形参活得长的风险。在这种情况下，你可能会争论说，没有悬空引用的危险，就不需要避免使用默认的引用捕获模式。例如，我们的过滤*lambda*只会用做C++11中`std::all_of`的一个实参，返回满足条件的所有元素：

```c++
template<typename C>
void workWithContainer(const C& container)
{
    auto calc1 = computeSomeValue1();               //同上
    auto calc2 = computeSomeValue2();               //同上
    auto divisor = computeDivisor(calc1, calc2);    //同上

    using ContElemT = typename C::value_type;       //容器内元素的类型
    using std::begin;                               //为了泛型，见条款13
    using std::end;

    if (std::all_of(                                //如果容器内所有值都为
            begin(container), end(container),       //除数的倍数
            [&](const ContElemT& value)
            { return value % divisor == 0; })
        ) {
        …                                           //它们...
    } else {
        …                                           //至少有一个不是的话...
    }
}
```

的确如此，这是安全的做法，但这种安全是不确定的。如果发现*lambda*在其它上下文中很有用（例如作为一个函数被添加在`filters`容器中），然后拷贝粘贴到一个`divisor`变量已经死亡，但闭包生命周期还没结束的上下文中，你又回到了悬空的使用上了。同时，在该捕获语句中，也没有特别提醒了你注意分析`divisor`的生命周期。

从长期来看，显式列出*lambda*依赖的局部变量和形参，是更加符合软件工程规范的做法。

额外提一下，C++14支持了在*lambda*中使用`auto`来声明变量，上面的代码在C++14中可以进一步简化，`ContElemT`的别名可以去掉，`if`条件可以修改为：

```c++
if (std::all_of(begin(container), end(container),
			   [&](const auto& value)               // C++14
			   { return value % divisor == 0; }))			
```

一个解决问题的方法是，`divisor`默认按值捕获进去，也就是说可以按照以下方式来添加*lambda*到`filters`：

```c++
filters.emplace_back( 							    //现在divisor不会悬空了
    [=](int value) { return value % divisor == 0; }
);
```

这足以满足本实例的要求，但在通常情况下，按值捕获并不能完全解决悬空引用的问题。这里的问题是如果你按值捕获的是一个指针，你将该指针拷贝到*lambda*对应的闭包里，但这样并不能避免*lambda*外`delete`这个指针的行为，从而导致你的副本指针变成悬空指针。

也许你要抗议说：“这不可能发生。看过了[第4章](../4.SmartPointers/item18.md)，我对智能指针的使用非常热衷。只有那些失败的C++98的程序员才会用裸指针和`delete`语句。”这也许是正确的，但却是不相关的，因为事实上你的确会使用裸指针，也的确存在被你`delete`的可能性。只不过在现代的C++编程风格中，不容易在源代码中显露出来而已。

假设在一个`Widget`类，可以实现向过滤器的容器添加条目：

```c++
class Widget {
public:
    …                       //构造函数等
    void addFilter() const; //向filters添加条目
private:
    int divisor;            //在Widget的过滤器使用
};
```

这是`Widget::addFilter`的定义：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}	
```

这个做法看起来是安全的代码。*lambda*依赖于`divisor`，但默认的按值捕获确保`divisor`被拷贝进了*lambda*对应的所有闭包中，对吗？

错误，完全错误。

捕获只能应用于*lambda*被创建时所在作用域里的non-`static`局部变量（包括形参）。在`Widget::addFilter`的视线里，`divisor`并不是一个局部变量，而是`Widget`类的一个成员变量。它不能被捕获。而如果默认捕获模式被删除，代码就不能编译了：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(                               //错误！
        [](int value) { return value % divisor == 0; }  //divisor不可用
    ); 
} 
```

另外，如果尝试去显式地捕获`divisor`变量（或者按引用或者按值——这不重要），也一样会编译失败，因为`divisor`不是一个局部变量或者形参。

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value)                //错误！没有名为divisor局部变量可捕获
        { return value % divisor == 0; }
    );
}
```

所以如果默认按值捕获不能捕获`divisor`，而不用默认按值捕获代码就不能编译，这是怎么一回事呢？

解释就是这里隐式使用了一个原始指针：`this`。每一个non-`static`成员函数都有一个`this`指针，每次你使用一个类内的数据成员时都会使用到这个指针。例如，在任何`Widget`成员函数中，编译器会在内部将`divisor`替换成`this->divisor`。在默认按值捕获的`Widget::addFilter`版本中，

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```

真正被捕获的是`Widget`的`this`指针，而不是`divisor`。编译器会将上面的代码看成以下的写法：

```c++
void Widget::addFilter() const
{
    auto currentObjectPtr = this;

    filters.emplace_back(
        [currentObjectPtr](int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```

明白了这个就相当于明白了*lambda*闭包的生命周期与`Widget`对象的关系，闭包内含有`Widget`的`this`指针的拷贝。特别是考虑以下的代码，参考[第4章](../4.SmartPointers/item18.md)的内容，只使用智能指针：

```c++
using FilterContainer = 					//跟之前一样
    std::vector<std::function<bool(int)>>;

FilterContainer filters;                    //跟之前一样

void doSomeWork()
{
    auto pw =                               //创建Widget；std::make_unique
        std::make_unique<Widget>();         //见条款21

    pw->addFilter();                        //添加使用Widget::divisor的过滤器

    …
}                                           //销毁Widget；filters现在持有悬空指针！
```

当调用`doSomeWork`时，就会创建一个过滤器，其生命周期依赖于由`std::make_unique`产生的`Widget`对象，即一个含有指向`Widget`的指针——`Widget`的`this`指针——的过滤器。这个过滤器被添加到`filters`中，但当`doSomeWork`结束时，`Widget`会由管理它的`std::unique_ptr`来销毁（见[Item18](../4.SmartPointers/item18.md)）。从这时起，`filter`会含有一个存着悬空指针的条目。

这个特定的问题可以通过给你想捕获的数据成员做一个局部副本，然后捕获这个副本去解决：

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [divisorCopy](int value)                //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```

事实上如果采用这种方法，默认的按值捕获也是可行的。

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [=](int value)                          //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```

但为什么要冒险呢？当一开始你认为你捕获的是`divisor`的时候，默认捕获模式就是造成可能意外地捕获`this`的元凶。

在C++14中，一个更好的捕获成员变量的方式时使用通用的*lambda*捕获：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(                   //C++14：
        [divisor = divisor](int value)      //拷贝divisor到闭包
        { return value % divisor == 0; }	//使用这个副本
    );
}
```

这种通用的*lambda*捕获并没有默认的捕获模式，因此在C++14中，本条款的建议——避免使用默认捕获模式——仍然是成立的。

使用默认的按值捕获还有另外的一个缺点，它们预示了相关的闭包是独立的并且不受外部数据变化的影响。一般来说，这是不对的。*lambda*可能会依赖局部变量和形参（它们可能被捕获），还有**静态存储生命周期**（static storage duration）的对象。这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为`static`。这些对象也能在*lambda*里使用，但它们不能被捕获。但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。参考下面版本的`addDivisorFilter`函数：

```c++
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();    //现在是static
    static auto calc2 = computeSomeValue2();    //现在是static
    static auto divisor =                       //现在是static
    computeDivisor(calc1, calc2);

    filters.emplace_back(
        [=](int value)                          //什么也没捕获到！
        { return value % divisor == 0; }        //引用上面的static
    );

    ++divisor;                                  //调整divisor
}
```

随意地看了这份代码的读者可能看到“`[=]`”，就会认为“好的，*lambda*拷贝了所有使用的对象，因此这是独立的”。但其实不独立。这个*lambda*没有使用任何的non-`static`局部变量，所以它没有捕获任何东西。然而*lambda*的代码引用了`static`变量`divisor`，在每次调用`addDivisorFilter`的结尾，`divisor`都会递增，通过这个函数添加到`filters`的所有*lambda*都展示新的行为（分别对应新的`divisor`值）。这个*lambda*是通过引用捕获`divisor`，这和默认的按值捕获表示的含义有着直接的矛盾。如果你一开始就避免使用默认的按值捕获模式，你就能解除代码的风险。

**请记住：**

+ 默认的按引用捕获可能会导致悬空引用。
+ 默认的按值捕获对于悬空指针很敏感（尤其是`this`指针），并且它会误导人产生*lambda*是独立的想法。

