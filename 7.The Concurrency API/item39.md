## Item 39: Consider void futures for one-shot event communication

## Item 39:对于一次性事件通信考虑使用无返回futures

有时，一个任务通知另一个异步执行的任务发生了特定的事件很有用，因为第二个任务要等到特定事件发生之后才能继续执行。事件也许是数据已经初始化，也许是计算阶段已经完成，或者检测到重要的传感器值。这种情况，什么是线程间通信的最佳方案？

一个明显的方案就是使用条件变量（condvar）。如果我们将检测条件的任务称为检测任务，对条件作出反应的任务称为反应任务，策略很简单：反应任务等待一个条件变量，检测任务在事件发生时改变条件变量。代码如下：

```cpp
std::condition_variable cv; // condvar for event
std::mutex m; // mutex for use with cv
```

检测任务中的代码不能再简单了：

```cpp
... // detect event
cv.notify_one(); // tell reacting task
```

如果有多个反应任务需要被通知，使用`notify_all()代替notify_one()`，但是这里，我们假定只有一个反应任务需要通知。

反应任务对的代码稍微复杂一点，因为在调用`wait`条件变量之前，必须通过`std::unique_lock`对象锁住`mutex`来同步（lock a mutex是等待条件变量的经典实现。`std::unique_lock`是C++11的易用API），代码如下：

```cpp
... // propare to react
{ // open critical section
  std::unique_lock<std::mutex> lk(m); // lock mutex
  cv.wait(lk); // wati for notify; this isn't correct!
  ... // react to event(m is blocked)
} // close crit. section; unlock m via lk's dtor
... // continue reacting (m now unblocked)
```

这份代码的第一个问题就是