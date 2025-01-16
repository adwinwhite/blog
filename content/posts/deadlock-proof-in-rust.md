+++
date = '2025-01-15T10:25:44+08:00'
draft = false 
title = '用Rust创造一个没有死锁的世界'
+++
tl;dr: 使用 type level programming 在编译期排除死锁。
<br/>

原文请看RustConf 2024演讲[Safety in an Unsafe World](https://joshlf.com/files/talks/Safety%20in%20an%20Unsafe%20World.pdf)，原文描述了一个实现更多编译期安全特性的框架，本文只是其中例子之一的学习笔记与补充。

<br/>

我们都知道死锁是因为多个线程尝试获取被彼此持有的Mutex,造成循环等待。
如何防止死锁出现呢？只要严格遵循古训 “用到多个锁的地方按照相同的顺序来获取和释放” 就可以。
问题是还是人，程序控制流变得复杂之后我们很难遵守上述准则，谁知道在调用本函数前有什么锁是锁住的。


那么我们能让可能死锁的程序从一开始就通不过编译嘛？
参考线程安全的实现，我们可以用marker trait来描述类型的一些性质。
了解logic与trait之间的对应关系对理解下文有帮助，如果没了解过请看 [type level programming](https://willcrichton.net/notes/type-level-programming/)。
这里我简单介绍一下，核心思想是trait定义关系，trait的参数包括`Self`、泛型参数和关联类型都是这个关系的参数，trait bounds定义推导规则。
用亲属关系举个例子，
我们使用伪prolog代码来描述logic关系，`head:- body`表示如果`body`成立，则`head`成立。
```prolog
female(alice).
male(bob).
male(charles).
parent(alice, charles).
parent(bob, charles).

father(F, C):- parent(F, C), male(F).
mother(M, C):- parent(M, C), female(M).
```
翻译成traits，
```rust
trait Female {}
struct Alice;
impl Female for Alice {}

trait Male {}
struct Bob;
struct Charles;
impl Male for Bob {}
impl Male for Charles {}

trait Parent<C> {}
impl Parent<Charles> for Alice {}
impl Parent<Charles> for Bob {}

trait Father<C> {}
impl<F, C> Father<C> for F
where 
    F: Male,
    F: Parent<C> {}

trait Mother<C> {}
impl<M, C> Mother<C> for M
where 
    M: Female,
    M: Parent<C> {}

fn i_am_your_father<F, C>(_: F, _: C) 
where
    F: Father<C>
{
    println!("I'm your father");
}

fn main() {
    i_am_your_father(Bob, Charles);
}
```
翻译过程还是挺直白的，可以在[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=045b35890f9e5cd1d9b99481a1e70c0c)看运行结果。


接下来我们试试用 trait 表达锁之间的获取顺序关系！
因为我们只能描述类型之间的关系而不是值间的关系，首先得让不同地方的锁拥有不同类型，给它们命个名，
用一个带Id类型参数的Wrapper. 
```rust
struct GoodLock<Id, T> {
    inner: Mutex<T>,
    id: PhantomData<Id>,
}
```
锁之间的调用顺序其实是一种DAG图或者说[strict partial order](https://en.wikipedia.org/wiki/Partially_ordered_set#Strict_partial_orders)关系，它具有传递性、非反射性、非对称性。
```prolog
% transitivity.
% C < A:- C < B, B < A.
LockAfter(C, A):- LockAfter(C, B), LockAfter(B, A).

% irreflexivity.
!LockAfter(A, A).
% asymmetry.
!LockAfter(A, B):- LockAfter(B, A).
```

根据logic与trait之间的对应关系，我们可以写出
```rust
// Self < M.
// LockAfter(Self, M).
unsafe trait LockAfter<M> {}

macro define_lock_order {
    ($A:ty => $B:ty) => {
        // b < a.
        // LockAfter(b, a).
        unsafe impl LockAfter<$A> for $B {}

        // b < X:- a < X. We know b < a from the above.
        // LockAfter(b, X):- LockAfter(a, X).
        unsafe impl<X> LockAfter<X> for $B where $A: LockAfter<X> {}
    }
}

```
可以看到我们并没有去实现非反射性和非对称性，这是因为 Rust 的trait唯一实现规则帮我们确保了它们的成立，可以想想为什么。

我们可以用这个宏来定义锁之间的顺序关系了，
```rust
// A => B => C.
struct A;
struct B;
sturct C;
define_lock_order!(A => B);
define_lock_order!(B => C);
```

接下来我们要如何利用这个关系呢？
答案是利用[witness pattern](https://willcrichton.net/rust-api-type-patterns/witnesses.html)，即在获取一个锁的时候要求提供这次获取合乎顺序的证据。
我们用一个token来记录已被获取的锁的身份，它也是下个锁需要的证据。
```rust
struct LockToken<Id> {
    id: PhantomData<Id>,
}

impl GoodLock<Id, T> {
    fn lock(&'s self, _token: &'t mut LockToken<PrevId>) -> (MutexGuard<'t, T>, LockToken<Id>) 
    where
        Id: LockAfter<PrevId>,
        's: 't
    {
        (self.inner.lock().unwrap(), LockToken { id: PhantomData })
    }

}
```
可以看到我们想获取一个锁的时候需要一个token，且它记录的Id必须排在我们这个锁之前，这确保了我们是按规定的顺序获取锁的。
为什么token参数是通过可变引用传入呢？为了允许我们这个锁释放后再次使用这个token。
这里的生命周期关系使上一个锁的token不能再次使用，除非这个锁已经被释放。

最后我们要为每个线程准备一个唯一的起始token,
```rust
pub struct Unlocked;

thread_local! {
    static HAS_LOCK_ROOT: Cell<bool> = const { Cell::new(false) };
}

impl LockToken<Unlocked> {
    pub fn new() -> Self {
        if HAS_LOCK_ROOT.get() {
            panic!("A thread can only have a single `UNLOCKED` token.");
        } else {
            HAS_LOCK_ROOT.set(true);
            LockToken { id: PhantomData }
        }
    }
}
```
大功告成！我们来用用看！
```rust
//   A
//  / \
// B   C
//  \ /
//   D
// For diamond graph, we just assign an arbitrary order between branches.
// E.g. A -> B -> C -> D or A -> C -> B -> D.

struct A;
struct B;
struct C;
struct D;

impl_lock_order!(Unlocked => A);
impl_lock_order!(A => B);
impl_lock_order!(B => C);
impl_lock_order!(C => D);

fn main() {
    let lock_a: Arc<GoodLock<A, _>> = GoodLock::new(0_i32).into();
    let lock_b: Arc<GoodLock<B, _>> = GoodLock::new(0_i32).into();
    let lock_c: Arc<GoodLock<C, _>> = GoodLock::new(0_i32).into();
    let lock_d: Arc<GoodLock<D, _>> = GoodLock::new(0_i32).into();
    {
        let lock_a = lock_a.clone();
        let lock_c = lock_c.clone();
        let lock_d = lock_d.clone();
        std::thread::spawn(move || {
            let mut unlocked_token = LockToken::new();
            let (a, mut a_token) = lock_a.lock(&mut unlocked_token);
            let (c, mut c_token) = lock_c.lock(&mut a_token);
            let (d, _) = lock_d.lock(&mut c_token);
            println!("{}, {}, {}", a, c, d);
        });
    }
    let mut unlocked_token = LockToken::new();
    let (a, mut a_token) = lock_a.lock(&mut unlocked_token);
    let (b, mut b_token) = lock_b.lock(&mut a_token);
    let (d, _) = lock_d.lock(&mut b_token);
    println!("{}, {}, {}", a, b, d);
}
```
更多测试和完整实现可以看看我的[repo](https://github.com/adwinwhite/goodlord)。

生产中使用可以看看fuchsia的[lock_order](https://fuchsia-docs.firebaseapp.com/rust/lock_order/index.html)。


拓展问题：
- 如何使这个库更易用? 每个要获取锁的函数都得传一个`Token`参数感觉不太方便。




拓展阅读：[ghostcell](https://docs.rs/ghost-cell/latest/ghost_cell/), 编译期安全的`RefCell`。
