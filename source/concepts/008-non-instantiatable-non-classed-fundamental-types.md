# 不可实例化类型

许多类型不能通过类型系统实例化，并且没有类。这些类型中的大部分是基本类型，例如：gchar并已由GLib默认注册。

如果需要注册这类型类型，可以用零填充`GTypeInfo`：

```c
GTypeInfo info = {
    .class_size = 0,

    .base_init = NULL,
    .base_finalize = NULL,

    .class_init = NULL,
    .class_finalize = NULL,
    .class_data = NULL,

    .instance_size = 0,
    .n_preallocs = 0,
    .instance_init = NULL,

    .value_table = NULL,
};

static const GTypeValueTable value_table = {
    .value_init = value_init_long0,
    .value_free = NULL,
    .value_copy = value_copy_long0,
    .value_peek_pointer = NULL,

    .collect_format = "i",
    .collect_value = value_collect_int,
    .lcopy_format = "p",
    .lcopy_value = value_lcopy_char,
};

info.value_table = &value_table;

type = g_type_register_fundamental (G_TYPE_CHAR, "gchar", &info, &finfo, 0);
```


