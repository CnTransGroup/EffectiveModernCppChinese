## Item 2:Understand auto type deduction
条款二:理解auto类型推导

如果你已经读过Item1的模板类型推导，那么你几乎已经知道了auto类型推导的大部分内容，至于为什么不是全部是因为这里有一个auto不同于模板类型推导的例外。但这怎么可能，模板类型推导包括模板，函数，形参，但是auto不处理这些东西啊。

你是对的，但没关系。auto类型推导和模板类型推导有一个直接的映射关系。它们之间可以通过一个非常规范非常系统化的转换流程来转换彼此。

在Item1中，模板类型推导使用下面这个函数模板来解释：
````cpp
template<typename T>
void f(ParmaType param);	//使用一些表达式调用f
````
在f的调用中，编译器使用expr推导T和ParamType。当一个变量使用auto进行声明时，auto扮演了模板的角色，变量的类型说明符扮演了ParamType的角色。废话少说，这里便是更直观的代码描述，考虑这个例子：
````cpp
auto x = 27;
````
这里x的类型说明符是auto，另一方面，在这个声明中：
````cpp
const auto cx = x;
````
类型说明符是**const auto**。另一个：
````cpp
const auto & rx=cx;
````
类型说明符是**const auto&**。在这里例子中要推导x rx cx的类型，编译器的行为看起来就像是认为这里每个声明都有一个模板，然后使用合适的初始化表达式进行处理：
````cpp
template<typename T>	//理想化的模板用来推导x的类型
void func_for_x(T param);

func_for_x(27);

template<typename T>	//理想化的模板用来推导cx 的类型
void func_for_cx(const T param);

func_for_cx(x);

template<typename T>	//理想化的模板用来推导rx的类型
void func_for_rx(const T & param);

func_for_rx(x);
````