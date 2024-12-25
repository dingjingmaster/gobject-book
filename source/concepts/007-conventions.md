# 约定

在创建头文件中需要导出的新类型时候，用于应该遵循一些约定：
- 类型名（包括对象名）必须至少为3个字符长度
- 函数名使用`object_method`：比如使用file对象的save方法，则调用`file_save`
- 使用命名空间来与其他方法做区分，`namespace_object_method`类型
- 创建名为`PREFIX_TYPE_OBJECT`的宏，调用`prefix_object_get_type`函数获取GType类型
- 使用`G_DECLARE_FINAL_TYPE`或`G_DECLARE_DERIVALE_TYPE`为对象定义其他各种常量宏
- `PREFIX_OBJECT(obj)`，它返回一个指向`PrefixObject`类型的指针。此宏用来做安全类型转换
- `PREFIX_OBJECT_CLASS(klass)`，它返回一个指向`PrefixObjectClass`类型的指针。此空用来做安全类型转换
- `PREFIX_IS_OBJECT(obj)`
- `PREFIX_IS_OBJECT_CLASS(klass)`
- `PREFIX_OBJECT_GET_CLASS(obj)`

这些宏的实现非常简单：在gtype.h中提供了一些易于使用的宏。

```c
#define VIEWER_TYPE_FILE viewer_file_get_type()

G_DECLARE_FINAL_TYPE (ViewerFile, viewer_file, VIEWER, FILE, GObject)
```

除非你的代码由特殊要求，否则可以使用`G_DEFINE_TYPE`宏来定义类：
```c
G_DEFINE_TYPE (ViewerFile, viewer_file, G_TYPE_OBJECT)
```

否则 `viewer_file_get_type` 函数必须手动实现：
```c
GType viewer_file_get_type (void)
{
    static GType type = 0;
    if (type == 0) {
        const GTypeInfo info = {
            /* You fill this structure. */
        };
        type = g_type_register_static (G_TYPE_OBJECT, "ViewerFile", &info, 0);
    }
    return type;
}
```


