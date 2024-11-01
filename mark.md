# 描述

在网络上搜索各种设计模式，你肯定会发现最常提到的一种是 `Singleton`。然而，尝试将 `Singleton` 付诸实践，你几乎肯定会遇到一个重大限制：按照传统实现方式（我们将在下面解释），`Singleton` 不是线程安全的。

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
    if(pInstance==0){
        pInstance=new Singleton;
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

(这是伪代码啊)这种解决方案的缺点是它可能很开销很大。每次访问 `Singleton` 都需要获取锁，但实际上，我们仅在初始化 `pInstance` 时才需要锁。这应该只在第一次调用 `instance` 时发生。如果在程序运行过程中调用了 `n` 次 `instance` ，我们只需要在第一次调用时获取锁。既然您知道其中 `n - 1` 次是不必要的，为什么还要为 `n` 次锁付出额外的开销呢？DCLP 旨在防止您这样做。

# 3 DCLP

DCLP 的关键在于我们发现大多数对实例的调用都会得出 **`pInstance` 不为空**，因此甚至不会尝试初始化它。因此，DCLP 在尝试获取锁之前会检测 `pInstance` 是否为空。**只有检测成功（即，如果 `pInstance` 尚未初始化）才会获取锁**，之后会再次执行测试以确保 `pInstance` 仍为空（因此称为双重检查锁定）。第二个测试是必要的，因为正如我们刚刚看到的，在第一次测试 `pInstance` 和获取锁之间，可能恰好有另一个线程初始化了 `pInstance`。

以下是经典的 DCLP 实现：











