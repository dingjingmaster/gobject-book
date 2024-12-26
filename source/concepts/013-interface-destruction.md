# 接口销毁

当注册接口实现的可实例化类型的最后一个实例被销毁时，与该类型关联的接口实现也将被销毁。

要销毁接口实现，GType首先调用实现的interface_finalize函数，然后调用接口最派生的base_finalize函数。

同样，理解这一点很重要，正如在“接口初始化”一节中所述，对于接口的每个实现的销毁，interface_finalize和base_finalize都只调用一次。因此，如果要使用这些函数中的一个，则需要使用静态整数变量来保存接口实现的实例数量，以便接口的类只被销毁一次（当整数变量达到零时）。

上述过程归纳如下：
- `interface_finalize`函数，输入参数：接口的vtable
- `base_finalize`函数，输入参数：接口的vtable


