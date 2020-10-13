## Item21: 首选std::make_unique和std::make_shared相比直接使用new

让我们首先介绍一下std::make_unique和std::make_shared，std::make_shared是C++11的部分，但是不幸的是，std::make_unique是C++14才引入到标准库的。如果你使用的是C++11，不用担心，因为std::make_unique的基本版本很容易自己实现，看：

```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
  return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

如你所看到的，make_unique只是通过完美转发参数到对象的构造器中被创建，从new出来的原始指针构造一个std::unique_ptr的只能指针，然后返回。这种形式的函数不支持数组和自定义删除器（查看item18），但是它表明，只需要一点点努力，你就可以自行实现需要的make_unique。请记住不要把自己的版本放在命名空间std中，因为在升级到C++14之后，避免出现冲突。

Std::make_unique和make_shared是三大make函数中的两个：接收任意参数集的函数，将其完美转发给动态分配对象的构造函数，然后返回指向该对象的只能指针。第三个make函数是std::allocate_shared。它的行为就像std::make_shared，不同之处在于它的第一个参数是用于动态分配内存的分配器对象。

使用和不使用make函数进行只能指针创建的最不必要的比较，也能揭示为什么此类函数是首选的原因。考虑：

```cpp
auto upw1(std::make_unique<Widget>());
std::unique_ptr<Widget> upw2(new Widget);
auto spw1(std::make_shared<Widget>());
std::shared_ptr<Widget> spw2(new Widget);
```

已经强调了本质区别：从写法上使用new版本会重复创建的类型，但是make版本不会。重复类型与软件工程的一项主要原则相违背