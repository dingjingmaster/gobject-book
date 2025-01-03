# 对象内存管理

对象的内存管理API有点复杂，但是其背后的思想非常简单：提供一个基于引用技术的灵活模型，该模型可以集成到使用或需要不同内存管理模型（如：垃圾回收）的应用程序中。下面描述了用于操作此引用技术的方法。

## 引用计数

函数`g_object_ref()`和`g_object_unref()`分别增加和减少引用计数。这些函数是线程安全的。`g_clear_object()`是`g_object_unref()`的方便包装，它也清除传递给它的指针。

引用计数由`g_object_new()`初始化为1，这意味着调用者当前是新创建的引用的唯一所有者。（如果对象是从`GInitiallyUnowned`派生的，这个引用是“浮动的”，必须被“沉没”，即转换为一个真正的引用。）当引用计数达到零时，即当`g_object_unref()`被对象引用的最后一个所有者调用时，将调用`dispose()`和`finalize()`类方法。

最后，在调用`finalize()`之后，调用`g_type_free_instance()`来释放对象实例。根据注册类型时决定的内存分配策略（通过g_type_register_*函数之一），对象的实例内存将被释放或返回到该类型的对象池中。一旦对象被释放，如果它是该类型的最后一个实例，则该类型的类将被销毁，如“可实例化的类类型：对象”和“不可实例化的类类型：接口”一节所述。

以下总结了一个GObject对象销毁的过程：
- `dispose`函数：
- `finalize`函数：
- `interface_finalize`函数：
- `base_finalize`函数：
- `class_finalize`函数：
- `base_finalize`函数：

## 弱引用

弱引用用于监视对象的终结：`g_object_weak_ref()`添加了一个监视回调，该回调不保存对对象的引用，但在对象运行其dispose方法时调用。对象上的弱引用在处置实例时自动删除，因此不需要从`GWeakNotify`回调调用`g_object_weak_unref()`。记住，对象实例不会传递给`GWeakNotify`回调，因为对象已经被`disposed`了。相反，回调函数接收一个指向对象先前所在位置的指针。

弱引用也用于实现`g_object_add_weak_pointer()`和`g_object_remove_weak_pointer()`。这些函数为它们所应用的对象添加了一个弱引用，以确保在对象结束时用户给出的指针无效。

类似地，如果需要线程安全，可以使用`GWeakRef`来实现弱引用。

## 引用计数与循环引用

`GObject`的内存管理模型被设计成可以很容易地集成到使用垃圾收集的现有代码中。这就是销毁过程分为两个阶段的原因：在`dispose()`处理程序中执行的第一阶段应该释放对其他成员对象的所有引用。由`finalize()`处理程序执行的第二阶段应该完成对象的销毁过程。对象方法应该能够在两个阶段之间运行而不会出现程序错误。

这个两步销毁过程对于打破引用计数循环非常有用。虽然循环的检测取决于外部代码，但是一旦检测到循环，外部代码就可以调用`g_object_run_dispose()`，这确实会中断任何现有的循环，因为它将运行与对象关联的dispose处理程序，从而释放对其他对象的所有引用。

这解释了前面提到的关于`dispose()`处理程序的一条规则：`dispose()`处理程序可以被多次调用。假设我们有一个引用计数循环：对象a引用对象B，而对象B本身又引用对象a。假设我们已经检测到这个循环，我们想要销毁这两个对象。一种方法是在其中一个对象上调用`g_object_run_dispose()`。

如果对象A释放了它对所有对象的所有引用，这意味着它释放了它对对象B的引用。如果对象B不属于任何人，这是它的最后一个引用计数，这意味着最后一个unref运行B的`dispose`处理程序，反过来，释放B对对象A的引用。如果这是A的最后一个引用计数，最后一个unref运行A的dispose处理程序，该处理程序在A的finalize处理程序被调用之前第二次运行！

上面的例子看起来有点做作，但如果对象是由语言绑定处理的，就会发生这种情况——因此应该严格遵守对象销毁的规则。
