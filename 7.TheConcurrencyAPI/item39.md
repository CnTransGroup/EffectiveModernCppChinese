## 条款三十九：对于一次性事件通信考虑使用`void`的*futures*

**Item 39: Consider `void` futures for one-shot event communication**

有时，一个任务通知另一个异步执行的任务发生了特定的事件很有用，因为第二个任务要等到这个事件发生之后才能继续执行。事件也许是一个数据结构已经初始化，也许是计算阶段已经完成，或者检测到重要的传感器值。这种情况下，线程间通信的最佳方案是什么？

一个明显的方案就是使用条件变量（*condition variable*，简称*condvar*）。如果我们将检测条件的任务称为**检测任务**（*detecting task*），对条件作出反应的任务称为**反应任务**（*reacting task*），策略很简单：反应任务等待一个条件变量，检测任务在事件发生时改变条件变量。代码如下：

```cpp
std::condition_variable cv;         //事件的条件变量
std::mutex m;                       //配合cv使用的mutex
```

检测任务中的代码不能再简单了：

```cpp
…                                   //检测事件
cv.notify_one();                    //通知反应任务
```

如果有多个反应任务需要被通知，使用`notify_all`代替`notify_one`，但是这里，我们假定只有一个反应任务需要通知。

反应任务的代码稍微复杂一点，因为在对条件变量调用`wait`之前，必须通过`std::unique_lock`对象锁住互斥锁。（在等待条件变量前锁住互斥锁对线程库来说是经典操作。通过`std::unique_lock`锁住互斥锁的需要仅仅是C++11 API要求的一部分。）概念上的代码如下：

```cpp
…                                       //反应的准备工作
{                                       //开启关键部分

    std::unique_lock<std::mutex> lk(m); //锁住互斥锁

    cv.wait(lk);                        //等待通知，但是这是错的！
    
    …                                   //对事件进行反应（m已经上锁）
}                                       //关闭关键部分；通过lk的析构函数解锁m
…                                       //继续反应动作（m现在未上锁）
```

这份代码的第一个问题就是有时被称为**代码异味**（*code smell*）的一种情况：即使代码正常工作，但是有些事情也不是很正确。在这个情况中，这种问题源自于使用互斥锁。互斥锁被用于保护共享数据的访问，但是可能检测任务和反应任务可能不会同时访问共享数据，比如说，检测任务会初始化一个全局数据结构，然后给反应任务用，如果检测任务在初始化之后不会再访问这个数据结构，而在检测任务表明数据结构准备完了之前反应任务不会访问这个数据结构，这两个任务在程序逻辑下互不干扰，也就没有必要使用互斥锁。但是条件变量的方法必须使用互斥锁，这就留下了令人不适的设计。

即使你忽略掉这个问题，还有两个问题需要注意：

- **如果在反应任务`wait`之前检测任务通知了条件变量，反应任务会挂起**。为了能使条件变量唤醒另一个任务，任务必须等待在条件变量上。如果在反应任务`wait`之前检测任务就通知了条件变量，反应任务就会丢失这次通知，永远不被唤醒。

- **`wait`语句虚假唤醒**。线程API的存在一个事实（在许多语言中——不只是C++），等待一个条件变量的代码即使在条件变量没有被通知时，也可能被唤醒，这种唤醒被称为**虚假唤醒**（*spurious wakeups*）。正确的代码通过确认要等待的条件确实已经发生来处理这种情况，并将这个操作作为唤醒后的第一个操作。C++条件变量的API使得这种问题很容易解决，因为允许把一个测试要等待的条件的*lambda*（或者其他函数对象）传给`wait`。因此，可以将反应任务`wait`调用这样写：

  ```cpp
  cv.wait(lk, 
          []{ return whether the evet has occurred; });
  ```

  要利用这个能力需要反应任务能够确定其等待的条件是否为真。但是我们考虑的场景中，它正在等待的条件是检测线程负责识别的事件的发生情况。反应线程可能无法确定等待的事件是否已发生。这就是为什么它在等待一个条件变量！

在很多情况下，使用条件变量进行任务通信非常合适，但是也有不那么合适的情况。

对于很多开发者来说，他们的下一个诀窍是共享的布尔型flag。flag被初始化为`false`。当检测线程识别到发生的事件，将flag置位：

```cpp
std::atomic<bool> flag(false);          //共享的flag；std::atomic见条款40
…                                       //检测某个事件
flag = true;                            //告诉反应线程
```

就其本身而言，反应线程轮询该flag。当发现flag被置位，它就知道等待的事件已经发生了：

```cpp
…                                       //准备作出反应
while (!flag);                          //等待事件
…                                       //对事件作出反应
```

这种方法不存在基于条件变量的设计的缺点。不需要互斥锁，在反应任务开始轮询之前检测任务就对flag置位也不会出现问题，并且不会出现虚假唤醒。好，好，好。

不好的一点是反应任务中轮询的开销。在任务等待flag被置位的时间里，任务基本被阻塞了，但是一直在运行。这样，反应线程占用了可能能给另一个任务使用的硬件线程，每次启动或者完成它的时间片都增加了上下文切换的开销，并且保持核心一直在运行状态，否则的话本来可以停下来省电。一个真正阻塞的任务不会发生上面的任何情况。这也是基于条件变量的优点，因为`wait`调用中的任务真的阻塞住了。

将条件变量和flag的设计组合起来很常用。一个flag表示是否发生了感兴趣的事件，但是通过互斥锁同步了对该flag的访问。因为互斥锁阻止并发访问该flag，所以如[Item40](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item40.md)所述，不需要将flag设置为`std::atomic`。一个简单的`bool`类型就可以，检测任务代码如下：

```cpp
std::condition_variable cv;             //跟之前一样
std::mutex m;
bool flag(false);                       //不是std::atomic
…                                       //检测某个事件
{
    std::lock_guard<std::mutex> g(m);   //通过g的构造函数锁住m
    flag = true;                        //通知反应任务（第1部分）
}                                       //通过g的析构函数解锁m
cv.notify_one();                        //通知反应任务（第2部分）
```

反应任务代码如下:

```cpp
…                                       //准备作出反应
{                                       //跟之前一样
    std::unique_lock<std::mutex> lk(m); //跟之前一样
    cv.wait(lk, [] { return flag; });   //使用lambda来避免虚假唤醒
    …                                   //对事件作出反应（m被锁定）
}
…                                       //继续反应动作（m现在解锁）
```

这份代码解决了我们一直讨论的问题。无论在检测线程对条件变量发出通知之前反应线程是否调用了`wait`都可以工作，即使出现了虚假唤醒也可以工作，而且不需要轮询。但是仍然有些古怪，因为检测任务通过奇怪的方式与反应线程通信。（译者注：下面的话挺绕的，可以参考原文）检测任务通过通知条件变量告诉反应线程，等待的事件可能发生了，但是反应线程必须通过检查flag来确保事件发生了。检测线程置位flag来告诉反应线程事件确实发生了，但是检测线程仍然还要先需要通知条件变量，以唤醒反应线程来检查flag。这种方案是可以工作的，但是不太优雅。

一个替代方案是让反应任务通过在检测任务设置的*future*上`wait`来避免使用条件变量，互斥锁和flag。这可能听起来也是个古怪的方案。毕竟，[Item38](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item38.md)中说明了*future*代表了从被调用方到（通常是异步的）调用方的通信信道的接收端，这里的检测任务和反应任务没有调用-被调用的关系。然而，[Item38](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item38.md)中也说说明了发送端是个`std::promise`，接收端是个*future*的通信信道不是只能用在调用-被调用场景。这样的通信信道可以用在任何你需要从程序一个地方传递信息到另一个地方的场景。这里，我们用来在检测任务和反应任务之间传递信息，传递的信息就是感兴趣的事件已经发生。

方案很简单。检测任务有一个`std::promise`对象（即通信信道的写入端），反应任务有对应的*future*。当检测任务看到事件已经发生，设置`std::promise`对象（即写入到通信信道）。同时，反应任务。`wait`会锁住反应任务直到`std::promise`被设置。

现在，`std::promise`和*futures*（即`std::future`和`std::shared_future`）都是需要类型形参的模板。形参表明通过通信信道被传递的信息的类型。在这里，没有数据被传递，只需要让反应任务知道它的*future*已经被设置了。我们在`std::promise`和*future*模板中需要的东西是表明通信信道中没有数据被传递的一个类型。这个类型就是`void`。检测任务使用`std::promise<void>`，反应任务使用`std::future<void>`或者`std::shared_future<void>`。当感兴趣的事件发生时，检测任务设置`std::promise<void>`，反应任务在*future*上`wait`。尽管反应任务不从检测任务那里接收任何数据，通信信道也可以让反应任务知道，检测任务什么时候已经通过对`std::promise<void>`调用`set_value`“写入”了`void`数据。

所以，有

```cpp
std::promise<void> p;                   //通信信道的promise
```

检测任务代码很简洁：

```cpp
…                                       //检测某个事件
p.set_value();                          //通知反应任务
```

反应任务代码也同样简单：

```cpp
…                                       //准备作出反应
p.get_future().wait();                  //等待对应于p的那个future
…                                       //对事件作出反应
```

像使用flag的方法一样，此设计不需要互斥锁，无论在反应线程调用`wait`之前检测线程是否设置了`std::promise`都可以工作，并且不受虚假唤醒的影响（只有条件变量才容易受到此影响）。与基于条件变量的方法一样，反应任务在调用`wait`之后是真被阻塞住的，不会一直占用系统资源。是不是很完美？

当然不是，基于*future*的方法没有了上述问题，但是有其他新的问题。比如，[Item38](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item38.md)中说明，`std::promise`和*future*之间有个共享状态，并且共享状态是动态分配的。因此你应该假定此设计会产生基于堆的分配和释放开销。

也许更重要的是，`std::promise`只能设置一次。`std::promise`和*future*之间的通信是**一次性**的：不能重复使用。这是与基于条件变量或者基于flag的设计的明显差异，条件变量和flag都可以通信多次。（条件变量可以被重复通知，flag也可以重复清除和设置。）

一次通信可能没有你想象中那么大的限制。假定你想创建一个挂起的系统线程。就是，你想避免多次线程创建的那种经常开销，以便想要使用这个线程执行程序时，避免潜在的线程创建工作。或者你想创建一个挂起的线程，以便在线程运行前对其进行设置这样的设置包括优先级或者核心亲和性（*core affinity*）。C++并发API没有提供这种设置能力，但是`std::thread`提供了`native_handle`成员函数，它的结果就是提供给你对平台原始线程API的访问（通常是POSIX或者Windows的线程）。这些低层次的API使你可以对线程设置优先级和亲和性。

假设你仅仅想要对某线程挂起一次（在创建后，运行线程函数前），使用`void`的*future*就是一个可行方案。这是这个技术的关键点：

```cpp
std::promise<void> p;
void react();                           //反应任务的函数
void detect()                           //检测任务的函数
{
    std::thread t([]                    //创建线程
                  {
                      p.get_future().wait();    //挂起t直到future被置位
                      react();
                  });
    …                                   //这里，t在调用react前挂起
    p.set_value();                      //解除挂起t（因此调用react）
    …                                   //做其他工作
    t.join();                           //使t不可结合（见条款37）
}
```

因为所有离开`detect`的路径中`t`都要是不可结合的，所以使用类似于[Item37](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item37.md)中`ThreadRAII`的RAII类很明智。代码如下：

```cpp
void detect()
{
    ThreadRAII tr(                      //使用RAII对象
        std::thread([]
                    {
                        p.get_future().wait();
                        react();
                    }),
        ThreadRAII::DtorAction::join    //有危险！（见下）
    );
    …                                   //tr中的线程在这里被挂起
    p.set_value();                      //解除挂起tr中的线程
    …
}
```

这样看起来安全多了。问题在于第一个“…”区域中（注释了“`tr`中的线程在这里被挂起”的那句），如果异常发生，`p`上的`set_value`永远不会调用，这意味着*lambda*中的`wait`永远不会返回。那意味着在*lambda*中运行的线程不会结束，这是个问题，因为RAII对象`tr`在析构函数中被设置为在（`tr`中创建的）那个线程上实行`join`。换句话说，如果在第一个“…”区域中发生了异常，函数挂起，因为`tr`的析构函数永远无法完成。

有很多方案解决这个问题，但是我把这个经验留给读者。（一个开始研究这个问题的好地方是我的博客*[The View From Aristeia](http://scottmeyers.blogspot.com/)*中，2013年12月24日的文章“[ThreadRAII + Thread Suspension = Trouble?](http://scottmeyers.blogspot.com/2013/12/threadraii-thread-suspension-trouble.html)”）这里，我只想展示如何扩展原始代码（即不使用RAII类）使其挂起然后取消挂起不仅一个反应任务，而是多个任务。简单概括，关键就是在`react`的代码中使用`std::shared_future`代替`std::future`。一旦你知道`std::future`的`share`成员函数将共享状态所有权转移到`share`产生的`std::shared_future`中，代码自然就写出来了。唯一需要注意的是，每个反应线程都需要自己的`std::shared_future`副本，该副本引用共享状态，因此通过`share`获得的`shared_future`要被在反应线程中运行的*lambda*按值捕获：

```cpp
std::promise<void> p;                   //跟之前一样
void detect()                           //现在针对多个反映线程
{
    auto sf = p.get_future().share();   //sf的类型是std::shared_future<void>
    std::vector<std::thread> vt;        //反应线程容器
    for (int i = 0; i < threadsToRun; ++i) {
        vt.emplace_back([sf]{ sf.wait();    //在sf的局部副本上wait；
                              react(); });  //emplace_back见条款42
    }
    …                                   //如果这个“…”抛出异常，detect挂起！
    p.set_value();                      //所有线程解除挂起
    …
    for (auto& t : vt) {                //使所有线程不可结合；
        t.join();                       //“auto&”见条款2
    }
}
```

使用*future*的设计可以实现这个功能值得注意，这也是你应该考虑将其应用于一次通信的原因。

**请记住：**

- 对于简单的事件通信，基于条件变量的设计需要一个多余的互斥锁，对检测和反应任务的相对进度有约束，并且需要反应任务来验证事件是否已发生。
- 基于flag的设计避免的上一条的问题，但是是基于轮询，而不是阻塞。
- 条件变量和flag可以组合使用，但是产生的通信机制很不自然。
- 使用`std::promise`和*future*的方案避开了这些问题，但是这个方法使用了堆内存存储共享状态，同时有只能使用一次通信的限制。
