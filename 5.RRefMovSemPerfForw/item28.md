## 条款二十八：理解引用折叠

**Item 28: Understand reference collapsing**

[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)中指出，当实参传递给模板函数时，被推导的模板形参`T`根据实参是左值还是右值来编码。但是那条款并没有提到只有当实参被用来实例化通用引用形参时，上述推导才会发生，但是有充分的理由忽略这一点：因为通用引用是[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中才提到。回过头来看，对通用引用和左值/右值编码的观察意味着对于这个模板，

```cpp
template<typename T>
void func(T&& param);
```

不管传给param的实参是左值还是右值，模板形参`T`都会编码。

编码机制是简单的。当左值实参被传入时，`T`被推导为左值引用。当右值被传入时，`T`被推导为非引用。（请注意不对称性：左值被编码为左值引用，右值被编码为**非引用**。）因此：

```cpp
Widget widgetFactory();     //返回右值的函数
Widget w;                   //一个变量（左值）
func(w);                    //用左值调用func；T被推导为Widget&
func(widgetFactory());      //用又值调用func；T被推导为Widget
```

上面的两种`func`调用中，`Widget`被传入，因为一个是左值，一个是右值，模板形参`T`被推导为不同的类型。正如我们很快看到的，这决定了通用引用成为左值还是右值，也是`std::forward`的工作基础。

在我们更加深入`std::forward`和通用引用之前，必须明确在C++中引用的引用是非法的。不知道你是否尝试过下面的写法，编译器会报错：

```cpp
int x;
…
auto& & rx = x;             //错误！不能声明引用的引用
```

考虑下，如果一个左值传给接受通用引用的模板函数会发生什么：

```cpp
template<typename T>
void func(T&& param);       //同之前一样

func(w);                    //用左值调用func；T被推导为Widget&
```

如果我们用`T`推导出来的类型（即`Widget&`）初始化模板，会得到：

```cpp
void func(Widget& && param);
```

引用的引用！但是编译器没有报错。我们从[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中了解到因为通用引用`param`被传入一个左值，所以`param`的类型应该为左值引用，但是编译器如何把`T`推导的类型带入模板变成如下的结果，也就是最终的函数签名？

```cpp
void func(Widget& param);
```

答案是**引用折叠**（*reference collapsing*）。是的，禁止**你**声明引用的引用，但是**编译器**会在特定的上下文中产生这些，模板实例化就是其中一种情况。当编译器生成引用的引用时，引用折叠指导下一步发生什么。

存在两种类型的引用（左值和右值），所以有四种可能的引用组合（左值的左值，左值的右值，右值的右值，右值的左值）。如果一个上下文中允许引用的引用存在（比如，模板的实例化），引用根据规则**折叠**为单个引用：

> 如果任一引用为左值引用，则结果为左值引用。否则（即，如果引用都是右值引用），结果为右值引用。

在我们上面的例子中，将推导类型`Widget&`替换进模板`func`会产生对左值引用的右值引用，然后引用折叠规则告诉我们结果就是左值引用。

引用折叠是`std::forward`工作的一种关键机制。就像[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)中解释的一样，`std::forward`应用在通用引用参数上，所以经常能看到这样使用：

```cpp
template<typename T>
void f(T&& fParam)
{
    …                                   //做些工作
    someFunc(std::forward<T>(fParam));  //转发fParam到someFunc
}
```

因为`fParam`是通用引用，我们知道类型参数`T`的类型根据`f`被传入实参（即用来实例化`fParam`的表达式）是左值还是右值来编码。`std::forward`的作用是当且仅当传给`f`的实参为右值时，即`T`为非引用类型，才将`fParam`（左值）转化为一个右值。

`std::forward`可以这样实现：

```cpp
template<typename T>                                //在std命名空间
T&& forward(typename
                remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}
```

这不是标准库版本的实现（忽略了一些接口描述），但是为了理解`std::forward`的行为，这些差异无关紧要。

假设传入到`f`的实参是`Widget`的左值类型。`T`被推导为`Widget&`，然后调用`std::forward`将实例化为`std::forward<Widget&>`。`Widget&`带入到上面的`std::forward`的实现中：

```cpp
Widget& && forward(typename 
                       remove_reference<Widget&>::type& param)
{ return static_cast<Widget& &&>(param); }
```

`std::remove_reference<Widget&>::type`这个*type trait*产生`Widget`（查看[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)），所以`std::forward`成为：

```cpp
Widget& && forward(Widget& param)
{ return static_cast<Widget& &&>(param); }
```

根据引用折叠规则，返回值和强制转换可以化简，最终版本的`std::forward`调用就是：

```cpp
Widget& forward(Widget& param)
{ return static_cast<Widget&>(param); }
```

正如你所看到的，当左值实参被传入到函数模板`f`时，`std::forward`被实例化为接受和返回左值引用。内部的转换不做任何事，因为`param`的类型已经是`Widget&`，所以转换没有影响。左值实参传入`std::forward`会返回左值引用。通过定义，左值引用就是左值，因此将左值传递给`std::forward`会返回左值，就像期待的那样。

现在假设一下，传递给`f`的实参是一个`Widget`的右值。在这个例子中，`f`的类型参数`T`的推导类型就是`Widget`。`f`内部的`std::forward`调用因此为`std::forward<Widget>`，`std::forward`实现中把`T`换为`Widget`得到：

```cpp
Widget&& forward(typename
                     remove_reference<Widget>::type& param)
{ return static_cast<Widget&&>(param); }
```

将`std::remove_reference`引用到非引用类型`Widget`上还是相同的类型（`Widget`），所以`std::forward`变成：

```cpp
Widget&& forward(Widget& param)
{ return static_cast<Widget&&>(param); }
```

这里没有引用的引用，所以不需要引用折叠，这就是`std::forward`的最终实例化版本。

从函数返回的右值引用被定义为右值，因此在这种情况下，`std::forward`会将`f`的形参`fParam`（左值）转换为右值。最终结果是，传递给`f`的右值参数将作为右值转发给`someFunc`，正是想要的结果。

在C++14中，`std::remove_reference_t`的存在使得实现变得更简洁：

```cpp
template<typename T>                        //C++14；仍然在std命名空间
T&& forward(remove_reference_t<T>& param)
{
  return static_cast<T&&>(param);
}
```

引用折叠发生在四种情况下。第一，也是最常见的就是模板实例化。第二，是`auto`变量的类型生成，具体细节类似于模板，因为`auto`变量的类型推导基本与模板类型推导雷同（参见[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)）。考虑本条款前面的例子：

```cpp
Widget widgetFactory();     //返回右值的函数
Widget w;                   //一个变量（左值）
func(w);                    //用左值调用func；T被推导为Widget&
func(widgetFactory());      //用又值调用func；T被推导为Widget
```

在auto的写法中，规则是类似的。声明

```cpp
auto&& w1 = w;
```

用一个左值初始化`w1`，因此为`auto`推导出类型`Widget&`。把`Widget&`代回`w1`声明中的`auto`里，产生了引用的引用，
```cpp
Widget& && w1 = w;
```
应用引用折叠规则，就是
```cpp
Widget& w1 = w
```
结果就是`w1`是一个左值引用。

另一方面，这个声明，
```cpp
auto&& w2 = widgetFactory();
```
使用右值初始化`w2`，为`auto`推导出非引用类型`Widget`。把`Widget`代入`auto`得到：
```cpp
Widget&& w2 = widgetFactory()
```
没有引用的引用，这就是最终结果，`w2`是个右值引用。

现在我们真正理解了[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中引入的通用引用。通用引用不是一种新的引用，它实际上是满足以下两个条件下的右值引用：

- **类型推导区分左值和右值**。`T`类型的左值被推导为`T&`类型，`T`类型的右值被推导为`T`。
- **发生引用折叠**。

通用引用的概念是有用的，因为它使你不必一定意识到引用折叠的存在，从直觉上推导左值和右值的不同类型，在凭直觉把推导的类型代入到它们出现的上下文中之后应用引用折叠规则。

我说了有四种情况会发生引用折叠，但是只讨论了两种：模板实例化和`auto`的类型生成。第三种情况是`typedef`和别名声明的产生和使用中（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）。如果，在创建或者评估`typedef`过程中出现了引用的引用，则引用折叠就会起作用。举例子来说，假设我们有一个`Widget`的类模板，该模板具有右值引用类型的嵌入式`typedef`：

```cpp
template<typename T>
class Widget {
public:
    typedef T&& RvalueRefToT;
    …
};
```

假设我们使用左值引用实例化`Widget`：

```cpp
Widget<int&> w;
```

`Widget`模板中把`T`替换为`int&`得到：

```cpp
typedef int& && RvalueRefToT;
```

引用折叠就会发挥作用：

```cpp
typedef int& RvalueRefToT;
```

这清楚表明我们为`typedef`选择的名字可能不是我们希望的那样：当使用左值引用类型实例化`Widget`时，`RvalueRefToT`是**左值引用**的`typedef`。

最后一种引用折叠发生的情况是，`decltype`使用的情况。如果在分析`decltype`期间，出现了引用的引用，引用折叠规则就会起作用（关于`decltype`，参见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）

**请记住：**

- 引用折叠发生在四种情况下：模板实例化，`auto`类型推导，`typedef`与别名声明的创建和使用，`decltype`。
- 当编译器在引用折叠环境中生成了引用的引用时，结果就是单个引用。有左值引用折叠结果就是左值引用，否则就是右值引用。
- 通用引用就是在特定上下文的右值引用，上下文是通过类型推导区分左值还是右值，并且发生引用折叠的那些地方。
