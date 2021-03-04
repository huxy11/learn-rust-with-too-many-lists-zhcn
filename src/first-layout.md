# 代码布局

首先想一个问题，什么是链表？简单来说，链表就是堆里的一块块通过指针相互链接起来的数据，是过程式语言最好别用但在函数式语言里哪儿都要用的结构。如果我们找一个函数式语言大师，问他什么是链表。他大概会说：

```haskell
List a = Empty | Elem a (List a)
```

这大概可以理解为“链表要么为空，要么是一块数据，数据后面跟着另一个链表”。这个*和类型(sum type)*是一个递归定义，用人话来说就是“一个可能含有各种类型的各种值的类型”。Rust 里我们也把这样的类型成为 `枚举量(enum)`。如果你学过类C的话，没错这个枚举就是你心里想的那个枚举，只不过这个枚举要刺激得多。

用 Rust 的枚举来完成这个定义（我们先暂时忘掉泛型这个东西，像 Go 开发者一样…）

```rust ,ignore
// in first.rs
// pub 表示我们想把这个枚举量公开出去给外部调用者使用
pub enum List {
    Empty,
    Elem(i32, List),
}
```

emm…代码量有点大，写完记得擦擦汗。
然后我们来编译这段代码：

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

??? 这和函数式语言大师说好的不一样啊。行不行啊大师……

好吧，抱怨完我们先看看错误信息。嗯？好像编译器正在教我们怎么解决这个问题：

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

等等，`box` 是个啥？百度一下 `rust box`...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

介绍如下，比较简约

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation.(一个用于堆内存分配的指针。）
> See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.(点击链接查阅更多信息。）

那就*点击链接*我们看看更多的信息


> `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
> (`Box<T>`,也就是'box',在 Rust 要进行堆内存分配的话，box 用是最简单的方法，它对分配出的内存具有所有权，会自己生命周期结束之后把分配的这段堆内存释放掉。)
>
> Examples
> (用法举例）
>
> Creating a box:
> (创建box指针)
>
> `let x = Box::new(5);`
>
> Creating a recursive data structure:
> (创建递归定义的数据结构)
>
```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```rust
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
> (打印结果是`Cons(1, Box(Cons(2, Box(Nil))))`)
>
> Recursive structures must be boxed, because if the definition of Cons looked like this:
> (递归定义的结构必须封装入如 box 的指针里，如果按照如下的定义:)
>
> `Cons(T, List<T>),`
>
> It wouldn't work. This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons. By introducing a Box, which has a defined size, we know how big Cons needs to be.
> 定义就错误了。因为 List 的大小取决于它所包含的各个成员。但是我们无法确定 Cons 有多大，需要分配给它多少内存。而 box 指针则是确定大小的，所以用 box 指针作为 Cons 的成员的话，我们就能确定分配给 Cons 指针多少内存空间。

完美！我从未见过如此详细有用的文档。直接解释了我们写的代码为什么错了，还给出了解决办法。

Rust 文档NB！

照猫画虎：

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

哈，编译成功！
……不过说实话，这个定义，字里行间都透露出一丝蠢萌，最好别让同学同事看到……

傻在哪儿呢，比如说吧，我们现在的链表，有两种节点成员：


```text
[] = 在栈里
() = 在堆里

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *一堆垃圾信息*)
```

俩缺点：
* 我们在链表最尾的节点申请了一块内存用来…嗯…啥也没干，就是申请了，任性。
* 链表的头节点在栈里，不像其他的节点一样在堆里。

仔细想想，这俩货就不能结合结合吗？一个是多申请了内存不干事儿，一个是该申请内存放堆里它偏不。

于是乎，我们改进了一下：


```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

在这样的结构里，我们把所有节点都通过申请堆内存来储存。然后用一个空指针替换了*一堆垃圾信息*。所谓的一堆垃圾信息是什么？好吧，要解答这个问题，我们先了解一下枚举类型的内存布局。
举例说明，我们现有枚举类型：

```rust ,ignore
enum Value {
	Empty,
    Int(i32),
    Flt(f32),
	Str(String),
    ...
    Dic(struct {key: String, val:String}),
}
```


A Foo will need to store some integer to indicate which *variant* of the enum it
represents (`D1`, `D2`, .. `Dn`). This is the *tag* of the enum. It will also
need enough space to store the *largest* of `T1`, `T2`, .. `Tn` (plus some extra
space to satisfy alignment requirements).

The big takeaway here is that even though `Empty` is a single bit of
information, it necessarily consumes enough space for a pointer and an element,
because it has to be ready to become an `Elem` at any time. Therefore the first
layout heap allocates an extra element that's just full of junk, consuming a
bit more space than the second layout.

One of our nodes not being allocated at all is also, perhaps surprisingly,
*worse* than always allocating it. This is because it gives us a *non-uniform*
node layout. This doesn't have much of an appreciable effect on pushing and
popping nodes, but it does have an effect on splitting and merging lists.

Consider splitting a list in both layouts:

```text
layout 1:

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

split off C:

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
layout 2:

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

split off C:

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

Layout 2's split involves just copying B's pointer to the stack and nulling
the old value out. Layout 1 ultimately does the same thing, but also has to
copy C from the heap to the stack. Merging is the same process in reverse.

One of the few nice things about a linked list is that you can construct the
element in the node itself, and then freely shuffle it around lists without
ever moving it. You just fiddle with pointers and stuff gets "moved". Layout 1
trashes this property.

Alright, I'm reasonably convinced Layout 1 is bad. How do we rewrite our List?
Well, we could do something like:

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

Hopefully this seems like an even worse idea to you. Most notably, this really
complicates our logic, because there is now a completely invalid state:
`ElemThenNotEmpty(0, Box(Empty))`. It also *still* suffers from non-uniformly
allocating our elements.

However it does have *one* interesting property: it totally avoids allocating
the Empty case, reducing the total number of heap allocations by 1. Unfortunately,
in doing so it manages to waste *even more space*! This is because the previous
layout took advantage of the *null pointer optimization*.

We previously saw that every enum has to store a *tag* to specify which variant
of the enum its bits represent. However, if we have a special kind of enum:

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

the null pointer optimization kicks in, which *eliminates the space needed for
the tag*. If the variant is A, the whole enum is set to all `0`'s. Otherwise,
the variant is B. This works because B can never be all `0`'s, since it contains
a non-zero pointer. Slick!

Can you think of other enums and types that could do this kind of optimization?
There's actually a lot! This is why Rust leaves enum layout totally unspecified.
There are a few more complicated enum layout optimizations that Rust will do for
us, but the null pointer one is definitely the most important!
It means `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`, and
several other important types in Rust have no overhead when put in an `Option`!
(We'll get to most of these in due time.)

So how do we avoid the extra junk, uniformly allocate, *and* get that sweet
null-pointer optimization? We need to better separate out the idea of having an
element from allocating another list. To do this, we have to think a little more
C-like: structs!

While enums let us declare a type that can contain *one* of several values,
structs let us declare a type that contains *many* values at once. Let's break
our List into two types: A List, and a Node.

As before, a List is either Empty or has an element followed by another List.
By representing the "has an element followed by another List" case by an
entirely separate type, we can hoist the Box to be in a more optimal position:

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

Let's check our priorities:

* Tail of a list never allocates extra junk: check!
* `enum` is in delicious null-pointer-optimized form: check!
* All elements are uniformly allocated: check!

Alright! We actually just constructed exactly the layout that we used to
demonstrate that our first layout (as suggested by the official Rust
documentation) was problematic.

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

Rust is mad at us again. We marked the `List` as public (because we want people
to be able to use it), but not the `Node`. The problem is that the internals of
an `enum` are totally public, and we're not allowed to publicly talk about
private types. We could make all of `Node` totally public, but generally in Rust
we favour keeping implementation details private. Let's make `List` a struct, so
that we can hide the implementation details:


```rust ,ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

Because `List` is a struct with a single field, its size is the same as that
field. Yay zero-cost abstractions!

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

Alright, that compiled! Rust is pretty mad, because as far as it can tell,
everything we've written is totally useless: we never use `head`, and no one who
uses our library can either since it's private. Transitively, that means Link
and Node are useless too. So let's solve that! Let's implement some code for our
List!
