# inside-rust-std-library

本书主要对 RUST 的标准库代码进行分析.

本书尽可能给读者找出一条标准库代码的阅读脉络. 同时, 分析不仅仅针对代码的功能, 也针对代码背后的需求及若干代码设计的思路.

C 语言精通的标志是对指针的精通. RUST 的裸指针也是 RUST 的最基础及最核心的难点之一. 所以, 将裸指针及相关的内存模块作为代码分析的起始点, 熟悉了裸指针及内存, 自然也就对所有权, 借用, 生命周期的本质有了深刻的理解, RUST 语言的最难关便过了.

泛型是 RUST 不可分割的语法之一, 而对于其他语言, 没有泛型不影响语言的使用. 泛型及基于 trait 的泛型约束是 RUST 的另一个代码基础.

针对基本类型的分析, 可以看到 RUST 利用 trait 语法使之具备了无限的扩展性, 这是 RUST 更有表现力的语法能力的展现.

`Option<T>`/`Result<T,E >` 等类型实际完全是由标准库定义的, 并不是 RUST 语言最底层的基本内容, 可以从代码分析中发现这一点.

所有的运算符都可以重载, 且可以跨越类型重载, RUST 的运算符重载揭示了 RUST 很多的编码奥秘及技巧.

`Iterator` 加闭包是函数式编程的基础构架,`Iterator` 的适配器构成了函数式编程的基础设施, RUST 完整的实现了这些内容, 并且几乎为每个类型都实现了迭代器, 并尽可能的为函数式编程做好了准备.

`Cell<T>`/`RefCell<T>`/`Pin<T>`/`Lazy<T >` 代码证明了在 RUST 的基础语法下, 如何创造性的解决问题.

`Box<T>`/`RawVec<T>` 是两个堆内存申请的基本结构, 善用这两个结构, 除非写内存管理, 基本上就不必再接触底层的堆内存申请及释放.

每一个智能指针实际上也是 RUST 对经典的数据结构实现的精妙例程.

RUST 对不同操作系统的适配让程序员不必象 C 那样再重复的耗费精力并且还沾沾自喜于此份工作.

仅支持异步编程的 `async`/`await`,`Future` 也体现了 RUST 的作最基础的工作的态度.

