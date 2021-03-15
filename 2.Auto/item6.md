## 条款六：`auto`推导若非己愿，使用显式类型初始化惯用法

**Item 6: Use the explicitly typed initializer idiom when `auto` deduces undesired types**

在[Item5](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/2.Auto/item5.md)中解释了比起显式指定类型使用`auto`声明变量有若干技术优势，但是有时当你想向左转`auto`却向右转。举个例子，假如我有一个函数，参数为`Widget`，返回一个`std::vector<bool>`，这里的`bool`表示`Widget`是否提供一个独有的特性。
````cpp
std::vector<bool> features(const Widget& w);
````
更进一步假设第5个*bit*表示`Widget`是否具有高优先级，我们可以写这样的代码：
````cpp
Widget w;
…
bool highPriority = features(w)[5];     //w高优先级吗？
…
processWidget(w, highPriority);         //根据它的优先级处理w
````
这个代码没有任何问题。它会正常工作，但是如果我们使用`auto`代替`highPriority`的显式指定类型做一些看起来很无害的改变：
````cpp
auto highPriority = features(w)[5];     //w高优先级吗？
````
情况变了。所有代码仍然可编译，但是行为不再可预测：
````cpp
processWidget(w,highPriority);          //未定义行为！
````
就像注释说的，这个`processWidget`是一个未定义行为。为什么呢？答案有可能让你很惊讶，使用`auto`后`highPriority`不再是`bool`类型。虽然从概念上来说`std::vector<bool>`意味着存放`bool`，但是`std::vector<bool>`的`operator[]`不会返回容器中元素的引用（这就是`std::vector::operator[]`可返回**除了`bool`以外**的任何类型），取而代之它返回一个`std::vector<bool>::reference`的对象（一个嵌套于`std::vector<bool>`中的类）。

`std::vector<bool>::reference`之所以存在是因为`std::vector<bool>`规定了使用一个打包形式（packed form）表示它的`bool`，每个`bool`占一个*bit*。那给`std::vector`的`operator[]`带来了问题，因为`std::vector<T>`的`operator[]`应当返回一个`T&`，但是C++禁止对`bit`s的引用。无法返回一个`bool&`，`std::vector<bool>`的`operator[]`返回一个**行为类似于**`bool&`的对象。要想成功扮演这个角色，`bool&`适用的上下文`std::vector<bool>::reference`也必须一样能适用。在`std::vector<bool>::reference`的特性中，使这个原则可行的特性是一个可以向`bool`的隐式转化。（不是`bool&`，是**`bool`**。要想完整的解释`std::vector<bool>::reference`能模拟`bool&`的行为所使用的一堆技术可能扯得太远了，所以这里简单地说隐式类型转换只是这个大型马赛克的一小块）

有了这些信息，我们再来看看原始代码的一部分：
````cpp
bool highPriority = features(w)[5];     //显式的声明highPriority的类型
````
这里，`features`返回一个`std::vector<bool>`对象后再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，然后再通过隐式转换赋值给`bool`变量`highPriority`。`highPriority`因此表示的是`features`返回的`std::vector<bool>`中的第五个*bit*，这也正如我们所期待的那样。

然后再对照一下当使用`auto`时发生了什么：

````cpp
auto highPriority = features(w)[5];     //推导highPriority的类型
````
同样的，`features`返回一个`std::vector<bool>`对象，再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，但是现在这里有一点变化了，`auto`推导`highPriority`的类型为`std::vector<bool>::reference`，但是`highPriority`对象没有第五*bit*的值。

这个值取决于`std::vector<bool>::reference`的具体实现。其中的一种实现是这样的（`std::vector<bool>::reference`）对象包含一个指向机器字（*word*）的指针，然后加上方括号中的偏移实现被引用*bit*这样的行为。然后再来考虑`highPriority`初始化表达的意思，注意这里假设`std::vector<bool>::reference`就是刚提到的实现方式。

调用`features`将返回一个`std::vector<bool>`临时对象，这个对象没有名字，为了方便我们的讨论，我这里叫他`temp`。`std::vector<bool>::reference`包含一个指向*word*的指针（`temp`管理这个*word*中的*bit*s），还有相应于第5个*bit*的偏移。`highPriority`是这个`std::vector<bool>::reference`的拷贝，所以`highPriority`也包含一个指针，指向`temp`中的这个*word*，加上相应于第5个*bit*的偏移。在这个语句结束的时候`temp`将会被销毁，因为它是一个临时变量。因此`highPriority`包含一个悬置的（*dangling*）指针，如果用于`processWidget`调用中将会造成未定义行为：

````cpp
processWidget(w, highPriority);         //未定义行为！
                                        //highPriority包含一个悬置指针！
````

`std::vector<bool>::reference`是一个代理类（*proxy class*）的例子：所谓代理类就是以模仿和增强一些类型的行为为目的而存在的类。很多情况下都会使用代理类，`std::vector<bool>::reference`展示了对`std::vector<bool>`使用`operator[]`来实现引用*bit*这样的行为。另外，C++标准模板库中的智能指针（见[第4章](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)）也是用代理类实现了对原始指针的资源管理行为。代理类的功能已被大家广泛接受。事实上，“Proxy”设计模式是软件设计这座万神庙中一直都存在的高级会员。

一些代理类被设计于用以对客户可见。比如`std::shared_ptr`和`std::unique_ptr`。其他的代理类则或多或少不可见，比如`std::vector<bool>::reference`就是不可见代理类的一个例子，还有它在`std::bitset`的胞弟`std::bitset::reference`。

在后者的阵营（注：指不可见代理类）里一些C++库也是用了表达式模板（*expression templates*）的黑科技。这些库通常被用于提高数值运算的效率。给出一个矩阵类`Matrix`和矩阵对象`m1`，`m2`，`m3`，`m4`，举个例子，这个表达式
````cpp
Matrix sum = m1 + m2 + m3 + m4;
````
可以使计算更加高效，只需要使让`operator+`返回一个代理类代理结果而不是返回结果本身。也就是说，对两个`Matrix`对象使用`operator+`将会返回如`Sum<Matrix, Matrix>`这样的代理类作为结果而不是直接返回一个`Matrix`对象。在`std::vector<bool>::reference`和`bool`中存在一个隐式转换，同样对于`Matrix`来说也可以存在一个隐式转换允许`Matrix`的代理类转换为`Matrix`，这让表达式等号“`=`”右边能产生代理对象来初始化`sum`。（这个对象应当编码整个初始化表达式，即类似于`Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>`的东西。客户应该避免看到这个实际的类型。）

作为一个通则，不可见的代理类通常不适用于`auto`。这样类型的对象的生命期通常不会设计为能活过一条语句，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路。`std::vector<bool>::reference`就是这种情况，我们看到违反这个基本假设将导致未定义行为。

因此你想避开这种形式的代码：
````cpp
auto someVar = expression of "invisible" proxy class type;
````
但是你怎么能意识到你正在使用代理类？应用他们的软件不可能宣告它们的存在。它们被设计为**不可见**，至少概念上说是这样！每当你发现它们，你真的应该舍弃[Item5](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/2.Auto/item5.md)演示的`auto`所具有的诸多好处吗？

让我们首先回到如何找到它们的问题上。虽然“不可见”代理类都在程序员日常使用的雷达下方飞行，但是很多库都证明它们可以上方飞行。当你越熟悉你使用的库的基本设计理念，你的思维就会越活跃，不至于思维僵化认为代理类只能在这些库中使用。

当缺少文档的时候，可以去看看头文件。很少会出现源代码全都用代理对象，它们通常用于一些函数的返回类型，所以通常能从函数签名中看出它们的存在。这里有一份`std::vector<bool>::operator[]`的说明书：
````cpp
namespace std{                                  //来自于C++标准库
    template<class Allocator>
    class vector<bool, Allocator>{
    public:
        …
        class reference { … };

        reference operator[](size_type n);
        …
    };
}
````
假设你知道对`std::vector<T>`使用`operator[]`通常会返回一个`T&`，在这里`operator[]`不寻常的返回类型提示你它使用了代理类。多关注你使用的接口可以暴露代理类的存在。

实际上， 很多开发者都是在跟踪一些令人困惑的复杂问题或在单元测试出错进行调试时才看到代理类的使用。不管你怎么发现它们的，一旦看到`auto`推导了代理类的类型而不是被代理的类型，解决方案并不需要抛弃`auto`。`auto`本身没什么问题，问题是`auto`不会推导出你想要的类型。解决方案是强制使用一个不同的类型推导形式，这种方法我通常称之为显式类型初始器惯用法（*the explicitly typed initialized idiom*)。

显式类型初始器惯用法使用`auto`声明一个变量，然后对表达式强制类型转换（*cast*）得出你期望的推导结果。举个例子，我们该怎么将这个惯用法施加到`highPriority`上？
````cpp
auto highPriority = static_cast<bool>(features(w)[5]);
````
这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`对象，就像之前那样，但是这个转型使得表达式类型为`bool`，然后`auto`才被用于推导`highPriority`。在运行时，对`std::vector<bool>::operator[]`返回的`std::vector<bool>::reference`执行它支持的向`bool`的转型，在这个过程中指向`std::vector<bool>`的指针已经被解引用。这就避开了我们之前的未定义行为。然后5将被用于指向*bit*的指针，`bool`值被用于初始化`highPriority`。

对于`Matrix`来说，显式类型初始器惯用法是这样的：
````cpp
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
````
应用这个惯用法不限制初始化表达式产生一个代理类。它也可以用于强调你声明了一个变量类型，它的类型不同于初始化表达式的类型。举个例子，假设你有这样一个表达式计算公差值：
````cpp
double calcEpsilon();                           //返回公差值
````
`calcEpsilon`清楚的表明它返回一个`double`，但是假设你知道对于这个程序来说使用`float`的精度已经足够了，而且你很关心`double`和`float`的大小。你可以声明一个`float`变量储存`calEpsilon`的计算结果。

````cpp
float ep = calcEpsilon();                       //double到float隐式转换
````
但是这几乎没有表明“我确实要减少函数返回值的精度”。使用显式类型初始器惯用法我们可以这样：
````cpp
auto ep = static_cast<float>(calcEpsilon());
````
出于同样的原因，如果你故意想用整数类型存储一个表达式返回的浮点数类型的结果，你也可以使用这个方法。假如你需要计算一个随机访问迭代器（比如`std::vector`，`std::deque`或者`std::array`）中某元素的下标，你被提供一个`0.0`到`1.0`的`double`值表明这个元素离容器的头部有多远（`0.5`意味着位于容器中间）。进一步假设你很自信结果下标是`int`。如果容器是`c`，`d`是`double`类型变量，你可以用这样的方法计算容器下标：
````cpp
int index = d * c.size();
````
但是这种写法并没有明确表明你想将右侧的`double`类型转换成`int`类型，显式类型初始器可以帮助你正确表意：
````cpp
auto index = static_cast<int>(d * size());
````

**请记住：**

+ 不可见的代理类可能会使`auto`从表达式中推导出“错误的”类型
+ 显式类型初始器惯用法强制`auto`推导出你想要的结果
