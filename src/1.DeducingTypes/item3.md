## 条款三：理解`decltype`

**Item 3: Understand decltype**

`decltype`是一个奇怪的东西。给它一个名字或者表达式`decltype`就会告诉你这个名字或者表达式的类型。通常，它会精确的告诉你你想要的结果。但有时候它得出的结果也会让你挠头半天，最后只能求助网上问答或参考资料寻求启示。

我们将从一个简单的情况开始，没有任何令人惊讶的情况。相比模板类型推导和`auto`类型推导（参见[Item1](../1.DeducingTypes/item1.md)和[Item2](../1.DeducingTypes/item2.md)），`decltype`只是简单的返回名字或者表达式的类型：
````cpp
const int i = 0;                //decltype(i)是const int

bool f(const Widget& w);        //decltype(w)是const Widget&
                                //decltype(f)是bool(const Widget&)

struct Point{
    int x,y;                    //decltype(Point::x)是int
};                              //decltype(Point::y)是int

Widget w;                       //decltype(w)是Widget

if (f(w))…                      //decltype(f(w))是bool

template<typename T>            //std::vector的简化版本
class vector{
public:
    …
    T& operator[](std::size_t index);
    …
};

vector<int> v;                  //decltype(v)是vector<int>
…
if (v[0] == 0)…                 //decltype(v[0])是int&
````
看见了吧？没有任何奇怪的东西。

在C++11中，`decltype`最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型。举个例子，假定我们写一个函数，一个形参为容器，一个形参为索引值，这个函数支持使用方括号的方式（也就是使用“`[]`”）访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。函数的返回类型应该和索引操作返回的类型相同。

对一个`T`类型的容器使用`operator[]` 通常会返回一个`T&`对象，比如`std::deque`就是这样。但是`std::vector`有一个例外，对于`std::vector<bool>`，`operator[]`不会返回`bool&`，它会返回一个全新的对象（译注：MSVC的STL实现中返回的是`std::_Vb_reference<std::_Wrap_alloc<std::allocator<unsigned int>>>`对象）。关于这个问题的详细讨论请参见[Item6](../2.Auto/item6.md)，这里重要的是我们可以看到对一个容器进行`operator[]`操作返回的类型取决于容器本身。

使用`decltype`使得我们很容易去实现它，这是我们写的第一个版本，使用`decltype`计算返回类型，这个模板需要改良，我们把这个推迟到后面：
````cpp
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
````

函数名称前面的`auto`不会做任何的类型推导工作。相反的，他只是暗示使用了C++11的**尾置返回类型**语法，即在函数形参列表后面使用一个”`->`“符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使用函数形参相关的信息。在`authAndAccess`函数中，我们使用`c`和`i`指定返回类型。如果我们按照传统语法把函数返回类型放在函数名称之前，`c`和`i`就未被声明所以不能使用。

在这种声明中，`authAndAccess`函数返回`operator[]`应用到容器中返回的对象的类型，这也正是我们期望的结果。

C++11允许自动推导单一语句的*lambda*表达式的返回类型， C++14扩展到允许自动推导所有的*lambda*表达式和函数，甚至它们内含多条语句。对于`authAndAccess`来说这意味着在C++14标准下我们可以忽略尾置返回类型，只留下一个`auto`。使用这种声明形式，auto标示这里会发生类型推导。更准确的说，编译器将会从函数实现中推导出函数的返回类型。
````cpp
template<typename Container, typename Index>    //C++14版本，
auto authAndAccess(Container& c, Index i)       //不那么正确
{
    authenticateUser();
    return c[i];                                //从c[i]中推导返回类型
}
````
[Item2](../1.DeducingTypes/item2.md)解释了函数返回类型中使用`auto`，编译器实际上是使用的模板类型推导的那套规则。如果那样的话这里就会有一些问题。正如我们之前讨论的，`operator[]`对于大多数`T`类型的容器会返回一个`T&`，但是[Item1](../1.DeducingTypes/item1.md)解释了在模板类型推导期间，表达式的引用性（reference-ness）会被忽略。基于这样的规则，考虑它会对下面用户的代码有哪些影响：

````cpp
std::deque<int> d;
…
authAndAccess(d, 5) = 10;               //认证用户，返回d[5]，
                                        //然后把10赋值给它
                                        //无法通过编译器！
````
在这里`d[5]`本该返回一个`int&`，但是模板类型推导会剥去引用的部分，因此产生了`int`返回类型。函数返回的那个`int`是一个右值，上面的代码尝试把10赋值给右值`int`，C++11禁止这样做，所以代码无法编译。

要想让`authAndAccess`像我们期待的那样工作，我们需要使用`decltype`类型推导来推导它的返回值，即指定`authAndAccess`应该返回一个和`c[i]`表达式类型一样的类型。C++期望在某些情况下当类型被暗示时需要使用`decltype`类型推导的规则，C++14通过使用`decltype(auto)`说明符使得这成为可能。我们第一次看见`decltype(auto)`可能觉得非常的矛盾（到底是`decltype`还是`auto`？），实际上我们可以这样解释它的意义：`auto`说明符表示这个类型将会被推导，`decltype`说明`decltype`的规则将会被用到这个推导过程中。因此我们可以这样写`authAndAccess`：
````cpp
template<typename Container, typename Index>    //C++14版本，
decltype(auto)                                  //可以工作，
authAndAccess(Container& c, Index i)            //但是还需要
{                                               //改良
    authenticateUser();
    return c[i];
}
````
现在`authAndAccess`将会真正的返回`c[i]`的类型。现在事情解决了，一般情况下`c[i]`返回`T&`，`authAndAccess`也会返回`T&`，特殊情况下`c[i]`返回一个对象，`authAndAccess`也会返回一个对象。

`decltype(auto)`的使用不仅仅局限于函数返回类型，当你想对初始化表达式使用`decltype`推导的规则，你也可以使用：

````cpp
Widget w;

const Widget& cw = w;

auto myWidget1 = cw;                    //auto类型推导
                                        //myWidget1的类型为Widget
decltype(auto) myWidget2 = cw;          //decltype类型推导
                                        //myWidget2的类型是const Widget&
````

但是这里有两个问题困惑着你。一个是我之前提到的`authAndAccess`的改良至今都没有描述。让我们现在加上它。

再看看C++14版本的`authAndAccess`声明：
````cpp
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i);
````
容器通过传引用的方式传递非常量左值引用（lvalue-reference-to-non-**const**），因为返回一个引用允许用户可以修改容器。但是这意味着在不能给这个函数传递右值容器，右值不能被绑定到左值引用上（除非这个左值引用是一个const（lvalue-references-to-**const**），但是这里明显不是）。

公认的向`authAndAccess`传递一个右值是一个[edge case](https://en.wikipedia.org/wiki/Edge_case)（译注：在极限操作情况下会发生的事情，类似于会发生但是概率较小的事情）。一个右值容器，是一个临时对象，通常会在`authAndAccess`调用结束被销毁，这意味着`authAndAccess`返回的引用将会成为一个悬置的（dangle）引用。但是使用向`authAndAccess`传递一个临时变量也并不是没有意义，有时候用户可能只是想简单的获得临时容器中的一个元素的拷贝，比如这样：
````cpp
std::deque<std::string> makeStringDeque();      //工厂函数

//从makeStringDeque中获得第五个元素的拷贝并返回
auto s = authAndAccess(makeStringDeque(), 5);
````
要想支持这样使用`authAndAccess`我们就得修改一下当前的声明使得它支持左值和右值。重载是一个不错的选择（一个函数重载声明为左值引用，另一个声明为右值引用），但是我们就不得不维护两个重载函数。另一个方法是使`authAndAccess`的引用可以绑定左值和右值，[Item24](../5.RRefMovSemPerfForw/item24.md)解释了那正是通用引用能做的，所以我们这里可以使用通用引用进行声明：
````cpp
template<typename Containter, typename Index>   //现在c是通用引用
decltype(auto) authAndAccess(Container&& c, Index i);
````
在这个模板中，我们不知道我们操纵的容器的类型是什么，那意味着我们同样不知道它使用的索引对象（index objects）的类型。对一个未知类型的对象（这里指索引对象，也就是authAndAccess的函数的第二个参数）使用传值通常会造成不必要的拷贝，对程序的性能有极大的影响，还会造成对象切片行为（参见[item41](../8.Tweaks/item41.md)），以及给同事落下笑柄。但是就容器索引来说，我们遵照标准模板库对于索引的处理是有理由的（比如`std::string`，`std::vector`和`std::deque`的`operator[]`），所以我们坚持传值调用。

然而，我们还需要更新一下模板的实现，让它能听从[Item25](../5.RRefMovSemPerfForw/item25.md)的告诫应用`std::forward`实现通用引用：
````cpp
template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
````
这样就能对我们的期望交上一份满意的答卷，但是这要求编译器支持C++14。如果你没有这样的编译器，你还需要使用C++11版本的模板，它看起来和C++14版本的极为相似，除了你不得不指定函数返回类型之外：
````cpp
template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
````

另一个问题是就像我在条款的开始唠叨的那样，`decltype`通常会产生你期望的结果，但并不总是这样。在**极少数情况下**它产生的结果可能让你很惊讶。老实说如果你不是一个大型库的实现者你不太可能会遇到这些异常情况。

为了**完全**理解`decltype`的行为，你需要熟悉一些特殊情况。它们大多数都太过晦涩以至于几乎没有书进行有过权威的讨论，这本书也不例外，但是其中的一个会让我们更加理解`decltype`的使用。

将`decltype`应用于变量名会产生该变量名的声明类型。虽然变量名都是左值表达式，但这不会影响`decltype`的行为。（译者注：这里是说对于单纯的变量名，`decltype`只会返回变量的声明类型）然而，对于比单纯的变量名更复杂的左值表达式，`decltype`可以确保报告的类型始终是左值引用。也就是说，如果一个不是单纯变量名的左值表达式的类型是`T`，那么`decltype`会把这个表达式的类型报告为`T&`。这几乎没有什么太大影响，因为大多数左值表达式的类型天生具备一个左值引用修饰符。例如，返回左值的函数总是返回左值引用。

这个行为暗含的意义值得我们注意，在：
````cpp
int x = 0;
````
中，`x`是一个变量的名字，所以`decltype(x)`是`int`。但是如果用一个小括号包覆这个名字，比如这样`(x)` ，就会产生一个比名字更复杂的表达式。对于名字来说，`x`是一个左值，C++11定义了表达式`(x)`也是一个左值。因此`decltype((x))`是`int&`。用小括号覆盖一个名字可以改变`decltype`对于名字产生的结果。

在C++11中这稍微有点奇怪，但是由于C++14允许了`decltype(auto)`的使用，这意味着你在函数返回语句中细微的改变就可以影响类型的推导：
````cpp
decltype(auto) f1()
{
    int x = 0;
    …
    return x;                            //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);                          //decltype((x))是int&，所以f2返回int&
}
````
注意不仅`f2`的返回类型不同于`f1`，而且它还引用了一个局部变量！这样的代码将会把你送上未定义行为的特快列车，一辆你绝对不想上第二次的车。

当使用`decltype(auto)`的时候一定要加倍的小心，在表达式中看起来无足轻重的细节将会影响到`decltype(auto)`的推导结果。为了确认类型推导是否产出了你想要的结果，请参见[Item4](../1.DeducingTypes/item4.md)描述的那些技术。

同时你也不应该忽略`decltype`这块大蛋糕。没错，`decltype`（单独使用或者与`auto`一起用）可能会偶尔产生一些令人惊讶的结果，但那毕竟是少数情况。通常，`decltype`都会产生你想要的结果，尤其是当你对一个变量使用`decltype`时，因为在这种情况下，`decltype`只是做一件本分之事：它产出变量的声明类型。

**请记住：**

+ `decltype`总是不加修改的产生变量或者表达式的类型。
+ 对于`T`类型的不是单纯的变量名的左值表达式，`decltype`总是产出`T`的引用即`T&`。
+ C++14支持`decltype(auto)`，就像`auto`一样，推导出类型，但是它使用`decltype`的规则进行推导。

