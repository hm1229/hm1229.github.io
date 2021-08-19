# 实验步骤

## 动态内存分配

### 连续内存分配算法

假设我们已经有一整块虚拟内存用来分配，那么如何进行分配呢？

我们可能会想到一些简单粗暴的方法，比如对于一个分配任务，贪心地将其分配到可行的最小地址去。这样一直分配下去的话，我们分配出去的内存都是连续的，看上去很合理的利用了内存。

但是一旦涉及到回收的话，设想我们在连续分配出去的很多块内存中间突然回收掉一块，它虽然是可用的，但是由于上下两边都已经被分配出去，它就只有这么大而不能再被拓展了，这种可用的内存我们称之为**外碎片**。

随着不断回收会产生越来越多的碎片，某个时刻我们可能会发现，需要分配一块较大的内存，几个碎片加起来大小是足够的，但是单个碎片是不够的。我们会想到通过**碎片整理**将几个碎片合并起来。但是这个过程的开销极大。

我们了解到有若干更有效的内存分配算法，包括伙伴系统（Buddy System）和 SLAB 分配器等算法，我们在这里使用 Buddy System 来实现这件事情。

> ## buddy system 算法简介
>
> 可分配的分区大小：`2^u`
>
> 当需要的大小满足：`2^(u-1) <= S <= 2^u` 时，将整块内存分配给S
>
> 数据结构：空闲块按大小和起始地址组成二维数组
>
> 初始状态：只有一块`2^u`大小的空闲块
>
> 分配过程：由小到大在空闲块数组中找最小的可用空闲块
>
> ​				   如果该空闲块过大，对可用空闲块进行二等分，直到合适。
>
> 释放过程：把释放的块放入空闲块数组
>
> ​				   合并满足条件的空闲块
>
> 合并条件：大小相同：2 ^ i
>
> ​				  地址相邻
>
> ​				   起始地址较小的块的起始地址必须时 2 ^ (i+1)的倍数
>
> 详细信息可以查看[伙伴系统]([Linux伙伴系统(一)--伙伴系统的概述_vanbreaker的专栏-CSDN博客_伙伴系统](https://blog.csdn.net/vanbreaker/article/details/7605367))



### 支持动态内存分配

为了避免重复造轮子，我们可以直接开一个静态的 8M 数组作为堆的空间，然后调用 [@jiege](https://github.com/jiegec/) 开发的 [Buddy System Allocator](https://github.com/rcore-os/buddy_system_allocator)。

```rust
//! os/src/memory/config.rs
//! 定义一些内存相关的常量
/// 操作系统动态分配内存所用的堆大小（8M）
pub const KERNEL_HEAP_SIZE: usize = 0x80_0000;
```

```rust
//! os/src/memory/heap.rs
//! 实现操作系统动态内存分配所用的堆
//!
//! 基于 `buddy_system_allocator` crate
use super::config::KERNEL_HEAP_SIZE;
use buddy_system_allocator::LockedHeap;

/// 进行动态内存分配所用的堆空间
/// 
/// 大小为 [`KERNEL_HEAP_SIZE`]  
/// 这段空间编译后会被放在操作系统执行程序的 bss 段
static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];

/// 堆，动态内存分配器
/// 
/// ### `#[global_allocator]`
/// [`LockedHeap`] 实现了 [`alloc::alloc::GlobalAlloc`] trait，
/// 可以为全局需要用到堆的地方分配空间。例如 `Box` `Arc` 等
#[global_allocator]
static HEAP: LockedHeap = LockedHeap::empty();

/// 初始化操作系统运行时堆空间
pub fn init() {
    // 告诉分配器使用这一段预留的空间作为堆
    unsafe {
        HEAP.lock().init(
            HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE     
        )
    }
}

/// 空间分配错误的回调，直接 panic 退出
#[alloc_error_handler]
fn alloc_error_handler(_: alloc::alloc::Layout) -> ! {
    panic!("空间分配错误")
}
```

> ### as_ptr()
>
> 将类型转化为裸指针

> ### [Rust 数组](https://kaisery.github.io/trpl-zh-cn/ch03-02-data-types.html)
>
> **数组**（*array*）。与元组不同，数组中的每个元素的类型必须相同。Rust 中的数组与一些其他语言中的数组不同，因为 Rust 中的数组是固定长度的：一旦声明，它们的长度不能增长或缩小。
>
> Rust 中，数组中的值位于中括号内的逗号分隔的列表中：
>
> 文件名: src/main.rs
>
> ```rust
> fn main() {
> let a = [1, 2, 3, 4, 5];
> }
> ```
>
> 当你想要在栈（stack）而不是在堆（heap）上为数据分配空间（第四章将讨论栈与堆的更多内容），或者是想要确保总是有固定数量的元素时，数组非常有用。但是数组并不如 vector 类型灵活。vector 类型是标准库提供的一个 **允许** 增长和缩小长度的类似数组的集合类型。当不确定是应该使用数组还是 vector 的时候，你可能应该使用 vector。第八章会详细讨论 vector。
>
> 一个你可能想要使用数组而不是 vector 的例子是，当程序需要知道一年中月份的名字时。程序不大可能会去增加或减少月份。这时你可以使用数组，因为我们知道它总是包含 12 个元素：
>
> ```rust
> let months = ["January", "February", "March", "April", "May", "June", "July",
>         "August", "September", "October", "November", "December"];
> ```
>
> 可以像这样编写数组的类型：在方括号中包含每个元素的类型，后跟分号，再后跟数组元素的数量。
>
> ```rust
> let a: [i32; 5] = [1, 2, 3, 4, 5];
> ```
>
> 这里，`i32` 是每个元素的类型。分号之后，数字 `5` 表明该数组包含五个元素。
>
> 以这种方式编写数组的类型看起来类似于初始化数组的另一种语法：如果要为每个元素创建包含相同值的数组，可以指定初始值，后跟分号，然后在方括号中指定数组的长度，如下所示：
>
> ```rust
> let a = [3; 5];
> ```
>
> 变量名为 `a` 的数组将包含 `5` 个元素，这些元素的值最初都将被设置为 `3`。这种写法与 `let a = [3, 3, 3, 3, 3];` 效果相同，但更简洁。



将函数声明：

```rust
//! os/src/memory/mod.rs
//! 内存管理模块
//!
//! 负责空间分配和虚拟地址映射

// 因为模块内包含许多基础设施类别，实现了许多以后可能会用到的函数，
// 所以在模块范围内不提示「未使用的函数」等警告
#![allow(dead_code)]

pub mod config;
pub mod heap;

/// 初始化内存相关的子模块
///
/// - [`heap::init`]
pub fn init() {
    heap::init();
    // 允许内核读写用户态内存
    unsafe { riscv::register::sstatus::set_sum() };
    println!("内存初始化成功");
}
```

在`Cargo.toml`中添加依赖

```toml
# os/Cargo.toml
[dependencies]
buddy_system_allocator = "0.6.0"
lazy_static = { version = "1.4.0", features = ["spin_no_std"] }
```



### 动态内存分配测试

现在我们来测试一下动态内存分配是否有效，分配一个动态数组：

```rust
//! os/src/main.rs
//! # 全局属性
//! - `#![no_std]`  
//!   禁用标准库
#![no_std]
//!
//! - `#![no_main]`  
//!   不使用 `main` 函数等全部 Rust-level 入口点来作为程序入口
#![no_main]
//! # 一些 unstable 的功能需要在 crate 层级声明后才可以使用
//!
//! - `#![feature(alloc_error_handler)]`
//!   我们使用了一个全局动态内存分配器，以实现原本标准库中的堆内存分配。
//!   而语言要求我们同时实现一个错误回调，这里我们直接 panic
#![feature(alloc_error_handler)]
//! # 一些 unstable 的功能需要在 crate 层级声明后才可以使用
//! - `#![feature(llvm_asm)]`  
//!   内嵌汇编
#![feature(llvm_asm)]
//!
//! - `#![feature(global_asm)]`  
//!   内嵌整个汇编文件
#![feature(global_asm)]
//!
//! - `#![feature(panic_info_message)]`  
//!   panic! 时，获取其中的信息并打印
#![feature(panic_info_message)]

#[macro_use]
mod console;
mod panic;
mod sbi;
mod interrupt;
mod memory;
extern crate alloc;
// 汇编编写的程序入口，具体见该文件
global_asm!(include_str!("entry.asm"));

/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
#[no_mangle]
pub extern "C" fn rust_main() -> ! {
    // 初始化各种模块
    interrupt::init();
    memory::init();

    // 动态内存分配测试
    use alloc::boxed::Box;
    use alloc::vec::Vec;
    let v = Box::new(5);       //分配一段内存存放数字5
    assert_eq!(*v, 5);
    core::mem::drop(v);

    let mut vec = Vec::new();   //新建vector数组
    for i in 0..10000 {
        vec.push(i);		   //在数组中添加测试
    }
    assert_eq!(vec.len(), 10000);
    for (i, value) in vec.into_iter().enumerate() {    //遍历vec，生成索引序列(数据下标,数据)
        assert_eq!(value, i);
    }
    println!("堆栈测试通过！");

    println!("Hello rCore-Tutorial!");
    
    loop{}
}
```



> ### `for i in 0..10000{}`
>
> 循环10000次，`i` 从0到9999



> ### Macro [std](https://doc.rust-lang.org/std/index.html)::[assert_eq](https://doc.rust-lang.org/std/macro.assert_eq.html)
>
> ```rust
> macro_rules! assert_eq {
> 	($left:expr, $right:expr $(,)?) => { ... };
> 	($left:expr, $right:expr, $($arg:tt)+) => { ... };
> }
> ```
>
> 断言两个表达式彼此相等（使用[`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)）。
>
> 不相等时，此宏将打印表达式的值及其调试表示并**引发中断**。**值相等时不进行操作。**
>
> 像 [`assert!`](https://doc.rust-lang.org/std/macro.assert.html) 一样，此宏具有第二种形式，可以在其中提供自定义的紧急消息。



> ## [智能指针 `Box `](https://kaisery.github.io/trpl-zh-cn/ch15-01-box.html?highlight=Box#使用box-t指向堆上的数据)
>
> 最简单直接的智能指针是 *box*，其类型是 `Box<T>`。 box 允许你将一个值放在堆上而不是栈上。留在栈上的则是指向堆数据的指针。
>
> 除了数据被储存在堆上而不是栈上之外，box 没有性能损失。不过也没有很多额外的功能。它们多用于如下场景：
>
> - 当有一个在编译时未知大小的类型，而又想要在需要确切大小的上下文中使用这个类型值的时候
> - 当有大量数据并希望在确保数据不被拷贝的情况下转移所有权的时候
> - 当希望拥有一个值并只关心它的类型是否实现了特定 trait 而不是其具体类型的时候



> ## [使用 `Box` 在堆上储存数据](https://kaisery.github.io/trpl-zh-cn/ch15-01-box.html?highlight=Box#使用-boxt-在堆上储存数据)
>
> 在讨论 `Box<T>` 的用例之前，让我们熟悉一下语法以及如何与储存在 `Box<T>` 中的值进行交互。
>
> 示例 15-1 展示了如何使用 box 在堆上储存一个 `i32`：
>
> ```rust
> fn main() {
>  let b = Box::new(5);
>  println!("b = {}", b);
> }
> b = 5
> ```
>
> 示例 15-1：使用 box 在堆上储存一个 `i32` 值
>
> 这里定义了变量 `b`，其值是一个指向被分配在堆上的值 `5` 的 `Box`。这个程序会打印出 `b = 5`；在这个例子中，我们可以像数据是储存在栈上的那样访问 box 中的数据。正如任何拥有数据所有权的值那样，当像 `b` 这样的 box 在 `main` 的末尾离开作用域时，它将被释放。这个释放过程作用于 box 本身（位于栈上）和它所指向的数据（位于堆上）。
>
> 将一个单独的值存放在堆上并不是很有意义，所以像示例 15-1 这样单独使用 box 并不常见。将像单个 `i32` 这样的值储存在栈上，也就是其默认存放的地方在大部分使用场景中更为合适。让我们看看一个不使用 box 时无法定义的类型的例子。



> ### Function [core](http://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/core/index.html)::[mem](http://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/core/mem/index.html)::[drop](http://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/core/mem/fn.drop.html)
>
> ```
> pub fn drop<T>(_x: T)
> ```
>
> 删除值，释放其内存，但不删除其借用



> ### Module [alloc](http://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/alloc/index.html)::[vec](http://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/alloc/vec/index.html)
>
> 可分配堆内容的连续可增长数组类型。
>
> 具有索引、push、pop 功能



最后，运行一下会看到 `heap test passed` 类似的输出。有了这个工具之后，后面我们就可以使用一系列诸如 `Vec` 等基于动态分配实现的库中的结构了。



## 物理内存探测

### 物理内存的相关概念

我们知道，物理地址访问的通常是一片 DRAM，我们可以把它看成一个以字节为单位的大数组，通过物理地址找到对应的位置进行读写。但是，物理地址并不仅仅只能访问 DRAM，也可以用来访问其他的外设，因此你也可以认为 DRAM 也算是一种外设，物理地址则是一个对可以存储的介质的一种抽象。

而如果访问其他外设要使用不同的指令（如 x86 单独提供了 `in` 和 `out` 等指令来访问不同于内存的 IO 地址空间），会比较麻烦；于是，很多指令集架构（如 RISC-V、ARM 和 MIPS 等）通过 MMIO（Memory Mapped I/O）技术将外设映射到一段物理地址，这样我们访问其他外设就和访问物理内存一样了。

我们先不管那些外设，来看物理内存。



### 物理内存探测

操作系统怎样知道物理内存所在的那段物理地址呢？在 RISC-V 中，这个一般是由 bootloader，即 OpenSBI 固件来完成的。它来完成对于包括物理内存在内的各外设的扫描，将扫描结果以 DTB（Device Tree Blob）的格式保存在物理内存中的某个地方。随后 OpenSBI 固件会将设备树的地址保存在 `a1` 寄存器中，给我们使用。

这个扫描结果描述了所有外设的信息，当中也包括 QEMU 模拟的 RISC-V Virt 计算机中的物理内存。

>  **QEMU 模拟的 RISC-V Virt 计算机中的物理内存**
>
>  通过查看 QEMU 代码中 [`hw/riscv/virt.c`](https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c) 的 `virt_memmap[]` 的定义，可以了解到 QEMU 模拟的 RISC-V Virt 计算机的详细物理内存布局。可以看到，整个物理内存中有不少内存空洞（即含义为 unmapped 的地址空间），也有很多外设特定的地址空间，现在我们看不懂没有关系，后面会慢慢涉及到。目前只需关心最后一块含义为 DRAM 的地址空间，这就是 OS 将要管理的 128 MB 的内存空间。
>
>  |  起始地址  |  终止地址  | 含义                        |
>  | :--------: | :--------: | :-------------------------- |
>  |    0x0     |   0x100    | QEMU VIRT_DEBUG             |
>  |   0x100    |   0x1000   | unmapped                    |
>  |   0x1000   |  0x12000   | QEMU MROM                   |
>  |  0x12000   |  0x100000  | unmapped                    |
>  |  0x100000  |  0x101000  | QEMU VIRT_TEST              |
>  |  0x101000  | 0x2000000  | unmapped                    |
>  | 0x2000000  | 0x2010000  | QEMU VIRT_CLINT             |
>  | 0x2010000  | 0x3000000  | unmapped                    |
>  | 0x3000000  | 0x3010000  | QEMU VIRT_PCIE_PIO          |
>  | 0x3010000  | 0xc000000  | unmapped                    |
>  | 0xc000000  | 0x10000000 | QEMU VIRT_PLIC              |
>  | 0x10000000 | 0x10000100 | QEMU VIRT_UART0             |
>  | 0x10000100 | 0x10001000 | unmapped                    |
>  | 0x10001000 | 0x10002000 | QEMU VIRT_VIRTIO            |
>  | 0x10002000 | 0x20000000 | unmapped                    |
>  | 0x20000000 | 0x24000000 | QEMU VIRT_FLASH             |
>  | 0x24000000 | 0x30000000 | unmapped                    |
>  | 0x30000000 | 0x40000000 | QEMU VIRT_PCIE_ECAM         |
>  | 0x40000000 | 0x80000000 | QEMU VIRT_PCIE_MMIO         |
>  | 0x80000000 | 0x88000000 | DRAM 缺省 128MB，大小可配置 |

不过为了简单起见，我们并不打算自己去解析这个结果。因为我们知道，QEMU 规定的 DRAM 物理内存的起始物理地址为 0x80000000 。而在 QEMU 中，可以使用 `-m` 指定 RAM 的大小，默认是 128 MB 。因此，默认的 DRAM 物理内存地址范围就是 [0x80000000, 0x88000000)。

因为后面还会涉及到虚拟地址、物理页和虚拟页面的概念，为了进一步区分而不是简单的只是使用 `usize` 类型来存储，我们首先建立一个 `PhysicalAddress` 的类，然后对其实现一系列的 `usize` 的加、减和输出等等操作，这部分实现更偏向于 Rust 语法而非 OS.

```rust
//! os/src/memory/address.rs
//! 定义地址类型和地址常量
//!
//! 我们为物理地址设立类型，利用编译器检查来防止混淆。
//!
//! # 类型
//!
//! - 物理地址 [`PhysicalAddress`]
//! - 物理页号 [`PhysicalPageNumber`]
//!
//! 两四种类型均由一个 `usize` 来表示
//!
//! # 类型转换
//!
//! ### 与基本类型的转换
//!
//! - 两种类型均实现了 `From<usize>` 和 `Into<usize>`
//!
//! 物理 → 物理
//!
//! - 页号至地址：直接乘以页面大小
//! - 地址至页号：应当使用页号类型的 [`floor`] 和 [`ceil`] 静态方法来转换

//! # 基本运算
//!
//! - 两种类型都可以直接与 `usize` 进行加减，返回结果为原本类型
//! - 两种类型都可以与自己类型进行加减，返回结果为 `usize`

use super::config::PAGE_SIZE;
/// 物理地址
#[repr(C)]
#[derive(Copy, Clone, Debug, Default, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct PhysicalAddress(pub usize);

/// 物理页号
#[repr(C)]
#[derive(Copy, Clone, Debug, Default, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct PhysicalPageNumber(pub usize);

macro_rules! implement_address_to_page_number {
    // 这里面的类型转换实现 [`From`] trait，会自动实现相反的 [`Into`] trait
    ($address_type: ty, $page_number_type: ty) => {
        impl From<$page_number_type> for $address_type {
            /// 从页号转换为地址
            fn from(page_number: $page_number_type) -> Self {
                Self(page_number.0 * PAGE_SIZE)
            }
        }
        impl From<$address_type> for $page_number_type {
            /// 从地址转换为页号，直接进行移位操作
            ///
            /// 不允许转换没有对齐的地址，这种情况应当使用 `floor()` 和 `ceil()`
            fn from(address: $address_type) -> Self {
                assert!(address.0 % PAGE_SIZE == 0);
                Self(address.0 / PAGE_SIZE)
            }
        }
        impl $page_number_type {
            /// 将地址转换为页号，向下取整
            pub const fn floor(address: $address_type) -> Self {
                Self(address.0 / PAGE_SIZE)
            }
            /// 将地址转换为页号，向上取整
            pub const fn ceil(address: $address_type) -> Self {
                Self(address.0 / PAGE_SIZE + (address.0 % PAGE_SIZE != 0) as usize)
            }
        }
    };
}
implement_address_to_page_number! {PhysicalAddress, PhysicalPageNumber}



/// 为各种仅包含一个 usize 的类型实现运算操作
macro_rules! implement_usize_operations {
    ($type_name: ty) => {
        /// `+`
        impl core::ops::Add<usize> for $type_name {
            type Output = Self;
            fn add(self, other: usize) -> Self::Output {
                Self(self.0 + other)
            }
        }
        /// `+=`
        impl core::ops::AddAssign<usize> for $type_name {
            fn add_assign(&mut self, rhs: usize) {
                self.0 += rhs;
            }
        }
        /// `-`
        impl core::ops::Sub<usize> for $type_name {
            type Output = Self;
            fn sub(self, other: usize) -> Self::Output {
                Self(self.0 - other)
            }
        }
        /// `-`
        impl core::ops::Sub<$type_name> for $type_name {
            type Output = usize;
            fn sub(self, other: $type_name) -> Self::Output {
                self.0 - other.0
            }
        }
        /// `-=`
        impl core::ops::SubAssign<usize> for $type_name {
            fn sub_assign(&mut self, rhs: usize) {
                self.0 -= rhs;
            }
        }
        /// 和 usize 相互转换
        impl From<usize> for $type_name {
            fn from(value: usize) -> Self {
                Self(value)
            }
        }
        /// 和 usize 相互转换
        impl From<$type_name> for usize {
            fn from(value: $type_name) -> Self {
                value.0
            }
        }
        impl $type_name {
            /// 是否有效（0 为无效）
            pub fn valid(&self) -> bool {
                self.0 != 0
            }
        }
        /// {} 输出
        impl core::fmt::Display for $type_name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                write!(f, "{}(0x{:x})", stringify!($type_name), self.0)
            }
        }
    };
}
implement_usize_operations! {PhysicalAddress}
implement_usize_operations! {PhysicalPageNumber}
```

> ### 元组结构体
>
> 与元组类似的结构体，称为 **元组结构体**（*tuple structs*）。元组结构体有着结构体名称提供的含义，但没有具体的字段名，只有字段的类型。当你想给整个元组取一个名字，并使元组成为与其他元组不同的类型时，元组结构体是很有用的，这时像常规结构体那样为每个字段命名就显得多余和形式化了。
>
> 在其他方面，元组结构体实例类似于元组：可以将其解构为单独的部分，也可以使用 `.` 后跟索引来访问单独的值，等等。

然后，我们直接将 DRAM 物理内存结束地址硬编码到内核中，同时因为我们操作系统本身也用了一部分空间，我们也记录下操作系统用到的地址结尾（即 linker script 中的 `kernel_end`）。

```rust
//! os/src/memory/config.rs

use super::address::*;
use lazy_static::lazy_static;

/// 操作系统动态分配内存所用的堆大小（8M）
pub const KERNEL_HEAP_SIZE: usize = 0x80_0000;
/// 定义一些目前暂不需要使用的常量来维持保证address.rs能被正确初始化,如页的大小
pub const PAGE_SIZE: usize = 4096;

lazy_static! {
    /// 内核代码结束的地址，即可以用来分配的内存起始地址
    ///
    /// 因为 Rust 语言限制，我们只能将其作为一个运行时求值的 static 变量，而不能作为 const
    pub static ref KERNEL_END_ADDRESS: PhysicalAddress = PhysicalAddress(kernel_end as usize);
}

extern "C" {
    /// 由 `linker.ld` 指定的内核代码结束位置
    ///
    /// 作为变量存在 [`KERNEL_END_ADDRESS`]
    fn kernel_end();
}
```

这里使用了 `lazy_static` 库，由于 Rust 语言的限制，我们能对编译时 `kernel_end` 做一个求值然后赋值到 `KERNEL_END_ADDRESS` 中；所以，`lazy_static!` 宏帮助我们在第一次使用 `lazy_static!` 宏包裹的变量时自动完成这些求值工作。

> ## `lazy_static!`
>
> 给静态变量延迟赋值的宏。
>
> 使用这个宏,所有 `static`类型的变量可在执行的代码在运行时被初始化。 这包括任何需要堆分配,如`vector`或`hashmap`,以及任何非常量函数调用。

在`src/memmory/mod.rs`中添加对address的声明：

```rust
//os/src/memmory/mod.rs

pub mod address;
```

在 `os/src/main.rs` 尝试输出。

```rust
/// os/src/main.rs
/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
#[no_mangle]
pub extern "C" fn rust_main() -> ! {
    // 初始化各种模块
    interrupt::init();
    memory::init();
    
    println!("Hello rCore-Tutorial!");
    // 注意这里的 KERNEL_END_ADDRESS 为 ref 类型，需要加 *
    println!("内核结束地址:{}", *memory::config::KERNEL_END_ADDRESS);

    panic!()
}
```

最后运行，可以看到成功显示了我们内核使用的结尾地址 `PhysicalAddress(0x80a187a8)`；注意到这里，你的输出可能因为实现上的细节并不完全一样。



## 物理内存管理

### 物理页

通常，我们在分配物理内存时并不是以字节为单位，而是以一**物理页（Frame）**，即连续的 4 KB 字节为单位分配。我们希望用物理页号（Physical Page Number，PPN）来代表一物理页，实际上代表物理地址范围在[PPN×4KB,(PPN+1)×4KB] 的一物理页。

不难看出，物理页号与物理页形成一一映射。为了能够使用物理页号这种表达方式，每个物理页的开头地址必须是 4 KB 的倍数。但这也给了我们一个方便：对于一个物理地址，其除以 4096（或者说右移 12 位）的商即为这个物理地址所在的物理页号。

同样的，我们还是用一个新的结构来封装一下物理页，一是为了和其他类型地址作区分；二是我们可以同时实现一些页帧和地址相互转换的功能。为了后面的方便，我们也把虚拟地址和虚拟页（概念还没有涉及，后面的指导会进一步讲解）一并实现出来，这部分代码请参考 `os/src/memory/address.rs`。

同时，我们也需要在 `os/src/memory/config.rs` 中加入相关的设置：

```rust
/// os/src/memory/config.rs
/// 页 / 帧大小，必须是 2^n
pub const PAGE_SIZE: usize = 4096;

/// 可以访问的内存区域起始地址
pub const MEMORY_START_ADDRESS: PhysicalAddress = PhysicalAddress(0x8000_0000);
/// 可以访问的内存区域结束地址
pub const MEMORY_END_ADDRESS: PhysicalAddress = PhysicalAddress(0x8800_0000);
```

> ## `0x8800_0000` 的下划线是什么
>
> 在数字中加下划线。它们被用来分隔数字组，就像在非编程中使用逗号一样。下划线在数字中被完全忽略，就像注释一样。



### 分配和回收

为了方便管理所有的物理页，我们需要实现一个分配器可以进行分配和回收的操作，在这之前，我们需要先把物理页的概念进行封装。注意到，物理页实际上是一块连续的内存区域，这里我们只是把内存区域的起始物理地址封装到 `FrameTracker` 里面。

```rust
/// os/src/memory/frame/frame_tracker.rs
/// 分配出的物理页
///
/// # `Tracker` 是什么？
///
/// > 可以理解为 [`Box`](alloc::boxed::Box)，而区别在于，其空间不是分配在堆上，
/// > 而是直接在内存中划一片（一个物理页）。
///
/// 在我们实现操作系统的过程中，会经常遇到「指定一块内存区域作为某种用处」的情况。
/// 此时，我们说这块内存可以用，但是因为它不在堆栈上，Rust 编译器并不知道它是什么，所以
/// 我们需要 unsafe 地将其转换为 `&'static mut T` 的形式（`'static` 一般可以省略）。
///
/// 但是，比如我们用一块内存来作为页表，而当这个页表我们不再需要的时候，就应当释放空间。
/// 我们其实更需要一个像「创建一个有生命期的对象」一样的模式来使用这块内存。因此，
/// 我们不妨用 `Tracker` 类型来封装这样一个 `&'static mut` 引用。
///
/// 使用 `Tracker` 其实就很像使用一个 smart pointer。如果需要引用计数，
/// 就在外面再套一层 [`Arc`](alloc::sync::Arc) 就好
use crate::memory::{PhysicalPageNumber,PhysicalAddress,FRAME_ALLOCATOR};

pub struct FrameTracker(pub(super) PhysicalPageNumber);

impl FrameTracker {
    /// 帧的物理地址
    pub fn address(&self) -> PhysicalAddress {
        self.0.into()
    }
    /// 帧的物理页号
    pub fn page_number(&self) -> PhysicalPageNumber {
        self.0
    }
}

/// 帧在释放时会放回 [`static@FRAME_ALLOCATOR`] 的空闲链表中
impl Drop for FrameTracker {
    fn drop(&mut self) {
        FRAME_ALLOCATOR.lock().dealloc(self);
    }
}
```



> ### [``&self` 、 `Self` 和 `self`](https://rust-by-example.budshome.com/fn/methods.html)
>
> `&self` 是 `self: &Self` 的语法糖（sugar）
>
> `Self` 是方法调用者的类型
>
> `self` 为 `self: Self` 的语法糖

> ###  特性 `trait`
>
> `trait` 是对未知类型 `Self` 定义的方法集。该类型也可以访问同一个 trait 中定义的 其他方法。
>
> 对任何数据类型都可以实现 trait。在下面例子中，我们定义了包含一系列方法 的 `Animal`。然后针对 `Sheep` 数据类型实现 `Animal` `trait`，因而 `Sheep` 的实例可以使用 `Animal` 中的所有方法。
>
> ```rust
> struct Sheep { naked: bool, name: &'static str }
> 
> trait Animal {
> // 静态方法签名；`Self` 表示实现者类型（implementor type）。
> fn new(name: &'static str) -> Self;
> 
> // 实例方法签名；这些方法将返回一个字符串。
> fn name(&self) -> &'static str;
> fn noise(&self) -> &'static str;
> 
> // trait 可以提供默认的方法定义。
> fn talk(&self) {
>   println!("{} says {}", self.name(), self.noise());
> }
> }
> 
> impl Sheep {
> fn is_naked(&self) -> bool {
>   self.naked
> }
> 
> fn shear(&mut self) {
>   if self.is_naked() {
>       // 实现者可以使用它的 trait 方法。
>       println!("{} is already naked...", self.name());
>   } else {
>       println!("{} gets a haircut!", self.name);
> 
>       self.naked = true;
>   }
> }
> }
> 
> // 对 `Sheep` 实现 `Animal` trait。
> impl Animal for Sheep {
> // `Self` 是实现者类型：`Sheep`。
> fn new(name: &'static str) -> Sheep {
>   Sheep { name: name, naked: false }
> }
> 
> fn name(&self) -> &'static str {
>   self.name
> }
> 
> fn noise(&self) -> &'static str {
>   if self.is_naked() {
>       "baaaaah?"
>   } else {
>       "baaaaah!"
>   }
> }
> 
> // 默认 trait 方法可以重载。
> fn talk(&self) {
>   // 例如我们可以增加一些安静的沉思。
>   println!("{} pauses briefly... {}", self.name, self.noise());
> }
> }
> 
> fn main() {
> // 这种情况需要类型标注。
> let mut dolly: Sheep = Animal::new("Dolly");
> // 试一试 ^ 移除类型标注。
> 
> dolly.talk();
> dolly.shear();
> dolly.talk();
> }
> 
> ```

这里，我们实现了 `FrameTracker` 这个结构，而区分于实际在内存中的 4KB 大小的 "Frame"，我们设计的初衷是分配器分配给我们 `FrameTracker` 作为一个帧的标识，而随着程序不再需要这个物理页，我们需要回收，我们利用 Rust 的 drop 机制在析构的时候自动实现回收。

最后，我们封装一个物理页分配器，为了符合更 Rust 规范的设计，这个分配器将不涉及任何的具体算法，具体的算法将用一个名为 `Allocator` 的 Rust trait 封装起来，而我们的 `FrameAllocator` 会依赖于具体的 trait 实现例化。

```rust
//! os/src/memory/frame/allocator.rs
//! 提供帧分配器 [`FRAME_ALLOCATOR`](FrameAllocator)
//!
//! 返回的 [`FrameTracker`] 类型代表一个帧，它在被 drop 时会自动将空间补回分配器中。

use super::*;
use crate::memory::*;
use algorithm::*;
use lazy_static::*;
use spin::Mutex;

lazy_static! {
    /// 帧分配器
    pub static ref FRAME_ALLOCATOR: Mutex<FrameAllocator<AllocatorImpl>> = Mutex::new(FrameAllocator::new(Range::from(
            PhysicalPageNumber::ceil(PhysicalAddress::from(*KERNEL_END_ADDRESS))..PhysicalPageNumber::floor(MEMORY_END_ADDRESS),
        )
    ));
}

/// 基于线段树的帧分配 / 回收
pub struct FrameAllocator<T: Allocator> {
    /// 可用区间的起始
    start_ppn: PhysicalPageNumber,
    /// 分配器
    allocator: T,
}

impl<T: Allocator> FrameAllocator<T> {
    /// 创建对象
    pub fn new(range: impl Into<Range<PhysicalPageNumber>> + Copy) -> Self {
        FrameAllocator {
            start_ppn: range.into().start,
            allocator: T::new(range.into().len()),
        }
    }

    /// 分配帧，如果没有剩余则返回 `Err`
    pub fn alloc(&mut self) -> MemoryResult<FrameTracker> {
        self.allocator
            .alloc()
            .ok_or("no available frame to allocate")
            .map(|offset| FrameTracker(self.start_ppn + offset))
    }

    /// 将被释放的帧添加到空闲列表的尾部
    ///
    /// 这个函数会在 [`FrameTracker`] 被 drop 时自动调用，不应在其他地方调用
    pub(super) fn dealloc(&mut self, frame: &FrameTracker) {
        self.allocator.dealloc(frame.page_number() - self.start_ppn);
    }
}
```

> ### trait 作为参数
>
> 知道了如何定义 trait 和在类型上实现这些 trait 之后，我们可以探索一下如何使用 trait 来接受多种不同类型的参数。
>
> 我们可以定义一个函数 `notify` 来调用其参数 `item` 上的 `summarize` 方法，该参数是实现了 `Summary` trait 的某种类型。为此可以使用 `impl Trait` 语法，像这样：
>
> ```rust
> pub fn notify(item: impl Summary) {
>  println!("Breaking news! {}", item.summarize());
> }
> ```
>
> 对于 `item` 参数，我们指定了 `impl` 关键字和 trait 名称，而不是具体的类型。该参数支持任何实现了指定 trait 的类型。在 `notify` 函数体中，可以调用任何来自 `Summary` trait 的方法，比如 `summarize`。我们可以传递任何 `NewsArticle` 或 `Tweet` 的实例来调用 `notify`。任何用其它如 `String` 或 `i32` 的类型调用该函数的代码都不能编译，因为它们没有实现 `Summary`。
>
> #### Trait Bound 语法
>
> `impl Trait` 语法适用于直观的例子，它不过是一个较长形式的语法糖。这被称为 *trait bound*，这看起来像：
>
> ```rust
> pub fn notify<T: Summary>(item: T) {
>  println!("Breaking news! {}", item.summarize());
> }
> ```
>
> 这与之前的例子相同，不过稍微冗长了一些。trait bound 与泛型参数声明在一起，位于尖括号中的冒号后面。
>
> `impl Trait` 很方便，适用于短小的例子。trait bound 则适用于更复杂的场景。
>
> #### 通过 `+` 指定多个 trait bound
>
> 如果 `notify` 需要显示 `item` 的格式化形式，同时也要使用 `summarize` 方法，那么 `item` 就需要同时实现两个不同的 trait：`Display` 和 `Summary`。这可以通过 `+` 语法实现：
>
> ```rust
> pub fn notify(item: impl Summary + Display) {
> ```
>
> `+` 语法也适用于泛型的 trait bound：
>
> ```rust
> pub fn notify<T: Summary + Display>(item: T) {
> ```
>
> 通过指定这两个 trait bound，`notify` 的函数体可以调用 `summarize` 并使用 `{}` 来格式化 `item`。

> ### 泛型的实现
>
> `<T>` 必须在类型之前写出来，以使类型 `T` 代表泛型。
>
> ```rust
> impl<T: Allocator> FrameAllocator<T> {
> 	...
> }
> ```

> ### 组合算子：`map`
>
> `match` 是处理 `Option` 的一个可用的方法，但你会发现大量使用它会很繁琐，特别是当 操作只对一种输入是有效的时。这时，可以使用组合算子（combinator），以 模块化的风格来管理控制流。
>
> `Option` 有一个内置方法 `map()`，这个组合算子可用于 `Some -> Some` 和 `None -> None` 这样的简单映射。多个不同的 `map()` 调用可以串起来，这样更加灵活。

> ### 闭包
>
> Rust 的 **闭包**（*closures*）是可以保存进变量或作为参数传递给其他函数的匿名函数。可以在一个地方创建闭包，然后在不同的上下文中执行闭包运算。不同于函数，闭包允许捕获调用者作用域中的值。我们将展示闭包的这些功能如何复用代码和自定义行为。
>
> 我们可以定义一个闭包并将其储存在变量中。实际上可以选择将整个 `simulated_expensive_calculation` 函数体移动到这里引入的闭包中：
>
> ```rust
> /*	函数写法
> fn simulated_expensive_calculation(intensity: u32) -> u32 {
> println!("calculating slowly...");
> thread::sleep(Duration::from_secs(2));
> intensity
> }
> */
> 
> //闭包写法
> let expensive_closure = |num| {
> println!("calculating slowly...");
> thread::sleep(Duration::from_secs(2));
> num
> };
> ```
>
> 它们的语法和能力使它们在临时（on the fly）使用时相当方便。调用一个闭包和调用一个 函数完全相同，不过调用闭包时，输入和返回类型两者都**可以**自动推导，而输入变量名**必须**指明。
>
> 其他的特点包括：
>
> - 声明时使用 `||` 替代 `()` 将输入参数括起来。
> - 函数体定界符（`{}`）对于单个表达式是可选的，其他情况必须加上。
> - 有能力捕获外部环境的变量。

添加 `os/src/memory/frame/mod.rs` 模块 

```rust
 //! os/src/memory/frame/mod.rs
 //! 物理页的分配与回收
 
 mod allocator;
 mod frame_tracker;
 
 pub use allocator::FRAME_ALLOCATOR;
 pub use frame_tracker::FrameTracker;
```

这个分配器会以一个 `PhysicalPageNumber` 的 `Range` 初始化

```rust
//! os/src/memory/range.rs
//! 表示一个页面区间 [`Range`]，提供迭代器功能

/// 表示一段连续的页面
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub struct Range<T: From<usize> + Into<usize> + Copy> {
    pub start: T,
    pub end: T,
}

/// 创建一个区间
impl<T: From<usize> + Into<usize> + Copy, U: Into<T>> From<core::ops::Range<U>> for Range<T> {
    fn from(range: core::ops::Range<U>) -> Self {
        Self {
            start: range.start.into(),
            end: range.end.into(),
        }
    }
}

impl<T: From<usize> + Into<usize> + Copy> Range<T> {
    /// 检测两个 [`Range`] 是否存在重合的区间
    pub fn overlap_with(&self, other: &Range<T>) -> bool {
        self.start.into() < other.end.into() && self.end.into() > other.start.into()
    }

    /// 迭代区间中的所有页
    pub fn iter(&self) -> impl Iterator<Item = T> {
        (self.start.into()..self.end.into()).map(T::from)
    }

    /// 区间大小
    pub fn len(&self) -> usize {
        self.end.into() - self.start.into()
    }

    /// 支持物理 / 虚拟页面区间互相转换
    pub fn into<U: From<usize> + Into<usize> + Copy + From<T>>(self) -> Range<U> {
        Range::<U> {
            start: U::from(self.start),
            end: U::from(self.end),
        }
    }

    /// 从区间中用下标取元素
    pub fn get(&self, index: usize) -> T {
        assert!(index < self.len());
        T::from(self.start.into() + index)
    }

    /// 区间是否包含指定的值
    pub fn contains(&self, value: T) -> bool {
        self.start.into() <= value.into() && value.into() < self.end.into()
    }
}
```

然后把起始地址记录下来，把整个区间的长度告诉具体的分配器算法，当分配的时候就从算法中取得一个可用的位置作为 `offset`，再加上起始地址返回回去。

有关具体的算法，我们封装了一个分配器需要的 Rust trait：

在 `os/src` 目录下新建algorithm库：

```shell
$cargo new algorithm --lib
```

在 `os/src/algorithm/Cargo.toml` 中添加依赖

```toml
# os/src/algorithm/Cargo.toml
[dependencies]
bit_field = "0.10.0"
```

声明分配器需要的 Rust trait

```rust
//! os/src/algorithm/src/allocator/mod.rs
//! 负责分配 / 回收的数据结构

mod segment_tree_allocator;
mod stacked_allocator;

/// 分配器：固定容量，每次分配 / 回收一个元素
pub trait Allocator {
    /// 给定容量，创建分配器
    fn new(capacity: usize) -> Self;
    /// 分配一个元素，分配成功返回页偏移值，无法分配则返回 `None`
    fn alloc(&mut self) -> Option<usize>;
    /// 回收一个元素
    fn dealloc(&mut self, index: usize);
}

pub use segment_tree_allocator::SegmentTreeAllocator;
pub use stacked_allocator::StackedAllocator;

/// 默认使用的分配器
pub type AllocatorImpl = StackedAllocator;
```

并在 `os/src/algorithm/src/allocator/` 中分别实现了栈式分配和线段树分配，具体内容可以参考代码。

栈式分配:

```rust
//os/src/algorithm/src/allocator/stacked_allocator.rs
//! 提供栈结构实现的分配器 [`StackedAllocator`]

use super::Allocator;
use alloc::{vec, vec::Vec};

/// 使用栈结构实现分配器
///
/// 在 `Vec` 末尾进行加入 / 删除。
/// 每个元素 tuple `(start, end)` 表示 [start, end) 区间为可用。
pub struct StackedAllocator {
    list: Vec<(usize, usize)>,
}

impl Allocator for StackedAllocator {
    //新建一个栈式分配器
    fn new(capacity: usize) -> Self {
        Self {
            list: vec![(0, capacity)],
        }
    }
	
    //使用栈结构实现内存分配
    fn alloc(&mut self) -> Option<usize> {
        if let Some((start, end)) = self.list.pop() { //获得剩余内存大小的start end
            if end - start > 1 {					//当空间>1页时
                self.list.push((start + 1, end));	  //分配一页后，将剩余重新压入栈	
            }
            Some(start)								//返回当前页偏移
        } else {
            None								    //如果大小不够，返回none
        }
    }

    fn dealloc(&mut self, index: usize) {
        self.list.push((index, index + 1));			//释放这一页地内容
    }
}
```

> ### `if let`简单控制流
>
> `if let` 语法让我们以一种不那么冗长的方式结合 `if` 和 `let`，来处理只匹配一个模式的值而忽略其他模式的情况
>
> ```rust
> //使用match
> let some_u8_value = Some(0u8);
> match some_u8_value {
>  Some(3) => println!("three"),
>  _ => (),
> }
> ```
>
> 我们可以使用 `if let` 这种更短的方式编写
>
> ```rust
> //使用if let
> let some_u8_value = Some(3);
> if let Some(3) = some_u8_value {
>  println!("three");
> }
> ```
>
> 我们还可以通过`if let`进行赋值操作
>
> ```rust
> //使用if let进行赋值
> let some_u8_value = Some(3);
> if let Some(x) = some_u8_value {
>  println!("{}",x);
> }
> //输出为 3
> ```

> ### 栈式分配
>
> 运行时，每当一个分程序或过程，其中各项数据项所需的存储空间就动态地分配于栈顶，退出时，则释放所占用地内存空间

线段树分配:

```rust
//! os/src/algorithm/src/allocator/segment_tree_allocator.rs
//! 提供线段树实现的分配器 [`SegmentTreeAllocator`]

use super::Allocator;
use alloc::{vec, vec::Vec};
use bit_field::BitArray;

/// 使用线段树实现分配器
pub struct SegmentTreeAllocator {
    /// 树本身
    tree: Vec<u8>,
}

impl Allocator for SegmentTreeAllocator {
    fn new(capacity: usize) -> Self {
        assert!(capacity >= 8);
        // 完全二叉树的树叶数量
        let leaf_count = capacity.next_power_of_two();
        let mut tree = vec![0u8; 2 * leaf_count];
        // 去除尾部超出范围的空间
        for i in ((capacity + 7) / 8)..(leaf_count / 8) {
            tree[leaf_count / 8 + i] = 255u8;
        }
        for i in capacity..(capacity + 8) {
            tree.set_bit(leaf_count + i, true);
        }
        // 沿树枝向上计算
        for i in (1..leaf_count).rev() {
            let v = tree.get_bit(i * 2) && tree.get_bit(i * 2 + 1);
            tree.set_bit(i, v);
        }
        Self { tree }
    }

    fn alloc(&mut self) -> Option<usize> {
        if self.tree.get_bit(1) {
            None
        } else {
            let mut node = 1;
            // 递归查找直到找到一个值为 0 的树叶
            while node < self.tree.len() / 2 {
                if !self.tree.get_bit(node * 2) {
                    node *= 2;
                } else if !self.tree.get_bit(node * 2 + 1) {
                    node = node * 2 + 1;
                } else {
                    panic!("tree is full or damaged");
                }
            }
            // 检验
            assert!(!self.tree.get_bit(node), "tree is damaged");
            // 修改树
            self.update_node(node, true);
            Some(node - self.tree.len() / 2)
        }
    }

    fn dealloc(&mut self, index: usize) {
        let node = index + self.tree.len() / 2;
        assert!(self.tree.get_bit(node));
        self.update_node(node, false);
    }
}

impl SegmentTreeAllocator {
    /// 更新线段树中一个树叶，然后递归更新其祖先
    fn update_node(&mut self, mut index: usize, value: bool) {
        self.tree.set_bit(index, value);
        while index > 1 {
            index /= 2;
            let v = self.tree.get_bit(index * 2) && self.tree.get_bit(index * 2 + 1);
            self.tree.set_bit(index, v);
        }
    }
}
```

> ### 线段树
>
> 假设有编号从1到n的n个点，每个点都存了一些信息，用[L,R]表示下标从L到R的这些点。
>
> 线段树的用处就是，对编号连续的一些点进行修改或者统计操作，修改和统计的复杂度都是O(log2(n)).
>
> 线段树的原理，就是，将[1,n]分解成若干特定的子区间(数量不超过4*n),然后，将每个区间[L,R]都分解为
>
> 少量特定的子区间，通过对这些少量子区间的修改或者统计，来实现快速对[L,R]的修改或者统计。

需要注意，我们使用了 `lazy_static!` 和 `Mutex` 来包装分配器，且对于 `static mut` 类型的修改操作是 unsafe 的。对于静态全局数据，所有的线程都能访问。当一个线程正在访问这段数据的时候，如果另一个线程也来访问，就可能会产生冲突，并带来难以预测的结果。我们在后面的章节会进一步介绍线程和 Mutex 等概念。

所以我们的方法是使用 `spin::Mutex<T>` 给这段数据加一把锁，一个线程试图通过 `lock()` 打开锁来获取内部数据的可变引用，如果钥匙被别的线程所占用，那么这个线程就会一直卡在这里；直到那个占用了钥匙的线程对内部数据的访问结束，锁被释放，将钥匙交还出来，被卡住的那个线程拿到了钥匙，就可打开锁获取内部引用，访问内部数据。

这里使用的是 `spin::Mutex<T>`，我们需要在 `os/Cargo.toml` 中添加依赖。幸运的是，它也无需任何操作系统支持（即支持 `no_std`），我们可以放心使用。

更改 `lib.rs` 声明模块

```rust
//! os/src/algorithm/src/lib.rs
//! 一些可能用到，而又不好找库的数据结构
//!
//! 以及有多种实现，会留作业的数据结构
#![no_std]
#![feature(drain_filter)]

extern crate alloc;

mod allocator;

pub use allocator::*;
```

将上面所写的程序进行声明：

```rust
// os/src/memory/mod.rs
//! 内存管理模块
//!
//! 负责空间分配和虚拟地址映射

// 因为模块内包含许多基础设施类别，实现了许多以后可能会用到的函数，
// 所以在模块范围内不提示「未使用的函数」等警告
#![allow(dead_code)]

pub mod address;
pub mod config;
pub mod frame;
pub mod heap;
pub mod range;

/// 一个缩写，模块中一些函数会使用
pub type MemoryResult<T> = Result<T, &'static str>;

pub use {
    address::*,
    config::*,
    frame::FRAME_ALLOCATOR,
    range::Range,
};

/// 初始化内存相关的子模块
///
/// - [`heap::init`]
pub fn init() {
    heap::init();
    // 允许内核读写用户态内存
    unsafe { riscv::register::sstatus::set_sum() };

    println!("内存初始化成功");
}
```

在`os/Cargo.toml`中添加依赖

```toml
# os/Cargo.toml
[dependencies]
spin = "0.5.2"
algorithm = { path = 'src/algorithm' }
```

最后，我们把新写的模块加载进来，并在 main 函数中进行简单的测试：

```rust
// os/src/main.rs
/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
#[no_mangle]
pub extern "C" fn rust_main() -> ! {
    // 初始化各种模块
    interrupt::init();
    memory::init();

    // 物理页分配
    for _ in 0..2 {
        let frame_0 = match memory::frame::FRAME_ALLOCATOR.lock().alloc() {
            Result::Ok(frame_tracker) => frame_tracker,
            Result::Err(err) => panic!("{}", err)
        };
        let frame_1 = match memory::frame::FRAME_ALLOCATOR.lock().alloc() {
            Result::Ok(frame_tracker) => frame_tracker,
            Result::Err(err) => panic!("{}", err)
        };
        println!("{} and {}", frame_0.address(), frame_1.address());
    }

    panic!()
}
```

可以看到类似这样的输出：

运行输出

```
PhysicalAddress(0x80a1e000) and PhysicalAddress(0x80a1f000)
PhysicalAddress(0x80a1e000) and PhysicalAddress(0x80a1f000)
```

我们可以看到 `frame_0` 和 `frame_1` 会被自动析构然后回收，第二次又分配同样的地址。