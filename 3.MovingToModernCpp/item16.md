## 条款十六：让`const`成员函数线程安全
**Item 16: Make `const` member functions thread safe**

如果我们在数学领域中工作，我们就会发现用一个类表示多项式是很方便的。在这个类中，使用一个函数来计算多项式的根是很有用的，也就是多项式的值为零的时候。这样的一个函数它不会更改多项式。所以，它自然被声明为`const`函数。

```c++
class Polynomial {
public:
    using RootsType =           //数据结构保存多项式为零的值
          std::vector<double>;  //（“using” 的信息查看条款9）
    …
    RootsType roots() const;
    …
};
```

计算多项式的根是很复杂的，因此如果不需要的话，我们就不做。如果必须做，我们肯定不想再做第二次。所以，如果必须计算它们，就缓存多项式的根，然后实现`roots`来返回缓存的值。下面是最基本的实现：

```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        if (!rootsAreValid) {               //如果缓存不可用
            …                               //计算根
                                            //用rootVals存储它们
            rootsAreValid = true;
        }
        
        return rootVals;
    }
    
private:
    mutable bool rootsAreValid{ false };    //初始化器（initializer）的
    mutable RootsType rootVals{};           //更多信息请查看条款7
};
```

从概念上讲，`roots`并不改变它所操作的`Polynomial`对象。但是作为缓存的一部分，它也许会改变`rootVals`和`rootsAreValid`的值。这就是`mutable`的经典使用样例，这也是为什么它是数据成员声明的一部分。

假设现在有两个线程同时调用`Polynomial`对象的`roots`方法:

```c++
Polynomial p;
…

/*------ Thread 1 ------*/      /*-------- Thread 2 --------*/
auto rootsOfp = p.roots();      auto valsGivingZero = p.roots();
```

这些用户代码是非常合理的。`roots`是`const`成员函数，那就表示着它是一个读操作。在没有同步的情况下，让多个线程执行读操作是安全的。它最起码应该做到这点。在本例中却没有做到线程安全。因为在`roots`中，这些线程中的一个或两个可能尝试修改成员变量`rootsAreValid`和`rootVals`。这就意味着在没有同步的情况下，这些代码会有不同的线程读写相同的内存，这就是数据竞争（*data race*）的定义。这段代码的行为是未定义的。

问题就是`roots`被声明为`const`，但不是线程安全的。`const`声明在C++11中与在C++98中一样正确（检索多项式的根并不会更改多项式的值），因此需要纠正的是线程安全的缺乏。

解决这个问题最普遍简单的方法就是——使用`mutex`（互斥量）：
```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);       //锁定互斥量
        
        if (!rootsAreValid) {                   //如果缓存无效
            …                                   //计算/存储根值
            rootsAreValid = true;
        }
        
        return rootsVals;
    }                                           //解锁互斥量
    
private:
    mutable std::mutex m;
    mutable bool rootsAreValid { false };
    mutable RootsType rootsVals {};
};
```

`std::mutex m`被声明为`mutable`，因为锁定和解锁它的都是non-`const`成员函数。在`roots`（`const`成员函数）中，`m`却被视为`const`对象。

值得注意的是，因为`std::mutex`是一种只可移动类型（*move-only type*，一种可以移动但不能复制的类型），所以将`m`添加进`Polynomial`中的副作用是使`Polynomial`失去了被复制的能力。不过，它仍然可以移动。

在某些情况下，互斥量的副作用显会得过大。例如，如果你所做的只是计算成员函数被调用了多少次，使用`std::atomic` 修饰的计数器（保证其他线程视它的操作为不可分割的整体，参见[item40](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item40.md)）通常会是一个开销更小的方法。（然而它是否轻量取决于你使用的硬件和标准库中互斥量的实现。）以下是如何使用`std::atomic`来统计调用次数。

```c++
class Point {                                   //2D点
public:
    …
    double distanceFromOrigin() const noexcept  //noexcept的使用
    {                                           //参考条款14
        ++callCount;                            //atomic的递增
        
        return std::sqrt((x * x) + (y * y));
    }

private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```

与`std::mutex`一样，`std::atomic`是只可移动类型，所以在`Point`中存在`callCount`就意味着`Point`也是只可移动的。

因为对`std::atomic`变量的操作通常比互斥量的获取和释放的消耗更小，所以你可能会过度倾向与依赖`std::atomic`。例如，在一个类中，缓存一个开销昂贵的`int`，你就会尝试使用一对`std::atomic`变量而不是互斥量。

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;              //第一步
            cacheValid = true;                      //第二步
            return cachedValid;
        }
    }
    
private:
    mutable std::atomic<bool> cacheValid{ false };
    mutable std::atomic<int> cachedValue;
};
```

这是可行的，但难以避免有时出现重复计算的情况。考虑：

+ 一个线程调用`Widget::magicValue`，将`cacheValid`视为`false`，执行这两个昂贵的计算，并将它们的和分配给`cachedValue`。
+ 此时，第二个线程调用`Widget::magicValue`，也将`cacheValid`视为`false`，因此执行刚才完成的第一个线程相同的计算。（这里的“第二个线程”实际上可能是其他**几个**线程。）

这种行为与使用缓存的目的背道而驰。将`cachedValue`和`CacheValid`的赋值顺序交换可以解决这个问题，但结果会更糟：

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheValid = true;                      //第一步
            return cachedValue = val1 + val2;       //第二步
        }
    }
    …
}
```

假设`cacheValid`是false，那么：

+ 一个线程调用`Widget::magicValue`，刚执行过`cacheValid`被设置成`true`的点。
+ 在这时，第二个线程调用`Widget::magicValue`，检查`cacheValid`。看到它是`true`，就返回`cacheValue`，即使第一个线程还没有给它赋值。因此返回的值是不正确的。

这里有一个坑。对于需要同步的是单个的变量或者内存位置，使用`std::atomic`就足够了。不过，一旦你需要对两个以上的变量或内存位置作为一个单元来操作的话，就应该使用互斥量。对于`Widget::magicValue`是这样的。

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        std::lock_guard<std::mutex> guard(m);   //锁定m
        
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;
            return cachedValue;
        }
    }                                           //解锁m
    …

private:
    mutable std::mutex m;
    mutable int cachedValue;                    //不再用atomic
    mutable bool cacheValid{ false };           //不再用atomic
};

```

现在，这个条款是基于，多个线程可以同时在一个对象上执行一个`const`成员函数这个假设的。如果你不是在这种情况下编写一个`const`成员函数——你可以**保证**在一个对象上永远不会有多个线程执行该成员函数——该函数的线程安全是无关紧要的。比如，为独占单线程使用而设计的类的成员函数是否线程安全并不重要。在这种情况下，你可以避免因使用互斥量和`std::atomics`所消耗的资源，以及包含它们的类只能使用移动语义带来的副作用。然而，这种线程无关的情况越来越少见，而且很可能会越来越少。可以肯定的是，`const`成员函数应支持并发执行，这就是为什么你应该确保`const`成员函数是线程安全的。

**请记住：**

+ 确保`const`成员函数线程安全，除非你**确定**它们永远不会在并发上下文（*concurrent context*）中使用。
+ 使用`std::atomic`变量可能比互斥量提供更好的性能，但是它只适合操作单个变量或内存位置。
