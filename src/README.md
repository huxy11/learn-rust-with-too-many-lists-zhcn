# Learn Rust With Entirely Too Many Linked Lists

> 遇到任何问题或者想直接看到最终代码?
> [Everything's on Github!][github]

> **NOTE**: 本书基于随 rustc 1.31 首发的 Rust 2018 而写。如果你使用的 Rust 版本够新，`cargo new` 所创建的 Cargo.toml文件应该会包含`edition = "2018"`(在遥远的未来，也可能会变成更大的数)。使用较老版本的 Rust 完全没有问题，除了可能引发的、本书完全不会提及的一系列额外的编译错误。

我经常被请教如何用 Rust 实现一个链表。具体的实现因具体的需求不同而不同，并不好一概而论。因此我写下这本书，好好地解决一下这个问题。
本书中，我们会通过实现六种链表学到 Rust 编程基础与一些进阶内容。包括:

* 这些类型的“指针”: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`
* 所有权、所有权借用、可变性继承、内部可变性、Copy
* Rust 语言关键字: struct, enum, fn, pub, impl, use, ...
* 模式匹配、泛型、析构器
* 测试
* 基础的 Unsafe Rust

没错，你得学会这些东西才能够去写六种链表。

侧边栏能看到所有内容。不过作为快捷的参考，这里列出了我们要做的事:

1. [不太行的单链栈](first.md)
2. [还行的单链栈](second.md)
3. [固定的单链栈](third.md)
4. [不太行但安全的双向链表](fourth.md)
5. [不安全的单向链表](fifth.md)
6. [TODO: 还行的但不安全的双向链表](sixth.md)
7. [额外福利: 一堆相当不行的表](infinity.md)

需要强调的是，我们会用`命令行`来进行各个步骤，也会用 Rust 标准的包管理器 `Cargo` 来构建我们的项目。虽然对 Rust 开发来说，`Cargo` 并不是必要的，但是用它比直接用 rustc 要好太多了。不过如果你只想随便玩玩，也可在 [play.rust-lang.org][play] 直接运行简单的代码。

开始构建我们的项目吧:

```text
> cargo new --lib lists
> cd lists
```

我们会吧不同的表放分别放在各自的文件夹里，以免丢失掉已完成的工作。

值得注意的是，*真正的* Rust 学习包括了亲手写下代码，承受编译器抛出的各种错误，然后试着去理解这些错误意味着什么。本书会贴心地保证你能够经常接触他们。要成为高水平的 Rust 使用者，学会去阅读并理解它（大部分情况下)非常优秀的编译器所抛出的错误与相关文档是*尤为*重要的。

事实上，在写本书的时候，我遇到的编译错误要比展示给你的多得多得多。比如，我并不打算提及诸如"我打(control+v)错了"此类的错误。本书主要目的是引导你去承受编译器的咆哮。

我们的进展不会很快，而且老实说我也不会严肃认真。我认为编程嘛，开心是最重要的。如果你喜欢严肃一些、信息密度大一些的内容的话，这本书就不适合你了。应该说，我写的所有东西都不适合你，你找错人了。


# An Obligatory Public Service Announcement

事先声明: 我恨链表，非常恨。链表是狠糟糕的数据结构。诚然链表的确有重要的应用场合：

* 当你需要对大容量的表进行*大量*的分割与合并操作，注意是*大量的*。
* 当你要做一些高端的无锁并发操作。 
* 当你想用侵入式链表结构来实现内核/嵌入式的一些功能的时候。
* 当你在用一个纯函数式语言，由于语法与不变性的限制让链表更好用时。
* …以及剩下的一些其他情况。

但是对于 Rust 使用者来说，上面的情况都*及其地*稀有。99%的情况下我们应该用Vec（数组栈），剩下的1%里又有99%的情况下我们应该用VecDeque（数组双端队列）。大部分情况下，因为其1.更少的内存申请与内存浪费、2.真·随机读取、3.局部性，是更好的数据结构。

链表的使用几乎和和字典树一样地*小众*和*模糊*。有人会说字典树确实小众，但链表则并不小众，反而盛名在外，每一个本科生去都被教导过如何去实现一个链表。链表是唯一一个我没法儿从[std::collections][rust-std-list]里剔除掉的小众数据结构。

我们应该团结起来，勇敢地向把链表当作“标准”数据的行为说*不*。它是个好数据结构，只不过好用的场景是特别的，并不通常。

有部分人像是只会读本声明的第一段一样。用我在第二段就写出来的例子来反驳我。

为了能够更详细直接地阐述我的观点，我列出了一些反驳我的话以及我的回应。如果你只是单纯地想学 Rust 请直接跳到[第一章](fisrt.md)

## 性能并不总是关键

说得没错！可能你的程序是I/O密集型的，或者代码运行次数太低所以性能根本不重要。但是这根本不足以让人去用链表。事实上，在这种情况下用什么都可以。那为什么用链表呢？用散列链表不香吗？ 

反正性能都不重要，简单粗暴用数组不就得了。




## 链表的分割、附加、插入和删除操作的时间复杂度都是O(1)

对！如 [BjarneStrostrpnotes][jarne] 所说。不过如果指针寻址的时间比复制整个数组的值所用时间（这个复制真的很快）还多的话，就节省不了时间了。

除非你现有的任务主要由分割和合并构成，因为此外的其他操作会因为缓存不友好和代码复杂度的问题拖慢程序的运行，减弱应用链表带来的理论优势。

*不过的确，如果性能分析的结果是大量时间被用在分割和合并上的话。应用链表可以获得更好的性能表现*






## 我没法忍受复杂度均摊（armortization)

你已经进入到一个十分狭窄的空间——大多数人都能忍受复杂度均摊。数组只在最坏情况下进行复杂度均摊。使用数组也并不意味着你一定要进行均摊。如果你可以预测有多少元素要存储（或者有一个上界），你可以预先分配好所有需要的空间。在我的经验里，能够预测需要的元素数量是非常常见的。对于Rust来说，所有的迭代器都为这种情况提供了一个 `size_hint`。

在这种情况下,`push`和`pop`就会真正成为 `O(1)` 操作。而且它们将会比在链表上的 `push`和 `pop`高出一个数量级。你进行一次指针偏移，写入字节，递增一个整数。不需要访问任何的内存分配器。

如果你要求低延迟的话，这样不是更好么？

*但是没错，如果你无法预测你的工作负载，那最坏情况下的延迟降低也要考虑在内！*






## 链表浪费的空间更少

呃，这东西比较复杂。一个“标准”的数组大小重分配策略会将数组增长或缩小，来保证最多只有一半空间是空的。这确实会很多浪费的空间。尤其是在Rust中，我们不会自动缩减集合的内存占用（如果你要把它填充回去，这只会造成浪费），因此浪费程度可以达到正无穷！

但这是最坏情况的状态。在最优情况下，一个数组栈管理整个数组只需要三个指针的额外开销——基本可以忽略。

而链表则对每个元素都无条件的浪费内存空间。一个单向链表的元素浪费了一个指针，而双向链表浪费两个。和数组不一样，链表的相对浪费量和元素数量呈正比。如果一个元素所占空间非常巨大，浪费会趋近于0。如果每个元素所占空间很小（例如，比特），这可能造成最多16倍的内存浪费（如果是32位，8倍）！

实际的数字应该更接近23倍（或者32位时的11倍），因为那一个字节需要进行位对齐，让整个节点的大小对齐到一个指针。

这也是在对内存分配器进行最优条件假设的前提下得出的结论：节点的内存分配和释放会紧密的进行，并且你不会因为内存碎片化而丢失空间。

*但是没错，如果每个元素所占空间巨大，你无法预测负载，并且拥有一个高效的内存分配器，那这确实可以节省内存！*






## 我在某函数式语言中一直使用链表

棒极了！在函数式语言中使用链表是非常优雅的，因为你可以在不涉及任何可变性的情况下操作它们，可以递归的描述它们，甚至可以借助惰性求值的魔法来操作无穷大列表。

特别的，链表因为无需任何可变状态就可以表示迭代而显得特别优雅。迭代的下一步就是访问下一个子列表而已。

不过应该注意的是，Rust可以对数组进行模式匹配，并且使用切片来处理子数组！从某些角度来说，它甚至比一个函数式列表更富表达力，因为你可以专门处理最后一个元素或者甚至“没有第一个和最后两个元素的数组”或者任何你想要的疯狂的东西。

你不能通过切片来构造一个列表倒是真的。你只能把它们撕成小片。

对于惰性求值，我们有迭代器作为替代。它可以是无穷的，而你可以像对待一个函数式列表一样对它们 map, filter, reverse 和 concatenate，这些操作都会被惰性的执行。不必说，数组切片也可以被转换（coerce）成一个迭代器。

*不过没错，如果你只限制于使用不可变的语义，链表是很好用的。*

注意我并没有说函数式编程一定是弱的或糟糕的。然而它确实是从根本上语义受限的：你很大程度上只被允许讨论事情是怎么样，而非它们应该如何被完成。这实际上是一个特性，因为它让编译器得以进行成吨的诡异变换来找出潜在的最佳工作方式而不需要你去担心它。然而，这也带来了完全无法去担心它的代价。通常的情况下可以找到应急出口，但到达某个限度后你又会开始写过程式的代码了。

即便在函数式语言中，你也应该在确实需要用到数据结构时选择恰当的数据结构。没错，链表是操作控制流的主要工具，但是如果要在里面存储一堆数据并且查询的话，它们是非常糟糕的。



## 在构建并行数据结构时，链表是很好用的！

没错！尽管如此，实现一个并行数据结构真的是另一个很大的话题，不应该被轻视。这显然都不是很多人会考虑去做的事。当它实际被实现以后，你也真的不会真的选择使用链表。你会选择使用一个MPSC队列或者其他什么东西。在这个情况下实现策略实际上已经在考虑范围之外了！

*不过说得没错，链表是无锁并行开发的深海中手持三叉戟的守护者。*






## Mumble mumble kernel embedded something something intrusive.

It's niche. You're talking about a situation where you're not even using
your language's *runtime*. Is that not a red flag that you're doing something
strange?

It's also wildly unsafe.

*But sure. Build your awesome zero-allocation lists on the stack.*





## Iterators don't get invalidated by unrelated insertions/removals

That's a delicate dance you're playing. Especially if you don't have
a garbage collector. I might argue that your control flow and ownership
patterns are probably a bit too tangled, depending on the details.

*But yes, you can do some really cool crazy stuff with cursors.*





## They're simple and great for teaching!

Well, yeah. You're reading a book dedicated to that premise.
Well, singly-linked lists are pretty simple. Doubly-linked lists
can get kinda gnarly, as we'll see.




# Take a Breath

Ok. That's out of the way. Let's write a bajillion linked lists.

[On to the first chapter!](first.md)


[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/Gankro/too-many-lists
[bjarne]: https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/
