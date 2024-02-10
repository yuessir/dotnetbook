### CLR 异常

> [讨论链接](https://github.com/sidristij/dotnetbook/issues/51)

有一些异常情况，可以说...比其他异常情况更为特殊。如果试图对它们进行分类，就像在本章开头所说的那样，有一些异常是来自于.NET应用程序本身，还有一些异常是来自于不安全的世界。它们可以进一步分为两个子类别：CLR核心异常情况（本质上是不安全的）和任何外部库中的不安全代码。

[todo]

#### ThreadAbortException

总的来说，这可能看起来不太明显，但有四种类型的线程中止。

- 粗暴的线程中止，一旦触发就无法停止，并且不会启动任何异常处理程序，包括`finally`块
- 在当前线程上调用Thread.Abort()方法
- 由其他线程引发的异步线程中止异常
- 如果在卸载AppDomain时存在正在运行为该域编译的方法的线程，则会中止这些线程

> 值得注意的是，ThreadAbortException在 _大部分_ .NET Framework 中被广泛使用，但在CoreCLR、.NET Core或Windows 8“Modern app profile”中不存在。让我们尝试了解原因。

因此，当处理不是致命的线程中止类型时，即我们仍然可以对其进行处理（即第二、第三和第四种情况），虚拟机在出现此类异常时会遍历所有异常处理程序，并按照通常的方式查找那些异常类型与被抛出的异常类型相匹配或更基本的异常类型。在我们的情况下，有三种类型：`ThreadAbortException`、`Exception`和`object`（记住Exception本质上是数据存储库，异常类型可以是任何类型，甚至是`int`）。通过执行所有适当的`catch`块，虚拟机将ThreadAbortException传递到异常处理链中，并同时进入所有的`finally`块。总的来说，下面描述的两个示例中的情况是完全相同的：

```csharp
var thread = new Thread(() =>
{
    try {
        // ...
    } catch (Exception ex)
    {
        // ...
    }
});
thread.Start();
//...
thread.Abort();

var thread = new Thread(() =>
{
    try {
        // ...
    } catch (Exception ex)
    {
        // ...
        if(ex is ThreadAbortException)
        {
            throw;
        }
    }
});
thread.Start();
//...
thread.Abort();
```

当然，总会有一种情况，即 ThreadAbort 的发生是可以被我们完全预料到的。这时可能会有一种合理的愿望去处理它。正是为了这种情况，才开发并公开了 `Thread.ResetAbort()` 方法，它正是我们需要的：停止异常在所有处理程序链中的传递，使其被处理：

```csharp
void Main()
{
    var barrier = new Barrier(2);

    var thread = new Thread(() =>
    {
        try {
            barrier.SignalAndWait();  // Breakpoint #1
            Thread.Sleep(TimeSpan.FromSeconds(30));
        }
        catch (ThreadAbortException exception)
        {
            "Resetting abort".Dump();
            Thread.ResetAbort();
        }

        "Caught successfully".Dump();
        barrier.SignalAndWait();     // Breakpoint #2
    });

    thread.Start();
    barrier.SignalAndWait();         // Breakpoint #1

    thread.Abort();
    barrier.SignalAndWait();         // Breakpoint #2
}

Output:
Resetting abort
Catched successfully
```

然而，真的值得使用这个吗？值得责怪CoreCLR的开发人员吗，因为他们干脆删除了这段代码？想象一下，您是代码的用户，您认为代码“卡住”了，您产生了一种无法抑制的欲望来调用`ThreadAbortException`。当您想要终止线程的生命时，您只希望它真正完成其工作。更糟糕的是，罕见的算法只是终止线程并将其丢弃，然后继续他们的工作。通常外部算法决定等待操作的正确完成。或者相反：它可能决定线程将不再执行任何操作，递减某些内部计数器，并且不再依赖于任何多线程处理任何代码。总的来说，这并不是更糟糕的选择。我甚至要告诉您：作为一名程序员工作多年，我仍然无法为您提供一种完美的调用和处理方法。您自己判断：您不是立即抛出`ThreadAbort`，而是在意识到情况无法挽回后的一段时间后。也就是说，您可能会命中`ThreadAbortException`处理程序，也可能会错过它：“卡住的代码”可能根本没有卡住，而只是在长时间运行。正好在您想要终止它的时候，它可能会从等待中挣脱并正确继续工作。也就是说，没有额外的修辞，退出`try-catch(ThreadAbortException) { Thread.ResetAbort(); }`块。我们会得到什么？一个无辜的被终止的线程。保洁工走过来，拔掉了电线，网络断开了。方法等待超时，保洁工插回电线，一切都恢复正常，但您的控制代码却没有等待，却终止了线程。好吗？不好。有什么办法可以防范？没有。但让我们回到强调合法性`Thread.Abort()`的固执想法：我们用锤子砸向线程，并期望它以100%的概率被终止，但这可能不会发生。首先，不清楚如何在这种情况下终止它。因为情况可能会更加复杂：在卡住的线程中可能存在捕获`ThreadAbortException`、通过`ResetAbort`停止它，但由于破损的逻辑而继续卡住的逻辑。那么怎么办？无条件地执行`thread.Interrupt()`？这似乎是试图用粗暴的方法绕过程序逻辑中的错误。此外，我可以向您保证，您将遇到泄漏问题：`thread.Interrupt()`不会调用`catch`和`finally`，这意味着即使您有经验和技巧来清理资源，您也无法做到：您的线程将简单消失，而在相邻线程中，您可能不知道哪些资源被垂死线程占用。另外，请注意，如果`ThreadAbortException`错过了`catch(ThreadAbortException) { Thread.ResetAbort(); }`，资源也会泄漏。

在您刚刚阅读的内容之后，我希望您现在可能感到有些困惑，并希望重新阅读这一段。这是完全正确的想法：这将证明使用 `Thread.Abort()` 是不可取的。同样也不应该使用 `thread.Interrupt();`。这两种方法都会导致您的应用程序出现不受控制的行为。从本质上讲，它们违反了完整性原则：这是.NET Framework的基本原则。

然而，要理解这个方法被引入到实际应用中的目的，只需查看.NET Framework的源代码，并找到使用 `Thread.ResetAbort()` 的地方。因为实际上正是它的存在合法化了 `thread.Abort()`。

**ISAPIRuntime类** [ISAPIRuntime.cs](https://referencesource.microsoft.com/#System.Web/Hosting/ISAPIRuntime.cs,192)

```csharp
try {

    // ...

}
catch(Exception e) {
    try {
        WebBaseEvent.RaiseRuntimeError(e, this);
    } catch {}

    // 我们是否调用了HSE_REQ_DONE_WITH_SESSION？如果是，则不要重新抛出异常。
    if (wr != null && wr.Ecb == IntPtr.Zero) {
        if (pHttpCompletion != IntPtr.Zero) {
            UnsafeNativeMethods.SetDoneWithSessionCalled(pHttpCompletion);
        }
        // 如果这是一个线程中止异常，取消中止
        if (e is ThreadAbortException) {
            Thread.ResetAbort();
        }
        // 重要提示：如果此线程因为AppDomain.Unload而被中止，
        // CLR仍会抛出AppDomainUnloadedException。本机调用者必须特别处理COR_E_APPDOMAINUNLOADED(0x80131014)，
        // 并且不要多次调用HSE_REQ_DONE_WITH_SESSION。
        return 0;
    }

    // 如果我们没有调用HSE_REQ_DONE_WITH_SESSION，则重新抛出异常
    throw;
}
```

在这个例子中，调用了一些外部代码，如果它以不正确的方式结束，即抛出 `ThreadAbortException`，则在特定条件下将标记线程为不可中断。实质上是在处理 `ThreadAbort`。为什么在这种特定情况下我们中止 `Thread.Abort`？因为在这种情况下，我们正在处理服务器端代码，而无论我们的错误如何，服务器都应该向调用方返回正确的错误代码。中止线程会导致服务器无法向用户返回所需的错误代码，这是完全不正确的。此外，在 `AppDomin.Unload()` 期间留下了关于 `Thread.Abort()` 的注释，这对于 ThreadAbort 来说是一个极端情况，因为这样的进程无法停止，即使您执行 `Thread.ResetAbort` 也是如此。这将停止自身的中止，但不会停止线程与其所在的域的卸载：线程无法执行加载在已卸载的域中的代码指令。

**HttpContext 类** [HttpContext.cs](https://referencesource.microsoft.com/#System.Web/HttpContext.cs,1864)

```csharp
internal void InvokeCancellableCallback(WaitCallback callback, Object state) {
    // ...
 
    try {
        BeginCancellablePeriod();  // 从这一点开始可以取消请求
        try {
            callback(state);
        }
        finally {
            EndCancellablePeriod();  // 直到这一点可以取消请求
        }
        WaitForExceptionIfCancelled();  // 在 finally 之外等待
    }
    catch (ThreadAbortException e) {
        if (e.ExceptionState != null &&
            e.ExceptionState is HttpApplication.CancelModuleException &&
            ((HttpApplication.CancelModuleException)e.ExceptionState).Timeout) {

            Thread.ResetAbort();
            PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_TIMED_OUT);

            throw new HttpException(SR.GetString(SR.Request_timed_out),
                                null, WebEventCodes.RuntimeErrorRequestAbort);
        }
    }
}
```

这里展示了一个很好的例子，从不可控的异步异常 `ThreadAbortException` 转变为可控的带有性能计数器日志记录的 `HttpException`。

**HttpApplication 类** [HttpApplication.cs](https://referencesource.microsoft.com/#System.Web/HttpApplication.cs,2270)

```csharp
 internal Exception ExecuteStep(IExecutionStep step, ref bool completedSynchronously) 
 {
    Exception error = null;

    try {
        try {

        // ...

        }
        catch (Exception e) {
            error = e;

            // ...

            // 这可能会自动引发 ThreadAbortException，因为我们消耗了一个隐藏在其后的 ThreadAbortException 的异常

```

这段代码描述了一个非常有趣的情况：当我们等待的不是真正的 `ThreadAbort` 时（我有点为 CLR 和 .NET Framework 团队感到惋惜，想想他们不得不处理多少非标准情况，令人不寒而栗）。处理这种情况分为两个阶段：在内部处理程序中，我们捕获 `ThreadAbortException`，但同时检查我们的线程是否被标记为中止请求。如果线程没有被标记为中止请求，那么实际上这不是真正的 `ThreadAbortException。我们必须以适当的方式处理这种情况：安静地捕获异常并处理它。如果我们收到真正的 ThreadAbort，它将进入外部的 `catch`，因为 `ThreadAbortException` 必须进入所有适当的处理程序。如果它满足必要条件，它也将通过清除 `ThreadState.AbortRequested` 标志的 `Thread.ResetAbort()` 方法来处理。

谈到调用 `Thread.Abort()` 的示例，.NET Framework 中的所有代码示例都可以重写而不使用它。为了说明这一点，我只引用一个示例：

**QueuePathDialog 类** [QueuePathDialog.cs](https://referencesource.microsoft.com/#System.Messaging/System/Messaging/Design/QueuePathDialog.cs,364)

```csharp
protected override void OnHandleCreated(EventArgs e)
{
    if (!populateThreadRan)
    {
        populateThreadRan = true;
        populateThread = new Thread(new ThreadStart(this.PopulateThread));
        populateThread.Start();
    }

    base.OnHandleCreated(e);
}

protected override void OnFormClosing(FormClosingEventArgs e)
{
    this.closed = true;

    if (populateThread != null)
    {
        populateThread.Abort();
    }

    base.OnFormClosing(e);
}

private void PopulateThread()
{
    try
    {
        IEnumerator messageQueues = MessageQueue.GetMessageQueueEnumerator();
        bool locate = true;
        while (locate)
        {
            // ...
            this.BeginInvoke(new FinishPopulateDelegate(this.OnPopulateTreeview), new object[] { queues });
        }
    }
    catch
    {
        if (!this.closed)
            this.BeginInvoke(new ShowErrorDelegate(this.OnShowError), null);
    }

    if (!this.closed)
        this.BeginInvoke(new SelectQueueDelegate(this.OnSelectQueue), new object[] { this.selectedQueue, 0 });
}
```

##### 在 AppDomain.Unload 期间的 TheradAbortException

让我们尝试在加载了代码的AppDomain中卸载AppDomain。为此，我们人为地创造了一个不太正常但从代码执行的角度来看相当有趣的情况。在这个示例中，我们有两个线程：一个用于在其中引发ThreadAbortException，另一个是主线程。在主线程中，我们创建一个新的域，并在其中启动一个新线程。这个线程的任务是返回到主域，以便子域的方法只留在堆栈跟踪中。之后，主域卸载子域：

发生了非常有趣的事情。除了卸载域本身之外，域的卸载代码还会查找在该域中调用的方法，这些方法尚未完成工作，包括调用方法的堆栈深度，并在这些线程中引发ThreadAbortException。这一点很重要，尽管不太明显：如果域已被卸载，就无法返回到调用主域方法的方法，但该方法位于即将卸载的域中。换句话说，AppDomain.Unload可能会抛出正在执行其他域代码的线程。在这种情况下，无法取消`Thread.Abort`：您无法执行已卸载域的代码，因此`Thread.Abort`将完成其工作，即使您调用`Thread.ResetAbort`也无济于事。

##### ThreadAbortException的结论

- 这是一个异步异常，这意味着它可能会在您的代码的任何地方发生（但值得注意的是，需要努力才能发生）；
- 通常代码只处理预期的错误：如文件访问受限、字符串解析错误等。异步异常（在代码的任何地方发生）的存在会导致try-catch可能无法处理：您无法在应用程序的任何地方准备好处理ThreadAbort。这意味着无论如何这个异常都会导致泄漏；
- 线程中断也可能是由于某个域的卸载而发生。如果线程的堆栈跟踪中存在来自正在卸载的域的方法调用，线程将收到ThreadAbortException，无法ResetAbort；
- 通常情况下，不应该出现需要调用Thread.Abort()的情况，因为结果几乎总是不可预测的。
- CoreCLR不再包含手动调用Thread.Abort()：它已从类中删除。但这并不意味着无法获取它。

#### ExecutionEngineException

在此异常的注释中，有一个带有“Obsolete”属性的注释：

> 此类型先前表示运行时中的未指定致命错误。运行时不再引发此异常，因此此类型已过时

实际上，这是不正确的。也许注释作者非常希望这种情况终有一天成为事实，但要证明这不是真的，只需回顾一下在`FirstChanceException`中的异常示例：

```csharp
void Main()
{
    var counter = 0;

    AppDomain.CurrentDomain.FirstChanceException += (_, args) => {
        Console.WriteLine(args.Exception.Message);
        if(++counter == 1) {
            throw new ArgumentOutOfRangeException();
        }
    };

    throw new Exception("Hello!");
}
```

此代码的结果将是`ExecutionEngineException`，尽管我预期的未处理异常应该是`ArgumentOutOfRangeException`，即`throw new Exception("Hello!")`。也许这对核心开发人员来说似乎很奇怪，他们可能认为抛出`ExecutionEngineException`更为合适。

获取`ExecutionEngineException`的另一种相当简单的方法是在不安全的环境中不正确地配置封送。如果您在类型大小方面出错，传递了过多的数据，可能会破坏例如线程堆栈等内容，那么就会出现`ExecutionEngineException`。这将是一个合乎逻辑的、正确的结果：因为在这种情况下，CLR进入了一个它发现不一致的状态。不清楚如何恢复它。结果就是`ExecutionEngineException`。

另外，让我们谈谈如何诊断`ExecutionEngineException`。它发生的原因是什么？如果异常突然出现在您的代码中，您需要回答几个问题：

- 您的应用程序是否使用了不安全的库？是您自己使用还是第三方库。首先尝试弄清楚应用程序在哪里遇到了这个错误。如果代码进入了不安全的环境并在那里收到`ExecutionEngineException`，那么就需要仔细检查方法签名的一致性：在我们的代码和导入的代码中。请记住，如果导入的模块是用Delphi等Pascal语言的变体编写的，则参数应该以相反的顺序传递（在`DllImport`中进行设置：`CallingConvention.StdCall`）；
- 您是否订阅了FirstChanceException？也许它的代码引发了异常。在这种情况下，只需将处理程序包装在`try-catch(Exception)`中，并确保将发生的情况记录在错误日志中；
- 您的应用程序可能部分在一个平台上编译，部分在另一个平台上编译。尝试清除nuget包缓存，从头开始完全重新构建应用程序：手动清理obj/bin文件夹；
- 有时问题可能出现在框架本身。例如，在早期版本的.NET Framework 4.0中。在这种情况下，应该在调用错误的代码段上进行测试--使用更新版本的框架；

总的来说，不必害怕这个异常：它出现得相当少，以至于您可以在下一次与它愉快地相遇之前将其忘记。

#### 空引用异常

> 待办事项

#### 安全异常

> 待办事项

#### 内存不足异常

> 待办事项

### 损坏状态异常

在平台的建立和推广之后，随着大量程序员开始从C/C++和MFC（Microsoft Foundation Classes）迁移到更友好的开发环境，除了.NET Framework之外，Qt、Java和C++ Builder等环境也开始受到青睐。我们得到了一个指向应用程序代码执行虚拟化的方向。随着时间的推移，精心设计的.NET Framework开始占据自己的市场份额。随着岁月的积淀和发展，我们可以开始从力量的角度思考问题，而不是从适应性的角度。以前，我们大多数情况下不得不处理基于COM/ATL/ActiveX编写的庞大组件集（你还记得在Borland C++ Builder中将COM/ActiveX组件图标拖放到窗体上吗？），现在我们都过上了更轻松的生活。因为现在相关技术已经足够稀缺，我们不必担心，也有可能让它们变得不那么方便，以至于人们开始完全放弃它们，转而使用现代且优雅的.NET Framework。那些旧技术，不仅存在于5-10年前，实际上至今仍然存在并健在，对我们来说似乎有些过时、被遗忘、"不完美"和过时。因此，我们可以迈出更进一步的一步，关闭沙盒：使其更加难以渗透，使其更易于管理。

其中一个步骤是引入“Corrupted State Exceptions”概念，实质上是将一系列异常情况排除在法外。让我们来看看这些异常情况是什么，再次以`AccessViolationException`为例来追溯其中的一个：

**util.cpp文件** [util.cpp](https://github.com/dotnet/coreclr/blob/479b1e654cd5a13bb1ce47288cf78776b858bced/src/utilcode/util.cpp#L3163-L3197)

```cpp
BOOL IsProcessCorruptedStateException(DWORD dwExceptionCode, BOOL fCheckForSO /*=TRUE*/)
{
    // ...

    // 如果我们被要求在CSE检查中不包括SO，并且代码代表SO，则立即退出。
    if ((fCheckForSO == FALSE) && (dwExceptionCode == STATUS_STACK_OVERFLOW))
    {
        return fIsCorruptedStateException;
    }

    switch(dwExceptionCode)
    {
        case STATUS_ACCESS_VIOLATION:
        case STATUS_STACK_OVERFLOW:
        case EXCEPTION_ILLEGAL_INSTRUCTION:
        case EXCEPTION_IN_PAGE_ERROR:
        case EXCEPTION_INVALID_DISPOSITION:
        case EXCEPTION_NONCONTINUABLE_EXCEPTION:
        case EXCEPTION_PRIV_INSTRUCTION:
        case STATUS_UNWIND_CONSOLIDATE:
            fIsCorruptedStateException = TRUE;
            break;
    }

    return fIsCorruptedStateException;
}
```

让我们来看看我们特殊情况的描述：

| 错误代码                           | 描述                                                                                           |
|------------------------------------|------------------------------------------------------------------------------------------------|
| STATUS_ACCESS_VIOLATION            | 经常出现的错误，尝试访问没有权限的内存区域。从进程的角度来看，内存是线性的，但并不是所有范围都可以操作：只能操作那些被操作系统分配的“岛屿”，以及有权限的部分（例如，有些范围只有操作系统拥有，或者只能执行代码而不能读取数据） |
| STATUS_STACK_OVERFLOW              | 众所周知的错误：线程栈在调用下一个方法时内存不足                                           |
| EXCEPTION_ILLEGAL_INSTRUCTION      | 处理器从方法体中读取的代码未被识别为指令                                                     |
| EXCEPTION_IN_PAGE_ERROR            | 线程尝试访问不存在的内存页                                                                   |
| EXCEPTION_INVALID_DISPOSITION      | 异常处理机制返回了错误的处理程序。在高级语言编写的程序中（例如C++），这种异常永远不应该发生 |
| EXCEPTION_NONCONTINUABLE_EXCEPTION | 线程尝试在发生异常后继续执行程序，但无法继续执行导致错误的代码。这里指的不是catch/fault/finally块，而是类似异常过滤器，允许修复导致异常的错误并尝试再次执行导致错误的代码 |
| EXCEPTION_PRIV_INSTRUCTION         | 尝试执行处理器的特权指令                                                                     |
| STATUS_UNWIND_CONSOLIDATE          | 与堆栈展开相关的异常，不是我们讨论的重点                                                     |

请注意，实质上只有两种异常值得拦截：`STATUS_ACCESS_VIOLATION`和`STATUS_STACK_OVERFLOW`。其他异常即使在异常情况下也是异常的。它们更多地属于致命错误类别，我们不会考虑它们。因此，让我们更详细地讨论这两种异常：

#### AccessViolationException

获得这个异常是一个消息，任何人都不想收到的。当你遇到这种情况时，不清楚该怎么处理。`AccessViolationException`是在尝试读取或写入受保护内存区域时发生的异常，实质上是指应用程序试图访问未分配给其的内存区域或已释放的内存区域。这里的“保护”意味着尝试处理尚未分配的内存区域或已释放的内存区域。请注意，这里并不涉及垃圾回收器分配和释放内存的过程。垃圾回收器只是为自己和您的需求分配已经分配的内存。内存在某种程度上具有分层结构。当垃圾回收器的内存管理层之后是CLR核心库的内存分配管理层，再之后是操作系统，从可用的线性地址空间片段池中分配。因此，当应用程序超出其内存范围并尝试处理未分配的区域或不适用于应用程序的区域时，就会引发此异常。当出现这种情况时，您的分析选项并不多：

如果 StackTrace 进入 CLR 的深层，那么你的运气可能不太好：这很可能是核心错误。然而，这种情况几乎从不发生。解决方法之一是要么采取不同的行动，要么尽可能更新核心版本；

如果 StackTrace 进入某个库的不安全代码，那么有以下可能性：要么您在配置封送处理时出错了，要么在不安全库中存在严重错误。仔细检查方法的参数：可能本机方法的参数具有不同的位数或顺序，或者干脆大小不同。确保结构体按引用传递到正确的地方，按值传递到正确的地方。

目前，要捕获这种异常，必须向 JIT 编译器表明这确实是必要的。否则，它将不会被捕获，您将收到应用程序崩溃的结果。然而，当您确信能够正确处理它时，当然值得捕获它：其存在可能表明在调用点和引发 `AccessViolationException` 异常的点之间使用不安全方法分配了内存泄漏，这样虽然应用程序不会“崩溃”，但其工作可能变得不正确：因为一旦捕获到方法调用的故障，您很可能会尝试再次调用该方法。在这种情况下，任何事情都可能发生：您无法知道应用程序状态是如何在上一次被破坏的。然而，如果您仍然希望捕获这种异常，请查看在不同版本的 .NET Framework 中捕获此异常的可能性表：

.NET Framework 版本 | AccessViolationException
----------------------|-----------------------------------------------------------
1.0                   | NullReferenceException
2.0, 3.5              | 可以捕获 AccessViolation
4.0+                  | 可以捕获 AccessViolation，但需要配置
.NET Core             | 无法捕获 AccessViolation

换句话说，如果你遇到了一个非常古老的运行在.NET Framework 1.0上的应用程序，你将会得到一个NRE，这在某种程度上是一种欺骗：你传递了一个大于零的指针，却得到了NullReferenceException。然而，在我看来，这种行为是合理的：在托管代码的世界中，你最不希望做的就是去研究非托管代码的错误类型和NRE，它本质上就是.NET世界中的“坏对象指针”--这样说也不为过。然而，如果一切都如此简单就好了。在实际情况中，用户坚决缺乏这种类型的异常，因此--很快--在2.0版本中引入了它。在经历了几年的捕获模式后，异常不再是可捕获的，但是出现了一个特殊的设置，可以启用捕获。总的来说，CLR团队在每个阶段的选择看起来是合理的。您自己来判断：

1.0 在管理的世界中，分配内存的地方是`new`操作符。在非托管的世界中，任何代码段都可能成为出现此类错误的地方。虽然从哲学角度来看，NRE（空引用异常）和AVE（访问违例异常）的含义截然相反（NRE是使用未初始化的指针，AVE是使用不正确初始化的指针），但从.NET的角度来看，指针不可能不正确初始化。我们可以将这两种情况归结为同一种并赋予其哲学意义：指针设置不正确。因此，让我们这样做：在这两种情况下都抛出`NullReferenceException`。

2.0 在.NET Framework 的早期阶段发现，通过COM库继承的代码比自身的代码更多：存在大量商业组件的代码库，用于与网络、用户界面、数据库和其他子系统进行交互。因此，确实需要考虑是否应该获取`AccessViolationException`：错误诊断可能会使问题捕获过程变得更加昂贵。在.NET Framework 中引入了`AccessViolationException`异常。

4.0 .NET Framework 已经扎根，挤压传统的低级语言开发。COM组件的数量急剧减少：几乎所有主要任务都已在框架内解决，而使用不安全代码开始被视为奇怪和不正确的。在这种情况下，可以回到最初引入的理念：.NET 只适用于.NET。不安全代码不是规范，而是一种不得已的状态，因此捕获`AccessViolationException`与框架概念相悖——作为一个平台（即模拟具有自己规则的完整沙盒）。然而，我们仍然处于我们工作的平台的现实中，在许多情况下捕获这个异常仍然是必要的：引入专门的捕获模式：仅在输入相应配置时。

.NET Core 最终实现了CLR团队的梦想：.NET 不再假设与不安全代码的合法性，因此即使在配置级别，`AccessViolationException`的存在也已经不合法。.NET 已经发展到足以自行制定规则。现在，应用程序中存在这种异常将导致其崩溃，因此任何不安全代码（即CLR本身）都必须对这种异常是安全的。如果它出现在不安全库中，它们将无法正常工作，这意味着使用不安全语言的第三方组件开发人员将更加谨慎并在自己的代码中处理它。

通过一个例外情况的例子，我们可以追溯.NET Framework作为一个平台的发展历程：从对外部规则的不确定服从到由平台自身建立规则。

在讨论完所有内容后，我们需要揭示最后一个话题：如何在`4.0+`中启用对该异常的处理。因此，要在特定方法中启用对该类型异常的处理，需要：

- 在`configuration/runtime`部分添加以下代码：`<legacyCorruptedStateExceptionsPolicy enabled="true|false"/>`
- 对于每个需要处理`AccessViolationException`的方法，需要添加两个属性：`HandleProcessCorruptedStateExceptions`和`SecurityCritical`。这些属性允许为_特定_方法启用对损坏状态异常的处理，而不是全部方法。这种方法非常正确，因为您必须确切地知道自己想要处理它们，并且知道在哪里：有时更正确的选择是让应用程序崩溃。

为了演示如何启用`CSE`处理程序以及它们的基本处理，让我们看下面的代码示例：

```csharp
[HandleProcessCorruptedStateExceptions, SecurityCritical]
public bool TryCallNativeApi()
{
    try
    {
        // 调用可能引发AccessViolationException的方法
    }
    catch (Exception e)
    {
        // 记录日志，退出
        System.Console.WriteLine(e.Message);
        return false;
    }

  return true;
}
```

#### StackOverflowException

最后一个需要讨论的异常类型是堆栈溢出错误。当实际上分配给堆栈的数组耗尽内存时，就会发生这种情况。我们已经在相应章节中详细讨论了堆栈的结构（[线程堆栈](./ThreadStack.md)），在这里我们不深入讨论堆栈的结构，而是专注于错误本身。

当堆栈内存不足（或者已经达到下一个已被占用的内存区域，无法分配下一页虚拟内存），或者线程已经达到内存范围的限制时，就会尝试访问一个被称为Guard page的地址空间。实际上，这个范围是一个陷阱，不占用任何物理内存。在这种情况下，处理器不会进行实际的读取或写入操作，而是触发一个专门的中断，该中断需要向操作系统请求为堆栈增长分配新的内存区域。当达到最大允许值时，操作系统不会分配新的内存区域，而是生成一个带有`STATUS_STACK_OVERFLOW`代码的异常情况，该异常会通过.NET中的`Structured Exception Handling`机制传递，导致当前执行线程被视为不正确而终止。

请注意，虽然这种异常被称为`Corrupted State Exception`，但无法通过`HandleProcessCorruptedStateExceptions`来捕获它。也就是说，下面的代码不会起作用：

```csharp
// Main.cs
[HandleProcessCorruptedStateExceptions, SecurityCritical]
static void Main()
{
    try
    {
        Recursive();
    } catch (Exception exception)
    {
        Console.WriteLine("Catched Stack Overflow!");
    }
}

static void Recursive()
{
    Recursive();
}

// app.config:
<configuration>
  <runtime>
    <legacyCorruptedStateExceptionsPolicy enabled="true"/>
  </runtime>
</configuration>
```

这似乎是不可能的，因为堆栈溢出可能是由两种方式引起的：第一种是有意调用递归方法，但方法并不够谨慎以控制自身的深度。在这种情况下，可能会想要通过捕获异常来纠正情况。然而，如果仔细考虑，我们实际上是在合法化这种情况，允许它再次发生，这看起来更像是缺乏远见而不是关心的表现。第二种情况是偶然性的 - 在一个完全正常的调用中发生 StackOverflowException。只是在这个时刻，调用堆栈的深度变得过于关键。在这个例子中捕获异常看起来有点荒谬：应用程序正常运行，一切都很顺利，突然之间一个合法的方法调用在算法正常运行的情况下引发异常，并展开堆栈直到达到代码段预期的行为。嗯...再说一遍：我们期待在下一个代码段中什么都不会执行，因为堆栈内存已经用尽。在我看来，这完全是荒谬的。