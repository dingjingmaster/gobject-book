# 接口初始化

当实现接口的可实例化类类型第一次被创建时，它的类结构将按照“可实例化类类型：对象”一节中描述的过程初始化。之后，与该类型关联的接口实现被初始化。

首先分配一个内存缓存区来保存接口结构。然后将父接口结构复制到新的接口结构中（此时父接口已经初始化）。如果没有父接口，则用零初始化接口结构。然后初始化`g_type`和`g_instance_type`字段。

调用接口的`base_init`函数，然后调用接口的`default_init`函数。最后，如果类型注册了接口的实现，则调用实现的`interface_init`函数。如果一个接口有多个实现，则`base_init`和`initerface_init`函数将为每个初始化的实现的调用一次。

因此，建议使用`default_init`函数初始化接口。无论有多少实现，这个函数只对接口调用一次。`default_init`函数由`G_DEFINE_INTERFACE`声明，可以用来定义接口：
```c
G_DEFINE_INTERFACE (ViewerEditable, viewer_editable, G_TYPE_OBJECT)
static void viewer_editable_default_init (ViewerEditableInterface *iface)
{
    /* add properties and signals here, will only be called once */
}
```
或者手动实现`get_type`函数：
```c
GType viewer_editable_get_type (void)
{
    static gsize type_id = 0;

    if (g_once_init_enter (&type_id)) {
        const GTypeInfo info = {
            sizeof (ViewerEditableInterface),
            NULL,   /* base_init */
            NULL,   /* base_finalize */
            viewer_editable_default_init, /* class_init */
            NULL,   /* class_finalize */
            NULL,   /* class_data */
            0,      /* instance_size */
            0,      /* n_preallocs */
            NULL    /* instance_init */
        };

        GType type = g_type_register_static (G_TYPE_INTERFACE, "ViewerEditable", &info, 0);
        g_once_init_leave (&type_id, type);
    }
    return type_id;
}

static void viewer_editable_default_init (ViewerEditableInterface *iface)
{
    /* add properties and signals here, will only called once */
}
```

综上所述，接口初始化使用如下函数：

- `base_init`函数：很少需要实现此方法，实现接口的类的实现化时候调用一次
- `default_init`函数：在这里注册接口的信号、属性等，只调用一次
- `interface_init`函数：初始化接口实现。为实现接口的每个类调用。将接口结构中的接口方法指针初始化为实现类的实现



