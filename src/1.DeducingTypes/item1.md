# 第1章 类型推导

**CHAPTER 1 Deducing Types**

C++98有一套类型推导的规则：用于函数模板的规则。C++11修改了其中的一些规则并增加了两套规则，一套用于`auto`，一套用于`decltype`。C++14扩展了`auto`和`decltype`可能使用的范围。类型推导的广泛应用，让你从拼写那些或明显或冗杂的类型名的暴行中脱离出来。它让C++程序更具适应性，因为在源代码某处修改类型会通过类型推导自动传播到其它地方。但是类型推导也会让代码更复杂，因为由编译器进行的类型推导并不总是如我们期望的那样进行。

如果对于类型推导操作没有一个扎实的理解，要想写出有现代感的C++程序是不可能的。类型推导随处可见：在函数模板调用中，在大多数`auto`出现的地方，在`decltype`表达式出现的地方，以及C++14中令人费解的应用`decltype(auto)`的地方。

这一章是每个C++程序员都应该掌握的知识。它解释了模板类型推导是如何工作的，`auto`是如何依赖类型推导的，以及`decltype`是如何按照它自己那套独特的规则工作的。它甚至解释了你该如何强制编译器使类型推导的结果可视，这能让你确认编译器的类型推导是否按照你期望的那样进行。

## 条款一：理解模板类型推导

**Item 1: Understand template type deduction**


对于一个复杂系统的用户来说，很多时候他们最关心的是它做了什么而不是它怎么做的。在这一点上，C++中的模板类型推导表现得非常出色。数百万的程序员只需要向模板函数传递实参，就能通过编译器的类型推导获得令人满意的结果，尽管他们中的大多数在被逼无奈的情况下，对于传递给函数的那些实参是如何引导编译器进行类型推导的，也只能给出非常模糊的描述。

如果那些人中包括你，我有一个好消息和一个坏消息。好消息是现在C++最重要最吸引人的特性`auto`是建立在模板类型推导的基础上的。如果你满意C++98的模板类型推导，那么你也会满意C++11的`auto`类型推导。坏消息是当模板类型推导规则应用于`auto`环境时，有时不如应用于template时那么直观。由于这个原因，真正理解`auto`基于的模板类型推导的方方面面非常重要。这项条款便包含了你需要知道的东西。

如果你不介意浏览少许伪代码，我们可以考虑像这样一个函数模板：
````cpp
template<typename T>
void f(ParamType param);
````
它的调用看起来像这样
````cpp
f(expr);                        //使用表达式调用f
````
在编译期间，编译器使用`expr`进行两个类型推导：一个是针对`T`的，另一个是针对`ParamType`的。这两个类型通常是不同的，因为`ParamType`包含一些修饰，比如`const`和引用修饰符。举个例子，如果模板这样声明：
````cpp
template<typename T>
void f(const T& param);         //ParamType是const T&
````
然后这样进行调用
````cpp
int x = 0;
f(x);                           //用一个int类型的变量调用f
````
`T`被推导为`int`，`ParamType`却被推导为`const int&`

我们可能很自然的期望`T`和传递进函数的实参是相同的类型，也就是，`T`为`expr`的类型。在上面的例子中，事实就是那样：`x`是`int`，`T`被推导为`int`。但有时情况并非总是如此，`T`的类型推导不仅取决于`expr`的类型，也取决于`ParamType`的类型。这里有三种情况：

+ `ParamType`是一个指针或引用，但不是通用引用（关于通用引用请参见[Item24](../5.RRefMovSemPerfForw/item24.md)。在这里你只需要知道它存在，而且不同于左值引用和右值引用）
+ `ParamType`一个通用引用
+ `ParamType`既不是指针也不是引用

我们下面将分成三个情景来讨论这三种情况，每个情景的都基于我们之前给出的模板：
````cpp
template<typename T>
void f(ParamType param);

f(expr);                        //从expr中推导T和ParamType
````

### 情景一：`ParamType`是一个指针或引用，但不是通用引用

最简单的情况是`ParamType`是一个指针或者引用，但非通用引用。在这种情况下，类型推导会这样进行：

1. 如果`expr`的类型是一个引用，忽略引用部分
2. 然后`expr`的类型与`ParamType`进行模式匹配来决定`T`

举个例子，如果这是我们的模板，
````cpp
template<typename T>
void f(T& param);               //param是一个引用
````
我们声明这些变量，
````cpp
int x=27;                       //x是int
const int cx=x;                 //cx是const int
const int& rx=x;                //rx是指向作为const int的x的引用
````
在不同的调用中，对`param`和`T`推导的类型会是这样：
````cpp
f(x);                           //T是int，param的类型是int&
f(cx);                          //T是const int，param的类型是const int&
f(rx);                          //T是const int，param的类型是const int&
````
在第二个和第三个调用中，注意因为`cx`和`rx`被指定为`const`值，所以`T`被推导为`const int`，从而产生了`const int&`的形参类型。这对于调用者来说很重要。当他们传递一个`const`对象给一个引用类型的形参时，他们期望对象保持不可改变性，也就是说，形参是reference-to-`const`的。这也是为什么将一个`const`对象传递给以`T&`类型为形参的模板安全的：对象的常量性`const`ness会被保留为`T`的一部分。

在第三个例子中，注意即使`rx`的类型是一个引用，`T`也会被推导为一个非引用 ，这是因为`rx`的引用性（reference-ness）在类型推导中会被忽略。

这些例子只展示了左值引用，但是类型推导会如左值引用一样对待右值引用。当然，右值只能传递给右值引用，但是在类型推导中这种限制将不复存在。

如果我们将`f`的形参类型`T&`改为`const T&`，情况有所变化，但不会变得那么出人意料。`cx`和`rx`的`const`ness依然被遵守，但是因为现在我们假设`param`是reference-to-`const`，`const`不再被推导为`T`的一部分：

```cpp
template<typename T>
void f(const T& param);         //param现在是reference-to-const

int x = 27;                     //如之前一样
const int cx = x;               //如之前一样
const int& rx = x;              //如之前一样

f(x);                           //T是int，param的类型是const int&
f(cx);                          //T是int，param的类型是const int&
f(rx);                          //T是int，param的类型是const int&
```

同之前一样，`rx`的reference-ness在类型推导中被忽略了。

如果`param`是一个指针（或者指向`const`的指针）而不是引用，情况本质上也一样：

```cpp
template<typename T>
void f(T* param);               //param现在是指针

int x = 27;                     //同之前一样
const int *px = &x;             //px是指向作为const int的x的指针

f(&x);                          //T是int，param的类型是int*
f(px);                          //T是const int，param的类型是const int*
```

到现在为止，你会发现你自己打哈欠犯困，因为C++的类型推导规则对引用和指针形参如此自然，书面形式来看这些非常枯燥。所有事情都那么理所当然！那正是在类型推导系统中你所想要的。

### 情景二：`ParamType`是一个通用引用

模板使用通用引用形参的话，那事情就不那么明显了。这样的形参被声明为像右值引用一样（也就是，在函数模板中假设有一个类型形参`T`，那么通用引用声明形式就是`T&&`)，它们的行为在传入左值实参时大不相同。完整的叙述请参见[Item24](../5.RRefMovSemPerfForw/item24.md)，在这有些最必要的你还是需要知道：

+ 如果`expr`是左值，`T`和`ParamType`都会被推导为左值引用。这非常不寻常，第一，这是模板类型推导中唯一一种`T`被推导为引用的情况。第二，虽然`ParamType`被声明为右值引用类型，但是最后推导的结果是左值引用。
+ 如果`expr`是右值，就使用正常的（也就是**情景一**）推导规则

举个例子：
````cpp
template<typename T>
void f(T&& param);              //param现在是一个通用引用类型
		
int x=27;                       //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;              //如之前一样

f(x);                           //x是左值，所以T是int&，
                                //param类型也是int&

f(cx);                          //cx是左值，所以T是const int&，
                                //param类型也是const int&

f(rx);                          //rx是左值，所以T是const int&，
                                //param类型也是const int&

f(27);                          //27是右值，所以T是int，
                                //param类型就是int&&
````
[Item24](../5.RRefMovSemPerfForw/item24.md)详细解释了为什么这些例子是像这样发生的。这里关键在于通用引用的类型推导规则是不同于普通的左值或者右值引用的。尤其是，当通用引用被使用时，类型推导会区分左值实参和右值实参，但是对非通用引用时不会区分。

### 情景三：`ParamType`既不是指针也不是引用

当`ParamType`既不是指针也不是引用时，我们通过传值（pass-by-value）的方式处理：
````cpp
template<typename T>
void f(T param);                //以传值的方式处理param
````
这意味着无论传递什么`param`都会成为它的一份拷贝——一个完整的新对象。事实上`param`成为一个新对象这一行为会影响`T`如何从`expr`中推导出结果。

1. 和之前一样，如果`expr`的类型是一个引用，忽略这个引用部分
2. 如果忽略`expr`的引用性（reference-ness）之后，`expr`是一个`const`，那就再忽略`const`。如果它是`volatile`，也忽略`volatile`（`volatile`对象不常见，它通常用于驱动程序的开发中。关于`volatile`的细节请参见[Item40](../7.TheConcurrencyAPI/item40.md)）

因此
````cpp
int x=27;                       //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;              //如之前一样

f(x);                           //T和param的类型都是int
f(cx);                          //T和param的类型都是int
f(rx);                          //T和param的类型都是int
````
注意即使`cx`和`rx`表示`const`值，`param`也不是`const`。这是有意义的。`param`是一个完全独立于`cx`和`rx`的对象——是`cx`或`rx`的一个拷贝。具有常量性的`cx`和`rx`不可修改并不代表`param`也是一样。这就是为什么`expr`的常量性`const`ness（或易变性`volatile`ness)在推导`param`类型时会被忽略：因为`expr`不可修改并不意味着它的拷贝也不能被修改。

认识到只有在传值给形参时才会忽略`const`（和`volatile`）这一点很重要，正如我们看到的，对于reference-to-`const`和pointer-to-`const`形参来说，`expr`的常量性`const`ness在推导时会被保留。但是考虑这样的情况，`expr`是一个`const`指针，指向`const`对象，`expr`通过传值传递给`param`：
````cpp
template<typename T>
void f(T param);                //仍然以传值的方式处理param

const char* const ptr =         //ptr是一个常量指针，指向常量对象 
    "Fun with pointers";

f(ptr);                         //传递const char * const类型的实参
````
在这里，解引用符号（\*）的右边的`const`表示`ptr`本身是一个`const`：`ptr`不能被修改为指向其它地址，也不能被设置为null（解引用符号左边的`const`表示`ptr`指向一个字符串，这个字符串是`const`，因此字符串不能被修改）。当`ptr`作为实参传给`f`，组成这个指针的每一比特都被拷贝进`param`。像这种情况，`ptr`**自身的值会被传给形参**，根据类型推导的第三条规则，`ptr`自身的常量性`const`ness将会被省略，所以`param`是`const char*`，也就是一个可变指针指向`const`字符串。在类型推导中，这个指针指向的数据的常量性`const`ness将会被保留，但是当拷贝`ptr`来创造一个新指针`param`时，`ptr`自身的常量性`const`ness将会被忽略。

### 数组实参

上面的内容几乎覆盖了模板类型推导的大部分内容，但这里还有一些小细节值得注意，比如数组类型不同于指针类型，虽然它们两个有时候是可互换的。关于这个错觉最常见的例子是，在很多上下文中数组会退化为指向它的第一个元素的指针。这样的退化允许像这样的代码可以被编译：
````cpp
const char name[] = "J. P. Briggs";     //name的类型是const char[13]

const char * ptrToName = name;          //数组退化为指针
````
在这里`const char*`指针`ptrToName`会由`name`初始化，而`name`的类型为`const char[13]`，这两种类型（`const char*`和`const char[13]`）是不一样的，但是由于数组退化为指针的规则，编译器允许这样的代码。


但要是一个数组传值给一个模板会怎样？会发生什么？
````cpp
template<typename T>
void f(T param);                        //传值形参的模板

f(name);                                //T和param会推导成什么类型?
````
我们从一个简单的例子开始，这里有一个函数的形参是数组，是的，这样的语法是合法的，
````cpp
void myFunc(int param[]);
````
但是数组声明会被视作指针声明，这意味着`myFunc`的声明和下面声明是等价的：
````cpp
void myFunc(int* param);                //与上面相同的函数
````
数组与指针形参这样的等价是C语言的产物，C++又是建立在C语言的基础上，它让人产生了一种数组和指针是等价的的错觉。

因为数组形参会视作指针形参，所以传值给模板的一个数组类型会被推导为一个指针类型。这意味着在模板函数`f`的调用中，它的类型形参`T`会被推导为`const char*`：
````cpp
f(name);                        //name是一个数组，但是T被推导为const char*
````
但是现在难题来了，虽然函数不能声明形参为真正的数组，但是**可以**接受指向数组的**引用**！所以我们修改`f`为传引用：
````cpp
template<typename T>
void f(T& param);                       //传引用形参的模板
````
我们这样进行调用，
````cpp
f(name);                                //传数组给f
````
`T`被推导为了真正的数组！这个类型包括了数组的大小，在这个例子中`T`被推导为`const char[13]`，`f`的形参（对这个数组的引用）的类型则为`const char (&)[13]`。是的，这种语法看起来简直有毒，但是知道它将会让你在关心这些问题的人的提问中获得大神的称号。

有趣的是，可声明指向数组的引用的能力，使得我们可以创建一个模板函数来推导出数组的大小：
````cpp
//在编译期间返回一个数组大小的常量值（//数组形参没有名字，
//因为我们只关心数组的大小）
template<typename T, std::size_t N>                     //关于
constexpr std::size_t arraySize(T (&)[N]) noexcept      //constexpr
{                                                       //和noexcept
    return N;                                           //的信息
}                                                       //请看下面
````
在[Item15](../3.MovingToModernCpp/item15.md)提到将一个函数声明为`constexpr`使得结果在编译期间可用。这使得我们可以用一个花括号声明一个数组，然后第二个数组可以使用第一个数组的大小作为它的大小，就像这样：
````cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };             //keyVals有七个元素

int mappedVals[arraySize(keyVals)];                     //mappedVals也有七个
````
当然作为一个现代C++程序员，你自然应该想到使用`std::array`而不是内置的数组：
````cpp
std::array<int, arraySize(keyVals)> mappedVals;         //mappedVals的大小为7
````
至于`arraySize`被声明为`noexcept`，会使得编译器生成更好的代码，具体的细节请参见[Item14](../3.MovingToModernCpp/item14.md)。

### 函数实参

在C++中不只是数组会退化为指针，函数类型也会退化为一个函数指针，我们对于数组类型推导的全部讨论都可以应用到函数类型推导和退化为函数指针上来。结果是：
````cpp
void someFunc(int, double);         //someFunc是一个函数，
                                    //类型是void(int, double)

template<typename T>
void f1(T param);                   //传值给f1

template<typename T>
void f2(T & param);                 //传引用给f2

f1(someFunc);                       //param被推导为指向函数的指针，
                                    //类型是void(*)(int, double)
f2(someFunc);                       //param被推导为指向函数的引用，
                                    //类型是void(&)(int, double)
````
这个实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。

这里你需要知道：`auto`依赖于模板类型推导。正如我在开始谈论的，在大多数情况下它们的行为很直接。在通用引用中对于左值的特殊处理使得本来很直接的行为变得有些污点，然而，数组和函数退化为指针把这团水搅得更浑浊。有时你只需要编译器告诉你推导出的类型是什么。这种情况下，翻到[item4](../1.DeducingTypes/item4.md),它会告诉你如何让编译器这么做。

**请记住：**

+ 在模板类型推导时，有引用的实参会被视为无引用，他们的引用会被忽略
+ 对于通用引用的推导，左值实参会被特殊对待
+ 对于传值类型推导，`const`和/或`volatile`实参会被认为是non-`const`的和non-`volatile`的
+ 在模板类型推导时，数组名或者函数名实参会退化为指针，除非它们被用于初始化引用 

