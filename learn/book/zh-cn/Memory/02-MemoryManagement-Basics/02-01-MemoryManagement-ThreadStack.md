# 线程栈

有一个很少被谈论的内存区域，但它可能是应用程序中最重要的区域。它是最常用的、具有即时分配和释放内存的区域，被称为“线程栈”。由于指向它的指针实际上是编码在进程上下文中的寄存器中的，而这些寄存器是线程上下文的一部分，所以在执行任何线程内的代码时，线程栈是特定于该线程的。它的存在有什么意义呢？

现在，让我们来看一个简单的代码示例：

```csharp
void Method1()
{
    Method2(123);
}

void Method2(int arg)
{
    // ...
}
```

在这段代码中没有什么特别的事情发生，但我们不要忽视它，相反，让我们尽可能仔细地观察它。当任何`Method1`调用任何`Method2`时，任何这样的调用（不仅在.NET中，而且在其他平台上）都会执行以下操作：

  1. JIT编译的代码首先要做的是将方法参数保存到栈中（从第三个参数开始）。前两个参数通过寄存器传递。重要的是要记住，对于实例方法，第一个参数是指向方法所操作的对象的指针，即`this`指针。因此，在这些（几乎所有）情况下，寄存器只有一个参数，而其他所有参数都存储在栈中；
  2. 然后编译器插入`call`指令，该指令将返回地址（即`call`指令后面的地址）压入栈中。这样，任何方法都知道在哪里返回，以便调用代码可以继续执行；
  3. 在传递所有参数并调用方法之后，我们需要找到一种方法来恢复栈，以便在退出方法时可以恢复它，而无需担心计算我们在栈中占用的字节数。为此，我们保存EBP寄存器的值，该寄存器始终指向当前栈帧的开头（即存储特定调用方法的信息的区域）。通过在每次调用时保存该寄存器的值，我们实际上创建了一个单链表的栈帧。但请注意，实际上它们是紧密相连的，没有任何间隙。然而，为了简化栈帧的释放和应用程序的调试（调试器使用这些指针来显示堆栈跟踪），一个单链表被构建起来；
  4. 在调用方法时的最后一件事是为局部变量分配内存。由于编译器事先知道需要多少内存，它会立即进行分配，将栈顶指针（SP/ESP/RSP）移动相应的字节数；
  5. 最后，在第五个阶段，方法的代码和有用的操作被执行；
  6. 当退出方法时，栈顶指针从EBP中恢复，EBP是存储当前栈帧开头的位置；
  7. 接下来，通过`ret`指令从方法中退出。它从栈中取出之前由`call`指令留下的返回地址，并跳转到该地址。

同样的过程可以在下图中看到：

![](./imgs/ThreadStack/AnyMethodCall.png)

请注意，栈是“向上增长”的，从高地址开始，到低地址结束，即反向。

通过观察所有这些，不禁得出这样的结论：如果不是大多数，那么至少一半的处理器操作都是用于处理程序结构，而不是其有效负载。也就是说，处理方法调用、类型检查、泛型编译、在接口表中查找方法等等。特别是当我们想起大多数现代代码都是使用通过接口进行工作的方法，将其分解为许多小方法，每个方法都执行自己的操作时，这种基础设施代码的浪费可能会显现出来。而且，这些操作通常涉及基本类型和类型转换，有时转换为接口，有时转换为派生类。在所有这些输入条件下，基础设施代码的浪费是可以想象的。唯一我能告诉你的是：编译器，包括JIT编译器，具有许多技术可以生成更高效的代码。在可能的情况下，方法调用的代码被完全插入，而在可能的情况下，直接调用方法而不是在VSD接口表中查找方法。最令人沮丧的是，很难测量基础设施负载：需要JITter或其他编译器在基础设施代码工作之前和之后插入一些度量标准。即在调用方法之前，在方法内部（在初始化堆栈帧之后）之后。在退出方法之前，在退出方法之后。在编译之前，在编译之后。等等。然而，让我们不要谈论沮丧的事情，而是讨论一下我们可以如何利用这些信息。

## 关于x86平台上的异常的一些基本知识

如果我们深入方法的代码，我们会注意到另一个与线程栈相关的结构。看看这个例子：

```csharp
void Method1()
{
    try
    {
        Method2(123);
    } catch {
        // ...
    }
}

void Method2(int arg)
{
    Method3();
}

void Method3()
{
    try
    {
        //...
    } catch {
        //...
    }
}
```

如果在从`Method3`调用的任何方法中引发异常，控制权将返回到`Method3`的`catch`块。如果异常没有被处理，它将在`Method1`中开始处理。但是，如果什么都没有发生，`Method3`将完成其工作，控制权将转移到`Method2`，在那里也可能引发异常。但是由于自然原因，它将不会在`Method3`中处理，而是在`Method1`中处理。这种方便的自动化的问题在于，处理异常的数据结构也位于声明它们的方法的栈帧中。关于异常本身我们将在另外的地方讨论，这里只说一下，.NET Framework CLR和Core CLR的异常模型是不同的。CoreCLR必须在不同的平台上有所不同，因此它的异常模型通过PAL（Platform Adaption Layer）的适配器实现。对于大型的.NET Framework CLR来说，这是不必要的：它生活在Windows平台的生态系统中，这个生态系统已经有了多年的通用异常处理机制，称为SEH（Structured Exception Handling）。几乎所有的编程语言（在最终编译时）都使用它，因为它提供了在不同编程语言编写的模块之间无缝处理异常的能力。它的工作原理大致如下：

  1. 进入try块时，将一个结构压入栈中，该结构的第一个字段指向前一个异常处理块（例如，调用方法，该方法也有try-catch），块的类型，异常代码和处理程序的地址；
  1. 在线程的TEB（Thread Environment Block，实际上是线程上下文）中，将当前异常处理块链的地址更改为我们创建的地址。这样，我们将我们的块添加到链中；
  1. 当try块结束时，执行相反的操作：将旧的顶部写入TEB，从而从链中删除我们的处理程序；
  1. 如果发生异常，从TEB中取出顶部，并按顺序调用处理程序，检查它们是否适用于特定的异常。如果是，执行处理块（例如，catch）；
  1. 在TEB中恢复SEH结构的地址，该地址位于处理异常的方法之前的堆栈中。

正如你所看到的，这并不复杂。然而，所有这些信息也存储在栈中。

## 关于x64、AMD64平台的基本知识 [进行中]

待办事项

## 关于x64、AMD64平台上的异常的一些基本知识 [进行中]

待办事项

## 关于线程栈的一些缺陷

待办事项

让我们稍微考虑一下安全性和可能出现的问题，这些问题在理论上可能会出现。为此，让我们再次看一下线程堆栈的结构，它本质上是一个普通的数组。构建帧的内存范围是从末尾向开头增长的。也就是说，较新的帧位于较早的地址上。正如前面所说，帧通过单链表连接在一起。这是因为帧的大小不是固定的，任何调试器都应该能够“读取”它。在此过程中，处理器不会对帧进行区分：任何方法都可以自行读取整个内存区域。而且，考虑到我们处于虚拟内存中，该内存被分割为作为实际分配内存的部分，我们可以使用特殊的WinAPI函数根据堆栈上的任何地址获取分配内存的范围。然后，我们可以分析单链表-这是技术问题：

```csharp
    // 变量位于堆栈中
    int x;

    // 获取分配给堆栈的内存区域的信息
    MEMORY_BASIC_INFORMATION *stackData = new MEMORY_BASIC_INFORMATION();
    VirtualQuery((void *)&x, stackData, sizeof(MEMORY_BASIC_INFORMATION));
```

这使我们能够获取和修改作为调用我们的方法的本地变量的所有数据。如果应用程序没有配置一个沙盒，其中调用扩展应用程序功能的第三方库，则即使您提供给它的API不允许，该第三方库也可以获取数据。这种方法可能看起来有些牵强，但在C/C++世界中，没有像AppDomain这样美妙的东西，可以配置权限，所以这是从应用程序入侵中最常见的事情。此外，您可以通过反射查看所需的类型，复制其结构，并通过从堆栈上的引用进行跟踪，将VMT地址替换为我们的地址，从而将与特定实例的所有操作重定向到我们这里。顺便说一下，SEH也广泛用于应用程序入侵。通过更改异常处理程序的地址，您还可以使操作系统执行恶意代码。但是，从所有这些中得出的结论非常简单：当您想要使用扩展应用程序功能的库时，始终配置沙盒。当然，我指的是各种插件、附加组件和其他扩展。

## 大例子：在x86平台上克隆线程

为了记住我们已经详细阅读的所有内容，我们需要从多个角度来讨论某个主题的照明问题。看起来，我们可以为线程堆栈构建什么样的示例？调用另一个方法？魔术...当然不是：我们每天都在做这样的事情很多次。相反，我们将克隆执行线程。也就是说，我们将使在调用特定方法后，我们将拥有两个线程而不是一个：我们自己的线程和一个新的线程，但它将继续从克隆方法调用点执行代码，就好像它自己到达那里一样。它将如下所示：

```csharp
void MakeFork()
{
    // 为了确保一切都被克隆，我们使用局部变量：
    // 在新线程中，它们的值必须与父线程中的值相同
    var sameLocalVariable = 123;
    var sync = new object();

    // 计时
    var stopwatch = Stopwatch.StartNew();

    // 克隆线程
    var forked = Fork.CloneThread();

    // 从这一点开始，代码将由两个线程执行。
    // 对于子线程，forked = true，对于父线程，forked = false
    lock(sync)
    {
        Console.WriteLine("in {0} thread: {1}, local value: {2}, time to enter = {3} ms",
            forked ? "forked" : "parent",
            Thread.CurrentThread.ManagedThreadId,
            sameLocalVariable,
            stopwatch.ElapsedMilliseconds);
    }

    // 当退出方法时，父线程将返回到调用MakeFork()的方法中，
    // 继续工作，就像什么都没有发生一样，
    // 而子线程将终止执行。
}

// 示例输出：
// in forked thread: 2, local value: 123, time to enter = 2 ms
// in parent thread: 1, local value: 123, time to enter = 2 ms
```

可以说，这个概念很有趣。当然，对于这样的操作，我们可以争论很多关于是否合理的问题，但是此示例的目的是在理解这种数据结构的工作方式方面提供一个重要的观点。那么如何进行克隆？要回答这个问题，我们需要回答另一个问题：线程到底是由什么组成的？线程由以下结构和数据区域定义：

1. 一组处理器寄存器。所有寄存器都定义了执行指令的线程的状态：从当前执行指令的地址到线程堆栈和它操作的数据的地址；
2. [Thread Environment Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block)或TIB/TEB，它存储有关线程的系统信息，包括异常处理程序的地址；
3. 线程堆栈，其地址由寄存器SS:ESP定义；
4. 线程的平台上下文，其中包含线程的本地数据（从TIB引用）

当然，还有其他一些我们可能不知道的东西。但是我们不需要了解所有这些内容，因为对于示例来说，我们不需要将代码用于实际生产环境，而只需要它作为一个很好的示例来帮助我们理解这个主题。而且，为了使其基本工作，我们需要复制到新线程的寄存器集（修复SS:ESP，因为堆栈将是新的），并编辑堆栈本身，以确保它包含我们需要的内容。

所以，当我们的代码调用MakeFork时，它将调用CloneThread，它将进入未管理的CLI/C++世界，并调用克隆方法（实际实现）-在那里。让我们看一下伪代码：

```csharp
void RootMethod()
{
    MakeFork();
}
```

当MakeFork被调用时，从堆栈跟踪的角度来看，我们期望什么？我们期望在父线程中一切保持不变，而在子线程中，我们从线程池中获取一个线程（为了提高速度），在其中模拟调用MakeFork方法以及其局部变量，并且代码将继续执行，而不是从方法的开头开始，而是从CloneThread调用点之后的位置开始。换句话说，在我们的想象中，堆栈跟踪大致如下：

```csharp
// 父线程
RootMethod -> MakeFork

// 子线程
ThreadPool -> MakeFork
```

我们最初有什么？我们有我们的线程。我们还可以创建一个新线程或将任务计划到线程池中，在那里执行我们的代码。我们还明白，关于嵌套调用的信息存储在堆栈中，并且如果愿意，我们可以对其进行操作（例如，使用C++/CLI）。而且，如果我们遵循约定，并将返回地址放入堆栈顶部以供ret指令使用，将EBP寄存器的值放入其中，并为局部变量腾出空间（如果需要），那么我们可以模拟方法调用。在C#中，可以手动写入线程堆栈，但是我们需要寄存器以及非常小心地使用它们，因此我们无法避免进入C++。这就是CLI/C++在我生活中第一次（个人）发挥作用的地方，它允许编写混合代码：一部分指令是.NET的，一部分是C++的，有时甚至需要进入汇编级别。这正是我们需要的。

那么，当我们的代码调用MakeFork时，它将调用CloneThread，后者将进入未管理的CLI/C++世界，并调用克隆方法（实际实现）-在那里。让我们看一下堆栈如何在我们的代码调用MakeFork时看起来（当然，堆栈从高地址向低地址增长。从右到左）：

首先，我们的任务是模拟在新线程中启动`Fork.CloneThread()`方法。为此，我们需要在线程的堆栈末尾添加一系列帧：就好像从传递给线程池的委托中调用了`Fork.CloneThread()`，然后通过C++代码包装器的托管包装器调用了CLI/C++方法。为此，我们只需将堆栈的所需部分复制到数组中（请注意，从克隆的部分到旧部分，寄存器EBP的副本被复制，以确保帧链的构建）：

![](./imgs/ThreadStack/step4.png)

然后，为了确保在复制在上一步中克隆的部分之后堆栈的完整性，我们预先计算了新位置上字段`EBP`的地址，并立即在副本上进行了修正：

![](./imgs/ThreadStack/step5.png)

最后一步是非常小心地使用最少的寄存器将我们的数组复制到子线程的堆栈末尾，然后将ESP和EBP寄存器移动到新位置。从堆栈的角度来看，我们模拟了所有这些方法的调用：

![](./imgs/ThreadStack/step6.png)

但从代码的角度来看，我们需要进入刚刚创建的那些方法。最简单的方法是模拟方法的退出：将ESP恢复到EBP，将EBP放在它指向的位置，并调用ret指令，从而退出所谓的C++线程克隆方法，这将导致返回到实际的CLI/C++调用包装器，该包装器将返回到`MakeFork()`，但在子线程中。这种技术是有效的。

现在让我们看一下代码。我们首先要做的是让CLI/C++代码能够创建.NET线程。为此，我们必须在.NET中创建它：

```csharp
extern "C" __declspec(dllexport)
void __stdcall MakeManagedThread(AdvancedThreading_Unmanaged *helper, StackInfo *stackCopy)
{
    AdvancedThreading::Fork::MakeThread(helper, stackCopy);
}
```

请暂时忽略参数类型。它们用于传递有关在父线程中绘制到子线程的堆栈的哪个部分的信息。创建线程的方法将调用未管理的方法的委托进行包装，传递数据，并将委托排队等待线程池处理。

```csharp
[MethodImpl(MethodImplOptions::NoInlining | MethodImplOptions::NoOptimization | MethodImplOptions::PreserveSig)]
static void MakeThread(AdvancedThreading_Unmanaged *helper, StackInfo *stackCopy)
{
    ForkData^ data = gcnew ForkData();
    data->helper = helper;
    data->info = stackCopy;

    ThreadPool::QueueUserWorkItem(gcnew WaitCallback(&InForkedThread), data);
}

[MethodImpl(MethodImplOptions::NoInlining | MethodImplOptions::NoOptimization | MethodImplOptions::PreserveSig)]
static void InForkedThread(Object^ state)
{
    ForkData^ data = (ForkData^)state;
    data->helper->InForkedThread(data->info);
}
```

最后，克隆方法的.NET部分：

```csharp
[MethodImpl(MethodImplOptions::NoInlining | MethodImplOptions::NoOptimization | MethodImplOptions::PreserveSig)]
static bool CloneThread()
{
    ManualResetEvent^ resetEvent = gcnew ManualResetEvent(false);
    AdvancedThreading_Unmanaged *helper = new AdvancedThreading_Unmanaged();
    int somevalue;

    // *
    helper->stacktop = (int)(int *)&somevalue;
    int forked = helper->ForkImpl();
    if (!forked)
    {
        resetEvent->WaitOne();
    }
    else
    {
        resetEvent->Set();
    }
    return forked;
}
```

为了理解堆栈帧链中的位置，我们将保存堆栈变量的地址（*）。我们将在克隆方法中使用此地址，该方法稍后将在下面进行讨论。此外，为了让您了解正在讨论的内容，我将提供一个结构的代码，该结构用于存储有关堆栈副本的信息：

```csharp
public class StackInfo
{
public:
    // 寄存器值的副本
    int EAX, EBX, ECX, EDX;
    int EDI, ESI;
    int ESP;
    int EBP;
    int EIP;
    short CS;

    // 堆栈副本的地址
    void *frame;

    // 副本的大小
    int size;

    // 原始堆栈的地址范围是必需的，
    // 以便在需要时修复堆栈上的地址
    int origStackStart, origStackSize;
};
```

实际的算法分为两部分：在父线程中，我们准备数据以在子线程中绘制所需的堆栈帧。然后，第二部分是在子线程中恢复数据，将其覆盖到自己的执行线程的堆栈上，从而模拟实际未调用的方法。

### 准备复制

我将使用块来描述代码。即，统一的代码将被分成几个部分，并且每个部分将单独进行注释。好的，让我们开始吧。当外部代码调用`Fork.CloneThread()`时，通过对未管理代码的内部包装以及一系列其他方法（如果代码在调试模式下运行，则为所谓的调试助手）来调用。这就是为什么我们在.NET部分中保存了变量的地址：对于C++方法，该地址是一种标记：现在我们确切地知道可以安全地复制哪个堆栈部分。

```csharp
int AdvancedThreading_Unmanaged::ForkImpl()
{
    StackInfo copy;
    StackInfo* info;
```

首先，在执行任何操作之前，为了避免破坏寄存器，我们将它们本地复制一份。此外，还需要保存代码的地址，当堆栈模拟完成时，需要进行`goto`操作，并且需要退出所谓的`CloneThread`方法。我们选择`JmpPointOnMethodsChainCallEmulation`作为“退出点”，不是没有原因的：在保存了这个地址“供将来使用”之后，我们还将在堆栈中放置数字0。

```csharp
    // Save ALL registers
    _asm
    {
        mov copy.EAX, EAX
        mov copy.EBX, EBX
        mov copy.ECX, ECX
        mov copy.EDX, EBX
        mov copy.EDI, EDI
        mov copy.ESI, ESI
        mov copy.EBP, EBP
        mov copy.ESP, ESP

        // Save CS:EIP for far jmp
        mov copy.CS, CS
        mov copy.EIP, offset JmpPointOnMethodsChainCallEmulation

        // Save mark for this method, from what place it was called
        push 0
    }
```

然后，在`JmpPointOnMethodsChainCallEmulation`之后，我们从堆栈中取出该数字并检查：那里是0吗？如果是，我们仍然在同一个线程中：这意味着我们还有很多事情要做，我们转到`NonClonned`。如果不是0，而实际上是1，那么这意味着子线程已经完成了将堆栈绘制到所需状态的工作，并且在堆栈中放置了数字1，并从另一个方法中进行了goto。这意味着现在是时候从子线程中的`CloneThread`退出了。

```csharp
JmpPointOnMethodsChainCallEmulation:

    _asm
    {
        pop EAX
        cmp EAX, 0
        je NonClonned

        pop EBP
        mov EAX, 1
        ret
    }
NonClonned:
```

好的，我们确信我们仍然是我们自己，这意味着我们需要为子线程准备数据。为了不再降低到汇编级别，我们将使用先前保存的寄存器的结构进行操作。从中获取EBP寄存器的值：它实际上是堆栈帧链表中的“下一个”字段。通过跟随其中包含的地址，我们将进入调用我们的方法的方法的帧。如果我们再次获取第一个字段并跟随其中包含的地址，我们将进入更早的帧。这样，我们可以到达`CloneThread`的托管部分：毕竟，我们在其堆栈帧中保存了变量的地址，因此我们确切地知道在哪里停下来。这就是下面的循环所做的工作。

```csharp
    int *curptr = (int *)copy.EBP;
    int frames = 0;

    //
    //  Calculate frames count between current call and Fork.CloneTherad() call
    //
    while ((int)curptr < stacktop)
    {
        curptr = (int*)*curptr;
        frames++;
    }
```

获得`CloneThread`方法的起始地址后，我们现在知道要复制多少以模拟从`MakeFork`调用`CloneThread`。但是，由于我们还需要`MakeFork`（我们的任务是退出到它），所以我们额外再次进行了一次单链表遍历：`*(int *)curptr`。然后，我们创建一个数组以保存堆栈，并通过简单的复制将其保存。

```csharp
    //
    //  We need to copy stack part from our method to user code method including its locals in stack
    //
    int localsStart = copy.EBP;                             // our EBP points to EBP value for parent method + saved ESI, EDI
    int localsEnd = *(int *)curptr;                         // points to end of user's method's locals (additional leave)

    byte *arr = new byte[localsEnd - localsStart];
    memcpy(arr, (void*)localsStart, localsEnd - localsStart);
```

我们还需要解决的另一个问题是修复堆栈上的指向堆栈的变量的地址。为了解决这个问题，我们获取操作系统为线程分配的堆栈页的范围。保存获得的信息并计划克隆过程的第二部分，将委托排队等待线程池处理：

```csharp
    // Get information about stack pages
    MEMORY_BASIC_INFORMATION *stackData = new MEMORY_BASIC_INFORMATION();
    VirtualQuery((void *)copy.EBP, stackData, sizeof(MEMORY_BASIC_INFORMATION));

    // fill StackInfo structure
    info = new StackInfo(copy);
    info->origStackStart = (int)stackData->BaseAddress;
    info->origStackSize = (int)stackData->RegionSize;
    info->frame = arr;
    info->size = (localsEnd - localsStart);

    // call managed ThreadPool.QueueUserWorkitem to make fork
    MakeManagedThread(this, info);

    return 0;
}
```

### 从副本中恢复

此方法作为前一个方法的结果调用：我们获得了父线程的堆栈部分的副本以及其完整的寄存器集。我们的任务是在来自线程池的线程中绘制出所有从父线程复制的调用，就好像我们自己进行了这些调用一样。完成工作后，MakeFork的子线程将返回到此方法，该方法完成工作并释放线程，将其返回到线程池。

```csharp
void AdvancedThreading_Unmanaged::InForkedThread(StackInfo * stackCopy)
{
    StackInfo copy;
```



首先，我们保存工作寄存器的值，以便在`MakeFork`完成工作后能够无痛恢复它们。为了尽量减少对寄存器的影响，我们将传递给我们的参数存储在堆栈上。我们只能通过`SS:ESP`来访问它们，这对我们来说是可预测的。

```csharp
    short CS_EIP[3];

    // 保存原始寄存器以便恢复
    __asm pushad

    // 安全复制而不改变寄存器
    for(int i = 0; i < sizeof(StackInfo); i++)
        ((byte *)&copy)[i] = ((byte *)stackCopy)[i];

    // 设置FWORD用于远程jmp
    *(int*)CS_EIP = copy.EIP;
    CS_EIP[2] = copy.CS;
```

我们的下一个任务是修复堆栈副本中形成新位置的`EBP`值，这些值形成了单链表帧。为此，我们计算线程堆栈地址和父线程堆栈之间的差值，以及父线程堆栈副本和父线程本身之间的差值。

```csharp
    // 计算范围
    int beg = (int)copy.frame;
    int size = copy.size;
    int baseFrom = (int) copy.origStackStart;
    int baseTo = baseFrom + (int)copy.origStackSize;
    int ESPr;

    __asm mov ESPr, ESP

    // 目标 = EBP[ - locals - EBP - ret - whole stack frames copy]
    int targetToCopy = ESPr - 8 - size;

    // 父堆栈和当前堆栈之间的偏移量;
    int delta_to_target = (int)targetToCopy - (int)copy.EBP;

    // 父堆栈开始和其副本之间的偏移量;
    int delta_to_copy = (int)copy.frame - (int)copy.EBP;
```

利用这些数据，我们在堆栈副本中循环，并修复地址以便它们指向新位置。

```csharp
    // 在堆栈副本中，我们有许多保存的EPB，它们实际上是单向链表。
    // 我们需要修复副本，使这些指针对我们线程的堆栈正确。
    int ebp_cur = beg;
    while(true)
    {
        int val = *(int*)ebp_cur;

        if(baseFrom <= val && val < baseTo)
        {
            int localOffset = val + delta_to_copy;
            *(int *)ebp_cur += delta_to_target;
            ebp_cur = localOffset;
        }
        else
            break;
    }
```

当单链表修复完成后，我们需要修复副本中的寄存器值，以便如果它们引用堆栈，它们将被修复。实际上，这个算法并不准确。因为如果堆栈中出现了堆栈地址范围之外的不幸数字，它将被错误地修复。但是我们的目标不是为了创建一个产品概念，而是为了理解线程堆栈的工作原理。因此，这种方法对于我们的目的是合适的。

```csharp
    CHECKREF(EAX);
    CHECKREF(EBX);
    CHECKREF(ECX);
    CHECKREF(EDX);

    CHECKREF(ESI);
    CHECKREF(EDI);
```

现在，是主要且最重要的部分。当我们将父线程的堆栈范围的副本复制到我们的堆栈末尾时，一切都很好，直到`MakeFork`在子线程中想要退出（执行`return`）。我们需要告诉它要退出的位置。为此，我们还模拟了从此方法调用`MakeFork`本身。我们将标签`RestorePointAfterClonnedExited`的地址放入堆栈，就好像处理器指令`call`将返回地址放入堆栈一样，还将当前的`EBP`放入堆栈，模拟构建方法帧链表。然后，我们使用普通的`push`操作将父堆栈的副本放入堆栈，从而绘制出在父堆栈中从`MakeFork`方法调用的所有方法，包括它自己。堆栈准备就绪！

```csharp
    // 准备__asm nret
    __asm push offset RestorePointAfterClonnedExited
    __asm push EBP

    for(int i = (size >> 2) - 1; i >= 0; i--)
    {
        int val = ((int *)beg)[i];
        __asm push val;
    };
```

然后，因为我们还需要恢复寄存器，我们恢复它们自己。

```
    // 恢复寄存器，push 1 for Fork() and jmp
    _asm {
        push copy.EAX
        push copy.EBX
        push copy.ECX
        push copy.EDX
        push copy.ESI
        push copy.EDI
        pop EDI
        pop ESI
        pop EDX
        pop ECX
        pop EBX
        pop EAX
```

现在是时候回想起那个奇怪的代码，将`0`放入堆栈并检查是否为`0`。在这个线程中，我们将`1`放入堆栈并远程跳转到`ForkImpl`方法的代码。因为我们在堆栈上，但实际上我们还在这里。当我们到达那里时，`ForkImpl`将识别到线程的切换并退出到`MakeFork`方法，因为我们稍早模拟了从这个点调用`MakeFork`。恢复寄存器到“刚从TheadPool调用”的状态后，我们完成工作，将线程返回到线程池。

```csharp
        push 1
        jmp fword ptr CS_EIP
    }

RestorePointAfterClonnedExited:

    // 恢复原始寄存器
    __asm popad
    return;
 }
```

我们来检查一下吧？这是克隆线程调用之前的屏幕截图：

![](./imgs/ThreadStack/ForkBeforeEnter.png)

然后是调用之后：

![](./imgs/ThreadStack/ForkAfterEnter.png)

正如我们所看到的，现在我们在`ForkImpl`方法中看到了两个线程。它们都从这个方法退出。

## 关于更低级别的一些话

如果我们用余光瞥一眼更低级别，我们会发现或者回想起内存实际上是虚拟的，并且被划分为8或4 KB的页面。每个页面可以实际存在，也可以不存在。如果它存在，它可以映射到文件或实际的物理内存。这种虚拟化机制使应用程序能够拥有彼此分离的内存，并在应用程序和操作系统之间提供安全级别。那么线程堆栈与此有什么关系呢？与任何其他应用程序内存一样，线程堆栈是其一部分，并且由4或8 KB的页面组成。在为堆栈分配的空间的两侧，有两个页面，访问它们会导致系统异常，通知操作系统应用程序正在尝试访问未分配的内存区域。在这个区域内，实际上只有通过应用程序访问的页面是真正分配的：也就是说，如果应用程序将2 MB的内存保留给线程，这并不意味着它们会立即分配。实际上，它们将根据需要分配：如果线程堆栈增长到1 MB，这意味着应用程序已经获得了1 MB的内存用于堆栈。

当应用程序为局部变量分配内存时，会发生两件事：ESP寄存器的值增加，并且变量本身的内存被清零。因此，如果您编写了一个进入无限递归的递归方法，您将收到StackOverflowException：当堆栈占用了分配的所有内存（整个可用区域）时，您将遇到一个特殊的页面，Guard Page，访问它将导致操作系统通知，该通知将触发操作系统级别的StackOverflow，该StackOverflow将返回到.NET，然后.NET将抛出StackOverflowException异常。

## 在堆栈上分配内存：stackalloc

在C#中，有一个非常有趣但很少使用的关键字`stackalloc`。它在代码中非常罕见（我甚至用“有点”一词来低估了它的使用频率。实际上，“从不”更准确），很难找到一个合适的使用示例，更不用说想出一个了：因为很少使用，所以对它的使用经验非常有限。但是为什么呢？因为对于那些最终决定弄清楚这个命令是做什么的人来说，`stackalloc`变得比有用更令人生畏：`stackalloc`返回的结果不是托管指针：它是指向未受保护内存块的普通指针。如果在方法完成后对该地址进行写入，您将开始写入某个方法的局部变量，或者甚至覆盖方法的返回地址，然后应用程序将以错误结束。然而，我们的任务是深入了解并理解为什么给了我们这个工具，而不仅仅是为了我们能够找到秘密的草地并全力以赴。相反，我们被赋予了这个工具，以便我们能够使用它并创建真正快速的软件。我希望我激发了你的灵感？那么让我们开始吧。

要找到正确使用这个关键词的示例，首先要追溯到它的作者：Microsoft公司，并查看他们如何使用它。可以通过在[coreclr](https://github.com/dotnet/coreclr)存储库中进行全文搜索来实现。除了关键词本身的各种测试之外，我们还将在库的代码中找到不超过25个关键词的使用。我希望在前面的段落中我已经充分激励了你，以至于你不会因为看到这个小数字而停止阅读并关闭我的努力。说实话：CLR团队比.NET Framework团队更有远见和专业素养。如果他们做了某些事情，那么这对我们来说肯定会有很大帮助。如果在.NET Framework中没有使用它...嗯，我们可以假设那里并不是所有的工程师都知道有这样一个强大的优化工具。否则，它的使用量将会更大。

**类 Interop.ReadDir**
[/src/mscorlib/shared/Interop/Unix/System.Native/Interop.ReadDir.cs](https://github.com/dotnet/coreclr/blob/b29f6328510207970763580d6f4db864e4b198af/src/mscorlib/shared/Interop/Unix/System.Native/Interop.ReadDir.cs#L71-L83)

```csharp
unsafe
{
    // 当本机实现不支持读取到缓冲区时，s_readBufferSize为零。
    byte* buffer = stackalloc byte[s_readBufferSize];
    InternalDirectoryEntry temp;
    int ret = ReadDirR(dir.DangerousGetHandle(), buffer, s_readBufferSize, out temp);
    // 我们将数据复制到DirectoryEntry中，以确保没有悬空引用。
    outputEntry = ret == 0 ?
                new DirectoryEntry() { InodeName = GetDirectoryEntryName(temp), InodeType = temp.InodeType } :
                default(DirectoryEntry);

    return ret;
}
```

这里为什么使用`stackalloc`？我们可以看到，在分配内存后，代码进入了一个不安全的方法，用于将创建的缓冲区填充数据。也就是说，不安全的方法需要一个用于写入的区域，它直接在堆栈上动态分配了空间。这是一个很好的优化，因为考虑到替代方案：从Windows请求内存块或使用.NET的固定（pinned）数组，后者除了对堆的负载外，还会通过将数组钉在那里来增加GC的负担，以防止GC在访问数据时将其移动。通过在堆栈上分配内存，我们没有任何风险：分配几乎是瞬间完成的，我们可以安心地填充它并退出方法。随着方法的退出，堆栈帧也会消失。总之，这大大节省了时间。

让我们再看一个例子：

**类 Number.Formatting::FormatDecimal**
[/src/mscorlib/shared/System/Number.Formatting.cs](https://github.com/dotnet/coreclr/blob/efebb38f3c18425c57f94ff910a50e038d13c848/src/mscorlib/shared/System/Number.Formatting.cs#L287-L311)

```csharp
public static string FormatDecimal(decimal value, ReadOnlySpan<char> format, NumberFormatInfo info)
{
    char fmt = ParseFormatSpecifier(format, out int digits);

    NumberBuffer number = default;
    DecimalToNumber(value, ref number);

    ValueStringBuilder sb;
    unsafe
    {
        char* stackPtr = stackalloc char[CharStackBufferSize];
        sb = new ValueStringBuilder(new Span<char>(stackPtr, CharStackBufferSize));
    }

    if (fmt != 0)
    {
        NumberToString(ref sb, ref number, fmt, digits, info, isDecimal:true);
    }
    else
    {
        NumberToStringFormat(ref sb, ref number, format, info);
    }

    return sb.ToString();
}
```

这是一个基于更有趣的[ValueStringBuilder](https://github.com/dotnet/coreclr/blob/efebb38f3c18425c57f94ff910a50e038d13c848/src/mscorlib/shared/System/Text/ValueStringBuilder.cs)类的例子，它基于`Span<T>`工作。这段代码的要点是，为了尽快地构建格式化数字的文本表示，代码不使用分配累积字符缓冲区的方法。这段精彩的代码直接在方法的堆栈帧中分配内存，从而避免了使用基于StringBuilder的实例时垃圾收集器的工作。此外，减少了方法本身的运行时间：在堆中分配内存也需要时间。而使用`Span<T>`类型而不是裸指针在基于stackalloc的代码中增加了安全性感。

此外，在进入结论之前，还有一点需要提及，即不应该如何做。换句话说，哪些代码可能运行良好，但在最不合适的时刻会出现问题。再次，看一个例子：

```csharp
void GenerateNoise(int noiseLength)
{
    var buf = new Span(stackalloc int[noiseLength]);
    // generate noise
}
```

这段代码看似简单：不能从外部获取要在堆栈上分配的内存的大小。如果你真的需要从外部获取指定大小，那么接受缓冲区本身是更好的选择：

```csharp
void GenerateNoise(Span<int> noiseBuf)
{
    // generate noise
}
```

这段代码更具信息性，因为它迫使用户在选择数字时思考并小心。在不利情况下，第一种情况可能会在堆栈帧相对较浅的情况下抛出`StackOverflowException`，只需将大数作为参数传递即可。第二种情况是当该方法在特定情况下调用，并且调用代码“知道”该方法的工作原理时。如果没有关于方法内部结构的了解，就没有关于noiseLength的可能范围的具体理解，因此可能会出现错误。

我看到的第二个问题是，如果我们随机地无法满足我们自己在堆栈上分配的缓冲区的大小，并且我们不希望失去可用性，那么当然可以采取几种方法：要么在堆栈上分配更多的内存，要么在堆中分配内存。而且，很可能在大多数情况下，第二种选择更可取（就像在`ValueStringBuffer`的情况下一样），因为从GC的角度来看，它更安全，可以避免`StackOverflowException`。

## stackalloc的结论

那么，何时最好使用`stackalloc`？

- 对于与非托管代码的交互，当需要用非托管方法填充某个缓冲区数据或从非托管方法接收将在方法生命周期内使用的缓冲区数据时；

- 对于需要数组的方法，但同样是在方法的运行时间内。格式化的示例非常好：该方法可能被频繁调用，以至于它不会在堆中分配临时数组；

使用这个分配器可以大大提高应用程序的性能。

## 对本节的结论

当然，在产品代码中我们没有必要编辑堆栈：除非你想在空闲时间里做一些有趣的事情。然而，对其结构的理解使我们能够以简单的方式获取其中的数据并进行编辑。也就是说，如果您为扩展应用程序功能开发API，并且该API不提供对任何数据的访问，这并不意味着无法获取这些数据。因此，始终检查您的应用程序是否能够抵御攻击。