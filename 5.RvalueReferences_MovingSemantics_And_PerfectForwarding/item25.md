## Item25: 在右值引用上使用`std::move`，在通用引用上使用`std::forward`

右值引用仅绑定可以移动的对象。如果你有一个右值引用参数，你就知道这个对象可能会被移动：

```cpp
class Widget {
  Widget(Widget&& rhs); //rhs definitely refers to an object eligible for moving
  ...
};
```

这是个例子，你将希望通过可以利用该对象右值性的方式传递给其他使用对象的函数。这样做的方法是将绑定次类对象的参数转换为右值。如Item23中所述，这不仅是`std::move`所做，而且是为它创建：

```cpp
class Widget {
public:
  Widget(Widget&& rhs) :name(std::move(rhs.name)), p(std::move(rhs.p)) {...}
  ...
private:
  std::string name;
  std::shared_ptr<SomeDataStructure> p;
};
```

另一方面（查看Item24），通用引用可能绑定到有资格移动的对象上。通用引用使用右值初始化时，才将其强制转换为右值。Item23阐释了这正是`std::forward`所做的：

```cpp

```

