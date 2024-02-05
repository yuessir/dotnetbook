# IDisposable模式

现在，可能几乎每一个在.NET平台上开发的程序员都会说，没有比这个模式更简单的了。这是一个众所周知的模式，是在该平台上应用最广泛的模式之一。然而，即使在最简单和最著名的问题领域中，总会有更深层次的东西，还有一些你从未探查过的隐藏角落。但是，无论是第一次接触这个主题的人，还是其他所有人（只是为了让你们每个人都回顾一下基础知识（不要跳过这些段落（我在监督！）））-- 我们将从头到尾详细描述一切。

## IDisposable

如果问你什么是IDisposable，你肯定会回答说，那就是

```csharp
public interface IDisposable
{
    void Dispose();
}
```

为什么要创建接口呢？如果我们有一个智能的垃圾收集器，它可以为我们清理所有内存，并且让我们根本不需要考虑如何清理内存，那么似乎就不太清楚为什么还要去关注这个问题。然而，这里有一些细节。存在一个误解，认为```IDisposable```是为了释放非托管资源而设计的。但这只是部分真相。要立刻理解这个观点是错误的，只需回想一下非托管资源的例子。```File```类算是非托管资源吗？不是。那么```DbContext```呢？同样不是。非托管资源是指不属于.NET类型系统的东西。即那些没有被平台创建，且位于其作用域之外的东西。一个简单的例子是操作系统中打开的文件的描述符。描述符是一个能够唯一标识操作系统打开的文件的数字。不是由你，而是由操作系统（你只是请求，而实际上是操作系统打开它）。即所有管理结构（如文件在文件系统上的坐标、在碎片化情况下的文件片段以及其他服务信息、磁盘的柱面号、磁头号、扇区号——在磁盘HDD的情况下）都不在.NET平台内部，而是在操作系统内部。唯一进入.NET平台的非托管资源是IntPtr——一个数字。这个数字反过来被FileSafeHandle包装，然后又被File类包装。即File类本身不是非托管资源，但它通过使用IntPtr这样一个额外的层来积累非托管资源——打开的文件的描述符。如何从这样的文件中读取数据呢？通过一系列WinAPI或Linux OS的方法。

第二个非托管资源的例子是多线程和多进程程序中的同步原语，如互斥锁、信号量。或者是通过P/Invoke传递的数据数组。

值得注意的是，操作系统（ОС）不仅仅是将未管理资源的描述符传递给应用程序，而且还会额外地将其保存在进程的打开描述符表中。同时保留了在应用程序结束运行时正确关闭这些资源的能力。也就是说，换句话说，无论如何，在退出应用程序时资源都将被关闭。然而，应用程序的运行时间可能不同，结果可能是资源被长时间锁定。

好的，我们已经弄清楚了未管理资源。那么，为什么在这些情况下需要IDisposable呢？原因在于，.NET Framework对发生在其之外的事情一无所知。如果您使用操作系统的函数打开一个文件，.NET不会知道这件事。如果您为自己的需要分配内存区域（例如，通过VirtualAlloc），.NET同样不会知道。如果它不知道，它就不会释放通过VirtualAlloc调用占用的内存。或者不会关闭直接通过操作系统API调用打开的文件。这可能会导致完全不同且不可预测的后果。如果您分配了太多内存而不释放它（例如，只是简单地将旧内存的指针置零），您可能会遇到内存不足的情况；或者，如果文件是通过操作系统工具打开但未关闭的，您可能会长时间锁定文件共享上的文件。文件共享的例子尤其说明了问题，因为即使在与服务器的连接关闭后，IIS侧的锁定仍然存在。您可能没有解除锁定的权限，需要向管理员请求进行`iisreset`或使用专门的软件手动关闭资源。因此，在远程服务器上解决这个问题可能不是一个简单的任务。

在所有这些情况下，都需要一个通用且可识别的_交互协议_，在类型系统和程序员之间建立明确的沟通，准确地识别出那些需要强制关闭的类型。这个_协议_就是IDisposable接口。其大致意思是：如果一个类型实现了IDisposable接口，那么当你完成对其实例的操作后，你必须调用Dispose()方法。

正因为此，存在两种标准的调用方式。通常，你要么是创建一个实体的实例，以便在一个方法中快速使用它，要么是在该实体实例的生命周期内使用它。

第一种方式是通过```using(...){ ... }```来包裹实例。也就是说，你直接指定在using块结束时对象应该被销毁。也就是说，应该调用Dispose()方法。第二种方式是在包含引用的对象的生命周期结束时销毁它。但是，在.NET中，除了终结器方法外，没有任何东西暗示对象会被自动销毁，对吗？但终结器对我们来说并不适用，因为它会在不确定的时间被调用。而我们需要在不再需要某个对象时立即释放它，例如，不再需要一个打开的文件。这正是为什么我们也需要实现IDisposable，并在Dispose方法中调用我们拥有的所有对象的Dispose方法，以便也释放它们。通过这种方式，我们遵守了_协议_，这非常重要。因为如果有人开始遵守某个协议，那么所有参与者都必须遵守：否则会出现问题。

## IDisposable实现的变体

让我们从简单到复杂看一下IDisposable的实现。

最简单也是最直接的实现方式，就是直接实现IDisposable：

```csharp
public class ResourceHolder : IDisposable
{
    DisposableResource _anotherResource = new DisposableResource();

    public void Dispose()
    {
        _anotherResource.Dispose();
    }
}
```

也就是说，首先我们创建了一个需要被释放的资源的实例：这个资源就是在Dispose()方法中被释放的。唯一缺失的，也是使得实现不一致的地方，是在通过Dispose()方法销毁类实例后，继续使用类实例的能力：

```csharp
public class ResourceHolder : IDisposable
{
    private DisposableResource _anotherResource = new DisposableResource();
    private bool _disposed;

    public void Dispose()
    {
        if(_disposed) return;

        _anotherResource.Dispose();
        _disposed = true;
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private void CheckDisposed()
    {
        if(_disposed) {
            throw new ObjectDisposedException();
        }
    }
}
```

需要在类的所有公共方法中首先调用CheckDisposed()。然而，如果是为了销毁一个受管理的资源，如`DisposableResource`，那么`ResourceHolder`类的结构看起来是正常的，但对于封装一个非受管理资源的情况则不是。

让我们想一个处理非受管理资源的方案。

```csharp
public class FileWrapper : IDisposable
{
    IntPtr _handle;

    public FileWrapper(string name)
    {
        _handle = CreateFile(name, 0, 0, 0, 0, 0, IntPtr.Zero);
    }

    public void Dispose()
    {
        CloseHandle(_handle);
    }

    [DllImport("kernel32.dll", EntryPoint = "CreateFile", SetLastError = true)]
    private static extern IntPtr CreateFile(String lpFileName,
        UInt32 dwDesiredAccess, UInt32 dwShareMode,
        IntPtr lpSecurityAttributes, UInt32 dwCreationDisposition,
        UInt32 dwFlagsAndAttributes,
        IntPtr hTemplateFile);

    [DllImport("kernel32.dll", SetLastError=true)]
    private static extern bool CloseHandle(IntPtr hObject);
}
```

那么，这两个最后的例子在行为上有什么区别呢？在第一个例子中，我们描述了一个受控资源与另一个受控资源的交互。这意味着，如果程序正确运行，资源将在任何情况下被释放。因为```DisposableResource```是受控的，这意味着.NET CLR非常清楚它的存在，如果出现不正确的行为，它会释放其占用的内存。注意，我故意不对```DisposableResource```类型封装的内容做任何假设。它可以包含任何逻辑和结构。它可能包含受控资源和非受控资源。*这不应该是我们关心的问题*。我们不需要每次都去反编译别人的库来查看它们使用了哪些类型的资源：受控还是非受控资源。但如果*我们的类型*使用了非受控资源，我们必须知道。这就是我们在```FileWrapper```类中所做的。那么，在这种情况下会发生什么呢？

如果我们使用非受控资源，那么我们又有两种情况：一种是一切正常，Dispose方法被调用（那么一切都好），另一种是出现了某些问题，Dispose方法无法执行。让我们先说明为什么后者可能发生：

- 如果我们使用`using(obj) { ... }`，那么在内部代码块中可能会出现异常，这个异常会被我们看不到的`finally`块捕获（这是C#的语法糖）。在这个块中，Dispose会被隐式调用。但是，有些情况下，Dispose不会被调用。例如，`StackOverflowException`，它既不会被`catch`捕获，也不会被`finally`捕获。这总是需要考虑的。因为如果你有一个线程进入递归，在某个点抛出`StackOverflowException`，那么那些被占用而未被释放的资源，将会被.NET运行时忘记。因为它不知道如何释放非托管资源：它们会在内存中悬挂，直到操作系统自己释放它们（例如，在你的程序退出时，有时甚至在应用程序结束运行后的不确定时间）。

- 如果我们从另一个Dispose()中调用Dispose()。那么可能会发生的情况是，我们再次无法到达它。这里的问题并不是应用程序作者的健忘：比如，忘记调用Dispose()。不是这样。问题再次出现在任何异常上。但现在我们讨论的不仅仅是那些导致应用程序线程崩溃的异常。现在我们讨论的是任何导致算法无法到达外部Dispose()调用的异常。

在所有这些情况下，都会出现悬而未决的非托管资源的情况。因为垃圾收集器不知道需要收集它们。它最多能做的是在下一次遍历时，发现包含我们的`FileWrapper`类型对象的对象图上的最后一个引用丢失了，内存将被那些仍然有引用的对象覆盖。

如何防御此类情况？在这些情况下，我们必须实现对象的终结器。终结器之所以被这样命名并非偶然。它绝不是析构函数，尽管由于C#中终结器的声明与C++中析构函数的声明相似，最初可能会给人这样的印象。与析构函数不同，终结器会*保证被调用*，而析构函数可能不会被调用（就像```Dispose()```一样）。终结器在启动垃圾收集（Garbage Collection, GC）时被调用（目前了解到这里就足够了，但实际上情况更复杂），旨在保证释放被占用的资源，如果*出了什么问题*。而对于释放非托管资源的情况，我们*必须*实现终结器。同样，我要重申，由于终结器是在GC启动时调用的，通常你根本不知道这会在何时发生。

让我们扩展我们的代码：

```csharp
public class FileWrapper : IDisposable
{
    IntPtr _handle;

    public FileWrapper(string name)
    {
        _handle = CreateFile(name, 0, 0, 0, 0, 0, IntPtr.Zero);
    }

    public void Dispose()
    {
        InternalDispose();
        GC.SuppressFinalize(this);
    }

    private void InternalDispose()
    {
        CloseHandle(_handle);
    }

    ~FileWrapper()
    {
        InternalDispose();
    }

    /// 其他方法
}
```

通过增加对终结过程的了解，我们加强了示例，从而保护了应用程序免受资源信息丢失的风险，如果出了问题且Dispose()没有被调用。此外，我们调用了GC.SuppressFinalize来禁用类型实例的终结化，如果已经调用了Dispose()。我们不需要两次释放同一个资源，对吧？还有另一个原因：我们通过减轻终结队列的负担，加速了与将来某个随机时间点并行运行的终结化代码的随机代码段。

现在让我们进一步加强我们的示例：

```csharp
public class FileWrapper : IDisposable
{
    IntPtr _handle;
    bool _disposed;

    public FileWrapper(string name)
    {
        _handle = CreateFile(name, 0, 0, 0, 0, 0, IntPtr.Zero);
    }

    public void Dispose()
    {
        if(_disposed) return;
        _disposed = true;

        InternalDispose();
        GC.SuppressFinalize(this);
    }


    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private void CheckDisposed()
    {
        if(_disposed) {
            throw new ObjectDisposedException();
        }
    }

    private void InternalDispose()
    {
        CloseHandle(_handle);
    }
```

```cpp
~FileWrapper()
{
    InternalDispose();
}

/// 其他方法
}
```

现在，我们对封装非托管资源的类型的实现示例看起来是完整的。不幸的是，重复调用```Dispose()```已经成为平台的事实标准，我们允许它被调用。我注意到，人们经常允许重复调用```Dispose()```是为了避免与调用代码相关的麻烦，这是不正确的。然而，使用您库的用户可能会基于MS的文档不这么认为，并允许多次调用Dispose()。无论如何，调用其他公共方法都会破坏对象的完整性。如果我们破坏了对象，这意味着我们不能再与它工作。这反过来意味着我们必须在每个公共方法的开始处插入```CheckDisposed```调用。

然而，这段代码存在一个非常严重的问题，这将阻止它按照我们的设想工作。如果我们回想一下垃圾收集过程是如何工作的，我们会注意到一个细节。在垃圾收集时，GC *首先*对所有直接继承自*Object*的内容进行终结，之后才处理实现了*CriticalFinalizerObject*的对象。但我们设计的两个类都继承自Object：这就是问题所在。我们不知道将以何种顺序进入“最后一程”。尽管如此，更高级别的对象可能会尝试在其终结器中与存储非托管资源的对象一起工作（尽管这听起来像是个坏主意）。在这里，我们真的需要一个终结顺序。为了设定这个顺序——我们必须让封装非托管资源的类型继承自`CriticalFinalizerObject`。

第二个原因有着更深层的根源。想象一下，你允许自己编写了一个不太关心内存使用的应用程序。它在没有缓存和其他技巧的情况下分配了大量内存。有一天，这样的应用程序会因为OutOfMemoryException而崩溃。当应用程序因为这个异常而崩溃时，会出现特殊的代码执行条件：它不能尝试分配任何东西。因为这会导致重复的异常，即使之前的异常被捕获了。这并不意味着我们不应该创建新的对象实例。普通的方法调用可能会导致这个异常。例如，调用终结方法。提醒一下，方法在第一次被调用时编译，这是正常的行为。那么如何避免这个问题呢？其实很简单。如果你将对象继承自*CriticalFinalizerObject*，那么这个类型的*所有*方法都会在类型加载到内存时立即编译。不仅如此，如果你用[PrePrepareMethod]属性标记方法，它们也会被预先编译，并且在资源不足时调用是安全的。

为什么这么重要？为什么要花费这么多努力在那些即将离我们而去的人身上？问题在于，不受控制的资源可能会在系统中悬挂很长时间。即使在你的应用程序结束运行之后。即使在计算机重启之后：如果用户在你的应用程序中打开了一个网络磁盘上的文件，那么它将被远程主机锁定，并且只有在超时或当你释放资源，关闭文件时才会被释放。如果你的应用程序在文件打开的时候崩溃，那么即使重启后文件也不会被关闭。你必须等待相当长的时间，以便远程主机释放它。此外，你不能在终结器中抛出异常——这会导致CLR迅速死亡并最终从应用程序中崩溃：终结器的调用不会被*try .. catch*包围。也就是说，在释放资源时，你必须确信它还可以被释放。最后一个同样有趣的事实是——如果CLR进行紧急卸载域，那么继承自*CriticalFinalizerObject*的类型的终结器也会被调用，与直接从*Object*继承的那些不同。

## SafeHandle / CriticalHandle / SafeBuffer / 衍生类

我有种感觉，我现在要为你们打开潘多拉的盒子。让我们来谈谈特殊类型：SafeHandle、CriticalHandle及其衍生类。但在此之前，让我们尝试列举一下通常来自unmanaged世界的所有东西：

- 第一个也是最常见的，通常从那里得到的是句柄（handles）。对于.NET开发者来说，这可能是一个完全陌生的词，但它是操作系统世界中非常重要的一个组成部分。本质上，句柄是一个32位或64位的数字，用于定义与操作系统的开放会话。也就是说，例如，你打开一个文件以便处理，然后从WinApi函数得到一个句柄。之后，使用这个句柄，你可以继续进行具体的操作：执行*Seek*、*Read*、*Write*操作。第二个例子：为了网络通信打开一个套接字。同样，操作系统会给你一个句柄。在.NET世界中，句柄存储在*IntPtr*类型中；
- 第二个是数据数组。有几种处理非托管数组的方法：要么通过unsafe代码（关键字unsafe）来处理，要么使用SafeBuffer，后者会用一个方便的.NET类包装数据缓冲区。值得注意的是，虽然第一种方法更快（例如，你可以大幅优化循环），但第二种方法要安全得多。因为它使用SafeHandle作为工作的基础；
- 字符串。处理字符串要简单一些，因为我们的任务是确定我们获取的字符串的格式和编码。然后字符串被复制到我们这里（string类是不可变的），我们就不需要再考虑其他事情了。
- 值类型（ValueTypes），这些通过复制获取，完全不需要考虑它们的命运。

SafeHandle是一个特殊的.NET CLR类，它继承了CriticalFinalizerObject，旨在尽可能安全和方便地封装操作系统的句柄。

```csharp

[SecurityCritical, SecurityPermission(SecurityAction.InheritanceDemand, UnmanagedCode=true)]
public abstract class SafeHandle : CriticalFinalizerObject, IDisposable
{
    protected IntPtr handle;        // 从操作系统得到的句柄
    private int _state;             // 状态（有效性，引用计数）
    private bool _ownsHandle;       // 是否可以释放句柄的标志。可能会出现我们封装了一个外部句柄并且没有权利释放它的情况
    private bool _fullyInitialized; // 实例是否已完全初始化

    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.MayFail)]
    protected SafeHandle(IntPtr invalidHandleValue, bool ownsHandle)
    {
    }

```

```plaintext
// 根据模板，析构函数调用 Dispose(false)
[SecuritySafeCritical]
~SafeHandle()
{
    Dispose(false);
}

// 设置 handle 可以手动进行，也可以通过 p/invoke Marshal 自动完成
[ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
protected void SetHandle(IntPtr handle)
{
    this.handle = handle;
}

// 此方法是为了能够直接使用 IntPtr。它用于判断是否成功创建了句柄，通过将其与之前定义的已知值进行比较。
// 注意，此方法有两个风险：
//  - 如果句柄通过 SetHandleAsInvalid 标记为无效，DangerousGetHandle 仍然会返回句柄的原始值。
//  - 返回的句柄可以在任何地方被重用。这至少意味着它可能会在没有反馈的情况下停止工作。在最坏的情况下，
//    如果直接将 IntPtr 传递到其他地方，它可能会进入不可靠的代码中，并通过在一个 IntPtr 上替换资源
//    成为攻击应用程序的攻击向量
[ResourceExposure(ResourceScope.None), ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
public IntPtr DangerousGetHandle()
{
    return handle;
}

// 资源已关闭（不再可用）
public bool IsClosed {
    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
    get { return (_state & 1) == 1; }
}

// 资源不可用。您可以通过改变逻辑来重写此属性。
public abstract bool IsInvalid {
    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
    get;
}

// 通过 Close() 模板关闭资源
[SecurityCritical, ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
public void Close() {
    Dispose(true);
}

// 通过 Dispose() 模板关闭资源
[SecuritySafeCritical, ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
public void Dispose() {
    Dispose(true);
}

[SecurityCritical, ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
protected virtual void Dispose(bool disposing)
{
    // ...
}

// 每当您认为 handle 不再有效时，您都应该调用此方法。
// 如果您不这样做，可能会导致资源泄露
[SecurityCritical, ResourceExposure(ResourceScope.None)]
[ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
[MethodImplAttribute(MethodImplOptions.InternalCall)]
public extern void SetHandleAsInvalid();
```

```chinese
// 重写此方法以指定释放资源的方式。编写代码时必须极其小心，因为不能调用未编译的方法、
// 创建新对象或抛出异常。返回值是释放资源操作成功的标志。
// 如果返回值为 false，则会抛出 SafeHandleCriticalFailure 异常，
// 如果启用了 SafeHandleCriticalFailure 的 Managed Debugger Assistant，
// 则会进入断点。
[ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
protected abstract bool ReleaseHandle();

// 关于引用计数器的操作。将在文本后面解释
[SecurityCritical, ResourceExposure(ResourceScope.None)]
[ReliabilityContract(Consistency.WillNotCorruptState, Cer.MayFail)]
[MethodImplAttribute(MethodImplOptions.InternalCall)]
public extern void DangerousAddRef(ref bool success);
public extern void DangerousRelease();
```

要评估从 SafeHandle 派生的类群的有用性，只需回想一下所有 .NET 类型的优点：自动垃圾收集。因此，通过封装非托管资源，SafeHandle 赋予它与托管资源相同的属性。此外，它还包含一个内部的外部引用计数器，这些引用计数器不能被 CLR 账户识别。即来自 unsafe 代码的引用。手动增加或减少计数器几乎没有必要：当您将任何从 SafeHandle 派生的类型声明为 unsafe 方法的参数时，进入方法时计数器将增加，而退出时将减少。之所以引入这个特性，是因为当您进入 unsafe 代码并将描述符传递给它时，如果您在另一个线程中（当然，如果您在多个线程中使用同一个描述符）将对它的引用置零，您将得到一个已收集的 SafeHandle。但是有了引用计数器，只要不将计数器归零，SafeHandle 就不会被收集。这就是为什么不建议手动更改计数器。或者，只有在这变得可能时才非常小心地返回它。
```

引用计数器的第二个用途是确定```CriticalFinalizerObject```的终结顺序，这些对象相互引用。如果一个基于SafeHandle的类型引用了另一个基于SafeHandle的类型，那么在引用者的构造函数中需要额外增加引用计数，在ReleaseHandle方法中则需要减少。这样，您的对象不会被销毁，直到您引用的对象被销毁。然而，为了避免混淆，最好避免这种情况。

让我们用我们对SafeHandlers的最新知识来编写我们类的最终版本：

```csharp
public class FileWrapper : IDisposable
{
    SafeFileHandle _handle;
    bool _disposed;

    public FileWrapper(string name)
    {
        _handle = CreateFile(name, 0, 0, 0, 0, 0, IntPtr.Zero);
    }

    public void Dispose()
    {
        if(_disposed) return;
        _disposed = true;
        _handle.Dispose();
    }


    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    private void CheckDisposed()
    {
        if(_disposed) {
            throw new ObjectDisposedException();
        }
    }

    [DllImport("kernel32.dll", EntryPoint = "CreateFile", SetLastError = true)]
    private static extern SafeFileHandle CreateFile(String lpFileName,
        UInt32 dwDesiredAccess, UInt32 dwShareMode,
        IntPtr lpSecurityAttributes, UInt32 dwCreationDisposition,
        UInt32 dwFlagsAndAttributes,
        IntPtr hTemplateFile);

    /// 其他方法
}
```

它有什么不同？知道如果在DllImport方法中将返回值设置为**任何**（包括自定义的）基于SafeHandle的类型，那么Marshal将会正确地创建并初始化它，将使用计数设置为1，我们将SafeFileHandle类型作为内核函数CreateFile的返回值。得到它后，我们在调用ReadFile和WriteFile时将使用它（因为调用时计数会增加，退出时会减少，这保证了在读写文件期间handle的存在）。这个类型设计得很正确，这意味着它将保证关闭文件描述符，即使进程意外终止。这意味着我们不需要实现自己的finalizer以及与之相关的所有内容。我们的类型大大简化了。

### 在实例方法运行期间触发finalizer

在垃圾收集过程中有一个优化，旨在尽早收集尽可能多的对象。让我们来看看下面的代码：

```csharp

```

```csharp
public void SampleMethod()
{
    var obj = new object();
    obj.ToString();
    
    // ...
    // 如果在这个点GC（垃圾回收）触发，obj有一定概率会被回收
    // 因为它不再被使用
    // ...
    
    Console.ReadLine();
}

// 从一方面来看，代码看起来足够安全，不会立即明白为什么这会关系到我们。但是，只需回想一下存在一些类，它们封装了非托管资源，就会立刻明白，如果类设计不正确，很可能会收到来自非托管世界的异常，告诉我们之前获得的句柄已经不再活跃：

void Main()
{
    var inst = new SampleClass();
    inst.ReadData(); 
    // 之后inst不再被使用
}

public sealed class SampleClass : CriticalFinalizerObject, IDisposable
{
    private IntPtr _handle;

    public SampleClass()
    {
        _handle = CreateFile("test.txt", 0, 0, IntPtr.Zero, 0, 0, IntPtr.Zero);
    }

    public void Dispose()
    {
        if (_handle != IntPtr.Zero)
        {
            CloseHandle(_handle);
            _handle = IntPtr.Zero;
        }
    }

    ~SampleClass()
    {
        Console.WriteLine("Finalizing instance.");
        Dispose();
    }

    public unsafe void ReadData()
    {
        Console.WriteLine("Calling GC.Collect...");
        
        // 我特意将其转换为局部变量
        // 以避免在GC.Collect()之后使用this;
        var handle = _handle;

        // 模拟完全的GC.Collect
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();

        Console.WriteLine("Finished doing something.");
        var overlapped = new NativeOverlapped();

        // 执行一些不重要的操作
        ReadFileEx(handle, new byte[] { }, 0, ref overlapped, (a, b, c) => {;});
    }

    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Auto, BestFitMapping = false)]
    static extern IntPtr CreateFile(String lpFileName, int dwDesiredAccess, int dwShareMode,
    IntPtr securityAttrs, int dwCreationDisposition, int dwFlagsAndAttributes, IntPtr hTemplateFile);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool ReadFileEx(IntPtr hFile, [Out] byte[] lpBuffer, uint nNumberOfBytesToRead,
    [In] ref NativeOverlapped lpOverlapped, IOCompletionCallback lpCompletionRoutine);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool CloseHandle(IntPtr hObject);
}
```

您会同意的：这段代码看起来还算体面。至少，它明显没有表明有任何问题。但实际上存在一个非常严重的问题。当类的终结器尝试关闭文件时，可能正处于从文件读取数据的过程中。这几乎肯定会导致错误。而且，由于在这种情况下错误会被返回（`IntPtr == -1`），我们不会看到它，`_handle`会被置零，后续的`Dispose`不会关闭文件，而我们会遇到资源泄露。为了解决这个问题，需要使用`SafeHandle`、`CriticalHandle`、`SafeBuffer`及其派生类，它们不仅在unmanaged世界中有使用计数器，而且这些计数器在传递给unmanaged方法时会自动增加，在退出时减少。

## 多线程

现在让我们谈谈薄冰。在之前关于IDisposable的讨论中，我们提到了一个非常重要的概念，这个概念不仅是设计Disposable类型的基础，也是设计任何类型的基础：对象的完整性概念。这意味着在任何时间点，对象都处于一个严格定义的状态，任何对它的操作都会将其状态转变为预先定义的某个状态——在设计该对象类型时。换句话说——任何对对象的操作都不应该有可能将其状态转变为未定义的状态。由此产生了之前设计的类型中的一个问题：它们不是线程安全的。存在在对象被销毁时调用这些类型的公共方法的潜在可能性。让我们解决这个问题，并决定是否真的需要解决它。

```csharp
public class FileWrapper : IDisposable
{
    IntPtr _handle;
    bool _disposed;
    object _disposingSync = new object();

    public FileWrapper(string name)
    {
        _handle = CreateFile(name, 0, 0, 0, 0, 0, IntPtr.Zero);
    }

    public void Seek(int position)
    {
        lock(_disposingSync)
        {
            CheckDisposed();
            // Seek API调用
        }
    }

    public void Dispose()
    {
        lock(_disposingSync)
        {
            if(_disposed) return;
            _disposed = true;
        }
        InternalDispose();
        GC.SuppressFinalize(this);
    }
```

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private void CheckDisposed()
{
    lock(_disposingSync)
    {
        if(_disposed) {
            throw new ObjectDisposedException();
        }
    }
}

private void InternalDispose()
{
    CloseHandle(_handle);
}

~FileWrapper()
{
    InternalDispose();
}

/// 其他方法
}

在Dispose()中对```_disposed```进行检查的代码上设置临界区，以及实际上对所有公共方法的代码设置临界区。这解决了我们的问题，即同时进入类型实例的公共方法和其销毁方法，但也为其他一些问题设置了一个延时炸弹：

  - 密集地使用类型实例的方法，以及创建和销毁对象的工作，将导致性能大幅下降。问题在于，获取锁需要一些时间。这个时间是必要的，用于分配SyncBlockIndex表，检查当前线程等等（我们将单独讨论这一切 - 在关于多线程的部分）。也就是说，为了对象“生命的最后一英里”，我们将在其整个生命周期内付出性能的代价！
  - 同步对象的额外内存流量
  - 在GC时遍历对象图的额外步骤

第二点，也是我认为最重要的。我们允许对象同时被销毁的情况发生，同时还可能再次使用它。我们应该期待什么呢？如果Dispose先执行，那么之后对对象方法的任何调用都必须导致```ObjectDisposedException```。因此，有一个简单的结论：必须将Dispose()调用与类型的其他公共方法之间的同步委托给服务方。也就是说，那些创建了```FileWrapper```类实例的代码。因为只有创建方知道它打算如何使用类实例以及何时销毁它。
```

从另一方面来说，根据对实现IDisposable接口的类的架构要求，Dispose调用应该只抛出关键错误（例如`OutOfMemoryException`，但不是IOException等）。这特别意味着，如果Dispose被多个线程同时调用，可能会出现两个线程同时销毁实体的情况（我们会跳过`if(_disposed) return;`的检查）。这取决于情况：如果资源释放*可以*多次进行，则不需要额外的检查。如果不是，就需要保护：

```csharp
// 我故意不展示整个模板，因为例子会很长
// 而且不会展示出本质
class Disposable : IDisposable
{
    private volatile int _disposed;

    public void Dispose()
    {
        if(Interlocked.CompareExchange(ref _disposed, 1, 0) == 0)
        {
            // dispose
        }
    }
}
```

## Disposable设计原则的两个层次

在.NET开发的书籍和互联网上，最流行的实现```IDisposable```接口的模式是什么？当你去面试一个潜在的新工作时，公司里的人期待你使用哪种模式？很可能是这个：

```csharp
public class Disposable : IDisposable
{
    bool _disposed;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if(disposing)
        {
            // 释放托管资源
        }
        // 释放非托管资源
    }

    protected void CheckDisposed()
    {
        if(_disposed)
        {
            throw new ObjectDisposedException();
        }
    }

    ~Disposable()
    {
        Dispose(false);
    }
}
```

这里有什么问题，为什么我们之前在这本书中从未这样写过？实际上，这个模板很好，简洁地涵盖了所有生活情境。但在我看来，普遍使用它并不是良好风格的规则：因为在实践中我们几乎从未见到真正的不可控资源，这种情况下半个模板的功能就白费了。不仅如此，它还违反了责任分离原则。因为它同时管理可控资源和不可控资源。在我谦虚的看法中，这完全不正确。让我们来看看一个稍微不同的方法。*Disposable 设计原则*。简而言之，其要点如下：

Disposing 分为两个级别的类：

  - Level 0 类型直接封装不可控资源
    - 它们要么是抽象的，要么是封装的
    - 所有方法必须标记为：
      - PrePrepareMethod，以便在类型加载时一同编译该方法
      - SecuritySafeCritical，以便对运行在限制模式下的代码的调用提供保护
      - ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success / MayFail)] 以便为方法及其所有子调用设置 CER
    - 可以引用 Level 0 类型，但必须增加引用计数，以确保正确退出“最后一程”
  - Level 1 类型只封装可控资源
    - 只能继承自 Level 1 类型或直接实现 IDisposable
    - 不得继承 Level 0 类型或 CriticalFinalizerObject
    - 可以封装 Level 1 或 Level 0 的可控类型
    - 通过销毁封装的对象来实现 IDisposable.Dispose，顺序为：先是 Level 0 类型，然后是 Level 1 类型
    - 由于它们没有不可控资源，因此不实现 finalizer
    - 必须包含一个 protected 属性，以访问 Level 0 类型。

正因为此，我从一开始就引入了两种类型的区分：包含可控资源的和包含不可控资源的。它们应该以完全不同的方式运作。

## Dispose 的其他用途

从理念上讲，IDisposable 被创建是为了释放非托管资源。但就像许多其他模式一样，它也被发现对其他任务非常有用。例如，用于释放对托管资源的引用。这听起来可能不是很有用：释放托管资源。毕竟，我们被告知，托管资源之所以被称为托管，是因为我们可以放松，微笑着看向 C/C++ 开发者。然而，情况并非完全如此。我们总是可能遇到这样的情况：我们丢失了对对象的引用，并认为一切都好：垃圾收集器（GC）会收集垃圾，包括我们的对象。但是，结果发现内存增长了，我们深入到内存分析程序中，发现实际上这个对象被其他东西持有。问题在于，无论是在 .NET 平台还是在外部类的架构中，都可能存在对您的实体进行隐式引用捕获的逻辑。之后，由于捕获的隐式性，程序员可能会错过释放它的必要性，最终导致内存泄漏。

### 委托，事件

让我们看一个合成的例子：

```csharp
class Secondary
{
    Action _action;

    void SaveForUseInFuture(Action action)
    {
        _action = action;
    }

    public void CallAction()
    {
        _action();
    }
}

class Primary
{
    Secondary _foo = new Secondary();

    public void PlanSayHello()
    {
        _foo.SaveForUseInFuture(Strategy);
    }

    public void SayHello()
    {
        _foo.CallAction();
    }

    void Strategy()
    {
        Console.WriteLine("Hello!");
    }
}
```

这里展示的问题是什么？`Secondary`类存储了一个类型为`Action`的委托在`_action`字段中，该委托通过`SaveForUseInFuture`方法接收。接下来，在`Primary`类中，`PlanSayHello`方法将`Strategy`方法的签名传递给`Secondary`。有趣的是，无论您传递的是静态方法还是实例方法，`SaveForUseInFuture`的调用方式都不会改变：只是*隐式地*传递或不传递`Primary`类实例的引用。也就是说，表面上看起来你只是指定了应该调用哪个方法。实际上，除了方法签名之外，委托还基于类实例的指针构建。调用方必须明白，它需要为哪个类实例调用`Strategy`方法！也就是说，`Secondary`类的实例隐式地保持了对`Primary`类实例的引用，尽管这并没有明确指出。对我们来说，这只意味着一件事：如果我们将`_foo`指针传递给其他地方，而且失去了对`Primary`的引用，那么GC**不会回收**`Primary`对象，因为它被`Secondary`保持着。如何避免这种不愉快的情况？需要一种确定性的方法来释放对我们的引用。这时，一个非常适合我们需求的机制来帮助我们：`IDisposable`

```csharp
// 为了简单，这里给出了简化版本的实现
class Secondary : IDisposable
{
    Action _action;

    public event Action<Secondary> OnDisposed;

    public void SaveForUseInFuture(Action action)
    {
        _action = action;
    }

    public void CallAction()
    {
        _action?.Invoke();
    }

    void Dispose()
    {
        _action = null;
        OnDisposed?.Invoke(this);
    }
}
```

现在这个例子看起来可以接受了：如果类的实例被传递给第三方，但在工作过程中丢失了对委托`_action`的引用，那么我们将其置空，第三方将被通知类实例的销毁，并将其引用删除，送往另一个世界。

基于委托工作的代码的第二个危险在于`event`的工作机制。让我们看看它们是如何展开的：

```csharp
 // 私有处理器字段
private Action<Secondary> _event;

// add/remove方法标记为[MethodImpl(MethodImplOptions.Synchronized)]，
// 这等同于lock(this)
public event Action<Secondary> OnDisposed {
    add { lock(this) { _event += value; } }
    remove { lock(this) { _event -= value; } }
}
```

在C#中，消息机制隐藏了事件（event）的内部结构，并保持所有订阅了更新的对象通过`event`连接。如果出现问题，对外部对象的引用将保留在`OnDisposed`中，并持续保持它。这就造成了一个奇怪的情况：从架构上我们有一个“事件源”的概念，按逻辑它不应该保持任何东西。但实际上，我们有了对订阅更新的对象的隐式保持。同时，我们无法在这个代理数组内部做任何改变：虽然这个实体是我们的一部分，但我们没有被赋予这样做的权利。我们唯一能做的就是完全擦除整个列表，通过将事件源设为null。第二种方法是显式实现`add`/`remove`方法，以引入对代理集合的控制。

> 顺便说一下，这里还有另一个隐含的情况：可能看起来，如果你将事件源设为null，那么后续对事件的订阅将导致`NullReferenceException`。在我谦虚的看法中，这将是更合逻辑的。然而，情况并非如此：如果外部代码在事件源被清除后订阅事件，FCL将创建一个新的Action类实例并将其放入`OnDisposed`。这种在C#语言中的不明确性可能会使程序员感到困惑：处理被置为null的字段应该引起你的不安，而不是平静。这里展示了一种过度放松可能导致程序员内存泄漏的方法。

### Lambda表达式和闭包

使用诸如lambda表达式这样的语法糖时，我们面临着特别的危险。

> 我想谈谈语法糖的问题。在我看来，应该非常谨慎地使用它，只有当你绝对确定它会导致什么结果时。lambda表达式的例子包括闭包，Expressions中的闭包，以及许多其他可能带来的问题。

嗯，老实对自己说：是的，我知道lambda表达式会创建闭包并带来资源泄露的风险。但它就是那么...简洁...令人愉悦...怎能抵挡住不用lambda而去单独定义一个方法的诱惑呢，尤其是当这个方法需要在使用点之外单独描述时？实际上，我们确实不应该轻易被这种诱惑所吸引，虽然并非每个人都能做到。

让我们来看一个例子：

```csharp
button.Clicked += () => service.SendMessageAsync(MessageType.Deploy);
```

你得承认，这行代码看起来非常安全。但它实际上隐藏了一个大问题：现在`button`变量隐式地引用了`service`并持有它。即使我们决定不再需要`service`，`button`也不会这么认为：只要按钮还活着，它就会持续保持对该引用。

解决的一个方法是使用`IDisposable`模式从任何`Action`创建（`System.Reactive.Disposables`）：

```csharp
// 从lambda创建一个委托
Action action = () => service.SendMessageAsync(MessageType.Deploy);

// 订阅
button.Clicked += action;

// 创建取消订阅
var subscription = Disposable.Create(() => button.Clicked -= action);

// 在需要取消订阅的地方
subscription.Dispose();
```

你会同意，这变得有些啰嗦，同时也失去了使用lambda表达式的全部意义。使用普通的私有方法在避免变量隐式捕获方面会简单得多，也更安全。

### 防止ThreadAbort异常

当开发给外部开发者使用的库时，您无法保证它在别人的应用程序中的表现。有时候，我们只能猜测是外部程序员对我们的库做了什么，导致了某种结果的出现。一个例子是在多线程环境中工作，此时资源清理的完整性问题可能变得非常紧迫。而且，尽管我们在编写`Dispose()`方法的代码时可以保证不会出现异常情况，但我们无法保证在`Dispose()`方法运行期间不会出现`ThreadAbortException`异常，该异常会终止我们的执行线程。这里需要记住的一点是，当抛出`ThreadAbortException`时，无论如何都会执行所有的catch/finally块（在catch/finally结束后ThreadAbort会被再次抛出）。因此，为了确保能够做到某事（通过Thread.Abort保证不间断），需要将关键部分包裹在`try { ... } finally { ... }`中。这样即使抛出ThreadAbort，代码也会被执行。

```csharp
void Dispose()
{
    if(_disposed) return;

    _someInstance.Unsubscribe(this);
    _disposed = true;
}
```

可以在任何点被`Thread.Abort`中断。这特别会导致对象部分被破坏，并允许其后续继续工作。而下面的代码：

```csharp
void Dispose()
{
    if(_disposed) return;

    // 保护免受ThreadAbortException
    try {}
    finally
    {
        _someInstance.Unsubscribe(this);
        _disposed = true;
    }
}
```

则免受此类中断的影响，并且即使在调用`Unsubscribe`方法的操作和执行其指令之间发生`Thread.Abort`，也能保证正确完成。

## 结论

### 实现的优点

因此，我们了解了很多关于这个最简单模式的新信息。让我们确定它的优点：

1. 模板的主要优点是能够确定性地释放资源：即在需要时进行。
2. 引入了一种众所周知的方法，以了解特定类型在使用结束时需要销毁其实例。
3. 如果模板实现得当，设计的类型的工作将从使用第三方组件的角度以及在进程崩溃时（例如，由于内存不足）卸载和销毁资源的角度变得安全。

### 实现的缺点

我认为模板的缺点远远多于优点：

1. 从一方面来看，任何实现这个模板的类型，都相当于向所有使用它的人发出了一个指令：使用我，你就接受了一个公开的提议。而且这种通知是如此的隐晦，以至于就像在公开提议的情况下，类型的用户并不总是知道类型具有这个接口。例如，不得不依赖IDE的提示（输入点，打出Dis...并检查过滤后的类成员列表中是否有这个方法）。如果发现了Dispose，就要在自己那里实现这个模板。有时候，这可能不会立即发生，那么就必须通过参与功能的类型系统来推进模板的实现。一个好例子是：你知道`IEnumerator<T>`会带来`IDisposable`吗？
2. 往往在设计某个接口时，就需要在类型的接口系统中插入IDisposable：当其中一个接口被迫继承IDisposable时。在我看来，这对我们设计的接口造成了“扭曲”。因为当设计接口时，你首先是在设计一种交互协议。那些可以通过接口做到的*与某物的交互*的动作集合。Dispose()方法——销毁类实例的方法。这与*交互协议*的本质相悖。这实质上是实现细节，却渗透到了接口中；
3. 尽管Dispose()是确定性的，但它并不意味着对象的直接销毁。对象在其*销毁*后仍将存在，只是处于另一种状态。为了使这成为现实，你必须在每个公共方法的开始调用CheckDisposed()。这看起来像是一个很好的补丁，伴随着“繁衍和增殖”的话语送给我们；
4. 还有一种小概率事件，即通过*显式*实现获得实现`IDisposable`的类型。或者获得一个实现了IDisposable但无法确定谁应该销毁它的类型：是发出它的一方，还是你自己。这产生了多次调用Dispose()的反模式，本质上，它允许解决已销毁对象的问题；
5. 完整实现很复杂。而且对于托管和非托管资源来说是不同的。在这方面，通过GC来简化开发人员生活的尝试看起来有些荒谬。当然，可以引入某种DisposableObject类型，它实现了整个模板，提供了`virtual void Dispose()`方法以供重写，但这并不能解决与模板相关的其他问题；
6. 实现`Dispose()`方法通常在文件的末尾，而`ctor`则声明在开头。在修改类并引入新资源时，很容易犯错，忘记为它们注册disposing。
7. 最后，使用在完全或部分实现了该模板的对象图上的模板，是在多线程环境中确定*销毁*顺序的一大难题。我主要是指Dispose()可能从图的不同端开始的情况。在这种情况下，最好使用其他模板。例如，Lifetime模板。
8. 平台开发者希望将内存管理自动化的愿望，加上现实：应用程序非常频繁地与非托管代码交互+需要控制对象引用的释放，以便垃圾收集器可以收集它们，这在理解“如何正确实现模板？是否真的有模板？”的问题上引入了大量的混乱。也许调用`delete obj; delete[] arr;`会更简单？

## 卸载域和退出应用程序

如果您已经阅读到这里，这意味着您至少对接下来的面试更加自信了。然而，与这个看似简单的模式相关的问题我们还没有讨论完。最后一个问题是：应用程序在简单的GC、在卸载域时的GC以及退出应用程序时的GC中的行为是否有所不同？只有在某种程度上，`Dispose()`过程才涉及到这个问题... 但是`Dispose()`和终结化（Finalization）是并行的，我们很少看到一个类的实现中有终结化但没有`Dispose()`方法。因此，让我们这样约定：我们将在专门讨论终结化的部分描述终结化本身，在这里我们只添加一些重要的点。

当应用程序域被卸载时，不仅会卸载在该域中加载的程序集，还会卸载在该域中创建的所有对象。这意味着，实际上，这些对象将被清理（GC收集），并且将调用它们的终结器。如果我们的终结器逻辑等待其他对象的终结以便以正确的顺序被销毁，那么可能需要注意`Environment.HasShutdownStarted`属性，该属性表示应用程序当前处于内存卸载状态，以及`AppDomain.CurrentDomain.IsFinalizingForUnload()`方法，它表明当前域正在卸载，这是终结化的原因。因为如果这些事件发生了，总体上我们不必在乎资源应该以何种顺序被终结。我们不能延迟域和应用程序的卸载：我们的任务是尽可能快地完成所有操作。

这个任务是在[LoaderAllocatorScout](http://referencesource.microsoft.com/#mscorlib/system/reflection/loaderallocator.cs,25551b0f6db5f579)类中解决的：

```csharp
// 程序集和LoaderAllocators将在AppDomain关闭时在非托管代码中被清理
// 因此，在appdomain关闭期间跳过重新注册和清理以进行终结是可以的。
// 我们还避免了由于AD卸载时对象位于DelayedFinalizationList内而导致LoaderAllocatorScout过早终结。
if (!Environment.HasShutdownStarted &&
    !AppDomain.CurrentDomain.IsFinalizingForUnload())
{
    // 如果托管的LoaderAllocator仍然存活，Destroy返回false。
    if (!Destroy(m_nativeLoaderAllocator))
    {
        // 可能有人通过弱引用持有我们的引用。
        // 我们将继续尝试。希望它最终会被释放。
        GC.ReRegisterForFinalize(this);
    }
}
```

## 典型的实现错误

正如我向您展示的，不存在一个通用、万能的IDisposable实现模板。不仅如此，对内存管理自动化的某种信心使人们混淆并在实现模板时做出复杂的决策。例如，整个.NET Framework就充满了其实现中的错误。为了不空谈，让我们通过.NET Framework的例子来看看这些错误。所有实现都可以通过以下链接找到：[IDisposable Usages](http://referencesource.microsoft.com/#mscorlib/system/idisposable.cs,1f55292c3174123d,references)

**FileEntry类** [cmsinterop.cs](http://referencesource.microsoft.com/#mscorlib/system/deployment/cmsinterop.cs,eeedb7095d7d3053,references)

> 这段代码显然是匆忙编写的，为了快速解决问题。作者显然想要做某事，但后来改变了主意，留下了一个错误的解决方案

```csharp
internal class FileEntry : IDisposable
{
    // 其他字段
    // ...
    [MarshalAs(UnmanagedType.SysInt)] public IntPtr HashValue;
    // ...

    ~FileEntry()
    {
        Dispose(false);
    }

    // 实现被隐藏，使得调用*正确*版本的方法变得困难
    void IDisposable.Dispose() { this.Dispose(true); }

    // 方法是公开的：这是一个严重的错误，允许不正确地销毁
    // 类的实例。而且，从外部这个方法不会被调用
    public void Dispose(bool fDisposing)
    {
        if (HashValue != IntPtr.Zero)
        {
            Marshal.FreeCoTaskMem(HashValue);
            HashValue = IntPtr.Zero;
        }

        if (fDisposing)
        {
            if( MuiMapping != null)
            {
                MuiMapping.Dispose(true);
                MuiMapping = null;
            }

            System.GC.SuppressFinalize(this);
        }
    }
}
```

**SemaphoreSlim类** [System/Threading/SemaphoreSlim.cs](https://github.com/dotnet/coreclr/blob/cbcdbd25e74ff9d963eafa202dd63504ca537f7e/src/mscorlib/src/System/Threading/SemaphoreSlim.cs)

> 这是.NET Framework关于IDisposable的顶级错误之一：对一个没有终结器的类调用SuppressFinalize。这种情况非常常见。

```csharp
public void Dispose()
{
    Dispose(true);

    // 类没有终结器 -- 完全没有必要调用GC.SuppressFinalize
    GC.SuppressFinalize(this);
}

// 实现模板意味着应该有一个终结器。但是没有。
// 完全可以只用一个public virtual void Dispose()
protected virtual void Dispose(bool disposing)
{
    if (disposing)
    {
        if (m_waitHandle != null)
        {
            m_waitHandle.Close();
            m_waitHandle = null;
        }
        m_lockObj = null;
        m_asyncHead = null;
        m_asyncTail = null;
    }
}
```

**调用Close+Dispose** [某个项目NativeWatcher的代码](https://github.com/alexguirre/NativeWatcher/blob/7208d463c41a709f29c60264bc518c6c0c5713cc/NativeWatcher/Forms/FormsManager.cs)

有时候人们会同时调用 Close 和 Dispose。但这并不正确（尽管不会引发错误，因为重复的 Dispose 不会导致异常）。问题在于，Close 是另一种模式，其引入是为了让人们更容易理解。但结果却变得更加难以理解。

```csharp
public void Dispose()
{
    if (MainForm != null)
    {
        MainForm.Close();
        MainForm.Dispose();
    }
    MainForm = null;
}
```

## 总结

1. IDisposable 是平台的标准，其实现的质量决定了整个应用程序的质量。更重要的是，在某些情况下，它还关系到您的应用程序的安全性，该应用程序可能会因为不受管理的资源而遭受攻击。
2. IDisposable 的实现应该尽可能高效。这特别适用于终结器部分，它与所有其他代码并行工作，给垃圾收集器带来负担。
3. 在实现 IDisposable 时，应避免将 Dispose() 调用与类的公共方法同步的想法。销毁不能与使用同时进行：在设计将使用 IDisposable 对象的类型时，这一点需要考虑。
4. 然而，应该防止两个线程同时调用 `Dispose()`：这是基于 Dispose() 不应该引发错误的声明。
5. 对不受管理的资源的封装应该与其他类型分开实现。也就是说，如果您封装了一个不受管理的资源，应该为此分配一个单独的类型：带有终结器，继承自 `SafeHandle / CriticalHandle / CriticalFinalizerObject`。这种责任分离将导致类型系统的改进支持和销毁类型实例的系统设计简化：使用类型不需要实现终结器。
6. 总的来说，这个模式既不方便使用，也不方便代码维护。可能应该转向通过 `Lifetime` 模式控制对象状态销毁过程的控制反转，下一部分将讨论这个话题。