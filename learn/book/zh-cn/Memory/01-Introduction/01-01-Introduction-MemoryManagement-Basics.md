我昨天去了一家新开的餐厅，名字叫做“花园餐厅”。这家餐厅的装修非常漂亮，里面有很多绿色植物和花朵，给人一种舒适和放松的感觉。菜单上有各种各样的菜品，包括中餐和西餐。我点了一份糖醋排骨和一份意大利面。糖醋排骨的味道非常好，酸甜适中，肉质鲜嫩多汁。意大利面也很美味，面条煮得刚刚好，配料丰富。整个用餐过程非常愉快，服务员们也非常友好和专业。我会推荐这家餐厅给我的朋友们。

# 内存管理{.wide}

[>m]: ## 音频书籍

   [书籍链接](https://music.yandex.ru/album/9613103)

当我在2015年左右与不同的人交谈并讲述垃圾回收器的工作原理时（对我来说，这是一个有趣而奇怪的爱好），整个故事最多只需要15分钟。然后，我被问了一个问题：“为什么要知道这个？毕竟它总是工作的。”然后我的脑海中开始混乱：一方面，我知道他们在大多数情况下是正确的。我讲述了那些在这些知识中表现出色并被广泛使用的少数情况。但是由于这些情况确实是少数，对方仍然有一些不信任感。

根据我们之前在少数来源中获得的知识，被面试成为开发人员的人通常会说：有三代，一对大堆和小堆。最多还可以听到一些关于段和卡表的内容。但通常人们不会进一步讨论代和堆。这是为什么呢？因为**并不是因为他们不知道什么，而是因为不清楚为什么要知道**。因为以前给我们提供的信息看起来像是一个大而封闭的广告手册。我们知道有三代，那又怎样？.. 你必须承认，这些都是一些虚无缥缈的东西。

现在，当微软开放了源代码时，我对此有一些期望。第一个好处是社区会积极参与并开始修复一些错误。他们果然参与了！他们纠正了评论中的所有语法错误。他们修复了许多逗号和拼写错误。有时甚至写道，例如，属性`IsEnabled`返回一个指示某物是否启用的标志。你甚至可以根据这些评论（当然，不会成功）申请加入.NET基金会。第二个预期的好处是以现成的形式提供新的有用功能。据我所知，这个好处也偶尔发生，但这是非常罕见的情况。例如，有一次开发人员大大加快了通过索引获取字符的速度。结果发现，以前这个过程效率不高。

[>]: 加入.NET基金会现在需要通过辛勤工作和有趣的提交来赢得。您可以通过优化某些用户案例来开始在方法编译器（JIT）的代码中寻找好的机会。例如，自动向量化某些计算。

我们的内存管理故事将从整体到局部进行。也就是说，首先我们将从鸟瞰高度来看待算法，而不会深入细节。因为如果我们立即开始讨论细节，我们将不得不引用未来的章节，然后再回到早期的章节。这对于写作和阅读来说都非常不方便。相反，通过介绍我们将理解所有基础知识。然后我们将开始深入细节。

## 内存管理简介

我们编写各种程序：控制台应用程序、服务、Web服务等。它们都工作得差不多。其中一个非常重要的区别是内存管理的方式。控制台应用程序很可能在应用程序启动时分配的基本内存中运行。这样的应用程序完全或部分使用它，并且不会再请求任何东西：它启动并退出。有时我们谈论的是长时间运行的服务，它们不断地重新分配内存。它们不是根据孤立的请求进行操作，而是像某种计算服务一样：它们有一个输入数据流，服务使用它（例如，来自`RabbitMQ`的命令流），并且可以长时间运行，分配和释放内存。这是一种完全不同的内存消耗方式：因为在这种情况下，我们需要控制内存，查看它如何消耗、泄漏或不泄漏。

如果是ASP.NET，那么这是第三种内存管理方式。我们必须明白，我们是被外部代码调用的，我们将迅速完成工作并消失。因此，如果我们在请求期间分配了一些内存，我们可以做到不必担心释放它：毕竟，方法将完成其工作，所有对象都将失去其根：处理请求的方法的局部变量。但是，我们也不应该浪费：在高负载下，您将获得对象流量。也许有些对象应该成为结构？

我们如何管理这一切？从垃圾回收器（GC）的角度来看，从内存管理系统（Memory Management System）的角度来看，我们有完全不同的风格，我们必须在其中表现得非常好。我们可能有一台机器，上面运行着控制台应用程序，还有一台机器，应用程序占用了256 GB。除了占用内存的大小之外，这些系统在许多其他方面也有所不同：例如，它们在分配和释放内存的方式上的差异（通过将引用设置为null），以及在某些类型的对象的分配和释放的密度（例如，一个应用程序经常使用字符串，而另一个应用程序经常使用数据数组）。根据数据类型的不同，可能会发现在内存管理方面采取不同的方法更好。因此，在考虑如何实现某个内存管理器之前，应该思考一个问题：如何对这些内存进行分类？从内存分类开始，我们可以根据我们当前处理的内存类别来优化其分配和释放。让我们以我们自己开发垃圾回收器的方式来考虑这个问题：一步一步地，我们将得出与平台开发人员得出的相同结论。

## 如何对内存进行分类？

如何对内存进行分类？从直观上讲，我们可以根据分配的对象的大小来划分内存区域。例如，显然，如果我们讨论大型数据结构，我们需要以完全不同的方式管理它们，而不是小型数据结构：因为它们很重，如果需要（例如，为了减少C/C++编写的常规程序的碎片化），它们很难移动。而小型数据结构占用的空间较小，并且由于它们形成组，因此易于移动。然而，由于它们数量众多，从内存管理器的角度来看，它们更难以管理：毕竟，要管理它们，需要知道每个对象在内存中的位置。因此，对于它们来说，没有统计数据，我们很清楚应该有一种不同的方法。

如果根据生命周期划分，也会有一些想法。例如，如果对象的生命周期很短，那么我们可能需要更频繁地关注它们，以便尽快摆脱它们（最好是一旦它们不再需要）。那么GC将更快地运行：如果工作量较小，那么完成得越快。如果对象的生命周期很长，那么我们可以考虑一下统计数据。例如，我们可以幻想一下，决定较少地分析内存区域以查找不需要的对象：毕竟，如果对象的生命周期很长，我们只需要较少地检查它们的“死亡”。而如果很少检查，这将减少总体的垃圾收集时间，但会增加每次GC调用的持续时间：在此期间，可能会积累大量已经死亡的长寿命对象。这使人想到，长寿命对象应该与短寿命对象分开存储：这样，我们可以使用一组算法来分析短寿命对象，另一组算法来分析长寿命对象。

还可以尝试根据数据类型对内存进行分类。可以轻松推断出所有从`Attribute`类型继承的类型或位于`Reflection`区域的类型（尤其是我们无法访问的`runtime`部分）将永远存在-因此我们也可以以特殊方式处理它们。

[>]: 请注意，`runtime`部分的`reflection`位于我们无法访问的堆中。

同样，对于表示字符数组的字符串，可以应用一些特殊的方法：字符串可能经常重复。这意味着可以考虑消除字符串的重复（字符串的合并）。

分类的可能性有很多，根据分类的不同，我们可能会发现，对于特定组的内存管理可能更有效。

当我们设计我们的GC架构时，我们选择了前两种分类：大小和生命周期（尽管如果我们考虑类型的划分为类和结构，我们可以认为实际上有三种分类。然而，类和结构的属性的区别可以归结为大小和生命周期）。让我们解释一下这个选择的理由。

## 我们的工作方式。架构师的选择理由

如果我们要深入研究为什么选择了这两种内存管理算法：*Sweep*（标记可用块以供后续重用）和*Compact*（压缩堆以减少碎片化），我们将不得不研究世界上存在的数十种内存管理算法：从普通字典到非常复杂的无锁结构。而不是这样做，我们将只是为我们的选择提供*理由*，从而*理解*为什么选择了这样的选择。我们不再翻阅火箭的宣传手册：我们手头有完整的文档。

让我们明确术语：内存管理是一组数据结构和一系列算法，它们允许“分配”内存并将其提供给外部消费者，并通过将其注册为自由块来释放内存。也就是说，如果我们采取某个字节数组（线性内存块），编写将数组分割为.NET对象的算法（请求新对象：我们计算其大小，将其标记为新对象，将对象的指针传递给外部），并编写释放内存的算法（当我们被告知对象不再需要时，我们可以将内存释放给其他人），那么我们就得到了内存管理器。

[>]: 这在.NET中是不可能的，但在C++中是很常见的，当需要它们时，重写`new`和`delete`运算符。编写额外的方法，它们将租用大量虚拟内存并将其划分为对象。然后释放，再次分配... 这是出于各种原因。减少碎片化是其中之一。

基于我们对分配的对象进行的分类，我们可以将内存分配的位置分为两个大的部分：小于85K字节的对象的位置和大于或等于85K字节的对象的位置。这样做是为了能够使用压缩堆算法-`Compact`。如果没有将内存分为两个堆，垃圾回收器的工作时间将非常长。

此外，对于小对象，我们还将内存分为三个代。这样做是为了尽量快速地释放内存。毕竟，如果对象经历了几次GC调用（在对象的流量适中的情况下），那么它将存在很长时间。这给了我们一个理由偶尔查看老一代的对象。

但是，还有一个问题：如果我们只有两个代，我们会遇到问题：

- 要么我们将努力使GC尽可能快速：这样，我们将尽量使*年轻一代的大小最小*。结果是，当GC在“现在就发生，在疯狂分配大量对象的情况下”发生时，最近创建的对象将无意中进入老一代（如果GC稍后发生，它们将留在年轻一代，其中它们在短时间内被收集）。
- 要么，为了最小化这种随机的“下降”，我们将*增加年轻一代的大小*。然而，在这种情况下，年轻一代的GC将相当长，从而减慢和延迟整个应用程序。

解决方案是引入一个“中间”代。它的目的是在*年轻一代的最小大小*和*最大稳定的老一代*之间取得平衡。这是一个区域，其中对象的命运尚未决定。第一个（不要忘记我们从零开始计算）代是固定大小的，以保证GC的时间不会超过某个限制。否则，这将对用户产生影响。在无限制的容量上，这个操作将是随机的长时间操作。
- 第二代是一个巨大的仓库，因此我们希望尽量少地查看它。

> 因此，我们得到了三个代的想法。

下一层优化是尝试摆脱压缩。毕竟，如果我们不这样做，我们就摆脱了大量的工作。让我们回到空闲区域的问题。

如果在我们在堆中使用了所有可用的内存之后，并且调用了GC，那么我们自然希望放弃压缩，而是继续在空闲的区域内分配内存，如果它们足够大以容纳一些对象。在这种情况下，我们遇到了GC的第二个内存释放算法，称为`Sweep`：不压缩内存，而是使用空闲的释放对象的空间来分配新对象。

> 这样我们描述并解释了GC的所有基础。

我们不会进一步深入，否则我将没有思考的余地。我只想指出，我们讨论了与现有内存管理算法相关的所有前提和结论。

## 结论

让我们快速回顾一下，以便更好地记住：
- 有两个对象堆：用于小于85K字节的对象和用于大于或等于85K字节的对象。这样做是为了使用压缩堆算法-`Compact`。如果没有将内存分为两个堆，垃圾回收器的工作时间将非常长。
- 对于小对象，堆被分为三个代。这样做是为了尽量快速地释放内存。毕竟，要是对象经历了几次GC调用（在对象的流量适中的情况下），那么它将存在很长时间。这给了我们一个理由偶尔查看老一代的对象。
- 内存是线性的，代是在内存中的连续区域，因此不同代的对象不会混合。相反，首先是第二代的对象，然后是第一代的对象，最后是第零代的对象。
- 除了`Compact`之外，还有第二个管理已用和空闲区域的算法：`Sweep`。实际上，这是堆中的空闲区域列表。它在两个堆中都工作：LOH和SOH。当堆中没有空间时，计算空闲区域。如果区域适合对象的分配，则继续分配。否则-启动`Compact`。对于SOH，`Sweep`是第一优先级，因为它更容易，而`Compact`是第二优先级。对于LOH-始终是`Compact`，因为压缩大型数组不是最佳选择。