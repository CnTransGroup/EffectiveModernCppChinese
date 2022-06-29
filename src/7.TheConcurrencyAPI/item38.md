## 条款三十八：关注不同线程句柄的析构行为

**Item 38：Be aware of varying thread handle destructor behavior**

[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)中说明了可结合的`std::thread`对应于执行的系统线程。未延迟（non-deferred）任务的*future*（参见[Item36](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item36.md)）与系统线程有相似的关系。因此，可以将`std::thread`对象和*future*对象都视作系统线程的**句柄**（*handles*）。

从这个角度来说，有趣的是`std::thread`和*future*在析构时有相当不同的行为。在[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)中说明，可结合的`std::thread`析构会终止你的程序，因为两个其他的替代选择——隐式`join`或者隐式`detach`都是更加糟糕的。但是，*future*的析构表现有时就像执行了隐式`join`，有时又像是隐式执行了`detach`，有时又没有执行这两个选择。它永远不会造成程序终止。这个线程句柄多种表现值得研究一下。

我们可以观察到实际上*future*是通信信道的一端，被调用者通过该信道将结果发送给调用者。（[Item39](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item39.md)说，与*future*有关的这种通信信道也可以被用于其他目的。但是对于本条款，我们只考虑它们作为这样一个机制的用法，即被调用者传送结果给调用者。）被调用者（通常是异步执行）将计算结果写入通信信道中（通常通过`std::promise`对象），调用者使用*future*读取结果。你可以想象成下面的图示，虚线表示信息的流动方向：

![item38_fig1](media/item38_fig1.png)

但是被调用者的结果存储在哪里？被调用者会在调用者`get`相关的*future*之前执行完成，所以结果不能存储在被调用者的`std::promise`。这个对象是局部的，当被调用者执行结束后，会被销毁。

结果同样不能存储在调用者的*future*，因为（当然还有其他原因）`std::future`可能会被用来创建`std::shared_future`（这会将被调用者的结果所有权从`std::future`转移给`std::shared_future`），而`std::shared_future`在`std::future`被销毁之后可能被复制很多次。鉴于不是所有的结果都可以被拷贝（即只可移动类型），并且结果的生命周期至少与最后一个引用它的*future*一样长，这些潜在的*future*中哪个才是被调用者用来存储结果的？

因为与被调用者关联的对象和与调用者关联的对象都不适合存储这个结果，所以必须存储在两者之外的位置。此位置称为**共享状态**（*shared state*）。共享状态通常是基于堆的对象，但是标准并未指定其类型、接口和实现。标准库的作者可以通过任何他们喜欢的方式来实现共享状态。

我们可以想象调用者，被调用者，共享状态之间关系如下图，虚线还是表示信息流方向：

![item38_fig2](media/item38_fig2.png)

共享状态的存在非常重要，因为*future*的析构函数——这个条款的话题——取决于与*future*关联的共享状态。特别地，

- **引用了共享状态——使用`std::async`启动的未延迟任务建立的那个——的最后一个*future*的析构函数会阻塞住**，直到任务完成。本质上，这种*future*的析构函数对执行异步任务的线程执行了隐式的`join`。
- **其他所有*future*的析构函数简单地销毁*future*对象**。对于异步执行的任务，就像对底层的线程执行`detach`。对于延迟任务来说如果这是最后一个*future*，意味着这个延迟任务永远不会执行了。

这些规则听起来好复杂。我们真正要处理的是一个简单的“正常”行为以及一个单独的例外。正常行为是*future*析构函数销毁*future*。就是这样。那意味着不`join`也不`detach`，也不运行什么，只销毁*future*的数据成员（当然，还做了另一件事，就是递减了共享状态中的引用计数，这个共享状态是由引用它的*future*和被调用者的`std::promise`共同控制的。这个引用计数让库知道共享状态什么时候可以被销毁。对于引用计数的一般信息参见[Item19](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item19.md)。）

正常行为的例外情况仅在某个`future`同时满足下列所有情况下才会出现：

- **它关联到由于调用`std::async`而创建出的共享状态**。
- **任务的启动策略是`std::launch::async`**（参见[Item36](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item36.md)），原因是运行时系统选择了该策略，或者在对`std::async`的调用中指定了该策略。
- **这个*future*是关联共享状态的最后一个*future***。对于`std::future`，情况总是如此，对于`std::shared_future`，如果还有其他的`std::shared_future`，与要被销毁的*future*引用相同的共享状态，则要被销毁的*future*遵循正常行为（即简单地销毁它的数据成员）。

只有当上面的三个条件都满足时，*future*的析构函数才会表现“异常”行为，就是在异步任务执行完之前阻塞住。实际上，这相当于对由于运行`std::async`创建出任务的线程隐式`join`。

通常会听到将这种异常的析构函数行为称为“`std::async`来的*futures*阻塞了它们的析构函数”。作为近似描述没有问题，但是有时你不只需要一个近似描述。现在你已经知道了其中真相。

你可能想要了解更加深入。比如“为什么由`std::async`启动的未延迟任务的共享状态，会有这么个特殊规则”，这很合理。据我所知，标准委员会希望避免隐式`detach`（参见[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)）的有关问题，但是不想采取强制程序终止这种激进的方案（就像对可结合的`sth::thread`做的那样（译者注：指析构时`std::thread`若可结合则调用`std::terminal`终止程序），同样参见[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)），所以妥协使用隐式`join`。这个决定并非没有争议，并且认真讨论过在C++14中放弃这种行为。最后，决定先不改变，所以C++11和C++14中这里的行为是一致的。

*future*的API没办法确定是否*future*引用了一个`std::async`调用产生的共享状态，因此给定一个任意的*future*对象，无法判断会不会阻塞析构函数从而等待异步任务的完成。这就产生了有意思的事情：

```cpp
//这个容器可能在析构函数处阻塞，因为其中至少一个future可能引用由std::async启动的
//未延迟任务创建出来的共享状态
std::vector<std::future<void>> futs;    //std::future<void>相关信息见条款39

class Widget {                          //Widget对象可能在析构函数处阻塞
public:
    …
private:
    std::shared_future<double> fut;
};
```

当然，如果你有办法知道给定的*future***不**满足上面条件的任意一条（比如由于程序逻辑造成的不满足），你就可以确定析构函数不会执行“异常”行为。比如，只有通过`std::async`创建的共享状态才有资格执行“异常”行为，但是有其他创建共享状态的方式。一种是使用`std::packaged_task`，一个`std::packaged_task`对象通过包覆（wrapping）方式准备一个函数（或者其他可调用对象）来异步执行，然后将其结果放入共享状态中。然后通过`std::packaged_task`的`get_future`函数可以获取有关该共享状态的*future*：

```cpp
int calcValue();                //要运行的函数
std::packaged_task<int()>       //包覆calcValue以异步运行
    pt(calcValue);
auto fut = pt.get_future();     //从pt获取future
```

此时，我们知道*future*没有关联`std::async`创建的共享状态，所以析构函数肯定正常方式执行。

一旦被创建，`std::packaged_task`类型的`pt`就可以在一个线程上执行。（也可以通过调用`std::async`运行，但是如果你想使用`std::async`运行任务，没有理由使用`std::packaged_task`，因为在`std::packaged_task`安排任务并执行之前，`std::async`会做`std::packaged_task`做的所有事。）

`std::packaged_task`不可拷贝，所以当`pt`被传递给`std::thread`构造函数时，必须先转为右值（通过`std::move`，参见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)）：

```cpp
std::thread t(std::move(pt));   //在t上运行pt
```

这个例子是你对于*future*的析构函数的正常行为有一些了解，但是将这些语句放在一个作用域的语句块里更容易看：

```cpp
{                                   //开始代码块
    std::packaged_task<int()>
        pt(calcValue); 
    
    auto fut = pt.get_future(); 
    
    std::thread t(std::move(pt));   //见下
    …
}                                   //结束代码块
```

此处最有趣的代码是在创建`std::thread`对象`t`之后，代码块结束前的“`…`”。使代码有趣的事是，在“`…`”中`t`上会发生什么。有三种可能性：

- **对`t`什么也不做**。这种情况，`t`会在语句块结束时是可结合的，这会使得程序终止（参见[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)）。
- **对`t`调用`join`**。这种情况，不需要`fut`在它的析构函数处阻塞，因为`join`被显式调用了。
- **对`t`调用`detach`**。这种情况，不需要在`fut`的析构函数执行`detach`，因为显式调用了。

换句话说，当你有一个关联了`std::packaged_task`创建的共享状态的*future*时，不需要采取特殊的销毁策略，因为通常你会代码中做终止、结合或分离这些决定之一，来操作`std::packaged_task`的运行所在的那个`std::thread`。

**请记住：**

- *future*的正常析构行为就是销毁*future*本身的数据成员。
- 引用了共享状态——使用`std::async`启动的未延迟任务建立的那个——的最后一个*future*的析构函数会阻塞住，直到任务完成。
