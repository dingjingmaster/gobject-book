# 初始化与销毁

类型实例化时可以用`g_type_create_instance()`，它会查找与请求类型相关联的结构类型。然后由用户声明的实例大小和实例化策略（如果`n_preallocs`字段设置为非0值，类型系统会分配连续的内存块保存类的实例化，而不是每个类型都实例化一次)来申请分配缓存区，用来存放对象的实例。

如果这是对象的第一个实例创建，类型系统必须创建一个类结构体。它初始化一个缓存区来保存对象的类结构并初始化它。类结构的第一部分通过从父类的类结构复制内容来初始化。其余的类结构被初始化为0。如果没有父类，则整个类结构初始化为0。然后类型系统依次调用此类父对象的`base_init`函数，然后在调用此类的`base_init`函数，接下来调用对象的`class_init`函数，完成类结构的初始化。

一旦类型系统有一个指向初始化类结构的指针，它就会将对象的实例类指针设置为对象的类结构，并依次调用此类父对象及此对象的`instance_init`函数。

通过`g_type_free_instance()`销毁对象过程非常简单：如果至少还有一个实例对象，则此类的实例结构返回到累得实例化池中，如果这是该对象的最后一个可用实例，则会销毁此类。

类销毁是类初始化的对称操作。然后从子类开始依次调用`class_finalize`函数。最后，从子类开始依次调用`base_finalize`函数。

初始化/销毁过程非常类似C++构造/析构函数程式。但是，实现细节是不同的。GType没有实例销毁机制，需要使用者在GType机制上实现销毁函数。

初始化/销毁过程总结如下：
## 1. 调用`g_type_create_instance()`

1. 调用类型的`base_init`函数，从最基本父类到此类`base_init`为每个类结构调用一次。
2. 调用类型的`class_init`函数，参数是类对象结构

## 2. 调用`g_type_free_instance()`

1. 类型的`class_finalize`，参数是类对象结构
2. 类型的`base_finalize`，从基类开始到此类，每个类结构上调用一次


