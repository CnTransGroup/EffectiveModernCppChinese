## 条款二十六：避免在通用引用上重载

**Item 26: Avoid overloading on universal references**

假定你需要写一个函数，它使用名字作为形参，打印当前日期和时间到日志中，然后将名字加入到一个全局数据结构中。你可能写出来这样的代码：

```cpp
std::multiset<std::string> names;           //全局数据结构
void logAndAdd(const std::string& name)
{
    auto now =                              //获取当前时间
        std::chrono::system_clock::now();
    log(now, "logAndAdd");                  //志记信息
    names.emplace(name);                    //把name加到全局数据结构中；
}                                           //emplace的信息见条款42
```

这份代码没有问题，但是同样的也没有效率。考虑这三个调用：

```cpp
std::string petName("Darla");
logAndAdd(petName);                     //传递左值std::string
logAndAdd(std::string("Persephone"));	//传递右值std::string
logAndAdd("Patty Dog");                 //传递字符串字面值
```

在第一个调用中，`logAndAdd`的形参`name`绑定到变量`petName`。在`logAndAdd`中`name`最终传给`names.emplace`。因为`name`是左值，会拷贝到`names`中。没有方法避免拷贝，因为是左值（`petName`）传递给`logAndAdd`的。

在第二个调用中，形参`name`绑定到右值（显式从“`Persephone`”创建的临时`std::string`）。`name`本身是个左值，所以它被拷贝到`names`中，但是我们意识到，原则上，它的值可以被移动到`names`中。本次调用中，我们有个拷贝代价，但是我们应该能用移动勉强应付。

在第三个调用中，形参`name`也绑定一个右值，但是这次是通过“`Patty Dog`”隐式创建的临时`std::string`变量。就像第二个调用中，`name`被拷贝到`names`，但是这里，传递给`logAndAdd`的实参是一个字符串字面量。如果直接将字符串字面量传递给`emplace`，就不会创建`std::string`的临时变量，而是直接在`std::multiset`中通过字面量构建`std::string`。在第三个调用中，我们有个`std::string`拷贝开销，但是我们连移动开销都不想要，更别说拷贝的。

我们可以通过使用通用引用（参见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)）重写`logAndAdd`来使第二个和第三个调用效率提升，按照[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)的说法，`std::forward`转发这个引用到`emplace`。代码如下：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");           //跟之前一样
logAndAdd(petName);                     //跟之前一样，拷贝右值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string
```

非常好，效率优化了！

在故事的最后，我们可以骄傲的交付这个代码，但是我还没有告诉你客户不总是有直接访问`logAndAdd`要求的名字的权限。有些客户只有索引，`logAndAdd`拿着索引在表中查找相应的名字。为了支持这些客户，`logAndAdd`需要重载为：

```cpp
std::string nameFromIdx(int idx);   //返回idx对应的名字

void logAndAdd(int idx)             //新的重载
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

之后的两个调用按照预期工作：

```cpp
std::string petName("Darla");           //跟之前一样

logAndAdd(petName);                     //跟之前一样，
logAndAdd(std::string("Persephone")); 	//这些调用都去调用
logAndAdd("Patty Dog");                 //T&&重载版本

logAndAdd(22);                          //调用int重载版本
```

事实上，这只能基本按照预期工作，假定一个客户将`short`类型索引传递给`logAndAdd`：

```cpp
short nameIdx;
…                                       //给nameIdx一个值
logAndAdd(nameIdx);                     //错误！
```

最后一行的注释并不清楚明白，下面让我来说明发生了什么。

有两个重载的`logAndAdd`。使用通用引用的那个推导出`T`的类型是`short`，因此可以精确匹配。对于`int`类型参数的重载也可以在`short`类型提升后匹配成功。根据正常的重载解决规则，精确匹配优先于类型提升的匹配，所以被调用的是通用引用的重载。

在通用引用那个重载中，`name`形参绑定到要传入的`short`上，然后`name`被`std::forward`给`names`（一个`std::multiset<std::string>`）的`emplace`成员函数，然后又被转发给`std::string`构造函数。`std::string`没有接受`short`的构造函数，所以`logAndAdd`调用里的`multiset::emplace`调用里的`std::string`构造函数调用失败。（译者注：这句话比较绕，实际上就是调用链。）所有这一切的原因就是对于`short`类型通用引用重载优先于`int`类型的重载。

使用通用引用的函数在C++中是最贪婪的函数。它们几乎可以精确匹配任何类型的实参（极少不适用的实参在[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)中介绍）。这也是把重载和通用引用组合在一块是糟糕主意的原因：通用引用的实现会匹配比开发者预期要多得多的实参类型。

一个更容易掉入这种陷阱的例子是写一个完美转发构造函数。简单对`logAndAdd`例子进行改造就可以说明这个问题。不用写接受`std::string`或者用索引查找`std::string`的自由函数，只是想一个构造函数有着相同操作的`Person`类：

```cpp
class Person {
public:
    template<typename T>
    explicit Person(T&& n)              //完美转发的构造函数，初始化数据成员
    : name(std::forward<T>(n)) {}

    explicit Person(int idx)            //int的构造函数
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

就像在`logAndAdd`的例子中，传递一个不是`int`的整型变量（比如`std::size_t`，`short`，`long`等）会调用通用引用的构造函数而不是`int`的构造函数，这会导致编译错误。这里这个问题甚至更糟糕，因为`Person`中存在的重载比肉眼看到的更多。在[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)中说明，在适当的条件下，C++会生成拷贝和移动构造函数，即使类包含了模板化的构造函数，模板函数能实例化产生与拷贝和移动构造函数一样的签名，也在合适的条件范围内。如果拷贝和移动构造被生成，`Person`类看起来就像这样：

```cpp
class Person {
public:
    template<typename T>            //完美转发的构造函数
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx);       //int的构造函数

    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
    …
};
```

只有你在花了很多时间在编译器领域时，下面的行为才变得直观（译者注：这里意思就是这种实现会导致不符合人类直觉的结果，下面就解释了这种现象的原因）：

```cpp
Person p("Nancy"); 
auto cloneOfP(p);                   //从p创建新Person；这通不过编译！
```

这里我们试图通过一个`Person`实例创建另一个`Person`，显然应该调用拷贝构造即可。（`p`是左值，我们可以把通过移动操作来完成“拷贝”的想法请出去了。）但是这份代码不是调用拷贝构造函数，而是调用完美转发构造函数。然后，完美转发的函数将尝试使用`Person`对象`p`初始化`Person`的`std::string`数据成员，编译器就会报错。

“为什么？”你可能会疑问，“为什么拷贝构造会被完美转发构造替代？我们显然想拷贝`Person`到另一个`Person`”。确实我们是这样想的，但是编译器严格遵循C++的规则，这里的相关规则就是控制对重载函数调用的解析规则。

编译器的理由如下：`cloneOfP`被non-`const`左值`p`初始化，这意味着模板化构造函数可被实例化为采用`Person`类型的non-`const`左值。实例化之后，`Person`类看起来是这样的：

```cpp
class Person {
public:
    explicit Person(Person& n)          //由完美转发模板初始化
    : name(std::forward<Person&>(n)) {}

    explicit Person(int idx);           //同之前一样

    Person(const Person& rhs);          //拷贝构造函数（编译器生成的）
    …
};
```

在这个语句中，

```cpp
auto cloneOfP(p);
```

其中`p`被传递给拷贝构造函数或者完美转发构造函数。调用拷贝构造函数要求在`p`前加上`const`的约束来满足函数形参的类型，而调用完美转发构造不需要加这些东西。从模板产生的重载函数是更好的匹配，所以编译器按照规则：调用最佳匹配的函数。“拷贝”non-`const`左值类型的`Person`交由完美转发构造函数处理，而不是拷贝构造函数。

如果我们将本例中的传递的对象改为`const`的，会得到完全不同的结果：

```cpp
const Person cp("Nancy");   //现在对象是const的
auto cloneOfP(cp);          //调用拷贝构造函数！
```

因为被拷贝的对象是`const`，是拷贝构造函数的精确匹配。虽然模板化的构造函数可以被实例化为有完全一样的函数签名，

```cpp
class Person {
public:
    explicit Person(const Person& n);   //从模板实例化而来
  
    Person(const Person& rhs);          //拷贝构造函数（编译器生成的）
    …
};
```

但是没啥影响，因为重载规则规定当模板实例化函数和非模板函数（或者称为“正常”函数）匹配优先级相当时，优先使用“正常”函数。拷贝构造函数（正常函数）因此胜过具有相同签名的模板实例化函数。

（如果你想知道为什么编译器在生成一个拷贝构造函数时还会模板实例化一个相同签名的函数，参考[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)。）

当继承纳入考虑范围时，完美转发的构造函数与编译器生成的拷贝、移动操作之间的交互会更加复杂。尤其是，派生类的拷贝和移动操作的传统实现会表现得非常奇怪。来看一下：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }

    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
};
```

如同注释表示的，派生类的拷贝和移动构造函数没有调用基类的拷贝和移动构造函数，而是调用了基类的完美转发构造函数！为了理解原因，要知道派生类将`SpecialPerson`类型的实参传递给其基类，然后通过模板实例化和重载解析规则作用于基类`Person`。最终，代码无法编译，因为`std::string`没有接受一个`SpecialPerson`的构造函数。

我希望到目前为止，已经说服了你，如果可能的话，避免对通用引用形参的函数进行重载。但是，如果在通用引用上重载是糟糕的主意，那么如果需要可转发大多数实参类型的函数，但是对于某些实参类型又要特殊处理应该怎么办？存在多种办法。实际上，下一个条款，[Item27](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item27.md)专门来讨论这个问题，敬请阅读。

**请记住：**

- 对通用引用形参的函数进行重载，通用引用函数的调用机会几乎总会比你期望的多得多。
- 完美转发构造函数是糟糕的实现，因为对于non-`const`左值，它们比拷贝构造函数而更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用。
