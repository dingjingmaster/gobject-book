# Boxed Types

“boxed type”是任意C结构的通用包装机制。类型系统唯一需要知道的是如何复制它们（GBoxedCopyFunc）和如何释放它们（GBoxedFreeFunc）——除此之外，它们被视为不透明的内存块。

Boxed Types 对于简单的值保存结构非常有用。它们还可以用于包装在非基于对象的库中定义的结构。它们允许以统一的方式处理任意结构，允许对它们进行统一的复制（或引用）和释放（或取消引用），并统一表示所包含结构的类型。反过来，这允许任何可以被Boxed Types设置为GValue中的数据，这允许对更广泛的数据类型进行多态处理，因此可以使用GObject属性值等类型。

所有Boxed Types都继承G_TYPE_BOXED基本类型。

需要注意的是，Boxed Types不是深度可继承的：您不能注册从另一个Boxed Types继承的Boxted Type。这意味着您不能创建自己的自定义并行类型层次结构，并使用Boxted Types将其映射到GType。如果您希望拥有深度可继承的类型而不使用GObject，则需要使用GTypeInstance。

## 注册新的Boxed Types

注册一个新的Boxed Types的推荐方法是使用`G_DEFINE_BOXED_TYPE()`宏：

```c
// In the header

#define EXAMPLE_TYPE_RECTANGLE (example_rectangle_get_type())

typedef struct {
  double x, y;
  double width, height;
} ExampleRectangle;

GType
example_rectangle_get_type (void);

ExampleRectangle *
example_rectangle_copy (ExampleRectangle *r);

void
example_rectangle_free (ExampleRectangle *r);

// In the source
G_DEFINE_BOXED_TYPE (ExampleRectangle, example_rectangle,
                     example_rectangle_copy,
                     example_rectangle_free)
```

就像`G_DEFINE_TYPE`和`G_DEFINE_INTERFACE_TYPE`一样，`G_DEFINE_BOXED_TYPE`宏将提供`get_type()`函数的定义，该函数将使用给定的类型名称调用`g_boxed_type_register_static()`以及`GBoxedCopyFunc`和`GBoxedFreeFunc`函数。

## 使用Boxed Types

### 属性

为了在GObject属性中使用Boxed Types，你需要使用`g_param_spec_boxed()`来注册属性
```c
obj_props[PROP_BOUNDS] =
  g_param_spec_boxed ("bounds", NULL, NULL,
                      EXAMPLE_TYPE_RECTANGLE,
                      G_PARAM_READWRITE | G_PARAM_STATIC_STRINGS);
```

在set_property实现中，你可以使用g_value_get_boxed（）来检索一个指向Boxed Types的指针：
```c
switch (prop_id)
  {
  // ...
  case PROP_BOUNDS:
    example_object_set_bounds (self, g_value_get_boxed (value));
    break;
  // ...
  }
```

类似地，你可以在get_property虚函数的实现中使用g_value_set_boxed()：
```c
switch (prop_id)
  {
  // ...
  case PROP_BOUNDS:
    g_value_set_boxed (self, &self->bounds);
    break;
  // ...
  }
```

## 引用计数

使用类型的‘ref’函数作为GBoxedCopyFunc，它的‘unref’函数作为GBoxedFreeFunc。例如，对于GBytes，GBoxedCopyFunc是g_bytes_ref()， GBoxedFreeFunc是g_bytes_unref()。

