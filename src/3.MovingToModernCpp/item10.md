## 条款十：优先考虑限域`enum`而非未限域`enum`

**Item 10: Prefer scoped `enum`s to unscoped `enum`s**

通常来说，在花括号中声明一个名字会限制它的作用域在花括号之内。但这对于C++98风格的`enum`中声明的枚举名（译注：*enumerator*，连同下文“枚举名”都指*enumerator*）是不成立的。这些枚举名的名字（译注：*enumerator* names，连同下文“名字”都指names）属于包含这个`enum`的作用域，这意味着作用域内不能含有相同名字的其他东西：
```cpp
enum Color { black, white, red };   //black, white, red在
                                    //Color所在的作用域
auto white = false;                 //错误! white早已在这个作用
                                    //域中声明
```
这些枚举名的名字泄漏进它们所被定义的`enum`在的那个作用域，这个事实有一个官方的术语：未限域枚举(*unscoped `enum`*)。在C++11中它们有一个相似物，限域枚举(*scoped `enum`*)，它不会导致枚举名泄漏：
```cpp
enum class Color { black, white, red }; //black, white, red
                                        //限制在Color域内
auto white = false;                     //没问题，域内没有其他“white”

Color c = white;                        //错误，域中没有枚举名叫white

Color c = Color::white;                 //没问题
auto c = Color::white;                  //也没问题（也符合Item5的建议）
```
因为限域`enum`是通过“`enum class`”声明，所以它们有时候也被称为枚举类(*`enum` classes*)。

使用限域`enum`来减少命名空间污染，这是一个足够合理使用它而不是它的同胞未限域`enum`的理由，其实限域`enum`还有第二个吸引人的优点：在它的作用域中，枚举名是强类型。未限域`enum`中的枚举名会隐式转换为整型（现在，也可以转换为浮点类型）。因此下面这种歪曲语义的做法也是完全有效的：
```cpp
enum Color { black, white, red };       //未限域enum

std::vector<std::size_t>                //func返回x的质因子
  primeFactors(std::size_t x);

Color c = red;
…

if (c < 14.5) {                         // Color与double比较 (!)
    auto factors =                      // 计算一个Color的质因子(!)
      primeFactors(c);
    …
}
```
在`enum`后面写一个`class`就可以将非限域`enum`转换为限域`enum`，接下来就是完全不同的故事展开了。现在不存在任何隐式转换可以将限域`enum`中的枚举名转化为任何其他类型：
```cpp
enum class Color { black, white, red }; //Color现在是限域enum

Color c = Color::red;                   //和之前一样，只是
...                                     //多了一个域修饰符

if (c < 14.5) {                         //错误！不能比较
                                        //Color和double
    auto factors =                      //错误！不能向参数为std::size_t
      primeFactors(c);                  //的函数传递Color参数
    …
}
```
如果你真的很想执行`Color`到其他类型的转换，和平常一样，使用正确的类型转换运算符扭曲类型系统：
```cpp
if (static_cast<double>(c) < 14.5) {    //奇怪的代码，
                                        //但是有效
    auto factors =                                  //有问题，但是
      primeFactors(static_cast<std::size_t>(c));    //能通过编译
    …
}
```
似乎比起非限域`enum`而言，限域`enum`有第三个好处，因为限域`enum`可以被前置声明。也就是说，它们可以不指定枚举名直接声明：
```cpp
enum Color;         //错误！
enum class Color;   //没问题
```
其实这是一个误导。在C++11中，非限域`enum`也可以被前置声明，但是只有在做一些其他工作后才能实现。这些工作来源于一个事实：在C++中所有的`enum`都有一个由编译器决定的整型的底层类型。对于非限域`enum`比如`Color`，
```cpp
enum Color { black, white, red };
```
编译器可能选择`char`作为底层类型，因为这里只需要表示三个值。然而，有些`enum`中的枚举值范围可能会大些，比如：
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              indeterminate = 0xFFFFFFFF
            };
```
这里值的范围从`0`到`0xFFFFFFFF`。除了在不寻常的机器上（比如一个`char`至少有32bits的那种），编译器都会选择一个比`char`大的整型类型来表示`Status`。

为了高效使用内存，编译器通常在确保能包含所有枚举值的前提下为`enum`选择一个最小的底层类型。在一些情况下，编译器将会优化速度，舍弃大小，这种情况下它可能不会选择最小的底层类型，而是选择对优化大小有帮助的类型。为此，C++98只支持`enum`定义（所有枚举名全部列出来）；`enum`声明是不被允许的。这使得编译器能在使用之前为每一个`enum`选择一个底层类型。

但是不能前置声明`enum`也是有缺点的。最大的缺点莫过于它可能增加编译依赖。再次考虑`Status` `enum`：
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              indeterminate = 0xFFFFFFFF
            };
```
这种`enum`很有可能用于整个系统，因此系统中每个包含这个头文件的组件都会依赖它。如果引入一个新状态值，
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              audited = 500,
              indeterminate = 0xFFFFFFFF
            };
```
那么可能整个系统都得重新编译，即使只有一个子系统——或者只有一个函数——使用了新添加的枚举名。这是大家都**不希望**看到的。C++11中的前置声明`enum`s可以解决这个问题。比如这里有一个完全有效的限域`enum`声明和一个以该限域`enum`作为形参的函数声明：

```cpp
enum class Status;                  //前置声明
void continueProcessing(Status s);  //使用前置声明enum
```
即使`Status`的定义发生改变，包含这些声明的头文件也不需要重新编译。而且如果`Status`有改动（比如添加一个`audited`枚举名），`continueProcessing`的行为不受影响（比如因为`continueProcessing`没有使用这个新添加的`audited`），`continueProcessing`也不需要重新编译。

但是如果编译器在使用它之前需要知晓该`enum`的大小，该怎么声明才能让C++11做到C++98不能做到的事情呢？答案很简单：限域`enum`的底层类型总是已知的，而对于非限域`enum`，你可以指定它。

默认情况下，限域枚举的底层类型是`int`：

```cpp
enum class Status;                  //底层类型是int
```
如果默认的`int`不适用，你可以重写它：
```cpp
enum class Status: std::uint32_t;   //Status的底层类型
                                    //是std::uint32_t
                                    //（需要包含 <cstdint>）
```
不管怎样，编译器都知道限域`enum`中的枚举名占用多少字节。

要为非限域`enum`指定底层类型，你可以同上，结果就可以前向声明：

```cpp
enum Color: std::uint8_t;   //非限域enum前向声明
                            //底层类型为
                            //std::uint8_t
```
底层类型说明也可以放到`enum`定义处：
```cpp
enum class Status: std::uint32_t { good = 0,
                                   failed = 1,
                                   incomplete = 100,
                                   corrupt = 200,
                                   audited = 500,
                                   indeterminate = 0xFFFFFFFF
                                 };
```
限域`enum`避免命名空间污染而且不接受荒谬的隐式类型转换，但它并非万事皆宜，你可能会很惊讶听到至少有一种情况下非限域`enum`是很有用的。那就是牵扯到C++11的`std::tuple`的时候。比如在社交网站中，假设我们有一个*tuple*保存了用户的名字，email地址，声望值：
```cpp
using UserInfo =                //类型别名，参见Item9
    std::tuple<std::string,     //名字
               std::string,     //email地址
               std::size_t> ;   //声望
```
虽然注释说明了tuple各个字段对应的意思，但当你在另一文件遇到下面的代码那之前的注释就不是那么有用了：
```cpp
UserInfo uInfo;                 //tuple对象
…
auto val = std::get<1>(uInfo);	//获取第一个字段
```
作为一个程序员，你有很多工作要持续跟进。你应该记住第一个字段代表用户的email地址吗？我认为不。可以使用非限域`enum`将名字和字段编号关联起来以避免上述需求：
```cpp
enum UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;                         //同之前一样
…
auto val = std::get<uiEmail>(uInfo);    //啊，获取用户email字段的值
```
之所以它能正常工作是因为`UserInfoFields`中的枚举名隐式转换成`std::size_t`了，其中`std::size_t`是`std::get`模板实参所需的。

对应的限域`enum`版本就很啰嗦了：

```cpp
enum class UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;                         //同之前一样
…
auto val =
    std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
        (uInfo);
```
为避免这种冗长的表示，我们可以写一个函数传入枚举名并返回对应的`std::size_t`值，但这有一点技巧性。`std::get`是一个模板（函数），需要你给出一个`std::size_t`值的模板实参（注意使用`<>`而不是`()`），因此将枚举名变换为`std::size_t`值的函数必须**在编译期**产生这个结果。如[Item15](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item15.md)提到的，那必须是一个`constexpr`函数。

事实上，它也的确该是一个`constexpr`函数模板，因为它应该能用于任何`enum`。如果我们想让它更一般化，我们还要泛化它的返回类型。较之于返回`std::size_t`，我们更应该返回枚举的底层类型。这可以通过`std::underlying_type`这个*type trait*获得。（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)关于*type trait*的内容）。最终我们还要再加上`noexcept`修饰（参见[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)），因为我们知道它肯定不会产生异常。根据上述分析最终得到的`toUType`函数模板在编译期接受任意枚举名并返回它的值：

```cpp
template<typename E>
constexpr typename std::underlying_type<E>::type
    toUType(E enumerator) noexcept
{
    return
        static_cast<typename
                    std::underlying_type<E>::type>(enumerator);
}
```
在C++14中，`toUType`还可以进一步用`std::underlying_type_t`（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）代替`typename std::underlying_type<E>::type`打磨：

```cpp
template<typename E>                //C++14
constexpr std::underlying_type_t<E>
    toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
还可以再用C++14 `auto`（参见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）打磨一下代码：
```cpp
template<typename E>                //C++14
constexpr auto
    toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
不管它怎么写，`toUType`现在允许这样访问tuple的字段了：
```cpp
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```
这仍然比使用非限域`enum`要写更多的代码，但同时它也避免命名空间污染，防止不经意间使用隐式转换。大多数情况下，你应该会觉得多敲几个（几行）字符作为避免使用未限域枚举这种老得和2400波特率猫同时代技术的代价是值得的。

**记住**
+ C++98的`enum`即非限域`enum`。
+ 限域`enum`的枚举名仅在`enum`内可见。要转换为其它类型只能使用*cast*。
+ 非限域/限域`enum`都支持底层类型说明语法，限域`enum`底层类型默认是`int`。非限域`enum`没有默认底层类型。
+ 限域`enum`总是可以前置声明。非限域`enum`仅当指定它们的底层类型时才能前置。
