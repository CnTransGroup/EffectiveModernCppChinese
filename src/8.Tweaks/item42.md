## 条款四十二：考虑使用置入代替插入

**Item 42: Consider emplacement instead of insertion**

如果你拥有一个容器，例如放着`std::string`，那么当你通过插入（insertion）函数（例如`insert`，`push_front`，`push_back`，或者对于`std::forward_list`来说是`insert_after`）添加新元素时，你传入的元素类型应该是`std::string`。毕竟，这就是容器里的内容。

逻辑上看来如此，但是并非总是如此。考虑如下代码：

```cpp
std::vector<std::string> vs;        //std::string的容器
vs.push_back("xyzzy");              //添加字符串字面量
```

这里，容器里内容是`std::string`，但是你有的——你实际上试图通过`push_back`加入的——是字符串字面量，即引号内的字符序列。字符串字面量并不是`std::string`，这意味着你传递给`push_back`的实参并不是容器里的内容类型。

`std::vector`的`push_back`被按左值和右值分别重载：

```cpp
template <class T,                  //来自C++11标准
          class Allocator = allocator<T>>
class vector {
public:
    …
    void push_back(const T& x);     //插入左值
    void push_back(T&& x);          //插入右值
    …
};
```

在

```cpp
vs.push_back("xyzzy");
```

这个调用中，编译器看到实参类型（`const char[6]`）和`push_back`采用的形参类型（`std::string`的引用）之间不匹配。它们通过从字符串字面量创建一个`std::string`类型的临时对象来消除不匹配，然后传递临时变量给`push_back`。换句话说，编译器处理的这个调用应该像这样：

```cpp
vs.push_back(std::string("xyzzy")); //创建临时std::string，把它传给push_back
```

代码可以编译并运行，皆大欢喜。除了对于性能执着的人意识到了这份代码不如预期的执行效率高。

为了在`std::string`容器中创建新元素，调用了`std::string`的构造函数，但是这份代码并不仅调用了一次构造函数，而是调用了两次，而且还调用了`std::string`析构函数。下面是在`push_back`运行时发生了什么：

1. 一个`std::string`的临时对象从字面量“`xyzzy`”被创建。这个对象没有名字，我们可以称为`temp`。`temp`的构造是第一次`std::string`构造。因为是临时变量，所以`temp`是右值。
2. `temp`被传递给`push_back`的右值重载函数，绑定到右值引用形参`x`。在`std::vector`的内存中一个`x`的形参被创建。这次构造——也是第二次构造——在`std::vector`内部真正创建一个对象。（构造`x`使用的是移动构造函数，因为`x`在它被拷贝前被转换为一个右值，成为右值引用。有关将右值引用形参强制转换为右值的信息，请参见[Item25](../5.RRefMovSemPerfForw/item25.md)）。
3. 在`push_back`返回之后，`temp`立刻被销毁，调用了一次`std::string`的析构函数。

对于性能执着的人不禁注意到是否存在一种方法可以获取字符串字面量并将其直接传入到步骤2里在`std::vector`内构造`std::string`的代码中，可以避免临时对象`temp`的创建与销毁。这样的效率最好，对于性能执着的人也不会有什么意见了。

因为你是一个C++开发者，所以你更大可能是一个对性能执着的人。如果你不是C++开发者，你可能也会同意这个观点。（如果你根本不考虑性能，为什么你没在用Python？）所以我很高兴告诉你有一种方法，恰是在调用`push_back`中实现效率最大化。它不叫`push_back`。`push_back`函数不正确。你需要的是`emplace_back`。

`emplace_back`就是像我们想要的那样做的：使用传递给它的任何实参直接在`std::vector`内部构造一个`std::string`。没有临时变量会生成：

```cpp
vs.emplace_back("xyzzy");           //直接用“xyzzy”在vs内构造std::string
```

`emplace_back`使用完美转发，因此只要你没有遇到完美转发的限制（参见[Item30](../5.RRefMovSemPerfForw/item30.md)），就可以传递任何实参以及组合到`emplace_back`。比如，如果你想通过接受一个字符和一个数量的`std::string`构造函数，在`vs`中创建一个`std::string`，代码如下：

```cpp
vs.emplace_back(50, 'x');           //插入由50个“x”组成的一个std::string
```

`emplace_back`可以用于每个支持`push_back`的标准容器。类似的，每个支持`push_front`的标准容器都支持`emplace_front`。每个支持`insert`（除了`std::forward_list`和`std::array`）的标准容器支持`emplace`。关联容器提供`emplace_hint`来补充接受“hint”迭代器的`insert`函数，`std::forward_list`有`emplace_after`来匹配`insert_after`。

使得置入（emplacement）函数功能优于插入函数的原因是它们有灵活的接口。插入函数接受**对象**去插入，而置入函数接受**对象的构造函数接受的实参**去插入。这种差异允许置入函数避免插入函数所必需的临时对象的创建和销毁。

因为可以传递容器内元素类型的实参给置入函数（因此该实参使函数执行复制或者移动构造函数），所以在插入函数不会构造临时对象的情况，也可以使用置入函数。在这种情况下，插入和置入函数做的是同一件事，比如：

```cpp
std::string queenOfDisco("Donna Summer");
```

下面的调用都是可行的，对容器的实际效果也一样：

```cpp
vs.push_back(queenOfDisco);         //拷贝构造queenOfDisco
vs.emplace_back(queenOfDisco);      //同上
```

因此，置入函数可以完成插入函数的所有功能。并且有时效率更高，至少在理论上，不会更低效。那为什么不在所有场合使用它们？

因为，就像说的那样，只是“理论上”，在理论和实际上没有什么区别，但是实际上区别还是有的。在当前标准库的实现下，有些场景，就像预期的那样，置入执行性能优于插入，但是，有些场景反而插入更快。这种场景不容易描述，因为依赖于传递的实参的类型、使用的容器、置入或插入到容器中的位置、容器中类型的构造函数的异常安全性，和对于禁止重复值的容器（即`std::set`，`std::map`，`std::unordered_set`，`set::unordered_map`）要添加的值是否已经在容器中。因此，大致的调用建议是：通过benchmark测试来确定置入和插入哪种更快。

当然这个结论不是很令人满意，所以你会很高兴听到还有一种启发式的方法来帮助你确定是否应该使用置入。如果下列条件都能满足，置入会优于插入：

- **值是通过构造函数添加到容器，而不是直接赋值。** 例子就像本条款刚开始的那样（用“`xyzzy`”添加`std::string`到`std::vector`容器`vs`中），值添加到`vs`末尾——一个先前没有对象存在的地方。新值必须通过构造函数添加到`std::vector`。如果我们回看这个例子，新值放到已经存在了对象的一个地方，那情况就完全不一样了。考虑下：

  ```cpp
  std::vector<std::string> vs;        //跟之前一样
  …                                   //添加元素到vs
  vs.emplace(vs.begin(), "xyzzy");    //添加“xyzzy”到vs头部
  ```

  对于这份代码，没有实现会在已经存在对象的位置`vs[0]`构造这个添加的`std::string`。而是，通过移动赋值的方式添加到需要的位置。但是移动赋值需要一个源对象，所以这意味着一个临时对象要被创建，而置入优于插入的原因就是没有临时对象的创建和销毁，所以当通过赋值操作添加元素时，置入的优势消失殆尽。

  而且，向容器添加元素是通过构造还是赋值通常取决于实现者。但是，启发式仍然是有帮助的。基于节点的容器实际上总是使用构造添加新元素，大多数标准库容器都是基于节点的。例外的容器只有`std::vector`，`std::deque`，`std::string`。（`std::array`也不是基于节点的，但是它不支持置入和插入，所以它与这儿无关。）在不是基于节点的容器中，你可以依靠`emplace_back`来使用构造向容器添加元素，对于`std::deque`，`emplace_front`也是一样的。

- **传递的实参类型与容器的初始化类型不同。** 再次强调，置入优于插入通常基于以下事实：当传递的实参不是容器保存的类型时，接口不需要创建和销毁临时对象。当将类型为`T`的对象添加到`container<T>`时，没有理由期望置入比插入运行的更快，因为不需要创建临时对象来满足插入的接口。

- **容器不拒绝重复项作为新值。** 这意味着容器要么允许添加重复值，要么你添加的元素大部分都是不重复的。这样要求的原因是为了判断一个元素是否已经存在于容器中，置入实现通常会创建一个具有新值的节点，以便可以将该节点的值与现有容器中节点的值进行比较。如果要添加的值不在容器中，则链接该节点。然后，如果值已经存在，置入操作取消，创建的节点被销毁，意味着构造和析构时的开销被浪费了。这样的节点更多的是为置入函数而创建，相比起为插入函数来说。

本条款开始的例子中下面的调用满足上面的条件。所以`emplace_back`比`push_back`运行更快。

```cpp
vs.emplace_back("xyzzy");              //在容器末尾构造新值；不是传递的容器中元
                                       //素的类型；没有使用拒绝重复项的容器

vs.emplace_back(50, 'x');              //同上
```

在决定是否使用置入函数时，需要注意另外两个问题。首先是资源管理。假定你有一个盛放`std::shared_ptr<Widget>`s的容器，

```cpp
std::list<std::shared_ptr<Widget>> ptrs;
```

然后你想添加一个通过自定义删除器释放的`std::shared_ptr`（参见[Item19](../4.SmartPointers/item19.md)）。[Item21](../4.SmartPointers/item21.md)说明你应该使用`std::make_shared`来创建`std::shared_ptr`，但是它也承认有时你无法做到这一点。比如当你要指定一个自定义删除器时。这时，你必须直接`new`一个原始指针，然后通过`std::shared_ptr`来管理。

如果自定义删除器是这个函数，

```cpp
void killWidget(Widget* pWidget);
```

使用插入函数的代码如下：

```cpp
ptrs.push_back(std::shared_ptr<Widget>(new Widget, killWidget));
```

也可以像这样：

```cpp
ptrs.push_back({new Widget, killWidget});
```

不管哪种写法，在调用`push_back`前会生成一个临时`std::shared_ptr`对象。`push_back`的形参是`std::shared_ptr`的引用，因此必须有一个`std::shared_ptr`。

用`emplace_back`应该可以避免`std::shared_ptr`临时对象的创建，但是在这个场景下，临时对象值得被创建。考虑如下可能的时间序列：

1. 在上述的调用中，一个`std::shared_ptr<Widget>`的临时对象被创建来持有“`new Widget`”返回的原始指针。称这个对象为`temp`。
2. `push_back`通过引用接受`temp`。在存储`temp`的副本的*list*节点的内存分配过程中，内存溢出异常被抛出。
3. 随着异常从`push_back`的传播，`temp`被销毁。作为唯一管理这个`Widget`的`std::shared_ptr`，它自动销毁`Widget`，在这里就是调用`killWidget`。

这样的话，即使发生了异常，没有资源泄漏：在调用`push_back`中通过“`new Widget`”创建的`Widget`在`std::shared_ptr`管理下自动销毁。生命周期良好。

考虑使用`emplace_back`代替`push_back`：

```cpp
ptrs.emplace_back(new Widget, killWidget);
```

1. 通过`new Widget`创建的原始指针完美转发给`emplace_back`中，*list*节点被分配的位置。如果分配失败，还是抛出内存溢出异常。
2. 当异常从`emplace_back`传播，原始指针是仅有的访问堆上`Widget`的途径，但是因为异常而丢失了，那个`Widget`的资源（以及任何它所拥有的资源）发生了泄漏。

在这个场景中，生命周期不良好，这个失误不能赖`std::shared_ptr`。使用带自定义删除器的`std::unique_ptr`也会有同样的问题。根本上讲，像`std::shared_ptr`和`std::unique_ptr`这样的资源管理类的高效性是以资源（比如从`new`来的原始指针）被**立即**传递给资源管理对象的构造函数为条件的。实际上，`std::make_shared`和`std::make_unique`这样的函数自动做了这些事，是使它们如此重要的原因。

在对存储资源管理类对象的容器（比如`std::list<std::shared_ptr<Widget>>`）调用插入函数时，函数的形参类型通常确保在资源的获取（比如使用`new`）和资源管理对象的创建之间没有其他操作。在置入函数中，完美转发推迟了资源管理对象的创建，直到可以在容器的内存中构造它们为止，这给“异常导致资源泄漏”提供了可能。所有的标准库容器都容易受到这个问题的影响。在使用资源管理对象的容器时，必须注意确保在使用置入函数而不是插入函数时，不会为提高效率带来的降低异常安全性付出代价。

坦白说，无论如何，你不应该将“`new Widget`”之类的表达式传递给`emplace_back`或者`push_back`或者大多数这种函数，因为，就像[Item21](../4.SmartPointers/item21.md)中解释的那样，这可能导致我们刚刚讨论的异常安全性问题。消除资源泄漏可能性的方法是，使用独立语句把从“`new Widget`”获取的指针传递给资源管理类对象，然后这个对象作为右值传递给你本来想传递“`new Widget`”的函数（[Item21](../4.SmartPointers/item21.md)有这个观点的详细讨论）。使用`push_back`的代码应该如下：

```cpp
std::shared_ptr<Widget> spw(new Widget,      //创建Widget，让spw管理它
                            killWidget);
ptrs.push_back(std::move(spw));              //添加spw右值
```

`emplace_back`的版本如下：

```cpp
std::shared_ptr<Widget> spw(new Widget, killWidget);
ptrs.emplace_back(std::move(spw));
```

无论哪种方式，都会产生`spw`的创建和销毁成本。选择置入而非插入的动机是避免容器元素类型的临时对象的开销。但是对于`spw`的概念来讲，当添加资源管理类型对象到容器中，并根据正确的方式确保在获取资源和连接到资源管理对象上之间无其他操作时，置入函数不太可能胜过插入函数。

置入函数的第二个值得注意的方面是它们与`explicit`的构造函数的交互。鉴于C++11对正则表达式的支持，假设你创建了一个正则表达式对象的容器：

```cpp
std::vector<std::regex> regexes;
```

由于你同事的打扰，你写出了如下看似毫无意义的代码：

```cpp
regexes.emplace_back(nullptr);           //添加nullptr到正则表达式的容器中？
```

你没有注意到错误，编译器也没有提示你，所以你浪费了大量时间来调试。突然，你发现你插入了空指针到正则表达式的容器中。但是这怎么可能？指针不是正则表达式，如果你试图下面这样写，

```cpp
std::regex r = nullptr;                  //错误！不能编译
```

编译器就会报错。有趣的是，如果你调用`push_back`而不是`emplace_back`，编译器也会报错：

```cpp
regexes.push_back(nullptr);              //错误！不能编译
```

当前你遇到的奇怪行为来源于“可能用字符串构造`std::regex`对象”的事实，这就意味着下面代码合法：

```cpp
std::regex upperCaseWorld("[A-Z]+");
```

通过字符串创建`std::regex`要求相对较长的运行时开销，所以为了最小程度减少无意中产生此类开销的可能性，采用`const char*`指针的`std::regex`构造函数是`explicit`的。这就是下面代码无法编译的原因：

```cpp
std::regex r = nullptr;                  //错误！不能编译
regexes.push_back(nullptr);              //错误
```

在上面的代码中，我们要求从指针到`std::regex`的隐式转换，但是构造函数的`explicit`ness拒绝了此类转换。

但是在`emplace_back`的调用中，我们没有说明要传递一个`std::regex`对象。然而，我们传递了一个`std::regex`**构造函数实参**。那不被认为是个隐式转换要求。相反，编译器看你像是写了如下代码：

```cpp
std::regex r(nullptr);                   //可编译
```

如果简洁的注释“可编译”缺乏直观理解，好的，因为这个代码可以编译，但是行为不确定。使用`const char*`指针的`std::regex`构造函数要求字符串是一个有效的正则表达式，空指针并不满足要求。如果你写出并编译了这样的代码，最好的希望就是运行时程序崩溃掉。如果你不幸运，就会花费大量的时间调试。

先把`push_back`，`emplace_back`放在一边，注意到相似的初始化语句导致了多么不一样的结果：

```cpp
std::regex r1 = nullptr;                 //错误！不能编译
std::regex r2(nullptr);                  //可以编译
```


在标准的官方术语中，用于初始化`r1`的语法（使用等号）是所谓的**拷贝初始化**。相反，用于初始化`r2`的语法是（使用小括号，有时也用花括号）被称为**直接初始化**。

```cpp
using regex   = basic_regex<char>;

explicit basic_regex(const char* ptr,flag_type flags); //定义 (1)explicit构造函数

basic_regex(const basic_regex& right); //定义 (2)拷贝构造函数
```
拷贝初始化不被允许使用`explicit`构造函数（译者注：即没法调用相应类的`explicit`拷贝构造函数）：对于`r1`,使用赋值运算符定义变量时将调用拷贝构造函数`定义 (2)`，其形参类型为`basic_regex&`。因此`nullptr`首先需要隐式装换为`basic_regex`。而根据`定义 (1)`中的`explicit`，这样的隐式转换不被允许，从而产生编译时期的报错。对于直接初始化，编译器会自动选择与提供的参数最匹配的构造函数，即`定义 (1)`。就是初始化`r1`不能编译，而初始化`r2`可以编译的原因。


然后回到`push_back`和`emplace_back`，更一般来说是，插入函数和置入函数的对比。置入函数使用直接初始化，这意味着可能使用`explicit`的构造函数。插入函数使用拷贝初始化，所以不能用`explicit`的构造函数。因此：

```cpp
regexes.emplace_back(nullptr);           //可编译。直接初始化允许使用接受指针的
                                         //std::regex的explicit构造函数
regexes.push_back(nullptr);              //错误！拷贝初始化不允许用那个构造函数
```

获得的经验是，当你使用置入函数时，请特别小心确保传递了正确的实参，因为即使是`explicit`的构造函数也会被编译器考虑，编译器会试图以有效方式解释你的代码。

**请记住：**

- 原则上，置入函数有时会比插入函数高效，并且不会更差。
- 实际上，当以下条件满足时，置入函数更快：（1）值被构造到容器中，而不是直接赋值；（2）传入的类型与容器的元素类型不一致；（3）容器不拒绝已经存在的重复值。
- 置入函数可能执行插入函数拒绝的类型转换。
