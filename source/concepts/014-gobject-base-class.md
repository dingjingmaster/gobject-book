# GObject基类

之前内容介绍了GLib的动态类型系统。GObject库同样实现了称为GObject的基础类型。

GObject是一个基本的实例化类类型，它实现了：
- 据有引用计数的内存管理
- 类实例的构造和析构
- 具有set/get函数对的属性操作实现
- 易于使用的信号

所有的GNOME库上使用了GLib类型系统（如：GTK、GStreamer）都继承了GObject。
