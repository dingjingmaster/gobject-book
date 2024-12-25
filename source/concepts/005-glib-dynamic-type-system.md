# GLib动态类型系统

由GLib类型系统操作的类型比通常了解的对象类型系统通用的多。最好通过查看类型系统中用于注册新类型的结构和函数来解释。

```c
typedef struct _GTypeInfo GTypeInfo;
struct _GTypeInfo
{
    /* interface types, classed types, instantiated types */
    guint16                class_size;

    GBaseInitFunc          base_init;
    GBaseFinalizeFunc      base_finalize;

    /* classed types, instantiated types */
    GClassInitFunc         class_init;
    GClassFinalizeFunc     class_finalize;
    gconstpointer          class_data;

    /* instantiated types */
    guint16                instance_size;
    guint16                n_preallocs;
    GInstanceInitFunc      instance_init;

    /* value handling */
    const GTypeValueTable *value_table;
};

GType g_type_register_static (GType parent_type, const gchar* type_name, const GTypeInfo* info, GTypeFlags flags);
GType g_type_register_fundamental (GType type_id, const gchar* type_name, const GTypeInfo* info, const GTypeFundamentalInfo *finfo, GTypeFlags flags);
```

`g_type_register_static()`, `g_type_register_dynamic()` 和 `g_type_register_fundamental()`是C函数，在`gtype.h`中定义，并在`gtype.c`中实现，您应该使用它来在程序的类型系统中注册一个新的`GType`。您一般不会使用`g_type_register_fundamental()`，但是如果缺失需要，最后一章描述其如何使用它创建新的基础类型。

基本类型是最底层的类型，它不是从其它类型派生而来的。在初始化时，类型系统不仅初始化其内部数据结构，而且它还注册了一些核心类型：其中一些是基本类型，另一些是从这些基本类型派生而来的。

基本类型和非基本类型的定义：
- 类大小：在`GTypeInfo`中的`class_size`字段中定义
- 类初始化函数(C++构造函数)：在`GTypeInfo`中的`base_init`和`class_init`字段
- 类销毁函数(C++析构函数)：在`GTypeInfo`中的`base_finalize`和`class_finalize`字段
- 实例大小(C++的new参数)：在`GTypeInfo`中的`instance_size`字段
- 实例化策略(C++ new操作类型)：在`GTypeInfo`中的`n_preallocs`字段
- 复制函数(C++ 复制操作)：在`GTypeInfo`中的`value_table`字段
- 类型标志：`GTypeFlags`

基本类型也被存在`GTypeFundamentalInfo`中的`GTypeFundamentalFlags`定义。非基本类型由其父类型的类型定义，该类型作为父类型参数传递给`g_type_register_static()`和`g_type_register_dynamic()`实现。


