## 条款八：优先考虑`nullptr`而非`0`和`NULL`

**Item 8: Prefer `nullptr` to `0` and `NULL`**

你看这样对不对：字面值`0`是一个`int`不是指针。如果C++发现在当前上下文只能使用指针，它会很不情愿的把`0`解释为指针，但是那是最后的退路。一般来说C++的解析策略是把`0`看做`int`而不是指针。

实际上，`NULL`也是这样的。但在`NULL`的实现细节有些不确定因素，因为实现被允许给`NULL`一个除了`int`之外的整型类型（比如`long`）。这不常见，但也算不上问题所在。这里的问题不是`NULL`没有一个确定的类型，而是`0`和`NULL`都不是指针类型。

在C++98中，对指针类型和整型进行重载意味着可能导致奇怪的事情。如果给下面的重载函数传递`0`或`NULL`，它们绝不会调用指针版本的重载函数：
````cpp
void f(int);        //三个f的重载函数
void f(bool);
void f(void*);

f(0);               //调用f(int)而不是f(void*)

f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)
````
而`f(NULL)`的不确定行为是由`NULL`的实现不同造成的。如果`NULL`被定义为`0L`（指的是`0`为`long`类型），这个调用就具有二义性，因为从`long`到`int`的转换或从`long`到`bool`的转换或`0L`到`void*`的转换都同样好。有趣的是源代码**表现出**的意思（“我使用空指针`NULL`调用`f`”）和**实际表达出**的意思（“我是用整型数据而不是空指针调用`f`”）是相矛盾的。这种违反直觉的行为导致C++98程序员都将避开同时重载指针和整型作为编程准则（译注：请务必注意结合上下文使用这条规则）。在C++11中这个编程准则也有效，因为尽管我这个条款建议使用`nullptr`，可能很多程序员还是会继续使用`0`或`NULL`，哪怕`nullptr`是更好的选择。

`nullptr`的优点是它不是整型。老实说它也不是一个指针类型，但是你可以把它认为是**所有**类型的指针。`nullptr`的真正类型是`std::nullptr_t`，在一个完美的循环定义以后，`std::nullptr_t`又被定义为`nullptr`。`std::nullptr_t`可以隐式转换为指向任何内置类型的指针，这也是为什么`nullptr`表现得像所有类型的指针。

使用`nullptr`调用`f`将会调用`void*`版本的重载函数，因为`nullptr`不能被视作任何整型：
````cpp
f(nullptr);         //调用重载函数f的f(void*)版本
````
使用`nullptr`代替`0`和`NULL`可以避开了那些令人奇怪的函数重载决议，这不是它的唯一优势。它也可以使代码表意明确，尤其是当涉及到与`auto`声明的变量一起使用时。举个例子，假如你在一个代码库中遇到了这样的代码：
````cpp
auto result = findRecord( /* arguments */ );
if (result == 0) {
    …
} 
````
如果你不知道`findRecord`返回了什么（或者不能轻易的找出），那么你就不太清楚到底`result`是一个指针类型还是一个整型。毕竟，`0`（用来测试`result`的值的那个）也可以像我们之前讨论的那样被解析。但是换一种假设如果你看到这样的代码：
````cpp
auto result = findRecord( /* arguments */ );

if (result == nullptr) {  
    …
}
````
这就没有任何歧义：`result`的结果一定是指针类型。

当模板出现时`nullptr`就更有用了。假如你有一些函数只能被合适的已锁互斥量调用。每个函数都有一个不同类型的指针：
````cpp
int    f1(std::shared_ptr<Widget> spw);     //只能被合适的
double f2(std::unique_ptr<Widget> upw);     //已锁互斥量
bool   f3(Widget* pw);                      //调用
````
如果这样传递空指针：
````cpp
std::mutex f1m, f2m, f3m;       //用于f1，f2，f3函数的互斥量

using MuxGuard =                //C++11的typedef，参见Item9
    std::lock_guard<std::mutex>;
…

{  
    MuxGuard g(f1m);            //为f1m上锁
    auto result = f1(0);        //向f1传递0作为空指针
}                               //解锁 
…
{  
    MuxGuard g(f2m);            //为f2m上锁
    auto result = f2(NULL);     //向f2传递NULL作为空指针
}                               //解锁 
…
{
    MuxGuard g(f3m);            //为f3m上锁
    auto result = f3(nullptr);  //向f3传递nullptr作为空指针
}                               //解锁 
````
令人遗憾前两个调用没有使用`nullptr`，但是代码可以正常运行，这也许对一些东西有用。但是重复的调用代码——为互斥量上锁，调用函数，解锁互斥量——更令人遗憾。它让人很烦。模板就是被设计于减少重复代码，所以让我们模板化这个调用流程：
````cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,                 
                 MuxType& mutex,                 
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);  
    return func(ptr); 
}
````
如果你对函数返回类型（`auto ... -> decltype(func(ptr))`）感到困惑不解，[Item3](../1.DeducingTypes/item3.md)可以帮助你。在C++14中代码的返回类型还可以被简化为`decltype(auto)`：

````cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,       //C++14
                           MuxType& mutex,
                           PtrType ptr)
{ 
    MuxGuard g(mutex);  
    return func(ptr); 
}
````
可以写这样的代码调用`lockAndCall`模板（两个版本都可）：
````cpp
auto result1 = lockAndCall(f1, f1m, 0);         //错误！
...
auto result2 = lockAndCall(f2, f2m, NULL);      //错误！
...
auto result3 = lockAndCall(f3, f3m, nullptr);   //没问题
````
代码虽然可以这样写，但是就像注释中说的，前两个情况不能通过编译。在第一个调用中存在的问题是当`0`被传递给`lockAndCall`模板，模板类型推导会尝试去推导实参类型，`0`的类型总是`int`，所以这就是这次调用`lockAndCall`实例化出的`ptr`的类型。不幸的是，这意味着`lockAndCall`中`func`会被`int`类型的实参调用，这与`f1`期待的`std::shared_ptr<Widget>`形参不符。传递`0`给`lockAndCall`本来想表示空指针，但是实际上得到的一个普通的`int`。把`int`类型看做`std::shared_ptr<Widget>`类型给`f1`自然是一个类型错误。在模板`lockAndCall`中使用`0`之所以失败是因为在模板中，传给的是`int`但实际上函数期待的是一个`std::shared_ptr<Widget>`。

第二个使用`NULL`调用的分析也是一样的。当`NULL`被传递给`lockAndCall`，形参`ptr`被推导为整型（译注：由于依赖于具体实现所以不一定是整数类型，所以用整型泛指`int`，`long`等类型），然后当`ptr`——一个`int`或者类似`int`的类型——传递给`f2`的时候就会出现类型错误，`f2`期待的是`std::unique_ptr<Widget>`。

然而，使用`nullptr`是调用没什么问题。当`nullptr`传给`lockAndCall`时，`ptr`被推导为`std::nullptr_t`。当`ptr`被传递给`f3`的时候，隐式转换使`std::nullptr_t`转换为`Widget`，因为`std::nullptr_t`可以隐式转换为任何指针类型。


模板类型推导将`0`和`NULL`推导为一个错误的类型（即它们的实际类型，而不是作为空指针的隐含意义），这就导致在当你想要一个空指针时，它们的替代品`nullptr`很吸引人。使用`nullptr`，模板不会有什么特殊的转换。另外，使用`nullptr`不会让你受到同重载决议特殊对待`0`和`NULL`一样的待遇。当你想用一个空指针，使用`nullptr`，不用`0`或者`NULL`。

**记住**
+ 优先考虑`nullptr`而非`0`和`NULL`
+ 避免重载指针和整型

