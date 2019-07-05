## Item 20:像std::shared_ptr一样使用std::weak_ptr可能造成dangle


自相矛盾的是，如果有一个像`std::shared_ptr`的指针但是不参与资源所有权共享的指针是很方便的。换句话说，类似`std::shared_ptr`的指针但是不影响对象的引用计数。这种类型的智能指针