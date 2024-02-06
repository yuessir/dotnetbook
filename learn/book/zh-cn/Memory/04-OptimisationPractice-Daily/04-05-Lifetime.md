## 生命周期模板

在我们如此详细地探讨了名为IDisposable的资源释放模式之后，我们指出了这一模式的一系列问题区域，这些问题使得使用它变得复杂。为了解决这些问题，我们需要开发一些对策，这将引导我们走向一个全新且强大的工具。那么，如果我们想要修改IDisposable模式或用其他东西替换它，同时消除其固有的缺点，我们应该怎么做呢？让我们从上一章中提到的一组缺点开始，尝试在可能的情况下改变行为，以便获得优势：

1. 为了避免对象销毁性的声明不明确，我们需要引入一种新的通知方式，表明类是可销毁的。而且，必须确保实例的用户无法绕过这一机制。那么，实例的哪种代码无论如何都会执行呢？只有构造函数。这意味着，构造函数必须具有某些属性，这些属性能够提示调用它的代码实例是可销毁的。这里可以有两种机制：要么类名必须包含关键字（例如，与`Async`类似，使用`Disposable`），要么必须有一个参数需要传递给构造函数；
2. 拖着某个接口以便销毁实体的问题，可以通过将实体的生命周期控制从实体本身中移出来解决。本质上，是通过将对象销毁过程的控制反转到某个外部机制来实现；
3. 第三个问题是对象销毁后的存在。遗憾的是，这个问题简单无法解决：对象底层内存的不确定删除使得这个任务无解；
4. 如果不实现`IDisposable`，那么关于`explicit`实现`IDisposable`的问题就自然消失了；
5. 资源分配和释放分布在不同方法中的问题，可以通过在某个容器中注册资源并自动释放这些资源来解决；
6. 如果讨论销毁分层对象图的复杂性，这些层自行销毁（例如，可能需要销毁图的某一层，而其他层需要继续正确存在），通过取消订阅相邻层的变化，可以明确指出一个共同特点：无论讨论什么，逻辑上我们都在讨论某些对象组，它们的生命周期相互依赖。即，通过引入一个实体的生命周期依赖于另一个实体的生命周期的概念，我们可以解决清理特定对象组而不影响另一组的问题；

将上述所有内容综合起来，我们可以得出一个基本结论：IDisposable并不提供我们完全灵活地管理对象状态销毁的能力。它无疑在某些场景下非常有用，但并不是一个万能工具，能够解决所有问题。

那么，让我们找到那个旨在帮助我们销毁复杂对象结构的机制。因此，基于之前关于改进IDisposable模式的讨论，我们列出以下论点：

  - 我们的类要么需要有特殊的命名，要么需要接受某些参数作为构造函数的参数；
  - 销毁过程需要被外部化；
  - 销毁过程需要被自动化；
  - 还应该存在某种对象生命周期之间的依赖关系；
  - 我们不应该每次都编写整个基础设施，就像我们对待IDisposable那样；

因此：

  - 销毁过程应由一个单独的实体负责。我们称之为`Lifetime`；
  - 类的构造函数应该接受一个`Lifetime`类型的实例作为输入，如果我们类的实例的生命周期依赖于外部`Lifetime`的生命周期，或者如果不存在依赖关系，则创建自己的`Lifetime`；

现在，在新的情况下，外部代码将向我们传递一个特殊的`Lifetime`实例，我们将依赖于它，并根据它的生命周期存在。而为了销毁我们，外部类型不会直接销毁我们：相反，它会结束`Lifetime`实例的生命周期，后者将自行结束所有订阅者的生命周期。

采用这种方法，我们不再让任何拥有我们对象链接的人有机会摧毁我们。还记得我们在IDisposable模板中有什么吗？在设计类时，我们必须实现IDisposable接口：为所有人公开`Dispose()`方法。这意味着任何获得我们类型实例链接的人也获得了销毁我们类型实例的手段。现在情况完全不同了：类型实例的构造函数将自身生命周期作为外部依赖的参数接收。同时，不向外暴露任何方法。这意味着在这种情况下，只有创建它的人才拥有我们类型实例的生命周期控制权，其他人则没有。使用IDisposable模板，绕过公开方法的唯一途径是隐式实现该接口，从而隐藏`Dispose`方法。然而，接口的隐式实现会引发许多问题，因为现代IDE允许轻松简单地看到类型结构中该方法的存在。

现在还剩下一个问题需要解决：责任区域的划分。因为如果我们需要将自己的`Lifetime`传递给别人，我们不希望放弃控制调用终止方法的能力，该方法将结束所有订阅了传递的`Lifetime`实例的人的生命。

同样，为了防止使用`Lifetime`的实体自行中断其生命周期，需要在过程参与者之间引入责任区域的划分：

1. `LifetimeDef`。这种类型的实例存储在将要拥有其依赖对象生命周期结束的对象中；
2. `Lifetime`由`LifetimeDef`实例创建，并将被传递给所有以某种方式依赖于所有者的人。它们自己不能调用`Terminate`方法：这个方法只对`LifetimeDef`可用。获得`Lifetime`的类型实例的责任仅仅是向其中添加将会销毁它们的行为。
3. `OuterLifetime` -- 封装了readonly `Lifetime`的概念。换句话说，这是一种保护措施，防止最终程序员能够向传递的依赖中添加销毁自己的行为。当您将`Lifetime`实例交给其他人时，使用这种类型，以便相对于它只能创建新的依赖的`LifetimeDef`。但添加自己的操作将是不可能的。公开地，像`Lifetime`一样，`OuterLifetime`只包含`IsTerminated`属性，但基于它可以创建依赖的`LifetimeDef`实例，基于此可以管理自己实例的生命周期。

让我们考虑类型接口的最基本变体：

```csharp
public class Lifetime
{
    public static Lifetime Eternal = Define("Eternal").Lifetime;

    public bool IsTerminated { get; internal set; }

    public void Add(Action action);
}
```

我们在这里看到了什么？有一个新的实体，`Lifetime`。这个实体有一个`IsTerminated`属性，旨在帮助理解Lifetime及其依赖者的生命周期是否已经结束。可以通过使用`Add`方法将一系列在`Lifetime`实例死亡时应该发生的行为添加到内部列表中。

我们还来看看这个模式的另一个基石：拥有结束`Lifetime`实例生命周期权利的`LifetimeDef`类。

```csharp
public class LifetimeDef : IDisposable
{
    public Lifetime Lifetime { get; }
    public string Name { get; }

    private const string Noname = "Unnamed";

    public LifetimeDef(string name = null)
    {
        Name = name ?? Noname;
        Lifetime = new Lifetime();
    }

    public void Terminate()
    {
        Lifetime.Terminate();
    }

    public void Dispose()
    {
        Terminate();
    }
}
```

为了方便起见，它包含了一个可选字段`Name`。这样做是为了在调试时，多个`LifetimeDef`实例不会引起混淆，能够轻松地找到所需的实体。`Terminate`方法终止`Lifetime`类实例的生命周期，而`IDisposable`接口的实现允许在经典场景中使用`LifetimeDef`。例如，在`using`块中：

```csharp
[Test]
void EntityTest()
{
    Entity entity;
    using(var ldf = Lifetime.Define())
    {
        entity = new Entity(ldf.Lifetime);
        entity.OpenConnection();
        //...
    }

    Assert.Equal(State.Closed, entity.ConnectionState);
}
```

## 使用场景

为了更好地理解我们所处理的内容，我建议考虑一些例子。

### 示例 1

```csharp

public class Subscriber
{
    public Subscriber(DataSource dataSource, Lifetime lft)
    {
        dataSource.DataArrived.Add(this.OnDataArrived);
        lft.Add(() => dataSource.DataArrived.Remove(this.OnDataArrived));
    }

    // 其他方法
}

```