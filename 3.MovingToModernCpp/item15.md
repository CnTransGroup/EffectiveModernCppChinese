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

