你好，请问有什么可以帮助你的吗？

# 同步上下文

线程池是一个非常方便且大多数人都理解的东西，因为它非常简单。然而，如果能引入一个更抽象的概念会更方便，这个概念可以表示我想要在某个线程或一组线程中同步或异步执行某些代码。

我们生活中有很多这样的行为例子：比如 UI。在那里经常需要在执行线程中调用某些代码。但我们不能这样调用：

```csharp
Method.InvokeOn(thread2);
```
或者
```csharp
thread2.Invoke(Method)
```

这是不可能的。无法要求线程执行某个方法的代码。只有您的代码可以从某处获取某种方式传递的方法并执行它。

如果不使用任何中间抽象，我们会创建一个 ConcurrentQueue 实例，然后客户端代码会将委托放入队列中，组件的代码会在自己的线程中循环遍历这个队列并执行委托。通过 SyncronizationContext，问题可以以不同的方式解决。SynchronizationContext 充当访问某些线程的接口。就像 `IEnumerable<T>` 一样，将 SynchronizationContext 提供给接收方时，您并没有向接收方透露实现细节。接收方根据传递的具体上下文来计划代码的执行，可以是在 ThreadPool 上，也可以是在 UI 线程上，或者其他地方。

在某个线程组上计划工作的接口如下所示：

```csharp
public class SynchronizationContext
{
  void Post(..); // （异步）

  void Send(..); // （同步）
}
```

这个同步上下文可以隐藏任何东西：线程池、任何其他线程池，甚至只有一个线程。然而，值得一提的是，有一个默认的同步上下文，访问它的方式不太方便。需要创建它的实例：

```csharp
var ctx = new SynchronizationContext();

ctx.Post(...);
```

默认的同步上下文包装了 ThreadPool。因此，当您计划在那里工作时，委托会在线程池中执行。要创建自己的 SynchronizationContext，例如，包装您自己的线程，您需要从 SynchronizationContext 类继承。

## SynchronizationContext.Current

然而，假设我们在 ThreadPool 中的一个线程中工作。在这个委托中，我们被要求在当前同步上下文中运行某个委托（`SynchronizationContext.Current`）。但是它如何知道哪一个是当前的？如果我编写自己的线程池，如何为其中存在的线程设置当前的同步上下文？

为当前线程设置同步上下文是通过调用：

```csharp
SynchronizationContext.SetSynchronizationContext(myContext);
```

然后，例如，如果要创建自己的 ThreadPool，则顺序如下：
- 从 SynchronizationContext 继承您的类 MySynchronizationContext
- 在构造函数中接受线程池类的实例
- 在启动新线程时，在线程池中最开始调用 `SynchronizationContext.SetSynchronizationContext`，为它们设置它们的同步上下文
- 由于它们都是相同的，调用 `ctx.Post/.Send` 将重定向到 `MySynchronizationContext`

这有什么好处？通过提供一个 `SynchronizationContext` 实例，您实际上提供了一个“访问在某个线程组中安排任务的能力”的抽象。在我们的示例中，我们的线程池将被隐藏在这个抽象之下。

## 示例

非常好的同步上下文示例，不同于 ThreadPool 包装，可以是 WinForms 和 WPF 中的 UI 线程包装。

众所周知，WPF 和 WinForms 代码在一个单一的 UI 线程中执行。这与许多原因有关，但其中之一是 Windows 消息循环。为此循环创建一个新线程（或使用应用程序的主线程），并在该线程中创建一个无限循环以选择来自 Windows 的消息：按键、移动鼠标、拖放等。通过 UI 线程的同步上下文，您可以将一些委托交给它以在 UI 线程中执行，然后在下一个消息循环中选择委托并在 UI 线程中执行。

然而，对于以下算法：

```csharp
public void DoSomethig(int x, int y, SynchronizationContext ctx = default)
{
    ((ctx ?? SynchronizationContext.Current) ?? new SynchronizationContext())
        .Post(() => x + y);
}
```
解决方案如下：
- 如果传递了 SynchronizationContext，则使用它；
- 否则使用当前的；
- 如果当前未设置，则使用默认的上下文。

并在这个上下文中调用两个数字相加的代码：
- 如果使用 UI 的同步上下文调用此代码，则两个数字相加将在 UI 线程中发生；
- 如果不传递任何内容，则将选择当前的；
- 如果当前未设置，则将选择默认的上下文（选择方式非常不好）。