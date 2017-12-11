## Item 10:优先考虑限域枚举而非未限域枚举
条款10:优先考虑限域枚举而非未限域枚举

通常来说，在花括号中声明一个名字会限制它的作用域在花括号之内。但这对于C++98风格的enum中声明的枚举名是不成立的。这些在enum作用域中声明的枚举名所在的作用域也包括enum本身，也就是说这些枚举名和enum所在的作用域中声明的相同名字没有什么不同
```cpp
enum Color { black, white, red };   // black, white, red 和
                                    // Color一样都在相同作用域
auto white = false;                 // 错误! white早已在这个作用
                                    // 域中存在
```
事实上这些枚举名泄漏进和它们所被定义的enum域一样的作用域有一个官方的术语：未限域(unscoped)枚举。在C++11中它们有一个相似物，限域枚举，它不会导致枚举名泄漏：
```cpp
enum class Color { black, white, red }; // black, white, red
                                        // 限制在Color域内
auto white = false;                     // 没问题，同样域内没有这个名字

Color c = white;                        //错误，这个域中没有white

Color c = Color::white;                 // 没问题
auto c = Color::white;                  // 也没问题（也符合条款5的建议）
```
因为限域枚举是通过**enum class**声明，所以它们有时候也被称为枚举类(enum classes)。

使用限域枚举减少命名空间污染是一个足够合理使用它而不是它的同胞未限域枚举的理由，其实限域枚举还有第二个吸引人的优点：在它的作用域中的枚举名是强类型。未限域枚举中的枚举名会隐式的转换为整型（现在，也可以转换为浮点类型）。因此下面这种歪曲语义的做法也是完全有效的：
```cpp
enum Color { black, white, red };       // 未限域枚举
std::vector<std::size_t>                // func返回x的质因子
primeFactors(std::size_t x);            
Color c = red;
…
if (c < 14.5) {                         // Color与double比较 (!)
    auto factors =                      // 计算一个Color的质因子(!)
    primeFactors(c);                    
…
}
```
在**enum**后面写一个**class**就可以将非限域枚举转换为限域枚举，接下来就是完全不同的故事展开了。
现在不存在任何隐式的转换可以将限域枚举中的枚举量转化为任何其他类型。
```cpp
enum class Color { black, white, red }; // Color现在是限域枚举
Color c = Color::red;                   // 和之前一样，只是
…                                       // 多了一个域修饰符
if (c < 14.5) {                         // 错误！不能比较
                                        // Color和double
    auto factors =                      // 错误! 不能向参数为std::size_t的函数
        primeFactors(c);                // 传递Color参数
    …
}
```
如果你真的很想执行Color到其他类型的转换，和平常一样，使用正确的类型转换运算符扭曲类型系统：
```cpp
if (static_cast<double>(c) < 14.5) { // 奇怪的代码，但是
                                     // 有效
    auto factors = // suspect, but
        primeFactors(static_cast<std::size_t>(c)); // 能通过编译
    …
}
```
看来比起非限域枚举而言限域枚举有第三个好处，因为限域枚举可以前声明。比如，它们可以不指定枚举量直接前向声明：
```cpp
enum Color;         // 错误！
enum class Color;   // 没问题
```
这是一个误导。在C++11中，非限域枚举也可以被前向声明，但是只有在做一些其他工作后才能实现。这些工作来源于一个事实：
在C++中所有的枚举都有一个由编译器决定的基础整型类型。对于非限域枚举比如Color，
```cpp
enum Color { black, white, red };
```
编译器可能选择**char**作为基础类型，因为这里只需要表示三个值。然而，有些枚举中的枚举值范围可能会大些，比如：
```cpp
enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
                };
```
这里值的范围从**0**到**0xFFFFFFFF**。除了在不寻常的机器上（比如一个**char**至少有32bits的那种），编译器都会选择一个比**char**大的整型类型来表示**Status**。

为了高效使用内存，编译器通常在确保能包含所有枚举值的前提下为枚举选择一个最小的基础类型。在一些情况下，编译器
将会优化速度，舍弃大小，这种情况下它可能不会选择最小的基础类型，而是选择对优化大小有帮助的类型。为此，C++98
只支持枚举定义（所有枚举量全部列出来）；枚举声明是不被允许的。这使得编译器能为之前使用的每一个枚举选择一个基础类型。

但是不能前向声明枚举也是有缺点的。最大的缺点莫过于它可能增加编译依赖。再次考虑**Status**枚举：
```cpp
enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
                };
```
This is the kind of enum that’s likely to be used throughout a system, hence included
in a header file that every part of the system is dependent on. If a new status value is
then introduced,
```cpp
enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                audited = 500,
                indeterminate = 0xFFFFFFFF
                };
```
it’s likely that the entire system will have to be recompiled, even if only a single subsystem—
possibly only a single function!—uses the new enumerator. This is the kind
of thing that people hate. And it’s the kind of thing that the ability to forward-declare
enums in C++11 eliminates. For example, here’s a perfectly valid declaration of a
scoped enum and a function that takes one as a parameter:
```cpp
enum class Status; // forward declaration
void continueProcessing(Status s); // use of fwd-declared enum
```
The header containing these declarations requires no recompilation if Status’s
definition is revised. Furthermore, if Status is modified (e.g., to add the audited
enumerator), but continueProcessing’s behavior is unaffected (e.g., becausecontinueProcessing doesn’t use audited), continueProcessing’s implementation
need not be recompiled, either.
But if compilers need to know the size of an enum before it’s used, how can C++11’s
enums get away with forward declarations when C++98’s enums can’t? The answer is
simple: the underlying type for a scoped enum is always known, and for unscoped
enums, you can specify it.
By default, the underlying type for scoped enums is int:
```cpp
enum class Status; // underlying type is int
```
If the default doesn’t suit you, you can override it:
```cpp
enum class Status: std::uint32_t;   // underlying type for
                                    // Status is std::uint32_t
                                    // (from <cstdint>)
```
Either way, compilers know the size of the enumerators in a scoped enum.
To specify the underlying type for an unscoped enum, you do the same thing as for a
scoped enum, and the result may be forward-declared:
```cpp
enum Color: std::uint8_t;   // fwd decl for unscoped enum;
                            // underlying type is
                            // std::uint8_t
```
Underlying type specifications can also go on an enum’s definition:
```cpp
enum class Status: std::uint32_t { good = 0,
                                    failed = 1,
                                    incomplete = 100,
                                    corrupt = 200,
                                    audited = 500,
                                    indeterminate = 0xFFFFFFFF
                                    };
```
In view of the fact that scoped enums avoid namespace pollution and aren’t susceptible
to nonsensical implicit type conversions, it may surprise you to hear that there’s
at least one situation where unscoped enums may be useful. That’s when referring to
fields within C++11’s std::tuples. For example, suppose we have a tuple holding
values for the name, email address, and reputation value for a user at a social networking
website:
```cpp
using UserInfo = // type alias; see Item 9
    std::tuple<std::string, // name
    std::string, // email
    std::size_t> ; // reputation
```
