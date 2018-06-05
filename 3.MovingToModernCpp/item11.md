## Item 11:优先考虑使用deleted函数而非使用未定义的私有声明
条款11:优先考虑使用deleted函数而非使用未定义的私有声明

如果你写的代码要被其他人使用，你不想让他们调用某个特殊的函数，你通常不会声明这个函数。无声明，不函数。简简单单！但有时C++会给你自动声明一些函数，如果你想防止客户调用这些函数，事情就不那么简单了。

上述场景见于特殊的成员函数，即当有必要时C++自动生成的那些函数。Item 17 详细讨论了这些函数，但是现在，我们只关心拷贝构造函数和拷贝赋值运算符重载。This chapter is largely devoted to common practices in
C++98 that have been superseded by better practices in C++11, and in C++98, if you
want to suppress use of a member function, it’s almost always the copy constructor,
the assignment operator, or both.

在C++98中防止调用这些函数的方法是将它们声明为私有成员函数。举个例子，在C++ 标准库*iostream*继承链的顶部是模板类`basic_ios`。所有`istream`和`ostream`类都继承此类(直接或者间接)。拷贝`istream`和`ostream`是不合适的，因为要进行哪些操作是模棱两可的。比如一个`istream`对象，代表一个输入值的流，流中有一些已经被读取，有一些可能马上要被读取。如果一个`istream`被拷贝，需要像拷贝将要被读取的值那样也拷贝已经被读取的值吗？解决这个问题最好的方法是不定义这个操作。直接禁止拷贝流。

要使`istream`和`ostream`类不可拷贝，`basic_ios`在C++98中是这样声明的(包括注释)：
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
private:
    basic_ios(const basic_ios& ); // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
```
将它们声明为私有成员可以防止客户端调用这些函数。故意不定义它们意味着假如还是有代码用它们就会在链接时引发缺少函数定义(missing function definitions)错误。

在C++11中有一种更好的方式，只需要使用相同的结尾：用`= delete`将拷贝构造函数和拷贝赋值运算符标记为`deleted`函数。上面相同的代码在C++11中是这样声明的：
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};
```
删除这些函数(译注：添加"= delete")和声明为私有成员可能看起来只是方式不同，别无其他区别。其实还有一些实质性意义。`deleted`函数不能以任何方式被调用，即使你在成员函数或者友元函数里面调用`deleted`函数也不能通过编译。这是较之C++98行为的一个改进，后者不正确的使用这些函数在链接时才被诊断出来。
