# 对象属性

`GObject`的一个很好的特性是它对对象属性的通用`get/set`机制。当对象被实例化时，应该使用对象的`class_init`处理程序将对象的属性注册到`g_object_class_install_properties()`中。

理解对象属性如何工作的最好方法是看一个真实的例子:

```c
// Implementation

typedef enum
{
    PROP_FILENAME = 1,
    PROP_ZOOM_LEVEL,
    N_PROPERTIES
} ViewerFileProperty;

static GParamSpec *obj_properties[N_PROPERTIES] = { NULL, };

static void viewer_file_set_property (GObject* object, guint property_id, const GValue* value, GParamSpec* pspec)
{
    ViewerFile *self = VIEWER_FILE (object);

    switch ((ViewerFileProperty) property_id) {
        case PROP_FILENAME:
        g_free (self->filename);
        self->filename = g_value_dup_string (value);
        g_print ("filename: %s\n", self->filename);
        break;

        case PROP_ZOOM_LEVEL:
        self->zoom_level = g_value_get_uint (value);
        g_print ("zoom level: %u\n", self->zoom_level);
        break;

        default:
        /* We don't have any other property... */
        G_OBJECT_WARN_INVALID_PROPERTY_ID (object, property_id, pspec);
        break;
    }
}

static void viewer_file_get_property (GObject* object, guint property_id, GValue* value, GParamSpec* pspec)
{
    ViewerFile *self = VIEWER_FILE (object);

    switch ((ViewerFileProperty) property_id) {
        case PROP_FILENAME:
            g_value_set_string (value, self->filename);
            break;

        case PROP_ZOOM_LEVEL:
            g_value_set_uint (value, self->zoom_level);
            break;

        default:
            /* We don't have any other property... */
            G_OBJECT_WARN_INVALID_PROPERTY_ID (object, property_id, pspec);
        break;
    }
}

static void viewer_file_class_init (ViewerFileClass *klass)
{
    GObjectClass *object_class = G_OBJECT_CLASS (klass);

    object_class->set_property = viewer_file_set_property;
    object_class->get_property = viewer_file_get_property;

    obj_properties[PROP_FILENAME] = g_param_spec_string ("filename",
                                        "Filename",
                                        "Name of the file to load and display from.",
                                        NULL  /* default value */,
                                        G_PARAM_CONSTRUCT_ONLY | G_PARAM_READWRITE);

    obj_properties[PROP_ZOOM_LEVEL] = g_param_spec_uint ("zoom-level",
                       "Zoom level",
                       "Zoom level to view the file at.",
                       0  /* minimum value */,
                       10 /* maximum value */,
                       2  /* default value */,
                       G_PARAM_READWRITE);

    g_object_class_install_properties (object_class, N_PROPERTIES, obj_properties);
}
```

```c
// Use

ViewerFile *file;
GValue val = G_VALUE_INIT;

file = g_object_new (VIEWER_TYPE_FILE, NULL);

g_value_init (&val, G_TYPE_UINT);
g_value_set_char (&val, 11);

g_object_set_property (G_OBJECT (file), "zoom-level", &val);

g_value_unset (&val);
```

上面的客户端代码看起来很简单，但在底层发生了很多事情：

`g_object_set_property()`首先确保在文件的`class_init`处理程序中注册了具有此名称的属性。如果是这样，它将遍历类层次结构，从最底部的派生类型到最顶部的基本类型，以找到注册该属性的类。然后，它尝试将用户提供的GValue转换为与关联属性类型相同的GValue。

如果用户提供了一个有符号char GValue，如下所示，并且如果对象的属性被注册为`unsigned int`，`g_value_transform()`将尝试将输入的有符号char转换为`unsigned int`。当然，转换的成功取决于所需转换函数的可用性。在实践中，几乎总是会有一个匹配的转换，如果需要的话，会进行转换。

转换后，GValue由`g_param_value_validate()`进行验证，以确保存储在GValue中的用户数据与属性的GParamSpec指定的特征匹配。这里，我们在`class_init`中提供的`GParamSpec`有一个验证函数，它确保GValue包含一个符合GParamSpec最小和最大边界的值。在上面的例子中，客户端的GValue不尊重这些约束（它被设置为11，而最大值是10）。因此，`g_object_set_property()`函数将返回一个错误。

如果用户的GValue被设置为一个有效值，`g_object_set_property()`将继续调用对象的`set_property`类方法。在这里，由于ViewerFile的实现确实覆盖了这个方法，在从GParamSpec中检索到`g_object_class_install_property()`存储的param_id之后，执行将跳转到`viewer_file_set_property`。

一旦对象的`set_property`类方法设置了属性，执行返回到`g_object_set_property()`，它确保在对象的实例上发出“notify”信号，并将更改的属性作为参数，除非通知被`g_object_freeze_notify()`冻结。

`g_object_thaw_notify()`可用于通过“notify”信号重新启用属性修改通知。重要的是要记住，即使在冻结属性更改通知时更改了属性，一旦属性更改通知解冻，“notify”信号将为每个更改的属性发出一次：“notify”信号不会丢失属性更改，尽管单个属性的多个通知被压缩了。信号只能通过通知冻结机制延迟。

每次想要修改属性时都要设置GValues，这听起来像是一项乏味的任务。实际上很少有人会这样做。函数`g_object_set_property()`和`g_object_get_property()`是由语言绑定使用的。对于应用程序，有一种更简单的方法，下面将介绍。

## 一次访问多个属性

有趣的是，`g_object_set()`和`g_object_set_valist()`（可变版本）函数可用于一次设置多个属性。上面显示的客户端代码可以重写为:

```c
ViewerFile *file;
file = /* */;
g_object_set (G_OBJECT (file), "zoom-level", 6, "filename", "~/some-file.txt", NULL);
```

这使我们免于管理使用`g_object_set_property()`时需要处理的GValues。上面的代码将为修改的每个属性触发一个通知信号发射。

等价的`_get`版本也可用：`g_object_get()`和`g_object_get_valist()`（可变版本）可用于一次获取多个属性。

这些高级函数有一个缺点——它们不提供返回值。在使用它们时，应该注意参数的类型和范围。已知的错误来源是传递与属性期望不同的类型；例如，当属性期望一个浮点值时传递一个整数，从而将所有后续参数移动一定数量的字节。另外，忘记终止NULL将导致未定义的行为。

这解释了`g_object_new()`、`g_object_newv()`和`g_object_new_valist()`是如何工作的：它们解析用户提供的可变数量的参数，并仅在对象成功构造之后对参数调用`g_object_set()`。将为每个属性集发出“notify”信号。


