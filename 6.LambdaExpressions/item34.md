## 条款三十四：考虑*lambda*而非`std::bind`

**Item 34: Prefer lambdas to `std::bind`**

C++11中的`std::bind`是C++98的`std::bind1st`和`std::bind2nd`的后续，但在2005年已经非正式成为了标准库的一部分。那时标准化委员采用了TR1的文档，其中包含了`bind`的规范。（在TR1中，`bind`位于不同的命名空间，因此它是`std::tr1::bind`，而不是`std::bind`，接口细节也有所不同）。这段历史意味着一些程序员有十年及以上的`std::bind`使用经验。如果你是其中之一，可能会不愿意放弃一个对你有用的工具。这是可以理解的，但是在这种情况下，改变是更好的，因为在C++11中，*lambda*几乎总是比`std::bind`更好的选择。 从C++14开始，*lambda*的作用不仅强大，而且是完全值得使用的。

这个条款假设你熟悉`std::bind`。 如果不是这样，你将需要获得基本的了解，然后再继续。 无论如何，这样的理解都是值得的，因为你永远不知道何时会在阅读或维护的代码库中遇到`std::bind`。

与[Item32](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item32.md)中一样，我们将从`std::bind`返回的函数对象称为**bind对象**（*bind objects*）。​

优先*lambda*而不是`std::bind`的最重要原因是*lambda*更易读。 例如，假设我们有一个设置警报器的函数：

```c++
//一个时间点的类型定义（语法见条款9）
using Time = std::chrono::steady_clock::time_point;

//“enum class”见条款10
enum class Sound { Beep, Siren, Whistle };

//时间段的类型定义
using Duration = std::chrono::steady_clock::duration;

//在时间t，使用s声音响铃时长d
void setAlarm(Time t, Sound s, Duration d);
```

进一步假设，在程序的某个时刻，我们已经确定需要设置一个小时后响30秒的警报器。 但是，具体声音仍未确定。我们可以编写一个*lambda*来修改`setAlarm`的界面，以便仅需要指定声音：

```c++
//setSoundL（“L”指代“lambda”）是个函数对象，允许指定一小时后响30秒的警报器的声音
auto setSoundL =
    [](Sound s) 
    {
        //使std::chrono部件在不指定限定的情况下可用
        using namespace std::chrono;

        setAlarm(steady_clock::now() + hours(1),    //一小时后响30秒的闹钟
                 s,
                 seconds(30));
    };
```

我们在*lambda*中高亮了对`setAlarm`的调用。这看来起是一个很正常的函数调用，即使是几乎没有*lambda*经验的读者也可以看到：传递给*lambda*的形参`s`又作为实参被传递给了`setAlarm`。

我们通过使用标准后缀如秒（`s`），毫秒（`ms`）和小时（`h`）等简化在C++14中的代码，其中标准后缀基于C++11对用户自定义常量的支持。这些后缀在`std::literals`命名空间中实现，因此上述代码可以按照以下方式重写：

```c++
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;
        using namespace std::literals;      //对于C++14后缀

        setAlarm(steady_clock::now() + 1h,	//C++14写法，但是含义同上
                 s,
                 30s);
    };
```

下面是我们第一次编写对应的`std::bind`调用。这里存在一个我们后续会修复的错误，但正确的代码会更加复杂，即使是此简化版本也会凸显一些重要问题：

```c++
using namespace std::chrono;                //同上
using namespace std::literals;
using namespace std::placeholders;          //“_1”使用需要

auto setSoundB =                            //“B”代表“bind”
    std::bind(setAlarm,
              steady_clock::now() + 1h,     //不正确！见下
              _1,
              30s);
```

我想像在之前的*lambda*中一样高亮对`setAlarm`的调用，但是没法这么做。这段代码的读者只需知道，调用`setSoundB`会使用在对`std::bind`的调用中所指定的时间和持续时间来调用`setAlarm`。对于门外汉来说，占位符“`_1`”完全是一个魔法，但即使是知情的读者也必须从思维上将占位符中的数字映射到其在`std::bind`形参列表中的位置，以便明白调用`setSoundB`时的第一个实参会被传递进`setAlarm`，作为调用`setAlarm`的第二个实参。在对`std::bind`的调用中未标识此实参的类型，因此读者必须查阅`setAlarm`声明以确定将哪种实参传递给`setSoundB`。

但正如我所说，代码并不完全正确。在*lambda*中，表达式`steady_clock::now() + 1h`显然是`setAlarm`的实参。调用`setAlarm`时将对其进行计算。可以理解：我们希望在调用`setAlarm`后一小时响铃。但是，在`std::bind`调用中，将`steady_clock::now() + 1h`作为实参传递给了`std::bind`，而不是`setAlarm`。这意味着将在调用`std::bind`时对表达式进行求值，并且该表达式产生的时间将存储在产生的bind对象中。结果，警报器将被设置为在**调用`std::bind`后一小时**发出声音，而不是在调用`setAlarm`一小时后发出。

要解决此问题，需要告诉`std::bind`推迟对表达式的求值，直到调用`setAlarm`为止，而这样做的方法是将对`std::bind`的第二个调用嵌套在第一个调用中：

```c++
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<>(), steady_clock::now(), 1h),
              _1,
              30s);
```

如果你熟悉C++98的`std::plus`模板，你可能会惊讶地发现在此代码中，尖括号之间未指定任何类型，即该代码包含“`std::plus<>`”，而不是“`std::plus<type>`”。 在C ++14中，通常可以省略标准运算符模板的模板类型实参，因此无需在此处提供。 C++11没有提供此类功能，因此等效于*lambda*的C++11 `std::bind`为：

```c++
using namespace std::chrono;                //同上
using namespace std::placeholders;
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(),
                        steady_clock::now(),
                        hours(1)),
              _1,
              seconds(30));
```

如果此时*lambda*看起来还没有吸引力，那么应该检查一下视力了。

当`setAlarm`重载时，会出现一个新问题。 假设有一个重载函数，其中第四个形参了音量：

```c++
enum class Volume { Normal, Loud, LoudPlusPlus };

void setAlarm(Time t, Sound s, Duration d, Volume v);
```

*lambda*能继续像以前一样使用，因为根据重载规则选择了`setAlarm`的三实参版本：

```c++
auto setSoundL =                            //和之前一样
    [](Sound s)
    {
        using namespace std::chrono;
        setAlarm(steady_clock::now() + 1h,  //可以，调用三实参版本的setAlarm
                 s,
                 30s);
    };
```

然而，`std::bind`的调用将会编译失败：

```c++
auto setSoundB =                            //错误！哪个setAlarm？
    std::bind(setAlarm,
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```

这里的问题是，编译器无法确定应将两个`setAlarm`函数中的哪一个传递给`std::bind`。 它们仅有的是一个函数名称，而这个单一个函数名称是有歧义的。

要使对`std::bind`的调用能编译，必须将`setAlarm`强制转换为适当的函数指针类型：

```c++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundB =                                            //现在可以了
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h), 
              _1,
              30s);
```

但这在*lambda*和`std::bind`的使用上带来了另一个区别。 在`setSoundL`的函数调用操作符（即*lambda*的闭包类对应的函数调用操作符）内部，对`setAlarm`的调用是正常的函数调用，编译器可以按常规方式进行内联：

```c++
setSoundL(Sound::Siren);    //setAlarm函数体在这可以很好地内联
```

但是，对`std::bind`的调用是将函数指针传递给`setAlarm`，这意味着在`setSoundB`的函数调用操作符（即绑定对象的函数调用操作符）内部，对`setAlarm`的调用是通过一个函数指针。 编译器不太可能通过函数指针内联函数，这意味着与通过`setSoundL`进行调用相比，通过`setSoundB`对`setAlarm的`调用，其函数不大可能被内联：

```c++
setSoundB(Sound::Siren); 	//setAlarm函数体在这不太可能内联
```

因此，使用*lambda*可能会比使用`std::bind`能生成更快的代码。

`setAlarm`示例仅涉及一个简单的函数调用。如果你想做更复杂的事情，使用*lambda*会更有利。 例如，考虑以下C++14的*lambda*使用，它返回其实参是否在最小值（`lowVal`）和最大值（`highVal`）之间的结果，其中`lowVal`和`highVal`是局部变量：

```c++
auto betweenL =
    [lowVal, highVal]
    (const auto& val)                           //C++14
    { return lowVal <= val && val <= highVal; };
```

使用`std::bind`可以表达相同的内容，但是该构造是一个通过晦涩难懂的代码来保证工作安全性的示例：

```c++
using namespace std::placeholders;              //同上
auto betweenB =
    std::bind(std::logical_and<>(),             //C++14
              std::bind(std::less_equal<>(), lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal));
```

在C++11中，我们必须指定要比较的类型，然后`std::bind`调用将如下所示：

```c++
auto betweenB =
    std::bind(std::logical_and<bool>(),         //C++14版本
              std::bind(std::less_equal<int>(), lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal));
```

当然，在C++11中，*lambda*也不能采用`auto`参数，因此它也必须指定一个类型：

```c++
auto betweenL =                                 //C++11版本
    [lowVal, highVal]
    (int val)
    { return lowVal <= val && val <= highVal; };
```

无论哪种方式，我希望我们都能同意，*lambda*版本不仅更短，而且更易于理解和维护。

之前我就说过，对于那些没有`std::bind`使用经验的人，其占位符（例如`_1`，`_2`等）都是魔法。 但是这不仅仅在于占位符的行为是不透明的。 假设我们有一个函数可以创建`Widget`的压缩副本，

```c++
enum class CompLevel { Low, Normal, High }; //压缩等级

Widget compress(const Widget& w,            //制作w的压缩副本
                CompLevel lev);
```

并且我们想创建一个函数对象，该函数对象允许我们指定`Widget w`的压缩级别。这种使用`std::bind`的话将创建一个这样的对象：

```c++
Widget w;
using namespace std::placeholders;
auto compressRateB = std::bind(compress, w, _1);
```

现在，当我们将`w`传递给`std::bind`时，必须将其存储起来，以便以后进行压缩。它存储在对象`compressRateB`中，但是它是如何被存储的呢——是通过值还是引用？之所以会有所不同，是因为如果在对`std::bind`的调用与对`compressRateB`的调用之间修改了`w`，则按引用捕获的`w`将反映这个更改，而按值捕获则不会。

答案是它是按值捕获的（`std::bind`总是拷贝它的实参，但是调用者可以使用引用来存储实参，这要通过应用`std::ref`到实参上实现。`auto compressRateB = std::bind(compress, std::ref(w), _1);`的结果就是`compressRateB`行为像是持有`w`的引用而非副本。），但唯一知道的方法是记住`std::bind`的工作方式；在对`std::bind`的调用中没有任何迹象。然而在*lambda*方法中，其中`w`是通过值还是通过引用捕获是显式的：

```c++
auto compressRateL =                //w是按值捕获，lev是按值传递
    [w](CompLevel lev)
    { return compress(w, lev); };
```

同样明确的是形参是如何传递给*lambda*的。 在这里，很明显形参`lev`是通过值传递的。 因此：

```c++
compressRateL(CompLevel::High);     //实参按值传递
```

但是在对由`std::bind`生成的对象调用中，实参如何传递？

```c++
compressRateB(CompLevel::High);     //实参如何传递？
```

同样，唯一的方法是记住`std::bind`的工作方式。（答案是传递给bind对象的所有实参都是通过引用传递的，因为此类对象的函数调用运算符使用完美转发。）

与*lambda*相比，使用`std::bind`进行编码的代码可读性较低，表达能力较低，并且效率可能较低。 在C++14中，没有`std::bind`的合理用例。 但是，在C ++11中，可以在两个受约束的情况下证明使用`std::bind`是合理的：

* **移动捕获。 **C++11的*lambda*不提供移动捕获，但是可以通过结合*lambda*和`std::bind`来模拟。 有关详细信息，请参阅[Item32](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item32.md)，该条款还解释了在C++14中，*lambda*对初始化捕获的支持消除了这个模拟的需求。
* **多态函数对象。 **因为绑定对象上的函数调用运算符使用完全转发，所以它可以接受任何类型的参数（以[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)中描述的完全转发的限制为例子）。当你要绑定带有模板化函数调用运算符的对象时，此功能很有用。 例如这个类，

```c++
class PolyWidget {
public:
    template<typename T>
    void operator()(const T& param);
    …
};
```

`std::bind`可以如下绑定一个`PolyWidget`对象：

```c++
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```

`boundPW`可以接受任意类型的对象了：

```c++
boundPW(1930);              //传int给PolyWidget::operator()
boundPW(nullptr);           //传nullptr给PolyWidget::operator()
boundPW("Rosebud"); 		//传字面值给PolyWidget::operator()
```

这一点无法使用C++11的*lambda*做到。 但是，在C++14中，可以通过带有`auto`参数的*lambda*轻松实现：

```c++
auto boundPW = [pw](const auto& param)  //C++14 
               { pw(param); };
```

当然，这些是特殊情况，并且是暂时的特殊情况，因为支持C++14 *lambda*的编译器越来越普遍了。

当`bind`在2005年被非正式地添加到C++中时，与1998年的前身相比有了很大的改进。 在C++11中增加了*lambda*支持，这使得`std::bind`几乎已经过时了，从C++14开始，更是没有很好的用例了。

**请记住：**

+ 与使用`std::bind`相比，*lambda*更易读，更具表达力并且可能更高效。
+ 只有在C++11中，`std::bind`可能对实现移动捕获或绑定带有模板化函数调用运算符的对象时会很有用。
