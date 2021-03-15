## 条款二十五：对右值引用使用`std::move`，对通用引用使用`std::forward`

**Item 25: Use `std::move` on rvalue references, `std::forward` on universal references**

右值引用仅绑定可以移动的对象。如果你有一个右值引用形参就知道这个对象可能会被移动：

```cpp
class Widget {
    Widget(Widget&& rhs);       //rhs定义上引用一个有资格移动的对象
    …
};
```

这是个例子，你将希望传递这样的对象给其他函数，允许那些函数利用对象的右值性（rvalueness）。这样做的方法是将绑定到此类对象的形参转换为右值。如[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)中所述，这不仅是`std::move`所做，而且它的创建就是为了这个目的：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)        //rhs是右值引用
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
      { … }
    …
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

另一方面（查看[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），通用引用**可能**绑定到有资格移动的对象上。通用引用使用右值初始化时，才将其强制转换为右值。[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)阐释了这正是`std::forward`所做的：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)           //newName是通用引用
    { name = std::forward<T>(newName); }

    …
};
```

总而言之，当把右值引用转发给其他函数时，右值引用应该被**无条件转换**为右值（通过`std::move`），因为它们**总是**绑定到右值；当转发通用引用时，通用引用应该**有条件地转换**为右值（通过`std::forward`），因为它们只是**有时**绑定到右值。

[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)解释说，可以在右值引用上使用`std::forward`表现出适当的行为，但是代码较长，容易出错，所以应该避免在右值引用上使用`std::forward`。更糟的是在通用引用上使用`std::move`，这可能会意外改变左值（比如局部变量）：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)       //通用引用可以编译，
    { name = std::move(newName); }  //但是代码太太太差了！
    …

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();        //工厂函数

Widget w;

auto n = getWidgetName();           //n是局部变量

w.setName(n);                       //把n移动进w！

…                                   //现在n的值未知
```

上面的例子，局部变量`n`被传递给`w.setName`，调用方可能认为这是对`n`的只读操作——这一点倒是可以被原谅。但是因为`setName`内部使用`std::move`无条件将传递的引用形参转换为右值，`n`的值被移动进`w.name`，调用`setName`返回时`n`最终变为未定义的值。这种行为使得调用者蒙圈了——还有可能变得狂躁。

你可能争辩说`setName`不应该将其形参声明为通用引用。此类引用不能使用`const`（见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），但是`setName`肯定不应该修改其形参。你可能会指出，如果为`const`左值和为右值分别重载`setName`可以避免整个问题，比如这样：

```cpp
class Widget {
public:
    void setName(const std::string& newName)    //用const左值设置
    { name = newName; }
    
    void setName(std::string&& newName)         //用右值设置
    { name = std::move(newName); }
    
    …
};
```

这样的话，当然可以工作，但是有缺点。首先编写和维护的代码更多（两个函数而不是单个模板）；其次，效率下降。比如，考虑如下场景：

```cpp
w.setName("Adela Novak");
```

使用通用引用的版本的`setName`，字面字符串“`Adela Novak`”可以被传递给`setName`，再传给`w`内部`std::string`的赋值运算符。`w`的`name`的数据成员通过字面字符串直接赋值，没有临时`std::string`对象被创建。但是，`setName`重载版本，会有一个临时`std::string`对象被创建，`setName`形参绑定到这个对象，然后这个临时`std::string`移动到`w`的数据成员中。一次`setName`的调用会包括`std::string`构造函数调用（创建中间对象），`std::string`赋值运算符调用（移动`newName`到`w.name`），`std::string`析构函数调用（析构中间对象）。这比调用接受`const char*`指针的`std::string`赋值运算符开销昂贵许多。增加的开销根据实现不同而不同，这些开销是否值得担心也跟应用和库的不同而有所不同，但是事实上，将通用引用模板替换成对左值引用和右值引用的一对函数重载在某些情况下会导致运行时的开销。如果把例子泛化，`Widget`数据成员是任意类型（而不是知道是个`std::string`），性能差距可能会变得更大，因为不是所有类型的移动操作都像`std::string`开销较小（参看[Item29](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item29.md)）。

但是，关于对左值和右值的重载函数最重要的问题不是源代码的数量，也不是代码的运行时性能。而是设计的可扩展性差。`Widget::setName`有一个形参，因此需要两种重载实现，但是对于有更多形参的函数，每个都可能是左值或右值，重载函数的数量几何式增长：n个参数的话，就要实现2<sup>n</sup>种重载。这还不是最坏的。有的函数——实际上是函数模板——接受**无限制**个数的参数，每个参数都可以是左值或者右值。此类函数的典型代表是`std::make_shared`，还有对于C++14的`std::make_unique`（见[Item21](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item21.md)）。查看他们的的重载声明：

```cpp
template<class T, class... Args>                //来自C++11标准
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>                //来自C++14标准
unique_ptr<T> make_unique(Args&&... args);
```

对于这种函数，对于左值和右值分别重载就不能考虑了：通用引用是仅有的实现方案。对这种函数，我向你保证，肯定使用`std::forward`传递通用引用形参给其他函数。这也是你应该做的。

好吧，通常，最终。但是不一定最开始就是如此。在某些情况，你可能需要在一个函数中多次使用绑定到右值引用或者通用引用的对象，并且确保在完成其他操作前，这个对象不会被移动。这时，你只想在最后一次使用时，使用`std::move`（对右值引用）或者`std::forward`（对通用引用）。比如：

```cpp
template<typename T>
void setSignText(T&& text)                  //text是通用引用
{
  sign.setText(text);                       //使用text但是不改变它
  
  auto now = 
      std::chrono::system_clock::now();     //获取现在的时间
  
  signHistory.add(now, 
                  std::forward<T>(text));   //有条件的转换为右值
}
```

这里，我们想要确保`text`的值不会被`sign.setText`改变，因为我们想要在`signHistory.add`中继续使用。因此`std::forward`只在最后使用。

对于`std::move`，同样的思路（即最后一次用右值引用的时候再调用`std::move`），但是需要注意，在有些稀少的情况下，你需要调用`std::move_if_noexcept`代替`std::move`。要了解何时以及为什么，参考[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)。

如果你在**按值**返回的函数中，返回值绑定到右值引用或者通用引用上，需要对返回的引用使用`std::move`或者`std::forward`。要了解原因，考虑两个矩阵相加的`operator+`函数，左侧的矩阵为右值（可以被用来保存求值之后的和）：

```cpp
Matrix                              //按值返回
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs);	        //移动lhs到返回值中
}
```

通过在`return`语句中将`lhs`转换为右值（通过`std::move`），`lhs`可以移动到返回值的内存位置。如果省略了`std::move`调用，

```cpp
Matrix                              //同之前一样
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs;                     //拷贝lhs到返回值中
}
```

`lhs`是个左值的事实，会强制编译器拷贝它到返回值的内存空间。假定`Matrix`支持移动操作，并且比拷贝操作效率更高，在`return`语句中使用`std::move`的代码效率更高。

如果`Matrix`不支持移动操作，将其转换为右值不会变差，因为右值可以直接被`Matrix`的拷贝构造函数拷贝（见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)）。如果`Matrix`随后支持了移动操作，`operator+`将在下一次编译时受益。就是这种情况，通过将`std::move`应用到按值返回的函数中要返回的右值引用上，不会损失什么（还可能获得收益）。

使用通用引用和`std::forward`的情况类似。考虑函数模板`reduceAndCopy`收到一个未规约（unreduced）对象`Fraction`，将其规约，并返回一个规约后的副本。如果原始对象是右值，可以将其移动到返回值中（避免拷贝开销），但是如果原始对象是左值，必须创建副本，因此如下代码：

```cpp
template<typename T>
Fraction                            //按值返回
reduceAndCopy(T&& frac)             //通用引用的形参
{
    frac.reduce();
    return std::forward<T>(frac);		//移动右值，或拷贝左值到返回值中
}
```

如果`std::forward`被忽略，`frac`就被无条件复制到`reduceAndCopy`的返回值内存空间。

有些开发者获取到上面的知识后，并尝试将其扩展到不适用的情况。“如果对要被拷贝到返回值的右值引用形参使用`std::move`，会把拷贝构造变为移动构造，”他们想，“我也可以对我要返回的局部对象应用同样的优化。”换句话说，他们认为有个按值返回局部对象的函数，像这样，

```cpp
Widget makeWidget()                 //makeWidget的“拷贝”版本
{
    Widget w;                       //局部对象
    …                               //配置w
    return w;                       //“拷贝”w到返回值中
}
```

他们想要“优化”代码，把“拷贝”变为移动：

```cpp
Widget makeWidget()                 //makeWidget的移动版本
{
    Widget w;
    …
    return std::move(w);            //移动w到返回值中（不要这样做！）
}
```

我的注释告诉你这种想法是有问题的，但是问题在哪？

这是错的，因为对于这种优化，标准化委员会远领先于开发者。早就为人认识到的是，`makeWidget`的“拷贝”版本可以避免复制局部变量`w`的需要，通过在分配给函数返回值的内存中构造`w`来实现。这就是所谓的**返回值优化**（*return value optimization*，RVO），这在C++标准中已经实现了。

对这种好事遣词表达是个讲究活，因为你想只在那些不影响软件外在行为的地方允许这样的**拷贝消除**（copy elision）。对标准中教条的（也可以说是有毒的）絮叨做些解释，这个特定的好事就是说，编译器可能会在按值返回的函数中消除对局部对象的拷贝（或者移动），如果满足（1）局部对象与函数返回值的类型相同；（2）局部对象就是要返回的东西。（适合的局部对象包括大多数局部变量（比如`makeWidget`里的`w`），还有作为`return`语句的一部分而创建的临时对象。函数形参不满足要求。一些人将RVO的应用区分为命名的和未命名的（即临时的）局部对象，限制了RVO术语应用到未命名对象上，并把对命名对象的应用称为**命名返回值优化**（*named return value optimization*，NRVO）。）把这些记在脑子里，再看看`makeWidget`的“拷贝”版本：

```cpp
Widget makeWidget()                 //makeWidget的“拷贝”版本
{
    Widget w;
    …
    return w;                       //“拷贝”w到返回值中
}
```

这里两个条件都满足，你一定要相信我，对于这些代码，每个合适的C++编译器都会应用RVO来避免拷贝`w`。那意味着`makeWidget`的“拷贝”版本实际上不拷贝任何东西。

移动版本的`makeWidget`行为与其名称一样（假设`Widget`有移动构造函数），将`w`的内容移动到`makeWidget`的返回值位置。但是为什么编译器不使用RVO消除这种移动，而是在分配给函数返回值的内存中再次构造`w`呢？答案很简单：它们不能。条件（2）中规定，仅当返回值为局部对象时，才进行RVO，但是`makeWidget`的移动版本不满足这条件，再次看一下返回语句：

```cpp
return std::move(w);
```

返回的已经不是局部对象`w`，而是**`w`的引用**——`std::move(w)`的结果。返回局部对象的引用不满足RVO的第二个条件，所以编译器必须移动`w`到函数返回值的位置。开发者试图对要返回的局部变量用`std::move`帮助编译器优化，反而限制了编译器的优化选项。

但是RVO就是个优化。编译器不被**要求**消除拷贝和移动操作，即使他们被允许这样做。或许你会疑惑，并担心编译器用拷贝操作惩罚你，因为它们确实可以这样。或者你可能有足够的了解，意识到有些情况很难让编译器实现RVO，比如当函数不同控制路径返回不同局部变量时。（编译器必须产生一些代码在分配的函数返回值的内存中构造适当的局部变量，但是编译器如何确定哪个变量是合适的呢？）如果这样，你可能会愿意以移动的代价来保证不会产生拷贝。那就是，极可能仍然认为应用`std::move`到一个要返回的局部对象上是合理的，只因为可以不再担心拷贝的代价。

那种情况下，应用`std::move`到一个局部对象上**仍然**是一个坏主意。C++标准关于RVO的部分表明，如果满足RVO的条件，但是编译器选择不执行拷贝消除，则返回的对象**必须被视为右值**。实际上，标准要求当RVO被允许时，或者实行拷贝消除，或者将`std::move`隐式应用于返回的局部对象。因此，在`makeWidget`的“拷贝”版本中，

```cpp
Widget makeWidget()                 //同之前一样
{
    Widget w;
    …
    return w;
}
```

编译器要不消除`w`的拷贝，要不把函数看成像下面写的一样：

```cpp
Widget makeWidget()
{
    Widget w;
    …
    return std::move(w);            //把w看成右值，因为不执行拷贝消除
}
```

这种情况与按值返回函数形参的情况很像。形参们没资格参与函数返回值的拷贝消除，但是如果作为返回值的话编译器会将其视作右值。结果就是，如果代码如下：

```cpp
Widget makeWidget(Widget w)         //传值形参，与函数返回的类型相同
{
    …
    return w;
}
```

编译器必须看成像下面这样写的代码：

```cpp
Widget makeWidget(Widget w)
{
    …
    return std::move(w);
}
```

这意味着，如果对从按值返回的函数返回来的局部对象使用`std::move`，你并不能帮助编译器（如果不能实行拷贝消除的话，他们必须把局部对象看做右值），而是阻碍其执行优化选项（通过阻止RVO）。在某些情况下，将`std::move`应用于局部变量可能是一件合理的事（即，你把一个变量传给函数，并且知道不会再用这个变量），但是满足RVO的`return`语句或者返回一个传值形参并不在此列。

**请记住：**

- 最后一次使用时，在右值引用上使用`std::move`，在通用引用上使用`std::forward`。
- 对按值返回的函数要返回的右值引用和通用引用，执行相同的操作。
- 如果局部对象可以被返回值优化消除，就绝不使用`std::move`或者`std::forward`。
