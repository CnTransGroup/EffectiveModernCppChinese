## Item 15:尽可能的使用constexpr
条款 15:尽可能的使用constexpr

如果要颁一个C++11中最令人困惑的词的奖，**constexpr**可能会赢得这个奖。当用于对象上面，它本质上就是**const**的加强形式，但是用于函数上，它就有不同的意思了。斩断困惑是值得的做的，因为当**constexpr**符合你想表达的东西，你绝对会想用它。

从概念上来说，**constexpr**指示一个值不仅仅是常量，它还是编译期可知的。这个概念只
是故事的一部分，因为当**constexpr**被用于函数的时候，事情就有一些细微差别了。
为了避免我毁了结局带来的surprise，我现在只想说，你不能假设**constexpr**函数是**const**，也不能保证它们的（译注：返回）值是在编译期可知的。最有意思的是，这些是特性。关于**constexpr**函数返回的结果不需要是**const**，也不需要编译期可知这一点是良好的行为。

不过我们还是先从**constexpr**对象开始说起。这些对象，实际上，和**const**一样，它们是编译期可知的。（专业来说，它们的值在翻译期（translation）决议，所谓翻译不仅仅包含是编译（compilation）也包含链接（linking），除非你准备写C++的编译器和链接器，否则这些对你不会造成影响，所以你编程时无需担心，把这些**constexpr**对象值看做编译期决议也无妨的。）

编译期可知的值是保密的。它们可能被放到只读存储空间中，尤其是对于那些嵌入式系统的开发者，这个特性是相当重要的。更广泛的应用是常量整数在编译期可知，并且能被用于C++要求出现整数常量表达式（ __integral constant expression__ ）的上下文。这类上下文包括数组大小，整数模板参数（包括`std::array`对象的长度），枚举量，对齐修饰符（译注：[`alignas(val)`](https://en.cppreference.com/w/cpp/language/alignas)），等等。如果你想在这些上下文中使用变量，你一定会希望将它们声明为**constexpr**，因为编译器会确保它们是编译期可知值：
```cpp
int sz;                             // 非constexpr变量
…
constexpr auto arraySize1 = sz;     // 错误! sz的值在
                                    // 编译期不可知
std::array<int, sz> data1;          // 错误!一样的问题
constexpr auto arraySize2 = 10;     // 没问题，10是编译
                                    // 期可知常量
std::array<int, arraySize2> data2;  // 没问题, arraySize2是constexpr
 ```
 注意**const**不提供**constexpr**所能保证之事，因为**const**对象不需要在编译期初始化它的值。
 ```cpp
 int sz;                            // 和之前一样
 const auto arraySize = sz;         // 没问题，arraySize是sz的常量复制
 std::array<int, arraySize> data;   // 错误，arraySize值在编译期不可知
 ```
 简而言之，所有**constexpr**对象都是**const**，但不是所有**const**对象都是**constexpr**。如果你想编译器保证一个变量有一个可以放到那些需要编译期常量的上下文的值，你需要的工具是**constexpr**而不是**const**。

如果使用场景涉及函数，那 **constexpr**就更有趣了。如果实参是编译期常量，它们将产出编译期值；如果是运行时值，它们就将产出运行时值。这听起来就像你不知道它们要做什么一样，那么想是错误的，请这么看：
+ **constexpr**函数可以用于需求编译期常量的上下文。如果你传给**constexpr**函数的实参在编译期可知，那么结果将在编译期计算。如果实参的值在编译期不知道，你的代码就会被拒绝。
+ 当一个**constexpr**函数被一个或者多个编译期不可知值调用时，它就像普通函数一样，运行时计算它的结果。这意味着你不需要两个函数，一个用于编译期计算，一个用于运行时计算。**constexpr**全做了。

假设我们需要一个数据结构来存储一个实验的结果，而这个实验可能以各种方式进行。实验期间风扇转速，温度等等都可能导致亮度值改变，亮度值可以是高，低，或者无。如果有n个实验相关的环境条件。它们每一个都有三个状态，最终可以得到的组合有`3^n`个。储存所有实验结果的所有组合需要这个数据结构足够大。假设每个结果都是**int**并且**n**是编译期已知的（或者可以被计算出的），一个`std::array`是一个合理的选择。我们需要一个方法在编译期计算`3^n`。C++标准库提供了`std::pow`，它的数学意义正是我们所需要的，但是，对我们来说，这里还有两个问题。第一，`std::pow`是为浮点类型设计的 我们需要整型结果。第二，`std::pow`不是**constexpr**（即，使用编译期可知值调用得到的可能不是编译期可知的结果），所以我们不能用它作为`std::array`的大小。

幸运的是，我们可以应需写个`pow`。我将展示怎么快速完成它，不过现在让我们先看看它应该怎么被声明和使用：
```cpp
constexpr                               // pow是constexpr函数
int pow(int base, int exp) noexcept     // 绝不抛异常
{
 …                                      // 实现在这里
}
constexpr auto numConds = 5;            //条件个数
std::array<int, pow(3, numConds)> results; // 结果有3^numConds个元素
```
回忆下`pow`前面的**constexpr**没有告诉我们`pow`返回一个**const**值，它只说了如果**base**和**exp**是编译期常量，`pow`返回值可能是编译期常量。如果**base** 和/或 **exp**不是编译期常量，`pow`结果将会在运行时计算。这意味着**pow**不知可以用于像`std::array`的大小这种需要编译期常量的地方，它也可以用于运行时环境：
```cpp
auto base = readFromDB("base");     // 运行时获取三个值
auto exp = readFromDB("exponent"); 
auto baseToExp = pow(base, exp);    // 运行时调用pow
```
因为**constexpr**函数必须能在编译期值调用的时候返回编译器结果，就必须对它的实现施加一些限制