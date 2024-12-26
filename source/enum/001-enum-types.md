# 枚举类型

GLib类型系统为枚举和标志类型提供了基本类型。枚举类型是与数值相关联的标识符的集合；标志类型类似于枚举，但允许它们的值按位或按位组合。

注册的枚举或标志类型将名称和昵称与每个允许的值关联起来，并且`g_enum_get_value_by_name()`、`g_enum_get_value_by_nick()、`g_flags_get_value_by_name()`和`g_flags_get_value_by_nick()`方法可以通过名称或昵称查找值。

当枚举或标志类型注册到GLib类型系统时，可以使用`g_param_spec_enum()`或`g_param_spec_flags()`将其用作对象属性的值类型。

object附带了一个名为glib-mkenums的实用程序，它可以从C枚举定义构造合适的类型注册函数。

如何获取枚举值的字符串表示形式的示例：
```c
GEnumClass *enum_class;
GEnumValue *enum_value;

enum_class = g_type_class_ref (EXAMPLE_TYPE_ENUM);
enum_value = g_enum_get_value (enum_class, EXAMPLE_ENUM_FOO);

g_print ("Name: %s\n", enum_value->value_name);

g_type_class_unref (enum_class);
```


