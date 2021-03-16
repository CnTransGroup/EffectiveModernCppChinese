## 条款二：理解`auto`类型推导

**Item 2: Understand `auto` type deduction**

如果你已经读过[Item1](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item1.md)的模板类型推导，那么你几乎已经知道了`auto`类型推导的大部分内容，至于为什么不是全部是因为这里有一个`auto`不同于模板类型推导的例外。但这怎么可能？模板类型推导包括模板，函数，形参，但`auto`不处理这些东西啊。

你是对的，但没关系。`auto`类型推导和模板类型推导有一个直接的映射关系。它们之间可以通过一个非常规范非常系统化的转换流程来转换彼此。

在[Item1](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)中，模板类型推导使用下面这个函数模板
````cpp
template<typename T>
void f(ParmaType param);
````
和这个调用来解释：

```cpp
f(expr);                        //使用一些表达式调用f
```

在`f`的调用中，编译器使用`expr`推导`T`和`ParamType`的类型。

当一个变量使用`auto`进行声明时，`auto`扮演了模板中`T`的角色，变量的类型说明符扮演了`ParamType`的角色。废话少说，这里便是更直观的代码描述，考虑这个例子：

````cpp
auto x = 27;
````
这里`x`的类型说明符是`auto`自己，另一方面，在这个声明中：
````cpp
const auto cx = x;
````
类型说明符是`const auto`。另一个：
````cpp
const auto & rx=cx;
````
类型说明符是`const auto&`。在这里例子中要推导`x`，`rx`和`cx`的类型，编译器的行为看起来就像是认为这里每个声明都有一个模板，然后使用合适的初始化表达式进行调用：
````cpp
template<typename T>            //概念化的模板用来推导x的类型
void func_for_x(T param);

func_for_x(27);                 //概念化调用：
                                //param的推导类型是x的类型

template<typename T>            //概念化的模板用来推导cx的类型
void func_for_cx(const T param);

func_for_cx(x);                 //概念化调用：
                                //param的推导类型是cx的类型

template<typename T>            //概念化的模板用来推导rx的类型
void func_for_rx(const T & param);

func_for_rx(x);                 //概念化调用：
                                //param的推导类型是rx的类型
````
正如我说的，`auto`类型推导除了一个例外（我们很快就会讨论），其他情况都和模板类型推导一样。

[Item1](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item1.md)基于`ParamType`——在函数模板中`param`的类型说明符——的不同特征，把模板类型推导分成三个部分来讨论。在使用`auto`作为类型说明符的变量声明中，类型说明符代替了`ParamType`，因此Item1描述的三个情景稍作修改就能适用于auto：

+ 情景一：类型说明符是一个指针或引用但不是通用引用
+ 情景二：类型说明符一个通用引用
+ 情景三：类型说明符既不是指针也不是引用

我们早已看过情景一和情景三的例子：
````cpp
auto x = 27;                    //情景三（x既不是指针也不是引用）
const auto cx = x;              //情景三（cx也一样）
const auto & rx=cx;             //情景一（rx是非通用引用）
````
情景二像你期待的一样运作：

```cpp
auto&& uref1 = x;               //x是int左值，
                                //所以uref1类型为int&
auto&& uref2 = cx;              //cx是const int左值，
                                //所以uref2类型为const int&
auto&& uref3 = 27;              //27是int右值，
                                //所以uref3类型为int&&
```

[Item1](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item1.md)讨论并总结了对于non-reference类型说明符，数组和函数名如何退化为指针。那些内容也同样适用于`auto`类型推导：

````cpp
const char name[] =             //name的类型是const char[13]
 "R. N. Briggs";

auto arr1 = name;               //arr1的类型是const char*
auto& arr2 = name;              //arr2的类型是const char (&)[13]

void someFunc(int, double);     //someFunc是一个函数，
                                //类型为void(int, double)

auto func1 = someFunc;          //func1的类型是void (*)(int, double)
auto& func2 = someFunc;         //func2的类型是void (&)(int, double)
````
就像你看到的那样，`auto`类型推导和模板类型推导几乎一样的工作，它们就像一个硬币的两面。

讨论完相同点接下来就是不同点，前面我们已经说到`auto`类型推导和模板类型推导有一个例外使得它们的工作方式不同，接下来我们要讨论的就是那个例外。
我们从一个简单的例子开始，如果你想声明一个带有初始值27的`int`，C++98提供两种语法选择：

````cpp
int x1 = 27;
int x2(27);
````
C++11由于也添加了用于支持统一初始化（**uniform initialization**）的语法：
````cpp
int x3 = { 27 };
int x4{ 27 };
````
总之，这四种不同的语法只会产生一个相同的结果：变量类型为`int`值为27

但是[Item5](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/2.Auto/item5.md)解释了使用`auto`说明符代替指定类型说明符的好处，所以我们应该很乐意把上面声明中的`int`替换为`auto`，我们会得到这样的代码：
````cpp
auto x1 = 27;
auto x2(27);
auto x3 = { 27 };
auto x4{ 27 };
````
这些声明都能通过编译，但是他们不像替换之前那样有相同的意义。前面两个语句确实声明了一个类型为`int`值为27的变量，但是后面两个声明了一个存储一个元素27的 `std::initializer_list<int>`类型的变量。
````cpp
auto x1 = 27;                   //类型是int，值是27
auto x2(27);                    //同上
auto x3 = { 27 };               //类型是std::initializer_list<int>，
                                //值是{ 27 }
auto x4{ 27 };                  //同上
````
这就造成了`auto`类型推导不同于模板类型推导的特殊情况。当用`auto`声明的变量使用花括号进行初始化，`auto`类型推导推出的类型则为`std::initializer_list`。如果这样的一个类型不能被成功推导（比如花括号里面包含的是不同类型的变量），编译器会拒绝这样的代码：
````cpp
auto x5 = { 1, 2, 3.0 };        //错误！无法推导std::initializer_list<T>中的T
````
就像注释说的那样，在这种情况下类型推导将会失败，但是对我们来说认识到这里确实发生了两种类型推导是很重要的。一种是由于`auto`的使用：`x5`的类型不得不被推导。因为`x5`使用花括号的方式进行初始化，`x5`必须被推导为`std::initializer_list`。但是`std::initializer_list`是一个模板。`std::initializer_list<T>`会被某种类型`T`实例化，所以这意味着`T`也会被推导。 推导落入了这里发生的第二种类型推导——模板类型推导的范围。在这个例子中推导之所以失败，是因为在花括号中的值并不是同一种类型。

对于花括号的处理是`auto`类型推导和模板类型推导唯一不同的地方。当使用`auto`声明的变量使用花括号的语法进行初始化的时候，会推导出`std::initializer_list<T>`的实例化，但是对于模板类型推导这样就行不通：
````cpp
auto x = { 11, 23, 9 };         //x的类型是std::initializer_list<int>

template<typename T>            //带有与x的声明等价的
void f(T param);                //形参声明的模板

f({ 11, 23, 9 });               //错误！不能推导出T
````
然而如果在模板中指定`T`是`std::initializer_list<T>`而留下未知`T`,模板类型推导就能正常工作：
````cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 });               //T被推导为int，initList的类型为
                                //std::initializer_list<int>
````
因此`auto`类型推导和模板类型推导的真正区别在于，`auto`类型推导假定花括号表示`std::initializer_list`而模板类型推导不会这样（确切的说是不知道怎么办）。

你可能想知道为什么`auto`类型推导和模板类型推导对于花括号有不同的处理方式。我也想知道。哎，我至今没找到一个令人信服的解释。但是规则就是规则，这意味着你必须记住如果你使用`auto`声明一个变量，并用花括号进行初始化，`auto`类型推导总会得出`std::initializer_list`的结果。如果你使用**uniform initialization（花括号的方式进行初始化）**用得很爽你就得记住这个例外以免犯错，在C++11编程中一个典型的错误就是偶然使用了`std::initializer_list<T>`类型的变量，这个陷阱也导致了很多C++程序员抛弃花括号初始化，只有不得不使用的时候再做考虑。（在[Item7](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item7.md)讨论了必须使用时该怎么做）

对于C++11故事已经说完了。但是对于C++14故事还在继续，C++14允许`auto`用于函数返回值并会被推导（参见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)），而且C++14的*lambda*函数也允许在形参声明中使用`auto`。但是在这些情况下`auto`实际上使用**模板类型推导**的那一套规则在工作，而不是`auto`类型推导，所以说下面这样的代码不会通过编译：
````cpp
auto createInitList()
{
    return { 1, 2, 3 };         //错误！不能推导{ 1, 2, 3 }的类型
}
````
同样在C++14的lambda函数中这样使用auto也不能通过编译：
````cpp
std::vector<int> v;
…
auto resetV = 
    [&v](const auto& newValue){ v = newValue; };        //C++14
…
resetV({ 1, 2, 3 });            //错误！不能推导{ 1, 2, 3 }的类型
````

**请记住：**

+ `auto`类型推导通常和模板类型推导相同，但是`auto`类型推导假定花括号初始化代表`std::initializer_list`，而模板类型推导不这样做
+ 在C++14中`auto`允许出现在函数返回值或者*lambda*函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是`auto`类型推导
