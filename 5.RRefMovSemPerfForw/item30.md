## 条款三十：熟悉完美转发失败的情况

**Item 30: Familiarize yourself with perfect forwarding failure cases**

C++11最显眼的功能之一就是完美转发功能。**完美**转发，太**完美**了！哎，开始使用，你就发现“完美”，理想与现实还是有差距。C++11的完美转发是非常好用，但是只有当你愿意忽略一些误差情况（译者注：就是完美转发失败的情况），这个条款就是使你熟悉这些情形。

在我们开始误差探索之前，有必要回顾一下“完美转发”的含义。“转发”仅表示将一个函数的形参传递——就是**转发**——给另一个函数。对于第二个函数（被传递的那个）目标是收到与第一个函数（执行传递的那个）完全相同的对象。这规则排除了按值传递的形参，因为它们是原始调用者传入内容的**拷贝**。我们希望被转发的函数能够使用最开始传进来的那些对象。指针形参也被排除在外，因为我们不想强迫调用者传入指针。关于通常目的的转发，我们将处理引用形参。

**完美转发**（*perfect forwarding*）意味着我们不仅转发对象，我们还转发显著的特征：它们的类型，是左值还是右值，是`const`还是`volatile`。结合到我们会处理引用形参，这意味着我们将使用通用引用（参见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），因为通用引用形参被传入实参时才确定是左值还是右值。

假定我们有一些函数`f`，然后想编写一个转发给它的函数（事实上是一个函数模板）。我们需要的核心看起来像是这样：

```cpp
template<typename T>
void fwd(T&& param)             //接受任何实参
{
    f(std::forward<T>(param));  //转发给f
}
```

从本质上说，转发函数是通用的。例如`fwd`模板，接受任何类型的实参，并转发得到的任何东西。这种通用性的逻辑扩展是，转发函数不仅是模板，而且是可变模板，因此可以接受任何数量的实参。`fwd`的可变形式如下：

```cpp
template<typename... Ts>
void fwd(Ts&&... params)            //接受任何实参
{
    f(std::forward<Ts>(params)...); //转发给f
}
```

这种形式你会在标准化容器放置函数（emplace functions）中（参见[Item42](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item42.md)）和智能指针的工厂函数`std::make_unique`和`std::make_shared`中（参见[Item21](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item21.md)）看到，当然还有其他一些地方。

给定我们的目标函数`f`和转发函数`fwd`，如果`f`使用某特定实参做一件事，但是`fwd`使用相同的实参做另一件事，完美转发就会失败：

```cpp
f( expression );        //如果这个做某件事，
fwd( expression );		//但是这个做另外的某件事，fwd完美转发expression给f会失败
```

导致这种失败的实参种类有很多。知道它们是什么以及如何解决它们很重要，因此让我们来看看无法做到完美转发的实参类型。

### 花括号初始化

假定`f`这样声明：

```cpp
void f(const std::vector<int>& v);
```

在这个例子中，用花括号初始化调用`f`通过编译，

```cpp
f({ 1, 2, 3 });         //可以，“{1, 2, 3}”隐式转换为std::vector<int>
```

但是传递相同的列表初始化给fwd不能编译

```cpp
fwd({ 1, 2, 3 });       //错误！不能编译
```

这是因为这是完美转发失效的一种情况。

所有这种错误有相同的原因。在对`f`的直接调用（例如`f({ 1, 2, 3 })`），编译器看看调用地传入的实参，看看`f`声明的形参类型。它们把调用地的实参和声明的实参进行比较，看看是否匹配，并且必要时执行隐式转换操作使得调用成功。在上面的例子中，从`{ 1, 2, 3 }`生成了临时`std::vector<int>`对象，因此`f`的形参`v`会绑定到`std::vector<int>`对象上。

当通过调用函数模板`fwd`间接调用`f`时，编译器不再把调用地传入给`fwd`的实参和`f`的声明中形参类型进行比较。而是**推导**传入给`fwd`的实参类型，然后比较推导后的参数类型和`f`的形参声明类型。当下面情况任何一个发生时，完美转发就会失败：

- **编译器不能推导出`fwd`的一个或者多个形参类型。**这种情况下代码无法编译。
- **编译器推导“错”了`fwd`的一个或者多个形参类型。**在这里，“错误”可能意味着`fwd`的实例将无法使用推导出的类型进行编译，但是也可能意味着使用`fwd`的推导类型调用`f`，与用传给`fwd`的实参直接调用`f`表现出不一致的行为。这种不同行为的原因可能是因为`f`是个重载函数的名字，并且由于是“不正确的”类型推导，在`fwd`内部调用的`f`重载和直接调用的`f`重载不一样。

在上面的`fwd({ 1, 2, 3 })`例子中，问题在于，将花括号初始化传递给未声明为`std::initializer_list`的函数模板形参，被判定为——就像标准说的——“非推导上下文”。简单来讲，这意味着编译器不准在对`fwd`的调用中推导表达式`{ 1, 2, 3 }`的类型，因为`fwd`的形参没有声明为`std::initializer_list`。对于`fwd`形参的推导类型被阻止，编译器只能拒绝该调用。

有趣的是，[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)说明了使用花括号初始化的`auto`的变量的类型推导是成功的。这种变量被视为`std::initializer_list`对象，在转发函数应推导出类型为`std::initializer_list`的情况，这提供了一种简单的解决方法——使用`auto`声明一个局部变量，然后将局部变量传进转发函数：

```cpp
auto il = { 1, 2, 3 };  //il的类型被推导为std::initializer_list<int>
fwd(il);                //可以，完美转发il给f
```

### `0`或者`NULL`作为空指针

[Item8](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item8.md)说明当你试图传递`0`或者`NULL`作为空指针给模板时，类型推导会出错，会把传来的实参推导为一个整型类型（典型情况为`int`）而不是指针类型。结果就是不管是`0`还是`NULL`都不能作为空指针被完美转发。解决方法非常简单，传一个`nullptr`而不是`0`或者`NULL`。具体的细节，参考[Item8](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item8.md)。

### 仅有声明的整型`static const`数据成员

通常，无需在类中定义整型`static const`数据成员；声明就可以了。这是因为编译器会对此类成员实行**常量传播**（*const propagation*），因此消除了保留内存的需要。比如，考虑下面的代码：

```cpp
class Widget {
public:
    static const std::size_t MinVals = 28;  //MinVal的声明
    …
};
…                                           //没有MinVals定义

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);        //使用MinVals
```

这里，我们使用`Widget::MinVals`（或者简单点`MinVals`）来确定`widgetData`的初始容量，即使`MinVals`缺少定义。编译器通过将值28放入所有提到`MinVals`的位置来补充缺少的定义（就像它们被要求的那样）。没有为`MinVals`的值留存储空间是没有问题的。如果要使用`MinVals`的地址（例如，有人创建了指向`MinVals`的指针），则`MinVals`需要存储（这样指针才有可指向的东西），尽管上面的代码仍然可以编译，但是链接时就会报错，直到为`MinVals`提供定义。

按照这个思路，想象下`f`（`fwd`要转发实参给它的那个函数）这样声明：

```cpp
void f(std::size_t val);
```

使用`MinVals`调用`f`是可以的，因为编译器直接将值28代替`MinVals`：

```cpp
f(Widget::MinVals);         //可以，视为“f(28)”
```

不过如果我们尝试通过`fwd`调用`f`，事情不会进展那么顺利：

```cpp
fwd(Widget::MinVals);       //错误！不应该链接
```

代码可以编译，但是不应该链接。如果这让你想到使用`MinVals`地址会发生的事，确实，底层的问题是一样的。

尽管代码中没有使用`MinVals`的地址，但是`fwd`的参数是通用引用，而引用，在编译器生成的代码中，通常被视作指针。在程序的二进制底层代码中（以及硬件中）指针和引用是一样的。在这个水平上，引用只是可以自动解引用的指针。在这种情况下，通过引用传递`MinVals`实际上与通过指针传递`MinVals`是一样的，因此，必须有内存使得指针可以指向。通过引用传递的整型`static const`数据成员，通常需要定义它们，这个要求可能会造成在不使用完美转发的代码成功的地方，使用等效的完美转发失败。（译者注：这里意思应该是没有定义，完美转发就会失败）

可能你也注意到了在上述讨论中我使用了一些模棱两可的词。代码“不应该”链接。引用“通常”被看做指针。传递整型`static const`数据成员“通常”要求定义。看起来就像有些事情我没有告诉你......

确实，根据标准，通过引用传递`MinVals`要求有定义。但不是所有的实现都强制要求这一点。所以，取决于你的编译器和链接器，你可能发现你可以在未定义的情况使用完美转发，恭喜你，但是这不是那样做的理由。为了具有可移植性，只要给整型`static const`提供一个定义，比如这样：

```cpp
const std::size_t Widget::MinVals;  //在Widget的.cpp文件
```

注意定义中不要重复初始化（这个例子中就是赋值28）。但是不要忽略这个细节。如果你忘了，并且在两个地方都提供了初始化，编译器就会报错，提醒你只能初始化一次。

### 重载函数的名称和模板名称

假定我们的函数`f`（我们想通过`fwd`完美转发实参给的那个函数）可以通过向其传递执行某些功能的函数来自定义其行为。假设这个函数接受和返回值都是`int`，`f`声明就像这样：

```cpp
void f(int (*pf)(int));             //pf = “process function”
```

值得注意的是，也可以使用更简单的非指针语法声明。这种声明就像这样，含义与上面是一样的：

```cpp
void f(int pf(int));                //与上面定义相同的f
```

无论哪种写法都可，现在假设我们有了一个重载函数，`processVal`：

```cpp
int processVal(int value);
int processVal(int value, int priority);
```

我们可以传递`processVal`给`f`，

```cpp
f(processVal);                      //可以
```

但是我们会发现一些吃惊的事情。`f`要求一个函数指针作为实参，但是`processVal`不是一个函数指针或者一个函数，它是同名的两个不同函数。但是，编译器可以知道它需要哪个：匹配上`f`的形参类型的那个。因此选择了一个`int`参数的`processVal`地址传递给`f`。

工作的基本机制是`f`的声明让编译器识别出哪个是需要的`processVal`。但是，`fwd`是一个函数模板，没有它可接受的类型的信息，使得编译器不可能决定出哪个函数应被传递：

```cpp
fwd(processVal);                    //错误！那个processVal？
```

单用`processVal`是没有类型信息的，所以就不能类型推导，完美转发失败。

如果我们试图使用函数模板而不是（或者也加上）重载函数的名字，同样的问题也会发生。一个函数模板不代表单独一个函数，它表示一个函数族：

```cpp
template<typename T>
T workOnVal(T param)                //处理值的模板
{ … }

fwd(workOnVal);                     //错误！哪个workOnVal实例？
```

要让像`fwd`的完美转发函数接受一个重载函数名或者模板名，方法是指定要转发的那个重载或者实例。比如，你可以创造与`f`相同形参类型的函数指针，通过`processVal`或者`workOnVal`实例化这个函数指针（这可以引导选择正确版本的`processVal`或者产生正确的`workOnVal`实例），然后传递指针给`fwd`：

```cpp
using ProcessFuncType =                         //写个类型定义；见条款9
    int (*)(int);

ProcessFuncType processValPtr = processVal;     //指定所需的processVal签名

fwd(processValPtr);                             //可以
fwd(static_cast<ProcessFuncType>(workOnVal));   //也可以
```

当然，这要求你知道`fwd`转发的函数指针的类型。没有理由去假定完美转发函数会记录着这些东西。毕竟，完美转发被设计为接受任何内容，所以如果没有文档告诉你要传递什么，你又从何而知这些东西呢？

### 位域

完美转发最后一种失败的情况是函数实参使用位域这种类型。为了更直观的解释，IPv4的头部有如下模型：（这假定的是位域是按从最低有效位（*least significant bit*，lsb）到最高有效位（*most significant bit*，msb）布局的。C++不保证这一点，但是编译器经常提供一种机制，允许程序员控制位域布局。）

```cpp
struct IPv4Header {
    std::uint32_t version:4,
                  IHL:4,
                  DSCP:6,
                  ECN:2,
                  totalLength:16;
    …
};
```

如果声明我们的函数`f`（转发函数`fwd`的目标）为接收一个`std::size_t`的参数，则使用`IPv4Header`对象的`totalLength`字段进行调用没有问题：

```cpp
void f(std::size_t sz);         //要调用的函数

IPv4Header h;
…
f(h.totalLength);               //可以
```

如果通过`fwd`转发`h.totalLength`给`f`呢，那就是一个不同的情况了：

```cpp
fwd(h.totalLength);             //错误！
```

问题在于`fwd`的形参是引用，而`h.totalLength`是non-`const`位域。听起来并不是那么糟糕，但是C++标准非常清楚地谴责了这种组合：non-`const`引用不应该绑定到位域。禁止的理由很充分。位域可能包含了机器字的任意部分（比如32位`int`的3-5位），但是这些东西无法直接寻址。我之前提到了在硬件层面引用和指针是一样的，所以没有办法创建一个指向任意*bit*的指针（C++规定你可以指向的最小单位是`char`），同样没有办法绑定引用到任意*bit*上。

一旦意识到接收位域实参的函数都将接收位域的**副本**，就可以轻松解决位域不能完美转发的问题。毕竟，没有函数可以绑定引用到位域，也没有函数可以接受指向位域的指针，因为不存在这种指针。位域可以传给的形参种类只有按值传递的形参，有趣的是，还有reference-to-`const`。在传值形参的情况中，被调用的函数接受了一个位域的副本；在传reference-to-`const`形参的情况中，标准要求这个引用实际上绑定到存放位域值的副本对象，这个对象是某种整型（比如`int`）。reference-to-`const`不直接绑定到位域，而是绑定位域值拷贝到的一个普通对象。

传递位域给完美转发的关键就是利用传给的函数接受的是一个副本的事实。你可以自己创建副本然后利用副本调用完美转发。在`IPv4Header`的例子中，可以如下写法：

```cpp
//拷贝位域值；参看条款6了解关于初始化形式的信息
auto length = static_cast<std::uint16_t>(h.totalLength);

fwd(length);                    //转发这个副本
```

### 总结

在大多数情况下，完美转发工作的很好。你基本不用考虑其他问题。但是当其不工作时——当看起来合理的代码无法编译，或者更糟的是，虽能编译但无法按照预期运行时——了解完美转发的缺陷就很重要了。同样重要的是如何解决它们。在大多数情况下，都很简单。

**请记住：**

- 当模板类型推导失败或者推导出错误类型，完美转发会失败。
- 导致完美转发失败的实参种类有花括号初始化，作为空指针的`0`或者`NULL`，仅有声明的整型`static const`数据成员，模板和重载函数的名字，位域。
