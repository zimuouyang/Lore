## 重组

在命令式界面模型中，如需更改某个 widget，您可以在该 widget 上调用 setter 以更改其内部状态。在 Compose 中，您可以使用新数据再次调用可组合函数。这样做会导致函数进行重组 -- 系统会根据需要使用新数据重新绘制函数发出的 widget。Compose 框架可以智能地仅重组已更改的组件。

## 声明性范式转变

在 Compose 的声明性方法中，widget 相对无状态，并且不提供 setter 或 getter 函数。实际上，widget 不会以对象形式提供。您可以通过调用带有不同参数的同一可组合函数来更新界面。这使得向架构模式（如 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel)）提供

## 可组合函数可以并行运行

### 附带效应：附带效应是指对应用的其余部分可见的任何更改

Compose 可以通过并行运行可组合函数来优化重组。这样一来，Compose 就可以利用多个核心，并以较低的优先级运行可组合函数（不在屏幕上）。

这种优化意味着，可组合函数可能会在后台线程池中执行。如果某个可组合函数对 `ViewModel` 调用一个函数，则 Compose 可能会同时从多个线程调用该函数。

为了确保应用正常运行，所有可组合函数都不应有附带效应，而应通过始终在界面线程上执行的 `onClick` 等回调触发附带效应。

调用某个可组合函数时，调用可能发生在与调用方不同的线程上。这意味着，应避免使用修改可组合 lambda 中的变量的代码，既因为此类代码并非线程安全代码，又因为它是可组合 lambda 不允许的附带效应。

以下示例展示了一个可组合项，它显示一个列表及其项数：

```kotlin
@Composable
fun ListComposable(myList: List<String>) {
    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
            }
        }
        Text("Count: ${myList.size}")
    }
}
```

此代码没有附带效应，它会将输入列表转换为界面。此代码非常适合显示小列表。不过，如果函数写入局部变量，则这并非线程安全或正确的代码：

```kotlin
@Composable
@Deprecated("Example with bug")
fun ListWithBug(myList: List<String>) {
    var items = 0

    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
                items++ // Avoid! Side-effect of the column recomposing.
            }
        }
        Text("Count: $items")
    }
}
```

在本例中，每次重组时，都会修改 `items`。这可以是动画的每一帧，或是在列表更新时。但不管怎样，界面都会显示错误的项数。因此，Compose 不支持这样的写入操作；通过禁止此类写入操作，我们允许框架更改线程以执行可组合 lambda。