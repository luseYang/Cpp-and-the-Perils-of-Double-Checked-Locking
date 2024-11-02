# 前言

该文章只是对于《C++ and the Perils of Double-Checked Locking》这篇论文的全文翻译加上一些我写的鬼话，论文作者就是我们 C++ 程序员所熟知的 Scott Meyers 和 Andrei Alexandrescu，他们真的有很多好的作品，即使这些书有一些跟不上时代了，但一定比某些华而不实的走量不走心的教程好，推荐阅读学习。需要注意的是这篇论文创作于 2004 年，有一些设施在 C++11 或更新的 C++ 版本中或许已经司空见惯，但当时确实是不存在于标准当中的，带着这种想法阅读本文，一定可以体验到编程的有趣。

当然，开头第一句引言就很有意思：Multithreading is just one damn thing after, before, or simultaneous with another. 自己翻译一下吧

# 1 描述

在网络上搜索各种设计模式，你肯定会发现最常提到的一种是 `Singleton`。然而，尝试将 `Singleton` 付诸实践，你包会遇到一个常见的问题：按照传统实现方式（我们将在下面解释），`Singleton` 不是线程安全的。

人们付出了很多努力来解决这一缺陷。其中最流行的方法之一就是双重检查锁定模式 (DCLP)。DCLP 旨在为共享资源（如单例）的初始化添加有效的线程安全性，但它有一个问题：它不可靠。此外，在不大幅修改传统模式实现的情况下，几乎没有任何可移植的方法可以在 C++（或 C 语言）中使其可靠。更有趣的是，DCLP 可能会因不同的原因在单处理器和多处理器架构上失败。所以面试的时候尽量避免说双检锁实现 `Singleton` 或者说清楚错误的原因。

本文解释了 `Singleton` 为何不是线程安全的，DCLP 如何尝试解决该问题，DCLP 为何在单处理器和多处理器架构上都可能失败，以及为什么您无法（可移植地）对此采取任何措施。在此过程中，它阐明了源代码中的语句顺序、序列点、编译器和硬件优化以及语句执行的实际顺序之间的关系。最后，它总结了一些关于如何向 Singleton（和类似构造）添加线程安全性的建议，以使生成的代码既可靠又高效。

# 2 单例模式和多线程

单例模式的传统实现是在第一次请求对象时让指针指向一个新对象：

*这是 2004 年 7 月（第一部分）和 8 月（第二部分）《Dr. Dobbs Journal》上发表的一篇文章的略微修改版本*

```cpp
// from the header file
class Singleton {
public:
    static Singleton* instance();
    ...
private:
    static Singleton*pInstance;
};

// from the implementation file
Singleton* Singleton::pInstance = 0;

Singleton* Singleton::instance(){
    if(pInstance == 0){
        pInstance = new Singleton;
    }
    return pInstance;
}
```

在单线程环境中，这通常可以正常工作，但如果触发中断可能会有问题。如果您在 `Singleton::instance` 中，收到中断，并从处理程序调用 `Singleton::instance`，您就可以看到自己会遇到什么样的麻烦。

然而，除了中断之外，此实现在单线程环境中工作正常，但不幸的是，这种实现在**多线程环境下并不可靠**。假设线程 A 进入实例函数，执行到第 14 行 `if(pInstance == 0)`，然后被挂起。在被挂起的地方，它刚刚确定 `pInstance` 为 `null`，即尚未创建任何 `Singleton` 对象。线程 B 现在进入实例并执行第 14 行。它看到 `pInstance` 为空，因此继续执行第 15 行并创建一个供 `pInstance` 指向的单例。然后它将 `pInstance` 返回给实例的调用，稍后，线程 A 被允许继续运行，它做的第一件事就是移动到第 15 行，在那里它召唤出另一个 `Singleton` 对象并让 `pInstance` 指向它。很明显，这违反了单例的含义，因为现在有两个 `Singleton` 对象。从技术上讲，第 11 行 `Singleton* Singleton::pInstance = 0` 是 `pInstance` 初始化的地方，但从实际目的来看，第 15 行 `pInstance = new Singleton;` 才是让它指向我们想要的位置，因此在本文的其余部分，我们将第 15 行视为 `pInstance` 初始化的位置。让经典的 `Singleton` 实现线程安全很容易。只需在检测 `pInstance` 之前获取一个锁即可：

```cpp
Singleton* Singleton::instance(){
    Lock lock;     // acquire lock (params omitted for simplicity)
    if(pInstance == 0){
        pInstance = new Singleton;
    }
    return pInstance; 
}        // release lock (via Lock destructor)
```

这种解决方案的缺点是它可能很开销很大。每次访问 `Singleton` 都需要获取锁(我们简化了锁的写法，这里如果不想手动 `unlock` 的话可以使用 `lock_gurd`)，但实际上，我们仅在初始化 `pInstance` 时才需要锁。这应该只在第一次调用 `instance` 时发生。如果在程序运行过程中调用了 `n` 次 `instance` ，我们只需要在第一次调用时获取锁。既然您知道其中 `n - 1` 次是不必要的，为什么还要为 `n` 次锁付出额外的开销呢？DCLP 旨在防止您这样做。

# 3 DCLP

DCLP 的关键在于我们发现大多数对实例的调用都会得出 **`pInstance` 不为空**，因此甚至不会尝试初始化它。因此，DCLP 在尝试获取锁之前会检测 `pInstance` 是否为空。**只有检测成功（即，如果 `pInstance` 尚未初始化）才会获取锁**，之后会再次执行测试以确保 `pInstance` 仍为空（因此称为双重检查锁定）。第二个测试是必要的，因为正如我们刚刚看到的，在第一次测试 `pInstance` 和获取锁之间，可能恰好有另一个线程初始化了 `pInstance`。

以下是经典的 DCLP 实现：

```cpp
Singleton* Singleton::instance(){
    if(pInstance==0){    // 1st test
        Lock lock;
    if(pInstance ==0){    // 2nd test
        pInstance=new Singleton;
    }
    return pInstance,
```

定义 DCLP 讨论了一些实现问题（例如，对单例指针进行 `volatile` 限定的重要性以及单独缓存对多处理器系统的影响，我们将在下文中讨论这些问题；以及确保某些读写的原子性的必要性，我们在本文中不讨论这些问题），但它们没有考虑一个更基本的问题，即**确保在 DCLP 期间执行的机器指令以可接受的顺序执行**(人话就是要考虑指令重排引发的问题)。

# 4 DCLP 和 指令重排

再次考虑初始化 pInstance 的那一行：

```cpp
pInstance = new Singleton;
```

接下来的话引起重视，**此语句会导致三件事发生**：1. 分配内存以保存 `Singleton` 对象。2. 在分配的内存中构造 `Singleton` 对象。3. 使 `pInstance` 指向分配的内存。

**至关重要的是，编译器并不一定要按此顺序执行这些步骤！** 具体来说，编译器有时可以交换步骤 2 和步骤 3。“为什么它们会这样做”，别急我们稍后会回答这个问题。现在，让我们关注一下如果它们这样做会发生什么？

考虑以下代码，我们将 `pInstance` 的初始化行详细扩展为上面提到的三个组成任务，并将步骤 1（内存分配）和步骤 3（`pInstance` 赋值）合并为一个位于步骤 2（单例构造）之前的语句。我们的想法其实不是人类会编写的代码。相反，而是编译器可能会生成与此等效的代码，以响应人类编写的传统 DCLP 源代码（如前所示），对这很抽象。

```cpp
Singleton* Singleton::instance(){
    if(pInstance == 0){
        Lock lock;
        if(pInstance == 0){
            pInstance =                             // Step 3------->合
                operator new(sizeof(Singleton));    // Step 1------->并
            new(pInstance) Singleton;               // Step 2
        }
    }
    return pInstance;
}
```

一般来说，这不是原始 DCLP 源代码的有效转换，因为步骤 2 中调用的 `Singleton` 构造函数可能会抛出异常，如果抛出异常，则重要的是 `pInstance` 尚未被修改。这就是为什么编译器**不能**将步骤 3 移到步骤 2 之上。但是，在某些情况下，这种转换是合法的。也许最简单的条件是编译器可以证明 `Singleton` 构造函数不能抛出异常（例如，通过[后内联流分析](#后内联流分析)），但这不是唯一的条件。一些抛出的构造函数也可能使其指令重新排序，从而出现此问题。

根据上述解释，考虑以下事件序列：

- 线程A进入 `instance`，对 `pInstance `进行第一次测试，获取锁，执行步骤 1 和 3 的语句，然后被挂起，此时 `pInstance` 不为 `null`，但是 `pInstance `指向的内存中还没有构造 `Singleton` 对象。
- 线程 B 进入实例，确定 `pInstance` 非空，并将其返回给实例的调用者。然后，调用者取消引用指针以访问尚未构造的 `Singleton`。

只有在步骤 3 之前完成步骤 1 和 2 时，DCLP 才会起作用，**但是没有办法用 C 或 C++ 表达这种约束**。这就是 DCLP 的致命弱点：我们需要定义相对指令顺序的约束，但我们的语言没有办法表达这种约束。是的，C 和 C++ 标准确实定义了序列点，它定义了对求值顺序的约束。例如，C++ 标准第 1.9 节第 7 段指出：

> At certain specified points in the execution sequence called sequence points, all side effects of previous evaluations shall be complete and no side effects of subsequent evaluations shall have taken place.

> 在执行序列中的某些指定点（称为序列点）处，先前评估的所有副作用都应完成，并且后续评估的任何副作用都应未发生。

此外，两个标准都规定，序列点出现在每个语句的末尾。因此，只要小心谨慎地对语句进行排序，一切就都水到渠成了。这两个标准都根据抽象机器的可观察行为来定义正确的程序行为。但并非有关该机器的所有事情都是可观察的。例如，考虑这个简单的函数：

```cpp
void Foo(){
    int x = 0, y = 0;            // Statement 1
    x = 5;                       // Statement 2
    y = 10;                      // Statement 3
    printf("%d, %d", x, y);      // Statement 4
}
```

这个函数看起来很呆，但它可能是内联 `Foo` 调用的一些其他函数的结果。

在 C 和 C++ 中，标准都保证 `Foo` 将打印“5, 10”，因此我们知道这会发生。但这就是我们所保证的程度，因此也是我们所知道的程度。我们不知道语句 1-3 是否会被执行，事实上，一个好的优化器会摆脱它们。如果执行了语句 1-3，我们知道语句 1 将先于语句 2-4，并且假设对 `printf` 的调用未内联且结果进一步优化，我们知道语句 4 将跟在语句 1-3 之后，但我们对语句 2 和 3 的相对顺序一无所知。编译器可能会选择先执行语句 2，先执行语句 3，甚至并行执行它们？假设硬件有某种方式来做到并行这一点。

> 它是很有可能的。现代处理器具有很大的字长和多个执行单元。两个或更多的算术单元很常见。 （例如，`Pentium` 4 有三个整数 `ALU`，`PowerPC` 的 `G4e` 有四个，而 `Itanium` 有六个。）它们的机器语言允许编译器生成在单个时钟周期内并行执行两个或多个指令的代码。

优化编译器会仔细分析并重新排序代码，以便尽可能多地同时执行任务（在可观察行为的限制范围内）。在常规串行代码中发现和利用这种并行性是重新排列代码吗，并引入乱序执行的最重要原因。但这不是唯一的原因。编译器（和链接器）也可能重新排序指令，以避免溢出寄存器中的数据，保持指令流水线满载，执行公共子表达式消除，并减少生成的可执行文件的大小等等等等（编译器所做的优化太多了，我也不是很懂）。

在执行这些类型的优化时，C 和 C++ 的编译器和链接器仅受语言标准定义的抽象机器上可观察行为的约束，这一点很重要，这些抽象机器是隐式单线程的。**作为语言，C 和 C++ 都没有线程，因此编译器在优化时不必担心破坏线程程序**。希望你们可以理解这段话，因此，有时它们会破坏线程程序，这应该并不奇怪。

既然如此，如何编写真正能运行的 C 和 C++ 多线程程序呢？使用为此目的定义的系统特定库。像 `Posix` 线程 (pthreads)这样的库为各种同步原语的执行语义提供了精确的规范。这些库对符合库要求的编译器可以生成的代码施加了限制，从而迫使这些编译器生成遵守这些库所依赖的执行顺序约束的代码。这就是为什么线程包中有部分是用汇编语言编写的，或者发出用汇编语言（或某种不可移植的语言）编写的系统调用的原因：您必须超越标准 C 和 C++ 来表达多线程程序所需的顺序约束。DCLP 试图仅使用语言结构来实现，从逻辑的角度来说就是站不住脚的。这就是 DCLP 不可靠的原因。

一般来说，程序员不喜欢被编译器摆布。也许吧你就是这样的程序员。如果是这样，你可能会试图通过调整源代码来越过一些编译器优化，使 `pInstance` 保持不变，直到 `Singleton` 的构造完成。想想一些有趣吧办法，例如，你可以尝试插入临时变量的使用：

```cpp
Singleton* Singleton::instance(){
    if(pInstance == 0){
        Lock lock;
        if(pInstance == 0){
            Singleton* temp = new Singleton;    // initialize to temp
            pInstance = temp;                   // assign temp to pInstance
        }
    }
    return pInstance;
}
```

相信到这里你来感觉了，因为这触及到了编译器的优化战争，是程序员和编译器之间的交互，让你自己真正有了能控制编译器作为得出自己想要的结果的工具的想法。（原谅我有点抽象）。你的编译器想要优化，但你不希望它优化，至少在这里不希望。但这不是你想卷入的战斗。*你的敌人狡猾而老练，他们满脑子都是几十年来那些整天、日复一日、年复一年思考这类事情的人想出的战略。除非你自己编写优化编译器，否则他们远远领先于你*

哈哈，其实在这种情况下，对于编译器来说，它会应用依赖性分析来确定 `temp` 是一个不必要的变量，从而将其消除，对于编译器来说这是一件很简单的事情，你的想法就像你精心编写的“不可优化”的代码一样，在顶尖程序员所贡献的编译器面前不堪一击，游戏结束。你输了。如果您再狠一点并尝试将 `temp` 移至更大的范围（例如将其设为静态文件），编译器仍然可以执行相同的分析并得出相同的结论。范围(scope)，傻瓜(schmope)。你又输了。

所以呢你请求一个备份，无所不用其极，声明 `temp extern` 并在单独的翻译单元中定义它，从而阻止你的编译器看到你正在做的事情。遗憾的是，有些编译器有“透视眼”的优化功能：它们执行过程间分析，发现你使用 `temp` 的诡计，然后再次优化它以使其不复存在。请记住，这些都是编译器优化。它们被设计就是应该用来追踪不必要的代码并将其消除，在绝大多数情况下这是利好程序员的。好嘛游戏结束。你又又输了。

因此，您尝试通过在另一个文件中定义辅助函数来禁用内联，从而迫使编译器假设构造函数可能会引发异常，因此延迟对 `pInstance` 的赋值，能想到这里的都不是泛泛之辈了。不错的尝试，但某些构建环境会执行**链接时内联**，然后进行更多代码优化。**游戏结束。你又又又输了**。

您所做的一切都无法改变根本问题：您需要能够指定指令排序的约束，但您的语言却无法做到这一点（本论文作者作于2004年，当时在 C++ 语言层面并没有实现如 `atomic`之类的操作，内存序也是C++11提出的）。

# 5 不日成名，volatile 关键字

对于特定指令的顺序需求使得许多人怀疑 `volatile` 关键字是否有助于多线程处理，尤其是在 DCLP 中。在本节中，我们将注意力集中在 C++ 中 `volatile` 的语义上，进一步讨论限制在其对 DCLP 的影响上。有关 `volatile` 的更广泛讨论，请参阅[volitale简史](#volatile简史)

C++ 标准的 1.9 节包含了此信息：
> The observable behavior of the [C++] abstract machine is its sequence of reads and writes to volatile data and calls to library I/O functions.
> 
> Accessing an object designated by a volatile lvalue, modifying an object, calling a library I/O function, or calling a function that does any of those operations are all side effects, which are changes in the state of the execution environment

> [C++] 抽象机的[可观察行为](#可观察行为)是其对易失性数据的读写序列以及对库 I/O 函数的调用。访问由易失性左值指定的对象、修改对象、调用库 I/O 函数或调用执行任何这些操作的函数都是[副作用](#副作用)，即执行环境状态的变化

结合我们之前的观察，（1）标准保证在达到序列点时所有副作用都会发生，（2）序列点出现在每个 C++ 语句的末尾，看来我们需要做的就是确保正确的指令顺序，即对适当的数据进行 `volatile` 限定，并仔细对语句进行排序

我们之前的分析表明，`pInstance` 需要声明为 `volatile`，事实上这一点在 DCLP 的论文中也有提及。然而，如果你是大侦探夏洛特福尔摩斯肯定会注意到，为了确保正确的指令顺序，`Singleton` 对象本身也必须是 `volatile` 的。这一点在原始 DCLP 论文中没有提到，这是一个重要的疏忽

要理解为什么仅仅将 `pInstance` 声明为 `volatile` 是不够的，请考虑以下情况：

```cpp
class Singleton {
public:
	static Singleton* instance();
    ...
private:
	static Singleton* volatile pInstance;    // // volatile added
	int x;
	Singleton() : x(5) {}
};

// from the implementation file
Singleton* volatile Singleton::pInstance = nullptr;

Singleton* Singleton::instance() {
	if (pInstance == nullptr) {
		Lock lock;
		if (pInstance == nullptr) {
			Singleton* volatile temp = new Singleton;
			pInstance = temp;
		}
	}
	return pInstance;
}

// After inlining the constructor, the code looks like this:
if(pInstance = nullptr){
    Lock lock;
    if(pInstance = nullptr){
        Singleton* volatile tmep =
            static_cast<Singleton*>(operator new(sizeof(Singleton)));
        temp->x = 5;
        pInstance = temp;
    }
}
```

尽管 temp 是 `volatile`的，但 `*temp` 不是，这意味着 `temp->x` 也不是。因为我们现在明白，对非 `volatile` 的赋值有时可能会被重新排序，所以很容易看出，编译器可以重新排序 `temp->x` 的赋值，以尽早对 `pInstance` 赋值。如果他们这样做了，`pInstance` 就会在它指向的数据初始化之前被赋值，这又导致了另一个线程读取未初始化的 `x` 的可能性。

针对这种病灶的一种有吸引力的治疗方法是对 `*pInstance` 以及 `pInstance` 本身进行验证，从而产生一个美化的 `Singleton` 版本，其中所有“东西”都被赋予了 `volatile`：

```cpp
class Singleton {
public:
	static volatile Singleton* volatile instance();
    ...
private:
	static volatile Singleton* volatile pInstance;
};

// from the implementation file
volatile Singleton* volatile Singleton::pInstance = nullptr;

volatile Singleton* volatile Singleton::instance() {
    if(pInstance == nullptr){
        Lock lock;
        if(pInstance == nullptr){
            volatile Singleton* volatile tmep =
                new volatile Singleton;
            pInstance = temp;
        }
    }
    return pInstance;
}
```

> 此时，人们可能会合理地想知道为什么 `Lock` 未声明为 `volatile`。毕竟，在尝试写入 `pInstance` 或 `temp` 之前初始化锁至关重要。好吧，`Lock` 来自一个线程库，因此我们可以假设它在其规范中规定了足够的限制，或者在其实现中加了了足够的魔法，从而可以在不需要 `volatile` 的情况下工作。我们知道的所有线程库其实都是这种情况。本质上，使用线程库中的实体（例如，对象、函数等）会导致在程序中施加“硬序列点” 适用于所有线程的序列点。出于本文的目的，我们假设此类“硬序列点”在代码优化期间充当指令重新排序的坚固障碍：与源代码中使用库实体之前的源语句相对应的指令不能移动到与使用该实体相对应的指令之后，并且与源代码中使用这些实体之后的源语句相对应的指令不能移动到与它们的使用相对应的指令之前。真正的线程库施加的限制不那么严格，但对于我们在此讨论的目的来说，细节并不重要。

人们可能希望上述完全 `volatile` 限定的代码能够由标准保证在多线程环境中正常工作，但它可能由于两个原因而失败。首先，标准对于可观察行为的限制仅适用于标准定义的抽象机器，而这个抽象机器是没有多线程执行的概念的。因此，尽管标准阻止编译器对线程内对 `volatile` 数据的读写进行重新排序，但它对跨线程的此类重新排序没有任何限制。至少大多数编译器是这样解释的。因此，在实践中，许多编译器可能会从上述源代码生成线程不安全的代码。如果您的多线程代码在使用 `volatile` 时正常工作，而在不使用 `volatile` 时无法正常工作，那么您的 C++ 实现要么仔细实现了 `volatile` 以与线程一起工作（可能性较小），要么您只是运气好（可能性较大）。无论哪种情况，您的代码都不可移植，这简直是灾难。第二，正如 `const` 限定的对象在其构造函数运行完成之前不会变为 `const` 一样，`volatile` 限定的对象只在退出期构造函数时才会变为 `volatile`。在语句 `volatile Singleton* volatile temp = new volatile Singleton;` 中，被创建的对象直到表达式 `new voaltile Singleton;` 已经运行完成，这意味着我们又回到了内存分配和对象初始化指令可能被任意重新排序的情况。

我们可以解决这个问题，尽管有点尴尬。在 `Singleton` 构造函数中，我们使用强制类型转换在初始化 `Singleton` 对象时临时为其每个数据成员添加“易失性”，从而防止执行初始化的指令发生相对移动。例如，下面是以这种方式编写的 `Singleton` 构造函数。（为了简化演示，我们使用赋值来赋予 `Singleton::x` 其第一个值，而不是成员初始化列表，就像我们在上面的代码中所做的那样。此更改对我们在此处解决的任何问题均无影响。）

```cpp
Singleton() {
	static_cast<volatile int&>(x) = 5;    // note cast to volatile
}

// 在 Singleton 版本中内联此函数后，其中 pInstance 已正确 volatile 限定，我们得到
class Singleton {
public:
	static Singleton* instance();
private:
	static Singleton* volatile pInstance;
	int x;
};

Singleton* Singleton::instance() {
	if (pInstance == 0) {
		Lock lock;
		if (pInstance == 0) {
			Singleton* volatile temp =
				static_cast<Singleton*>(operator new(sizeof(Singleton)));
			static_cast<volatile int&>(temp->x) = 5;
			pInstance = temp;
		}
	}
}
```

现在对 `x` 的赋值必须在对 `pInstance` 的赋值之前，因为它们都是 `volatile` 的。不幸的是，这一切都无助于解决第一个问题：C++ 的抽象机是单线程的，而且 C++ 编译器可能会选择从源代码生成线程不安全的代码，如上文所述。否则，失去优化机会会导致效率损失过大。经过所有这些讨论，我们又回到了原点。

# 6 多处理器计算机上的 DCLP

假设您在一台具有多个处理器的机器上，每个处理器都有自己的内存缓存，但所有处理器都共享一个公共内存空间。这种架构需要准确定义一个处理器执行的写入如何并且何时传播到共享内存，从而对其他处理器可见。很容易想象这样的情况：一个处理器更新了其自身缓存中的共享变量的值，但更新后的值尚未刷新到主内存中，更不用说加载到其他处理器的缓存中。共享变量值的这种缓存间不一致称为**缓存一致性问题**。

假设处理器 `A` 修改了共享变量 `x` 的内存，然后又修改了共享变量 `y` 的内存。这些新值必须刷新到主内存，以便其他处理器能够看到它们。但是，**按地址递增顺序刷新新的缓存值可能更有效**，因此如果 `y` 的地址先于 `x` 的地址，则 `y` 的新值可能会先于 `x` 的新值写入主内存。如果发生这种情况，其他处理器可能会先看到 `y` 的值发生变化，然后再看到 `x` 的值发生变化。这种可能性对于 DCLP 来说是一个严重的问题。正确的 `Singleton` 初始化需要初始化 `Singleton` 并且将 `pInstance` 更新为非空，并且这些操作按此顺序发生。如果处理器 A 上的线程执行步骤 1，然后执行步骤 2，但处理器 B 上的线程认为步骤 2 在步骤 1 之前执行，则处理器 B 上的线程可能再次引用未初始化的 `Singleton`

缓存一致性问题的一般解决方案是**使用内存屏障**（即隔离）：编译器、链接器和其他优化实体识别的指令，用于限制在多处理器系统中对共享内存的读写操作可能执行的重新排序类型。在 DCLP 的情况下，我们需要使用内存屏障来确保在对 `Singleton` 的写入完成之前，`pInstance` 不会被视为非空。以下是伪代码。我们仅显示插入内存屏障的语句的占位符，因为实际代码是特定于平台的（通常在汇编程序中）。

```cpp
Singleton* Singleton::instance(){
    Singleton* tmp = pInstance;
    ...                                // insert memory barrier
    if(tmp == 0){
        Lock lock;
        tmp = pInstance;
        if(tmp == 0){
            tmp = new Singleton;
            ...                        // insert memory barrier
            pInstance = tmp;
        }
    }
    return tmp;
}
```

现代C++早已经可以使用内存屏障来写一个简单的单例模式：https://godbolt.org/z/7bnqe839z

> Arch Robison 提出：从技术上讲，您不需要完全双向屏障。第一个屏障必须防止 `Singleton` 构造向下迁移（由另一个线程）；第二个屏障必须防止 `pInstance` 初始化向上迁移。这些被称为“获取”和“释放”操作，在硬件（如 Itanum）上，它们可能比完全屏障产生更好的性能。

尽管如此，这是一种实现 DCLP 的方法，只要您在支持内存屏障的机器上运行，它应该是可靠的。所有可以重新排序共享内存写入的机器都以某种形式支持内存屏障。有趣的是，这种方法在单处理器设置中同样有效。这是因为内存屏障还可以充当硬序列点，防止可能非常麻烦的指令重新排序。

# 7 结论和 DCLP 替代方案

首先，请记住，单处理器上基于时间片的并行性与跨多处理器的真正并行性不同。这就是为什么单处理器架构上特定编译器的线程安全解决方案在多处理器架构上可能不是线程安全的，即使您坚持使用相同的编译器也是如此。（这是一个普遍的观察。它并不特定于 DCLP。）其次，尽管 DCLP 本质上与 `Singleton` 没有联系，但使用 `Singleton` 往往会导致希望通过 DCLP “优化”线程安全访问。因此，您应确保避免使用 DCLP 实现 `Singleton`。如果您（或您的客户端）担心每次调用实例时锁定同步对象的成本，您可以建议客户端通过缓存实例返回的指针来尽量减少此类调用。例如，建议不要编写这样的代码，

```cpp
Singleton::instance()->transmogrify();
Singleton::instance()->metamorphose();
Singleton::instance()->transmute();

clients do things this way:

Singleton* const instance =
    Singleton::instance();             // cache instance pointer

instance->transmogrify();
instance->metamorphose();
instance->transmute();
```

这个程序旨在鼓励客户端在每个需要访问单例对象的线程开始时，对实例进行一次调用，将返回的指针缓存在线程本地存储中。因此，采用此技术的代码只需为每个线程付出一次锁定访问的开销。在建议缓存调用实例的结果之前，通常最好先验证这是否真的会带来显著的性能提升。使用线程库中的锁来确保线程安全的 Singleton 初始化，然后进行时序研究以查看成本是否真的值得担心。第三，除非真的需要，否则请避免使用延迟初始化的 `Singleton`（懒汉模式）。经典的 `Singleton` 实现基于在请求资源之前不初始化资源。另一种方法是即时初始化（饿汉模式），即在程序运行开始时初始化资源。由于多线程程序通常以单线程开始运行，因此这种方法可以将一些对象初始化推送到代码的单线程启动部分，从而无需担心初始化期间的线程问题。**在许多情况下，在单线程程序启动期间（例如，在执行 main 之前）初始化单例资源是提供快速、线程安全的单例访问的最简单方法**。

采用即时初始化的另一种方法是用 [`Monostate` 模式](#单态模式)代替 `Singleton` 模式。然而，`Monostate` 有不同的问题，特别是在控制组成其状态的非局部静态对象的初始化顺序时。Effective C++ 描述了这些问题，具有讽刺意味的是，它建议使用 `Singleton` 的变体来避免这些问题。（该变体不能保证是线程安全的。）另一种可能性是将全局单例替换为每个线程一个单例，然后使用线程本地存储单例数据。这允许延迟初始化而不必担心线程问题，但这也意味着多线程程序中可能会有多个“单例”。

最后，DCLP 及其在 C++ 和 C 中的问题体现了**在没有线程概念（或任何其他形式的并发性）的语言中编写线程安全代码的固有困难**。多线程考虑因素无处不在，因为它们影响代码生成的核心。正如 Peter Buhr 指出的，希望将多线程排除在语言之外并隐藏在库中是一种幻想。这样做，要么库最终会对编译器生成代码的方式施加限制（正如 Pthreads 已经做的那样），要么编译器和其他代码生成工具将被禁止执行有用的优化，即使是在单线程代码上。您只能在多线程、线程不感知语言和优化的代码生成组成的三驾马车中挑选两个。例如，Java 和 .NET CLI 分别通过向语言和语言基础结构引入线程感知来解决这种紧张关系。

## 后内联流分析

后内联流分析（Post-Inlining Flow Analysis）是一种在程序优化中使用的技术，主要用于提高编译器对代码行为的理解，尤其是在函数内联之后。这种分析通常在编译器的中间表示（IR）阶段进行，用于评估程序的控制流和数据流。

## 可观察行为

程序的可观察行为指的是程序在特定条件下对外部世界（如用户、文件、网络等）的影响和反应。这些行为是可以被观察到的结果，比如程序的输出、对数据的修改、产生的异常等。可观察行为通常包括以下几个方面：

- 输出结果：程序向控制台、文件或网络输出的数据。
- 状态变化：程序在运行过程中改变了某些数据结构或对象的状态。
- 异常和错误：程序在执行过程中是否抛出异常或返回错误码。

## 副作用

副作用（side effect）是指一个函数或表达式在执行过程中对其外部环境产生的影响，除了返回值之外的任何改变。副作用可能包括：

- 修改变量：改变外部变量的值。
- 输入/输出操作：进行文件读写、打印到控制台、网络通信等。
- 状态变化：改变对象的状态，如在一个对象上调用方法后，改变该对象的内部状态。
- 异常抛出：抛出异常可能影响程序的控制流。

## volatile简史

要找到易失性的根源，让我们回到 20 世纪 70 年代，当时 Gordon Bell（因 PDP-11 而出名）引入了内存映射 I/O (MMIO) 的概念。在此之前，处理器分配引脚并定义特殊指令来执行端口 I/O。MMIO 背后的想法是使用相同的引脚和指令进行内存和端口访问。处理器外部的硬件会拦截特定的内存地址并将其转换为 I/O 请求；因此处理端口就变成了简单地读取和写入特定于机器的内存地址，真是个好主意。减少引脚数量是好事引脚会减慢信号速度、增加缺陷率并使封装复杂化。此外，MMIO 不需要针对端口的特殊指令。程序只需使用内存，硬件会处理其余的事情

要了解为什么 MMIO 需要 `volatile` 变量，让我们考虑以下代码:

```cpp
unsigned int* p = GetMagicAddress();
unsigned int a, b;
a = *p;
b = *p;
```

如果 `p` 指向一个端口，则 `a` 和 `b` 应接收从该端口读取的两个连续字。但是，如果 `p` 指向一个真正的内存位置，则 `a` 和 `b` 会加载同一位置两次，因此比较结果相等。编译器在复制传播优化中利用了这一假设，将上面的最后一行转换为更高效的 `b = a;`，类似地，对于相同的 p、a 和 b，考虑 `*p = a, *p =  b;` 代码将两个字写入 `*p`，但优化器可能会假设 `*p` 是内存，并通过消除第一个赋值来执行死赋值消除优化。显然，这种“优化”会破坏代码。当主线代码和中断服务例程 (ISR) 都修改变量时，也会出现类似的情况。对于编译器来说，冗余的读取或写入可能实际上是必要的，以便主线代码与 ISR 进行通信。

因此，在处理某些内存位置（例如，内存映射端口或 ISR 引用的内存）时，必须暂停某些优化。`volatile` 的存在是为了指定对此类位置的特殊处理，具体来说：（1）`volatile` 变量的内容是“不稳定的”（可以通过编译器未知的方式更改），（2）对 `volatile` 数据的所有写入都是“可观察的”，因此必须严格执行，（3）对 `volatile` 数据的所有操作都执行按照它们在源代码中出现的顺序。前两个规则确保正确的读取和写入。最后一个规则允许实现混合输入和输出的 I/O 协议。这非正式地是 C 和 C++ 的 `volatile` 所保证的。Java 进一步完善了 `volatile`，保证了上述跨多线程的特性。这是一个非常重要的步骤，但还不足以使 `volatile` 可用于线程同步：`volatile` 和非 `volatile` 操作的相对顺序仍未指定。这种省略迫使许多变量必须为 `volatile`，以确保正确的顺序。Java 1.5 的 `volatile` 具有更严格但更简单的获取/释放语义：任何对 `volatile` 的读取都保证发生在后续语句中的任何内存引用（无论是否为 `volatile`）之前，任何对 `volatile` 的写入都保证发生在其前面的语句中的所有内存引用之后。.NET 还定义了 `volatile` 以纳入多线程语义，这与当前提议的 Java 语义非常相似。据我们所知，C 或 C++ 的 `volatile` 上没有进行类似的工作（C++现在也不适合在多线程中使用 `volatile`）。

## 单态模式

`Monostate` 模式的核心思想是使用一个静态变量（或静态成员）来存储状态，而不是依赖于单一的实例。通过这种方式，任何类的实例都共享同一状态。实现上就是在类中使用静态成员变量来存储共享状态。每个实例都可以通过这些静态成员来访问和修改状态。






