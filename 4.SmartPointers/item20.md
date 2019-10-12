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

你需要的是一个原子操作实现检查是否过期，如果没有过期就访问所指对象。有两种方式可以从`std::weak_ptr`上创建`std::shared_ptr`完成，具体用哪种取决于`std::weak_ptr`过期时你希望`std::shared_ptr`表现出什么行为。一种形式是`std::weak_ptr::lock`，它返回一个`std::shared_ptr`，如果`std::weak_ptr`过期这个`std::shared_ptr`为空：
```cpp
std::shared_ptr<Widget> spw1 = wpw.lock();  // 如果wpw过期
 											// spw1为空
auto spw2 = wpw.lock(); 					// 和上面一样，只是使用auto
```
另一种形式是以`std::weak_ptr`为实参构造`std::shared_ptr`。这种情况中，如果`std::weak_ptr`过期，一个异常会抛出：
```cpp
std::shared_ptr<Widget> spw3(wpw);			// 如果wpw过期，抛出std::bad_weak_ptr异常
```
但是你可能还想知道为什么`std::weak_ptr`就有用了。考虑一个工厂函数，它基于一个UID从只读对象上产出智能指针。根据Item18的描述，工厂函数会返回一个该对象类型的`std::unique_ptr`：
```cpp
	std::unique_ptr<const Widget> loadWidget(WidgetID id);
```
如果调用**loadWidget**是一个昂贵的操作（比如它操作文件或者数据库I/O）并且对于ID来重复使用很常见，一个合理的优化是除了完成**loadWidget**做的事情之外再缓存它的结果。当请求获取一个Widget时阻塞在缓存操作上这本身也会导致性能问题，所以另一个合理的优化可以是当Widget不再使用的时候销毁它的缓存。

对于可缓存的工厂函数，返回`std::unique_ptr`不是好的选择。调用者应该明确接受缓存后的对象，调用者也应该明确这些对象的生命周期，但是缓存本身也需要一个指针指向它所缓的对象。缓存的执着需要弹出它缓的对象是否已经没了，因为当客户段用完工厂返回的对象后，那个对象将会销毁。因此，缓存指针应该是`std::weak_ptr`——一个可以探测所缓对象是否没了的指针。这意味着工厂函数的返回类型应该是`std::shared_ptr`，因为仅当对象生命周期由`std::shared_ptr`管理时`std::weak_ptr`才能探测是否该对象是否为没了。

下面是一个粗制滥造的缓存版本的**loadWidget**实现：
```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
	static std::unordered_map<WidgetID,
	std::weak_ptr<const Widget>> cache;
	auto objPtr = cache[id].lock(); 	// objPtr是std::shared_ptr
										// 指向被缓对象（为空表示被缓对象为空）
	if (!objPtr) { 						// 如果没在缓存中
		objPtr = loadWidget(id); 		// 加载
		cache[id] = objPtr; 			// 缓它
	}
	return objPtr;
}
```