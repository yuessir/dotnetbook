It seems like you haven't provided any text to translate. Could you please provide the text you'd like translated into Chinese?

# 引用类型和值类型

现在，当我们加强了对 .NET 中内存管理基础的理解后，让我们来讨论一下引用类型（Reference Types）和值类型（Value Types）。如果要谈论它们之间的区别以及每种类型的用途，首先我想提到的是我对它们命名的看法。在我谦虚的观点中，如果在俄语区它们被称为引用类型和值类型，而不是说 Value Types 和 Reference Types，那么理解它们之间的区别就会变得更加直观。

> 当被问及什么是引用类型和值类型时，人们经常回答说引用类型存在于堆中，而值类型存在于栈中。这是完全错误的。这只是真相的一小部分，根本不能算是真相。

为了理解它们之间的区别，让我们从不同点来学习它们：

  - *值类型*：值是**整个结构**。对于*引用类型*，值是对对象的**引用**；
  - 在内存结构上：值类型只包含你指定的数据。引用类型还包含两个系统字段。第一个用于存储 `SyncBlockIndex`，第二个用于存储类型信息：包括 Virtual Methods Table (VMT)；
  - 然而，引用类型可以继承，重写方法。值类型没有这样的能力；
  - 但是，要分配引用类型，需要在堆上分配空间。值类型*可以*在栈上工作，不进入堆，也可以成为引用类型的一部分。这个属性可以显著提高某些算法的性能；

然而，它们也有共同之处。

>{.big-quote}  两个子类都继承自 object 类型。<br>
    这意味着，它们可以作为它的代表：拥有完整的权利

让我们分别考虑每个特性。

## 复制

两种类型之间最主要和基本的区别可以大致描述如下：

  - 任何接受引用类型的变量、类/结构的字段或方法参数，实际上存储的是对值的**引用**；
  - 而任何接受值类型的变量、类/结构的字段或方法参数，实际上存储的是值本身。即整个结构；

这对我们意味着什么？这特别意味着任何赋值或通过方法参数传递都会导致值的复制。改变副本不会改变原件。同时，如果你改变了引用类型的字段，所有拥有该类型实例引用的人都会看到这些更改。让我们用一个例子来看看：

{.wide}
```csharp

 DateTime dt = DateTime.Now;   // 这里首先在调用方法时为 DateTime 变量分配空间，
                               // 但它将被零填充。然后将 Now 属性的所有值复制到 dt 变量中
 DateTime dt2 = dt;            // 这里的值再次被复制

 object obj = new object();    // 创建一个对象，为其在 SOH 中分配内存，并将对象的指针放在 obj 变量中
 object obj2 = obj;            // 这里我们复制了对该对象的引用。即对象是一个，引用是两个

```     

这个属性导致了一些乍一看有歧义的代码结构。其中之一是在集合中更改值：

```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的数组并初始化 Data 字段为 5
var array = new [] { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4
array[0].Data = 4;

// 检查值
Console.WriteLine(array[0].Data);
```

在这段代码中有一个小技巧。代码看起来好像我们首先获取结构的实例，然后在得到的副本上设置新的 Data 字段值。这意味着在检查时我们应该再次得到 `5`。但事实并非如此。问题在于 MSIL 有一个特殊的指令用于设置数组中结构的字段值。它是为了提高性能而引入的。这段代码将按照其作者的意图执行：程序将在控制台中输出数字 `4`。

然而，如果稍作更改：

```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的列表并初始化 Data 字段为 5
var list = new List<ValueHolder> { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4
list[0].Data = 4;

// 检查值
Console.WriteLine(list[0].Data);
```

这样我们甚至无法编译。原因是当你写 `list[0].Data = 4` 时，你首先得到了结构的副本。实际上，你调用了 `List<T>` 类型的实例方法，它隐藏在索引访问背后。这个方法从内部数组中获取结构的副本（`List<T>` 在数组中存储数据），这个副本从索引访问方法返回给你。然后你尝试修改这个副本，它之后不会被使用。这不是错误，但绝对是无意义的代码。编译器知道人们对值类型感到困惑，因此禁止这种行为。因此，示例应该被重写如下：

{.wide}
```csharp
// 声明一个结构
struct ValueHolder
{
    public int Data;
}

// 创建这样一个结构的列表并初始化 Data 字段为 5
var list = new List<ValueHolder> { new ValueHolder { Data = 5 } };

// 通过索引获取结构并将 Data 字段设置为 4，然后再保存回去
var copy = list[0];
copy.Data = 4;
list[0] = copy;

// 检查值
Console.WriteLine(list[0].Data);
```

尽管看起来有些啰嗦，但它是正确的。当程序执行时，控制台将输出数字 `4`。

我想通过第二个例子向你展示的是“结构的值是整个结构本身”的含义：

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

int x = 5;
PersonInfo person;
int y = 6;

// 方案 2
int x = 5;
int Height;
int Width;
int HairColor;
int y = 6;
```

从数据在内存中的位置来看，这两个例子是相同的。因为结构的值是整个结构。它占据了自己的内存空间。

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

class Employee
{
    public int x;
    public PersonInfo person;
    public int y;
}

// 方案 2
class Employee
{
    public int x;
    public int Height;
    public int Width;
    public int HairColor;
    public int y;
}
```

这些例子在内存中元素位置的角度来看也是相同的，因为结构简单地占据了它被定义的位置：在类的字段中。我并不是说这完全相同：毕竟在结构中你可以通过结构的方法操作它的字段。

如果我们谈论引用类型，那么情况显然不同。实例本身位于不可达的 Small Object Heap (SOH) 或 Large Object Heap (LOH) 中，而在类的字段中只会记录指向实例的指针：32位或64位数字。

最后一个例子，我希望不会让你感到困惑。但我想在这个问题上画上句号。

```csharp
// 方案 1
struct PersonInfo
{
    public int Height;
    public int Width;
    public int HairColor;
}

void Method(int x, PersonInfo person, int y);

// 方案 2
void Method(int x, int HairColor, int Width, int Height, int y);
```

你完全正确地理解了：从内存工作的角度来看，这两种方案将以相同的方式工作（但不是从架构的角度来看！这并不意味着它们是可变参数的替代品！）。为什么顺序改变了？因为方法参数是依次声明的，并且按照这个顺序堆叠在线程的栈上。然而，栈是从高地址向低地址增长的，这意味着按顺序堆叠的顺序将与整个结构堆叠的顺序不同。

## 可重写方法和继承

它们之间的第二个全球性差异是结构中没有虚方法表。这意味着：

  1. 在结构中不能声明 `virtual` 方法，也不能重写它们；
  2. 结构原则上不能相互继承。唯一的继承模拟方法是将基本类型结构作为第一个字段放置。然后它们的偏移将匹配，"继承"结构的字段将位于"基本"字段之后，从逻辑上你将实现继承；
  3. 与类不同，结构可以被传递到非托管代码中。我指的是它的值。方法信息当然会丢失。因为结构只是数据填充的内存段，没有类型信息。这意味着它可以不加改变地传递给例如 C++ 编写的非托管方法。

>{.big-quote}   虽然缺少虚方法表剥夺了结构一些继承带来的“魔法”，但也赋予了它们一系列优势。

首先也是最重要的一点已经提到：我们可以轻松地将这样的结构实例传递给外部世界（超出 .NET Framework 的范围）。这只是一段内存！或者我们可以从非托管代码接收一段内存并将其转换为我们的结构，以便更方便地访问其字段。对于类来说，这种行为是不可能的：类有两个不可访问的字段：SyncBlockIndex 和虚方法表的地址。如果这两个字段进入非托管代码，这将是非常危险的。因为通过虚方法表，你可以巧妙地访问任何类型并更改它，对应用程序发起攻击。

让我们证明这只是一段没有任何附加逻辑的内存：

{.wide}
```csharp
unsafe void Main()
{
    int secret = 666;
    HeightHolder hh;
    hh.Height = 5;

    WidthHolder wh;
    unsafe
    {
        // 如果结构中有类型信息，这种类型转换将无法工作：
        // CLR 在类型转换前会检查继承层次结构，如果在其中找不到 WidthHolder，
        // 将抛出 InvalidCastException。但由于结构只是一段内存，
        // 在 unsafe 世界中，没有人能阻止你将它解释为任何结构
        wh = *(WidthHolder*)&hh;
    }
    Console.WriteLine("Width: " + wh.Width);
    Console.WriteLine("Secret: " + wh.Secret);
}

struct WidthHolder
{
    public int Width;
    public int Secret;
}

struct HeightHolder
{
    public int Height;
}
```

在这个例子中，我们执行了一种在严格类型化的角度来看是不允许的操作：我们将一种类型转换为完全不兼容的另一种类型，后者有一个额外的字段。在 `Main` 方法中，我们引入了一个额外的变量，其值在理论上是秘密的，不应该被读取。但事实并非如此。示例确信地在屏幕上显示了 Main() 方法中的变量值，尽管它不在任何结构中。此时你的脸上应该露出微笑，脑海中闪过一句话：“天哪，这是个安全漏洞！！！”... 但实际上并不是那么明显。保护你的代码免受调用的非托管代码的影响几乎是不可能的。这主要是因为线程栈的结构，我们稍后会讨论，你可以轻松地进入被调用的代码，并搞乱局部变量。防止这种类型的攻击的保护措施是通过其他方式建立的。例如，通过随机化栈帧大小或擦除 EBP 寄存器信息来增加栈帧恢复的难度。但我们不会深入讨论这个话题：这是另一个话题。唯一值得在这个例子中提到的是，为什么尽管变量 `secret` 在变量 `hh` **之前**定义，而在结构 `WidthHolder` 中**之后**（即在视觉上完全不同的位置），它的值仍然可以被完美地读取。这是因为栈的增长方向不是从左到右，而是相反——从右到左。即首先声明的变量将位于更高的地址上，而后声明的变量将位于更早的地址上。

## 调用实例方法的行为

两种数据类型还有另一个有趣的特性，这个特性不是显而易见的，它可以为我们对两种类型的结构提供更多的理解。这个特性与调用实例方法有关。

```csharp

// 引用类型的示例
class FooClass 
{
    private int x;

    public void ChangeTo(int val)
    {
        x = val;
    }
}

// 值类型的示例
struct FooStruct
{
    private int x;

    public void ChangeTo(int val)
    {
        x = val;
    }
}

FooClass klass = new FooClass();
FooStruct strukt = new FooStruct();

klass.ChangeTo(10);
strukt.ChangeTo(10);
```

如果逻辑地推理，那么可以很容易地得出结论，方法的主体只编译一次。即没有这样的情况，每个类型的实例都编译自己的方法集，这些方法集完全相同，但属于不同的实例。然而，被调用的方法完全知道它是为哪个实例被调用的。这是通过将实例类型作为第一个参数传递来实现的。我们可以轻松地重写我们的示例，这将与上面写的完全相同（我故意不使用虚方法的示例。它们的工作方式不同）：

```csharp

// 引用类型的示例
class FooClass 
{
    public int x;
}

// 值类型的示例
struct FooStruct
{
    public int x;
}

public void ChangeTo(FooClass klass, int val)
{
    klass.x = val;
}

public void ChangeTo(ref FooStruct strukt, int val)
{
    strukt.x = val;
}

FooClass klass = new FooClass();
FooStruct strukt = new FooStruct();

ChangeTo(klass, 10);
ChangeTo(ref strukt, 10);
```


值得解释一下，为什么我使用了关键字`ref`。如果我不使用它，就会出现这样一种情况：我通过方法参数得到了结构的**副本**而不是原件，我修改了它，但原件保持不变。我就必须将修改后的副本从方法返回给调用方（又一次复制），而调用方则需要将这个值再保存回变量中（又一次复制）。相反，实例方法接收的是指向结构的指针，通过它来进行修改。直接修改原件。注意，通过指针传递对性能没有任何影响，因为任何处理器级别的操作本来就是通过指针进行的。也就是说，ref只是C#世界中的一个概念，仅此而已。

最后一个值得一提的例子使用了在C# 9中引入的方法指针：

## 能够指定元素的位置

两种类型的类的另一种可能性是能够精确指明相对于结构体起始位置在内存中的偏移量，某个字段是如何放置的。这样做有几个原因：
  - 为了与位于非托管世界的外部API进行工作，以避免通过未使用的字段来“跳转”到所需字段；
  - 例如，可以指示编译器将某个字段确切地放置在类型的最开始位置（`[FieldOffset(0)]`）。这样会加快对它的操作速度。如果这个字段被频繁使用，那么可以在此基础上节省不少。只需注意一个重要细节。这只适用于值类型。因为在引用类型中，零偏移位置放置的是虚方法表的地址，它占用一个处理器字。即使您访问类的第一个字段，访问仍然会通过更复杂的地址（地址 + 偏移）进行。顺便说一下，这样做是有原因的：类中最常用的字段正是虚方法表的地址，因为正是通过它来调用虚方法；
  - 您可以在同一个地址上设置几个字段。那么同一个值可以被解释为不同的数据类型。在C++中，这种数据类型称为`union`；
  - 也可以不声明任何东西：编译器会按照它认为最优的方式来放置字段。即最终的字段顺序可能会有所不同；

**基本原则**

- **自动**：运行环境自动选择类或结构的所有字段的位置和打包方式。使用此枚举成员定义的结构不能被提供给托管代码之外的使用。尝试这样做会导致异常的产生；
- **显式**：程序员明确地控制对象每个字段的确切位置。每个字段必须使用FieldOffsetAttribute来指定其确切的位置；
- **顺序**：对象的成员按照设计类型时指定的顺序顺序排列。它们也根据指定的StructLayoutAttribute.Pack值进行打包步骤的排列。

**使用FieldOffset跳过结构中未使用的区域**

这里当然会产生一个问题，为什么会有根本不使用的字段。来自unmanaged世界的结构可能包含备用字段，这些字段可能在库的未来版本中被使用。如果在C/C++世界中通过添加`reserved1, reserved2, ..`字段来进行跳过，那么在.NET中，我们有一个绝佳的机会，只需使用`FieldOffsetAttribute`和`[StructLayout(LayoutKind.Explicit)]`属性，就可以简单地设置到字段开始的偏移量：

```csharp
[StructLayout(LayoutKind.Explicit)]
public struct SYSTEM_INFO
{
    [FieldOffset(0)] public ulong OemId;
    // 92字节 - 备用
    [FieldOffset(100)] public ulong PageSize;
    [FieldOffset(108)] public ulong ActiveProcessorMask;
    [FieldOffset(116)] public ulong NumberOfProcessors;
    [FieldOffset(124)] public ulong ProcessorType;
}
```

请注意，跳过的部分也是占用的，但未使用的空间。结构的大小将是`132`字节，而不是最初看起来的`40`字节。

**联合**

通过使用`FieldOffsetAttribute`，您可以模拟C/C++世界中的`union`类型。`union`是一种特殊的类型，它允许将同一数据作为不同类型的实体访问。让我们看一个模拟的例子：

```csharp
// 如果读取RGBA.Value，我们将读取一个Int32值，该值是所有其他字段的累积。
// 但是，如果我们尝试读取RGBA.R、RGBA.G、RGBA.B、RGBA.Alpha，那么我们将读取Int32数字的单独组件
[StructLayout(LayoutKind.Explicit)]
public struct RGBA
{
    [FieldOffset(0)] public uint Value;
    [FieldOffset(0)] public byte R;
    [FieldOffset(1)] public byte G;
    [FieldOffset(2)] public byte B;
    [FieldOffset(3)] public byte Alpha;
}
```

在这里，您可能会思考并说，这种行为只可能对于值类型，但这不是真的。尽管听起来很奇怪，但这种行为也可以通过在同一地址上覆盖两个引用类型或引用类型与值类型来实现：

```csharp
class Program
{
    public static void Main()
    {
        Union x = new Union();
        x.Reference.Value = "Hello!";
        Console.WriteLine(x.Value.Value);
    }

    [StructLayout(LayoutKind.Explicit)]
    public class Union
    {
        public Union()
        {
            Value = new Holder<IntPtr>();
            Reference = new Holder<object>();
        }

        [FieldOffset(0)]
        public Holder<IntPtr> Value;
        
        [FieldOffset(0)]
        public Holder<object> Reference;
    }

    public class Holder<T>
    {
        public T Value;
    }
}
```

请注意，我故意通过泛型类型进行了覆盖：在常规覆盖的情况下，在应用程序域中加载此类型时会生成`TypeLoadException`异常。实际上，这只是从外部看起来像是应用程序安全性的漏洞（特别是对于应用程序的“插件”而言），但如果我们尝试在受保护的域下运行此代码，我们将得到同样的`TypeLoadException`。

## 分配的差异

另一个重要的特性，对于两种类型来说有着根本的不同，就是为对象或结构分配内存的过程。问题在于，为了给对象分配内存，CLR必须首先回答自己一系列问题。首先，对象的大小是多少？它小于还是大于85K字节？如果小于，那么Small Objects Heap中剩余的空间是否足够放置该对象？如果不够，就会启动垃圾回收，它实际上需要首先遍历对象图，然后尝试通过填充新对象到占用空间之间的空隙来进行。如果没有这样的空隙，就移动对象以释放空间。如果在这个操作之后SOH中仍然没有空间（例如，没有释放任何东西），那么就会启动一个过程来分配额外的虚拟内存页面，以增加Small Objects Heap的大小。只有在所有这些步骤完成之后，才会为对象分配空间，分配的内存区域会被清除垃圾（归零），标记SyncBlockIndex和VirtualMethodsTable，然后对象的引用返回给用户。

如果分配的对象大小超过85K，那么我们就涉及到Large Objects Heap。这通常是巨大字符串和数组的情况。在这种情况下，我们需要从已释放的内存列表中找到最合适的内存块，如果没有，则分配一个新的区块。这些程序默认不是很快，但我们假设对于这种大小的对象，我们会特别小心地处理，它们在这次讨论的背景下不是主要关注点。

也就是说，对于RefTypes，我们有几种情况：

  - RefType的大小 < 85K，SOH中有空间：内存分配相对较快；
  - RefType的大小 < 85K，SOH中的空间不足：内存分配可能非常慢；
  - RefType的大小 > 85K，内存分配相对较慢。考虑到这些操作很少见，并且由于它们的大小，它们不能与ValTypes竞争，这现在并不是我们非常关心的问题。

那么，为值类型分配内存的算法是什么呢？实际上并没有这样的算法。为值类型分配内存根本不需要任何成本。唯一发生的就是字段的归零。让我们来理解为什么会这样：

1. 在方法体内声明变量的情况下，为结构分配空间的时间可以认为是接近零的。因为为局部变量分配空间的时间几乎不依赖于它们的数量；
2. 在将ValTypes作为RefTypes的字段放置的情况下，只会增加它们的大小。值类型被完整地放置，成为其一部分；
3. 如果ValTypes作为方法参数传递——在这里，就像在复制的情况下，会有一些差异——这取决于参数的大小和位置。

但无论如何，这都不会比从一个变量复制到另一个变量更长。

## 在class/struct之间选择的特点

让我们思考一下这两种类型的特点，它们的优点和缺点，并决定在哪里最好使用它们。这里，当然，值得回顾经典观点，即如果我们的类型不打算被继承，它在其生命周期内不会改变，且其大小不超过16字节，那么选择值类型是明智的。但事情并不那么明显。为了进行全面比较，我们需要从不同角度考虑类型选择，心理上思考其未来使用的场景。我建议将选择标准分为三组：

- 从您的类型将要互动的类型系统的架构角度来看；
- 从您作为系统程序员的角度来看：从性能的角度来看，哪种选择是最优的；
- 另外，别无选择。

每一个由您设计的实体都应该充分反映其用途。这不仅涉及其名称或交互界面（方法、属性），甚至在选择值类型与引用类型之间也可能基于架构考虑。让我们来思考一下，为什么从系统类型架构的角度可能会选择结构而不是类：

1. 如果我们设计的类型对其状态的语义负载具有不变性，这意味着其状态完全反映了某个过程或是某个值的表现。换句话说，类型的实例是完全常量的，其本质不能被改变。我们可以基于这个常量创建另一个类型的实例，指定一些偏移，或者从零开始创建，指定其属性。但我们没有权利去改变它。请注意，我并不是说结构是不可变类型。您可以随意更改字段。更重要的是，您可以通过`ref`参数将结构的引用传递给方法，并在方法返回时获取修改后的字段。然而，我是从架构的角度讲述其意义。我将通过例子来说明：

- DateTime是一个封装了时间点概念的结构体。它以`uint`的形式存储这些数据，但提供了访问时间点的各个特征的能力，例如：年、月、日、小时、分钟、秒、毫秒，甚至是处理器的时钟周期。然而，基于它所封装的内容，它本质上是不可变的。我们无法改变一个具体的时间点使其变成另一个时间点。我不能活在我的生命中的下一分钟，以体验我童年最好的生日。时间是不可变的。这就是为什么数据类型的选择可能是一个具有只读接口的类，它在每次属性变化时返回一个新的实例，或者是一个结构体，尽管它有能力改变其实例的字段，但按其定义不应该这样做：时间点的描述是一个*值*。就像一个数字。你不能进入一个数字的结构并改变它，对吧？如果你想要得到一个相对于原始时间偏移了一天的另一个时间点，你只需获取结构体的一个新实例；
- KeyValuePair<TKey, TValue>是一个封装了键值对概念的结构体。值得注意的是，这个结构体仅用于在遍历字典内容时向用户返回。为什么从架构的角度选择了结构体？答案很简单：因为在Dictionary<T>中，键和值是不可分割的概念。是的，内部的实现方式不同。内部我们有一个复杂的结构，其中键和值是分开存储的。然而，对于外部用户来说，从交互接口和数据结构的意义上讲，键值对是一个不可分割的概念。它是一个完整的*值*。如果我们将另一个值放在这个键下，这意味着整个对都改变了。对于外部观察者来说，没有单独的键或单独的值，它们是一个整体。这正是为什么在这种情况下结构体是理想的选择。

2. 如果我们设计的类型是外部类型的不可分割的一部分，但在结构上是不可分割的。也就是说，说外部类型引用了被封装实例是不正确的，但完全正确的是，被封装的是外部的一个完整部分，包括它的所有属性。通常，这在设计作为另一结构部分的结构时使用。

   - 例如，如果我们考虑文件头的结构，从一个文件到另一个文件提供一个链接是不公平的。比如说，头文件位于`header.txt`中。这在将文档插入到某个其他文档中时是合适的，但不是通过文件系统的相对链接来嵌入文件。Windows操作系统的快捷方式文件就是一个好例子。然而，如果我们谈论文件头（例如，JPEG文件的头，其中指定了图像大小、压缩技术、拍摄参数、GPS坐标和其他元信息），那么在设计用于解析头的类型时，使用结构将非常有用。因为，一旦所有头部都在结构中描述，你就会在内存中得到与文件中所有字段完全相同的位置。并且通过简单的unsafe转换`*(Header *)readedBuffer`，无需任何反序列化操作，就能完全填充数据结构。

3. 同时，请注意，这些例子中没有一个具有继承任何东西行为的属性。更重要的是，这些例子还表明，继承这些实体的行为完全没有任何意义。它们作为某物的单元是完全自足的。

如果我们从代码工作效率的角度来看这个问题，那么我们面临的选择将从另一个角度呈现：

1. 如果需要从非托管代码中获取一些结构化数据，或者需要向unsafe方法传递数据结构，那么选择结构体是必要的。引用类型完全不适合这种情况；
2. 如果某个类型将频繁用于方法调用中传递数据（无论是作为返回值还是方法参数），但同时不需要从不同位置引用同一个值，那么您的选择应该是结构体。例如，我可以提到元组。如果一个方法通过元组返回给您多个值，这意味着它将返回ValueTuple，它被声明为结构体。也就是说，在返回时，方法不会在堆上分配内存，而是使用线程堆栈，其内存分配对您来说完全是免费的；
3. 如果您正在设计一个系统，该系统创建大量设计类型的实例。同时，这些实例的大小相当小，生命周期非常短，那么使用引用类型将导致使用对象池，或者，如果不使用池，将导致堆的不受控制的污染。同时，一部分对象将进入老年代，这将导致GC的性能下降。在这些情况下使用值类型（如果可能的话）将提高性能，仅仅因为在SOH中不会有任何东西，这将减轻GC的负担，算法将运行得更快；

综合以上所述，我可以提供一些建议和备注关于结构体的使用：

1. 在选择集合时，应避免使用包含大型结构的大型数组。这也适用于基于数组的数据结构（它们占了大多数）。这可能导致进入大对象堆并引起其碎片化。仅仅计算出如果你的结构有4个byte类型的字段，那么它将占用4个字节是不够的。事实并非如此。需要理解的是，对于32位系统，结构的每个字段都会按4个字节对齐（每个字段的地址必须能被4整除），而在64位系统上，则按8个字节对齐。也就是说，数组的大小应该取决于结构的大小和运行应用程序的平台。在我们的例子中，对于4个字节 - 85K / (每个字段4到8字节 * 字段数量 = 4) 减去数组头部的大小：根据平台不同，大约可容纳2600个元素（当然，应该取较小的数值）。仅此而已！并不多！而你可能会认为，20,000个元素的魔法常数似乎非常合适！

2. 同样，应该意识到，如果你使用一个相当大的结构作为数据源，并将其作为字段放在某个类中，同时，例如，同一个副本被复制了一千次（仅仅因为这样做方便），那么你实际上增加了每个类实例的大小，这最终会导致0代膨胀并进入第1代甚至第2代。同时，如果类的实例实际上是短暂的，并且你希望它们在第0代被GC回收——在1毫秒内，那么你会对它们实际上进入了第一代甚至第二代感到非常失望。那么，实际上有什么区别呢？区别在于，如果第0代在1毫秒内被回收，那么第一代和第二代的回收速度会非常慢，这将导致无谓的性能下降；

3. 出于大致相同的原因，应避免通过方法调用链传递大型结构。因为如果所有人都开始相互调用，那么这样的调用将在栈中占用更多空间，将你的应用程序的生命推向通过`StackOverflowException`的死亡。第二个原因是性能。复制越多，一切运行得越慢；

因此，总的来说，在数据类型之间做选择是一个相当非平凡的过程。这往往可能涉及到过早优化，这是不推荐的做法。然而，如果你知道你的情况符合上述原则，那么你可以放心地选择一个重要的类型。

## 基本类型 -- Object 和实现接口的可能性。装箱。

我们似乎经历了火和水，可以通过任何面试。甚至可能加入.NET CLR团队。但让我们不要急着去microsoft.com搜索职位空缺部分：我们还有时间。让我们先回答这样一个问题。如果值类型不包含对SyncBlockIndex的引用，也不包含指向虚拟方法表的指针...那么，对不起，它们是如何继承`object`类型的呢？毕竟，按照所有规则，任何类型都是继承它的。不幸的是，这个问题的答案不能用一句话来概括，但它将给我们的类型系统带来如此深刻的理解，以至于最后的拼图终于会落到位。

那么，让我们再次回顾一下值类型在内存中的布局。无论它们位于何处，它们都被嵌入到它们所在的位置。它们成为了那个地方的一部分。与引用类型不同，后者的法则是必须位于小对象或大对象的堆中，而设置值的位置总是指向堆中我们对象所在位置的引用。

如果仔细思考，会发现任何有意义的类型都有 `ToString`、`Equals` 和 `GetHashCode` 方法，这些方法是虚拟的、可以被重写的，但是我们不能继承有意义的类型并重写这些方法。为什么呢？因为如果让有意义的类型拥有可以被重写的方法，那么它们就需要一个虚拟方法表来进行调用的路由。这反过来会导致将结构传递到非托管世界时出现问题：会有额外的字段被传递。最终，这意味着有意义类型的方法描述存在某处，但通过虚拟方法表无法直接访问它们。

这让人思考，缺乏继承是人为的：

- 从 object 继承是存在的，虽然不是直接的；
- 在基类型中有 ToString、Equals 和 GetHashCode，它们在有意义的类型中以不同的方式工作：这些方法在每个类型中都有自己的行为。这意味着，相对于 object，这些方法被重写了；
- 更进一步，如果你将类型转换为 `object`，你仍然可以完全有权调用 ToString、Equals 和 GetHashCode。
- 在对有意义的类型进行实例方法调用时，不会发生复制到方法中的操作。即，调用实例方法类似于调用静态方法：`Method(ref structInstance, newInternalFieldValue)`。实际上，这是一种传递 `this` 的调用，但有一个例外：JIT 必须构建方法体，以避免对结构体字段进行额外的偏移，跳过不存在于结构体本身中的虚拟方法表指针。*对于有意义的类型，它位于其他地方*。

也就是说，在某种意义上，我们并不是被欺骗，但有所保留：类型在行为上有很大的不同，但在 CLR 的实现层面，它们之间的差异并不那么显著。但我们稍后再详细讨论这个问题。

如果我们在程序中写下以下行：

``` csharp
var obj = (object)10;
```

当我们不再处理数字`10`时，将发生所谓的装箱（boxing）过程，即打包。也就是说，我们开始有能力通过基类来操作它。如果我们获得了这样的能力，这意味着我们可以访问VMT（虚方法表），通过它可以轻松调用ToString()、Equals和GetHashCode这些虚拟方法。而且，由于原始值可以存储在任何地方：无论是在栈上，还是作为类的字段，而将其转换为`object`类型，我们就有可能永久存储对这个数字的引用。因此，实际上装箱创建了一个值类型的副本，而不是对原始值的指针。也就是说，当发生装箱时：

  - CLR在堆上为值类型的结构+SyncBlockIndex+VMT分配空间（以便能够调用ToString, GetHashCode, Equals）；
  - 然后将值类型的实例复制到那里。

女士们和先生们。在体面的社会中，这样的话不太合适，但我们得到了一个值类型的引用版本。我再重复一遍：通过装箱，结构获得了**与引用类型完全相同的系统字段集合**，成为了一个完整的引用类型。结构变成了类。让我们称这一现象为Dotnet的空翻。我认为这个名称非常适合这种巧妙的转变。

顺便说一下，为了让你们相信我的话，只需要理解当你使用实现了某个接口的结构时会发生什么——通过这个接口。

```csharp

struct Foo : IBoo
{
    int x;
    void Boo()
    {
        x = 666;
    }
}

IBoo boo = new Foo();

boo.Boo();

```

因此，当Foo的实例被创建时，它的值实际上位于栈上。之后，我们将这个变量放入接口类型的变量中。将结构放入引用类型的变量中。发生了`boxing`。很好。我们得到了`object`类型。但我们的变量是接口类型的。这意味着需要进行类型转换。也就是说，调用可能是这样发生的：

```csharp

IBoo boo = (IBoo)(box_to_object)new Foo();
boo.Boo();

```

编写这样的代码是极其低效的。不仅仅是因为你会改变一个副本而不是原始对象：

```csharp
void Main()
{
    var foo = new Foo();
    foo.a = 1;
    Console.WriteLine(foo.a);  // -> 1
    
    IBoo boo = foo;
    boo.Boo();                 // 看起来像是将 foo.a 改为了 10
    Console.WriteLine(foo.a);  // -> 1
}

struct Foo : IBoo 
{
    public int a;
    public void Boo()
    {
        a = 10;
    }
}

interface IBoo 
{
    void Boo();
}
```

这看起来像是两次欺骗。首先，看这段代码我们不必知道我们在处理*外部*代码中的什么，看到下面的转换到接口`IBoo`，实际上这让我们认为Foo是一个类，而不是一个结构体。接着，完全没有视觉上的区分结构体和类，让人完全感觉通过接口修改的结果应该反映在foo上，但实际上没有，因为boo是foo的一个副本。这实际上是误导。在我看来，这样的代码应该加上注释，以便外部开发者能够正确理解它。

第二个观察，与我们之前的讨论有关，即我们可以将`object`类型转换为`IBoo`。这是另一个证明，即装箱的值类型并不特殊，实际上是值类型的引用版本。或者从另一个角度看，系统中的所有类型都是引用类型。只是对于结构体，我们可以像处理值类型那样工作，完整地“卸载”它们的值。就像在C++世界中所说的，解引用对象指针。

但你可能会反驳：如果真的像我说的那样，那么我们可以这样写：

```csharp
var referenceToInteger = (IInt32)10;
```

我们就会得到不仅仅是一个`object`，而是对装箱值类型的类型化引用。但这样做将破坏值类型的整个概念，朋友们。而值类型的主要概念是它们值的完整性，这允许基于它们的属性进行出色的优化。所以我们不要坐以待毙！让我们来破坏这个概念吧！

```csharp
public sealed class Boxed<T>
{
    public T Value;
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public override bool Equals(object obj)
    {
        return Value.Equals(obj);
    }
        
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public override string ToString()
    {
        return Value.ToString();
    }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public override int GetHashCode()
    {
        return Value.GetHashCode();
    }
}
```

我们刚刚得到了什么？我们得到了一个完全等同于装箱的模拟。但现在，我们可以通过调用其实例方法来更改其内容。而且，任何拥有对这个数据结构的引用的人都会接收到这些更改。

```csharp
var typedBoxing = new Boxed<int> { Value = 10 };
var pureBoxing = (object)10;
```

必须承认，第一个选项看起来有些不那么自信。我们在做一些不太清楚的事情，而不是进行常规的类型转换。第二行则简洁如日本诗歌。然而，它们实际上几乎完全相同。唯一的区别在于，常规装箱在堆内存分配后不会进行零内存清理：内存立即被所需的结构占用。而在第一个选项中，存在清理过程。仅因为这个原因，我们的版本比常规装箱慢了10%。

但现在，我们可以调用我们的装箱值的某些方法：

```csharp
struct Foo
{
    public int x;

    public void ChangeTo(int newx)
    {
        x = newx;
    }
}

var boxed = new Boxed<Foo> { Value = new Foo { x = 5 } };
boxed.Value.ChangeTo(10);
var unboxed = boxed.Value;
```

我们获得了一个新工具，但还不知道如何使用它。让我们通过推理来找到答案：

- 我们的`Boxed<T>`类型实际上做的和常规的一样：在堆中分配内存，将值传递给它，并允许通过执行一种特殊的`unbox`来取回它；
- 同样，如果失去了对装箱结构的引用，GC会收集它；
- 然而，现在我们有了与装箱类型工作的能力：调用它的方法；
- 同时，我们现在有机会将SOH/LOH中的值类型实例替换为另一个。我们之前无法做到这一点：我们必须进行`unboxing`，将结构更改为另一个，然后进行`boxing`回去，并向消费者分发新的引用。


我们也来思考一下，包装的主要问题是什么？它在内存中创建了流量。这些流量包含了数量不定的对象，其中一部分可能存活到第一代，在那里我们会遇到垃圾收集的问题：垃圾收集会存在，而且会很多，这明显是可以避免的。而当我们面对短命对象的流量时，首先想到的解决方案就是池化。这将是Dotnet空翻的完美收尾。

```csharp
var pool = new Pool<Boxed<Foo>>(maxCount:1000);
var boxed = pool.Box(10);
boxed.Value=70;

// 在这里使用boxed值

pool.Free(boxed);
```

也就是说，我们通过池实现了包装的工作能力，从而将包装的内存流量减少到零。开个玩笑，我们甚至可以让对象在finalize方法中复活自己，把自己重新放回对象池。这对于`boxed`结构被传递到外部异步代码中，无法知道何时不再需要它的情况非常有用。在这种情况下，它会在GC期间自动将自己返回到池中。

现在让我们总结一下：

  - 如果包装是偶然的，本不应该发生，那么请小心，不要让它发生：它可能导致性能问题；
  - 如果包装是您正在构建的系统架构要求的一部分，那么可能有几种选择：如果包装的结构流量很小，不明显，可以不用担心，继续通过包装工作。但如果流量变得明显，那么可能应该通过上面指出的解决方案进行包装的池化。是的，这会带来一些性能开销，但至少GC会平静，不会过度工作；

最后，让我们来看一个完全不实用的代码示例

{.wide}
```csharp
static unsafe void Main()
{
    // 创建一个boxed int
    object boxed = 10;

    // 获取指向VMT的指针的地址
    var address = (void**)EntityPtr.ToPointerWithOffset(boxed);

    unsafe
    {
        // 获取Virtual Methods Table的地址
        var structVmt = typeof(SimpleIntHolder).TypeHandle.Value.ToPointer();

        // 将一个进入Heap的整数的VMT地址更改为SimpleIntHolder的VMT地址，将Int转换为结构体
        *address = structVmt;
    }
```

```csharp
var structure = (IGetterByInterface)boxed;

Console.WriteLine(structure.GetByInterface());
}

interface IGetterByInterface
{
    int GetByInterface();
}

struct SimpleIntHolder : IGetterByInterface
{
    public int value;

    int IGetterByInterface.GetByInterface()
    {
        return value;
    }
}
```

这段代码是通过一个小功能实现的，它能够从对象引用中获取指针。该库位于[github上的这个地址](https://github.com/mumusan/dotnetex/blob/master/libs/)。这段代码展示了，普通的装箱操作如何将int转换为类型化的引用类型。让我们逐步来看：

  1. 对一个整数进行装箱
  2. 获取得到的对象的地址（该地址处是Int32的VMT）
  3. 获取SimpleIntHolder结构的VMT
  4. 将装箱的整数的VMT替换为SimpleIntHolder结构的VMT
  5. 将其拆箱为结构类型
  6. 输出字段值——即最初被装箱的那个Int32。

注意，我是故意通过接口来做这个的，以展示这样也是可行的。

### Nullable\<T>

还值得一提的是，对于Nullable值类型的`装箱`行为。这是Nullable值类型一个非常令人愉快的特性，因为装箱一个本质上为null的值类型，只会简单地返回null：

```csharp
int? x = 5;
int? y = null;

var boxedX = (object)x; // -> 5
var boxedY = (object)y; // -> null
```

由此得出一个有趣的结论：因为null没有类型，所以从装箱中提取出的类型，唯一的方式是提取一个与原始类型不同的类型，如下所示：

```csharp
int? x = null;
var pseudoBoxed = (object)x;
double? y = (double?)pseudoBoxed;
```

这段代码之所以能工作，仅仅是因为可以将null转换为任何类型。但这样的代码绝对能让你的同事感到惊讶，并用一个有趣的事实来调节夜晚的气氛。

## 我们更深入地探索装箱

作为小吃，我想给你们讲讲[System.Enum](http://referencesource.microsoft.com/#mscorlib/system/enum.cs,36729210e317a805)类型。不通过点击链接，请告诉我，这是值类型（Value Type）还是引用类型（Reference Type）？根据所有的规则和逻辑，这个类型理应是一个值类型。因为它只是一个普通的枚举：一个从数字到编程语言中名称的别名集合。我们本可以就此结束讨论，说：“你们全都想得完全正确！不浪费时间，让我们继续前进！”，但情况并非如此。`System.Enum`，所有`enum`数据类型的基类，无论是在你的代码中还是在.NET Framework中定义的，都是一个引用类型。也就是说，它是一个`class`数据类型。而且（！）这个类是抽象的，并且继承自`System.ValueType`。

```csharp
    [Serializable]
    [System.Runtime.InteropServices.ComVisible(true)]
    public abstract class Enum : ValueType, IComparable, IFormattable, IConvertible
    {
        // ...
    }
```

这是否意味着所有的枚举都在SOH堆中分配？这是否意味着，使用枚举时，我们会填满堆，以及随之而来的GC（垃圾回收）？因为我们只是在使用它们。不，不可能是这样。那么，看来是不是有一个枚举池，在那里它们被存放，而我们得到的是它们的实例。又或者不是：枚举可以在结构体中用于封送处理。枚举就是普通的数字。

但事实是，CLR在构建数据类型的结构时，如果遇到`enum`，会[将类转换为值类型](https://github.com/dotnet/coreclr/blob/4b49e4330441db903e6a5b6efab3e1dbb5b64ff3/src/vm/methodtablebuilder.cpp#L1425-L1445)：

```csharp
// 检查类是否为值类型；但我们不想将System.Enum标记为ValueType。
// 为了实现这一点，检查利用了System.ValueType和System.Enum按顺序紧接着加载的事实，
// 因此，如果父MethodTable是System.ValueType并且System.Enum MethodTable未设置，
// 那么我们必须在构建System.Enum，因此我们不将其标记为ValueType。
if(HasParent() &&
    ((g_pEnumClass != NULL && GetParentMethodTable() == g_pValueTypeClass) ||
    GetParentMethodTable() == g_pEnumClass))
{
    bmtProp->fIsValueClass = true;

    HRESULT hr = GetMDImport()->GetCustomAttributeByName(bmtInternal->pType->GetTypeDefToken(),
                                                            g_CompilerServicesUnsafeValueTypeAttribute,
                                                            NULL, NULL);
    IfFailThrow(hr);
    if (hr == S_OK)
    {
        SetUnsafeValueClass();
    }
}
```

为什么要这么做？特别是因为继承的概念：为了创建自定义的`enum`，需要提供关于其可能值的名称的信息，例如。那么如何在值类型上实现继承呢？无法实现。因此，他们想出了让它在引用时是引用类型，但在JIT（即时编译）阶段将其转换为值类型。这样做是为了让没人能够猜到。

## 如果想亲自看看装箱是如何工作的呢？

在我们这个美好的时代，为了查看值类型的装箱实现，幸运的是，没有必要加载反汇编器并深入到难以理解的内部。在我们这个美好的时代，我们有.NET平台核心的所有源代码，以及.NET Framework CLR和CoreCLR之间的许多部分是完全相同的。您可以通过以下链接查看源代码中的装箱实现：

- 存在一组针对不同处理器类型使用的优化，每种优化都是独立的：
  - *[JIT_BoxFastMP_InlineGetThread](https://github.com/dotnet/coreclr/blob/master/src/vm/amd64/JitHelpers_InlineGetThread.asm#L86-L148)*（AMD64 -- 多处理器或Server GC，隐式线程本地存储）
  - *[JIT_BoxFastMP](https://github.com/dotnet/coreclr/blob/8cc7e35dd0a625a3b883703387291739a148e8c8/src/vm/amd64/JitHelpers_Slow.asm#L201-L271)*（AMD64 -- 多处理器或Server GC）
  - *[JIT_BoxFastUP](https://github.com/dotnet/coreclr/blob/8cc7e35dd0a625a3b883703387291739a148e8c8/src/vm/amd64/JitHelpers_Slow.asm#L485-L554)*（AMD64 -- 单处理器或Workstation GC）
  - *[JIT_TrialAlloc::GenBox(..)](https://github.com/dotnet/coreclr/blob/38a2a69c786e4273eb1339d7a75f939c410afd69/src/vm/i386/jitinterfacex86.cpp#L756-L886)*（x86），通过JitHelpers加入
- 一般情况下，JIT内联调用辅助函数[Compiler::impImportAndPushBox(..)](https://github.com/dotnet/coreclr/blob/a14608efbad1bcb4e9d36a418e1e5ac267c083fb/src/jit/importer.cpp#L5212-L5221)
- 通用版本，使用较少优化的[MethodTable::Box(..)](https://github.com/dotnet/coreclr/blob/master/src/vm/methodtable.cpp#L3734-L3783)
  - 最后调用[CopyValueClassUnchecked(..)](https://github.com/dotnet/coreclr/blob/master/src/vm/object.cpp#L1514-L1581)，通过其代码可以清楚地看到，为什么最好选择大小最多包含8字节的结构。

为了进行比较，实现了一个用于解包的方法：*[JIT_Unbox(..)](https://github.com/dotnet/coreclr/blob/03bec77fb4efaa397248a2b9a35c547522221447/src/vm/jithelpers.cpp#L3603-L3626)*，它是围绕*[JIT_Unbox_Helper(..)](https://github.com/dotnet/coreclr/blob/03bec77fb4efaa397248a2b9a35c547522221447/src/vm/jithelpers.cpp#L3574-L3600)*的一个包装。

同样[一个有趣的事实是](https://stackoverflow.com/questions/3743762/unboxing-does-not-create-a-copy-of-the-value-is-this-right)，解包过程本身并不是从堆中复制数据：解包是传递一个带有类型兼容性检查的堆指针。而接下来的IL操作码将决定如何处理这个地址：数据可能会被复制到局部变量中，或者可能会被复制到栈中以调用方法。因为如果不是这样，我们就会面临双重复制的情况：首先是从堆中某处复制，然后是复制到目的地。

## 相关问题部分

### 为什么.NET CLR不自己对装箱进行池化？

如果我们与任何Java开发者交谈，我们会惊讶地发现两件事：

  - 在Java中，所有的值类型都被装箱，本质上并不是值类型。即，整数也是被装箱的；
  - 为了优化，-128到127之间的数字是从对象池中获取的。

那么为什么在.NET CLR中装箱时不做同样的事情呢？答案很简单：因为我们可以改变装箱值类型的内容。换句话说，我们可以这样做：

```csharp
object x = 1;
x.GetType().GetField("m_value", BindingFlags.Instance | BindingFlags.NonPublic).SetValue(x, 138);
Console.WriteLine(x); // -> 138
```

或者这样（C++/CLI）：

```cpp
void ChangeValue(Object^ obj)
{
    Int32^ i = (Int32^)obj;
    *i = 138;
}
```

即，如果我们处理的是池化，那么我们就会将应用程序中的所有1都改为138！你必须承认，这不是一个愉快的情况。

第二个原因是.NET中值类型的本质。它们是基于值工作的，这意味着更快。对我们来说，装箱是一个罕见的操作，而且装箱数字的加法更像是幻想和非常糟糕的架构。这根本不会是实现该功能的有用之处。

### 为什么在调用接受 object 类型参数的方法时，实际上是值类型，不能在栈上进行装箱，从而减轻堆的负担？

因为如果在栈上对值类型进行装箱，然后引用被传递到堆中，那么在方法内部，这个引用可能会被传递到其他地方，例如方法可能会将传入的引用写入某个类的字段。接着方法执行完毕，随后执行装箱的方法也结束了。结果是 —— 从外部类看，这个引用会指向一个“死亡”的栈区域。

### 为什么不能使用值类型本身作为字段？

有时候会有大胆的想法，想在一个结构体的字段中使用另一个结构体，而这个结构体又使用了第一个。或者更简单：在结构体的字段中使用它自己。不要问我为什么会有这种需求：实际上是没有的 :) 但为什么这样做不行呢？答案很简单：当你在使用一个结构体作为它自己的字段，或者通过依赖另一个结构体时，实际上你创建了一个递归：你创建了一个无限大小的结构体。然而，在 .NET Framework 中有这样做的地方。例如 `System.Char` 结构体就是一个例子，[它包含了它自己](http://referencesource.microsoft.com/#mscorlib/system/char.cs,02f2b1a33b09362d)：

```csharp
public struct Char : IComparable, IConvertible
{
    // 成员变量
    internal char m_value;
    //...
}
```

顺便说一下，几乎所有的 CLR 原始类型都是这样做的。对我们凡人来说这是不可能的，也没有这个必要：在 CLR 中这样做是为了给原始类型增加面向对象编程的特性。