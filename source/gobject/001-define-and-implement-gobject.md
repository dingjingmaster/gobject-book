# 定义和实现一个新的GObject

本文档重点介绍GObject子类型的实现，例如创建自定义类层次结构，或创建GTK小部件的子类。

在整个章节中，使用了一个文件查看程序的运行示例，它有一个`ViewerFile`类来表示正在查看的单个文件，以及用于具有特殊功能的不同类型文件的各种派生类，例如音频文件。示例应用程序还支持使用`ViewerEditable`界面编辑文件（例如，调整正在查看的照片）。

## 头文件代码

在为GObject编写代码之前的第一步是编写类型的头文件，其中包含所需的类型、函数和宏定义。这些元素中的每一个都是几乎所有GObject用户都遵循的约定，并且经过多年开发基于对象的代码的经验得到了改进。如果你正在编写一个库，对你来说，严格遵守这些约定是特别重要的；库的用户会认为您拥有。即使不编写库，它也会帮助其他想要参与您的项目的人。

为你的头文件和源代码选择一个名称约定，并坚持这样做：
- 使用破折号将前缀与类型名分开：viewer-file.h和viewer-file.c（这是大多数GNOME库和应用程序使用的约定）
- 使用下划线将前缀与类型名分开：viewer_file.h和viewer_file.c
- 不要将前缀与类型名分开：viewerfile.h和viewerfile.c（这是GTK使用的约定）

如果你想在命名空间“viewer”中声明一个名为“file”的类型，将类型实例命名为ViewerFile，并将其类命名为ViewerFileClass（名称区分大小写）。声明类型的推荐方法根据类型是最终类型还是可派生类型而有所不同。

final 类型不能进一步子类化，并且应该是新类型的默认选择——将最终类型更改为可派生类型总是与代码的现有用途兼容的更改，但反之通常会导致问题。最终类型是使用`G_DECLARE_FINAL_TYPE`宏声明的，并且需要一个结构来保存要在源代码（而不是头文件）中声明的实例数据。

```c
/*
 * Copyright/Licensing information.
 */

/* inclusion guard */
#pragma once

#include <glib-object.h>

/*
 * Potentially, include other headers on which this header depends.
 */

G_BEGIN_DECLS

/*
 * Type declaration.
 */
#define VIEWER_TYPE_FILE viewer_file_get_type()
G_DECLARE_FINAL_TYPE (ViewerFile, viewer_file, VIEWER, FILE, GObject)

/*
 * Method definitions.
 */
ViewerFile *viewer_file_new (void);

G_END_DECLS
```

可派生类型可以进一步子类化，它们的类和实例结构构成了公共API的一部分，如果关心API的稳定性，就不能改变这些结构。它们是用`G_DECLARE_DERIVABLE_TYPE`宏声明的：

```c
/*
 * Copyright/Licensing information.
 */

/* inclusion guard */
#pragma once

#include <glib-object.h>

/*
 * Potentially, include other headers on which this header depends.
 */

G_BEGIN_DECLS

/*
 * Type declaration.
 */
#define VIEWER_TYPE_FILE viewer_file_get_type()
G_DECLARE_DERIVABLE_TYPE (ViewerFile, viewer_file, VIEWER, FILE, GObject)

struct _ViewerFileClass
{
  GObjectClass parent_class;

  /* Class virtual function fields. */
  void (* open) (ViewerFile  *file,
                 GError     **error);

  /* Padding to allow adding up to 12 new virtual functions without
   * breaking ABI. */
  gpointer padding[12];
};

/*
 * Method definitions.
 */
ViewerFile *viewer_file_new (void);

G_END_DECLS
```

头文件包含的约定是在编译头文件所需的头文件顶部添加最少数量的#include指令。这使得客户端代码可以简单地#include "viewer-file.h"，而不需要知道viewer-file.h的先决条件。

## 源文件代码

在你的代码中，第一步是#include所需的头文件：

```c
/*
 * Copyright/Licensing information
 */

#include "viewer-file.h"

/* Private structure definition. */
typedef struct {
  char *filename;

  /* other private fields */
} ViewerFilePrivate;

/*
 * forward definitions
 */
```

如果使用`G_DECLARE_FINAL_TYPE`将类声明为final，它的实例结构应该在C文件中定义：

```c
struct _ViewerFile
{
  GObject parent_instance;

  /* Other members, including private data. */
};
```

调用`G_DEFINE_TYPE`宏（或者`G_DEFINE_TYPE_WITH_PRIVATE`，如果你的类需要私有数据——最终类型不需要私有数据），使用类型的名称、函数的前缀和父GType来减少所需的样板文件数量。这个宏将：
- 实现viewer_file_get_type函数
- 定义一个可以从整个.c文件访问的父类指针
- 将私有实例数据添加到类型（如果使用`G_DEFINE_TYPE_WITH_PRIVATE`）

如果使用`G_DECLARE_FINAL_TYPE`将类声明为final，则应该将私有数据放在实例结构中，应该使用viewfile和`G_DEFINE_TYPE`而不是`G_DEFINE_TYPE_WITH_PRIVATE`。`final`类的实例结构不会公开，也不会嵌入到任何派生类的实例结构中（因为该类是final类）；因此，它的大小可以改变，而不会导致使用该类的代码不兼容。相反，可派生类的私有数据必须包含在私有结构中，并且必须使用`G_DEFINE_TYPE_WITH_PRIVATE`。

```c
G_DEFINE_TYPE (ViewerFile, viewer_file, G_TYPE_OBJECT)
```
或
```c
G_DEFINE_TYPE_WITH_PRIVATE (ViewerFile, viewer_file, G_TYPE_OBJECT)
```

还可以使用`G_DEFINE_TYPE_WITH_CODE`宏来控制`get_type`函数的实现——例如，添加对`G_IMPLEMENT_INTERFACE`宏的调用来实现一个接口。

## 构造函数

人们在尝试构建对象时经常会感到困惑，因为在对象的构建过程中有很多不同的方法：很难确定哪一种是正确的，推荐的方法。

关于对象实例化的文档显示了在对象实例化期间调用哪些用户提供的函数以及调用它们的顺序。如果用户要查找简单c++构造函数的等价物，应该使用instance_init方法。它将在所有父类的instance_init函数被调用之后被调用。它不能接受任意的构造参数（就像在c++中一样），但是如果你的对象需要任意参数来完成初始化，你可以使用构造属性。

只有在所有instance_init函数运行之后才会设置构造属性。在设置了所有构造属性之前，不会向`g_object_new()`的客户端返回任何对象引用。

重要的是要注意对象构造永远不会失败。如果您需要一个易出错的GObject构造，您可以使用GIO库提供的GInitable和GAsyncInitable接口。

```c
G_DEFINE_TYPE_WITH_PRIVATE (ViewerFile, viewer_file, G_TYPE_OBJECT)

static void
viewer_file_class_init (ViewerFileClass *klass)
{
}

static void
viewer_file_init (ViewerFile *self)
{
  ViewerFilePrivate *priv = viewer_file_get_instance_private (self);

  /* initialize all public and private members to reasonable default values.
   * They are all automatically initialized to 0 to begin with. */
}
```

如果您需要特殊的构造属性（设置`G_PARAM_CONSTRUCT_ONLY`），请在`class_init()`函数中安装这些属性，覆盖GObject类的`set_property()`和`get_property()`方法，并按照“对象属性”一节的描述实现它们。

属性标识符必须从1开始，因为0保留给GObject内部使用。

```c
enum
{
  PROP_FILENAME = 1,
  PROP_ZOOM_LEVEL,
  N_PROPERTIES
};

static GParamSpec *obj_properties[N_PROPERTIES] = { NULL, };

static void
viewer_file_class_init (ViewerFileClass *klass)
{
  GObjectClass *object_class = G_OBJECT_CLASS (klass);

  object_class->set_property = viewer_file_set_property;
  object_class->get_property = viewer_file_get_property;

  obj_properties[PROP_FILENAME] =
    g_param_spec_string ("filename",
                         "Filename",
                         "Name of the file to load and display from.",
                         NULL  /* default value */,
                         G_PARAM_CONSTRUCT_ONLY | G_PARAM_READWRITE);

  obj_properties[PROP_ZOOM_LEVEL] =
    g_param_spec_uint ("zoom-level",
                       "Zoom level",
                       "Zoom level to view the file at.",
                       0  /* minimum value */,
                       10 /* maximum value */,
                       2  /* default value */,
                       G_PARAM_READWRITE);

  g_object_class_install_properties (object_class,
                                     N_PROPERTIES,
                                     obj_properties);
}
```

如果需要这样做，请确保可以构建并运行类似于上面所示代码的代码。此外，要确保在构造过程中设置构造属性时不会产生副作用。

有些人有时需要在传递给构造函数的属性设置完成之后才完成类型实例的初始化。这可以通过使用`constructor()`类方法实现，如“对象实例化”一节所述，或者更简单地说，使用`constructor()`类方法。注意，`construct()`虚函数只会在标记为`G_PARAM_CONSTRUCT_ONLY`或`G_PARAM_CONSTRUCT`的属性被使用之后调用，但是在传递给`g_object_new()`的常规属性被设置之前调用。

## 析构

同样，通常很难弄清楚使用哪种机制挂钩到对象的销毁过程：当执行最后一个`g_object_unref()`函数调用时，会发生文档“对象内存管理”一节中描述的许多事情。

对象的销毁过程分为两个阶段：dispose和finalize。由于GObject使用的引用计数机制的性质，这种分割对于处理潜在的循环是必要的，并且在销毁序列期间信号发射的情况下处理实例的临时恢复也是必要的。

```c
struct _ViewerFilePrivate
{
  gchar *filename;
  guint zoom_level;

  GInputStream *input_stream;
};

G_DEFINE_TYPE_WITH_PRIVATE (ViewerFile, viewer_file, G_TYPE_OBJECT)

static void viewer_file_dispose (GObject *gobject)
{
  ViewerFilePrivate *priv = viewer_file_get_instance_private (VIEWER_FILE (gobject));

  /* In dispose(), you are supposed to free all types referenced from this
   * object which might themselves hold a reference to self. Generally,
   * the most simple solution is to unref all members on which you own a
   * reference.
   */

  /* dispose() might be called multiple times, so we must guard against
   * calling g_object_unref() on an invalid GObject by setting the member
   * NULL; g_clear_object() does this for us.
   */
  g_clear_object (&priv->input_stream);

  /* Always chain up to the parent class; there is no need to check if
   * the parent class implements the dispose() virtual function: it is
   * always guaranteed to do so
   */
  G_OBJECT_CLASS (viewer_file_parent_class)->dispose (gobject);
}

static void viewer_file_finalize (GObject *gobject)
{
  ViewerFilePrivate *priv = viewer_file_get_instance_private (VIEWER_FILE (gobject));

  g_free (priv->filename);

  /* Always chain up to the parent class; as with dispose(), finalize()
   * is guaranteed to exist on the parent's class virtual function table
   */
  G_OBJECT_CLASS (viewer_file_parent_class)->finalize (gobject);
}

static void viewer_file_class_init (ViewerFileClass *klass)
{
  GObjectClass *object_class = G_OBJECT_CLASS (klass);

  object_class->dispose = viewer_file_dispose;
  object_class->finalize = viewer_file_finalize;
}

static void viewer_file_init (ViewerFile *self);
{
  ViewerFilePrivate *priv = viewer_file_get_instance_private (self);

  priv->input_stream = g_object_new (VIEWER_TYPE_INPUT_STREAM, NULL);
  priv->filename = /* would be set as a property */;
}
```
对象方法可能在dispose运行之后和finalize运行之前被调用。GObject不认为这是一个程序错误：您必须优雅地检测到这一点，既不崩溃也不警告用户，通过将已处置的实例恢复到惰性状态。

## 对象方法

就像c++一样，有许多不同的方法来定义和扩展对象方法：下面的列表和部分利用了c++词汇表。(期望读者了解基本的c++概念。那些最近没有编写c++代码的人可以参考c++教程来刷新他们的记忆。)

- 非虚拟公共方法
- 虚拟公共方法
- 虚拟私有方法
- 非虚拟私有方法

### 非虚拟方法

这些是最简单的，提供一个作用于对象的简单方法。在头文件中提供一个函数原型，并在源文件中提供该原型的实现。

```c
/* declaration in the header. */
void viewer_file_open (ViewerFile* self, GError** error);
```

```c
/* implementation in the source file */
void viewer_file_open (ViewerFile* self, GError** error)
{
  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  /* do stuff here. */
}
```

### 虚拟公共方法

这是创建具有可重写方法的对象的首选方法：
- 在public头文件的类结构中定义公共方法及其虚函数
- 在头文件中定义通用方法，并在源文件中实现它
- 在源文件中实现虚函数的基本版本，并在对象的class_init函数中初始化指向此实现的虚函数指针；或者为必须被派生类覆盖的“纯虚拟”方法保留NULL
- 在每个需要重写虚函数的派生类中重新实现虚函数

注意，虚函数只能在类是可派生的情况下定义，使用`G_DECLARE_DERIVABLE_TYPE`声明，这样就可以定义类结构。

```c
/* declaration in viewer-file.h. */
#define VIEWER_TYPE_FILE viewer_file_get_type ()
G_DECLARE_DERIVABLE_TYPE (ViewerFile, viewer_file, VIEWER, FILE, GObject)

struct _ViewerFileClass
{
  GObjectClass parent_class;

  /* stuff */
  void (*open) (ViewerFile  *self,
                GError     **error);

  /* Padding to allow adding up to 12 new virtual functions without
   * breaking ABI. */
  gpointer padding[12];
};

void viewer_file_open (ViewerFile* self, GError** error);
```

```c
/* implementation in viewer-file.c */
void
viewer_file_open (ViewerFile  *self,
                  GError     **error)
{
  ViewerFileClass *klass;

  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  klass = VIEWER_FILE_GET_CLASS (self);
  g_return_if_fail (klass->open != NULL);

  klass->open (self, error);
}
```

上面的代码只是将open调用重定向到相关的虚函数。

可以在对象的class_init函数中为该类方法提供默认实现：将class ->open字段初始化为指向实际实现的指针。默认情况下，未继承的类方法初始化为NULL，因此被认为是“纯虚的”。

```c
static void
viewer_file_real_close (ViewerFile  *self,
                        GError     **error)
{
  /* Default implementation for the virtual method. */
}

static void
viewer_file_class_init (ViewerFileClass *klass)
{
  /* this is not necessary, except for demonstration purposes.
   *
   * pure virtual method: mandates implementation in children.
   */
  klass->open = NULL;

  /* merely virtual method. */
  klass->close = viewer_file_real_close;
}

void
viewer_file_open (ViewerFile  *self,
                  GError     **error)
{
  ViewerFileClass *klass;

  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  klass = VIEWER_FILE_GET_CLASS (self);

  /* if the method is purely virtual, then it is a good idea to
   * check that it has been overridden before calling it, and,
   * depending on the intent of the class, either ignore it silently
   * or warn the user.
   */
  g_return_if_fail (klass->open != NULL);
  klass->open (self, error);
}

void
viewer_file_close (ViewerFile  *self,
                   GError     **error)
{
  ViewerFileClass *klass;

  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  klass = VIEWER_FILE_GET_CLASS (self);
  if (klass->close != NULL)
    klass->close (self, error);
}
```

### 虚拟私有方法

这些与虚拟公共方法非常相似。它们只是没有可以直接调用的公共函数。头文件只包含虚函数的声明：

```c
/* declaration in viewer-file.h. */
struct _ViewerFileClass
{
  GObjectClass parent;

  /* Public virtual method as before. */
  void     (*open)           (ViewerFile  *self,
                              GError     **error);

  /* Private helper function to work out whether the file can be loaded via
   * memory mapped I/O, or whether it has to be read as a stream. */
  gboolean (*can_memory_map) (ViewerFile *self);

  /* Padding to allow adding up to 12 new virtual functions without
   * breaking ABI. */
  gpointer padding[12];
};

void viewer_file_open (ViewerFile *self, GError **error);
```

这些虚函数通常用于将部分工作委托给子类：

```c
/* this accessor function is static: it is not exported outside of this file. */
static gboolean
viewer_file_can_memory_map (ViewerFile *self)
{
  return VIEWER_FILE_GET_CLASS (self)->can_memory_map (self);
}

void
viewer_file_open (ViewerFile  *self,
                  GError     **error)
{
  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (error == NULL || *error == NULL);

  /*
   * Try to load the file using memory mapped I/O, if the implementation of the
   * class determines that is possible using its private virtual method.
   */
  if (viewer_file_can_memory_map (self))
    {
      /* Load the file using memory mapped I/O. */
    }
  else
    {
      /* Fall back to trying to load the file using streaming I/O… */
    }
}
```

同样，可以为这个私有虚函数提供默认实现：

```c
static gboolean
viewer_file_real_can_memory_map (ViewerFile *self)
{
  /* As an example, always return false. Or, potentially return true if the
   * file is local. */
  return FALSE;
}

static void
viewer_file_class_init (ViewerFileClass *klass)
{
  /* non-pure virtual method; does not have to be implemented in children. */
  klass->can_memory_map = viewer_file_real_can_memory_map;
}
```

然后，派生类可以用如下代码重写该方法：

```c
static void
viewer_audio_file_class_init (ViewerAudioFileClass *klass)
{
  ViewerFileClass *file_class = VIEWER_FILE_CLASS (klass);

  /* implement pure virtual function. */
  file_class->can_memory_map = viewer_audio_file_can_memory_map;
}
```

### Chaining up

链接通常由以下条件松散地定义：
- 父类A定义了一个名为foo的公共虚方法，并提供了一个默认实现
- 子类B重新实现方法foo
- B的foo实现调用（‘链到’）它的父类A的foo实现

这个习语有多种用法：
- 您需要在不修改代码的情况下扩展类的行为。您创建一个子类来继承它的实现，重新实现一个公共虚拟方法来修改行为，并链接以确保之前的行为没有真正修改，只是扩展
- 您需要实现责任链模式：继承树的每个对象都链接到它的父对象（通常在方法的开始或结束处），以确保每个处理程序依次运行

要显式链接到父类中的虚方法实现，首先需要一个原始父类结构的句柄。然后可以使用该指针访问原始虚函数指针并直接调用它

上面句子中使用的“原始”形容词并非无害。为了完全理解它的含义，回想一下类结构是如何初始化的：对于每个对象类型，与此对象关联的类结构是通过首先复制其父类型的类结构（一个简单的memcpy），然后在结果类结构上调用class_init回调来创建的。由于class_init回调负责用类方法的用户重新实现覆盖类结构，因此不能使用存储在派生实例中的父类结构的修改副本。需要父类实例的类结构的副本。

为了链接，你可以使用由G_DEFINE_TYPE系列宏创建和初始化的parent_class指针，例如：
```c
static void
b_method_to_call (B *obj, int some_param)
{
  /* do stuff before chain up */

  /* call the method_to_call() virtual function on the
   * parent of BClass, AClass.
   *
   * remember the explicit cast to AClass*
   */
  A_CLASS (b_parent_class)->method_to_call (obj, some_param);

  /* do stuff after chain up */
}
```


