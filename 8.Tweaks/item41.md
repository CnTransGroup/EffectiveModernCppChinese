# 第8章 微调

**CHAPTER 8 Tweaks**

对于C++中的通用技术和特性，总是存在适用和不适用的场景。除了本章覆盖的两个例外，描述什么场景使用哪种通用技术通常来说很容易。这两个例外是传值（pass by value）和安置（emplacement）。决定何时使用这两种技术受到多种因素的影响，本书提供的最佳建议是在使用它们的同时仔细考虑清楚，尽管它们都是高效的现代C++编程的重要角色。接下来的条款提供了使用它们来编写软件是否合适的所需信息。

## 条款四十一：对于移动成本低且总是被拷贝的可拷贝形参，考虑按值传递

**Item 41: Consider pass by value for copyable parameters that are cheap to move and always copied**

有些函数的形参是可拷贝的。（在本条款中，“拷贝”一个形参通常意思是，将形参作为拷贝或移动操作的源对象。[简介](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/Introduction.md)中提到，C++没有术语来区分拷贝操作得到的副本和移动操作得到的副本。）比如说，`addName`成员函数可以拷贝自己的形参到一个私有容器。为了提高效率，应该拷贝左值，移动右值。

```cpp
class Widget {
public:
    void addName(const std::string& newName)    //接受左值；拷贝它
    { names.push_back(newName); }

    void addName(std::string&& newName)         //接受右值；移动它；std::move
    { names.push_back(std::move(newName)); }    //的使用见条款25
    …
private:
    std::vector<std::string> names;
};
```

这是可行的，但是需要编写两个函数来做本质相同的事。这有点让人难受：两个函数声明，两个函数实现，两个函数写说明，两个函数的维护。唉。

此外，目标代码中会有两个函数——你可能会担心程序的空间占用。这种情况下，两个函数都可能内联，可能会避免同时两个函数同时存在导致的代码膨胀问题，但是一旦没有被内联，目标代码就会出现两个函数。

另一种方法是使`addName`函数成为具有通用引用的函数模板（参考[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)）：

```cpp
class Widget {
public:
    template<typename T>                            //接受左值和右值；
    void addName(T&& newName) {                     //拷贝左值，移动右值；
        names.push_back(std::forward<T>(newName));  //std::forward的使用见条款25
    }
    …
};
```

这减少了源代码的维护工作，但是通用引用会导致其他复杂性。作为模板，`addName`的实现必须放置在头文件中。在编译器展开的时候，可能会产生多个函数，因为不止为左值和右值实例化，也可能为`std::string`和可转换为`std::string`的类型分别实例化为多个函数（参考[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)）。同时有些参数类型不能通过通用引用传递（参考[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)），而且如果传递了不合法的实参类型，编译器错误会令人生畏（参考[Item27](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item27.md)）。

是否存在一种编写`addName`的方法，可以左值拷贝，右值移动，只用处理一个函数（源代码和目标代码中），且避免使用通用引用？答案是是的。你要做的就是放弃你学习C++编程的第一条规则。这条规则是避免在传递用户定义的对象时使用传值方式。像是`addName`函数中的`newName`参数，按值传递可能是一种完全合理的策略。

在我们讨论为什么对于`addName`中的`newName`按值传递非常合理之前，让我们来考虑该会怎样实现：

```cpp
class Widget {
public:
    void addName(std::string newName) {         //接受左值或右值；移动它
        names.push_back(std::move(newName));
    }
    …
}
```

该代码唯一可能令人困惑的部分就是对形参`newName`使用`std::move`。`std::move`典型的应用场景是用在右值引用，但是在这里，我们了解到：（1）`newName`是，与调用者传进来的对象相完全独立的，一个对象，所以改变`newName`不会影响到调用者；（2）这就是`newName`的最后一次使用，所以移动它不会影响函数内此后的其他代码。

只有一个`addName`函数的事实，解释了在源代码和目标代码中，我们怎样避免了代码重复。我们没有使用通用引用，不会导致头文件膨胀、奇怪的失败情况、或者令人困惑的编译错误消息。但是这种设计的效率如何呢？我们在按值传递哎，会不会开销很大？

在C++98中，可以肯定确实开销会大。无论调用者传入什么，形参`newName`都是由拷贝构造出来。但是在C++11中，只有在左值实参情况下，`addName`被拷贝构造出来；对于右值，它会被移动构造。来看如下例子：

```cpp
Widget w;
…
std::string name("Bart");
w.addName(name);            //使用左值调用addName
…
w.addName(name + "Jenne");  //使用右值调用addName（见下）
```

第一处调用`addName`（当`name`被传递时），形参`newName`是使用左值被初始化。`newName`因此是被拷贝构造，就像在C++98中一样。第二处调用，`newName`使用`std::string`对象被初始化，这个`std::string`对象是调用`std::string`的`operator+`（即*append*操作）得到的结果。这个对象是一个右值，因此`newName`是被移动构造的。

就像我们想要的那样，左值拷贝，右值移动，优雅吧？

优雅，但是要牢记一些警示，回顾一下我们考虑过的三个版本的`addName`：

```cpp
class Widget {                                  //方法1：对左值和右值重载
public:
    void addName(const std::string& newName)
    { names.push_back(newName); } // rvalues
    void addName(std::string&& newName)
    { names.push_back(std::move(newName)); }
    …
private:
    std::vector<std::string> names;
};

class Widget {                                  //方法2：使用通用引用
public:
    template<typename T>
    void addName(T&& newName)
    { names.push_back(std::forward<T>(newName)); }
    …
};

class Widget {                                  //方法3：传值
public:
    void addName(std::string newName)
    { names.push_back(std::move(newName)); }
    …
};
```

我将前两个版本称为“按引用方法”，因为都是通过引用传递形参。

仍然考虑这两种调用方式：

```cpp
Widget w;
…
std::string name("Bart");
w.addName(name);                                //传左值
…
w.addName(name + "Jenne");                      //传右值
```

现在分别考虑三种实现中，给`Widget`添加一个名字的两种调用方式，拷贝和移动操作的开销。会忽略编译器对于移动和拷贝操作的优化，因为这些优化是与上下文和编译器有关的，实际上不会改变我们分析的本质。

- **重载**：无论传递左值还是传递右值，调用都会绑定到一个叫`newName`的引用上。从拷贝和移动操作方面看，这个过程零开销。左值重载中，`newName`拷贝到`Widget::names`中，右值重载中，移动进去。开销总结：左值一次拷贝，右值一次移动。
- **使用通用引用**：同重载一样，调用也绑定到`addName`这个引用上，没有开销。由于使用了`std::forward`，左值`std::string`实参会拷贝到`Widget::names`，右值`std::string`实参移动进去。对`std::string`实参来说，开销同重载方式一样：左值一次拷贝，右值一次移动。

  Item25 解释了如果调用者传递的实参不是`std::string`类型，将会转发到`std::string`的构造函数，几乎也没有`std::string`拷贝或者移动操作。因此通用引用的方式有同样效率，所以这不影响本次分析，简单假定调用者总是传入`std::string`类型实参即可。
- **按值传递**：无论传递左值还是右值，都必须构造`newName`形参。如果传递的是左值，需要拷贝的开销，如果传递的是右值，需要移动的开销。在函数的实现中，`newName`总是采用移动的方式到`Widget::names`。开销总结：左值参数，一次拷贝一次移动，右值参数两次移动。对比按引动传递的方法，对于左值或者右值，均多出一次移动操作。

再次回顾本Item的内容：

> 对于移动成本低且总是被拷贝的可拷贝形参，考虑按值传递


这样措辞是有原因的。实际上四个原因：

1. 应该仅**考虑**使用传值方式。确实，只需要编写一个函数。确实，只会在目标代码中生成一个函数。确实，避免了通用引用的种种问题。但是毕竟开销会比那些替代方式更高（译者注：指接受引用的两种实现方式），而且下面还会讨论，还会存在一些目前我们并未讨论到的开销。

2. 仅考虑对于**可拷贝形参**使用按值传递。不符合此条件的的形参必须有只可移动的类型（*move-only types*）（的数据成员），因为函数总是会做副本（译注：指的是传值时形参总是实参的一个副本），而如果它们不可拷贝，副本就必须通过移动构造函数创建。（这样的句子就说明有一个术语来区分拷贝操作制作的副本，和移动操作制作的副本，是非常好的。）回忆一下传值方案比“重载”方案的优势在于，仅有一个函数要写。但是对于只可移动类型，没必要为左值实参提供重载，因为拷贝左值需要拷贝构造函数，只可移动类型的拷贝构造函数是禁用的。那意味着只需要支持右值实参，“重载”方案只需要一个重载函数：接受右值引用的函数。

   考虑一个类：有个`std::unique_ptr<std::string>`的数据成员，对它有个赋值器（*setter*）。`std::unique_ptr`是只可移动的类型，所以赋值器的“重载”方式只有一个函数：

   ```cpp
   class Widget {
   public:
       …
       void setPtr(std::unique_ptr<std::string>&& ptr)
       { p = std::move(ptr); }
   
   private:
       std::unique_ptr<std::string> p;
   };
   ```

   调用者可能会这样写：

   ```cpp
   Widget w;
   …
   w.setPtr(std::make_unique<std::string>("Modern C++"));
   ```

   这样，从`std::make_unique`返回的右值`std::unique_ptr<std::string>`通过右值引用被传给`setPtr`，然后移动到数据成员`p`中。整体开销就是一次移动。

   如果`setPtr`使用传值方式接受形参：
   
   ```cpp
   class Widget {
   public:
       …
       void setPtr(std::unique_ptr<std::string> ptr)
       { p = std::move(ptr); }
       …
   };
   ```

   同样的调用就会先移动构造`ptr`形参，然后`ptr`再移动赋值到数据成员`p`，整体开销就是两次移动——是“重载”方法开销的两倍。
   
3. 按值传递应该仅考虑那些**移动开销小**的形参。当移动的开销较低，额外的一次移动才能被开发者接受，但是当移动的开销很大，执行不必要的移动就类似执行一个不必要的拷贝，而避免不必要的拷贝的重要性就是最开始C++98规则中避免传值的原因！

4. 你应该只对**总是被拷贝**的形参考虑按值传递。为了看清楚为什么这很重要，假定在拷贝形参到`names`容器前，`addName`需要检查新的名字的长度是否过长或者过短，如果是，就忽略增加名字的操作：

   ```cpp
   class Widget {
   public:
       void addName(std::string newName)
       {
           if ((newName.length() >= minLen) &&
               (newName.length() <= maxLen))
           {
               names.push_back(std::move(newName));
           }
       }
       …
   private:
    std::vector<std::string> names;
   };
   ```
   
   即使这个函数没有在`names`添加任何内容，也增加了构造和销毁`newName`的开销，而按引用传递会避免这笔开销。

即使你编写的函数对可拷贝类型执行无条件的复制，且这个类型移动开销小，有时也可能不适合按值传递。这是因为函数拷贝一个形参存在两种方式：一种是通过构造函数（拷贝构造或者移动构造），还有一种是赋值（拷贝赋值或者移动赋值）。`addName`使用构造函数，它的形参`newName`传递给`vector::push_back`，在这个函数内部，`newName`是通过拷贝构造在`std::vector`末尾创建一个新元素。对于使用构造函数拷贝形参的函数，之前的分析已经可以给出最终结论：按值传递对于左值和右值均增加了一次移动操作的开销。

当形参通过赋值操作进行拷贝时，分析起来更加复杂。比如，我们有一个表征密码的类，因为密码可能会被修改，我们提供了赋值器函数`changeTo`。用按值传递的策略，我们实现一个`Password`类如下：

```cpp
class Password {
public:
    explicit Password(std::string pwd)      //传值
    : text(std::move(pwd)) {}               //构造text
    void changeTo(std::string newPwd)       //传值
    { text = std::move(newPwd); }           //赋值text
    …
private:
    std::string text;                       //密码的text
};
```

将密码存储为纯文本格式恐怕将使你的软件安全团队抓狂，但是先忽略这点考虑这段代码：

```cpp
std::string initPwd("Supercalifragilisticexpialidocious");
Password p(initPwd);
```

毫无疑问：`p.text`被给定的密码构造，在构造函数用按值传递的方式增加了一次移动构造的开销，如果使用重载或者通用引用就会消除这个开销。一切都还好。

但是，该程序的用户可能对这个密码不太满意，因为“Supercalifragilisticexpialidocious”可以在许多字典中找到。他或者她因此采取等价于如下代码的一些动作：

```cpp
std::string newPassword = "Beware the Jabberwock";
p.changeTo(newPassword);
```

不用关心新密码是不是比旧密码更好，那是用户关心的问题。我们关心的是`changeTo`使用赋值来拷贝形参`newPwd`，可能导致函数的按值传递实现方案的开销大大增加。

传递给`changeTo`的实参是一个左值（`newPassword`），所以`newPwd`形参被构造时，`std::string`的拷贝构造函数会被调用，这个函数会分配新的存储空间给新密码。`newPwd`会移动赋值到`text`，这会导致`text`本来持有的内存被释放。所以`changeTo`存在两次动态内存管理的操作：一次是为新密码创建内存，一次是销毁旧密码的内存。

但是在这个例子中，旧密码比新密码长度更长，所以不需要分配新内存，销毁旧内存的操作。如果使用重载的方式，有可能两次动态内存管理操作得以避免：

```cpp
class Password {
public:
    …
    void changeTo(std::string& newPwd)      //对左值的重载
    {
        text = newPwd;                      //如果text.capacity() >= newPwd.size()，
                                            //可能重用text的内存
    }
    …
private:
    std::string text;                       //同上
};
```

这种情况下，按值传递的开销包括了内存分配和内存销毁——可能会比`std::string`的移动操作高出几个数量级。

有趣的是，如果旧密码短于新密码，在赋值过程中就不可避免要销毁、分配内存，这种情况，按值传递跟按引用传递的效率是一样的。因此，基于赋值的形参拷贝操作开销取决于具体的实参的值，这种分析适用于在动态分配内存中存值的形参类型。不是所有类型都满足，但是很多——包括`std::string`和`std::vector`——是这样。

这种潜在的开销增加仅在传递左值实参时才适用，因为执行内存分配和释放通常发生在真正的拷贝操作（即，不是移动）中。对右值实参，移动几乎就足够了。

结论是，使用通过赋值拷贝一个形参进行按值传递的函数的额外开销，取决于传递的类型，左值和右值的比例，这个类型是否需要动态分配内存，以及，如果需要分配内存的话，赋值操作符的具体实现，还有赋值目标占的内存至少要跟赋值源占的内存一样大。对于`std::string`来说，开销还取决于实现是否使用了小字符串优化（SSO——参考[Item29](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item29.md)），如果是，那么要赋值的值是否匹配SSO缓冲区。

所以，正如我所说，当形参通过赋值进行拷贝时，分析按值传递的开销是复杂的。通常，最有效的经验就是“在证明没问题之前假设有问题”，就是除非已证明按值传递会为你需要的形参类型产生可接受的执行效率，否则使用重载或者通用引用的实现方式。

到此为止，对于需要运行尽可能快的软件来说，按值传递可能不是一个好策略，因为避免即使开销很小的移动操作也非常重要。此外，有时并不能清楚知道会发生多少次移动操作。在`Widget::addName`例子中，按值传递仅多了一次移动操作，但是如果`Widget::addName`调用了`Widget::validateName`，这个函数也是按值传递。（假定它有理由总是拷贝它的形参，比如把验证过的所有值存在一个数据结构中。）并假设`validateName`调用了第三个函数，也是按值传递……

可以看到这将会通向何方。在调用链中，每个函数都使用传值，因为“只多了一次移动的开销”，但是整个调用链总体就会产生无法忍受的开销，通过引用传递，调用链不会增加这种开销。

跟性能无关，但总是需要考虑的是，按值传递不像按引用传递那样会收到切片问题的影响。这是C++98的事，在此不在详述，但是如果要设计一个函数，来处理这样的形参：基类**或者任何其派生类**，你肯定不想声明一个那个类型的传值形参，因为你会“切掉”传入的任意派生类对象的派生类特征：

```cpp
class Widget { … };                         //基类
class SpecialWidget: public Widget { … };   //派生类
void processWidget(Widget w);               //对任意类型的Widget的函数，包括派生类型
…                                           //遇到对象切片问题
SpecialWidget sw;
…
processWidget(sw);                          //processWidget看到的是Widget，
                                            //不是SpecialWidget！
```

如果不熟悉对象切片问题，可以先通过搜索引擎了解一下。这样你就知道切片问题是C++98中默认按值传递名声不好的另一个原因（要在效率问题的原因之上）。有充分的理由来说明为什么你学习C++编程的第一件事就是避免用户自定义类型进行按值传递。

C++11没有从根本上改变C++98对于按值传递的智慧。通常，按值传递仍然会带来你希望避免的性能下降，而且按值传递会导致切片问题。C++11中新的功能是区分了左值和右值实参。利用对于可拷贝类型的右值的移动语义，需要重载或者通用引用，尽管两者都有其缺陷。对于特殊的场景，可拷贝且移动开销小的类型，传递给总是会拷贝他们的一个函数，并且切片也不需要考虑，这时，按值传递就提供了一种简单的实现方式，效率接近传递引用的函数，但是避免了传引用方案的缺点。

**请记住：**

- 对于可拷贝，移动开销低，而且无条件被拷贝的形参，按值传递效率基本与按引用传递效率一致，而且易于实现，还生成更少的目标代码。
- 通过构造拷贝形参可能比通过赋值拷贝形参开销大的多。
- 按值传递会引起切片问题，所说不适合基类形参类型。
