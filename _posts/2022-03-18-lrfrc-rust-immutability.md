---
layout: post
title:  "架构杂谈：Rust语言中不可变性、可变性、内部可变性"
date:   2022-03-18 00:06:05
categories: Rust LRFRC Mut Cell RefCell
excerpt: 学习Rust
---

* content
{:toc}


作为Rust语言初学者，往往会忽视不可变性与mut关键字的使用，难以理解Cell和RefCell设计逻辑及其使用场景。

通过阅读这篇文章，期望大家能对Rust语言一些架构设计上的选择及使用有全面理解。

---
#### 一、不可变性和可变性
不可变性Immutability，其实在我们日常生活中经常遇见比如：我们说过的话，走过的路，发出去的信息，都会成为历史，是不能改变的，也不会改变，它具有明显的时间属性。

为了更正我们前面已发出去的信息，我们往往会重新发一条新信息，如此循环，以达到沟通交流的目的。

在会计记账时，也是类似的情况，提交过的帐本不容许再修改。

通过Git提交代码后，Commit过的log也不容许再修改。

在计算机科学中，不可变性Immutability的应用，其实也一直在不断发展变化中，从最初的Copy-On-Write，到数据库Log记录，再到LSM Tree(Log-Structured Merge-Tree)算法，CRDTs(Conflict-free Replicated Data Types)数据结构及算法等。

从中我们可以看到不可变性，其实是现实中的常态，人们为了节省空间和时间或效率等才使用可变性，认为这些发生过的事实是可以修改的，但是发生过的事实如只涉及一个人或一个实体，往往不会存在问题，一旦涉及多个实体，就会遇到多个实体对共享内容进行读、写的同步问题。

从计算机的角度来看，参与共享内容读写的实体从单线程，到多线程，再到多主机。参与的实体越来越多，需要协作共享的内容也越来越复杂，进而提出各种并发并行计算的理论及架构设计、应用等；

对不可变性，期望有深入的理解，可参考论文Immutability Changes Everything或书籍<The Art of Immutable Architecture>；

---
#### 二、Rust中的不可变性和可变性
Rust语言的设计初衷是为了内存安全，特别是多线程并发环境下的安全，它要求所有声明的变量默认具有不可变性，比如:let a = 32; 描述变量a表示的值一旦初始化后，是不可变的，不能再修改它；

但是它可以不可变的方式传递给不同的函数、通过Channel发送出去在其他线程中处理，在其传递使用过程中，有可能多个实体期望访问它，则使用&的方式来引用它，由于被引用的变量a本身不变性，那么其&a引用变量也不能修改它；

不可变性变量的传递，往往代表了初始构建者一种责任，一旦传递出去就不会再修改它，由接收者来从中读取或Copy出其内容再处理，跟我们平常用手机发送消息的过程非常的类似；

其好处在于遇到这样的变量及其引用&，使用者都可知道它是不可变的，编译器可进行针对性的alias优化；

如果某个变量b后续可能要修改，则须加上mut声明来标记比如：let mut b = 100，这时也可通过&mut借用的方式来修改该变量表示的内容，这样的变量如果通过参数传递给其他线程<另一个实体>来使用，则会遇到多个实体对共享内容进行读、写的同步问题。

Rust语言为了实现对共享内容的修改，针对不同使用场景，设计了所有权及引用借用规则及内部可变性；

---
#### 三、单个程序块内的共享修改

对于一个可变变量来讲，在单个程序块{}内，可能会读取它，修改后再读取它，为保证安全，有了读取引用&的读取和修改变量的修改不能交叉使用；

下面程序不会编译通过:
```
let mut b = 100;
let c = &b;
let d = &mut b;
*d = 200;
println!("c={}", c);
```

而下面的程序可以编译通过：
```
let mut b = 100;
let d = &mut b;
*d = 200;
let c = &b;
println!("c={}", c);
```

---
#### 四、单线程内的共享修改

上面示例代码共享修改变量的方式，存在一个明显弊端，变量b必须标记mut，才有后续&b和&mut b变量。

哪是否可对缺省的不可变变量进行共享修改呢？这往往是我们传统语言使用的方式，定义的变量都可以进行共享修改。

但这个需求从变量本身及语言设计的不可变语义角度来讲，其实是有冲突的。

为了完成这个需求并不改变语言设计的不可变性，在标准库中<注意不是语言内置的功能>实现了Cell和RefCell。

使用Cell/RefCell通过包装方式来实现对其内部变量的修改。

通过包装方式的原因在于包装对象Cell/RefCell本身具有不可变语义，但它提供了方法可用来修改内部变量，在Cell/RefCell内部通过运行时状态来保证可变性和不可变性的切换和不可变引用/可变引用的规则检查。

由于通过内置在Cell/RefCell中来修改变量，故称之为内部可变性Interior Mutability。

其中Cell提供set/replace来设置/替换内置的整个变量；

RefCell提供borrow/borrow_mut方法先取得内置变量的引用或借用，然后再进行读取或修改。

Cell示例：
```
use std::cell::Cell;

struct SomeStruct {
    regular_field: u8,
    special_field: Cell<u8>,
}

let my_struct = SomeStruct {
    regular_field: 0,
    special_field: Cell::new(1),
};

let new_value = 100;
// ERROR: my_struct变量是不可变，不能直接修改其字段
// my_struct.regular_field = new_value;
// WORKS:虽然my_struct变量是不可变，但其special_field是Cell对象，
// 它提供了set方法，可以来修改它的值
my_struct.special_field.set(new_value);
assert_eq!(my_struct.special_field.get(), new_value);
```

RefCell示例：
```
use std::cell::RefCell;

fn main() {
  // c具有不可变性
  let c = RefCell::new(5);
  // 需要使用独立{}块
  {
    // 通过mut借用来修改内部值
    let mut c_mut_ref = c.borrow_mut();
    *c_mut_ref = 6;
    assert!(c.try_borrow().is_err());
  }
  // 需要使用独立{}块来分隔借用和引用的交叉使用，否则程序会发生panic.
  {
    // 通过引用来读取内部值
    let c_ref = c.borrow();
    assert!(c.try_borrow().is_ok());
    println!("c={}", c_ref);
  }
}
```
虽然使用Cell/RefCell可以达到共享修改的需求，但只能在单个线程中使用，因为其内部运行时状态的实现不能保证跨线程安全；

为区分这种类型，Rust语言引入Sync来表示这种类型的特性，比如：T: Sync 等价于 &T: Send。

所以Cell/RefCell没有Sync特性，但可有Send特性。
```
unsafe impl<T: ?Sized> Send for Cell<T> where T: Send {}

impl<T: ?Sized> !Sync for Cell<T> {}

unsafe impl<T: ?Sized> Send for RefCell<T> where T: Send {}

impl<T: ?Sized> !Sync for RefCell<T> {}
```

---
#### 五、多线程之间的共享修改
由于涉及到多个线程的对象或引用传递及修改，虽然其使用场景更复杂，但基于上面提到的不可变性及内部可变性，加上系统/基础库提供的同步机制，标准库提供了Arc和Mutex/RwLock来满足多线程之间的共享修改，其具体实现及使用可参考Arc/Mutex相关示例；

---
#### 六、总结
在Rust中不管是单个代码块还是单个线程，甚至多个线程之间共享/传递数据，能使用不可变变量的话，尽可能使用不可变变量，也就是Rust缺省声明定义变量及参数的方式；

如果需要共享可变的变量，需按照使用场景来组合使用标准库中提供的Cell、RefCell、Arc、Mutex等；

在Rust异步编程await/async模型中也大量运用了不可变性；

综上所述，Rust语言从架构上缺省选择对象不可变性，是一种编程范式上转变和突破。

---
参考
* [<font color="blue">Interior mutability in Rust: what, why, how?</font>](https://ricardomartins.cc/2016/06/08/interior-mutability)
* [<font color="blue">Immutability Changes Everything</font>](http://cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

