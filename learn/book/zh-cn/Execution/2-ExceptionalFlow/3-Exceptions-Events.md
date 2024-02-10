## 关于异常情况的事件

> [讨论链接](https://github.com/sidristij/dotnetbook/issues/51)

通常情况下，我们并不总是知道在我们的程序中会发生哪些异常，因为我们几乎总是使用其他人编写并存在于其他子系统和库中的东西。除了您自己的代码中可能出现各种情况外，在其他库的代码中也可能存在许多与在隔离域中执行代码相关的问题。在这种情况下，学会获取有关隔离代码运行情况的数据将非常有用。毕竟，外部代码可能会捕获所有异常而不报告任何错误，通过使用 `fault` 块来掩盖它们：

```csharp
    try {
        // ...
    } catch {
        // 什么也不做，只是为了让代码调用更加安全
    }
```

在这种情况下，可能会发现代码的执行并不像看起来那么安全，但我们却没有任何关于发生问题的消息。另一种情况是应用程序抑制某些，即使是合法的，异常。结果是，在某个未来的随机时间点，下一个异常可能会由于一个看似错误的随机错误导致应用程序崩溃。在这种情况下，我们希望了解这个错误的历史。导致这种情况发生的事件。其中一种使这成为可能的方法是使用与异常情况相关的额外事件：`AppDomain.FirstChanceException` 和 `AppDomain.UnhandledException`。

实际上，当您“抛出异常”时，会调用某个内部子系统的常规 `Throw` 方法，该方法执行以下操作：

  - 调用 `AppDomain.FirstChanceException`
  - 在处理程序链中查找符合过滤条件的处理程序
  - 调用处理程序，先回滚到所需的帧
  - 如果未找到处理程序，则调用 `AppDomain.UnhandledException`，导致发生异常的线程崩溃。

首先要澄清一点，回答困扰许多人的问题：在执行在隔离域中的不受控制代码时，是否有可能取消发生的异常，而不会导致抛出异常的线程崩溃？简单明了的答案是：不可能。如果异常在调用方法的整个范围内没有被捕获，那么它原则上就无法被处理。否则就会出现一个奇怪的情况：如果我们通过 `AppDomain.FirstChanceException` 来处理异常（一种合成的 `catch`），那么线程的堆栈应该回滚到哪一帧呢？在.NET CLR的规则框架内如何设置？无法。这根本不可能。我们唯一能做的就是记录收到的信息以供将来研究之用。

其次，需要说明的是为什么这些事件是在 `AppDomain` 而不是 `Thread` 上引入的。毕竟，根据逻辑，异常发生在哪里？在执行命令的线程上。也就是说，实际上是在 `Thread` 上。那么为什么问题会出现在域上呢？答案很简单：`AppDomain.FirstChanceException` 和 `AppDomain.UnhandledException` 是为了哪些情况而创建的？除了其他用途之外，是为了为插件创建沙盒。也就是说，为了存在某种 `AppDomain`，该域被配置为 PartialTrust。在这个 `AppDomain` 内部可以发生任何事情：随时可能创建线程，或者使用来自线程池的现有线程。这样我们就无法在外部（我们没有编写那段代码）订阅内部线程的事件。仅仅因为我们不知道那里创建了什么线程。但我们肯定有一个 `AppDomain`，它组织了沙盒，我们有对它的引用。

因此，实际上我们只有两种极端情况：发生了意外事件（`FirstChanceExecption`），并且"一切糟糕"，没有人处理了异常情况：这种情况是未预料到的。因此，执行命令的线程是没有意义的，它（`Thread`）将被卸载。

当有这些事件数据时，我们能得到什么？为什么开发人员会绕过这些事件，这是不好的？

### AppDomain.FirstChanceException

这个事件本质上是一种纯粹的信息性事件，不能被“处理”。它的目的是通知您，在该域中发生了异常，并且在处理事件后，应用程序代码将开始处理它。它的执行具有一些需要在处理程序设计时记住的特点。

但让我们首先看一个简单的合成示例来处理它：

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

这段代码有什么特别之处？无论在哪里生成异常，第一件会发生的事情是将其记录到控制台日志中。换句话说，即使您忘记或无法处理某种类型的异常，它仍然会出现在您组织的事件日志中。第二个特点是内部异常抛出的一些奇怪条件。问题在于，在`FirstChanceException`处理程序内部，您不能简单地抛出另一个异常。更确切地说，在`FirstChanceException`处理程序内部，*您无法*抛出任何异常。如果您这样做了，可能会发生两种情况。首先，如果没有`if(++counter == 1)`条件，我们将不断地抛出新的`ArgumentOutOfRangeException`，导致无限的`FirstChanceException`抛出。这意味着在某个阶段我们将获得`StackOverflowException`：`throw new Exception("Hello!")`调用CLR方法`Throw`，该方法调用`FirstChanceException`，后者再调用`Throw`以抛出`ArgumentOutOfRangeException`，然后递归进行。第二种情况是：我们通过`counter`条件限制了递归深度。换句话说，在这种情况下，我们只抛出异常一次。结果非常出乎意料：我们将得到一个异常情况，实际上是在`Throw`指令内部处理的。那么对于这种类型的错误，什么更适合呢？根据ECMA-335，如果指令处于异常状态，则应该抛出`ExecutionEngineException`！我们无法处理这种异常情况。它将导致应用程序完全崩溃。那么我们有哪些安全处理选项呢？

首先想到的是在整个`FirstChanceException`处理程序代码上设置`try-catch`块：

```csharp
void Main()
{
    var fceStarted = false;
    var sync = new object();
    EventHandler<FirstChanceExceptionEventArgs> handler;
    handler = new EventHandler<FirstChanceExceptionEventArgs>((_, args) =>
    {
        lock (sync)
        {
            if (fceStarted)
            {
                // 这段代码实际上是一个占位符，旨在通知异常本质上并非在应用程序的主要代码中产生，而是在下面的try块中产生。
                Console.WriteLine($"FirstChanceException inside FirstChanceException ({args.Exception.GetType().FullName})");
                return;
            }
            fceStarted = true;

            try
            {
                // 不安全的记录日志到任何地方，比如数据库
                Console.WriteLine(args.Exception.Message);
                throw new ArgumentOutOfRangeException();
            }
            catch (Exception exception)
            {
                // 这种记录日志应该是最安全的
                Console.WriteLine("Success");
            }
            finally
            {
                fceStarted = false;
            }
        }
    });
    AppDomain.CurrentDomain.FirstChanceException += handler;

    try
    {
        throw new Exception("Hello!");
    } finally {
        AppDomain.CurrentDomain.FirstChanceException -= handler;
    }
}

输出：

Hello!
指定的参数超出有效值范围。
FirstChanceException inside FirstChanceException (System.ArgumentOutOfRangeException)
Success

!Exception: Hello!
```

换句话说，一方面我们有处理`FirstChanceException`事件的代码，另一方面是处理`FirstChanceException`本身的额外异常处理代码。然而，这两种情况下的日志记录方法应该有所不同。如果事件处理的日志记录可以随意进行，那么处理`FirstChanceException`的逻辑错误处理应该是无异常的。另一个你可能注意到的问题是多线程同步。这里可能会有一个问题：为什么要在这里同步，如果任何异常都是在某个线程中产生的，那么`FirstChanceException`理论上应该是线程安全的。然而，事实并非如此。`FirstChanceException`是在AppDomain中发生的。这意味着它对于在特定域中启动的任何线程都会发生。也就是说，如果我们有一个域，在该域中启动了多个线程，那么`FirstChanceException`可能会并行发生。这意味着我们需要通过同步来保护自己，例如使用`lock`。

第二种方法是尝试将处理移至属于另一个应用程序域的相邻线程。然而，值得注意的是，在这种实现中，我们应该建立一个专门的应用程序域来执行这个任务，以免其他线程（作为工作线程）可能会影响到这个应用程序域：

在这种情况下，处理 `FirstChanceException` 是非常安全的：在属于另一个应用程序域的相邻线程中进行处理。处理消息时的错误不会导致应用程序的工作线程崩溃。此外，您还可以单独监听日志消息记录域的 `UnhandledException`：日志记录中的致命错误不会导致整个应用程序崩溃。

我们可以拦截的第二条消息涉及处理异常情况的 `AppDomain.UnhandledException`。这对我们来说是一个非常糟糕的消息，因为它意味着在某个线程中没有找到任何人能够处理发生的错误。如果发生这种情况，我们唯一能做的就是“清理”这个错误的后果。也就是以某种方式清理属于该线程的资源，如果有的话。然而，更好的做法是在“根源”处处理异常，而不是让线程崩溃。实质上就是使用 `try-catch`。让我们尝试看看这种行为的合理性。

假设我们有一个库，需要创建线程并在这些线程中执行某些逻辑。作为这个库的用户，我们只关心 API 调用的保证以及错误消息的接收。如果库在不通知的情况下使线程崩溃，对我们几乎没有帮助。更糟糕的是，线程崩溃会导致 `AppDomain.UnhandledException` 消息，其中没有关于哪个具体线程失败的信息。如果是我们的代码导致线程崩溃，那对我们也没有什么用。至少我没有遇到这种情况的必要性。我们的任务是正确处理错误，将错误发生的信息记录到错误日志中，并正确地结束线程的工作。实质上，就是在启动线程的方法周围加上 `try-catch`：

```csharp
    ThreadPool.QueueUserWorkitem(_ => {
        using(Disposables aggregator = ...){
            try {
                // 在这里执行工作，另外：
                aggregator.Add(subscriptions);
                aggregator.Add(dependantResources);
            } catch (Exception ex)
            {
                logger.Error(ex, "未处理的异常");
            }
        }
    });
```

在这种方案中，我们将得到所需的结果：一方面，我们不会中断流程。另一方面，如果已经创建了本地资源，我们将正确地清理它们。此外，我们还将记录所收到的错误。但是，你可能会说。你们怎么突然就跳过了`AppDomain.UnhandledException`事件的问题呢？难道它完全不重要吗？是重要的。但只是为了提醒我们忘记用必要的逻辑将某些线程包装在`try-catch`中。这个逻辑包括日志记录和资源清理。否则，这将是完全不正确的：就好像根本没有异常一样，就随便处理和忽略所有异常。