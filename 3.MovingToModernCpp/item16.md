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

问题就是`roots`被声明为const，但不是线程安全的。













