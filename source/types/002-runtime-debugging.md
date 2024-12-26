# 运行时调试

当在编译期间定义`G_ENABLE_DEBUG`时，GObject库支持一个环境变量`GOBJECT_DEBUG`，可以将其设置为一个标志组合，以在运行时触发关于对象簿记和信号释放的调试消息。

目前支持的标志是：
- `objects`：在名为`debug_objects_ht`的全局哈希表中跟踪所有GObject实例，并在退出时打印仍然存活的对象。
- `instance-count`：跟踪每个GType的实例数量，并通过`g_type_get_instance_count()`函数使其可用。
- `signals`：当前未使用
