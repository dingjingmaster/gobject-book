# GObject消息传递系统

## 闭包（Closures）

闭包是在GTK和GNOME应用程序中广泛使用的异步信号传递概念的核心。闭包是一种抽象，是回调的通用表示。它是一个包含三个对象的小结构：

- 一个函数指针（回调本身），其原型看起来像：return_type function_callback(.., gpointer user_data)；
- user_data指针，在调用闭包时传递给回调函数
- 表示闭包析构函数的函数指针：每当闭包的refcount达到零时，该函数将在释放闭包结构之前被调用

GClosure结构代表了所有闭包实现的通用功能：对于每个想要使用GObject类型系统的独立运行时，存在不同的闭包实现。GObject库提供了一个简单的GCClosure类型，它是用于C/c++回调的闭包的特定实现。

GClosure提供简单的服务：
- 调用（g_closure_invoke()）：这是创建闭包的目的；它们对回调调用者隐藏了回调调用的细节。
- 通知：闭包通知侦听器某些事件，如调用闭包、闭包无效和闭包结束。监听器可以注册为`g_closure_add_finalize_notifier()`（结束通知），`g_closure_add_invalidate_notifier()`（无效通知）和`g_closure_add_marshal_guards()`（调用通知）。存在用于结束和无效事件的对称注销函数（`g_closure_remove_finalize_notifier()`和`g_closure_remove_invalidate_notifier()`），但不用于调用过程

## C 闭包

如果您使用C或c++将回调连接到给定事件，您将使用具有相当最小API的简单GCClosures或更简单的`g_signal_connect()`函数（稍后将介绍）。

`g_cclosure_new()`将创建一个新的闭包，它可以用用户提供的`user_data`作为最后一个参数调用用户提供的`callback_func`。当闭包完成时（销毁过程的第二阶段），如果用户提供了`destroy_data`函数，它将调用该函数。

`g_cclosure_new_swap()`将创建一个新的闭包，它可以用用户提供的user_data作为第一个参数调用用户提供的callback_func（而不是像`g_cclosure_new()`那样作为最后一个参数）。当闭包完成时（销毁过程的第二阶段），如果用户提供了destroy_data函数，它将调用该函数。

## 非C闭包

如上所述，闭包隐藏了回调调用的细节。在C语言中，回调调用就像函数调用一样：它是为被调用的函数创建正确的堆栈框架并执行调用汇编指令的问题。

C闭包编组器将表示目标函数参数的GValue数组转换为C风格的函数参数列表，用这个新的参数列表调用用户提供的C函数，获取函数的返回值，将其转换为GValue并将该GValue返回给编组器调用者。

通用的C闭包编组器为`g_cclosure_marshal_generic()`，它使用libffi实现对所有函数类型的编组。除了基于libffi的编组器可能太慢的性能关键代码外，不需要针对不同类型的自定义编组器。

下面给出了一个自定义编组程序的示例，说明了如何将GValues转换为C函数调用。该编组器用于C函数，该函数以整数作为其第一个参数并返回void。

```c
g_cclosure_marshal_VOID__INT (GClosure* closure, GValue* return_value, guint n_param_values, const GValue* param_values, gpointer invocation_hint, gpointer marshal_data)
{
    typedef void (*GMarshalFunc_VOID__INT) (gpointer data1, gint arg_1, gpointer data2);
    register GMarshalFunc_VOID__INT callback;
    register GCClosure *cc = (GCClosure*) closure;
    register gpointer data1, data2;

    g_return_if_fail (n_param_values == 2);

    data1 = g_value_peek_pointer (param_values + 0);
    data2 = closure->data;

    callback = (GMarshalFunc_VOID__INT) (marshal_data ? marshal_data : cc->callback);

    callback (data1, g_marshal_value_peek_int (param_values + 1), data2);
}
```

还有其他类型的编组器，例如有一个通用的Python编组器，它被所有Python闭包使用（Python闭包用于调用用Python编写的回调）。这个Python编组器将表示函数参数的输入GValue列表转换为Python元组，这是Python中的等效结构。

## 信号

GObject的信号与标准UNIX信号无关：它们将任意特定于应用程序的事件与任意数量的监听器连接起来。例如，在GTK中，从窗口系统接收每个用户事件（击键或鼠标移动），并在小部件对象实例上以信号发射的形式生成GTK事件。

在类型系统中，每个信号都与发出信号的类型一起被注册：当用户在信号发出时注册要调用的闭包时，该类型的用户被称为连接到给定类型实例上的信号。用户也可以自己发射信号，或者从连接到信号的一个闭包内停止发射信号。

当在给定类型实例上发出信号时，将调用该类型实例上连接到该信号的所有闭包。连接到这样一个信号的所有闭包都表示回调函数，其签名如下：

```c
return_type function_callback (gpointer instance, ..., gpointer user_data);
```

## 注册信号

要在现有类型上注册新信号，可以使用`g_signal_newv()`、`g_signal_new_valist()`或`g_signal_new()`函数中的任何一个：
```c
guint
g_signal_newv (const gchar        *signal_name,
               GType               itype,
               GSignalFlags        signal_flags,
               GClosure           *class_closure,
               GSignalAccumulator  accumulator,
               gpointer            accu_data,
               GSignalCMarshaller  c_marshaller,
               GType               return_type,
               guint               n_params,
               GType              *param_types);
```

这些函数的参数数量有点吓人，但它们相对简单：
- `signal_name`：是一个字符串，可用于唯一标识给定的信号
- `type`：可以发出该信号的实例类型
- `signal_flags`：部分定义了连接到信号的闭包被调用的顺序
- `class_closure`：这是信号的默认闭包：如果在信号发射时它不是NULL，它将在信号发射时被调用。与连接到该信号的其他闭包相比，调用此闭包的时间部分取决于signal_flags
- `accumulator`：这是一个函数指针，在每个闭包被调用后调用。如果返回FALSE，则停止信号发射。如果返回TRUE，则信号发射正常进行。它还用于根据所有调用的闭包的返回值计算信号的返回值。例如，累加器可以忽略来自闭包的NULL返回；或者，它可以构建闭包返回值的列表
- `accu_data`：这个指针将在每次触发累加器时向下传递
- `c_marshaller`：这是连接到此信号的任何闭包的默认C编组器
- `return_type`：这是信号返回值的类型
- `n_params`：这是这个信号接受的参数的数量
- `param_types`：这是一个GTypes数组，表示信号的每个参数的类型。这个数组的长度由n_params表示。

从上面的定义可以看出，信号基本上是可以连接到该信号的闭包的描述，以及连接到该信号的闭包将被调用的顺序的描述。

## 信号连接

如果你想用闭包连接一个信号，你有三种可能：
- 你可以在信号注册时注册一个类闭包：这是一个系统范围的操作。即：类闭包将在支持该信号的类型的任何实例的每次给定信号发射期间被调用
- 可以使用`g_signal_override_class_closure()`来覆盖给定类型的类闭包。可以只在注册信号的类型的派生类型上调用此函数。此函数仅用于语言绑定
- 您可以使用`g_signal_connect()`函数族注册闭包。这是一个特定于实例的操作：只有在给定实例上发出给定信号时才会调用闭包

也可以在给定信号上连接不同类型的回调：无论给定信号在哪个实例上发出，每当发出信号时都会调用发出钩子。排放钩子使用`g_signal_add_emission_hook()`连接，并使用`g_signal_remove_emission_hook()`删除。

## 信号发射

信号发射是通过使用`g_signal_emit()`函数族来完成的。

```c
void
g_signal_emitv (const GValue  instance_and_params[],
                guint         signal_id,
                GQuark        detail,
                GValue       *return_value);
```

- GValues的instance_and_params数组包含信号的输入参数列表。数组的第一个元素是调用信号的实例指针。数组的以下元素包含信号的参数列表
- signal_id标识要调用的信号
- detail标识要调用的信号的具体细节。细节是一种神奇的令牌/参数，在信号发射期间传递，并由连接到信号的闭包使用，以过滤掉不需要的信号发射。在大多数情况下，可以安全地将此值设置为零。有关此参数的更多信息，请参阅“详细参数”一节
- 如果没有指定累加器，return_value保存在发射期间调用的最后一个闭包的返回值。如果在信号创建期间指定了一个累加器，那么这个累加器将用于计算返回值，作为在发射期间调用的所有闭包返回值的函数。如果在发射期间没有调用闭包，return_value仍然初始化为0/NULL

信号发射可分为6步：
1. RUN_FIRST：如果在信号注册期间使用了`G_SIGNAL_RUN_FIRST`标志，并且存在该信号的类闭包，则调用类闭包。
2. EMISSION_HOOK：如果任何发射钩子被添加到信号中，它们将从第一个到最后一个被调用。累积返回值。
3. HANDLER_RUN_FIRST：如果任何闭包与`g_signal_connect()`系列函数连接，并且如果它们没有被阻塞（使用`g_signal_handler_block()`系列函数），它们将在这里运行，从第一个连接到最后一个连接。
4. RUN_LAST：如果在注册期间设置了G_SIGNAL_RUN_LAST标志，并且设置了类闭包，则在这里调用它。
5. HANDLER_RUN_LAST：如果任何闭包与`g_signal_connect_after()`函数族连接，如果它们在`HANDLER_RUN_FIRST`期间没有被调用，如果它们没有被阻塞，它们将在这里运行，从第一个连接到最后一个连接。
6. RUN_CLEANUP：如果在注册期间设置了`G_SIGNAL_RUN_CLEANUP`标志，并且设置了类闭包，则在这里调用它。信号发射在这里完成。

如果在发射期间的任何时刻（RUN_CLEANUP或EMISSION_HOOK状态除外），其中一个闭包使用`g_signal_stop_emission()`停止信号发射，则发射跳转到RUN_CLEANUP状态。

如果在发射期间的任何时刻，其中一个闭包或发射钩子在同一实例上发出相同的信号，则从RUN_FIRST状态重新启动发射。

在调用每个闭包之后，在所有状态下都调用accumulator函数（除了RUN_EMISSION_HOOK和RUN_CLEANUP）。它将闭包返回值累加到信号返回值中，并返回TRUE或FALSE。如果在任何时候它都没有返回TRUE，则释放跳转到RUN_CLEANUP状态。

如果没有提供累加器函数，则最后运行的处理程序返回的值将由`g_signal_emit()`返回。

## 参数细节

所有与信号发射或信号连接相关的函数都有一个名为detail的参数。有时，这个参数被API隐藏，但它总是以这样或那样的形式存在。

在三个主要的连接函数中，只有一个函数有一个明确的细节参数作为GQuark: `g_signal_connect_closure_by_id()`。

另外两个函数`g_signal_connect_closure()`和`g_signal_connect_data()`隐藏了信号名称标识中的detail参数。它们的detailed_signal参数是一个字符串，用于标识要连接的信号的名称。该字符串的格式应该匹配signal_name::detail_name。例如，连接到名为notify::cursor_position的信号实际上会连接到带有cursor_position细节的名为notify的信号。在内部，如果存在细节字符串，则将其转换为GQuark。

在四个主要的信号发射函数中，有一个函数将其隐藏在其信号名称参数`g_signal_emit_by_name()`中。其他三个也有一个显式的细节参数作为GQuark: g_signal_emit(), `g_signal_emitv()`和`g_signal_emit_valist()`。

如果用户向发射函数提供了一个细节，则在发射期间使用它来匹配也提供了一个细节的闭包。如果闭包的细节与用户提供的细节不匹配，它将不会被调用（即使它连接到正在发出的信号）。

这种完全可选的过滤机制主要用于对通常出于多种不同原因发出的信号进行优化：客户端可以在闭包的编组代码运行之前过滤出它们感兴趣的事件。例如，这被GObject的通知信号广泛使用：每当GObject上的属性被修改时，GObject不只是发出通知信号，而是将被修改属性的名称作为细节关联到该信号的发出。这允许希望只收到一个属性更改通知的客户端在接收大多数事件之前过滤它们。

作为一个简单的规则，用户可以并且应该将细节参数设置为零：这将完全禁用该信号的可选过滤。
