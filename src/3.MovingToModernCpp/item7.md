# 第3章 移步现代C++

**CHAPTER 3 Moving to Modern C++**

说起知名的特性，C++11/14有一大堆可以吹的东西，`auto`，智能指针（*smart pointer*），移动语义（*move semantics*），*lambda*，并发（*concurrency*）——每个都是如此的重要，这章将覆盖这些内容。掌握这些特性是必要的，要想成为高效率的现代C++程序员需要小步迈进。在从C++98小步迈进到现代C++过程中遇到的每个问题，本章都会一一回答。你什么时候应该用{}而不是()创建对象？为什么别名（*alias*）声明比`typedef`好？`constexpr`和`const`有什么不同？常量（`const`）成员函数和线程安全有什么关系？这个列表越列越多，这章将会逐个回答这些问题。

## 条款七：区别使用`()`和`{}`创建对象

**Item 7: Distinguish between `()` and `{}` when creating objects**

取决于你看问题的角度，C++11对象初始化的语法可能会让你觉得丰富的让人难以选择，亦或是乱的一塌糊涂。一般来说，初始化值要用圆括号()或者花括号{}括起来，或者放到等号"="的右边：
````cpp
int x(0);               //使用圆括号初始化

int y = 0;              //使用"="初始化

int z{ 0 };             //使用花括号初始化
````
在很多情况下，你可以使用"="和花括号的组合：
````cpp
int z = { 0 };          //使用"="和花括号
````
在这个条款的剩下部分，我通常会忽略"="和花括号组合初始化的语法，因为C++通常把它视作和只有花括号一样。

“乱的一塌糊涂”是指在初始化中使用"="可能会误导C++新手，使他们以为这里发生了赋值运算，然而实际并没有。对于像`int`这样的内置类型，研究两者区别就像在做学术，但是对于用户定义的类型而言，区别赋值运算符和初始化就非常重要了，因为它们涉及不同的函数调用：

````cpp
Widget w1;              //调用默认构造函数

Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数

w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
````
甚至对于一些初始化语法，在一些情况下C++98没有办法表达预期的初始化行为。举个例子，要想直接创建并初始化一个存放一些特殊值的STL容器是不可能的（比如1,3,5）。

C++11使用统一初始化（*uniform initialization*）来整合这些混乱且不适于所有情景的初始化语法，所谓统一初始化是指在任何涉及初始化的地方都使用单一的初始化语法。
它基于花括号，出于这个原因我更喜欢称之为括号初始化。（**译注：注意，这里的括号初始化指的是花括号初始化，在没有歧义的情况下下文的括号初始化指的都是用花括号进行初始化；当与圆括号初始化同时存在并可能产生歧义时我会直接指出。**）统一初始化是一个概念上的东西，而括号初始化是一个具体语法结构。

括号初始化让你可以表达以前表达不出的东西。使用花括号，创建并指定一个容器的初始元素变得很容易：

````cpp
std::vector<int> v{ 1, 3, 5 };  //v初始内容为1,3,5
````
括号初始化也能被用于为非静态数据成员指定默认初始值。C++11允许"="初始化不加花括号也拥有这种能力：
````cpp
class Widget{
    …

private:
    int x{ 0 };                 //没问题，x初始值为0
    int y = 0;                  //也可以
    int z(0);                   //错误！
}
````
另一方面，不可拷贝的对象（例如`std::atomic`——见[Item40](../7.TheConcurrencyAPI/item40.md)）可以使用花括号初始化或者圆括号初始化，但是不能使用"="初始化：
````cpp
std::atomic<int> ai1{ 0 };      //没问题
std::atomic<int> ai2(0);        //没问题
std::atomic<int> ai3 = 0;       //错误！
````
因此我们很容易理解为什么括号初始化又叫统一初始化，在C++中这三种方式都被看做是初始化表达式，但是只有花括号任何地方都能被使用。

括号表达式还有一个少见的特性，即它不允许内置类型间隐式的变窄转换（*narrowing conversion*）。如果一个使用了括号初始化的表达式的值，不能保证由被初始化的对象的类型来表示，代码就不会通过编译：
````cpp
double x, y, z;

int sum1{ x + y + z };          //错误！double的和可能不能表示为int
````
使用圆括号和"="的初始化不检查是否转换为变窄转换，因为由于历史遗留问题它们必须要兼容老旧代码：
````cpp
int sum2(x + y +z);             //可以（表达式的值被截为int）

int sum3 = x + y + z;           //同上
````
另一个值得注意的特性是括号表达式对于C++最令人头疼的解析问题有天生的免疫性。（译注：所谓最令人头疼的解析即*most vexing parse*，更多信息请参见[https://en.wikipedia.org/wiki/Most_vexing_parse](https://en.wikipedia.org/wiki/Most_vexing_parse)。）C++规定任何*可以被解析*为一个声明的东西*必须被解析*为声明。这个规则的副作用是让很多程序员备受折磨：他们可能想创建一个使用默认构造函数构造的对象，却不小心变成了函数声明。问题的根源是如果你调用带参构造函数，你可以这样做：

````cpp
Widget w1(10);                  //使用实参10调用Widget的一个构造函数
````
但是如果你尝试使用相似的语法调用`Widget`无参构造函数，它就会变成函数声明：
````cpp
Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
````
由于函数声明中形参列表不能带花括号，所以使用花括号初始化表明你想调用默认构造函数构造对象就没有问题：
````cpp
Widget w3{};                    //调用没有参数的构造函数构造对象
````
关于括号初始化还有很多要说的。它的语法能用于各种不同的上下文，它防止了隐式的变窄转换，而且对于C++最令人头疼的解析也天生免疫。既然好到这个程度那为什么这个条款不叫“优先考虑括号初始化语法”呢？

括号初始化的缺点是有时它有一些令人惊讶的行为。这些行为使得括号初始化、`std::initializer_list`和构造函数参与重载决议时本来就不清不楚的暧昧关系进一步混乱。把它们放到一起会让看起来应该左转的代码右转。举个例子，[Item2](../1.DeducingTypes/item2.md)解释了当`auto`声明的变量使用花括号初始化，变量类型会被推导为`std::initializer_list`，但是使用相同内容的其他初始化方式会产生更符合直觉的结果。所以，你越喜欢用`auto`，你就越不能用括号初始化。

在构造函数调用中，只要不包含`std::initializer_list`形参，那么花括号初始化和圆括号初始化都会产生一样的结果：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);      //构造函数未声明
    Widget(int i, double d);    //std::initializer_list这个形参 
    …
};
Widget w1(10, true);            //调用第一个构造函数
Widget w2{10, true};            //也调用第一个构造函数
Widget w3(10, 5.0);             //调用第二个构造函数
Widget w4{10, 5.0};             //也调用第二个构造函数
````
然而，如果有一个或者多个构造函数的声明包含一个`std::initializer_list`形参，那么使用括号初始化语法的调用更倾向于选择带`std::initializer_list`的那个构造函数。如果编译器遇到一个括号初始化并且有一个带std::initializer_list的构造函数，那么它一定会选择该构造函数。如果上面的`Widget`类有一个`std::initializer_list<long double>`作为参数的构造函数，就像这样：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);      //同上
    Widget(int i, double d);    //同上
    Widget(std::initializer_list<long double> il);      //新添加的
    …
}; 
````
`w2`和`w4`将会使用新添加的构造函数，即使另一个非`std::initializer_list`构造函数和实参更匹配：

````cpp
Widget w1(10, true);    //使用圆括号初始化，同之前一样
                        //调用第一个构造函数

Widget w2{10, true};    //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 true 转化为long double)

Widget w3(10, 5.0);     //使用圆括号初始化，同之前一样
                        //调用第二个构造函数 

Widget w4{10, 5.0};     //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 5.0 转化为long double)
````
甚至普通构造函数和移动构造函数都会被带`std::initializer_list`的构造函数劫持：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    Widget(std::initializer_list<long double> il);      //同之前一样
    operator float() const;                             //转换为float
    …
};

Widget w5(w4);                  //使用圆括号，调用拷贝构造函数

Widget w6{w4};                  //使用花括号，调用std::initializer_list构造
                                //函数（w4转换为float，float转换为double）

Widget w7(std::move(w4));       //使用圆括号，调用移动构造函数

Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
                                //函数（与w6相同原因）
````
编译器一遇到括号初始化就选择带`std::initializer_list`的构造函数的决心是如此强烈，以至于就算带`std::initializer_list`的构造函数不能被调用，它也会硬选。
````cpp
class Widget { 
public: 
    Widget(int i, bool b);                      //同之前一样
    Widget(int i, double d);                    //同之前一样
    Widget(std::initializer_list<bool> il);     //现在元素类型为bool
    …                                           //没有隐式转换函数
};

Widget w{10, 5.0};              //错误！要求变窄转换
````
这里，编译器会直接忽略前面两个构造函数（其中第二个构造函数是所有实参类型的最佳匹配），然后尝试调用`std::initializer_list<bool>`构造函数。调用这个函数将会把`int(10)`和`double(5.0)`转换为`bool`，由于会产生变窄转换（`bool`不能准确表示其中任何一个值），括号初始化拒绝变窄转换，所以这个调用无效，代码无法通过编译。

只有当没办法把括号初始化中实参的类型转化为`std::initializer_list`时，编译器才会回到正常的函数决议流程中。比如我们在构造函数中用`std::initializer_list<std::string>`代替`std::initializer_list<bool>`，这时非`std::initializer_list`构造函数将再次成为函数决议的候选者，因为没有办法把`int`和`bool`转换为`std::string`:

````cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    //现在std::initializer_list元素类型为std::string
    Widget(std::initializer_list<std::string> il);
    …                                                   //没有隐式转换函数
};

Widget w1(10, true);     // 使用圆括号初始化，调用第一个构造函数
Widget w2{10, true};     // 使用花括号初始化，现在调用第一个构造函数
Widget w3(10, 5.0);      // 使用圆括号初始化，调用第二个构造函数
Widget w4{10, 5.0};      // 使用花括号初始化，现在调用第二个构造函数
````
代码的行为和我们刚刚的论述如出一辙。这里还有一个有趣的[边缘情况](https://en.wikipedia.org/wiki/Edge_case)。假如你使用的花括号初始化是空集，并且你欲构建的对象有默认构造函数，也有`std::initializer_list`构造函数。你的空的花括号意味着什么？如果它们意味着没有实参，就该使用默认构造函数，但如果它意味着一个空的`std::initializer_list`，就该调用`std::initializer_list`构造函数。

最终会调用默认构造函数。空的花括号意味着没有实参，不是一个空的`std::initializer_list`：
````cpp
class Widget { 
public:  
    Widget();                                   //默认构造函数
    Widget(std::initializer_list<int> il);      //std::initializer_list构造函数

    …                                           //没有隐式转换函数
};

Widget w1;                      //调用默认构造函数
Widget w2{};                    //也调用默认构造函数
Widget w3();                    //最令人头疼的解析！声明一个函数
````
如果你**想**用空`std::initializer`来调用`std::initializer_list`构造函数，你就得创建一个空花括号作为函数实参——把空花括号放在圆括号或者另一个花括号内来界定你想传递的东西。
````cpp
Widget w4({});                  //使用空花括号列表调用std::initializer_list构造函数
Widget w5{{}};                  //同上
````
此时，括号初始化，`std::initializer_list`和构造函数重载的晦涩规则就会一下子涌进你的脑袋，你可能会想研究半天这些东西在你的日常编程中到底占多大比例。可能比你想象的要有用。因为`std::vector`作为受众之一会直接受到影响。`std::vector`有一个非`std::initializer_list`构造函数允许你去指定容器的初始大小，以及使用一个值填满你的容器。但它也有一个`std::initializer_list`构造函数允许你使用花括号里面的值初始化容器。如果你创建一个数值类型的`std::vector`（比如`std::vector<int>`），然后你传递两个实参，把这两个实参放到圆括号和放到花括号中有天壤之别：

````cpp
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20
````
让我们回到之前的话题。从以上讨论中我们得出两个重要结论。第一，作为一个类库作者，你需要意识到如果一堆重载的构造函数中有一个或者多个含有`std::initializer_list`形参，用户代码如果使用了括号初始化，可能只会看到你`std::initializer_list`版本的重载的构造函数。因此，你最好把你的构造函数设计为不管用户是使用圆括号还是使用花括号进行初始化都不会有什么影响。换句话说，了解了`std::vector`设计缺点后，你以后设计类的时候应该避免诸如此类的问题。

这里的暗语是如果一个类没有`std::initializer_list`构造函数，然后你添加一个，用户代码中如果使用括号初始化，可能会发现过去被决议为非`std::initializer_list`构造函数而现在被决议为新的函数。当然，这种事情也可能发生在你添加一个函数到那堆重载函数的时候：过去被决议为旧的重载函数而现在调用了新的函数。`std::initializer_list`重载不会和其他重载函数比较，它直接盖过了其它重载函数，其它重载函数几乎不会被考虑。所以如果你要加入`std::initializer_list`构造函数，请三思而后行。

第二，作为一个类库使用者，你必须认真的在花括号和圆括号之间选择一个来创建对象。大多数开发者都使用其中一种作为默认情况，只有当他们不能使用这种的时候才会考虑另一种。默认使用花括号初始化的开发者主要被适用面广、禁止变窄转换、免疫C++最令人头疼的解析这些优点所吸引。这些开发者知道在一些情况下（比如给定一个容器大小和一个初始值创建`std::vector`）要使用圆括号。默认使用圆括号初始化的开发者主要被C++98语法一致性、避免`std::initializer_list`自动类型推导、避免不会不经意间调用`std::initializer_list`构造函数这些优点所吸引。这些开发者也承认有时候只能使用花括号（比如创建一个包含着特定值的容器）。关于花括号初始化和圆括号初始化哪种更好大家没有达成一致，所以我的建议是选择一种并坚持使用它。

如果你是一个模板的作者，花括号和圆括号创建对象就更麻烦了。通常不能知晓哪个会被使用。举个例子，假如你想创建一个接受任意数量的参数来创建的对象。使用可变参数模板（*variadic template*）可以非常简单的解决：

````cpp
template<typename T,            //要创建的对象类型
         typename... Ts>        //要使用的实参的类型
void doSomeWork(Ts&&... params)
{
    create local T object from params...
    …
} 
````
在现实中我们有两种方式实现这个伪代码（关于`std::forward`请参见[Item25](../5.RRefMovSemPerfForw/item25.md)）：
````cpp
T localObject(std::forward<Ts>(params)...);             //使用圆括号
T localObject{std::forward<Ts>(params)...};             //使用花括号
````
考虑这样的调用代码：
````cpp
std::vector<int> v; 
…
doSomeWork<std::vector<int>>(10, 20);
````
如果`doSomeWork`创建`localObject`时使用的是圆括号，`std::vector`就会包含10个元素。如果`doSomeWork`创建`localObject`时使用的是花括号，`std::vector`就会包含2个元素。哪个是正确的？`doSomeWork`的作者不知道，只有调用者知道。

这正是标准库函数`std::make_unique`和`std::make_shared`（参见[Item21](../4.SmartPointers/item21.md)）面对的问题。它们的解决方案是使用圆括号，并被记录在文档中作为接口的一部分。（注：更灵活的设计——允许调用者决定从模板来的函数应该使用圆括号还是花括号——是有可能的。详情参见[Andrzej’s C++ blog](http://akrzemi1.wordpress.com/)在2013年6月5日的文章，“[Intuitive interface — Part I.](http://akrzemi1.wordpress.com/2013/06/05/intuitive-interface-part-i/)”）

**请记住：**

+ 括号初始化是最广泛使用的初始化语法，它防止变窄转换，并且对于C++最令人头疼的解析有天生的免疫性
+ 在构造函数重载决议中，编译器会尽最大努力将括号初始化与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
+ 对于数值类型的`std::vector`来说使用花括号初始化和圆括号初始化会造成巨大的不同
+ 在模板类选择使用圆括号初始化或使用花括号初始化创建对象是一个挑战。

