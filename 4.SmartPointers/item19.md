## Item 19:对于共享资源使用std::shared_ptr
条款十九:对于共享资源使用std::shared_ptr

程序员使用带垃圾回收的语言指着C++笑看他们如何防止资源泄露。“真是原始啊！”它们嘲笑着说。“你们没有从1960年Lisp那里得到启发吗，机器应该自己管理资源生命周期而不应该依赖人类。”C++程序眼滚动眼珠。“你得到的启发就是只有内存算资源，其他资源释放都是非确定性的？我们更喜欢通用，可预料的销毁，谢谢你。”但我们的虚张声势可能底气不足。因为垃圾回收真的很方便，而且手动管理生命周期真的就像是使用石头小刀和兽皮制作可记忆内存电路。为什么我们不能同时有两个完美的世界：一个自动工作的世界（垃圾回收），一个销毁可预测的世界（析构）？

C++11中的`std::shared_ptr`将两者组合了起来。一个通过`std::shared_ptr`访问的对象其生命周期由指向它的指针们共享所有权（shbared ownership）。没有特定的`std::shared_ptr`拥有该对象。相反，所有指向它的`std::shared_ptr`都能相互合作确保它不再使用的那个点进行析构。当最后一个`std::shared_ptr`到达那个点，`std::shared_ptr`会销毁它所指向的对象。就垃圾回收来说，客户端不需要关心指向对象的生命周期，而对象的析构是确定性的。

`std::shared_ptr`通过引用计数来确保它是否是最后一个指向某种资源的制作，引用计数即资源和一个值关联起来，这个值会跟踪有多少`std::shared_ptr`指向该资源。`std::shared_ptr`构造函数递增引用计数值（通常——产检下面），析构函数递减值，拷贝复制运算符可能递增也可能递减。（如果sp1和sp2是`std::shared_ptr`并且指向不同对象，赋值运算符`sp1=sp2`会使sp1指向sp2指向的对象。直接效果就是sp1引用计数减一，sp2引用计数加一。）如果`std::shared_ptr`发现引用计数值为零，没有其他`std::shared_ptr`指向该资源，它就会销毁资源。

引用计数暗示着性能问题：
+ **std::shared_ptr大小是原始指针的两倍**，因为它内部包含一个指向资源的原始指针，还包含一个资源的引用技术值
+ **引用计数值必须动态分配**。 理论上，引用计数与所指对象关联起来，但是被指向的对象不知道这件事情。因此它们没有办法存放一个引用计数值。Item21会解释使用`std::make_shared`创建`std::shared_ptr`可以避免引用计数值的动态分配，但是还有一些`std::make_shared`不能使用的场景，这时候引用计数就会动态分配。
+ **递增递减引用计数必须是原子性的**，因为多个reader、writer可能在不同的线程。比如，指向某种资源的一个`std::shared_ptr`可能在一个线程执行析构，在另一个不同的线程，`std::shared_ptr`指向相同的对象，但是执行的确实拷贝操作。原则操作通常比非原子操作要慢，所以即使是引用计数，你也应该假定读写它们是存在开销的。

我写道`std::shared_ptr`构造函数只是“通常”递增指向对象的引用计数值会不会让你有点好奇？创建一个指向对象的`std::shared_ptr`总是产生一个或多个指向对象的`std::shared_ptr`，所以为什么我没说**总是**增加引用计数值？

原因是移动构造函数。从另一个`std::shared_ptr`移动构造新`std::shared_ptr`会将原来的`std::shared_ptr`设置为null，那意味着老的`std::shared_ptr`不再指向资源，同时新的`std::shared_ptr`指向资源。这样的结果就是不需要修改引用计数值。因此移动`std::shared_ptr`会比拷贝它要快：拷贝要求递增引用计数值，移动不需要。移动赋值运算符同理，所以移动赋值运算符也比拷贝赋值运算符快。

类似`std::unique_ptr`（参加Item18），`std::shared_ptr`使用**delete**作为默认的资源销毁器，但是它也支持自定义的销毁器。这种支持有别于`std::unique_ptr`。对于`std::unique_ptr`来说，销毁器类型是智能指针类型的一部分。对于`std::shared_ptr`则不是：
```CPP
auto loggingDel = [](Widget *pw) 	//自定义销毁器
				{ 					// (和Item 18一样)
					makeLogEntry(pw);
					delete pw;
				};

std::unique_ptr< 					// 销毁器类型是
	Widget, decltype(loggingDel) 	// ptr类型的一部分
	> upw(new Widget, loggingDel);
std::shared_ptr<Widget> 			// 销毁器类型不是
spw(new Widget, loggingDel); 		// ptr类型的一部分
```
`std::shared_ptr`的设计更为灵活。考虑有两个`std::shared_ptr`，每个自带不同的销毁器（比如通过lambda表达式自定义销毁器）：
```CPP
auto customDeleter1 = [](Widget *pw) { … };
auto customDeleter2 = [](Widget *pw) { … }; 
std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);
```
因为**pw1**和**pw2**有相同的类型，所以它们都可以放到存放那个类型的对象的容器中：
```CPP
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```
它们也能相互赋值，也可以传入参数为`std::shared_ptr<Widget>`的函数。但是`std::unique_ptr`就不行，因为`std::unique_ptr`把销毁器视作类型的一部分。

另一个不同于`std::unique_ptr`的地方是，指定自定义销毁器不会改变`std::shared_ptr`对象的大小。不管销毁器是什么，一个`std::shared_ptr`对象都是两个指针大小。这是个好消息，但是它应该让你隐隐约约不安。自定义销毁器可以是函数对象，函数对象可以包含任意多的数据。它意味着函数对象是任意大的。`std::shared_ptr`怎么能引用一个任意大的销毁器而不使用更多的内存？

它不能。它必须使用更多的内存。然而，那部分内存不是`std::shared_ptr`对象的一部分。那部分在堆上面，只要`std::shared_ptr`之定义了分配器，那部分内存随便在哪都行。我前面提到了`std::shared_ptr`对象包含了所指对象的引用计数值。每次，但是有点误导人。因为引用计数是另一个更大的数据结构的一部分，那个数据结构通常叫做**控制块**（control block）。控制块包含除了引用计数值外的一个自定义销毁器的拷贝，当然前提是存在自定义销毁器。如果用户还指定了自定义分配器，控制器也会包含一个分配器的拷贝。控制块可能还包含一些额外的数据，正如Item21提到的，一个次级引用计数weak count，但是目前我们先忽略它。我们可以想象`std::shared_ptr`对象在内存中是这样：

![](../x.public/item19_1.png)

当`std::shared_ptr`对象一创建对象控制块就建立了。至少我们期望是如此。通常，对于一个创建指向对象的`std::shared_ptr`的函数来说不可能知晓是否有其他`std::shared_ptr`早已指向那个对象，所以控制块的创建会遵循下面几条规则：

+ **std::make_shared总是创建一个控制块**(参见Item21)。它创建一个指向新对象的指针，所以可以肯定此时没有其他`std::make_shared`的控制块被调用。
+ **当从独占指针上构造出`std::shared_ptr`时会创建控制块（即std::unique_ptr或者std::auto_ptr）**。独占指针没有使用控制块，所以指针指向的对象没有关联其他控制块。（作为构造的一部分，`std::shared_ptr`侵占独占指针指向对象的独占权，所以`std::unique_ptr`被设置为null）
+ **当从原始指针上构造出`std::shared_ptr`时会创建控制块**。如果你想从一个早已存在控制块的对象上创建`std::shared_ptr`，你将假定传递一个`std::shared_ptr`或者`std::weak_ptr`作为构造函数实参，而不是原始指针。用`std::shared_ptr`或者`std::weak_ptr`作为构造函数实参创建`std::shared_ptr`不会创建新控制块，因为它可以依赖传递来的智能指针的控制块。
