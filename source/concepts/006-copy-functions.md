# 复制函数

所有GLib类型的主要共同点是它们都可以通过某个API来复制/分配类型所需内存。

GValue结构被用作所有这些类型的抽象容器。它的简化API（在gobject/gvalue.h中定义）可以用于调用类型注册期间注册的`value_table`函数：例如`g_value_copy()`复制GValue。它类似C++的复制运算符。

下面代码显示了如何复制64bit的整数以及GObject实例指针：
```c
static void test_int (void)
{
    GValue a_value = G_VALUE_INIT;
    GValue b_value = G_VALUE_INIT;
    guint64 a, b;

    a = 0xdeadbeef;

    g_value_init (&a_value, G_TYPE_UINT64);
    g_value_set_uint64 (&a_value, a);

    g_value_init (&b_value, G_TYPE_UINT64);
    g_value_copy (&a_value, &b_value);

    b = g_value_get_uint64 (&b_value);

    if (a == b) {
        g_print ("Yay !! 10 lines of code to copy around a uint64.\n");
    } else {
        g_print ("Are you sure this is not a Z80 ?\n");
    }
}

static void test_object (void)
{
    GObject *obj;
    GValue obj_vala = G_VALUE_INIT;
    GValue obj_valb = G_VALUE_INIT;
    obj = g_object_new (VIEWER_TYPE_FILE, NULL);

    g_value_init (&obj_vala, VIEWER_TYPE_FILE);
    g_value_set_object (&obj_vala, obj);

    g_value_init (&obj_valb, G_TYPE_OBJECT);

    /* g_value_copy's semantics for G_TYPE_OBJECT types is to copy the reference.
     * This function thus calls g_object_ref.
     * It is interesting to note that the assignment works here because
     * VIEWER_TYPE_FILE is a G_TYPE_OBJECT.
     */
    g_value_copy (&obj_vala, &obj_valb);

    g_object_unref (G_OBJECT (obj));
    g_object_unref (G_OBJECT (obj));
}
```
关于上述代码的一个重要点是，复制调用的确切语义未定义，因为它们依赖于复制函数的实现。某些复制函数可能会决定分配一个新的内存块，然后将数据从源复制到目标。其它人可能希望简单的增加引用计数并将引用复制到新的GValue

用于指定这些赋值函数的值表在`GTypeValueTable`中的记录。

需要注意的是，在类型注册时也不太可能需要指定`value_table`, 因为这些值从非基础类型的父类型中继承。


