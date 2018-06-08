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

All these requirements for overriding mean that small mistakes can make a big difference.
Code containing overriding errors is typically valid, but its meaning isn’t what
you intended. You therefore can’t rely on compilers notifying you if you do something
wrong. For example, the following code is completely legal and, at first sight,
looks reasonable, but it contains no virtual function overrides—not a single derived
class function that is tied to a base class function. Can you identify the problem in
each case, i.e., why each derived class function doesn’t override the base class function
with the same name?