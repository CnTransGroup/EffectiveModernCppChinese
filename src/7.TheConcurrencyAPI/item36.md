## 条款三十六：如果有异步的必要请指定`std::launch::async`

**Item 36: Specify `std::launch::async` if asynchronicity is essential.**

当你调用`std::async`执行函数时（或者其他可调用对象），你通常希望异步执行函数。但是这并不一定是你要求`std::async`执行的操作。你事实上要求这个函数按照`std::async`启动策略来执行。有两种标准策略，每种都通过`std::launch`这个限域`enum`的一个枚举名表示（关于枚举的更多细节参见[Item10](../3.MovingToModernCpp/item10.md)）。假定一个函数`f`传给`std::async`来执行：

- **`std::launch::async`启动策略**意味着`f`必须异步执行，即在不同的线程。
- **`std::launch::deferred`启动策略**意味着`f`仅当在`std::async`返回的*future*上调用`get`或者`wait`时才执行。这表示`f`**推迟**到存在这样的调用时才执行（译者注：异步与并发是两个不同概念，这里侧重于惰性求值）。当`get`或`wait`被调用，`f`会同步执行，即调用方被阻塞，直到`f`运行结束。如果`get`和`wait`都没有被调用，`f`将不会被执行。（这是个简化说法。关键点不是要在其上调用`get`或`wait`的那个*future*，而是*future*引用的那个共享状态。（[Item38](../7.TheConcurrencyAPI/item38.md)讨论了*future*与共享状态的关系。）因为`std::future`支持移动，也可以用来构造`std::shared_future`，并且因为`std::shared_future`可以被拷贝，对共享状态——对`f`传到的那个`std::async`进行调用产生的——进行引用的*future*对象，有可能与`std::async`返回的那个*future*对象不同。这非常绕口，所以经常回避这个事实，简称为在`std::async`返回的*future*上调用`get`或`wait`。）

可能让人惊奇的是，`std::async`的默认启动策略——你不显式指定一个策略时它使用的那个——不是上面中任意一个。相反，是求或在一起的。下面的两种调用含义相同：

```cpp
auto fut1 = std::async(f);                      //使用默认启动策略运行f
auto fut2 = std::async(std::launch::async |     //使用async或者deferred运行f
                       std::launch::deferred,
                       f);
```

因此默认策略允许`f`异步或者同步执行。如同[Item35](../7.TheConcurrencyAPI/Item35.md)中指出，这种灵活性允许`std::async`和标准库的线程管理组件承担线程创建和销毁的责任，避免资源超额，以及平衡负载。这就是使用`std::async`并发编程如此方便的原因。

但是，使用默认启动策略的`std::async`也有一些有趣的影响。给定一个线程`t`执行此语句：

```cpp
auto fut = std::async(f);   //使用默认启动策略运行f
```

- **无法预测`f`是否会与`t`并发运行**，因为`f`可能被安排延迟运行。
- **无法预测`f`是否会在与某线程相异的另一线程上执行，这个某线程在`fut`上调用`get`或`wait`**。如果对`fut`调用函数的线程是`t`，含义就是无法预测`f`是否在异于`t`的另一线程上执行。
- **无法预测`f`是否执行**，因为不能确保在程序每条路径上，都会不会在`fut`上调用`get`或者`wait`。

默认启动策略的调度灵活性导致使用`thread_local`变量比较麻烦，因为这意味着如果`f`读写了**线程本地存储**（*thread-local storage*，TLS），不可能预测到哪个线程的变量被访问：

```cpp
auto fut = std::async(f);   //f的TLS可能是为单独的线程建的，
                            //也可能是为在fut上调用get或者wait的线程建的
```

这还会影响到基于`wait`的循环使用超时机制，因为在一个延时的任务（参见[Item35](../7.TheConcurrencyAPI/Item35.md)）上调用`wait_for`或者`wait_until`会产生`std::launch::deferred`值。意味着，以下循环看似应该最终会终止，但可能实际上永远运行：

```cpp
using namespace std::literals;      //为了使用C++14中的时间段后缀；参见条款34

void f()                            //f休眠1秒，然后返回
{
    std::this_thread::sleep_for(1s);
}

auto fut = std::async(f);           //异步运行f（理论上）

while (fut.wait_for(100ms) !=       //循环，直到f完成运行时停止...
       std::future_status::ready)   //但是有可能永远不会发生！
{
    …
}
```

如果`f`与调用`std::async`的线程并发运行（即，如果为`f`选择的启动策略是`std::launch::async`），这里没有问题（假定`f`最终会执行完毕），但是如果`f`是延迟执行，`fut.wait_for`将总是返回`std::future_status::deferred`。这永远不等于`std::future_status::ready`，循环会永远执行下去。

这种错误很容易在开发和单元测试中忽略，因为它可能在负载过高时才能显现出来。那些是使机器资源超额或者线程耗尽的条件，此时任务推迟执行才最有可能发生。毕竟，如果硬件没有资源耗尽，没有理由不安排任务并发执行。

修复也是很简单的：只需要检查与`std::async`对应的`future`是否被延迟执行即可，那样就会避免进入无限循环。不幸的是，没有直接的方法来查看`future`是否被延迟执行。相反，你必须调用一个超时函数——比如`wait_for`这种函数。在这个情况中，你不想等待任何事，只想查看返回值是否是`std::future_status::deferred`，所以无须怀疑，使用0调用`wait_for`：

```cpp
auto fut = std::async(f);               //同上

if (fut.wait_for(0s) ==                 //如果task是deferred（被延迟）状态
    std::future_status::deferred)
{
    …                                   //在fut上调用wait或get来异步调用f
} else {                                //task没有deferred（被延迟）
    while (fut.wait_for(100ms) !=       //不可能无限循环（假设f完成）
           std::future_status::ready) {
        …                               //task没deferred（被延迟），也没ready（已准备）
                                        //做并行工作直到已准备
    }
    …                                   //fut是ready（已准备）状态
}
```

这些各种考虑的结果就是，只要满足以下条件，`std::async`的默认启动策略就可以使用：

- 任务不需要和执行`get`或`wait`的线程并行执行。
- 读写哪个线程的`thread_local`变量没什么问题。
- 可以保证会在`std::async`返回的*future*上调用`get`或`wait`，或者该任务可能永远不会执行也可以接受。
- 使用`wait_for`或`wait_until`编码时考虑到了延迟状态。

如果上述条件任何一个都满足不了，你可能想要保证`std::async`会安排任务进行真正的异步执行。进行此操作的方法是调用时，将`std::launch::async`作为第一个实参传递：

```cpp
auto fut = std::async(std::launch::async, f);   //异步启动f的执行
```

事实上，对于一个类似`std::async`行为的函数，但是会自动使用`std::launch::async`作为启动策略的工具，拥有它会非常方便，而且编写起来很容易也使它看起来很棒。C++11版本如下：

```cpp
template<typename F, typename... Ts>
inline
std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params)          //返回异步调用f(params...)得来的future
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Ts>(params)...);
}
```

这个函数接受一个可调用对象`f`和0或多个形参`params`，然后完美转发（参见[Item25](../5.RRefMovSemPerfForw/item25.md)）给`std::async`，使用`std::launch::async`作为启动策略。就像`std::async`一样，返回`std::future`作为用`params`调用`f`得到的结果。确定结果的类型很容易，因为*type trait* `std::result_of`可以提供给你。（参见[Item9](../3.MovingToModernCpp/item9.md)关于*type trait*的详细表述。）

`reallyAsync`就像`std::async`一样使用：

```cpp
auto fut = reallyAsync(f); //异步运行f，如果std::async抛出异常它也会抛出
```

在C++14中，`reallyAsync`返回类型的推导能力可以简化函数的声明：

```cpp
template<typename F, typename... Ts>
inline
auto                                        // C++14
reallyAsync(F&& f, Ts&&... params)
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Ts>(params)...);
}
```

这个版本清楚表明，`reallyAsync`除了使用`std::launch::async`启动策略之外什么也没有做。

**请记住：**

- `std::async`的默认启动策略是异步和同步执行兼有的。
- 这个灵活性导致访问`thread_local`s的不确定性，隐含了任务可能不会被执行的意思，会影响调用基于超时的`wait`的程序逻辑。
- 如果异步执行任务非常关键，则指定`std::launch::async`。
