## Item 12:使用override声明重载函数
条款12:使用override声明重载函数

在C++面向对象的世界里，涉及的概念有类，继承，虚函数。这个世界最基本的概念是派生类的虚函数重写基类同名函数。令人遗憾的是虚函数重写可能一不小心就错了。给人感觉语言的这一部分设计观点是墨菲定律不是用来遵守的，只是值得尊敬的。

鉴于"重写"听起来像"重载"，尽管两者完全不相关，下面就通过一个派生类和基类来说明什么是虚函数重写：
```cpp
class Base {
public:
	virtual void doWork(); // 基类虚函数
	…
};
class Derived: public Base {
	public:
	virtual void doWork(); // 重写Base::doWork(这里"virtual"是可以省略的)
	… 
}; 	
std::unique_ptr<Base> upb = 		// 创建基类指针
	std::make_unique<Derived>(); 	// 指向派生类对象
									// 关于std：：make_unique请
… 									// 参见Item1
upb->doWork(); 	// 通过基类指针调用doWork
				// 实际上是派生类的doWork
				// 函数被调用
```
要想重写一个函数，必须满足下列要求：

+ 基类函数必须是`virtual`
+ 基类和派生类函数名必须完全一样（除非是析构函数
+ 基类和派生类函数参数必须完全一样
+ 基类和派生类函数常量性(constness)必须完全一样
+ 基类和派生类函数的返回值和异常说明(exception specifications)必须兼容
除了这些C++98就存在的约束外，C++11又添加了一个：
+ 函数的引用限定符（reference qualifiers）必须完全一样。成员函数的引用限定符是C++11很少抛头露脸的特性，所以如果你从没听过它无需惊讶。它可以限定成员函数只能用于左值或者右值。成员函数不需要`virtual`也能使用它们：
```cpp
class Widget {
public:
	…
	void doWork() &; //只有*this为左值的时候才能被调用
	void doWork() &&; //只有*this为右值的时候才能被调用
}; 
…
Widget makeWidget(); 	// 工厂函数（返回右值）
Widget w; 				// 普通对象（左值）
…
w.doWork(); // 调用被左值引用限定修饰的Widget::doWork版本
			// (即Widget::doWork &)
makeWidget().doWork(); // 调用被右值引用限定修饰的Widget::doWork版本
						// (即Widget::doWork &&)
```
后面我还会提到引用限定符修饰成员函数，但是现在，只需要记住如果基类的虚函数有引用限定符，派生类的重写就必须具有相同的引用限定符。如果没有，那么新声明的函数还是属于派生类，但是不会重写父类的任何函数。

这么多的重写需求意味着哪怕一个小小的错误也会造成巨大的不同。
代码中包含重写错误通常是有效的，但它的意图不是你想要的。因此你不能指望当你犯错时编译器能通知你。比如，下面的代码是完全合法的，咋一看，还很有道理，但是它包含了非虚函数重写。你能识别每个case的错误吗，换句话说，为什么派生类函数没有重写同名基类函数？
```cpp
class Base {
public:
	virtual void mf1() const;
	virtual void mf2(int x);
	virtual void mf3() &;
	void mf4() const;
};
class Derived: public Base {
public:
	virtual void mf1();
	virtual void mf2(unsigned int x);
	virtual void mf3() &&;
	void mf4() const;
};
```
需要一点帮助吗？
+ `mf1`在基类声明为`const`,但是派生类没有这个常量限定符
+ `mf2`在基类声明为接受一个`int`参数，但是在派生类声明为接受`unsigned int`参数
+ `mf3`在基类声明为左值引用限定，但是在派生类声明为右值引用限定
+ `mf4`在基类没有声明为虚函数
你可能会想，“哎呀，实际操作的时候，这些warnings都能被编译器探测到，所以我不需要担心。”可能你说的对，也可能不对。就我目前检查的两款编译器来说，这些代码编译时没有任何warnings，即使我开启了输出所有warnings（其他编译器可能会为这些问题的部分输出warnings，但不是全部）

由于正确声明派生类的重写函数很重要，但很容易出错，C++11提供一个方法让你可以显式的将派生类函数指定为应该是基类重写版本：将它声明为`override`。还是上面那个例子，我们可以这样做：
```cpp
class Derived: public Base {
	public:
	virtual void mf1() override;
	virtual void mf2(unsigned int x) override;
	virtual void mf3() && override;
	virtual void mf4() const override;
};
```
代码不能编译，当然了，因为这样写的时候，编译器会抱怨所有与重写有关的问题。这也是你想要的，以及为什么要在所有重写函数后面加上`override`。使用`override`的代码编译时看起来就像这样（假设我们的目的是重写基类的所有函数）:
```cpp
class Base {
public:
	virtual void mf1() const;
	virtual void mf2(int x);
	virtual void mf3() &;
	virtual void mf4() const;
};
class Derived: public Base {
public:
	virtual void mf1() const override;
	virtual void mf2(int x) override;
	virtual void mf3() & override;
	void mf4() const override; // 可以添加virtual，但不是必要
}; 
```
Note that in this example, part of getting things to work involves declaring mf4 virtual
in Base. Most overriding-related errors occur in derived classes, but it’s possible
for things to be incorrect in base classes, too.
A policy of using override on all your derived class overrides can do more than just
enable compilers to tell you when would-be overrides aren’t overriding anything. It
can also help you gauge the ramifications if you’re contemplating changing the signature
of a virtual function in a base class. If derived classes use override everywhere,
you can just change the signature, recompile your system, see how much damage
you’ve caused (i.e., how many derived classes fail to compile), then decide whether
the signature change is worth the trouble. Without override, you’d have to hope you
have comprehensive unit tests in place, because, as we’ve seen, derived class virtuals that are supposed to override base class functions, but don’t, need not elicit compiler
diagnostics.
C++ has always had keywords, but C++11 introduces two contextual keywords, over
ride and final.2 These keywords have the characteristic that they are reserved, but
only in certain contexts. In the case of override, it has a reserved meaning only
when it occurs at the end of a member function declaration. That means that if you
have legacy code that already uses the name override, you don’t need to change it
for C++11:
```cpp
class Warning { // potential legacy class from C++98
public:
	…
void override(); // legal in both C++98 and C++11
};
```
That’s all there is to say about override, but it’s not all there is to say about member
function reference qualifiers. I promised I’d provide more information on them later,
and now it’s later.
If we want to write a function that accepts only lvalue arguments, we declare a nonconst
lvalue reference parameter:
```cpp
void doSomething(Widget& w); // accept
If we want to write a function that accepts only rvalue arguments, we declare an
rvalue reference parameter:
```cpp
void doSomething(Widget&& w); // accepts only rvalue Widgets
```
Member function reference qualifiers simply make it possible to draw the same distinction
for the object on which a member function is invoked, i.e., *this. It’s precisely
analogous to the const at the end of a member function declaration, which
indicates that the object on which the member function is invoked (i.e., *this) is
const.
The need for reference-qualified member functions is not common, but it can arise.
For example, suppose our Widget class has a std::vector data member, and we
offer an accessor function that gives clients direct access to it:
```cpp
class Widget {
public:
	using DataType = std::vector<double>; // see Item 9 for
	… // info on "using"
	DataType& data() { return values; }
	…
	private:
	DataType values;
};
```