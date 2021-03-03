## 条款三十三：对`auto&&`形参使用`decltype`以`std::forward`它们

**Item 33: Use `decltype` on `auto&&` parameters to `std::forward` them**

**泛型*lambda***（*generic lambdas*）是C++14中最值得期待的特性之一——因为在*lambda*的形参中可以使用`auto`关键字。这个特性的实现是非常直截了当的：即在闭包类中的`operator()`函数是一个函数模版。例如存在这么一个*lambda*，

```c++
auto f = [](auto x){ return func(normalize(x)); };
```

对应的闭包类中的函数调用操作符看来就变成这样：

```c++
class SomeCompilerGeneratedClassName {
public:
    template<typename T>                //auto返回类型见条款3
    auto operator()(T x) const
    { return func(normalize(x)); }
    …                                   //其他闭包类功能
};
```

在这个样例中，*lambda*对变量`x`做的唯一一件事就是把它转发给函数`normalize`。如果函数`normalize`对待左值右值的方式不一样，这个*lambda*的实现方式就不大合适了，因为即使传递到*lambda*的实参是一个右值，*lambda*传递进`normalize`的总是一个左值（形参`x`）。

实现这个*lambda*的正确方式是把`x`完美转发给函数`normalize`。这样做需要对代码做两处修改。首先，`x`需要改成通用引用（见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），其次，需要使用`std::forward`将`x`转发到函数`normalize`（见[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)）。理论上，这都是小改动：

```c++
auto f = [](auto&& x)
         { return func(normalize(std::forward<???>(x))); };
```

在理论和实际之间存在一个问题：你传递给`std::forward`的参数是什么类型，即确定我在上面写的`???`该是什么。

一般来说，当你在使用完美转发时，你是在一个接受类型参数为`T`的模版函数里，所以你可以写`std::forward<T>`。但在泛型*lambda*中，没有可用的类型参数`T`。在*lambda*生成的闭包里，模版化的`operator()`函数中的确有一个`T`，但在*lambda*里却无法直接使用它，所以也没什么用。

[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)解释过如果一个左值实参被传给通用引用的形参，那么形参类型会变成左值引用。传递的是右值，形参就会变成右值引用。这意味着在这个*lambda*中，可以通过检查形参`x`的类型来确定传递进来的实参是一个左值还是右值，`decltype`就可以实现这样的效果（见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）。传递给*lambda*的是一个左值，`decltype(x)`就能产生一个左值引用；如果传递的是一个右值，`decltype(x)`就会产生右值引用。

[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)也解释过在调用`std::forward`时，惯例决定了类型实参是左值引用时来表明要传进左值，类型实参是非引用就表明要传进右值。在前面的*lambda*中，如果`x`绑定的是一个左值，`decltype(x)`就能产生一个左值引用。这符合惯例。然而如果`x`绑定的是一个右值，`decltype(x)`就会产生右值引用，而不是常规的非引用。

再看一下[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)中关于`std::forward`的C++14实现：

```c++
template<typename T>                        //在std命名空间
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

如果用户想要完美转发一个`Widget`类型的右值时，它会使用`Widget`类型（即非引用类型）来实例化`std::forward`，然后产生以下的函数：

```c++
Widget&& forward(Widget& param)             //当T是Widget时的std::forward实例
{
    return static_cast<Widget&&>(param);
}
```

思考一下如果用户代码想要完美转发一个`Widget`类型的右值，但没有遵守规则将`T`指定为非引用类型，而是将`T`指定为右值引用，这会发生什么。也就是，思考将`T`换成`Widget&&`会如何。在`std::forward`实例化、应用了`std::remove_reference_t`后，引用折叠之前，`std::forward`看起来像这样：

```c++
Widget&& && forward(Widget& param)          //当T是Widget&&时的std::forward实例
{                                           //（引用折叠之前）
    return static_cast<Widget&& &&>(param);
}
```

应用了引用折叠之后（右值引用的右值引用变成单个右值引用），代码会变成：

```c++
Widget&& forward(Widget& param)             //当T是Widget&&时的std::forward实例
{                                           //（引用折叠之后）
    return static_cast<Widget&&>(param);
}
```

对比这个实例和用`Widget`设置`T`去实例化产生的结果，它们完全相同。表明用右值引用类型和用非引用类型去初始化`std::forward`产生的相同的结果。

那是一个很好的消息，因为当传递给*lambda*形参`x`的是一个右值实参时，`decltype(x)`可以产生一个右值引用。前面已经确认过，把一个左值传给*lambda*时，`decltype(x)`会产生一个可以传给`std::forward`的常规类型。而现在也验证了对于右值，把`decltype(x)`产生的类型传递给`std::forward`是非传统的，不过它产生的实例化结果与传统类型相同。所以无论是左值还是右值，把`decltype(x)`传递给`std::forward`都能得到我们想要的结果，因此*lambda*的完美转发可以写成：

```c++
auto f =
    [](auto&& param)
    {
        return
            func(normalize(std::forward<decltype(param)>(param)));
    };
```

再加上6个点，就可以让我们的*lambda*完美转发接受多个参数了，因为C++14中的*lambda*也可以是可变参数的：

```c++
auto f =
    [](auto&&... params)
    {
        return
            func(normalize(std::forward<decltype(params)>(params)...));
    };
```

**请记住：**

+ 对`auto&&`形参使用`decltype`以`std::forward`它们。

