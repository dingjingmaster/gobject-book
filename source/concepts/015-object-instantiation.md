# 对象实例化

`g_object_new()`函数可用于实例化从GObject基类型继承的任何GType。所有这些函数确保类和实例结构已被GLib的类型系统正确初始化，然后在某个点或另一个点调用构造函数类方法，该方法用于：
- 通过`g_type_create_instance()`分配和清除内存
- 用构造属性初始化对象的实例

GObject显式的保证所有类和实例成员（除了指向父类的字段）都被设置为零。

完成所有构造操作并设置构造函数属性后，将调用构造的类方法。

从GObject继承的对象可以重写这个构造的类方法。下面的例子展示了ViewFile如何覆盖父类的构造过程：
```c
#define VIEWER_TYPE_FILE viewer_file_get_type ()
G_DECLARE_FINAL_TYPE (ViewerFile, viewer_file, VIEWER, FILE, GObject)

struct _ViewerFile
{
    GObject parent_instance;

    /* instance members */
    char *filename;
    guint zoom_level;
};

/* will create viewer_file_get_type and set viewer_file_parent_class */
G_DEFINE_TYPE (ViewerFile, viewer_file, G_TYPE_OBJECT)

static void viewer_file_constructed (GObject *obj)
{
    /* update the object state depending on constructor properties */

    /* Always chain up to the parent constructed function to complete object
     * initialisation. */
    G_OBJECT_CLASS (viewer_file_parent_class)->constructed (obj);
}

static void viewer_file_finalize (GObject *obj)
{
    ViewerFile *self = VIEWER_FILE (obj);

    g_free (self->filename);

    /* Always chain up to the parent finalize function to complete object
     * destruction. */
    G_OBJECT_CLASS (viewer_file_parent_class)->finalize (obj);
}

static void viewer_file_class_init (ViewerFileClass *klass)
{
    GObjectClass *object_class = G_OBJECT_CLASS (klass);

    object_class->constructed = viewer_file_constructed;
    object_class->finalize = viewer_file_finalize;
}

static void viewer_file_init (ViewerFile *self)
{
    /* initialize the object */
}
```
如果用户实例化一个`ViewerFile`对象：
```c
ViewerFile *file = g_object_new (VIEWER_TYPE_FILE, NULL);
```

如果这是此类对象的第一次实例化，则viewer_file_class_init函数将在viewer_file_base_class_init函数之后调用。这将确保这个新对象的类结构被正确初始化。在这里，viewer_file_class_init应该覆盖对象的类方法并设置类自己的方法。在上面的例子中，构造方法是唯一被覆盖的方法：它被设置为viewer_file_construct。

一旦`g_object_new()`获得了对初始化的类结构的引用，如果构造函数在viewer_file_class_init中被重写，则调用其构造函数方法来创建新对象的实例。被重写的构造函数必须链接到它们的父构造函数。为了找到父类并链接到父类构造函数，我们可以使用G_DEFINE_TYPE宏为我们设置的viewer_file_parent_class指针。

最后，在某个时刻，g_object_constructor由链中的最后一个构造函数调用。该函数通过`g_type_create_instance()`分配对象的实例缓冲区，这意味着如果注册了instance_init函数，则此时调用该函数。在instance_init返回后，对象被完全初始化，并且应该准备好让用户调用它的方法。当`g_type_create_instance()`返回时，g_object_constructor设置构造属性（即给定给`g_object_new()`的属性）并返回给用户的构造函数。

上面描述的过程可能看起来有点复杂，但可以通过下面的表轻松地总结出来，该表列出了`g_object_new()`调用的函数及其调用顺序：
- `base_init`函数：
- `class_init`函数：
- `base_init`函数：
- `interface_init`函数：
- `GObjectClass->constructor`函数：
- `instance_init`函数
- `GObjectClass->constructed`函数：

读者应该注意函数调用顺序中的一个小变化：虽然，从技术上讲，类的构造函数方法在GType的instance_init函数之前被调用（因为调用instance_init的`g_type_create_instance()`是由`g_object_constructor`调用的，g_object_constructor是顶级的类构造函数方法，并且期望用户链接到），在用户提供的构造函数中运行的用户代码将始终在GType的instance_init函数之后运行，因为用户提供的构造函数必须（你已经被警告过）在做任何有用的事情之前链接起来。


