# 可实例化类类型：objects

虽然`GObject`是最基础的可实例化类类型，但是其他可实例化类型也可以在不继承`GObject`时候实现，它们都是基于下面描述的基本特征构建的。

例如，下面的代码在不通过`GObject`提供的便利API前提下注册类型

```c
typedef struct 
{
    GObject parent_instance;

    /* instance members */
    char *filename;
} ViewerFile;

typedef struct 
{
    GObjectClass parent_class;

    /* class members */

    /* the first is public, pure and virtual */
    void (*open)  (ViewerFile* self, GError** error);

    /* the second is public and virtual */
    void (*close) (ViewerFile* self, GError** error);
} ViewerFileClass;

#define VIEWER_TYPE_FILE (viewer_file_get_type ())

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
            .instance_init = (GInstanceInitFunc) viewer_file_init,
        };
        type = g_type_register_static (G_TYPE_OBJECT, "ViewerFile", &info, 0);
    }
    return type;
}
```
在第一次调用 `viewer_file_get_type`时，名为`ViewerFile`的类型将注册为从`G_TYPE_OBJECT`类继承而来。

每个对象必须定义两个结构体：类结构体和实例结构体。所有类结构体第一个元素必须包含`GTypeClass`结构。所有实例结构体第一个元素必须是包含`GTypeInstance`结构。

```c
struct _GTypeClass
{
    GType g_type;
};

struct _GTypeInstance
{
    GTypeClass *g_class;
};
```

这些约束确保每个对象实例在其第一个字节中包含了指向该对象的类结构的一个指针。

假设对象B继承对象A
```c
/* A definitions */
typedef struct 
{
    GTypeInstance parent;
    int field_a;
    int field_b;
} A;

typedef struct 
{
    GTypeClass parent_class;
    void (*method_a) (void);
    void (*method_b) (void);
} AClass;

/* B definitions. */
typedef struct 
{
    A parent;
    int field_c;
    int field_d;
} B;

typedef struct 
{
    AClass parent_class;
    void (*method_c) (void);
    void (*method_d) (void);
} BClass;
```

C标准要求结构体的首地址与结构体第一个字段地址一致。这就意味着对象B的第一个字段是对象A的实例，也是`GTypeInstance`

由于这个简单的条件，可以通过执行以下操作来检测每个对象实例的类型：
```c
B *b;
b->parent.parent.g_class->g_type
```
或者：
```c
B *b;
((GTypeInstance *) b)->g_class->g_type
```


