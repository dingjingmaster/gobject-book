# 定义和实现接口

## 定义接口

GObject接口工作原理在“非实例化的类类型：接口”一节中给出；本节介绍如何定义和实现接口。

第一步是让头部正确。这个接口定义了三个方法：
```c
/*
 * Copyright/Licensing information.
 */

#pragma once

#include <glib-object.h>

G_BEGIN_DECLS

#define VIEWER_TYPE_EDITABLE viewer_editable_get_type()
G_DECLARE_INTERFACE (ViewerEditable, viewer_editable, VIEWER, EDITABLE, GObject)

struct _ViewerEditableInterface
{
  GTypeInterface parent_iface;

  void (*save) (ViewerEditable  *self,
                GError         **error);
  void (*undo) (ViewerEditable  *self,
                guint            n_steps);
  void (*redo) (ViewerEditable  *self,
                guint            n_steps);
};

void viewer_editable_save (ViewerEditable  *self,
                           GError         **error);
void viewer_editable_undo (ViewerEditable  *self,
                           guint            n_steps);
void viewer_editable_redo (ViewerEditable  *self,
                           guint            n_steps);

G_END_DECLS
```

这段代码与从GObject派生的普通GType的代码相同，除了一些细节：
- `_GET_CLASS`函数被称为`_GET_IFACE`（由`G_DECLARE_INTERFACE`定义）
- 实例类型`ViewerEditable`没有完全定义：它只是作为一个抽象类型使用，表示实现接口的任何对象的实例
- `ViewEditableInterface`的父类是`GTypeInterface`，而不是`GObjectClass`

ViewerEditable类型本身的实现是微不足道的：
- `G_DEFINE_INTERFACE`创建一个`viewer_editable_get_type`函数，该函数在类型系统中注册该类型。第三个参数用于定义先决条件接口（稍后将详细讨论）。当接口没有先决条件时，只需为该参数传递0
- 如果有接口的信号，`viewer_editable_default_init`应该注册（稍后我们将看到如何使用它们）。
- 接口方法`viewer_editable_save`，`viewer_editable_undo`和`viewer_editable_redo`解除接口结构的引用，以访问和调用其关联的接口函数

```c
G_DEFINE_INTERFACE (ViewerEditable, viewer_editable, G_TYPE_OBJECT)

static void
viewer_editable_default_init (ViewerEditableInterface *iface)
{
    /* add properties and signals to the interface here */
}

void
viewer_editable_save (ViewerEditable  *self,
                      GError         **error)
{
  ViewerEditableInterface *iface;

  g_return_if_fail (VIEWER_IS_EDITABLE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  iface = VIEWER_EDITABLE_GET_IFACE (self);
  g_return_if_fail (iface->save != NULL);
  iface->save (self, error);
}

void
viewer_editable_undo (ViewerEditable *self,
                      guint           n_steps)
{
  ViewerEditableInterface *iface;

  g_return_if_fail (VIEWER_IS_EDITABLE (self));

  iface = VIEWER_EDITABLE_GET_IFACE (self);
  g_return_if_fail (iface->undo != NULL);
  iface->undo (self, n_steps);
}

void
viewer_editable_redo (ViewerEditable *self,
                      guint           n_steps)
{
  ViewerEditableInterface *iface;

  g_return_if_fail (VIEWER_IS_EDITABLE (self));

  iface = VIEWER_EDITABLE_GET_IFACE (self);
  g_return_if_fail (iface->redo != NULL);
  iface->redo (self, n_steps);
}
```

## 实现接口

一旦定义了接口，实现它就相当简单了。
- 第一步是像往常一样定义一个普通的final GObject类。
- 第二步是通过使用`G_DEFINE_TYPE_WITH_CODE`和`G_IMPLEMENT_INTERFACE`来代替`G_DEFINE_TYPE`来实现`ViewerFile`：

```c
static void viewer_file_editable_interface_init (ViewerEditableInterface *iface);

G_DEFINE_TYPE_WITH_CODE (ViewerFile, viewer_file, G_TYPE_OBJECT,
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE,
                                                viewer_file_editable_interface_init))
```

这个定义非常像前面看到的所有类似的函数。这里出现的唯一特定于接口的代码是`G_IMPLEMENT_INTERFACE`的使用。

类可以通过在调用`G_DEFINE_TYPE_WITH_CODE`中使用对`G_IMPLEMENT_INTERFACE`的多次调用来实现多个接口。

`viewer_file_editable_interface_init`是接口初始化函数：在它内部，接口的每个虚方法都必须分配给它的实现：

```c
static void
viewer_file_editable_save (ViewerFile  *self,
                           GError     **error)
{
  g_print ("File implementation of editable interface save method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_undo (ViewerFile *self,
                           guint       n_steps)
{
  g_print ("File implementation of editable interface undo method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_redo (ViewerFile *self,
                           guint       n_steps)
{
  g_print ("File implementation of editable interface redo method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_interface_init (ViewerEditableInterface *iface)
{
  iface->save = viewer_file_editable_save;
  iface->undo = viewer_file_editable_undo;
  iface->redo = viewer_file_editable_redo;
}

static void
viewer_file_init (ViewerFile *self)
{
  /* Instance variable initialisation code. */
}
```

如果对象不是final类型，例如使用g_declare_derived _type声明，那么应该添加G_ADD_PRIVATE宏。私有结构的声明应该和普通的可派生对象完全一样。

```c
G_DEFINE_TYPE_WITH_CODE (ViewerFile, viewer_file, G_TYPE_OBJECT,
                         G_ADD_PRIVATE (ViewerFile)
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE,
                                                viewer_file_editable_interface_init))
```
## 接口定义前提条件

为了指定接口在实现时需要其他接口的存在，GObject引入了先决条件的概念：可以将先决条件类型列表关联到接口。例如，如果对象A希望实现接口I1，而接口I1在接口I2上有先决条件，则A必须同时实现I1和I2。

在实践中，上面描述的机制非常类似于Java的接口I1扩展接口I2。下面的例子显示了GObject的等价：

```c
/* Make the ViewerEditableLossy interface require ViewerEditable interface. */
G_DEFINE_INTERFACE (ViewerEditableLossy, viewer_editable_lossy, VIEWER_TYPE_EDITABLE)
```

在上面的`G_DEFINE_INTERFACE`调用中，第三个参数定义了先决条件类型。这是接口或类的GType。在这种情况下，`ViewerEditable`接口是`ViewerEditableLossy`的先决条件。下面的代码展示了一个实现如何实现两个接口并注册它们的实现：
```c
static void
viewer_file_editable_lossy_compress (ViewerEditableLossy *editable)
{
  ViewerFile *self = VIEWER_FILE (editable);

  g_print ("File implementation of lossy editable interface compress method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_lossy_interface_init (ViewerEditableLossyInterface *iface)
{
  iface->compress = viewer_file_editable_lossy_compress;
}

static void
viewer_file_editable_save (ViewerEditable  *editable,
                           GError         **error)
{
  ViewerFile *self = VIEWER_FILE (editable);

  g_print ("File implementation of editable interface save method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_undo (ViewerEditable *editable,
                           guint           n_steps)
{
  ViewerFile *self = VIEWER_FILE (editable);

  g_print ("File implementation of editable interface undo method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_redo (ViewerEditable *editable,
                           guint           n_steps)
{
  ViewerFile *self = VIEWER_FILE (editable);

  g_print ("File implementation of editable interface redo method: %s.\n",
           self->filename);
}

static void
viewer_file_editable_interface_init (ViewerEditableInterface *iface)
{
  iface->save = viewer_file_editable_save;
  iface->undo = viewer_file_editable_undo;
  iface->redo = viewer_file_editable_redo;
}

static void
viewer_file_class_init (ViewerFileClass *klass)
{
  /* Nothing here. */
}

static void
viewer_file_init (ViewerFile *self)
{
  /* Instance variable initialisation code. */
}

G_DEFINE_TYPE_WITH_CODE (ViewerFile, viewer_file, G_TYPE_OBJECT,
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE,
                                                viewer_file_editable_interface_init)
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE_LOSSY,
                                                viewer_file_editable_lossy_interface_init))
```

非常重要的是要注意，将接口实现添加到主对象的顺序不是随机的：`g_type_add_interface_static()` 由G_IMPLEMENT_INTERFACE调用，必须首先在没有先决条件的接口上调用，然后在其他接口上调用。

## 接口属性

对象接口也可以有属性。接口属性的声明类似于“对象属性”一节中解释的普通GObject类型的属性声明，除了使用`g_object_interface_install_property()`而不是`g_object_class_install_property()`来声明属性。

要在上面的ViewerEditable接口示例代码中包含一个名为‘autosave-frequency’的gdouble类型的属性，我们只需要在`viewer_editable_default_init()`中添加一个调用，如下所示：

```c
static void
viewer_editable_default_init (ViewerEditableInterface *iface)
{
  g_object_interface_install_property (iface,
    g_param_spec_double ("autosave-frequency",
                         "Autosave frequency",
                         "Frequency (in per-seconds) to autosave backups of the editable content at. "
                         "Or zero to disable autosaves.",
                         0.0,  /* minimum */
                         G_MAXDOUBLE,  /* maximum */
                         0.0,  /* default */
                         G_PARAM_READWRITE));
}
```

值得注意的一点是，声明的属性没有被分配一个整数ID。原因是属性的整数id只在get_property和set_property虚方法中使用。由于接口声明但不实现属性，因此不需要为它们分配整数id。

实现以“对象属性”一节中解释的常用方式声明和定义其属性，除了一个小变化：它可以使用g_object_class_override_property（）而不是g_object_class_install_property（）来声明它实现的接口的属性。下面的代码片段显示了上述ViewerFile声明和实现中所需的修改：

```c
struct _ViewerFile
{
  GObject parent_instance;

  double autosave_frequency;
};

enum
{
  PROP_AUTOSAVE_FREQUENCY = 1,
  N_PROPERTIES
};

static void
viewer_file_set_property (GObject      *object,
                          guint         prop_id,
                          const GValue *value,
                          GParamSpec   *pspec)
{
  ViewerFile *file = VIEWER_FILE (object);

  switch (prop_id)
    {
    case PROP_AUTOSAVE_FREQUENCY:
      file->autosave_frequency = g_value_get_double (value);
      break;

    default:
      G_OBJECT_WARN_INVALID_PROPERTY_ID (object, prop_id, pspec);
      break;
    }
}

static void
viewer_file_get_property (GObject    *object,
                          guint       prop_id,
                          GValue     *value,
                          GParamSpec *pspec)
{
  ViewerFile *file = VIEWER_FILE (object);

  switch (prop_id)
    {
    case PROP_AUTOSAVE_FREQUENCY:
      g_value_set_double (value, file->autosave_frequency);
      break;

    default:
      G_OBJECT_WARN_INVALID_PROPERTY_ID (object, prop_id, pspec);
      break;
    }
}

static void
viewer_file_class_init (ViewerFileClass *klass)
{
  GObjectClass *object_class = G_OBJECT_CLASS (klass);

  object_class->set_property = viewer_file_set_property;
  object_class->get_property = viewer_file_get_property;

  g_object_class_override_property (object_class, PROP_AUTOSAVE_FREQUENCY, "autosave-frequency");
}
```

## 覆盖接口方法

如果基类已经实现了一个接口，而派生类需要实现相同的接口，但需要覆盖某些方法，则必须重新实现该接口，并只设置需要覆盖的接口方法。

在这个例子中，ViewerAudioFile是从ViewerFile派生出来的。两者都实现了ViewerEditable接口。ViewerAudioFile只实现ViewerEditable接口的一个方法，并使用另一个方法的基类实现。

```c
static void
viewer_audio_file_editable_save (ViewerEditable  *editable,
                                 GError         **error)
{
  ViewerAudioFile *self = VIEWER_AUDIO_FILE (editable);

  g_print ("Audio file implementation of editable interface save method.\n");
}

static void
viewer_audio_file_editable_interface_init (ViewerEditableInterface *iface)
{
  /* Override the implementation of save(). */
  iface->save = viewer_audio_file_editable_save;

  /*
   * Leave iface->undo and ->redo alone, they are already set to the
   * base class implementation.
   */
}

G_DEFINE_TYPE_WITH_CODE (ViewerAudioFile, viewer_audio_file, VIEWER_TYPE_FILE,
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE,
                                                viewer_audio_file_editable_interface_init))

static void
viewer_audio_file_class_init (ViewerAudioFileClass *klass)
{
  /* Nothing here. */
}

static void
viewer_audio_file_init (ViewerAudioFile *self)
{
  /* Nothing here. */
}
```

要访问基类接口实现，请在接口的default_init函数中使用g_type_interface_peek_parent（）。

要从已重写接口方法的派生类调用接口方法的基类实现，请将g_type_interface_peek_parent（）返回的指针保存在全局变量中。

在这个例子中，ViewerAudioFile覆盖了save接口方法。在其重写的方法中，它调用相同接口方法的基类实现。

```c
static ViewerEditableInterface *viewer_editable_parent_interface = NULL;

static void
viewer_audio_file_editable_save (ViewerEditable  *editable,
                                 GError         **error)
{
  ViewerAudioFile *self = VIEWER_AUDIO_FILE (editable);

  g_print ("Audio file implementation of editable interface save method.\n");

  /* Now call the base implementation */
  viewer_editable_parent_interface->save (editable, error);
}

static void
viewer_audio_file_editable_interface_init (ViewerEditableInterface *iface)
{
  viewer_editable_parent_interface = g_type_interface_peek_parent (iface);

  iface->save = viewer_audio_file_editable_save;
}

G_DEFINE_TYPE_WITH_CODE (ViewerAudioFile, viewer_audio_file, VIEWER_TYPE_FILE,
                         G_IMPLEMENT_INTERFACE (VIEWER_TYPE_EDITABLE,
                                                viewer_audio_file_editable_interface_init))

static void
viewer_audio_file_class_init (ViewerAudioFileClass *klass)
{
  /* Nothing here. */
}

static void
viewer_audio_file_init (ViewerAudioFile *self)
{
  /* Nothing here. */
}
```

