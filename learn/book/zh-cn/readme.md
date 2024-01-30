## 引言

我构思这本书的目的是尽可能全面地描述.NET CLR的工作原理，部分涉及.NET Framework，并且首要目标是让读者从一个不同的角度来看待它的内部结构：不同于通常的方式。这主要与一个可能引起争议的观点有关：任何开发人员都应该接受C/C++的训练。为什么呢？因为从高级语言中，这两种语言最接近处理器，通过在它们上面编程，你会更加深刻地感受到程序的运行。然而，我理解到世界的运作方式有所不同，我们通常没有时间去学习我们不会直接使用的东西，所以我决定写这本书，其中所有问题的解释都从比通常更深入的角度出发，并且使用更复杂或者简单说是替代的例子。除了它们的标准任务-通过最简单的代码展示特定功能的工作原理，它还向另一个现实世界致敬，展示一切比起最初看起来要复杂得多。为什么呢？为了让你们也能对CLR的工作有一个全面的理解。

1. **第1部分。内存**
    1. **第1节。**内存管理入门
        1. [概述](./Memory/01-Introduction/01-00-Introduction-Intro.md)
        1. [内存管理入门](./Memory/01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#内存管理入门)
            1. [开始之前的几句话](./Memory/01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#内存管理wide)
            1. [内存管理入门](./Memory/01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#内存管理入门)
            1. [根据逻辑对内存进行可能的分类](./Memory/01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#如何对内存进行分类)
            1. [我们的工作原理。架构师的选择理由](./Memory/01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#我们的工作原理架构师的选择理由)
            1. [结论](./Memory01-Introduction/01-01-Introduction-MemoryManagement-Basics.md#结论)
        1. [线程堆栈](./Memory/02-MemoryManagement-Basics/02-01-MemoryManagement-ThreadStack.md)
            1. [基本结构，x86平台](./Memory/02-MemoryManagement-Basics/02-01-MemoryManagement-ThreadStack.md#x64-amd64平台的基本信息)
            1. [x86平台上的异常处理](./Memory/02-MemoryManagement-Basics/02-01-MemoryManagement-ThreadStack.md#x64-amd64平台上的异常处理)
            1. [线程堆栈的一些缺陷](./Memory/02-MemoryManagement-Basics/02-01-MemoryManagement-ThreadStack.md#线程堆栈的一些缺陷)
            1. [大例子：在x86平台上克隆线程](./Memory//02-MemoryManagement-Basics/02-01-MemoryManagement-ThreadStack.md#大例子在x86平台上克隆线程)
        1. [实体的生命周期](./Memory/02-MemoryManagement-Basics/02-02-MemoryManagement-EntitiesLifetime.md)
            1. [引用类型](./Memory/02-MemoryManagement-Basics/02-02-MemoryManagement-EntitiesLifetime.md#引用类型的生命周期)
                1. [总览](./Memory/02-MemoryManagement-Basics/02-02-MemoryManagement-EntitiesLifetime.md#总览)
                1. [支持当前方法的选择](./Memory/02-MemoryManagement-Basics/02-02-MemoryManagement-EntitiesLifetime.md#支持当前方法的选择.net平台)
                1. [初步结论](./Memory/02-MemoryManagement-Basics/02-02-MemoryManagement-EntitiesLifetime.md#初步结论)
        1. [RefTypes、ValueTypes、装箱和拆箱](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md)
            1. [引用类型和值类型](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#引用类型和值类型)
            1. [复制](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#复制)
            1. [重写方法和继承](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#重写方法和继承)
            1. [调用实例方法时的行为](/Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#调用实例方法时的行为)
            1. [指定元素位置的能力](/Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#指定元素位置的能力)
            1. [分配的差异](/Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#分配的差异)
            1. [class/struct选择的特点](/Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#classstruct选择的特点)
            1. [基本类型 - Object和实现接口的能力。装箱](/Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#基本类型----object和实现接口的能力装箱)
            1. [Nullable&lt;T&gt;](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#nullablet)
            1. [更深入地了解装箱](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#更深入地了解装箱)
            1. [为什么.NET CLR不自动进行装箱池化？](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#为什么net-clr不自动进行装箱池化)
            1. [为什么在调用一个接受object类型参数的方法时，实际上传递的是值类型，无法在堆栈上进行装箱并卸载堆？](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#为什么在调用一个接受object类型参数的方法时实际上传递的是值类型无法在堆栈上进行装箱并卸载堆)
            1. [为什么不能将值类型本身用作字段？](./Memory/02-MemoryManagement-Basics/02-03-MemoryManagement-RefVsValueTypes.md#为什么不能将值类型本身用作字段)
        1. [可释放模式](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md)
            1. [IDisposable](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#idisposable)
            1. [IDisposable的实现变体](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#idisposable的实现变体)
            1. [SafeHandle / CriticalHandle / SafeBuffer / 派生类](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#safehandle--criticalhandle--safebuffer--派生类)
                1. [实例方法的finalizer触发](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#实例方法的finalizer触发)
            1. [多线程](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#多线程)
            1. [两级可释放设计原则](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#两级可释放设计原则)
            1. [Dispose的其他用途](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#dispose的其他用途)
                1. [委托，事件](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#委托事件)
                1. [Lambda表达式，闭包](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#lambda表达式闭包)
            1. [防止ThreadAbort](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#防止threadabort)
            1. [总结](./Memory/02-MemoryManagement-Basics/02-04-MemoryManagement-IDisposable.md#总结)
        1. [终结器](./Memory/02-MemoryManagement-Basics/02-05-MemoryManagement-Finalizer.md)
        1. [结论](./Memory/02-MemoryManagement-Basics/02-08-MemoryManagement-Results.md)
    1. **第2节。**实践
        1. [Memory, Span](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md)
            1. [Span&lt;T&gt;, ReadOnlySpan&lt;T&gt;](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#spant-readonlyspant)
                1. [Span&lt;T&gt;的示例](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#spant的示例)
                1. [使用规则和实践](/Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#使用规则和实践)
                1. [Span&lt;T&gt;的工作原理](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#spant的工作原理)
                1. [将Span&lt;T&gt;作为返回值](/Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#将spant作为返回值)
            1. [Memory&lt;T&gt;和ReadOnlyMemory&lt;T&gt;](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#memoryt和readonlymemoryt)
                1. [Memory&lt;T&gt;.Span](/Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#memorytspan)
                1. [Memory&lt;T&gt;.Pin](/Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#memorytpin)
                1. [MemoryManager、IMemoryOwner、MemoryPool](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#memorymanager-imemoryowner-memorypool)
            1. [性能](./Memory/02-MemoryManagement-Basics/02-06-MemoryManagement-MemorySpan.md#性能)
    1. **第3节。**GC实现细节
        1. [对象分配](./Memory/02-MemoryManagement-Basics/02-07-MemoryManagement-Allocation.md) *(仅有口述文字)*
        1. [GC简介](./Memory/03-MemoryManagement-Advanced/03-01-MemoryManagement-GC-Intro.md) *(仅有口述文字)*
        1. [标记阶段](./Memory/03-MemoryManagement-Advanced/03-02-MemoryManagement-GC-Mark-Phase.md) *(仅有口述文字)*
        1. [计划阶段](./Memory/03-MemoryManagement-Advanced/03-03-MemoryManagement-GC-Planning-Phase.md) *(仅有口述文字)*
        1. [清理/收集阶段](./Memory/03-MemoryManagement-Advanced/03-04-MemoryManagement-GC-Sweep-Collect.md) *(仅有口述文字)*
        1. [内存管理和性能工作的结论](./Memory/03-MemoryManagement-Advanced/03-05-MemoryMenegement-GC-Results.md) *(仅有口述文字)*
    1. **第4节。**对象在内存中的结构
        1. [对象在内存中的结构](./Memory/03-MemoryManagement-Advanced/03-06-ObjectsStructure.md)
    1. **第5节。**乱序的内容
        1. [生命周期模板](./Memory/04-OptimisationPractice-Daily/04-05-Lifetime.md)
1. **第2部分。指令执行流程**
      1. 第1节。线程
          1. [线程简介](./Execution/01-Threads/01-01-Threads-Introduction.md)
          1. [线程调度](./Execution/01-Threads/01-02-Threads-Scheduler.md)
          1. [Thread和ThreadPool](./Execution/01-Threads/01-03-Threads-Thread-ThreadPool.md)
          1. [同步上下文](./Execution/01-Threads/01-04-Threads-SynchronizationContext.md)
      1. 第2节。异常情况
          1. [异常情况简介](./Execution/2-ExceptionalFlow/1-Exceptions-Intro.md)
          1. [异常情况架构](./Execution/2-ExceptionalFlow/2-Exceptions-Architecture.md)
          1. [异常情况事件](./Execution/2-ExceptionalFlow/3-Exceptions-Events.md)
          1. [异常情况类型](./Execution/2-ExceptionalFlow/4-Exceptions-Types.md)

# 许可证

请参阅[LICENSE](../../LICENSE)文件。