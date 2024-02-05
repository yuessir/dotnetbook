# Memory\<T> 和 Span\<T>

> [讨论链接](https://github.com/sidristij/dotnetbook/issues/55)

从 .NET Core 2.0 和 .NET Framework 4.5 版本开始，我们获得了新的数据类型：`Span` 和 `Memory`。要使用它们，只需安装 nuget 包 `System.Memory`：

  - `PM> Install-Package System.Memory`

这些数据类型之所以引人注目，是因为 `CLR` 团队为了实现它们在 .NET Core 2.1+ 的 JIT 编译器代码中的特殊支持而做了大量工作，将它们直接植入到核心中。这些数据类型是什么，以及为什么要在书中专门讨论它们？

如果谈到导致这些功能出现的问题，我会选择三个主要问题。其中第一个是非托管代码。

作为一种语言和平台，它已经存在了很多年：在这段时间里，一直存在着许多处理非托管代码的工具。那么，为什么现在又推出了另一个用于处理非托管代码的 API，尽管实际上它已经存在了很多年呢？要回答这个问题，只需了解我们以前缺少了什么。

平台的开发者们以前也试图帮助我们简化使用非托管资源的开发工作：这包括自动包装器用于导入的方法，以及在大多数情况下自动工作的封送处理。还有 `stackalloc` 指令，这在讨论线程栈的章节中有提及。然而，在我看来，如果早期使用 C# 语言的开发者是从 C++ 世界过来的（就像我一样），那么现在他们则是来自更高级别的语言（例如，我知道一个从 JavaScript 来的开发者）。这意味着，人们开始对非托管资源和类似 C/C++ 风格的构造持更多的怀疑态度，更不用说汇编语言了。

作为这种态度的结果 -- 项目中含有的不安全代码越来越少，对平台本身API的信任度越来越高。这一点通过搜索开源仓库中`stackalloc`的使用情况很容易验证：其实例极其罕见。但如果我们查看任何使用它的代码：

**Interop.ReadDir类**
[/src/mscorlib/shared/Interop/Unix/System.Native/Interop.ReadDir.cs](https://github.com/dotnet/coreclr/blob/b29f6328510207970763580d6f4db864e4b198af/src/mscorlib/shared/Interop/Unix/System.Native/Interop.ReadDir.cs#L71-L83)

```csharp
unsafe
{
    // 当本地实现不支持读取到缓冲区时，s_readBufferSize为零。
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

不受欢迎的原因变得明显。不深入代码，只问自己一个问题：你信任它吗？我猜答案可能是“不”。那么再问一个问题：为什么？答案很明显：除了我们看到的单词`Dangerous`，暗示着可能出现问题之外，影响我们态度的第二个因素是`unsafe`关键字和`byte* buffer = stackalloc byte[s_readBufferSize];`这行代码，更具体地说是`byte*`。这个表达式是任何人头脑中出现“难道就不能用其他方式实现吗？”这种想法的触发器。那么让我们再深入一点进行心理分析：为什么会出现这种想法？一方面，我们使用语言构造，这里提供的语法与例如C++/CLI相去甚远，后者允许做任何事情（包括在纯汇编上的插入），但另一方面，它看起来不太习惯。

第二个问题，显式或隐式地出现在许多开发者的脑海中——那就是类型不兼容：字符串`string`和字符数组`char[]`，尽管从逻辑上讲，字符串就是字符数组，但没有任何办法直接将string转换为char[]：只能创建一个新对象并将字符串内容复制到数组中。这种不兼容性是为了优化字符串的存储（不存在只读数组）而引入的，但当你开始处理文件时，问题就出现了。怎么读取？用字符串还是数组？因为如果用数组，就无法使用一些专为字符串设计的方法。如果用字符串？可能会太长。如果谈到后续的解析，就会出现选择解析器的问题：并不总是想要手动解析所有的基本类型（整数、浮点数，以不同格式记录的）。有很多经过多年验证的算法，可以快速高效地完成这项工作。但这样的算法往往只适用于只包含基本类型本身的字符串。换句话说，这是一个两难的问题。

第三个问题在于，某个算法所需的数据很少能从某个数据源读取的数组的开始到结束完全对应。例如，如果我们再次谈论文件或从套接字读取的数据，那么有一部分数据已经被某个算法处理过，接下来是我们的某个方法需要处理的数据，然后是我们还需要处理的数据。而这个方法理想情况下希望只接收它期待的数据。例如，一个解析整数的方法如果接收到一个包含随意对话的字符串，在某个位置包含了所需的数字，这个方法可能不会太感激。这种方法期望你只给它一个数字，仅此而已。如果我们整个数组都给出，就会出现需要指定数字相对于数组开始位置的偏移量的要求：

```csharp
int ParseInt(char[] input, int index)
{
    while(char.IsDigit(input[index]))
    {
        // ...
        index++;
    }
}
```

然而，这种方法的问题在于，方法接收到了它不需要的数据。换句话说，*方法没有被引入到它的任务上下文中*。除了它的任务外，方法还解决了一些外部问题。这是糟糕设计的标志。如何避免这个问题？一个选项是使用`ArraySegment<T>`类型，其本质是在某个数组中提供一个“窗口”：

```csharp
int ParseInt(IList<char>[] input)
{
    while(char.IsDigit(input.Array[index]))
    {
        // ...
        index++;
    }
}

var arraySegment = new ArraySegment(array, from, length);
var res = ParseInt((IList<char>)arraySegment);
```

但在我看来，这是过度的，无论是从逻辑角度还是从性能下降的角度来看。`ArraySegment`做得非常糟糕，相对于同样的操作但使用数组时，访问元素的速度慢了7倍。

如何解决所有这些问题呢？如何让开发者回归到不受管理的代码怀抱中，同时提供一个统一且快速处理各种数据源（如数组、字符串和不受管理的内存）的机制？需要给他们一种安心感，让他们知道自己不会因为无知而犯错。需要提供一个工具，其性能不亚于原生数据类型，但能解决上述问题。这个工具就是 `Span<T>` 和 `Memory<T>` 类型。

## Span\<T>, ReadOnlySpan\<T>

`Span` 类型代表了一个工具，用于操作某个数据数组的一部分或其值的子范围。它允许你像操作数组那样对这个范围内的元素进行读写，但有一个重要限制：你获得或创建 `Span<T>` 只是为了*临时*操作数组。仅限于一组方法调用的范围内，不超过这个范围。但为了加速理解，让我们比较一下 `Span` 类型实现的数据类型，并探讨使用它的可能场景。

首先要讨论的数据类型是普通数组。对于数组，使用 `Span` 的方式如下所示：

```csharp
    var array = new [] {1,2,3,4,5,6};
    var span = new Span<int>(array, 1, 3);
    var position = span.BinarySearch(3);
    Console.WriteLine(span[position]);  // -> 3
```

如我们在这个例子中看到的，首先我们创建了一个数据数组。然后我们创建了一个 `Span`（或子集），它引用了数组本身，并且只允许使用它的代码访问在初始化时指定的那个值范围。

这里我们看到了这种数据类型的第一个特性：创建某种上下文。让我们进一步探讨这个上下文的概念：

```csharp
void Main()
{
    var array = new [] {'1','2','3','4','5','6'};
    var span = new Span<char>(array, 1, 3);
    if(TryParseInt32(span, out var res))
    {
        Console.WriteLine(res);
    }
    else
    {
        Console.WriteLine("Failed to parse");
    }
}
```

```csharp
public bool TryParseInt32(Span<char> input, out int result)
{
    result = 0;
    for (int i = 0; i < input.Length; i++)
    {
        if(input[i] < '0' || input[i] > '9')
            return false;
    result = result * 10 + ((int)input[i] - '0');
    }
    return true;
}
```

如我们所见，`Span<T>`引入了对内存片段的访问抽象，无论是读还是写。这给我们带来了什么好处呢？如果回想一下`Span`还可以基于什么，我们会想到它既可以用于非托管资源，也可以用于字符串：

```csharp
// 托管数组
var array = new[] { '1', '2', '3', '4', '5', '6' };
var arrSpan = new Span<char>(array, 1, 3);
if (TryParseInt32(arrSpan, out var res1))
{
    Console.WriteLine(res1);
}

// 字符串
var srcString = "123456";
var strSpan = srcString.AsSpan();
if (TryParseInt32(arrSpan, out var res2))
{
    Console.WriteLine(res2);
}

// void *
Span<char> buf = stackalloc char[6];
buf[0] = '1'; buf[1] = '2'; buf[2] = '3';
buf[3] = '4'; buf[4] = '5'; buf[5] = '6';

if (TryParseInt32(arrSpan, out var res3))
{
    Console.WriteLine(res3);
}
```

也就是说，`Span<T>`是一种统一内存操作的手段：无论是托管还是非托管的，它都保证了在垃圾回收期间对此类数据的安全操作：如果托管数组的内存区域开始移动，对我们来说仍然是安全的。

但是，是否真的有理由如此高兴呢？我们是否可以通过其他方式达到同样的目的？比如说，对于托管数组，毫无疑问的是，可以简单地将数组封装在另一个类中（例如，早已存在的[ArraySegment](https://referencesource.microsoft.com/#mscorlib/system/arraysegment.cs,31)），提供类似的接口，一切就绪。不仅如此，对于字符串，也可以进行类似的操作：它们具有必要的方法。再次，只需将字符串封装在完全相同的类型中，并提供操作它的方法。然而，要在一个类型中存储字符串、缓冲区或数组，需要做很多工作，存储对每种可能的选项的引用（显然，一次只能有一个是活跃的）：

```csharp
public readonly ref struct OurSpan<T>
{
    private T[] _array;
    private string _str;
    private T * _buffer;

    // ...
}
```

或者，如果从架构的角度出发，那么可以创建三种类型，它们继承自一个共同的接口。这意味着，无法创建一个与`Span<T>`不同的统一接口工具来在这些数据类型之间进行操作，同时还保持最大的性能。

进一步讨论，那么`ref struct`在`Span`的概念中是什么呢？它正是那些“只存在于栈上的结构体”，这是我们在面试中经常听到的。这意味着，这种数据类型只能通过栈传递，不能被分配到堆上。因此，作为ref结构体的`Span`，是一种上下文数据类型，它支持方法的运行，而不是内存中的对象。这就是我们需要从这个角度来理解它的原因。

基于此，我们可以定义Span类型及其关联的readonly类型ReadOnlySpan：

> Span是一种数据类型，它提供了一个统一的接口来操作不同类型的数据数组，同时也允许将数组的一个子集传递给另一个方法，以这样的方式，无论取用的上下文深度如何，访问原始数组的速度都是常数级别的，并且非常高。

确实，如果我们有如下代码：

```csharp
public void Method1(Span<byte> buffer)
{
    buffer[0] = 0;
    Method2(buffer.Slice(1,2));
}
Method2(Span<byte> buffer)
{
    buffer[0] = 0;
    Method3(buffer.Slice(1,1));
}
Method3(Span<byte> buffer)
{
    buffer[0] = 0;
}
```

那么访问原始缓冲区的速度将是最高的：你操作的不是managed对象，而是managed指针。也就是说，不是.NET managed类型，而是被封装在managed壳中的unsafe类型。

### Span\<T>的示例

人的本性是这样的，通常直到他获得了某种经验，最终的理解，为什么需要这个工具，往往不会到来。因此，既然我们需要一些经验，让我们来看一些例子。

#### ValueStringBuilder

`ValueStringBuilder`类型是算法上最有趣的例子之一，它被深藏在`mscorlib`的内部，而且出于某种原因，像许多其他有趣的数据类型一样，它被标记为`internal`，这意味着如果不是通过研究mscorlib的源代码，我们永远不会知道这种优化方式的存在。

`StringBuilder`系统类型的主要缺点是什么呢？当然是它的本质：它本身以及它所基于的（即字符数组`char[]`）都是引用类型。这至少意味着两件事：我们仍然（虽然稍微）加载了堆，第二，我们增加了处理器缓存失误的机会。

我对`StringBuilder`的另一个问题是生成小字符串时的情况。也就是说，当最终字符串“绝对”很短时：例如，少于100个字符。当我们有足够短的格式化时，会对性能产生疑问：

```csharp
    $"{x} is in range [{min};{max}]"
```

这种写法比通过`StringBuilder`手动构造要差多少呢？答案并不总是显而易见的：这在很大程度上取决于构造的位置：这个方法被调用的频率有多高。因为最初`string.Format`会为内部的`StringBuilder`分配内存，它将创建一个字符数组（SourceString.Length + args.Length * 8），如果在数组构造过程中发现长度被错误估计，那么将创建另一个`StringBuilder`来继续构造，从而形成一个单链表。最终，需要返回构造的字符串：这又是一次复制。浪费和奢侈。如果能避免第一个构造字符串数组的堆分配就好了：我们肯定能解决一个问题。

让我们看看`mscorlib`深处的类型：

**ValueStringBuilder类**
[/src/mscorlib/shared/System/Text/ValueStringBuilder](https://github.com/dotnet/coreclr/blob/efebb38f3c18425c57f94ff910a50e038d13c848/src/mscorlib/shared/System/Text/ValueStringBuilder.cs)

```csharp
    internal ref struct ValueStringBuilder
    {
        // 如果我们有太多字符，这个字段将会被激活
        private char[] _arrayToReturnToPool;
        // 这个字段将会是主要的
        private Span<char> _chars;
        private int _pos;
        // 类型接受外部缓冲区，将其大小的选择委托给调用方
        public ValueStringBuilder(Span<char> initialBuffer)
        {
            _arrayToReturnToPool = null;
            _chars = initialBuffer;
            _pos = 0;
        }

        public int Length
        {
            get => _pos;
            set
            {
                int delta = value - _pos;
                if (delta > 0)
                {
                    Append('\0', delta);
                }
                else
                {
                    _pos = value;
                }
            }
        }

        // 获取字符串 - 从数组复制字符到数组
        public override string ToString()
        {
            var s = new string(_chars.Slice(0, _pos));
            Clear();
            return s;
        }

        // 插入到中间伴随着字符的移动
        // 为了插入所需的字符，通过复制
        public void Insert(int index, char value, int count)
        {
            if (_pos > _chars.Length - count)
            {
                Grow(count);
            }

            int remaining = _pos - index;
            _chars.Slice(index, remaining).CopyTo(_chars.Slice(index + count));
            _chars.Slice(index, count).Fill(value);
            _pos += count;
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void Append(char c)
        {
            int pos = _pos;
            if (pos < _chars.Length)
            {
                _chars[pos] = c;
                _pos = pos + 1;
            }
            else
            {
                GrowAndAppend(c);
            }
        }

        [MethodImpl(MethodImplOptions.NoInlining)]
        private void GrowAndAppend(char c)
        {
            Grow(1);
            Append(c);
        }

        // 如果构造函数传递的原始数组不足
        // 我们从空闲池中分配所需大小的数组
        // 实际上，如果算法能够额外创建
        // 数组大小的离散性，以避免池被碎片化，那将是理想的
        [MethodImpl(MethodImplOptions.NoInlining)]
        private void Grow(int requiredAdditionalCapacity)
        {
            Debug.Assert(requiredAdditionalCapacity > _chars.Length - _pos);

            char[] poolArray = ArrayPool<char>.Shared.Rent(Math.Max(_pos + requiredAdditionalCapacity, _chars.Length * 2));

            _chars.CopyTo(poolArray);
```

```csharp
char[] toReturn = _arrayToReturnToPool;
_arrayToReturnToPool = poolArray;
_chars = poolArray;
if (toReturn != null)
{
    ArrayPool<char>.Shared.Return(toReturn);
}
```

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private void Clear()
{
    char[] toReturn = _arrayToReturnToPool;
    this = default; // 为了安全起见，避免如果这个实例被错误地再次追加时使用池化数组
    if (toReturn != null)
    {
        ArrayPool<char>.Shared.Return(toReturn);
    }
}
```

```csharp
// 省略的方法：对它们来说一切都很清楚
private void AppendSlow(string s);
public bool TryCopyTo(Span<char> destination, out int charsWritten);
public void Append(string s);
public void Append(char c, int count);
public unsafe void Append(char* value, int length);
public Span<char> AppendSpan(int length);
```

这个类在功能上类似于它的大兄弟`StringBuilder`，但它有一个有趣且非常重要的特性：它是一个值类型。也就是说，它完全按值存储和传递。而新的`ref`类型修饰符，附加在类型声明的签名上，告诉我们这个类型有一个额外的限制：它只能存在于栈上。也就是说，将其实例输出到类字段将导致错误。这一切的意义何在？只需看一下我们刚刚描述的`StringBuilder`类的本质：

**StringBuilder 类** [/src/mscorlib/src/System/Text/StringBuilder.cs](https://github.com/dotnet/coreclr/blob/68f72dd2587c3365a9fe74d1991f93612c3bc62a/src/mscorlib/src/System/Text/StringBuilder.cs#L47-L62)

```csharp
public sealed class StringBuilder : ISerializable
{
    // StringBuilder 在内部表示为一个块链表，每个块持有字符串的一部分。
    // 事实证明，整个字符串也可以仅表示为一个块，
    // 所以这就是我们所做的。
    internal char[] m_ChunkChars;                // 本块中的字符
    internal StringBuilder m_ChunkPrevious;      // 逻辑上本块之前的块的链接
    internal int m_ChunkLength;                  // m_ChunkChars 中表示块末尾的索引
    internal int m_ChunkOffset;                  // 逻辑偏移量（前面所有块中的字符总和）
    internal int m_MaxCapacity = 0;

    // ...

    internal const int DefaultCapacity = 16;
```

`StringBuilder`是一个类，它内部包含了一个字符数组的引用。也就是说，当你创建一个`StringBuilder`对象时，实际上至少创建了两个对象：`StringBuilder`本身和一个至少有16个字符的字符数组（这也是为什么设置预期字符串长度很重要的原因：它的构建是通过生成一系列16字符数组的单链表来实现的。你会同意，这是一种浪费）。这在我们讨论`ValueStringBuilder`类型时意味着什么呢：默认情况下它没有容量，因为它从外部借用内存，加上它本身是一个值类型，迫使用户在栈上分配字符缓冲区。结果是，整个类型实例连同其内容一起被放置在栈上，这里的优化问题就得到了解决。没有在堆上分配内存？就没有堆性能下降的问题。但你可能会问：那为什么不总是使用`ValueStringBuilder`（或其自制版本：它是内部的，我们无法直接使用）呢？答案是：需要根据你要解决的问题来决定。结果字符串的大小是否已知？它是否有一个已知的最大长度？如果答案是“是”，并且同时字符串的大小在某些合理的范围内，那么可以使用`StringBuilder`的值类型版本。否则，如果我们预期字符串会很长，就转而使用常规版本。

#### ValueListBuilder

```csharp
internal ref partial struct ValueListBuilder<T>
{
    private Span<T> _span;
    private T[] _arrayFromPool;
    private int _pos;

    public ValueListBuilder(Span<T> initialSpan)
    {
        _span = initialSpan;
        _arrayFromPool = null;
        _pos = 0;
    }

    public int Length { get; set; }

    public ref T this[int index] { get; }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Append(T item);

    public ReadOnlySpan<T> AsSpan();

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Dispose();

    private void Grow();
}
```

另一个特别值得注意的数据类型是`ValueListBuilder`。它是为了那些需要短时间内创建一些元素集合并立即将其交给某个算法处理的情况而设计的。

你会同意：这个任务与`ValueStringBuilder`的任务非常相似。而且，它们的解决方式也非常相似：

**文件 [ValueListBuilder.cs](https://github.com/dotnet/coreclr/blob/dbaf2957387c5290a680c8918779683194137b1d/src/System.Private.CoreLib/shared/System/Collections/Generic/ValueListBuilder.cs)**

如果直说的话，这样的情况相当常见。但是，以前我们采用了另一种方式来解决这个问题：创建一个`List`，填充数据然后丢失引用。如果这个方法被频繁调用，就会出现一个令人遗憾的情况：大量的`List`类实例在堆中悬挂，与之关联的数组也一并悬挂在堆中。现在这个问题已经得到解决：不会再创建额外的对象。然而，就像`ValueStringBuilder`的情况一样，这个解决方案只适用于Microsoft的程序员：该类被标记为`internal`。

### 使用规则和实践

为了彻底理解这种新数据类型的本质，需要通过编写几个（最好是更多）使用它的方法来“实践”一下。不过，现在就可以掌握一些基本规则：

  - 如果你的方法将处理一些输入数据集，而不改变其大小，你可以尝试使用`Span`类型。如果在此过程中不对这个缓冲区进行修改，那么可以使用`ReadOnlySpan`类型；
  - 如果你的方法将处理字符串，计算某种统计数据或进行字符串的语法解析，那么你的方法_必须_接受`ReadOnlySpan<char>`。这是一个新规则，必须遵守：因为如果你接受一个字符串，你就迫使某人为你创建一个子字符串；
  - 如果在方法的工作过程中需要创建一个相对较短的数据数组（比如，最多10KB），那么你可以轻松地使用`Span<TType> buf = stackalloc TType[size]`来组织这样一个数组。当然，TType必须是值类型，因为`stackalloc`只适用于值类型。

在其他情况下，应该考虑使用`Memory`或传统的数据类型。

### Span的工作原理

此外，我还想谈谈`Span`的工作原理以及它有什么特别之处。这确实值得讨论：这种数据类型分为两个版本：适用于.NET Core 2.0+和所有其他版本。

**文件 [Span.Fast.cs, .NET Core 2.0](https://github.com/dotnet/coreclr/blob/38403e661a4202ca4c8a72e4bbd9a263bddeb891/src/System.Private.CoreLib/shared/System/Span.Fast.cs)**

```csharp
public readonly ref partial struct Span<T>
{
    /// .NET对象的引用或纯指针
    internal readonly ByReference<T> _pointer;
    /// 指针数据缓冲区的长度
    private readonly int _length;
    // ...
}
```

**文件 ??? [反编译的]**

```csharp
public ref readonly struct Span<T>
{
    private readonly System.Pinnable<T> _pinnable;
    private readonly IntPtr _byteOffset;
    private readonly int _length;
    // ...
}
```

问题在于，与.NET Core 2.0+版本不同，较大的.NET Framework和.NET Core 1.*没有特别修改的垃圾收集器，因此它们被迫携带一个额外的指针：指向正在操作的缓冲区的开始。也就是说，`Span`在其内部将.NET平台的托管对象视为非托管对象。看看第二个结构版本的内部：有三个字段。第一个字段是对托管对象的引用。第二个是相对于该对象开始的字节偏移量，以获取数据缓冲区的开始（在字符串中是`char`字符的缓冲区，在数组中是数组数据的缓冲区）。最后，第三个字段是这个缓冲区中连续排列的元素数量。

以`Span`对字符串的操作为例：

**文件 [coreclr::src/System.Private.CoreLib/shared/System/MemoryExtensions.Fast.cs](https://github.com/dotnet/coreclr/blob/2b50bba8131acca2ab535e144796941ad93487b7/src/System.Private.CoreLib/shared/System/MemoryExtensions.Fast.cs#L409-L416)**

```csharp
public static ReadOnlySpan<char> AsSpan(this string text)
{
    if (text == null)
        return default;

    return new ReadOnlySpan<char>(ref text.GetRawStringData(), text.Length);
}
```

其中`string.GetRawStringData()`定义如下：

**字段定义文件 [coreclr::src/System.Private.CoreLib/src/System/String.CoreCLR.cs](https://github.com/dotnet/coreclr/blob/2b50bba8131acca2ab535e144796941ad93487b7/src/System.Private.CoreLib/src/System/String.CoreCLR.cs#L16-L23)**

**GetRawStringData定义文件 [coreclr::src/System.Private.CoreLib/shared/System/String.cs](https://github.com/dotnet/coreclr/blob/2b50bba8131acca2ab535e144796941ad93487b7/src/System.Private.CoreLib/shared/System/String.cs#L462)**

```csharp
public sealed partial class String :
    IComparable, IEnumerable, IConvertible, IEnumerable<char>,
    IComparable<string>, IEquatable<string>, ICloneable
{

    //
    // 这些字段直接映射到EE StringObject中的字段。参见object.h了解布局。
    //
    [NonSerialized] private int _stringLength;

    // 对于空字符串，这将是'\0'，因为
    // 字符串既是以null结尾又是以长度为前缀的
    [NonSerialized] private char _firstChar;


    internal ref char GetRawStringData() => ref _firstChar;
}
```

也就是说，该方法直接深入字符串内部，而`ref char`的规范允许GC跟踪字符串内部的非托管引用，并在GC触发时将其与字符串一起移动。

同样的故事也发生在数组上：当创建`Span`时，JIT内部的某些代码会计算数组数据开始的偏移量，并用这个偏移量初始化`Span`。至于如何计算字符串和数组的偏移量，我们在[关于内存中对象结构的章节](.\ObjectsStructure.md)中已经学会了。

### 作为返回值的Span\<T>

尽管与`Span`相关的一切看起来很理想，但在将其作为方法返回值时，存在一些虽然合乎逻辑，但出乎意料的限制。看看下面的代码：

```csharp
unsafe void Main()
{
    var x = GetSpan();
}

public Span<byte> GetSpan()
{
    Span<byte> reff = new byte[100];
    return reff;
}
```

这看起来非常合理和正常。然而，将一条指令替换为另一条后：

```csharp
unsafe void Main()
{
    var x = GetSpan();
}

public Span<byte> GetSpan()
{
    Span<byte> reff = stackalloc byte[100];
    return reff;
}
```

编译器将禁止这种形式的指令。但在解释原因之前，我希望你们能自己想一想，这样的构造会带来什么问题。

所以，我希望你们已经思考过，做出了猜测和假设，甚至可能已经理解了原因。如果是这样，那么我之前详细讲解[线程栈](./ThreadStack.md)的章节就不是白费力气。因为通过这种方式引用方法的局部变量，当该方法结束运行后，你可以调用另一个方法，等待它结束运行，然后通过x[0.99]读取它的局部变量。

然而，幸运的是，当我们尝试编写这种代码时，编译器会给我们一个警告，提示：`CS8352 不能在此上下文中使用局部变量'reff'，因为它可能会将引用的变量暴露到它们声明的作用域之外`，并且这是正确的：因为如果绕过这个错误，就有可能出现这样的情况，比如，在一个插件中设置这样的情况，从而有可能窃取别人的密码或提升我们插件的执行权限。

## Memory\<T> 和 ReadOnlyMemory\<T>

`Memory<T>`与`Span<T>`之间的视觉差异有两点。首先，`Memory<T>`类型的声明中不包含`ref`限制。换句话说，`Memory<T>`类型不仅可以存在于栈上，作为局部变量、方法参数或其返回值，还可以存在于堆上，从那里引用内存中的某些数据。然而，这一小差异在`Memory<T>`与`Span<T>`的行为和能力上创造了巨大的差异。与`Span<T>`不同，后者是作为某些方法的数据缓冲区的*使用工具*，`Memory<T>`类型旨在*存储*有关缓冲区的信息，而不是用于操作它。这导致了API上的差异：

- `Memory<T>`不包含访问其管理的数据的方法。相反，它有一个`Span`属性和一个`Slice`方法，它们返回工作单元——`Span`类型的实例。
- `Memory<T>`还包含一个`Pin()`方法，该方法用于需要将存储的缓冲区传递给`unsafe`代码的场景。调用它时，如果内存是在.NET中分配的，缓冲区将被固定（pinned），并且在GC触发时不会移动，返回给用户一个封装了固定缓冲区在内存中生命周期概念的`MemoryHandle`结构实例。

不过，首先让我们来了解一下所有类的集合。作为其中的第一个，让我们看看`Memory<T>`结构本身（未展示类型的所有成员，只展示了看起来最重要的部分）：

```csharp
    public readonly struct Memory<T>
    {
        private readonly object _object;
        private readonly int _index, _length;

        public Memory(T[] array) { ... }
        public Memory(T[] array, int start, int length) { ... }
        internal Memory(MemoryManager<T> manager, int length) { ... }
        internal Memory(MemoryManager<T> manager, int start, int length) { ... }

        public int Length => _length & RemoveFlagsBitMask;
        public bool IsEmpty => (_length & RemoveFlagsBitMask) == 0;

        public Memory<T> Slice(int start, int length);
        public void CopyTo(Memory<T> destination) => Span.CopyTo(destination.Span);
        public bool TryCopyTo(Memory<T> destination) => Span.TryCopyTo(destination.Span);

        public Span<T> Span { get; }
        public unsafe MemoryHandle Pin();
    }
```

如我们所见，该结构体包含了一个基于数组的构造函数，但是在object中存储数据。这样做是为了能够额外引用那些没有为其提供构造函数的字符串，但为`string`类型提供了一个扩展方法`AsMemory()`，它返回`ReadOnlyMemory`。然而，由于这两种类型在二进制上必须是相同的，`_object`字段的类型是`Object`。

接下来，我们看到了两个基于`MemoryManager`的构造函数。我们稍后会讨论它们。还有获取大小的`Length`属性和检查是否为空集的`IsEmpty`属性。还有获取子集的`Slice`方法和`CopyTo`以及`TryCopyTo`的复制方法。

更具体地讨论`Memory`时，我想重点讨论这个类型的两个方法：`Span`属性和`Pin`方法。

### Memory\<T>.Span

```csharp
public Span<T> Span
{
    get
    {
        if (_index < 0)
        {
            return ((MemoryManager<T>)_object).GetSpan().Slice(_index & RemoveFlagsBitMask, _length);
        }
        else if (typeof(T) == typeof(char) && _object is string s)
        {
            // 这是危险的，为本应不可变的字符串返回一个可写的span。
            // 然而，我们需要处理从字符串创建ReadOnlyMemory<char>然后转换为Memory<T>的情况。
            // 这样的转换只能通过不安全或者封送代码完成，
            // 在这种情况下，那是开发者执行的危险操作，我们只是尽可能地跟进这里使其工作。
            return new Span<T>(ref Unsafe.As<char, T>(ref s.GetRawStringData()), s.Length).Slice(_index, _length);
        }
        else if (_object != null)
        {
            return new Span<T>((T[])_object, _index, _length & RemoveFlagsBitMask);
        }
        else
        {
            return default;
        }
    }
}
```

更具体地说，关于处理字符串的代码行。因为它们说明了，如果我们以某种方式将`ReadOnlyMemory<T>`转换为`Memory<T>`（而在二进制表示中，这两者是相同的。不仅如此，还有一个评论警告说，这两种类型在二进制上必须匹配，因为一个是通过调用`Unsafe.As`从另一个获得的），那么我们就获得了访问秘密房间的钥匙，即更改字符串的能力。这是一个极其危险的机制：

```csharp
unsafe void Main()
{
    var str = "Hello!";
    ReadOnlyMemory<char> ronly = str.AsMemory();
    Memory<char> mem = (Memory<char>)Unsafe.As<ReadOnlyMemory<char>, Memory<char>>(ref ronly);
    mem.Span[5] = '?';

    Console.WriteLine(str);
}
---
Hello?
```

这与字符串的内部化一起，可能导致非常糟糕的后果。

### Memory\<T>.Pin

另一个引起真正兴趣的方法是`Pin`方法：

```csharp
public unsafe MemoryHandle Pin()
{
    if (_index < 0)
    {
        return ((MemoryManager<T>)_object).Pin((_index & RemoveFlagsBitMask));
    }
    else if (typeof(T) == typeof(char) && _object is string s)
    {
        // 该情况只有在围绕字符串创建了一个ReadOnlyMemory<char>，
        // 然后通过不安全/封送代码将其转换为Memory<char>时才会发生。
        // 然而，这需要工作，以便使用单个Memory<char>字段来存储可读的ReadOnlyMemory<char>
        // 或可写的Memory<char>的代码仍然可以被固定并用于互操作目的。
        GCHandle handle = GCHandle.Alloc(s, GCHandleType.Pinned);
        void* pointer = Unsafe.Add<T>(Unsafe.AsPointer(ref s.GetRawStringData()), _index);
        return new MemoryHandle(pointer, handle);
    }
    else if (_object is T[] array)
    {
        // 数组已经预先固定
        if (_length < 0)
        {
            void* pointer = Unsafe.Add<T>(Unsafe.AsPointer(ref array.GetRawSzArrayData()), _index);
            return new MemoryHandle(pointer);
        }
        else
        {
            GCHandle handle = GCHandle.Alloc(array, GCHandleType.Pinned);
            void* pointer = Unsafe.Add<T>(Unsafe.AsPointer(ref array.GetRawSzArrayData()), _index);
            return new MemoryHandle(pointer, handle);
        }
    }
    return default;
}
```

它也是一个非常重要的统一工具：因为无论`Memory<T>`引用的数据类型是什么，如果我们想将缓冲区传递给非托管代码，我们需要做的唯一事情就是调用`Pin()`方法并传递一个指针，该指针将存储在获得的结构的属性中，传递给非托管代码：

```csharp
void PinSample(Memory<byte> memory)
{
    using(var handle = memory.Pin())
    {
        WinApi.SomeApiMethod(handle.Pointer);
    }
}
```

在这段代码中，调用`Pin()`的原因没有任何区别：无论是对`T[]`、`string`还是对非托管内存缓冲区的`Memory`。对于数组和字符串，将创建一个真正的`GCHandle.Alloc(array, GCHandleType.Pinned)`，而对于非托管内存，就什么都不会发生。

## MemoryManager, IMemoryOwner, MemoryPool

除了指出结构体的字段外，我还想额外指出，存在另外两个`internal`类型的构造器，它们基于另一个实体——`MemoryManager`工作，这个将会在稍后讨论，而且它并不是你可能刚刚想到的那样：在传统意义上的内存管理器。然而，就像`Span`一样，`Memory`同样包含了一个对象的引用，用于导航，以及内部缓冲区的偏移量和大小。另外还需要注意的是，`Memory`只能通过`new`操作符基于数组创建，以及通过扩展方法基于字符串、数组和`ArraySegment`创建。即，它不支持基于手动管理的内存创建。然而，我们看到，存在一种内部方法，基于`MemoryManager`创建这个结构体：

**文件 [MemoryManager.cs](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/Buffers/MemoryManager.cs)**

```csharp
public abstract class MemoryManager<T> : IMemoryOwner<T>, IPinnable
{
    public abstract MemoryHandle Pin(int elementIndex = 0);
    public abstract void Unpin();

    public virtual Memory<T> Memory => new Memory<T>(this, GetSpan().Length);
    public abstract Span<T> GetSpan();
    protected Memory<T> CreateMemory(int length) => new Memory<T>(this, length);
    protected Memory<T> CreateMemory(int start, int length) => new Memory<T>(this, start, length);

    void IDisposable.Dispose()
    protected abstract void Dispose(bool disposing);
}
```

它封装了内存段所有者的概念。换句话说，如果`Span`是用于操作内存的工具，而`Memory`是用于存储有关特定内存段的信息的工具，那么`MemoryManager`就是它的生命周期控制工具，它的所有者。例如，可以考虑`NativeMemoryManager<T>`类型，虽然它是为测试编写的，但它很好地反映了“所有权”的概念：

**文件 [NativeMemoryManager.cs](https://github.com/dotnet/corefx/blob/888088448ac5dd1053d88434dfd819dcbc0fd9a1/src/Common/tests/System/Buffers/NativeMemoryManager.cs)**

```csharp
internal sealed class NativeMemoryManager : MemoryManager<byte>
{
    private readonly int _length;
    private IntPtr _ptr;
    private int _retainedCount;
    private bool _disposed;

    public NativeMemoryManager(int length)
    {
        _length = length;
        _ptr = Marshal.AllocHGlobal(length);
    }

    public override void Pin() { ... }

    public override void Unpin()
    {
        lock (this)
        {
            if (_retainedCount > 0)
            {
                _retainedCount--;
                if (_retainedCount == 0)
                {
                    if (_disposed)
                    {
                        Marshal.FreeHGlobal(_ptr);
                        _ptr = IntPtr.Zero;
                    }
                }
            }
        }
    }

    // 其他方法
}
```

即，换句话说，这个类提供了嵌套调用`Pin()`方法的可能性，从而计算出来自`unsafe`世界的引用。

与`Memory`紧密相关的另一个实体是`MemoryPool`，它提供了`MemoryManager`实例的池化（实际上是`IMemoryOwner`）：

**文件 [MemoryPool.cs](https://github.com/dotnet/corefx/blob/f592e887e2349ed52af6a83070c42adb9d26408c/src/System.Memory/src/System/Buffers/MemoryPool.cs)**

```csharp
public abstract class MemoryPool<T> : IDisposable
{
    public static MemoryPool<T> Shared => s_shared;

    public abstract IMemoryOwner<T> Rent(int minBufferSize = -1);

    public void Dispose() { ... }
}
```

它旨在提供临时使用的所需大小的缓冲区。实现`IMemoryOwner<T>`接口的租用实例有一个`Dispose()`方法，该方法将租用的数组返回到数组池中。默认情况下，您可以使用基于`ArrayMemoryPool`构建的公共缓冲池：

**文件 [ArrayMemoryPool.cs](https://github.com/dotnet/corefx/blob/56dfb8834fa50f3bc61ea9b4bfdc9dcc759b6ec9/src/System.Memory/src/System/Buffers/ArrayMemoryPool.cs)**

```csharp
internal sealed partial class ArrayMemoryPool<T> : MemoryPool<T>
{
    private const int MaximumBufferSize = int.MaxValue;
    public sealed override int MaxBufferSize => MaximumBufferSize;
    public sealed override IMemoryOwner<T> Rent(int minimumBufferSize = -1)
    {
        if (minimumBufferSize == -1)
            minimumBufferSize = 1 + (4095 / Unsafe.SizeOf<T>());
        else if (((uint)minimumBufferSize) > MaximumBufferSize)
            ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.minimumBufferSize);

        return new ArrayMemoryPoolBuffer(minimumBufferSize);
    }
    protected sealed override void Dispose(bool disposing) { }
}
```

基于这种架构，可以描绘出以下世界观：

- 如果您在方法参数中需要使用数据类型`Span`，表示您要么是读取数据（`ReadOnlySpan`），要么是读取或写入数据（`Span`）。但不是将其保存在类字段中以供将来使用的任务
- 如果您需要从类字段中存储对数据缓冲区的引用，应该使用`Memory<T>`或`ReadOnlyMemory<T>`，具体取决于您的目的
- `MemoryManager<T>`是数据缓冲区的所有者（可以根据需要不使用）。当需要计算`Pin()`调用次数时，或者需要知道如何释放内存时，就需要它
- 如果`Memory`是围绕非托管内存块构建的，`Pin()`不会做任何事情。然而，这使得与不同类型的缓冲区工作统一化：无论是在托管代码还是非托管代码的情况下，交互接口都是相同的
- 每种类型都有公共构造函数。这意味着您可以直接使用`Span`，也可以从`Memory`获取其实例。您可以单独创建`Memory`，也可以为其组织一个`IMemoryOwner`类型，该类型将拥有`Memory`将引用的内存块。一个特殊情况可能是任何基于`MemoryManager`的类型：对内存块的某种局部拥有（例如，从`unsafe`世界中计算引用）。
如果此时需要这些缓冲区的池化（预期有大量大致相同大小的缓冲区频繁流动），可以使用`MemoryPool`类型。
- 如果您预计需要处理`unsafe`代码，向其传递某种数据缓冲区，应该使用`Memory`类型：它具有`Pin()`方法，自动固定.NET堆中创建的缓冲区。
- 如果您有缓冲区流量（例如，您正在解决程序文本解析或某种DSL的任务），应该使用`MemoryPool`类型，这可以非常恰当地组织，从池中提供合适大小的缓冲区（例如，如果找不到合适的，可以稍微大一些，但使用`originalMemory.Slice(requiredSize)`来避免池碎片化）

## 性能

为了理解新数据类型的性能，我决定使用已成为标准的库 [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)：

```csharp
[Config(typeof(MultipleRuntimesConfig))]
public class SpanIndexer
{
    private const int Count = 100;
    private char[] arrayField;
    private ArraySegment<char> segment;
    private string str;

    [GlobalSetup]
    public void Setup()
    {
        str = new string(Enumerable.Repeat('a', Count).ToArray());
        arrayField = str.ToArray();
        segment = new ArraySegment<char>(arrayField);
    }

    [Benchmark(Baseline = true, OperationsPerInvoke = Count)]
    public int ArrayIndexer_Get()
    {
        var tmp = 0;
        for (int index = 0, len = arrayField.Length; index < len; index++)
        {
            tmp = arrayField[index];
        }
        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public void ArrayIndexer_Set()
    {
        for (int index = 0, len = arrayField.Length; index < len; index++)
        {
            arrayField[index] = '0';
        }
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public int ArraySegmentIndexer_Get()
    {
        var tmp = 0;
        var accessor = (IList<char>)segment;
        for (int index = 0, len = accessor.Count; index < len; index++)
        {
            tmp = accessor[index];
        }
        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public void ArraySegmentIndexer_Set()
    {
        var accessor = (IList<char>)segment;
        for (int index = 0, len = accessor.Count; index < len; index++)
        {
            accessor[index] = '0';
        }
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public int StringIndexer_Get()
    {
        var tmp = 0;
        for (int index = 0, len = str.Length; index < len; index++)
        {
            tmp = str[index];
        }

        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public int SpanArrayIndexer_Get()
    {
        var span = arrayField.AsSpan();
        var tmp = 0;
        for (int index = 0, len = span.Length; index < len; index++)
        {
            tmp = span[index];
        }
        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public int SpanArraySegmentIndexer_Get()
    {
        var span = segment.AsSpan();
        var tmp = 0;
        for (int index = 0, len = span.Length; index < len; index++)
        {
            tmp = span[index];
        }
        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public int SpanStringIndexer_Get()
    {
        var span = str.AsSpan();
        var tmp = 0;
        for (int index = 0, len = span.Length; index < len; index++)
        {
            tmp = span[index];
        }
        return tmp;
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public void SpanArrayIndexer_Set()
    {
        var span = arrayField.AsSpan();
        for (int index = 0, len = span.Length; index < len; index++)
        {
            span[index] = '0';
        }
    }

    [Benchmark(OperationsPerInvoke = Count)]
    public void SpanArraySegmentIndexer_Set()
    {
        var span = segment.AsSpan();
        for (int index = 0, len = span.Length; index < len; index++)
        {
            span[index] = '0';
        }
    }
}

public class MultipleRuntimesConfig : ManualConfig
{
    public MultipleRuntimesConfig()
    {
        Add(Job.Default
            .With(CsProjClassicNetToolchain.Net471) // Span 不支持 CLR
            .WithId(".NET 4.7.1"));

        Add(Job.Default
            .With(CsProjCoreToolchain.NetCoreApp20) // Span 支持 CLR
            .WithId(".NET Core 2.0"));

        Add(Job.Default
            .With(CsProjCoreToolchain.NetCoreApp21) // Span 支持 CLR
            .WithId(".NET Core 2.1"));

        Add(Job.Default
            .With(CsProjCoreToolchain.NetCoreApp22) // Span 支持 CLR
            .WithId(".NET Core 2.2"));
    }
}
```

接下来我们来看看结果：

![性能图表](./imgs/Span/Performance.png)

通过它们，我们可以得到以下信息：

  - ArraySegment表现很糟糕。但是，通过将其封装在`Span`中，可以显著改善其性能，提升达到7倍；
  - 如果考虑`.NET Framework 4.7.1`（对于4.5来说也是类似的），那么转向Span会明显降低数据缓冲区的处理性能，大约下降30-35%；
  - 然而，如果看向.NET Core 2.1+，那里的性能将会相当，考虑到Span可以在数据缓冲区的一部分上工作，创建上下文，那么实际上会更快：因为ArraySegment具有类似的功能，但其运行速度极慢。

由此，我们可以得出关于使用这些数据类型的简单结论：

  - 对于`.NET Framework 4.5+`和`.NET Core`，使用它们的唯一优点是：它们比ArraySegment更快，尤其是在原始数组的子集上；
  - 对于`.NET Core 2.1+`，使用它们将提供无可争议的优势，不仅超过了使用ArraySegment，也超过了任何类型的手动实现`Slice`
  - 此外，没有任何一种数组统一方法能达到这三种方式的最大性能。