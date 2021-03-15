## 条款三十七：使`std::thread`在所有路径最后都不可结合

**Item 37: Make `std::thread`s unjoinable on all paths**

每个`std::thread`对象处于两个状态之一：**可结合的**（*joinable*）或者**不可结合的**（*unjoinable*）。可结合状态的`std::thread`对应于正在运行或者可能要运行的异步执行线程。比如，对应于一个阻塞的（*blocked*）或者等待调度的线程的`std::thread`是可结合的，对应于运行结束的线程的`std::thread`也可以认为是可结合的。

不可结合的`std::thread`正如所期待：一个不是可结合状态的`std::thread`。不可结合的`std::thread`对象包括：

- **默认构造的`std::thread`s**。这种`std::thread`没有函数执行，因此没有对应到底层执行线程上。
- **已经被移动走的`std::thread`对象**。移动的结果就是一个`std::thread`原来对应的执行线程现在对应于另一个`std::thread`。
- **已经被`join`的`std::thread`** 。在`join`之后，`std::thread`不再对应于已经运行完了的执行线程。
- **已经被`detach`的`std::thread`** 。`detach`断开了`std::thread`对象与执行线程之间的连接。

（译者注：`std::thread`可以视作状态保存的对象，保存的状态可能也包括可调用对象，有没有具体的线程承载就是有没有连接）

`std::thread`的可结合性如此重要的原因之一就是当可结合的线程的析构函数被调用，程序执行会终止。比如，假定有一个函数`doWork`，使用一个过滤函数`filter`，一个最大值`maxVal`作为形参。`doWork`检查是否满足计算所需的条件，然后使用在0到`maxVal`之间的通过过滤器的所有值进行计算。如果进行过滤非常耗时，并且确定`doWork`条件是否满足也很耗时，则将两件事并发计算是很合理的。

我们希望为此采用基于任务的设计（参见[Item35](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/Item35.md)），但是假设我们希望设置做过滤的线程的优先级。[Item35](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/Item35.md)阐释了那需要线程的原生句柄，只能通过`std::thread`的API来完成；基于任务的API（比如*future*）做不到。所以最终采用基于线程而不是基于任务。

我们可能写出以下代码：

代码如下：

```cpp
constexpr auto tenMillion = 10000000;           //constexpr见条款15

bool doWork(std::function<bool(int)> filter,    //返回计算是否执行；
            int maxVal = tenMillion)            //std::function见条款2
{
    std::vector<int> goodVals;                  //满足filter的值

    std::thread t([&filter, maxVal, &goodVals]  //填充goodVals
                  {
                      for (auto i = 0; i <= maxVal; ++i)
                          { if (filter(i)) goodVals.push_back(i); }
                  });

    auto nh = t.native_handle();                //使用t的原生句柄
    …                                           //来设置t的优先级

    if (conditionsAreSatisfied()) {
        t.join();                               //等t完成
        performComputation(goodVals);
        return true;                            //执行了计算
    }
    return false;                               //未执行计算
}
```

在解释这份代码为什么有问题之前，我先把`tenMillion`的初始化值弄得更可读一些，这利用了C++14的能力，使用单引号作为数字分隔符：

```cpp
constexpr auto tenMillion = 10'000'000;         //C++14
```

还要指出，在开始运行之后设置`t`的优先级就像把马放出去之后再关上马厩门一样（译者注：太晚了）。更好的设计是在挂起状态时开始`t`（这样可以在执行任何计算前调整优先级），但是我不想你为考虑那些代码而分心。如果你对代码中忽略的部分感兴趣，可以转到[Item39](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item39.md)，那个Item告诉你如何以开始那些挂起状态的线程。

返回`doWork`。如果`conditionsAreSatisfied()`返回`true`，没什么问题，但是如果返回`false`或者抛出异常，在`doWork`结束调用`t`的析构函数时，`std::thread`对象`t`会是可结合的。这造成程序执行中止。

你可能会想，为什么`std::thread`析构的行为是这样的，那是因为另外两种显而易见的方式更糟：

- **隐式`join`** 。这种情况下，`std::thread`的析构函数将等待其底层的异步执行线程完成。这听起来是合理的，但是可能会导致难以追踪的异常表现。比如，如果`conditonAreStatisfied()`已经返回了`false`，`doWork`继续等待过滤器应用于所有值就很违反直觉。

- **隐式`detach`** 。这种情况下，`std::thread`析构函数会分离`std::thread`与其底层的线程。底层线程继续运行。听起来比`join`的方式好，但是可能导致更严重的调试问题。比如，在`doWork`中，`goodVals`是通过引用捕获的局部变量。它也被*lambda*修改（通过调用`push_back`）。假定，*lambda*异步执行时，`conditionsAreSatisfied()`返回`false`。这时，`doWork`返回，同时局部变量（包括`goodVals`）被销毁。栈被弹出，并在`doWork`的调用点继续执行线程。

  调用点之后的语句有时会进行其他函数调用，并且至少一个这样的调用可能会占用曾经被`doWork`使用的栈位置。我们调用那么一个函数`f`。当`f`运行时，`doWork`启动的*lambda*仍在继续异步运行。该*lambda*可能在栈内存上调用`push_back`，该内存曾属于`goodVals`，但是现在是`f`的栈内存的某个位置。这意味着对`f`来说，内存被自动修改了！想象一下调试的时候“乐趣”吧。

标准委员会认为，销毁可结合的线程如此可怕以至于实际上禁止了它（规定销毁可结合的线程导致程序终止）。

这使你有责任确保使用`std::thread`对象时，在所有的路径上超出定义所在的作用域时都是不可结合的。但是覆盖每条路径可能很复杂，可能包括自然执行通过作用域，或者通过`return`，`continue`，`break`，`goto`或异常跳出作用域，有太多可能的路径。

每当你想在执行跳至块之外的每条路径执行某种操作，最通用的方式就是将该操作放入局部对象的析构函数中。这些对象称为**RAII对象**（*RAII objects*），从**RAII类**中实例化。（RAII全称为 “Resource Acquisition Is Initialization”（资源获得即初始化），尽管技术关键点在析构上而不是实例化上）。RAII类在标准库中很常见。比如STL容器（每个容器析构函数都销毁容器中的内容物并释放内存），标准智能指针（[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)-[20](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item20.md)解释了，`std::uniqu_ptr`的析构函数调用他指向的对象的删除器，`std::shared_ptr`和`std::weak_ptr`的析构函数递减引用计数），`std::fstream`对象（它们的析构函数关闭对应的文件）等。但是标准库没有`std::thread`的RAII类，可能是因为标准委员会拒绝将`join`和`detach`作为默认选项，不知道应该怎么样完成RAII。

幸运的是，完成自行实现的类并不难。比如，下面的类实现允许调用者指定`ThreadRAII`对象（一个`std::thread`的RAII对象）析构时，调用`join`或者`detach`：

```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };     //enum class的信息见条款10
    
    ThreadRAII(std::thread&& t, DtorAction a)   //析构函数中对t实行a动作
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {                                           //可结合性测试见下
        if (t.joinable()) {
            if (action == DtorAction::join) {
                t.join();
            } else {
                t.detach();
            }
        }
    }

    std::thread& get() { return t; }            //见下

private:
    DtorAction action;
    std::thread t;
};
```

我希望这段代码是不言自明的，但是下面几点说明可能会有所帮助：

- 构造器只接受`std::thread`右值，因为我们想要把传来的`std::thread`对象移动进`ThreadRAII`。（`std::thread`不可以复制。）

- 构造器的形参顺序设计的符合调用者直觉（首先传递`std::thread`，然后选择析构执行的动作，这比反过来更合理），但是成员初始化列表设计的匹配成员声明的顺序。将`std::thread`对象放在声明最后。在这个类中，这个顺序没什么特别之处，但是通常，可能一个数据成员的初始化依赖于另一个，因为`std::thread`对象可能会在初始化结束后就立即执行函数了，所以在最后声明是一个好习惯。这样就能保证一旦构造结束，在前面的所有数据成员都初始化完毕，可以供`std::thread`数据成员绑定的异步运行的线程安全使用。

- `ThreadRAII`提供了`get`函数访问内部的`std::thread`对象。这类似于标准智能指针提供的`get`函数，可以提供访问原始指针的入口。提供`get`函数避免了`ThreadRAII`复制完整`std::thread`接口的需要，也意味着`ThreadRAII`可以在需要`std::thread`对象的上下文环境中使用。

- 在`ThreadRAII`析构函数调用`std::thread`对象`t`的成员函数之前，检查`t`是否可结合。这是必须的，因为在不可结合的`std::thread`上调用`join`或`detach`会导致未定义行为。客户端可能会构造一个`std::thread`，然后用它构造一个`ThreadRAII`，使用`get`获取`t`，然后移动`t`，或者调用`join`或`detach`，每一个操作都使得`t`变为不可结合的。

  如果你担心下面这段代码

  ```cpp
  if (t.joinable()) {
      if (action == DtorAction::join) {
          t.join();
      } else {
          t.detach();
      }
  }
  ```

  存在竞争，因为在`t.joinable()`的执行和调用`join`或`detach`的中间，可能有其他线程改变了`t`为不可结合，你的直觉值得表扬，但是这个担心不必要。只有调用成员函数才能使`std::thread`对象从可结合变为不可结合状态，比如`join`，`detach`或者移动操作。在`ThreadRAII`对象析构函数调用时，应当没有其他线程在那个对象上调用成员函数。如果同时进行调用，那肯定是有竞争的，但是不在析构函数中，是在客户端代码中试图同时在一个对象上调用两个成员函数（析构函数和其他函数）。通常，仅当所有都为`const`成员函数时，在一个对象同时调用多个成员函数才是安全的。

在`doWork`的例子上使用`ThreadRAII`的代码如下：

```cpp
bool doWork(std::function<bool(int)> filter,        //同之前一样
            int maxVal = tenMillion)
{
    std::vector<int> goodVals;                      //同之前一样

    ThreadRAII t(                                   //使用RAII对象
        std::thread([&filter, maxVal, &goodVals]
                    {
                        for (auto i = 0; i <= maxVal; ++i)
                            { if (filter(i)) goodVals.push_back(i); }
                    }),
                    ThreadRAII::DtorAction::join    //RAII动作
    );

    auto nh = t.get().native_handle();
    …

    if (conditionsAreSatisfied()) {
        t.get().join();
        performComputation(goodVals);
        return true;
    }

    return false;
}
```

这种情况下，我们选择在`ThreadRAII`的析构函数对异步执行的线程进行`join`，因为在先前分析中，`detach`可能导致噩梦般的调试过程。我们之前也分析了`join`可能会导致表现异常（坦率说，也可能调试困难），但是在未定义行为（`detach`导致），程序终止（使用原生`std::thread`导致），或者表现异常之间选择一个后果，可能表现异常是最好的那个。

哎，[Item39](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item39.md)表明了使用`ThreadRAII`来保证在`std::thread`的析构时执行`join`有时不仅可能导致程序表现异常，还可能导致程序挂起。“适当”的解决方案是此类程序应该和异步执行的*lambda*通信，告诉它不需要执行了，可以直接返回，但是C++11中不支持**可中断线程**（*interruptible threads*）。可以自行实现，但是这不是本书讨论的主题。（关于这一点，Anthony Williams的《C++ Concurrency in Action》（Manning Publications，2012）的section 9.2中有详细讨论。）（译者注：此书中文版已出版，名为《C++并发编程实战》，且本文翻译时（2020）已有第二版出版。）

[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)说明因为`ThreadRAII`声明了一个析构函数，因此不会有编译器生成移动操作，但是没有理由`ThreadRAII`对象不能移动。如果要求编译器生成这些函数，函数的功能也正确，所以显式声明来告诉编译器自动生成也是合适的：

```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };         //跟之前一样

    ThreadRAII(std::thread&& t, DtorAction a)       //跟之前一样
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {
        …                                           //跟之前一样
    }

    ThreadRAII(ThreadRAII&&) = default;             //支持移动
    ThreadRAII& operator=(ThreadRAII&&) = default;

    std::thread& get() { return t; }                //跟之前一样

private: // as before
    DtorAction action;
    std::thread t;
};
```

**请记住：**

- 在所有路径上保证`thread`最终是不可结合的。
- 析构时`join`会导致难以调试的表现异常问题。
- 析构时`detach`会导致难以调试的未定义行为。
- 声明类数据成员时，最后声明`std::thread`对象。
