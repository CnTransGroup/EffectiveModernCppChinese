## Item42: 考虑使用emplacement代替insertion

如果你拥有一个容器，例如`std::string`，那么当你通过插入函数（例如`insert, push_front, push_back`，或者对于`std::forward_list`， `insert_after`）添加新元素时，你传入的元素类型应该是`std::string`。毕竟，这就是容器里的内容。

逻辑上看来如此，但是并非总是如此。考虑如下代码：

```cpp
std::vector<std::string> vs; // container of std::string
vs.push_back("xyzzy"); // add string literal
```

这里，容量里内容是`std::string`，但是你试图通过`push_back`加入字符串字面量，即引号内的字符序列。字符转字面量并不是`std::string`，这意味着你传递给`push_back`的参数并不是容器里的内容类型。

`std::vector`的`push_back`被按左值和右值分别重载：

```cpp
template<class T, class Allocator = allocator<T>>
class vector {
public:
  ...
  void push_back(const &T x); // insert lvalue
  void push_back(T&& x); // insert rvalue
  ...
};
```

在`vs.push_back("xyzzy")`这个调用中，编译器看到参数类型（const char[6]）和`push_back`采用的参数类型（`std::string`的引用）之间不匹配。它们通过从字符串字面量创建一个`std::string`类型的临时变量来消除不匹配，然后传递临时变量给`push_back`。换句话说，编译器处理的这个调用应该像这样：

```cpp
vs.push_back(std::string("xyzzy")); // create temp std::string and pass it to push_back
```

代码编译并运行，皆大欢喜。除了对于性能执着的人意识到了这份代码不如预期的执行效率高。

为了创建`std::string`类型的临时变量，调用了`std::string`的构造器，但是这份代码并不仅调用了一次构造器，调用了两次，而且还调用了析构器。这发生在`push_back`运行时：

1. 一个`std::string`的临时对象从字面量"xyzzy"被创建。这个对象没有名字，我们可以称为*temp*，*temp*通过`std::string`构造器生成，因为是临时变量，所以*temp*是右值。
2. *temp*被传递给`push_back`的右值x重载函数。在`std::vector`的内存中一个x的副本被创建。这次构造器是第二次调用，在`std::vector`内部重新创建一个对象。（将x副本复制到`std::vector`内部的构造器是移动构造器，因为x传入的是右值，有关将右值引用强制转换为右值的信息，请参见Item25）。
3. 在`push_back`返回之后，*temp*被销毁，调用了一次`std::string`的析构器。

性能执着者（译者注：直译性能怪人）不禁注意到是否存在一种方法可以获取字符串字面量并将其直接传入到步骤2中的`std::string`内部构造，可以避免临时对象*temp*的创建与销毁。这样的效率最好，性能执着者也不会有什么意见了。

因为你是一个C++开发者，所以你会有高于平均水平的要求。如果你不是C++开发者，你可能也会同意这个观点（如果你根本不考虑性能，为什么你没在用python？）。所以让我来告诉你如何使得`push_back`达到最高的效率。就是不使用`push_back`，你需要的是`emplace_back`。

`emplace_back`就是像我们想要的那样做的：直接把传递的参数（无论是不是`std::string`）直接传递到`std::vector`内部的构造器。没有临时变量会生成：

```cpp
vs.emplace_back("xyzzy"); // construct std::string inside vs directly from "xyzzy"
```

`emplace_back`使用完美转发，因此只要你没有遇到完美转发的限制（参见Item30），就可以传递任何参数以及组合到`emplace_back`。比如，如果你在vs传递一个字符和一个数量给`std::string`构造器创建`std::string`，代码如下：

```cpp
vs.emplace_back(50, 'x'); // insert std::string consisting of 50 'x' characters
```

`emplace_back`可以用于每个支持`push_back`的容器。类似的，每个支持`push_front`的标准容器支持`emplace_front`。每个支持`insert`（除了`std::forward_list`和`std::array`）的标准容器支持`emplace。`关联容器提供`emplace_hint`来补充带有“hint”迭代器的插入函数，`std::forward_list`有`emplace_after`来匹配`insert_after`。

使得emplacement函数功能优于insertion函数的原因是它们灵活的接口。insertion函数接受对象来插入，而emplacement函数接受构造器接受的参数插入。这种差异允许emplacement函数避免临时对象的创建和销毁。

因为可以传递容器内类型给emplacement函数（该参数使函数执行复制或者移动构造器），所以即使insertion函数不会构造临时对象，也可以使用emplacement函数。在这种情况下，insertion和emplacement函数做的是同一件事，比如：

```cpp
std::string queenOfDisco("Donna Summer");
```

下面的调用都是可行的，效率也一样：

```cpp
vs.push_back(queenOfDisco); // copy-construct queenOfDisco
vs.emplace_back(queenOfDisco); // ditto
```

因此，emplacement函数可以完成insertion函数的所有功能。并且有时效率更高，至上在理论上，不会更低效。那为什么不在所有场合使用它们？