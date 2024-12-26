# 不可实例化类型——接口

GType的接口类似于Java的接口，允许描述多个类实现共同的接口API。

要声明接口，需要注册一个`GTypeInterface`类型的不可实例化类型。
```c
#define VIEWER_TYPE_EDITABLE viewer_editable_get_type ()
G_DECLARE_INTERFACE (ViewerEditable, viewer_editable, VIEWER, EDITABLE, GObject)

struct _ViewerEditableInterface 
{
    GTypeInterface parent;

    void (*save) (ViewerEditable* self, GError** error);
};

void viewer_editable_save (ViewerEditable* self, GError** error);
```
接口函数`viewer_editable_save`实现如下：
```c
void viewer_editable_save (ViewerEditable* self, GError** error)
{
    ViewerEditableinterface *iface;

    g_return_if_fail (VIEWER_IS_EDITABLE (self));
    g_return_if_fail (error == NULL || *error == NULL);

    iface = VIEWER_EDITABLE_GET_IFACE (self);
    g_return_if_fail (iface->save != NULL);
    iface->save (self);
}
```
`viewer_editable_get_type`注册了一个称为`ViewerEditable`的类型，此类型继承自`G_TYPE_INTERFACE`。

> 注意：所有的接口必须是`G_TYPE_INTERFACE`的子类

接口定义时候，结构的第一个参数必须是`GTypeInterface`类型。接口结构体中包含了接口方法函数指针。为每个接口方法定义helper函数是一个不错的风格，这个helper仅仅是调用`viewer_editable_save`方法。

如果没有特殊需求，可以使用`G_IMPLEMENT_INTERFACE`宏实现接口：
```c
static void viewer_file_save (ViewerEditable *self)
{
    g_print ("File implementation of editable interface save method.\n");
}

static void viewer_file_editable_interface_init (ViewerEditableInterface *iface)
{
    iface->save = viewer_file_save;
}

G_DEFINE_TYPE_WITH_CODE (ViewerFile, viewer_file, VIEWER_TYPE_FILE, G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE, viewer_file_editable_interface_init))
```
如果有特殊需求，则需要编写自定义的`get_type`函数来实现GType的注册。例如，`ViewerFile`类实现了`ViewerEditable`接口：
```c
static void viewer_file_save (ViewerEditable *editable)
{
    g_print ("File implementation of editable interface save method.\n");
}

static void viewer_file_editable_interface_init (gpointer g_iface, gpointer iface_data)
{
    ViewerEditableInterface *iface = g_iface;

    iface->save = viewer_file_save;
}

GType viewer_file_get_type (void)
{
    static GType type = 0;

    if (type == 0) {
        const GTypeInfo info = {
            .class_size = sizeof (ViewerFileClass),
            .base_init = NULL,
            .base_finalize = NULL,
            .class_init = (GClassInitFunc) viewer_file_class_init,
            .class_finalize = NULL,
            .class_data = NULL,
            .instance_size = sizeof (ViewerFile),
            .n_preallocs = 0,
            .instance_init = (GInstanceInitFunc) viewer_file_init
        };

        const GInterfaceInfo editable_info = {
            .interface_init = (GInterfaceInitFunc) viewer_file_editable_interface_init,
            .interface_finalize = NULL,
            .interface_data = NULL,
        };

        type = g_type_register_static (VIEWER_TYPE_FILE, "ViewerFile", &info, 0);

        g_type_add_interface_static (type, VIEWER_TYPE_EDITABLE, &editable_info);
    }
    return type;
}
```
`g_type_add_interface_static()`将实现了`ViewerEditable`接口的`ViewerFile`类型注册到类型系统中。

`GInterfaceInfo`结构保存了实现的接口的信息
```c
struct _GInterfaceInfo
{
    GInterfaceInitFunc     interface_init;
    GInterfaceFinalizeFunc interface_finalize;
    gpointer               interface_data;
};
```

