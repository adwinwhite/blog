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
参考线程安全的实现，我们可以用marker trait来描述类型的一些性质。我们希望描述的关系是各个锁之间的调用顺序关系。
我们需要了解logic与trait之间的对应关系，如果没了解过请看 [type level programming](https://willcrichton.net/notes/type-level-programming/)。

因为我们只能描述类型之间的关系而不是值间的关系，首先得让不同地方的锁拥有不同类型，给它们命个名，
用一个带Id类型参数的Wrapper. 
```rust
struct GoodLock<Id, T> {
    inner: Mutex<T>,
    id: PhantomData<Id>,
}
```
锁之间的调用顺序其实是一种DAG图或者说[strict partial order](https://en.wikipedia.org/wiki/Partially_ordered_set#Strict_partial_orders)关系，它具有传递性、非反射性、非对称性。
我们使用伪prolog代码来描述logic关系，`head:- body`表示如果`body`成立，则`head`成立。
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

现在我们在锁之间定义顺序关系了，我们要如何利用呢？
答案是利用[witness pattern](https://willcrichton.net/rust-api-type-patterns/witnesses.html)，即在获取一个锁的时候要求提供这次获取合乎顺序的证据。
我们用一个token来记录获取的锁的身份，它也是下个锁需要的证据。
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
大功告成！

拓展问题：
- 如何使这个库更易用? 每个要获取锁的函数都得传一个`Token`参数感觉不太方便。


完整实现和一些测试可以看[github](https://github.com/adwinwhite/goodlord)。
生产中使用可以看看fuchsia的[lock_order](https://fuchsia-docs.firebaseapp.com/rust/lock_order/index.html)。


拓展阅读：[ghostcell](https://docs.rs/ghost-cell/latest/ghost_cell/), 编译期安全的`RefCell`。
