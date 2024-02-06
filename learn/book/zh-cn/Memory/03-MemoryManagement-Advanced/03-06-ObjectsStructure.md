# 内存中对象的结构

到目前为止，当我们讨论值类型和引用类型之间的差异时，我们是从最终开发者的角度来探讨这个话题的。也就是说，我们从未深入探讨它们实际上是如何构建的，以及在CLR级别上实现了哪些技巧。我们实际上是在观察最终结果，并从研究黑盒的角度进行推理。然而，为了更深入地理解事物的本质，以及为了抛开关于CLR内部发生的任何魔法的最后一丝想法，值得深入探究其内部结构并研究那些管理类型系统的算法。

## 类型实例的内部结构

在开始讨论类型系统控制块的结构之前，让我们先看看对象本身，任何类的实例。如果我们在内存中创建任何引用类型的实例，无论是类还是打包的结构，它将由三个字段组成：SyncBlockIndex（实际上不仅仅是它），类型描述符的指针和数据。数据区域可以包含很多字段，但为了不失一般性，在下面的示例中，我们假设数据中只包含一个字段。因此，如果我们以图形方式表示这个结构，我们将得到以下内容：

**System.Object**

```
  ----------------------------------------------
  |  SyncBlkIndx |   VMT_Ptr    |     Data     |
  ----------------------------------------------
  |  4 / 8 字节  |  4 / 8 字节  |  4 / 8 字节  |
  ----------------------------------------------
  |  0xFFF..FFF  |  0xXXX..XXX  |      0       |
  ----------------------------------------------
                 ^
                 | 对象的引用不是指向开头，而是指向VMT

  总大小 = 12 (x86) | 24 (x64)
```

也就是说，类型实例的实际大小取决于应用程序将运行的最终平台。

接下来，让我们按照 `VMT_Ptr` 指针继续查看，看看这个地址下有什么数据结构。对于整个类型系统来说，这个指针是最重要的：正是通过它来实现继承、接口实现、类型转换等等功能。这个指针是对 .NET CLR 类型系统的引用，是对象的“护照”，CLR 通过它来进行类型转换，理解对象占用的内存大小，正是通过它，GC 能够巧妙地遍历对象，确定哪些地址存放的是指向对象的指针，哪些是普通数字。正是通过它，我们可以了解对象的所有信息，并使 CLR 以不同的方式处理它。因此，我们将专注于它。

### 虚方法表结构

表的描述可以在 [GitHub CoreCLR](https://github.com/dotnet/coreclr/blob/master/src/vm/methodtable.h) 的地址找到，如果去掉所有不必要的部分（那里有4381行代码），[它看起来是这样的](https://github.com/dotnet/coreclr/blob/master/src/vm/methodtable.h#L4099-L4114)：

> 这是来自 CoreCLR 的版本。如果查看 .NET Framework 中字段的结构，那么字段的布局和系统信息的两个位字段 `m_wFlags` 和 `m_wFlags2` 中各个位的布局将会有所不同。

```cpp
    // 低 WORD 是数组和字符串类型的组件大小（HasComponentSize() 返回 true）。
    // 否则用于标志。
    DWORD m_dwFlags;

    // 在堆上分配此类实例的基本大小
    DWORD m_BaseSize;

    WORD  m_wFlags2;

    // 如果类标记适合16位，则为类标记。如果这是 (WORD)-1，则类标记存储在 TokenOverflow 可选成员中。
    WORD  m_wToken;

    // <NICE> 在正常情况下，我们不需要为每个这些占用一个完整的字 </NICE>
    WORD  m_wNumVirtuals;
    WORD  m_wNumInterfaces;
```

您会同意，这看起来有些吓人。而且，吓人的不是这里只有6个字段（其他的字段都去哪了？），而是为了达到这些字段，需要跳过4100行逻辑代码。我个人本来期待在这里看到一些现成的东西，不需要额外计算的。然而，情况并非如此简单：因为任何类型的方法和接口的数量可能各不相同，所以VMT表的大小也是变化的。这反过来意味着，为了填充它，需要计算出所有其余字段的位置。但让我们不要灰心，尝试立即从我们已经拥有的东西中获益：到目前为止，我们不知道其他字段是什么（除了最后两个），但是`m_BaseSize`字段看起来很吸引人。正如注释所提示的，这是类型实例的实际大小。我们刚刚找到了类的`sizeof`！我们试试看吧？

因此，要获取VMT的地址，我们可以采取两种方法：一种是复杂的，通过获取对象的地址，也就是VMT的地址：

```csharp
class Program
{
    public static unsafe void Main()
    {
        Union x = new Union();
        x.Reference.Value = "Hello!";

        // 第一个字段是指向VMT指针所在位置的指针
        // - (IntPtr*)x.Value.Value - 将数字转换为指针（为编译器更改类型）
        // - *(IntPtr*)x.Value.Value - 通过对象地址获取VMT地址
        // - (void *)*(IntPtr*)x.Value.Value - 转换为指针
        void *vmt = (void *)*(IntPtr*)x.Value.Value;

        // 在控制台输出VMT地址；
        Console.WriteLine((ulong)vmt);
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

或者简单的方法，使用.NET FCL API：

```csharp
    var vmt = typeof(string).TypeHandle.Value;
```

第二种方法当然更简单（尽管它的运行时间更长）。然而，从理解类型实例结构的角度来看，了解第一种方法非常重要。使用第二种方法增加了一种自信感：如果我们调用API方法，那么我们似乎是在使用一个有文档记录的处理VMT的方式，而如果我们通过指针来获取，那就不是了。但是，不应忘记，存储`VMT *`对于几乎任何面向对象编程语言和整个.NET平台来说都是标准的：这个引用总是位于同一个位置，作为类中最常用的字段。而类中最常用的字段应该放在第一位，这样可以无偏移地进行寻址，结果是速度更快。因此，我们得出结论，在类的情况下，字段的位置不会影响速度，但在结构中，最常用的字段可以放在第一位。当然，对于绝大多数.NET应用程序来说，这实际上没有任何效果：这个平台不是为了这样的任务而创建的。

让我们从实例大小的角度来研究类型结构的问题。我们需要的不仅仅是抽象地研究它们（这实在是太无聊了），而是额外尝试从中提取一些好处，这是用常规方法无法提取的。

为什么有值类型的sizeof，但没有引用类型的sizeof？
实际上，这个问题是开放的，因为没有人阻止我们计算引用类型的大小。唯一可能遇到的问题是两种引用类型：`Array`和`String`的大小不是固定的。还有`Generic`组，完全取决于具体的变体。也就是说，我们不能仅用`sizeof(..)`操作符来处理：需要与具体的实例一起工作。然而，没有人阻止CLR团队实现一个类似`static int System.Object.SizeOf(object obj)`的方法，它可以轻松简单地返回我们所需要的信息。那么为什么微软没有实现这个方法呢？有一个想法是，.NET平台在他们看来——又不是一个开发者会非常关心具体字节的平台。如果有需要，可以简单地在主板上添加更多内存条。尤其是我们实现的大多数数据类型，并不占用那么大的空间。

但让我们不要分心。因此，要获取固定大小类实例的大小，只需编写以下代码：

```csharp
unsafe int SizeOf(Type type)
{
    MethodTable *pvmt = (MethodTable *)type.TypeHandle.Value.ToPointer();
    return pvmt->Size;
}

[StructLayout(LayoutKind.Explicit)]
public struct MethodTable
{
    [FieldOffset(4)]
    public int Size;
}

class Sample
{
    int x;
}

// ...

Console.WriteLine(SizeOf(typeof(Sample)));
```

那么，我们刚才做了什么？第一步，我们获得了虚拟方法表的指针。之后，我们读取大小并得到`12`——这是32位平台上`SyncBlockIndex + VMT_Ptr + x字段`大小的总和。如果我们尝试不同的类型，我们将得到大致如下的表格，对于x86：

类型或其定义 | 大小 | 注释
----------------|------|----------------
Object | 12 | SyncBlk + VMT + 空字段
Int16 | 12 | 装箱Int16：SyncBlk + VMT + 数据（按4字节对齐）
Int32 | 12 | 装箱Int32：SyncBlk + VMT + 数据
Int64 | 16 | 装箱Int64：SyncBlk + VMT + 数据
Char | 12 | 装箱Char：SyncBlk + VMT + 数据（按4字节对齐）
Double | 16 | 装箱Double：SyncBlk + VMT + 数据
IEnumerable | 0 | 接口没有大小：需要使用obj.GetType()获取
List\<T> | 24 | List<T>中元素数量无关紧要，因为它总是占用相同的空间，因为它在数组中存储数据，这个数组的大小不计入
GenericSample\<int> | 12 | 正如您所见，泛型也能正确计算大小。大小没有变化，因为数据位于与装箱int相同的位置。结果：SyncBlk + VMT + 数据
GenericSample\<Int64> | 16 | 同上
GenericSample\<IEnumerable> | 12 | 同上
GenericSample\<DateTime> | 16 | 同上
string | 14 | 对于任何字符串，都会返回这个值，因为实际大小需要动态计算。但是，它适用于空字符串的大小。请注意，大小没有按字节对齐：实际上，这个字段不应该被使用
int[]{1} | 24554 | 对于数组，这里的数据完全不同，它们的大小也不是固定的，因此需要单独计算

正如您所见，当系统存储有关类型实例大小的数据时，它实际上存储的是该类型的引用形式的数据（即也包括值类型的引用形式）。让我们得出一些结论：

1. 如果您想知道一个重要类型作为值占用的空间大小，请使用 `sizeof(TType)`
2. 如果您想计算装箱操作的成本，您可以将 `sizeof(TType)` 向上取整到处理器字大小（4或8字节）并加上2个字的大小（`2 * sizeof(IntPtr)`）。或者，您也可以从类型的 `VMT` 中获取这个值。
3. 下面列出了堆上分配的内存量的计算方法，适用于以下类型：
   1. 普通的固定大小引用类型：我们可以从 `VMT` 中获取实例的大小；
   2. 字符串，需要手动计算其大小（这可能很少需要，但您同意，这很有趣）
   3. 数组，其大小也是单独计算的：基于其元素的大小和数量。这个任务可能更有用：因为数组是首先进入 `LOH`（大对象堆）的。

### System.String

关于字符串的实践问题，我们将单独讨论：对于这个相对较小的类，我们可以专门分配一个章节。在讨论 VMT 结构的章节中，我们将讨论字符串在底层的结构。字符串使用 UTF16 标准存储。这意味着每个字符占用2字节。另外，在每个字符串的末尾存储有一个 null-终结符 - 一个标识字符串结束的值。此外，实例中还存储了字符串的长度，以 `Int32` 数字形式存储 -- 这样就不需要每次需要时都计算长度了（关于编码我们将单独讨论）。下面的图表展示了字符串占用的内存信息：

```
  // 对于 .NET Framework 3.5 及更早版本
  ----------------------------------------------------------------------------------------------------
  |  SyncBlkIndx |    VMTPtr     |  ArrayLength   |     Length     |   char   |   char   |   Term    |
  ----------------------------------------------------------------------------------------------------
  |  4 / 8 字节  |  4 / 8 字节   |    4 字节      |    4 字节      |  2 字节  |  2 字节  |  2 字节   |
  ----------------------------------------------------------------------------------------------------
  |      -1      |  0xXXXXXXXX   |        3       |        2       |     a    |     b    |   <nil>   |
  ----------------------------------------------------------------------------------------------------
```

术语 - 空终结符
总大小 = (8|16) + 2 * 4 + 计数 * 2 + 2 -> 按字节大小向上取整。（示例中为24字节）
计数 - 字符串中的字符数量，不包括终结符

// 对于.NET Framework 4及更高版本
-----------------------------------------------------------------------------------
|  SyncBlkIndx  |    VMTPtr     |     Length     |   char   |   char   |   Term    |
-----------------------------------------------------------------------------------
|  4 / 8 字节    |  4 / 8 字节   |    4 字节      |  2 字节  |  2 字节  |  2 字节   |
-----------------------------------------------------------------------------------
|      -1       |  0xXXXXXXXX   |        2       |     a    |     b    |   <nil>   |
-----------------------------------------------------------------------------------
Term - 空终结符
总大小 = (8|16) + 4 + 计数 * 2 + 2) -> 按字节大小向上取整。（示例中为20字节）
计数 - 字符串中的字符数量，不包括终结符

```csharp
我们重写我们的方法，以教会它计算字符串的大小：

unsafe int SizeOf(object obj)
{
    var majorNetVersion = Environment.Version.Major;
    var type = obj.GetType();
    var href = Union.GetRef(obj).ToInt64();
    var DWORD = sizeof(IntPtr);
    var baseSize = 3 * DWORD;

    if (type == typeof(string))
    {
        if (majorNetVersion >= 4)
        {
            var length = (int)*(int*)(href + DWORD /* 跳过 vmt */);
            return DWORD * ((baseSize + 2 + 2 * length + (DWORD-1)) / DWORD);
        }
        else
        {
            // 在 1.0 -> 3.5 版本中，字符串有额外的 RealLength 字段
            var arrlength = *(int*)(href + DWORD /* 跳过 vmt */);
            var length = *(int*)(href + DWORD /* 跳过 vmt */ + 4 /* 跳过长度 */);
            return DWORD * ((baseSize + 4 + 2 * length + (DWORD-1)) / DWORD);
        }
    }
    else
    if (type.BaseType == typeof(Array) || type == typeof(Array))
    {
        return ((ArrayInfo*)href)->SizeOf();
    }
    return SizeOf(type);
}
```

其中 `SizeOf(type)` 将调用旧实现 -- 针对固定长度的引用类型。

让我们在实践中检验代码：

```csharp
    Action<string> stringWriter = (arg) =>
    {
        Console.WriteLine($"`{arg}` 字符串的长度: {SizeOf(arg)}");
    };

    stringWriter("a");
    stringWriter("ab");
    stringWriter("abc");
    stringWriter("abcd");
    stringWriter("abcde");
    stringWriter("abcdef");
}

-----

`a` 字符串的长度: 16
`ab` 字符串的长度: 20
`abc` 字符串的长度: 20
`abcd` 字符串的长度: 24
`abcde` 字符串的长度: 24
`abcdef` 字符串的长度: 28
```

计算显示，字符串的大小不是线性增加的，而是每增加两个字符就跳跃式增加。这是因为每个字符的大小是2字节，而最终的大小必须能够被处理器的字节大小整除（在示例中为x86），因此字符串的大小会相应地调整为2字节的倍数。我们的工作成果是美好的：我们可以计算出任何一个字符串的成本。最后一步是了解如何计算内存中数组的大小。

### 数组

数组的结构稍微复杂一些：因为数组可能有不同的结构形式：

1. 它们可以存储值类型，也可以存储引用类型
2. 数组既可以是一维的，也可以是多维的
3. 每个维度（尺度）的起始点既可以是`0`，也可以是任何其他数字（在我看来，这是一个非常有争议的功能，它免除了程序员在FCL级别编写`arr[i -- startIndex]`的需要。这似乎是为了与其他语言的兼容性而设计的，例如，在Pascal中，数组的索引可以不从`0`开始，而是从任何数字开始，但我觉得这是多余的。

由此产生了一些关于数组实现的混乱，以及无法准确预测最终数组大小的问题：仅仅将元素数量乘以它们的大小是不够的。当然，对于大多数情况来说，这已经足够了。当我们担心进入LOH（大对象堆）时，大小变得重要。但即便如此，我们也有选择：我们可以简单地在“粗略计算”的大小上加上某个常数（例如，100），以了解是否超过了85000的界限。然而，在这个部分的任务有些不同：理解类型结构。让我们来看看：

```
  // 标题
  ----------------------------------------------------------------------------------------
  |   SBI   |  VMT_Ptr |  Total  |  Len_1  |  Len_2  | .. |  Len_N  |  Term   | VMT_Child |
  ----------------------------------opt-------opt------------opt-------opt--------opt-----
  |  4 / 8  |  4 / 8   |    4    |    4    |    4    |    |    4    |    4    |    4/8    |
  ----------------------------------------------------------------------------------------
  | 0xFF.FF | 0xXX.XX  |    ?    |    ?    |    ?    |    |    ?    | 0x00.00 | 0xXX.XX  |
  ----------------------------------------------------------------------------------------

  - opt: 可选
  - SBI: 同步块索引
  - VMT_Child: 仅当数组存储引用类型数据时存在
  - Total: 为了优化而存在。考虑所有维度的数组元素总数
  - Len_2..Len_N, Term: 仅当数组维度超过1时存在（由VMT->Flags中的位控制）
```

正如我们所见，类型头部存储了有关数组维度的数据：它们的数量可以是1，也可以是相当大的：实际上它们的大小仅由表示枚举结束的null-终结符限制。这个例子在[GettingInstanceSize](./samples/GettingInstanceSize.linq)文件中完整可见，下面我只介绍它的最重要部分：

```csharp
public int SizeOf()
{
    var total = 0;
    int elementsize;

    fixed (void* entity = &MethodTable)
    {
        var arr = Union.GetObj<Array>((IntPtr)entity);
        var elementType = arr.GetType().GetElementType();

        if (elementType.IsValueType)
        {
            var typecode = Type.GetTypeCode(elementType);

            switch (typecode)
            {
                case TypeCode.Byte:
                case TypeCode.SByte:
                case TypeCode.Boolean:
                    elementsize = 1;
                    break;
                case TypeCode.Int16:
                case TypeCode.UInt16:
                case TypeCode.Char:
                    elementsize = 2;
                    break;
                case TypeCode.Int32:
                case TypeCode.UInt32:
                case TypeCode.Single:
                    elementsize = 4;
                    break;
                case TypeCode.Int64:
                case TypeCode.UInt64:
                case TypeCode.Double:
                    elementsize = 8;
                    break;
                case TypeCode.Decimal:
                    elementsize = 12;
                    break;
                default:
                    var info = (MethodTable*)elementType.TypeHandle.Value;
                    elementsize = info->Size - 2 * sizeof(IntPtr); // 同步块 + 虚方法表指针
                    break;
            }
        }
        else
        {
            elementsize = IntPtr.Size;
        }

        // 头部
        total += 3 * sizeof(IntPtr); // 同步块 + 虚方法表指针 + 总长度
        total += elementType.IsValueType ? 0 : sizeof(IntPtr); // 引用类型的方法表
        total += IsMultidimensional ? Dimensions * sizeof(int) : 0;
    }

    // 内容
    total += (int)TotalLength * elementsize;

    // 将大小对齐到IntPtr
    if ((total % sizeof(IntPtr)) != 0)
    {
        total += sizeof(IntPtr) - total % (sizeof(IntPtr));
    }
    return total;
}
```

这段代码考虑了数组类型的所有变化，并且可以用来计算它的大小：

```csharp
Console.WriteLine($"int[]{{1,2}}的大小: {SizeOf(new int[2])}");
Console.WriteLine($"int[2,1]{{1,2}}的大小: {SizeOf(new int[1,2])}");
Console.WriteLine($"int[2,3,4,5]{{...}}的大小: {SizeOf(new int[2, 3, 4, 5])}");
```

---
int[]{1,2}的大小: 20
int[2,1]{1,2}的大小: 32
int[2,3,4,5]{...}的大小: 512

在这个阶段，我们学会了一些相当重要的事情。首先，我们为自己将引用类型分为三组：固定大小的引用类型、泛型和可变大小的引用类型。我们还学会了理解任何类型的最终实例的结构（关于VMT的结构我暂时保持沉默。我们目前完全理解了其中只有一个字段：这也是一个重大成就）。无论是固定大小的引用类型（那里一切都非常简单）还是未定义大小的引用类型：数组或字符串。未定义，因为它的大小将在创建时确定。对于泛型实际上很简单：对于每个具体的泛型类型，都会创建自己的VMT，在其中会标明具体的大小。

## 虚拟方法表（VMT）中的方法表

关于方法表的工作原理的解释，大部分具有学术性质：因为深入这样的细节就像是自掘坟墓。一方面，这样的秘密藏着一些令人兴奋和有趣的东西，保留着一些数据，这些数据进一步揭示了对正在发生的事情的理解。然而，另一方面，我们都明白，微软不会给我们任何保证，他们会保持他们的运行时不变，例如，突然不会将方法表向前移动一个字段。因此，我要先声明：

> 本节中提供的信息仅供您了解基于CLR的应用程序是如何工作的，手动干预其工作不提供任何保证。然而，这是如此有趣，以至于我无法劝阻你。相反，我的建议是 -- 玩弄这些数据结构，也许你会获得软件开发中最难忘的经验之一。

好了，我已经提前警告过了。现在，让我们深入所谓的镜中世界。因为到目前为止，镜中世界的所有知识都归结于对象结构的知识：而按理来说，我们至少应该大致了解这一点。从本质上讲，这些知识并不属于镜中世界，而更像是进入镜中世界的入口。因此，让我们回到```MethodTable```结构，[在CoreCLR中的描述](https://github.com/dotnet/coreclr/blob/master/src/vm/methodtable.h#L4099-L4114):

```cpp
    // 低WORD是数组和字符串类型的组件大小（HasComponentSize()返回true）。
    // 否则用于标志。
    DWORD m_dwFlags;

    // 在堆上分配时此类实例的基本大小
    DWORD m_BaseSize;

    WORD  m_wFlags2;

    // 如果类标记适合16位，则为类标记。如果这是(WORD)-1，则类标记存储在TokenOverflow可选成员中。
    WORD  m_wToken;

    // <NICE> 在正常情况下，我们不应该需要为每个这些使用完整的字 </NICE>
    WORD  m_wNumVirtuals;
    WORD  m_wNumInterfaces;
```

特别是关于`m_wNumVirtuals`和`m_wNumInterfaces`字段。这两个字段回答了“一个类型有多少虚拟方法和接口？”的问题。这个结构中没有关于普通方法、字段、属性（它们合并了方法）的任何信息。即，这个结构**与反射无关**。本质上和目的上，这个结构是为了在CLR中（实际上在任何OOP中：无论是Java、C++、Ruby还是其他什么。只是字段的位置会有所不同）的方法调用而创建的。让我们来看看代码：

```csharp
public class Sample
{
    public int _x;

    public void ChangeTo(int newValue)
    {
        _x = newValue;
    }

    public virtual int GetValue()
    {
        return _x;
    }
}

public class OverridedSample : Sample
{
    public override int GetValue()
    {
        return 666;
    }
}
```

无论这些类看起来多么无意义，它们完全适合我们描述它们的VMT。为此，我们需要了解基本类型和继承类型在`ChangeTo`和`GetValue`方法上的区别。

`ChangeTo`方法存在于两种类型中：同时它不能被重写。这意味着它可以这样被重写，而且什么都不会改变：

```csharp
 public class Sample
 {
     public int _x;

     public static void ChangeTo(Sample self, int newValue)
     {
         self._x = newValue;
     }

     // ...
 }

// 或者如果它是结构体的情况
 public struct Sample
 {
     public int _x;

     public static void ChangeTo(ref Sample self, int newValue)
     {
         self._x = newValue;
     }

     // ...
 }
```

即便如此，除了架构上的意义之外，其他方面都不会有任何变化：请相信，在编译时，这两种方式的运行效果是相同的，因为对于实例方法来说，`this`不过是方法的第一个参数，它是隐式传递给我们的。

> 我将提前解释为什么关于继承的所有解释都是基于静态方法的例子：本质上，所有的方法都是静态的。无论是实例方法还是其他。在内存中，不会为每个类的实例单独编译方法。如果那样做，将会占用大量的内存：更简单的做法是，每次调用方法时，都传递一个对结构或类实例的引用。

对于`GetValue`方法，情况完全不同。我们不能简单地通过在派生类型中重写*静态的* `GetValue`方法来重写它：新方法只能获取那些将变量视为`OverridedSample`的代码段，而如果将变量视为基类型`Sample`的变量，只能调用基类型的`GetValue`方法，因为你根本不知道对象的实际类型是什么。为了理解变量的实际类型是什么，从而知道调用的具体是哪个方法，我们可以采取以下方式：

```csharp
void Main()
{
    var sample = new Sample();
    var overrided = new OverridedSample();

    Console.WriteLine(sample.Virtuals[Sample.GetValuePosition].DynamicInvoke(sample));
    Console.WriteLine(overrided.Virtuals[Sample.GetValuePosition].DynamicInvoke(sample));
}

public class Sample
{
    public const int GetValuePosition = 0;

    public Delegate[] Virtuals;

    public int _x;

    public Sample()
    {
        Virtuals = new Delegate[1] { 
            new Func<Sample, int>(GetValue) 
        };
    }

    public static void ChangeTo(Sample self, int newValue)
    {
        self._x = newValue;
    }

    public static int GetValue(Sample self)
    {
        return self._x;
    }
}

public class OverridedSample : Sample
{
    public OverridedSample() : base()
    {
        Virtuals[0] = new Func<Sample, int>(GetValue);
    }

    public static new int GetValue(Sample self)
    {
        return 666;
    }
}
```

在这个例子中，我们实际上是手动构建了一个虚拟方法表，而且是通过方法在这个表中的位置来进行调用的。如果你理解了这个例子的本质，那么你实际上就理解了在编译后的代码层面是如何构建继承的：方法是通过它们在虚拟方法表中的索引来调用的。只不过当你创建一个某个继承类型的实例时，编译器会在VMT中，基本类型的虚拟方法所在位置放置指向重写方法的指针，从基本类型复制那些未被重写的方法的指针。因此，我们例子与真实的VMT的区别仅在于，当编译器构建这个表时，它事先知道自己在处理什么，并且立即创建一个正确大小和内容的表：在我们的例子中，为了构建一个因添加新方法而变得更大的类型的表，将不得不付出相当的努力。但我们的任务在于其他方面，因此我们不会进行这样的尝试。

紧接着第一个问题的答案之后出现的第二个问题是：如果方法现在都清楚了，那么为什么`VMT`中还要有接口呢？如果逻辑上考虑，接口并不属于直接继承结构的一部分。它们存在于一旁，指示某些类型必须实现一套方法集。本质上，有一种交互协议。然而，尽管接口位于直接继承之外，但仍然可以调用方法。注意：如果你使用接口类型的变量，那么它背后可以隐藏任何类，其基本类型可能只是`System.Object`。也就是说，实现接口的方法在虚拟方法表中的位置可能完全不同。那么在这种情况下，方法的调用是如何工作的呢？

## 虚拟存根分发（VSD）[进行中]

要理解这个问题，我们需要额外回顾一下，实现接口有两种方式：可以是隐式(`implicit`)实现，也可以是显式(`explicit`)实现。而且，这种实现可以是部分的：一部分方法采用隐式实现，另一部分采用显式实现。这种可能性实际上是实现的结果，甚至可能并非预先考虑过的：实现接口时，你明示或暗示了它包含了什么。类的一部分方法可能不属于接口，而接口中存在的方法可能在类中不存在（它们当然存在于类中，但语法表明，它们在架构上不是类的一部分）：类和接口在某种意义上是平行的类型层次结构。此外，接口是一个独立的类型，这意味着每个接口都有自己的虚拟方法表，以便每个人都能调用接口的方法。

让我们来看一下表格：不同类型的VMT（虚拟方法表）可能是什么样的：

| 接口 IFoo       |  类 A : IFoo    | 类 B : IFoo     |
------------------|------------------|-----------------
| -> GetValue()   |  SampleMethod()  | RunProcess()    |
| -> SetValue()   |  Go()            | -> GetValue()   |
|                 | -> GetValue()    | -> SetValue()   |
|                 | -> SetValue()    | LookToMoon()    |

这三种类型的VMT都包含了必要的方法`GetValue`和`SetValue`，但它们位于不同的索引位置：它们不能在所有地方都有相同的索引，因为这将与类的其他接口竞争索引。实际上，对于每个接口，都会为其在每个类中的每个实现创建一个接口副本。我们在FCL/BCL类中有633个`IDisposable`的实现吗？这意味着我们有633个额外的`IDisposable`接口，以支持每个类的VMT到VMT的转换，加上每个类中有一个指向其接口实现的引用。我们称这些接口为**私有接口**。也就是说，每个类都有自己的**私有接口**，这些接口是“系统级”的，充当到实际接口的代理类型。

因此，结果如下：接口和类一样，也有虚拟*接口*方法的继承，但这种继承不仅在一个接口继承另一个接口时发生，也在类实现接口时发生。当一个类实现了某个接口，就会创建一个额外的接口，这个接口明确指出父接口的哪些方法应该映射到最终类的哪些方法上。通过接口变量调用方法，就像在类的情况下通过VMT数组的索引调用方法一样，但对于这个接口的实现，你会通过索引选择一个来自*继承的*、不可见的接口的槽，这个接口将原始接口`IDisposable`与我们的类连接起来，实现了接口。

通过虚拟存根分派（**Virtual Stub Dispatch (VSD)**）进行虚拟方法的分派在2006年就已经被开发出来，作为接口中虚拟方法表的替代。这种方法的主要思想在于简化代码生成和随后简化方法调用，因为基于VMT的接口的初步实现将会非常复杂，构建所有接口的所有结构需要大量的工作和时间。分派代码本质上位于四个文件中，总共大约6400行代码，我们并不打算完全理解它。我们将尝试大致理解这些代码中发生的过程的本质。

VSD分派的逻辑可以分为两大部分：分派和提供缓存调用方法地址的存根（stubs）机制，这些地址通过一对值[类型；槽号]来识别。

为了全面理解在构建VSD过程中发生的过程，让我们首先从一个非常高的层次来看，然后再深入到底层。如果谈到调度机制，它会因为接口类型层次的逻辑并行性，最终会有很多类型，以及因为大部分类型JIT永远不会创建（因为Framework中类型的存在并不意味着它们的实例会被创建），而推迟它们的创建。使用传统的VMT对于*私有接口*来说，会造成一种情况，即JIT必须从一开始就为每个*私有接口*创建VMT。也就是说，每个类型的创建速度至少会慢两倍。负责调度的主要类是`DispatchMap`类，它内部封装了一个接口类型表，每个表由进入这些接口的方法表组成。每个方法根据其生命周期的阶段，可以处于四种状态之一：类型为“方法尚未被调用过，需要编译并替换旧的占位符”的占位符状态，类型为“方法需要每次动态找到，因为不能明确确定”的占位符状态，类型为“方法可以通过明确的地址访问，因此无需任何搜索”的占位符状态，或者是方法的完整体。

让我们从它们的生成和所需的数据结构角度来考察这些结构的构建。

### DispatchMap

DispatchMap是一个动态构建的数据结构，本质上是CLR中接口工作依赖的主要数据结构。其结构如下所示：

```
DWORD: 类型数量 =
DWORD: 类型 №1
DWORD: 类型 №1的槽位数量 = M1
DWORD: bool: 偏移量可以是负数
DWORD: 槽位 №1
DWORD: 目标槽位 №1
...
DWORD: 槽位 №M1
DWORD: 目标槽位 №M1
...
DWORD: 类型 №
DWORD: 类型 №1的槽位数量 = M
DWORD: bool: 偏移量可以是负数
DWORD: 槽位 №1
DWORD: 目标槽位 №1
...
DWORD: 槽位 №M
DWORD: 目标槽位 №M
```

即首先记录实现某些接口的总体类型数量。之后，对于每个接口，记录其类型、该类型实现的槽位数量（用于导航表格），以及每个槽位的信息，还有在*特定接口*中包含当前类型方法实现的目标槽位。

为了导航这个数据结构，提供了`EncodedMapIterator`类，它是一个迭代器。即，除了`foreach`之外，没有其他访问`DispatchMap`的方式。而且，槽位编号是通过实际槽位编号与之前编码的槽位编号的差值获得的。即，只有从头开始浏览整个结构，才能在表中间获取槽位编号。这对于通过接口调用方法的性能提出了许多问题：因为如果我们有实现某个接口的对象数组，为了理解需要调用哪个方法，必须查看整个实现表。即，本质上是找到所需的。每次迭代的结果将是一个`DispatchMapEntry`结构，它将显示目标方法位于当前类型还是其他位置，以及为了获取所需方法需要从类型中取哪个槽位。

```csharp
// DispatchMapTypeID 允许进行方法的相对地址寻址。即，回答这样一个问题：相对于当前类型，所需的方法位于哪里？
// 是在当前类型中，还是在另一个类型中？
//
// 类型标识符（Type ID）在分派映射中使用，并且内部包含以下几种数据类型之一：
//   - 一个特殊值，表示这是 "this" 类
//   - 一个特殊值，表示这是一个接口类型，未由类实现
//   - 在InterfaceMap中的索引
class DispatchMapTypeID
{
private:
    static const UINT32 const_nFirstInterfaceIndex = 1;

    UINT32 m_typeIDVal;

    // ...
}

struct DispatchMapEntry
{
private:
    DispatchMapTypeID m_typeID;
    UINT16            m_slotNumber;
    UINT16            m_targetSlotNumber;

    enum
    {
        e_IS_VALID = 0x1
    };

    UINT16 m_flags;

    // ...
}
```

#### TypeID 映射

任何通过接口地址化的方法都由一对 <TypeId;SlotNumber> 编码。TypeId，顾名思义，是类型的标识符。这个字段回答了以下问题：这个标识符从哪里来，以及如何将其映射到实际的类型上。
`TypeIDMap` 类存储类型映射，作为某个 TypeId 到特定类型的 `MethodTable` 的映射，以及额外地，反向映射。这样做纯粹是出于性能考虑。这些哈希表的构建是动态的：根据对 PTR_MethodTable 的 TypeId 请求，返回的是 FatId 或者仅仅是 Id。在某种意义上，这只是需要记住的：FatId 和 Id 只是 TypeId 的两种形式。在某种意义上，这是对 MethodTable 的“指针”，因为它唯一地标识了它。

> **TypeId 是 MethodTable 的标识符**。它可以是两种形式：Id 和 FatId，本质上是一个普通的数字。

```csharp
class TypeIDMap
{
protected:
    HashMap             m_idMap;  // 存储 TypeID -> PTR_MethodTable 的映射
    HashMap             m_mtMap;  // 存储 PTR_MethodTable -> TypeID 的映射
    Crst                m_lock;
    TypeIDProvider      m_idProvider;
    BOOL                m_fUseFatIdsForUniqueness;
    UINT32              m_entryCount;

    // ...
}
```

尽管面对这些困难，JIT都能相对轻松地处理，它会在可能的情况下，将特定方法的调用嵌入到它们通过接口被调用的地方。如果JIT确定没有其他方法会被调用，它就会直接放置一个特定方法的调用。这是JIT编译器一个非常非常强大的特性，为我们提供了这种美妙的优化。

### 结论

对我们这些C#程序员来说，将应用程序分解为类和接口已经成为日常习惯，深深植根于我们的意识中，以至于我们甚至不需要思考就能理解如何进行分解，有时这种实现的复杂性是如此难以理解，以至于需要花费数周时间分析源代码来确定所有的依赖关系和逻辑。对我们来说，这种使用上的习以为常，以至于不会对实现的简单性产生任何怀疑，实际上可能隐藏了实现的复杂性。这告诉我们，实现这些想法的工程师在解决问题时非常聪明，仔细分析每一步。

这里给出的描述实际上是非常表面和简短的：它是非常高层次的。即使与任何一本关于.NET的书相比，我们已经深入得多，对VSD和VMT的构建的描述仍然是非常非常高层次的。毕竟，描述这两种数据结构的代码文件总共大约有20,000行代码。这还不包括一些负责泛型的部分。

然而，这使我们能够得出一些结论：

- 调用静态方法和实例方法几乎没有区别。这意味着我们不需要担心使用实例方法会以某种方式影响性能。在相同条件下，两种方法的性能完全相同。
- 虽然调用虚拟方法是通过VMT表进行的，但由于索引是预先知道的，每次调用只需要额外进行一次指针解引用。在几乎所有情况下，这都不会有任何影响：性能的下降（如果可以这么说的话）是如此之小，以至于基本上可以忽略不计。
- 谈到接口，需要记住的是分派，并理解通过接口工作大大增加了在底层实现类型子系统的复杂性，导致在频繁调用方法时，如果对于接口变量要调用哪个类的哪个方法没有明确性，可能会出现性能下降。然而，“智能”的JIT编译器在很多情况下允许不通过分派直接调用实现接口的方法。
- 谈到泛型，这里出现了另一个抽象层，它增加了在寻找实现泛型接口的类型所需调用的方法的复杂性。

## 关于此主题的问题部分

> 问题：为什么如果每个类都可以实现一个接口，那么就不能从对象中提取接口的具体实现？

答案很简单：这是CLR在设计语言时未覆盖的一个功能，CLR并不限制这个问题。而且，很有可能在C#的近期版本中添加这一功能，毕竟它们更新得很快。看一个例子：

```csharp
void Main()
{
    var foo = new Foo();
    var boo = new Boo();

    ((IDisposable)foo).Dispose();
    foo.Dispose();
    ((IDisposable)boo).Dispose();
    boo.Dispose();
}

class Foo : IDisposable
{
    void IDisposable.Dispose()
    {
        Console.WriteLine("Foo.IDisposable::Dispose");
    }
```

```csharp
public void Dispose()
{
    Console.WriteLine("Foo::Dispose()");
}
}

class Boo : Foo, IDisposable
{
    void IDisposable.Dispose()
    {
        Console.WriteLine("Boo.IDisposable::Dispose");
    }

    public new void Dispose()
    {
        Console.WriteLine("Boo::Dispose()");
    }
}
```

在这里，我们调用了四种不同的方法，它们的调用结果将是：

```csharp
Foo.IDisposable::Dispose
Foo::Dispose()
Boo.IDisposable::Dispose
Boo::Dispose()
```

尽管我们在两个类中都有显式接口实现，但在`Boo`类中，我们无法获得`Foo`的`IDisposable`接口的显式实现。即使我们这样写：

```csharp
((IDisposable)(Foo)boo).Dispose();
```

我们仍然会在屏幕上得到相同的结果：

```csharp
Boo.IDisposable::Dispose
```

## 隐式和多重接口实现有什么不好的地方？

作为一个“接口继承”的例子，这与类的继承类似，我们可以提供以下代码：

```vb
Class Foo
    Implements IDisposable

    Public Overridable Sub DisposeImp() Implements IDisposable.Dispose
        Console.WriteLine("Foo.IDisposable::Dispose")
    End Sub

    Public Sub Dispose()
        Console.WriteLine("Foo::Dispose()")
    End Sub

End Class

Class Boo
    Inherits Foo
    Implements IDisposable

    Public Sub DisposeImp() Implements IDisposable.Dispose
        Console.WriteLine("Boo.IDisposable::Dispose")
    End Sub

    Public Shadows Sub Dispose()
        Console.WriteLine("Boo::Dispose()")
    End Sub

End Class

''' <summary>
''' 隐式实现接口
''' </summary>
Class Doo
    Inherits Foo

    ''' <summary>
    ''' 重写显式实现
    ''' </summary>
    Public Overrides Sub DisposeImp()
        Console.WriteLine("Doo.IDisposable::Dispose")
    End Sub

    ''' <summary>
    ''' 隐式覆盖
    ''' </summary>
    Public Sub Dispose()
        Console.WriteLine("Doo::Dispose()")
    End Sub

End Class

Sub Main()
    Dim foo As New Foo
    Dim boo As New Boo
    Dim doo As New Doo

    CType(foo, IDisposable).Dispose()
    foo.Dispose()
    CType(boo, IDisposable).Dispose()
    boo.Dispose()
    CType(doo, IDisposable).Dispose()
    doo.Dispose()
End Sub
```

在这个例子中，我们可以看到`Doo`继承自`Foo`，隐式地实现了`IDisposable`接口，但同时重写了`IDisposable.Dispose`的显式实现，这将导致通过接口调用时调用重写的实现，从而展示了`Foo`和`Doo`类的“接口继承”。

从一方面来说，这根本不是问题：如果C# + CLR允许这样的花招，我们在某种意义上会得到类型结构的一致性被破坏。想想看：你设计了一个很酷的架构，一切都很好。但是有人出于某种原因不按照你设想的方式调用方法。这会很糟糕。另一方面，在C++中存在类似的功能，而且人们并不怎么抱怨这一点。为什么我说这可以被添加到C#中呢？因为同样糟糕的功能[已经在讨论中](https://github.com/dotnet/csharplang/issues/52)，并且它应该大致如下所示：

```csharp
interface IA
{
    void M() { WriteLine("IA.M"); }
}

interface IB : IA
{
    override void IA.M() { WriteLine("IB.M"); } // 明确命名
}

interface IC : IA
{
    override void M() { WriteLine("IC.M"); } // 隐式命名
}
```

为什么这很糟糕？因为实际上这产生了一整类的可能性。现在我们不需要每次都实现接口中的某些方法，这些方法在各处都是相同的实现。听起来很棒。但仅仅是听起来。因为接口是交互的协议。协议是一套规则，框架。它不应该允许存在实现。这里直接违反了这一原则，并引入了另一个原则：多重继承。老实说，我非常反对这样的改进，但...我有点跑题了。

[DispatchMap::CreateEncodedMapping](https://github.com/dotnet/coreclr/blob/master/src/vm/contractimpl.cpp#L295-L460)

> 接下来：[Memory&lt;T&gt; 和 Span&lt;T&gt;](./3-MemorySpan.md)