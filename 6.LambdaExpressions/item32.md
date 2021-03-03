## 条款三十二：使用初始化捕获来移动对象到闭包中

**Item 32: Use init capture to move objects into closures**

在某些场景下，按值捕获和按引用捕获都不是你所想要的。如果你有一个只能被移动的对象（例如`std::unique_ptr`或`std::future`）要进入到闭包里，使用C++11是无法实现的。如果你要复制的对象复制开销非常高，但移动的成本却不高（例如标准库中的大多数容器），并且你希望的是宁愿移动该对象到闭包而不是复制它。然而C++11却无法实现这一目标。

但那是C++11的时候。到了C++14就另一回事了，它能支持将对象移动到闭包中。如果你的编译器兼容支持C++14，那么请愉快地阅读下去。如果你仍然在使用仅支持C++11的编译器，也请愉快阅读，因为在C++11中有很多方法可以实现近似的移动捕获。

缺少移动捕获被认为是C++11的一个缺点，直接的补救措施是将该特性添加到C++14中，但标准化委员会选择了另一种方法。他们引入了一种新的捕获机制，该机制非常灵活，移动捕获是它可以执行的技术之一。新功能被称作**初始化捕获**（*init capture*），C++11捕获形式能做的所有事它几乎可以做，甚至能完成更多功能。你不能用初始化捕获表达的东西是默认捕获模式，但[Item31](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item31.md)说明提醒了你无论如何都应该远离默认捕获模式。（在C++11捕获模式所能覆盖的场景里，初始化捕获的语法有点不大方便。因此在C++11的捕获模式能完成所需功能的情况下，使用它是完全合理的）。

使用初始化捕获可以让你指定：

1. 从lambda生成的闭包类中的**数据成员名称**；
2. 初始化该成员的**表达式**；

这是使用初始化捕获将`std::unique_ptr`移动到闭包中的方法：

```c++
class Widget {                          //一些有用的类型
public:
    …
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    …
};

auto pw = std::make_unique<Widget>();   //创建Widget；使用std::make_unique
                                        //的有关信息参见条款21

…                                       //设置*pw

auto func = [pw = std::move(pw)]        //使用std::move(pw)初始化闭包数据成员
            { return pw->isValidated()
                     && pw->isArchived(); };
```

高亮的文本包含了初始化捕获的使用（译者注：高亮了“`pw = std::move(pw)`”），“`=`”的左侧是指定的闭包类中数据成员的名称，右侧则是初始化表达式。有趣的是，“`=`”左侧的作用域不同于右侧的作用域。左侧的作用域是闭包类，右侧的作用域和*lambda*定义所在的作用域相同。在上面的示例中，“`=`”左侧的名称`pw`表示闭包类中的数据成员，而右侧的名称`pw`表示在*lambda*上方声明的对象，即由调用`std::make_unique`去初始化的变量。因此，“`pw = std::move(pw)`”的意思是“在闭包中创建一个数据成员`pw`，并使用将`std::move`应用于局部变量`pw`的结果来初始化该数据成员”。

一般来说，*lambda*主体中的代码在闭包类的作用域内，因此`pw`的使用指的是闭包类的数据成员。

在此示例中，注释“设置`*pw`”表示在由`std::make_unique`创建`Widget`之后，*lambda*捕获到指向`Widget`的`std::unique_ptr`之前，该`Widget`以某种方式进行了修改。如果不需要这样的设置，即如果`std::make_unique`创建的`Widget`处于适合被*lambda*捕获的状态，则不需要局部变量`pw`，因为闭包类的数据成员可以通过`std::make_unique`直接初始化：

```c++
auto func = [pw = std::make_unique<Widget>()]   //使用调用make_unique得到的结果
            { return pw->isValidated()          //初始化闭包数据成员
                     && pw->isArchived(); };
```

这清楚地表明了，这个C++14的捕获概念是从C++11发展出来的的，在C++11中，无法捕获表达式的结果。 因此，初始化捕获的另一个名称是**通用*lambda*捕获**（*generalized lambda capture*）。

但是，如果你使用的一个或多个编译器不支持C++14的初始捕获怎么办？ 如何使用不支持移动捕获的语言完成移动捕获？

请记住，*lambda*表达式只是生成一个类和创建该类型对象的一种简单方式而已。没什么事是你用*lambda*可以做而不能自己手动实现的。 那么我们刚刚看到的C++14的示例代码可以用C++11重新编写，如下所示：

```c++
class IsValAndArch {                            //“is validated and archived”
public:
    using DataType = std::unique_ptr<Widget>;
    
    explicit IsValAndArch(DataType&& ptr)       //条款25解释了std::move的使用
    : pw(std::move(ptr)) {}
    
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
    
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

这个代码量比*lambda*表达式要多，但这并不难改变这样一个事实，即如果你希望使用一个C++11的类来支持其数据成员的移动初始化，那么你唯一要做的就是在键盘上多花点时间。

如果你坚持要使用*lambda*（并且考虑到它们的便利性，你可能会这样做），移动捕获可以在C++11中这样模拟：

1. **将要捕获的对象移动到由`std::bind`产生的函数对象中；**
2. **将“被捕获的”对象的引用赋予给*lambda*。**

如果你熟悉`std::bind`，那么代码其实非常简单。如果你不熟悉`std::bind`，那可能需要花费一些时间来习惯它，但这无疑是值得的。

假设你要创建一个本地的`std::vector`，在其中放入一组适当的值，然后将其移动到闭包中。在C++14中，这很容易实现：

```c++
std::vector<double> data;               //要移动进闭包的对象

…                                       //填充data

auto func = [data = std::move(data)]    //C++14初始化捕获
            { /*使用data*/ };
```

我已经对该代码的关键部分进行了高亮：要移动的对象的类型（`std::vector<double>`），该对象的名称（`data`）以及用于初始化捕获的初始化表达式（`std::move(data)`）。C++11的等效代码如下，其中我强调了相同的关键事项：

```c++
std::vector<double> data;               //同上

…                                       //同上

auto func =
    std::bind(                              //C++11模拟初始化捕获
        [](const std::vector<double>& data) //译者注：本行高亮
        { /*使用data*/ },
        std::move(data)                     //译者注：本行高亮
    );
```

如*lambda*表达式一样，`std::bind`产生函数对象。我将由`std::bind`返回的函数对象称为**bind对象**（*bind objects*）。`std::bind`的第一个实参是可调用对象，后续实参表示要传递给该对象的值。

一个bind对象包含了传递给`std::bind`的所有实参的副本。对于每个左值实参，bind对象中的对应对象都是复制构造的。对于每个右值，它都是移动构造的。在此示例中，第二个实参是一个右值（`std::move`的结果，请参见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)），因此将`data`移动构造到绑定对象中。这种移动构造是模仿移动捕获的关键，因为将右值移动到bind对象是我们解决无法将右值移动到C++11闭包中的方法。

当“调用”bind对象（即调用其函数调用运算符）时，其存储的实参将传递到最初传递给`std::bind`的可调用对象。在此示例中，这意味着当调用`func`（bind对象）时，`func`中所移动构造的`data`副本将作为实参传递给`std::bind`中的*lambda*。

该*lambda*与我们在C++14中使用的*lambda*相同，只是添加了一个形参`data`来对应我们的伪移动捕获对象。此形参是对bind对象中`data`副本的左值引用。（这不是右值引用，因为尽管用于初始化`data`副本的表达式（`std::move(data)`）为右值，但`data`副本本身为左值。）因此，*lambda*将对绑定在对象内部的移动构造的`data`副本进行操作。

默认情况下，从*lambda*生成的闭包类中的`operator()`成员函数为`const`的。这具有在*lambda*主体内把闭包中的所有数据成员渲染为`const`的效果。但是，bind对象内部的移动构造的`data`副本不是`const`的，因此，为了防止在*lambda*内修改该`data`副本，*lambda*的形参应声明为reference-to-`const`。 如果将*lambda*声明为`mutable`，则闭包类中的`operator()`将不会声明为`const`，并且在*lambda*的形参声明中省略`const`也是合适的：

```c++
auto func =
    std::bind(                                  //C++11对mutable lambda
        [](std::vector<double>& data) mutable	//初始化捕获的模拟
        { /*使用data*/ },
        std::move(data)
    );
```

因为bind对象存储着传递给`std::bind`的所有实参的副本，所以在我们的示例中，bind对象包含由*lambda*生成的闭包副本，这是它的第一个实参。 因此闭包的生命周期与bind对象的生命周期相同。 这很重要，因为这意味着只要存在闭包，包含伪移动捕获对象的bind对象也将存在。

如果这是你第一次接触`std::bind`，则可能需要先阅读你最喜欢的C++11参考资料，然后再讨论所有详细信息。 即使是这样，这些基本要点也应该清楚：

* 无法移动构造一个对象到C++11闭包，但是可以将对象移动构造进C++11的bind对象。
* 在C++11中模拟移动捕获包括将对象移动构造进bind对象，然后通过传引用将移动构造的对象传递给*lambda*。
* 由于bind对象的生命周期与闭包对象的生命周期相同，因此可以将bind对象中的对象视为闭包中的对象。

作为使用`std::bind`模仿移动捕获的第二个示例，这是我们之前看到的在闭包中创建`std::unique_ptr`的C++14代码：

```c++
auto func = [pw = std::make_unique<Widget>()]   //同之前一样
            { return pw->isValidated()          //在闭包中创建pw
                     && pw->isArchived(); };
```

这是C++11的模拟实现：

```c++
auto func = std::bind(
                [](const std::unique_ptr<Widget>& pw)
                { return pw->isValidated()
                         && pw->isArchived(); },
                std::make_unique<Widget>()
            );
```

具备讽刺意味的是，这里我展示了如何使用`std::bind`解决C++11 *lambda*中的限制，因为在[Item34](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item34.md)中，我主张使用*lambda*而不是`std::bind`。但是，该条款解释的是在C++11中有些情况下`std::bind`可能有用，这就是其中一种。 （在C++14中，初始化捕获和`auto`形参等特性使得这些情况不再存在。）

**请记住：**

* 使用C++14的初始化捕获将对象移动到闭包中。
* 在C++11中，通过手写类或`std::bind`的方式来模拟初始化捕获。
