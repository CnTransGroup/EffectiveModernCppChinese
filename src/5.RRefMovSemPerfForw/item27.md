## 条款二十七：熟悉通用引用重载的替代方法

**Item 27: Familiarize yourself with alternatives to overloading on universal references**

[Item26](../5.RRefMovSemPerfForw/item26.md)中说明了对使用通用引用形参的函数，无论是独立函数还是成员函数（尤其是构造函数），进行重载都会导致一系列问题。但是也提供了一些示例，如果能够按照我们期望的方式运行，重载可能也是有用的。这个条款探讨了几种，通过避免在通用引用上重载的设计，或者通过限制通用引用可以匹配的参数类型，来实现所期望行为的方法。

讨论基于[Item26](../5.RRefMovSemPerfForw/item26.md)中的示例，如果你还没有阅读那个条款，请先阅读那个条款再继续。

### 放弃重载

在[Item26](../5.RRefMovSemPerfForw/item26.md)中的第一个例子中，`logAndAdd`是许多函数的代表，这些函数可以使用不同的名字来避免在通用引用上的重载的弊端。例如两个重载的`logAndAdd`函数，可以分别改名为`logAndAddName`和`logAndAddNameIdx`。但是，这种方式不能用在第二个例子，`Person`构造函数中，因为构造函数的名字被语言固定了（译者注：即构造函数名与类名相同）。此外谁愿意放弃重载呢？

### 传递const T&

一种替代方案是退回到C++98，然后将传递通用引用替换为传递lvalue-refrence-to-`const`。事实上，这是[Item26](../5.RRefMovSemPerfForw/item26.md)中首先考虑的方法。缺点是效率不高。现在我们知道了通用引用和重载的相互关系，所以放弃一些效率来确保行为正确简单可能也是一种不错的折中。

### 传值

通常在不增加复杂性的情况下提高性能的一种方法是，将按传引用形参替换为按值传递，这是违反直觉的。该设计遵循[Item41](../8.Tweaks/item41.md)中给出的建议，即在你知道要拷贝时就按值传递，因此会参考那个条款来详细讨论如何设计与工作，效率如何。这里，在`Person`的例子中展示：

```cpp
class Person {
public:
    explicit Person(std::string n)  //代替T&&构造函数，
    : name(std::move(n)) {}         //std::move的使用见条款41
  
    explicit Person(int idx)        //同之前一样
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

因为没有`std::string`构造函数可以接受整型参数，所有`int`或者其他整型变量（比如`std::size_t`、`short`、`long`等）都会使用`int`类型重载的构造函数。相似的，所有`std::string`类似的实参（还有可以用来创建`std::string`的东西，比如字面量“`Ruth`”等）都会使用`std::string`类型的重载构造函数。没有意外情况。我想你可能会说有些人使用`0`或者`NULL`指代空指针会调用`int`重载的构造函数让他们很吃惊，但是这些人应该参考[Item8](../3.MovingToModernCpp/item8.md)反复阅读直到使用`0`或者`NULL`作为空指针让他们恶心。

### 使用*tag dispatch*

传递lvalue-reference-to-`const`以及按值传递都不支持完美转发。如果使用通用引用的动机是完美转发，我们就只能使用通用引用了，没有其他选择。但是又不想放弃重载。所以如果不放弃重载又不放弃通用引用，如何避免在通用引用上重载呢？

实际上并不难。通过查看所有重载的所有形参以及调用点的所有传入实参，然后选择最优匹配的函数——考虑所有形参/实参的组合。通用引用通常提供了最优匹配，但是如果通用引用是包含其他**非**通用引用的形参列表的一部分，则非通用引用形参的较差匹配会使有一个通用引用的重载版本不被运行。这就是*tag dispatch*方法的基础，下面的示例会使这段话更容易理解。

我们将标签分派应用于`logAndAdd`例子，下面是原来的代码，以免你再分心回去查看：

```cpp
std::multiset<std::string> names;       //全局数据结构

template<typename T>                    //志记信息，将name添加到数据结构
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clokc::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

就其本身而言，功能执行没有问题，但是如果引入一个`int`类型的重载来用索引查找对象，就会重新陷入[Item26](../5.RRefMovSemPerfForw/item26.md)中描述的麻烦。这个条款的目标是避免它。不通过重载，我们重新实现`logAndAdd`函数分拆为两个函数，一个针对整型值，一个针对其他。`logAndAdd`本身接受所有实参类型，包括整型和非整型。

这两个真正执行逻辑的函数命名为`logAndAddImpl`，即我们使用重载。其中一个函数接受通用引用。所以我们同时使用了重载和通用引用。但是每个函数接受第二个形参，表征传入的实参是否为整型。这第二个形参可以帮助我们避免陷入到[Item26](../5.RRefMovSemPerfForw/item26.md)中提到的麻烦中，因为我们将其安排为第二个实参决定选择哪个重载函数。

是的，我知道，“不要在啰嗦了，赶紧亮出代码”。没有问题，代码如下，这是最接近正确版本的：

```cpp
template<typename T>
void logAndAdd(T&& name) 
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral<T>());   //不那么正确
}
```

这个函数转发它的形参给`logAndAddImpl`函数，但是多传递了一个表示形参`T`是否为整型的实参。至少，这就是应该做的。对于右值的整型实参来说，这也是正确的。但是如同[Item28](../5.RRefMovSemPerfForw/item28.md)中说明，如果左值实参传递给通用引用`name`，对`T`类型推断会得到左值引用。所以如果左值`int`被传入`logAndAdd`，`T`将被推断为`int&`。这不是一个整型类型，因为引用不是整型类型。这意味着`std::is_integral<T>`对于任何左值实参返回false，即使确实传入了整型值。

意识到这个问题基本相当于解决了它，因为C++标准库有一个*type trait*（参见[Item9](../3.MovingToModernCpp/item9.md)），`std::remove_reference`，函数名字就说明做了我们希望的：移除类型的引用说明符。所以正确实现的代码应该是这样：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(
        std::forward<T>(name),
        std::is_integral<typename std::remove_reference<T>::type>()
    );
}
```

这个代码很巧妙。（在C++14中，你可以通过`std::remove_reference_t<T>`来简化写法，参看[Item9](../3.MovingToModernCpp/item9.md)）

处理完之后，我们可以将注意力转移到名为`logAndAddImpl`的函数上了。有两个重载函数，第一个仅用于非整型类型（即`std::is_integral<typename std::remove_reference<T>::type>`是false）：

```cpp
template<typename T>                            //非整型实参：添加到全局数据结构中
void logAndAddImpl(T&& name, std::false_type)	//译者注：高亮std::false_type
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

一旦你理解了高亮参数的含义，代码就很直观。概念上，`logAndAdd`传递一个布尔值给`logAndAddImpl`表明是否传入了一个整型类型，但是`true`和`false`是**运行时**值，我们需要使用重载决议——**编译时**决策——来选择正确的`logAndAddImpl`重载。这意味着我们需要一个**类型**对应`true`，另一个不同的类型对应`false`。这个需要是经常出现的，所以标准库提供了这样两个命名`std::true_type`和`std::false_type`。`logAndAdd`传递给`logAndAddImpl`的实参是个对象，如果`T`是整型，对象的类型就继承自`std::true_type`，反之继承自`std::false_type`。最终的结果就是，当`T`不是整型类型时，这个`logAndAddImpl`重载是个可供调用的候选者。

第二个重载覆盖了相反的场景：当`T`是整型类型。在这个场景中，`logAndAddImpl`简单找到对应传入索引的名字，然后传递给`logAndAdd`：

```cpp
std::string nameFromIdx(int idx);           //与条款26一样，整型实参：查找名字并用它调用logAndAdd
void logAndAddImpl(int idx, std::true_type) //译者注：高亮std::true_type
{
  logAndAdd(nameFromIdx(idx)); 
}
```

通过索引找到对应的`name`，然后让`logAndAddImpl`传递给`logAndAdd`（名字会被再`std::forward`给另一个`logAndAddImpl`重载），我们避免了将日志代码放入这个`logAndAddImpl`重载中。

在这个设计中，类型`std::true_type`和`std::false_type`是“标签”（tag），其唯一目的就是强制重载解析按照我们的想法来执行。注意到我们甚至没有对这些参数进行命名。他们在运行时毫无用处，事实上我们希望编译器可以意识到这些标签形参没被使用，然后在程序执行时优化掉它们。（至少某些时候有些编译器会这样做。）通过创建标签对象，在`logAndAdd`内部将重载实现函数的调用“分发”（*dispatch*）给正确的重载。因此这个设计名称为：*tag dispatch*。这是模板元编程的标准构建模块，你对现代C++库中的代码了解越多，你就会越多遇到这种设计。

就我们的目的而言，*tag dispatch*的重要之处在于它可以允许我们组合重载和通用引用使用，而没有[Item26](../5.RRefMovSemPerfForw/item26.md)中提到的问题。分发函数——`logAndAdd`——接受一个没有约束的通用引用参数，但是这个函数没有重载。实现函数——`logAndAddImpl`——是重载的，一个接受通用引用参数，但是重载规则不仅依赖通用引用形参，还依赖新引入的标签形参，标签值设计来保证有不超过一个的重载是合适的匹配。结果是标签来决定采用哪个重载函数。通用引用参数可以生成精确匹配的事实在这里并不重要。（译者注：这里确实比较啰嗦，如果理解了上面的内容，这段完全可以没有。）

### 约束使用通用引用的模板

*tag dispatch*的关键是存在单独一个函数（没有重载）给客户端API。这个单独的函数分发给具体的实现函数。创建一个没有重载的分发函数通常是容易的，但是[Item26](../5.RRefMovSemPerfForw/item26.md)中所述第二个问题案例是`Person`类的完美转发构造函数，是个例外。编译器可能会自行生成拷贝和移动构造函数，所以即使你只写了一个构造函数并在其中使用*tag dispatch*，有一些对构造函数的调用也被编译器生成的函数处理，绕过了分发机制。

实际上，真正的问题不是编译器生成的函数会绕过*tag dispatch*设计，而是不**总**会绕过去。你希望类的拷贝构造函数总是处理该类型的左值拷贝请求，但是如同[Item26](../5.RRefMovSemPerfForw/item26.md)中所述，提供具有通用引用的构造函数，会使通用引用构造函数在拷贝non-`const`左值时被调用（而不是拷贝构造函数）。那个条款还说明了当一个基类声明了完美转发构造函数，派生类实现自己的拷贝和移动构造函数时会调用那个完美转发构造函数，尽管正确的行为是调用基类的拷贝或者移动构造。

这种情况，采用通用引用的重载函数通常比期望的更加贪心，虽然不像单个分派函数一样那么贪心，而又不满足使用*tag dispatch*的条件。你需要另外的技术，可以让你确定允许使用通用引用模板的条件。朋友，你需要的就是`std::enable_if`。

`std::enable_if`可以给你提供一种强制编译器执行行为的方法，像是特定模板不存在一样。这种模板被称为被**禁止**（disabled）。默认情况下，所有模板是**启用**的（enabled），但是使用`std::enable_if`可以使得仅在`std::enable_if`指定的条件满足时模板才启用。在这个例子中，我们只在传递的类型不是`Person`时使用`Person`的完美转发构造函数。如果传递的类型是`Person`，我们要禁止完美转发构造函数（即让编译器忽略它），因为这会让拷贝或者移动构造函数处理调用，这是我们想要使用`Person`初始化另一个`Person`的初衷。

这个主意听起来并不难，但是语法比较繁杂，尤其是之前没有接触过的话，让我慢慢引导你。有一些`std::enbale_if`的contidion（条件）部分的样板，让我们从这里开始。下面的代码是`Person`完美转发构造函数的声明，多展示`std::enable_if`的部分来简化使用难度。我仅展示构造函数的声明，因为`std::enable_if`的使用对函数实现没影响。实现部分跟[Item26](../5.RRefMovSemPerfForw/item26.md)中没有区别。

```cpp
class Person {
public:
    template<typename T,
             typename = typename std::enable_if<condition>::type>   //译者注：本行高亮，condition为某其他特定条件
    explicit Person(T&& n);
    …
};
```

为了理解高亮部分发生了什么，我很遗憾的表示你要自行参考其他代码，因为详细解释需要花费一定空间和时间，而本书并没有足够的空间（在你自行学习过程中，请研究“SFINAE”以及`std::enable_if`，因为“SFINAE”就是使`std::enable_if`起作用的技术）。这里我想要集中讨论条件的表示，该条件表示此构造函数是否启用。

这里我们想表示的条件是确认`T`不是`Person`类型，即模板构造函数应该在`T`不是`Person`类型的时候启用。多亏了*type trait*可以确定两个对象类型是否相同（`std::is_same`），看起来我们需要的就是`!std::is_same<Person, T>::value`（注意语句开始的`!`，我们想要的是**不**相同）。这很接近我们想要的了，但是不完全正确，因为如同[Item28](../5.RRefMovSemPerfForw/item28.md)中所述，使用左值来初始化通用引用的话会推导成左值引用，比如这个代码:

```cpp
Person p("Nancy");
auto cloneOfP(p);       //用左值初始化
```

`T`的类型在通用引用的构造函数中被推导为`Person&`。`Person`和`Person&`类型是不同的，`std::is_same`的结果也反映了：`std::is_same<Person, Person&>::value`是false。

如果我们更精细考虑仅当`T`不是`Person`类型才启用模板构造函数，我们会意识到当我们查看`T`时，应该忽略：

- **是否是个引用**。对于决定是否通用引用构造函数启用的目的来说，`Person`，`Person&`，`Person&&`都是跟`Person`一样的。
- **是不是`const`或者`volatile`**。如上所述，`const Person`，`volatile Person` ，`const volatile Person`也是跟`Person`一样的。

这意味着我们需要一种方法消除对于`T`的引用，`const`，`volatile`修饰。再次，标准库提供了这样功能的*type trait*，就是`std::decay`。`std::decay<T>::type`与`T`是相同的，只不过会移除引用和cv限定符（*cv-qualifiers*，即`const`或`volatile`标识符）的修饰。（这里我没有说出另外的真相，`std::decay`如同其名一样，可以将数组或者函数退化成指针，参考[Item1](../1.DeducingTypes/item1.md)，但是在这里讨论的问题中，它刚好合适）。我们想要控制构造函数是否启用的条件可以写成：

```cpp
!std::is_same<Person, typename std::decay<T>::type>::value
```

即`Person`和`T`的类型不同，忽略了所有引用和cv限定符。（如[Item9](../3.MovingToModernCpp/item9.md)所述，`std::decay`前的“`typename`”是必需的，因为`std::decay<T>::type`的类型取决于模板形参`T`。）

将其带回上面`std::enable_if`样板的代码中，加上调整一下格式，让各部分如何组合在一起看起来更容易，`Person`的完美转发构造函数的声明如下：

```cpp
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_same<Person, 
                                     typename std::decay<T>::type
                                    >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
```

如果你之前从没有看到过这种类型的代码，那你可太幸福了。最后才放出这种设计是有原因的。当你有其他机制来避免同时使用重载和通用引用时（你总会这样做），确实应该那样做。不过，一旦你习惯了使用函数语法和尖括号的使用，也不坏。此外，这可以提供你一直想要的行为表现。在上面的声明中，使用`Person`初始化一个`Person`——无论是左值还是右值，`const`还是non-`const`，`volatile`还是non-`volatile`——都不会调用到通用引用构造函数。

成功了，对吗？确实！

啊，不对。等会再庆祝。[Item26](../5.RRefMovSemPerfForw/item26.md)还有一个情景需要解决，我们需要继续探讨下去。

假定从`Person`派生的类以常规方式实现拷贝和移动操作：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }
    
    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
    
    …
};
```

这和[Item26](../5.RRefMovSemPerfForw/item26.md)中的代码是一样的，包括注释也是一样。当我们拷贝或者移动一个`SpecialPerson`对象时，我们希望调用基类对应的拷贝和移动构造函数，来拷贝或者移动基类部分，但是这里，我们将`SpecialPerson`传递给基类的构造函数，因为`SpecialPerson`和`Person`类型不同（在应用`std::decay`后也不同），所以完美转发构造函数是启用的，会实例化为精确匹配`SpecialPerson`实参的构造函数。相比于派生类到基类的转化——这个转化对于在`Person`拷贝和移动构造函数中把`SpecialPerson`对象绑定到`Person`形参非常重要，生成的精确匹配是更优的，所以这里的代码，拷贝或者移动`SpecialPerson`对象就会调用`Person`类的完美转发构造函数来执行基类的部分。跟[Item26](../5.RRefMovSemPerfForw/item26.md)的困境一样。

派生类仅仅是按照常规的规则生成了自己的移动和拷贝构造函数，所以这个问题的解决还要落实在基类，尤其是控制是否使用`Person`通用引用构造函数启用的条件。现在我们意识到不只是禁止`Person`类型启用模板构造函数，而是禁止`Person`**以及任何派生自`Person`**的类型启用模板构造函数。讨厌的继承！

你应该不意外在这里看到标准库中也有*type trait*判断一个类型是否继承自另一个类型，就是`std::is_base_of`。如果`std::is_base_of<T1, T2>`是true就表示`T2`派生自`T1`。类型也可被认为是从他们自己派生，所以`std::is_base_of<T, T>::value`总是true。这就很方便了，我们想要修正控制`Person`完美转发构造函数的启用条件，只有当`T`在消除引用和cv限定符之后，并且既不是`Person`又不是`Person`的派生类时，才满足条件。所以使用`std::is_base_of`代替`std::is_same`就可以了：

```cpp
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_base_of<Person, 
                                        typename std::decay<T>::type
                                       >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
```

现在我们终于完成了最终版本。这是C++11版本的代码，如果我们使用C++14，这份代码也可以工作，但是可以使用`std::enable_if`和`std::decay`的别名模板来少写“`typename`”和“`::type`”这样的麻烦东西，产生了下面这样看起来舒爽的代码：

```cpp
class Person  {                                         //C++14
public:
    template<
        typename T,
        typename = std::enable_if_t<                    //这儿更少的代码
                       !std::is_base_of<Person,
                                        std::decay_t<T> //还有这儿
                                       >::value
                   >                                    //还有这儿
    >
    explicit Person(T&& n);
    …
};
```

好了，我承认，我又撒谎了。我们还没有完成，但是越发接近最终版本了。非常接近，我保证。

我们已经知道如何使用`std::enable_if`来选择性禁止`Person`通用引用构造函数，来使得一些实参类型确保使用到拷贝或者移动构造函数，但是我们还没将其应用于区分整型参数和非整型参数。毕竟，我们的原始目标是解决构造函数模糊性问题。

我们需要的所有东西——我确实意思是所有——是（1）加入一个`Person`构造函数重载来处理整型参数；（2）约束模板构造函数使其对于某些实参禁用。使用这些我们讨论过的技术组合起来，就能解决这个问题了：

```cpp
class Person {
public:
    template<
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)          //对于std::strings和可转化为
    : name(std::forward<T>(n))      //std::strings的实参的构造函数
    { … }

    explicit Person(int idx)        //对于整型实参的构造函数
    : name(nameFromIdx(idx))
    { … }

    …                               //拷贝、移动构造函数等

private:
    std::string name;
};
```

看！多么优美！好吧，优美之处只是对于那些迷信模板元编程之人，但是确实提出了不仅能工作的方法，而且极具技巧。因为使用了完美转发，所以具有最大效率，因为控制了通用引用与重载的结合而不是禁止它，这种技术可以被用于不可避免要用重载的情况（比如构造函数）。

### 折中

本条款提到的前三个技术——放弃重载、传递const T&、传值——在函数调用中指定每个形参的类型。后两个技术——*tag dispatch*和限制模板适用范围——使用完美转发，因此不需要指定形参类型。这一基本决定（是否指定类型）有一定后果。

通常，完美转发更有效率，因为它避免了仅仅去为了符合形参声明的类型而创建临时对象。在`Person`构造函数的例子中，完美转发允许将“`Nancy`”这种字符串字面量转发到`Person`内部的`std::string`的构造函数，不使用完美转发的技术则会从字符串字面值创建一个临时`std::string`对象，来满足`Person`构造函数指定的形参要求。

但是完美转发也有缺点。即使某些类型的实参可以传递给接受特定类型的函数，也无法完美转发。[Item30](../5.RRefMovSemPerfForw/item30.md)中探索了完美转发失败的例子。

第二个问题是当客户传递无效参数时错误消息的可理解性。例如假如客户传递了一个由`char16_t`（一种C++11引入的类型表示16位字符）而不是`char`（`std::string`包含的）组成的字符串字面值来创建一个`Person`对象：

```cpp
Person p(u"Konrad Zuse");   //“Konrad Zuse”由const char16_t类型字符组成
```

使用本条款中讨论的前三种方法，编译器将看到可用的采用`int`或者`std::string`的构造函数，它们或多或少会产生错误消息，表示没有可以从`const char16_t[12]`转换为`int`或者`std::string`的方法。

但是，基于完美转发的方法，`const char16_t`不受约束地绑定到构造函数的形参。从那里将转发到`Person`的`std::string`数据成员的构造函数，在这里，调用者传入的内容（`const char16_t`数组）与所需内容（`std::string`构造函数可接受的类型）发生的不匹配会被发现。由此产生的错误消息会让人更印象深刻，在我使用的编译器上，会产生超过160行错误信息。

在这个例子中，通用引用仅被转发一次（从`Person`构造函数到`std::string`构造函数），但是更复杂的系统中，在最终到达判断实参类型是否可接受的地方之前，通用引用会被多层函数调用转发。通用引用被转发的次数越多，产生的错误消息偏差就越大。许多开发者发现，这种特殊问题是发生在留有通用引用形参的接口上，这些接口以性能作为首要考虑点。

在`Person`这个例子中，我们知道完美转发函数的通用引用形参要作为`std::string`的初始化器，所以我们可以用`static_assert`来确认它可以起这个作用。`std::is_constructible`这个*type trait*执行编译时测试，确定一个类型的对象是否可以用另一个不同类型（或多个类型）的对象（或多个对象）来构造，所以代码可以这样：

```cpp
class Person {
public:
    template<                       //同之前一样
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)
    : name(std::forward<T>(n))
    {
        //断言可以用T对象创建std::string
        static_assert(
        std::is_constructible<std::string, T>::value,
        "Parameter n can't be used to construct a std::string"
        );

        …               //通常的构造函数的工作写在这

    }
    
    …                   //Person类的其他东西（同之前一样）
};
```

如果客户代码尝试使用无法构造`std::string`的类型创建`Person`，会导致指定的错误消息。不幸的是，在这个例子中，`static_assert`在构造函数体中，但是转发的代码作为成员初始化列表的部分在检查之前。所以我使用的编译器，结果是由`static_assert`产生的清晰的错误消息在常规错误消息（多达160行以上那个）后出现。

**请记住：**

- 通用引用和重载的组合替代方案包括使用不同的函数名，通过lvalue-reference-to-`const`传递形参，按值传递形参，使用*tag dispatch*。
- 通过`std::enable_if`约束模板，允许组合通用引用和重载使用，但它也控制了编译器在哪种条件下才使用通用引用重载。
- 通用引用参数通常具有高效率的优势，但是可用性就值得斟酌。
