## Item16:让const成员函数线程安全
条款16: 让const成员函数线程安全

如果我们工作在数学领域中，我们就会发现用一个类表示多项式是很方便的。在这个类中，使用一个函数来计算多项式的根是很有用的。即，多项式为零的时候。这样的一个函数它不会更改多项式，因此，它自然被声明为const函数。

```c++
class Polynomial {
public:
    using RootsType =           // 数据结构保存多项式为零的值
      std::vector<double>;      // （“using” 的信息查看条款9）

    RootsType roots() const;

};
```

计算多项式的根是很复杂的，因此如果不需要的话，我们就不做。如果必须做，我们肯定不会只做一次。所以，如果必须计算它们，就缓存多项式的根，然后实现`roots`来返回缓存的值。下面是最基本的实现：

```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        if (!rootsAreVaild) {                // 如果缓存不可用
                                             // 计算根
            rootsAreVaild = true;            // 用`rootVals`存储它们
        }
        
        return rootVals;
    }
    
private:
    mutable bool rootsAreVaild() { false };    // initializers 的更多信息
    mutable RootsType rootVals() {};           // 请查看条款7
};
```

从概念上讲，`roots`并不改变它所操作的多项式对象。但是作为缓存的一部分，它也许会改变`rootVals`和`rootsAreVaild`的值。这就是`mutable`的经典使用样例，这也是为什么它是数据成员声明的一部分。

假设现在有两个线程同时调用`Polynomial`对象的`roots`方法:

```c++
Polynomial p;


/*------ Thread 1 ------*/      /*-------- Thread 2 --------*/
auto rootsOfp = p.roots();      auto valsGivingZero = p.roots();
```

这些用户代码是非常合理的。`roots`是const 成员函数，那就表示着它是一个读操作。在没有同步的情况下，让多个线程执行读操作是安全的。它最起码应该做到这点。在本例中却没有做到线程安全。因为在`roots`中，这些线程中的一个或两个可能尝试修改成员变量`rootsAreVaild`和`rootVals`。这就意味着在没有同步的情况下，这些代码会有不同的线程读写相同的内存，这就`data race`的定义。这段代码的行为是未定义的。

问题就是`roots`被声明为const，但不是线程安全的。const声明在c++11和c++98 中都是正确的（检索多项式的根并不会更改多项式的值），因此需要纠正的是线程安全的缺乏。

解决这个问题最普遍简单的方法就是-------使用互斥锁：
```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);		// lock mutex
        
        if (!rootsAreVaild) {					// 如果缓存无效
                                                // 计算/存储roots
            rootsAreVaild = true;
        }
        
        return rootsVals;
    }											// unlock mutex
    
private:
    mutable std::mutex m;
    mutable bool rootsAreVaild { false };
    mutable RootsType rootsVals {};
};
```

`std::mutex m`被声明为`mutable`，因为锁定和解锁它的都是non-const函数。在`roots`（const成员函数）中，`m`将被视为const对象。
值得注意的是，因为`std::mutex`是一种`move-only`的类型（一种可以移动但不能复制的类型），所以将`m`添加进多项式中的副作用是使它失去了被复制的能力。不过，它仍然可以移动。

在某些情况下，互斥量是过度的（？）。例如，你所做的只是计算成员函数被调用了多少次。使用`std::atomic` 修饰的counter（保证其他线程视这个操作为不可分割的发生，参见item40）。（然而它是否轻量取决于你使用的硬件和标准库中互斥量的实现。）以下是如何使用`std::atomic`来统计调用次数。

```c++
class Point {									// 2D point
public:
    // noexcept的使用参考Item 14
    double distanceFromOrigin() const noexcept
    {
        ++callCount;                            // 原子的递增
        
        return std::sqrt((x * x) + (y * y));
    }

private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```

与`std::mutex`一样，`std::atomic`是`move-only`类型，所以在`Point`中调用`Count`的意思就是`Point`也是`move-only`的。

因为对`std::atomic`变量的操作通常比互斥量的获取和释放的消耗更小，所以你可能更倾向与依赖`std::atomic`。例如，在一个类中，缓存一个开销昂贵的`int`，你就会尝试使用一对`std::atomic`变量而不是互斥锁。

```c++
class Widget {
public:
    
    int magicValue() const
    {
        if (cacheVaild) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;				// 第一步
            cacheVaild = true;						// 第二步
            return cachedVaild;
        }
    }
    
private:
    mutable std::atomic<bool> cacheVaild{ false };
    mutable std::atomic<int> cachedValue;
};
```

着是可行的的，但有时运行的会比它应该做到的更加困难。考虑：

+ 一个线程调用`Widget::magicValue`，将`cacheValid`视为`false`，执行这两个昂贵的计算，并将它们的和分配给`cachedValue`。
+ 此时，第二个线程调用`Widget::magicValue`，也将`cacheValid`视为`false`，因此执行刚才完成的第一个线程相同的计算。（这里的“第二个线程”实际上可能是其他几个线程。）

这种行为与使用缓存的目的背道而驰。将`cachedValue`和`CacheValid`的顺序交换可以解决这个问题，但结果会更糟：

```c++
class Widget {
public:
    
    int magicValue() const
    {
        if (cacheVaild) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheVaild = true;						// 第一步
            return cachedValue = val1 + val2;		// 第二步
        }
    }
    
}
```

