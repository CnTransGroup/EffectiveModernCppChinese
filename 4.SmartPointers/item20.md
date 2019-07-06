## Item 20:像std::shared_ptr一样使用std::weak_ptr可能造成dangle


自相矛盾的是，如果有一个像`std::shared_ptr`的指针但是不参与资源所有权共享的指针是很方便的。换句话说，类似`std::shared_ptr`的指针但是不影响对象的引用计数。这种类型的智能指针必须要解决一个`std::shared_ptr`不存在的问题：可能指向已经销毁的对象。一个真正的智能指针应该跟踪所值对象，在dangle时知晓，比如当指向对象不再存在。那就是对`std::weak_ptr`最精确的描述。

你可能想知道什么时候该用`std::weak_ptr`。你可能想知道关于`std::weak_ptr`API的更多。它什么都好除了不太智能。`std::weak_ptr`不能解引用，也不能测试是否为空值。因为`std::weak_ptr`不是一个独立的智能指针。它是`std::shared_ptr`的增强。

这种关系在它创生之时就建立了。`std::weak_ptr`通常从`std::shared_ptr`上创建。当从`std::shared_ptr`上创建`std::weak_ptr`时两者指向相同的地方，但是`std::weak_ptr`不会影响所指对象的引用计数：
```cpp
auto spw = 					// spw构造后
std::make_shared<Widget>(); // 指向Widget
							// 引用计数(RC)为1. (参见
							// Item 21对于std::make_shared的描述)
…
std::weak_ptr<Widget> wpw(spw); // wpw指向相同的Widget
								// RC保持为1
…
spw = nullptr; 					// RC 变为0，并且
								// Widget销毁
								// wpw不再dangle
```
`std::weak_ptr`用**expired**来表示对象已经dangle。你可以用它直接做测试：
```CPP
if (wpw.expired()) … // 如果wpw不再指向对象
```
但是通常你期望的是检查`std::weak_ptr`缺少解引用操作，没有办法写这样的代码。即使有，将检查和解引用分开会引入竞态条件：在调用**expired**和解引用操作之间，另一个线程可能对是否过期，如果没有过期就再获取所指对象。听起来比做起来容易。因为`std::weak_ptr`缺少解引用操作，没有办法写这样的代码。即使有，将检查和解引用分开会引入竞态条件：在调用**expired**和解引用操作之间，另一个线程可能对指向的对象重新赋值或者析构，并由此造成对象已析构。这种情况下，你的解引用将会产生未定义行为。

你需要的是一个原子操作实现检查是否过期，如果没有过期就访问所指对象。可以从`std::weak_ptr`上创建`std::shared_ptr`完成。