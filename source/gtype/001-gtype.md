# GObject 相关数据结构

## GType定义

```c
// gobject/gtype.h
typedef gulong          GType;
```

## GTypeClass定义

所有类的基类
```c
// typedef struct _GTypeClass GTypeClass
struct _GTypeClass
{
    GType g_type;
};
```

## GTypeInstance

所有Instance的基础类
```c
struct _GTypeInstance
{
    GTypeClass* g_class;
};
```

## GTypeInterface

所有接口类的基类
```c
struct _GTypeInterface
{
    GType g_type;
    GType g_instance_type;
};
```

## GTypeQuert

保存类类型的基础信息
```c
struct _GTypeQuery
{
    GType type;
    const gchar* type_name;
    guint class_size;
    guint instance_size;
};
```
