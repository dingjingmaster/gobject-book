# 创建和使用信号

GType中的信号系统非常复杂和灵活：它的用户可以在运行时将任意数量的回调（用存在绑定的任何语言实现）连接到任何信号，并在信号发射过程的任何状态停止任何信号的发射。这种灵活性使得使用GSignal可以做的不仅仅是向多个客户端发送信号。

## 信号的简单使用

信号最基本的用途是实现事件通知。例如，给定一个带有write方法的ViewerFile对象，只要使用该方法更改文件，就会发出一个信号。下面的代码显示了用户如何将回调连接到“已更改”的信号。

```c
file = g_object_new (VIEWER_FILE_TYPE, NULL);

g_signal_connect (file, "changed", (GCallback) changed_event, NULL);

viewer_file_write (file, buffer, strlen (buffer));
```

ViewerFile信号在class_init函数中注册：

```c
file_signals[CHANGED] = 
  g_signal_newv ("changed",
                 G_TYPE_FROM_CLASS (object_class),
                 G_SIGNAL_RUN_LAST | G_SIGNAL_NO_RECURSE | G_SIGNAL_NO_HOOKS,
                 NULL /* closure */,
                 NULL /* accumulator */,
                 NULL /* accumulator data */,
                 NULL /* C marshaller */,
                 G_TYPE_NONE /* return_type */,
                 0     /* n_params */,
                 NULL  /* param_types */);
```

信号在viewer_file_write中发出：

```c
void
viewer_file_write (ViewerFile   *self,
                   const guint8 *buffer,
                   gsize         size)
{
  g_return_if_fail (VIEWER_IS_FILE (self));
  g_return_if_fail (buffer != NULL || size == 0);

  /* First write data. */

  /* Then, notify user of data written. */
  g_signal_emit (self, file_signals[CHANGED], 0 /* details */);
}
```
如上所示，如果不需要传递任何细节，可以安全地将details参数设置为零。有关它的用途的讨论，请参见“细节参数”一节。

C信号编组器应该始终为NULL，在这种情况下，GLib将选择给定闭包类型的最佳编组器。这可以是特定于闭包类型的内部编组器，也可以是g_cclosure_marshal_generic()，它实现了参数数组到C回调调用的通用转换。GLib过去常常要求用户编写或生成特定于类型的编组程序并传递该编组程序，但现在已不赞成这种做法，而是支持自动选择编组程序。

请注意，g_cclosure_marshal_generic（）比非通用编组器慢，因此对于性能关键的代码应该避免使用。然而，性能关键型代码无论如何都不应该使用信号，因为信号是同步的，并且直到所有侦听器都被调用时才会发出信号，这可能会带来无限的成本。
