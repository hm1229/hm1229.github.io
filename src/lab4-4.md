# 实验步骤

## 基本概念的抽象

### 重构上下文`context`

- 通用寄存器

  - `sp`：应当指向该线程的栈顶
  - `a0`-`a7`：按照函数调用规则，用来传递参数
  - `ra`：线程执行完应该跳转到哪里呢？在后续**系统调用**章节我们会介绍正确的处理方式。现在，我们先将其设为一个不可执行的地址，这样线程一结束就会触发页面异常

- `sepc`

  - 执行 `sret` 指令后会跳转到这里，所以 `sepc` 应当存储线程的入口地址（执行的函数地址）

- `sstatus`
- `spp` 位按照用户态或内核态有所不同
  - `spie` 位为 1

>  **`sstatus` 标志位的具体意义**
>
>  - `spp`：中断前系统处于内核态（1）还是用户态（0）
>  - `sie`：内核态是否允许中断。对用户态而言，无论 `sie` 取何值都开启中断
>  - `spie`：中断前是否开中断（用户态中断时可能 `sie` 为 0）
>
>  **硬件处理流程**
>
>  - 在中断发生时，系统要切换到内核态。此时，**切换前的状态**会被保存在 **`spp`** 位中（1 表示切换前处于内核态）。同时，**切换前是否开中断**会被保存在 **`spie`** 位中，而 `sie` 位会被置 0，表示关闭中断。
>  - 在中断结束，执行 `sret` 指令时，会根据 `spp` 位的值决定 `sret` 执行后是处于内核态还是用户态。与此同时，`spie` 位的值会被写入 `sie` 位，而 `spie` 位置 1。这样，特权状态和中断状态就全部恢复了。
>
>  **为何如此繁琐？**
>
>  - 特权状态：
>    中断处理流程必须切换到内核态，所以中断时需要用 `spp` 来保存之前的状态。
>    回忆计算机组成原理的知识，`sret` 指令必须同时完成跳转并切换状态的工作。
>  - 中断状态：
>    中断刚发生时，必须关闭中断，以保证现场保存的过程不会被干扰。同理，现场恢复的过程也必须关中断。因此，需要有以上两个硬件自动执行的操作。
>    由于中断可能嵌套，在保存现场后，根据中断的种类，可能会再开启部分中断的使能。

```rust
//os/src/interrupt/context.rs
//! 保存现场所用的 struct [`Context`]

use core::mem::zeroed;
use riscv::register::sstatus::{self, Sstatus, SPP::*};

/// 发生中断时，保存的寄存器
///
/// 包括所有通用寄存器，以及：
/// - `sstatus`：各种状态位
/// - `sepc`：产生中断的地址
///
/// ### `#[repr(C)]` 属性
/// 要求 struct 按照 C 语言的规则进行内存分布，否则 Rust 可能按照其他规则进行内存排布
#[repr(C)]
#[derive(Clone, Copy, Debug)]
pub struct Context {
    /// 通用寄存器
    pub x: [usize; 32],
    /// 保存诸多状态位的特权态寄存器
    pub sstatus: Sstatus,
    /// 保存中断地址的特权态寄存器
    pub sepc: usize,
}

/// 创建一个用 0 初始化的 Context
///
/// 这里使用 [`core::mem::zeroed()`] 来强行用全 0 初始化。
/// 因为在一些类型中，0 数值可能不合法（例如引用），所以 [`zeroed()`] 是 unsafe 的
impl Default for Context {
    fn default() -> Self {
        unsafe { zeroed() }
    }
}

#[allow(unused)]
impl Context {
    /// 获取栈指针
    pub fn sp(&self) -> usize {
        self.x[2]
    }

    /// 设置栈指针
    pub fn set_sp(&mut self, value: usize) -> &mut Self {
        self.x[2] = value;
        self
    }

    /// 获取返回地址
    pub fn ra(&self) -> usize {
        self.x[1]
    }

    /// 设置返回地址
    pub fn set_ra(&mut self, value: usize) -> &mut Self {
        self.x[1] = value;
        self
    }

    /// 按照函数调用规则写入参数
    ///
    /// 没有考虑一些特殊情况，例如超过 8 个参数，或 struct 空间展开
    pub fn set_arguments(&mut self, arguments: &[usize]) -> &mut Self {
        assert!(arguments.len() <= 8);
        self.x[10..(10 + arguments.len())].copy_from_slice(arguments);
        self
    }

    /// 为线程构建初始 `Context`
    pub fn new(
        stack_top: usize,
        entry_point: usize,
        arguments: Option<&[usize]>,
        is_user: bool,
    ) -> Self {
        let mut context = Self::default();

        // 设置栈顶指针
        context.set_sp(stack_top);
        // 设置初始参数
        if let Some(args) = arguments {
            context.set_arguments(args);
        }
        // 设置入口地址
        context.sepc = entry_point;

        // 设置 sstatus
        context.sstatus = sstatus::read();
        if is_user {
            context.sstatus.set_spp(User);
        } else {
            context.sstatus.set_spp(Supervisor);
        }
        // 这样设置 SPIE 位，使得替换 sstatus 后关闭中断，
        // 而在 sret 到用户线程时开启中断。详见 SPIE 和 SIE 的定义
        context.sstatus.set_spie(true);

        context
    }
}

```

### 线程的表示

在不同操作系统中，为每个线程所保存的信息都不同。在这里，我们提供一种基础的实现，每个线程会包括：

- **线程 ID**：用于唯一确认一个线程，它会在系统调用等时刻用到。
- **运行栈**：每个线程都必须有一个独立的运行栈，保存运行时数据。
- **线程执行上下文**：当线程不在执行时，我们需要保存其上下文（其实就是一堆**寄存器**的值），这样之后才能够将其恢复，继续运行。和之前实现的中断一样，上下文由 `Context` 类型保存。（注：这里的**线程执行上下文**与前面提到的**中断上下文**是不同的概念）
- **所属进程的记号**：同一个进程中的多个线程，会共享页表、打开文件等信息。因此，我们将它们提取出来放到线程中。
- **内核栈**：除了线程运行必须有的运行栈，中断处理也必须有一个单独的栈。之前，我们的中断处理是直接在原来的栈上进行（我们直接将 `Context` 压入栈）。但是在后面我们会引入用户线程，这时就只有上帝才知道发生了什么——栈指针、程序指针都可能在跨国（**国 == 特权态**）旅游。为了确保中断处理能够进行（让操作系统能够接管这样的线程），中断处理必须运行在一个准备好的、安全的栈上。这就是内核栈。不过，内核栈并没有存储在线程信息中。（注：**它的使用方法会有些复杂，我们会在后面讲解**。）

```rust
//os/src/process/thread.rs
//! 线程 [`Thread`]
use super::*;

/// 线程 ID 使用 `isize`，可以用负数表示错误
pub type ThreadID = isize;

/// 线程计数，用于设置线程 ID
static mut THREAD_COUNTER: ThreadID = 0;

/// 线程的信息
pub struct Thread {
    /// 线程 ID
    pub id: ThreadID,
    /// 线程的栈
    pub stack: Range<VirtualAddress>,
    /// 所属的进程
    pub process: Arc<Process>,
    /// 用 `Mutex` 包装一些可变的变量
    pub inner: Mutex<ThreadInner>,
}

/// 线程中需要可变的部分
pub struct ThreadInner {
    /// 线程执行上下文
    ///
    /// 当且仅当线程被暂停执行时，`context` 为 `Some`
    pub context: Option<Context>,
    /// 是否进入休眠
    pub sleeping: bool,
    /// 是否已经结束
    pub dead: bool,
}
```

注意到，因为线程一般使用 `Arc<Thread>` 来保存，它是不可变的，所以其中再用 `Mutex` 来包装一部分，让这部分可以修改。

>  **Rust语法卡片：引用计数 `Rc`**
>
> 为了启用多所有权，Rust 有一个叫做 `Rc<T>` 的类型。其名称为 **引用计数**（*reference counting*）的缩写。引用计数意味着记录一个值引用的数量来知晓这个值是否仍在被使用。如果某个值有零个引用，就代表没有任何有效引用并可以被清理。
>
> [引用计数 `Rc`详细信息](https://kaisery.github.io/trpl-zh-cn/ch15-04-rc.html)

> **Rust语法卡片：原子引用计数 `Arc`**
>
> `Arc<T>` 是个类似 `Rc<T>` 并可以安全的用于并发环境的类型。字母 “a” 代表 **原子性**（*atomic*），所以这是一个**原子引用计数**（*atomically reference counted*）类型。原子性是另一类这里还未涉及到的并发原语：请查看标准库中 `std::sync::atomic` 的文档来获取更多细节。其中的要点就是：原子性类型工作起来类似原始类型，不过可以安全的在线程间共享。
>
> [原子引用计数 `Arc` 详细信息](https://kaisery.github.io/trpl-zh-cn/ch16-03-shared-state.html#原子引用计数-arct)

> **原子性**
>
> 指事务的不可分割性，一个事物的所有操作要么不间断地全部被执行，要么一个也没有执行。

### 进程的表示

在我们实现的简单操作系统中，进程只需要维护页面映射，并且存储一点额外信息：

- **用户态标识**：我们会在后面进行区分内核态线程和用户态线程。
- **访存空间 `MemorySet`**：进程中的线程会共享同一个页表，即可以访问的虚拟内存空间（简称：访存空间）。

```rust
//os/src/process/process.rs
//! 进程 [`Process`]
use super::*;

/// 进程的信息
pub struct Process {
    /// 是否属于用户态
    pub is_user: bool,
    /// 用 `Mutex` 包装一些可变的变量
    pub inner: Mutex<ProcessInner>,
}

pub struct ProcessInner {
    /// 进程中的线程公用页表 / 内存映射
    pub memory_set: MemorySet,
}
```

同样地，线程也需要一部分是可变的。

### 处理器

有了线程和进程，现在，我们再抽象出「处理器」来存放和管理线程池。同时，也需要存放和管理目前正在执行的线程（即中断前执行的线程，因为操作系统在工作时是处于中断、异常或系统调用服务之中）。

```rust
//os/src/process/processor.rs
//! 实现线程的调度和管理 [`Processor`]

use super::*;
use algorithm::*;
use hashbrown::HashSet;
use lazy_static::*;

/// 线程调度和管理
///
/// 休眠线程会从调度器中移除，单独保存。在它们被唤醒之前，不会被调度器安排。
#[derive(Default)]
pub struct Processor {
    /// 当前正在执行的线程
    current_thread: Option<Arc<Thread>>,
    /// 线程调度器，记录活跃线程
    scheduler: SchedulerImpl<Arc<Thread>>,
    /// 保存休眠线程
    sleeping_threads: HashSet<Arc<Thread>>,
}
```

- `current_thread` 需要保存当前正在运行的线程，这样当出现系统调用的时候，操作系统便可以方便地知道是哪个线程在举手。

- `scheduler` 会负责调度线程，其接口就是简单的“添加”“移除”“获取下一个”，我们会在后面的线程调度中详细讲到，所以目前的程序是无法运行出来的。

- 休眠线程是指等待一些外部资源（例如硬盘读取、外设读取等）的线程，这时 CPU 如果给其时间片运行是没有意义的，因此它们也就需要移出调度器而单独保存。

添加依赖：

```toml
# os/Cargo.toml
[dependencies]
hashbrown = "0.8.1"
```

```rust
//os/src/process/processor.rs
lazy_static! {
    /// 全局的 [`Processor`]
    pub static ref PROCESSOR: Lock<Processor> = Lock::new(Processor::default());
}
lazy_static! {
    /// 空闲线程：当所有线程进入休眠时，切换到这个线程——它什么都不做，只会等待下一次中断
    static ref IDLE_THREAD: Arc<Thread> = Thread::new(
        Process::new_kernel().unwrap(),
        wait_for_interrupt as usize,
        None,
    ).unwrap();
}
/// 不断让 CPU 进入休眠等待下一次中断
unsafe fn wait_for_interrupt() {
    loop {
        llvm_asm!("wfi" :::: "volatile");
    }
}
```

> ## WFI和WFE
>
> 这两条指令的作用都是令MCU进入休眠/待机状态以便降低功耗，但是略有区别：
>
> WFI: wait for Interrupt 等待中断，即下一次中断发生前都在此hold住不干活
>
> WFE: wait for Events 等待事件，即下一次事件发生前都在此hold住不干活

注意到这里我们用了一个 `Lock`

```rust
//os/src/process/lock.rs
//! 一个关闭中断的互斥锁 [`Lock`]

use spin::{Mutex, MutexGuard};


/// 关闭中断的互斥锁
#[derive(Default)]
pub struct Lock<T>(pub(self) Mutex<T>);

/// 封装 [`MutexGuard`] 来实现 drop 时恢复 sstatus
pub struct LockGuard<'a, T> {
    /// 在 drop 时需要先 drop 掉 [`MutexGuard`] 再恢复 sstatus
    guard: Option<MutexGuard<'a, T>>,
    /// 保存的关中断前 sstatus
    sstatus: usize,
}

impl<T> Lock<T> {
    /// 创建一个新对象
    pub fn new(obj: T) -> Self {
        Self(Mutex::new(obj))
    }

    /// 获得上锁的对象
    pub fn lock(&self) -> LockGuard<'_, T> {
        let sstatus: usize;
        unsafe {
            llvm_asm!("csrrci $0, sstatus, 1 << 1" : "=r"(sstatus) ::: "volatile");   //将原始的sstasus寄存器的值赋给变量sstatus，并将寄存器sstatus的SIE位置零
        }
        LockGuard {
            guard: Some(self.0.lock()),
            sstatus,
        }
    }
}

/// 释放时，先释放内部的 MutexGuard，再恢复 sstatus 寄存器
impl<'a, T> Drop for LockGuard<'a, T> {
    fn drop(&mut self) {
        self.guard.take();
        unsafe { llvm_asm!("csrs sstatus, $0" :: "r"(self.sstatus & 2) :: "volatile") };     //将寄存器sstatus上的SIE位置1
    }
}

impl<'a, T> core::ops::Deref for LockGuard<'a, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        self.guard.as_ref().unwrap().deref()
    }
}

impl<'a, T> core::ops::DerefMut for LockGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.guard.as_mut().unwrap().deref_mut()
    }
}
```

> ## csrrci 
>
> csrrci rd, csr, imm[4:0]					t = CSRs[csr]; CSRs[csr] = t &~imm; x[rd] = t 
>
> 立即数读后清除控制状态寄存器 (Control and Status Register Read and Clear Immediate).
>
> 将csr索引寄存器值读出写到rd,5位立即数的值(高位补0）逐位比较，如果为1，则清零csr相应位。

> ## csrs
>
> csrs csr, rs1 									CSRs[csr] |= x[rs1] 
>
> 置位控制状态寄存器 (Control and Status Register Set). 伪指令(Pseudo instruction), RV32I and  RV64I. 对于 x[rs1]中每一个为 1 的位，把控制状态寄存器 csr 的的对应位置位，等同于 csrrs x0, csr,  rs1.

> ## csrrs
>
> csrrs rd, csr, rs1 								t = CSRs[csr]; CSRs[csr] = t | x[rs1]; x[rd] = t 
>
> 读后置位控制状态寄存器 (Control and Status Register Read and Set). I-type, RV32I and RV64I. 
>
> 记控制状态寄存器 csr 中的值为 t。把 t 和寄存器 x[rs1]按位或的结果写入 csr，再把 t 写入 x[rd]。

> ## sstatus 寄存器
>
> ![image-20210528203426583](C:\Users\qq171\AppData\Roaming\Typora\typora-user-images\image-20210528203426583.png)
>
> SIE 和 SPIE 中分别保存了当前的和异常发生之前的中断使能，RV32 的 XLEN 为 32，RV64 为 40。

它封装了 `spin::Mutex`，而在其基础上进一步关闭了中断。这是因为我们（以后）在内核线程中也有可能访问 `PROCESSOR`，但是此时我们不希望它被时钟打断，这样在中断处理中就无法访问 `PROCESSOR` 了，因为它已经被锁住。



## 线程的创建

接下来，我们的第一个目标就是创建一个线程并且让他运行起来。一个线程要开始运行，需要这些准备工作：

- 建立页表映射，需要包括以下映射空间：

  - 线程所执行的一段指令
  - 线程执行栈
  - *操作系统的部分内存空间*  *（当发生中断时，需要跳转到 `stvec` 所指向的中断处理过程。如果操作系统的内存不在页表之中，将无法处理中断。当然，也不是所有操作系统的代码都需要被映射，但是为了实现简便，我们会为每个进程的页表映射全部操作系统的内存。而由于这些页表都标记为**内核权限**（即 `U` 位为 0），也不必担心用户线程可以随意访问）。*
- 设置起始执行的地址
- 初始化各种寄存器，比如 `sp`
- 可选：设置一些执行参数（例如 `argc` 和 `argv`等 ）

新建线程并上锁：

```rust
////os/src/process/thread.rs
/// 创建一个线程
impl Thread{
    pub fn new(
        process: Arc<Process>,
        entry_point: usize,
        arguments: Option<&[usize]>,
    ) -> MemoryResult<Arc<Thread>> {
        // 让所属进程分配并映射一段空间，作为线程的栈
        let stack = process.alloc_page_range(STACK_SIZE, Flags::READABLE | Flags::WRITABLE)?;

        // 构建线程的 Context
        let context = Context::new(stack.end.into(), entry_point, arguments, process.is_user);

        // 打包成线程
        let thread = Arc::new(Thread {
            id: unsafe {
                THREAD_COUNTER += 1;
                THREAD_COUNTER
            },
            stack,
            process,
            inner: Mutex::new(ThreadInner {
                context: Some(context),
                sleeping: false,
                dead: false,
            }),
        });

        Ok(thread)
    }
    /// 上锁并获得可变部分的引用
    pub fn inner(&self) -> spin::MutexGuard<ThreadInner> {
        self.inner.lock()
    }
}    
```

在处理器中添加方法：

```rust
//os/src/process/processor.rs

impl Processor {
    /// 添加一个待执行的线程
    pub fn add_thread(&mut self, thread: Arc<Thread>) {
        self.scheduler.add_thread(thread);
    }
}
```

进程的新建,并上锁：

```rust
////os/src/process/process.rs
#[allow(unused)]
impl Process {
    /// 创建一个内核进程
    pub fn new_kernel() -> MemoryResult<Arc<Self>> {
        Ok(Arc::new(Self {
            is_user: false,
            inner: Mutex::new(ProcessInner {
                memory_set: MemorySet::new_kernel()?,
            }),
        }))
    }
    /// 上锁并获得可变部分的引用
    pub fn inner(&self) -> spin::MutexGuard<ProcessInner> {
        self.inner.lock()
    }
}
```

分配空间：

```rust
////os/src/process/process.rs: impl Process	
/// 分配一定数量的连续虚拟空间
///
/// 从 `memory_set` 中找到一段给定长度的未占用虚拟地址空间，分配物理页面并建立映射。返回对应的页面区间。
///
/// `flags` 只需包括 rwx 权限，user 位会根据进程而定。
pub fn alloc_page_range(
    &self,
    size: usize,
    flags: Flags,
) -> MemoryResult<Range<VirtualAddress>> {
    //创建一个内核进程，上锁并获得可变部分引用
    let memory_set = &mut self.inner().memory_set;

    // memory_set 只能按页分配，所以让 size 向上取整页
    let alloc_size = (size + PAGE_SIZE - 1) & !(PAGE_SIZE - 1);
    // 从 memory_set 中找一段不会发生重叠的空间
    let mut range = Range::<VirtualAddress>::from(0x1000000..0x1000000 + alloc_size);
    // 循环检测两个 [`Range`] 是否存在重合的区间
    while memory_set.overlap_with(range.into()) {
        range.start += alloc_size;
        range.end += alloc_size;
    }
    // 分配物理页面，建立映射
    memory_set.add_segment(
        Segment {
            map_type: MapType::Framed,
            range,
            flags: flags | Flags::user(self.is_user),
        },
        None,
    )?;
    // 返回地址区间（使用参数 size，而非向上取整的 alloc_size）
    Ok(Range::from(range.start..(range.start + size)))
}
```



> ### [高阶函数](https://rust-by-example.budshome.com/fn/hof.html#高阶函数)
>
> Rust 提供了高阶函数（Higher Order Function, HOF），指那些输入一个或多个 函数，并且/或者产生一个更有用的函数的函数。HOF 和惰性迭代器（lazy iterator）给 Rust 带来了函数式（functional）编程的风格。



### 执行第一个线程

因为启动线程需要修改各种寄存器的值，所以我们又要使用汇编了。不过，这一次我们只需要对 `interrupt.asm` 稍作修改就可以了。

在 `interrupt.asm` 中的 `__restore` 标签现在就能派上用途了。原本这段汇编代码的作用是将之前所保存的 `Context` 恢复到寄存器中，而现在我们让它使用一个精心设计的 `Context`，就可以让程序在恢复后直接进入我们的新线程。

首先我们稍作修改，添加一行 `mv sp, a0`。原本这里是读取之前存好的 `Context`，现在我们让其从 `a0` 中读取我们设计好的 `Context`。这样，我们可以直接在 Rust 代码中调用 `__restore(context)`。

```asm
#os/src/interrupt/interrupt.asm
__restore:
    mv      sp, a0  # 加入这一行
    # ...
```

我们只需要将`Context`应用到所有的寄存器上（即执行 `__restore`），就可以切换到第一个线程了。

```rust
//os/src/main.rs: rust_main()
extern "C" {
    fn __restore(context: usize);
}
// 获取第一个线程的 Context，具体原理后面讲解
let context = PROCESSOR.lock().prepare_next_thread();
// 启动第一个线程
unsafe { __restore(context as usize) };
unreachable!()
```



#### 为什么 `unreachable`

我们直接调用的 `__restore` 并没有 `ret` 指令，甚至 `ra` 都会被 `Context` 中的数值直接覆盖。这意味着，一旦我们执行了 `__restore(context)`，程序就无法返回到调用它的位置了。**注：直接 jump 是一个非常危险的操作**。

> 如果确定代码不可访问，则程序立即以终止[`panic!`](https://doc.rust-lang.org/std/macro.panic.html)。
>
> 该宏的不安全对应项是[`unreachable_unchecked`](https://doc.rust-lang.org/std/hint/fn.unreachable_unchecked.html)函数，如果到达代码，则将导致未定义的行为。

但是没有关系，我们也不需要这个函数返回。因为开始执行第一个线程，意味着操作系统的初始化已经完成，再回到 `rust_main()` 也没有意义了。甚至原本我们使用的栈 `bootstack`，也可以被回收（不过我们现在就丢掉不管吧）。



#### 在启动时不打开中断

现在，我们会在线程开始运行时开启中断，而在操作系统初始化的过程中是不应该有中断的。所以，我们删去之前设置「开启中断」的代码。

保留时钟中断会导致上下文切换时，线程还未搭建完成.

```rust
//os/interrupt/timer.rs
/// 初始化时钟中断
///
/// 开启时钟中断使能，并且预约第一次时钟中断
pub fn init() {
    unsafe {
        // 开启 STIE，允许时钟中断
        sie::set_stimer();
        // （删除）开启 SIE（不是 sie 寄存器），允许内核态被中断打断
        // sstatus::set_sie();
    }
    // 设置下一次时钟中断
    set_next_timeout();
}
```

### 小结

为了执行一个线程，我们需要初始化所有寄存器的值。为此，我们选择构建一个 `Context` 然后跳转至 `interrupt.asm` 中的 `__restore` 来执行，用这个 `Context` 来写入所有寄存器。



## 线程的切换

### 修改中断处理

在线程切换时（即时钟中断时），`handle_interrupt` 函数需要将上一个线程的 `Context` 保存起来，然后将下一个线程的 `Context` 恢复并返回。

> 注 1：为什么不直接 in-place 修改 `Context` 呢？
>
> 这是因为 `handle_interrupt` 函数返回的 `Context` 指针除了存储上下文以外，还提供了内核栈的地址。这个会在后面详细阐述。
>
> 注 2：在 Rust 中，引用 `&mut` 和指针 `*mut` 只是编译器的理解不同，其本质都是一个存储对象地址的寄存器。这里返回值使用指针而不是引用，是因为其指向的位置十分特殊，其生命周期在这里没有意义。

```rust
//os/src/interrupt/handler.rs
//在handler的开头添加使用（这句话不要加）
use crate::memory::*;
use crate::process::PROCESSOR;

/// 中断的处理入口
#[no_mangle]
pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) -> *mut Context {
    {
        let mut processor = PROCESSOR.lock();
        let current_thread = processor.current_thread();
        if current_thread.as_ref().inner().dead {
            println!("thread {} exit", current_thread.id);
            processor.kill_current_thread();
            return processor.prepare_next_thread();
        }
    }
    // 可以通过 Debug 来查看发生了什么中断
    // println!("{:x?}", context.scause.cause());
    match scause.cause() {
        // 断点中断（ebreak）
        Trap::Exception(Exception::Breakpoint) => breakpoint(context),
        // 时钟中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => supervisor_timer(context),
        // 其他情况，终止当前线程
        _ => fault("unimplemented interrupt type", scause, stval),
    }
}

/// 处理 ebreak 断点
fn breakpoint(context: &mut Context) -> *mut Context {
    println!("Breakpoint at 0x{:x}", context.sepc);
    context.sepc += 2;
    context
}

/// 处理时钟中断
fn supervisor_timer(context: &mut Context) -> *mut Context {
    timer::tick();
    PROCESSOR.lock().park_current_thread(context);
    PROCESSOR.lock().prepare_next_thread()
}

/// 出现未能解决的异常，终止当前线程
fn fault(msg: &str, scause: Scause, stval: usize) -> *mut Context {
    println!(
        "{:#x?} terminated: {}",
        PROCESSOR.lock().current_thread(),
        msg
    );
    println!("cause: {:?}, stval: {:x}", scause.cause(), stval);

    PROCESSOR.lock().kill_current_thread();
    // 跳转到 PROCESSOR 调度的下一个线程
    PROCESSOR.lock().prepare_next_thread()
}
```

可以看到，当发生断点中断时，直接返回原来的上下文（修改一下 `sepc`）；而如果是时钟中断的时候，我们执行了两个函数得到的返回值作为上下文，那它又是怎么工作的呢？

获取线程：

```rust
////os/src/process/processor.rs: impl Processor	
/// 获取一个当前线程的 `Arc` 引用
pub fn current_thread(&self) -> Arc<Thread> {
	self.current_thread.as_ref().unwrap().clone()
}
```

### 线程切换

让我们看一下 `Processor` 中的这两个方法是如何实现的。

（调度器 `scheduler` 会在后面的小节中讲解，我们只需要知道它能够返回下一个等待执行的线程。）

```rust
//os/src/process/processor.rs
impl Processor {
	/// 保存当前线程的 `Context`
    pub fn park_current_thread(&mut self, context: &Context) {
        self.current_thread().park(*context);
    }

    /// 在一个时钟中断时，替换掉 context
    pub fn prepare_next_thread(&mut self) -> *mut Context {
        // 向调度器询问下一个线程
        if let Some(next_thread) = self.scheduler.get_next() {
            // 准备下一个线程
            let context = next_thread.prepare();
            self.current_thread = Some(next_thread);
            context
        } else {
            // 没有活跃线程
            if self.sleeping_threads.is_empty() {
                // 也没有休眠线程，则退出
                panic!("all threads terminated, shutting down");
            } else {
                // 有休眠线程，则等待中断
                self.current_thread = Some(IDLE_THREAD.clone());
                IDLE_THREAD.prepare()
            }
        }
    }
    
    /// 令当前线程进入休眠
    pub fn sleep_current_thread(&mut self) {
        // 从 current_thread 中取出
        let current_thread = self.current_thread();
        // 记为 sleeping
        current_thread.inner().sleeping = true;
        // 从 scheduler 移出到 sleeping_threads 中
        self.scheduler.remove_thread(&current_thread);
        self.sleeping_threads.insert(current_thread);
    }
}
```

#### 上下文 `Context` 的保存和取出

在线程切换时，我们需要保存前一个线程的 `Context`，为此我们实现 `Thread::park` 函数。

```rust
//os/src/process/thread.rs: impl Thread
/// 发生时钟中断后暂停线程，保存状态
pub fn park(&self, context: Context) {
    // 检查目前线程内的 context 应当为 None
    assert!(self.inner().context.is_none());
    // 将 Context 保存到线程中
    self.inner().context.replace(context);
}
```

然后，我们需要取出下一个线程的 `Context`，为此我们实现 `Thread::prepare`。不过这次需要注意的是，启动一个线程除了需要 `Context`，还需要切换页表。这个操作我们也在这个方法中完成。

```rust
//os/src/process/thread.rs: impl Thread
/// 准备执行一个线程
///
/// 激活对应进程的页表，并返回其 Context
pub fn prepare(&self) -> *mut Context {
    // 激活页表
    self.process.inner().memory_set.activate();
    // 取出 Context
    let parked_frame = self.inner().context.take().unwrap();
    // 将 Context 放至内核栈顶
    unsafe { KERNEL_STACK.push_context(parked_frame) }
}
```

```rust
//os/src/process/thread.rs

use core::hash::{Hash, Hasher};

/// 通过线程 ID 来判等
///
/// 在 Rust 中，[`PartialEq`] trait 不要求任意对象 `a` 满足 `a == a`。
/// 将类型标注为 [`Eq`]，会沿用 `PartialEq` 中定义的 `eq()` 方法，
/// 同时声明对于任意对象 `a` 满足 `a == a`。
impl PartialEq for Thread {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}
impl Eq for Thread {}

/// 通过线程 ID 来哈希
impl Hash for Thread {
    fn hash<H: Hasher>(&self, state: &mut H) {
        state.write_isize(self.id);
    }
}

/// 打印线程除了父进程以外的信息
impl core::fmt::Debug for Thread {
    fn fmt(&self, formatter: &mut core::fmt::Formatter) -> core::fmt::Result {
        formatter
            .debug_struct("Thread")
            .field("thread_id", &self.id)
            .field("stack", &self.stack)
            .field("context", &self.inner().context)
            .finish()
    }
}
```

#### 内核栈？

现在，线程保存 `Context` 都是根据 `sp` 指针，在栈上压入一个 `Context` 来存储。但是，对于一个用户线程而言，它在用户态运行时用的是位于用户空间的用户栈。而它在用户态运行中如果触发中断，`sp` 指针指向的是用户空间的某地址，但此时 RISC-V CPU 会切换到内核态继续执行，就不能再用这个 `sp` 指针指向的用户空间地址了。这样，我们需要为 `sp` 指针准备好一个专门用于在内核态执行函数的内核栈。所以，为了不让一个线程的崩溃导致操作系统的崩溃，我们需要提前准备好内核栈，当线程发生中断时可用来存储线程的 `Context`。在下一节我们将具体讲解该如何做。



### 小结

为了实现线程的切换，我们让 `handle_interrupt` 返回一个 `*mut Context`。如果需要切换线程，就将前一个线程的 `Context` 保存起来换上新的线程的 `Context`。而如果不需要切换，那么直接返回原本的 `Context` 即可。



## 线程的结束

### 现有问题

当内核线程终止时，会发生什么？如果就按目前的实现，我们会发现线程所执行的函数末尾会触发 `Exception::InstructionPageFault` 而终止，其中访问的的地址 `stval = 0`。

这是因为内核线程在执行完 `entry_point` 所指向的函数后会返回到 `ra` 指向的地址，而我们没有为其赋初值（初值为 0）。此时，程序就会尝试跳转到 `0x0` 地址，而显然它是不存在的。



### 解决办法

很自然的，我们希望能够让内核线程在结束时触发一个友善的中断（而不是一个看上去像是错误的缺页异常），然后被操作系统释放。我们可能会想到系统调用，但很可惜我们无法使用它，因为系统调用的本质是一个环境调用 `ecall`，而在内核线程（内核态）中进行的环境调用是用来与 M 态通信的。我们之前实现的 SBI 调用就是使用的 S 态 `ecall`。

因此，我们设计一个折中的解决办法：内核线程将自己标记为“已结束”，同时触发一个普通的异常 `ebreak`。此时操作系统观察到线程的标记，便将其终止。

```rust
//os/src/main.rs
/// 内核线程需要调用这个函数来退出
fn kernel_thread_exit() {
    // 当前线程标记为结束
    PROCESSOR.lock().current_thread().as_ref().inner().dead = true;
    // 制造一个中断来交给操作系统处理
    unsafe { llvm_asm!("ebreak" :::: "volatile") };
}
```

然后，我们将这个函数作为内核线程的 `ra`，使得它执行的函数完成后便执行 `kernel_thread_exit()`

```rust
//os/src/main.rs
/// 创建一个内核进程
pub fn create_kernel_thread(
    process: Arc<Process>,
    entry_point: usize,
    arguments: Option<&[usize]>,
) -> Arc<Thread> {
    // 创建线程
    let thread = Thread::new(process, entry_point, arguments).unwrap();
    // 设置线程的返回地址为 kernel_thread_exit
    thread.as_ref().inner().context.as_mut().unwrap()
        .set_ra(kernel_thread_exit as usize);
    thread
}
```

处理器结束函数：

```rust
//os/src/process/processor.rs: impl Processor
/// 终止当前的线程
pub fn kill_current_thread(&mut self) {
    // 从调度器中移除
    let thread = self.current_thread.take().unwrap();
    self.scheduler.remove_thread(&thread);
}
```



## 内核栈

### 为什么 / 怎么做

在实现内核栈之前，让我们先检查一下需求和我们的解决办法。

- **不是每个线程都需要一个独立的内核栈**，因为内核栈只会在中断时使用，而中断结束后就不再使用。在只有一个 CPU 的情况下，不会有两个线程同时出现中断，**所以我们只需要实现一个共用的内核栈就可以了**。
- **每个线程都需要能够在中断时第一时间找到内核栈的地址**。这时，所有通用寄存器的值都无法预知，也无法从某个变量来加载地址。为此，**我们将内核栈的地址存放到内核态使用的特权寄存器 `sscratch` 中**。这个寄存器只能在内核态访问，这样在中断发生时，就可以安全地找到内核栈了。

因此，我们的做法就是：

- 预留一段空间作为内核栈
- 运行线程时，在 `sscratch` 寄存器中保存内核栈指针
- 如果线程遇到中断，则从将 `Context` 压入 `sscratch` 指向的栈中（`Context` 的地址为 `sscratch - size_of::<Context>()`），同时用新的栈地址来替换 `sp`（此时 `sp` 也会被复制到 `a0` 作为 `handle_interrupt` 的参数）
- 从中断中返回时（`__restore` 时），`a0` 应指向**被压在内核栈中的 `Context`**。此时出栈 `Context` 并且将栈顶保存到 `sscratch` 中

+

### 实现

#### 为内核栈预留空间

我们直接使用一个 `static mut` 来指定一段空间作为栈。

```rust
//os/src/process/kernel_stack.rs
//! 内核栈 [`KernelStack`]
//!
//! 用户态的线程出现中断时，因为用户栈无法保证可用性，中断处理流程必须在内核栈上进行。
//! 所以我们创建一个公用的内核栈，即当发生中断时，会将 Context 写到内核栈顶。
//!
//! ### 线程 [`Context`] 的存放
//! > 1. 线程初始化时，一个 `Context` 放置在内核栈顶，`sp` 指向 `Context` 的位置
//! >   （即栈顶 - `size_of::<Context>()`）
//! > 2. 切换到线程，执行 `__restore` 时，将 `Context` 的数据恢复到寄存器中后，
//! >   会将 `Context` 出栈（即 `sp += size_of::<Context>()`），
//! >   然后保存 `sp` 至 `sscratch`（此时 `sscratch` 即为内核栈顶）
//! > 3. 发生中断时，将 `sscratch` 和 `sp` 互换，入栈一个 `Context` 并保存数据
//!
//! 容易发现，线程的 `Context` 一定保存在内核栈顶。因此，当线程需要运行时，
//! 从 [`Thread`] 中取出 `Context` 然后置于内核栈顶即可

use super::*;
use core::mem::size_of;

/// 内核栈
#[repr(align(16))]
#[repr(C)]
pub struct KernelStack([u8; KERNEL_STACK_SIZE]);

/// 公用的内核栈
pub static mut KERNEL_STACK: KernelStack = KernelStack([0; KERNEL_STACK_SIZE]);
```

对`KERNEL_STACK_SIZE`给出定义：

```rust
//os/src/process/config.rs
//! 定义一些进程相关的常量

/// 每个线程的运行栈大小 512 KB
pub const STACK_SIZE: usize = 0x8_0000;

/// 共用的内核栈大小 512 KB
pub const KERNEL_STACK_SIZE: usize = 0x8_0000;
```

在我们创建线程时，需要使用的操作就是在内核栈顶压入一个初始状态 `Context`：

```rust
//os/src/process/kernel_stack.rs
impl KernelStack {
    /// 在栈顶加入 Context 并且返回新的栈顶指针
    pub fn push_context(&mut self, context: Context) -> *mut Context {
        // 栈顶
        let stack_top = &self.0 as *const _ as usize + size_of::<Self>();
        // Context 的位置
        let push_address = (stack_top - size_of::<Context>()) as *mut Context;
        unsafe {
            *push_address = context;
        }
        push_address
    }
}
```

#### 修改 `interrupt.asm`

在这个汇编代码中，我们需要加入对 `sscratch` 的判断和使用。

```asm
#os/src/interrput/interrupt.asm
__interrupt:
    # 因为线程当前的栈不一定可用，必须切换到内核栈来保存 Context 并进行中断流程
    # 因此，我们使用 sscratch 寄存器保存内核栈地址
    # 思考：sscratch 的值最初是在什么地方写入的？

    # 交换 sp 和 sscratch（切换到内核栈）
    csrrw   sp, sscratch, sp
    # 在内核栈开辟 Context 的空间
    addi    sp, sp, -36*8

    # 保存通用寄存器，除了 x0（固定为 0）
    SAVE    x1, 1
    # 将本来的栈地址 sp（即 x2）保存
    csrr    x1, sscratch
    SAVE    x1, 2

    # ...
```

以及事后的恢复：

```asm
# os/src/interrupt/interrupt.asm
# 离开中断
# 此时内核栈顶被推入了一个 Context，而 a0 指向它
# 接下来从 Context 中恢复所有寄存器，并将 Context 出栈（用 sscratch 记录内核栈地址）
# 最后跳转至恢复的 sepc 的位置
__restore:
    # 从 a0 中读取 sp
    # 思考：a0 是在哪里被赋值的？（有两种情况）
    mv      sp, a0
    # 恢复 CSR
    LOAD    t0, 32
    LOAD    t1, 33
    csrw    sstatus, t0
    csrw    sepc, t1
    # 将内核栈地址写入 sscratch
    addi    t0, sp, 36*8
    csrw    sscratch, t0

    # 恢复通用寄存器
    # ...
```

对上述所有功能的完成进行声明：

```rust
//os/src/process/mod.rs
//! 管理进程 / 线程

mod config;
mod kernel_stack;
mod lock;
#[allow(clippy::module_inception)]
mod process;
mod processor;
mod thread;

use crate::interrupt::*;
use crate::memory::*;
use alloc::{sync::Arc, vec, vec::Vec};
use spin::Mutex;

pub use config::*;
pub use kernel_stack::KERNEL_STACK;
pub use lock::Lock;
pub use process::Process;
pub use processor::PROCESSOR;
pub use thread::Thread;
```



### 小结

为了能够鲁棒地处理用户线程产生的异常，我们为线程准备好一个内核栈，发生中断时会切换到这里继续处理。

> **鲁棒性**（`robustness`）
>
> 指在异常下，系统仍然能够较为平稳的运行（我自己的理解，你再改改）



## 调度器

调度器的算法有许多种，我们将它提取出一个 trait 作为接口

先入先出队列的调度器：

```rust
//os/src/algorithm/src/scheduler/fifo_scheduler.rs
//! 先入先出队列的调度器 [`FifoScheduler`]

use super::Scheduler;
use alloc::collections::LinkedList;

/// 采用 FIFO 算法的线程调度器
pub struct FifoScheduler<ThreadType: Clone + Eq> {
    pool: LinkedList<ThreadType>,
}

/// `Default` 创建一个空的调度器
impl<ThreadType: Clone + Eq> Default for FifoScheduler<ThreadType> {
    fn default() -> Self {
        Self {
            pool: LinkedList::new(),
        }
    }
}

impl<ThreadType: Clone + Eq> Scheduler<ThreadType> for FifoScheduler<ThreadType> {
    type Priority = ();
    fn add_thread(&mut self, thread: ThreadType) {
        // 加入链表尾部
        self.pool.push_back(thread);
    }
    fn get_next(&mut self) -> Option<ThreadType> {
        // 从头部取出放回尾部，同时将其返回
        if let Some(thread) = self.pool.pop_front() {
            self.pool.push_back(thread.clone());
            Some(thread)
        } else {
            None
        }
    }
    fn remove_thread(&mut self, thread: &ThreadType) {
        // 移除相应的线程并且确认恰移除一个线程
        let mut removed = self.pool.drain_filter(|t| t == thread);
        assert!(removed.next().is_some() && removed.next().is_none());
    }
    fn set_priority(&mut self, _thread: ThreadType, _priority: ()) {}
}
```

> ## FIFO调度算法
>
> 1）概念：按照进程进入系统的先后次序进行调度，先进入系统者先调度；即启动等待时间最长的作业/进程。是一种最简单的调度算法，即可用于作业调度，也可用于进程调度
> ￼
> 2）优缺点
>
> * 比较有利于长作业（进程），而不利于短作业（进程）
> * 有利于CPU繁忙型作业（进程） ，而不利于I/O繁忙型作业（进程）
> * 用于批处理系统，不适于分时系统

最高响应比优先算法的调度器：

```rust
//os/src/algorithm/src/scheduler/hrrn_scheduler.rs
//! 最高响应比优先算法的调度器 [`HrrnScheduler`]

use super::Scheduler;
use alloc::collections::LinkedList;

/// 将线程和调度信息打包
struct HrrnThread<ThreadType: Clone + Eq> {
    /// 进入线程池时，[`current_time`] 中的时间
    birth_time: usize,
    /// 被分配时间片的次数
    service_count: usize,
    /// 线程数据
    pub thread: ThreadType,
}

/// 采用 HRRN（最高响应比优先算法）的调度器
pub struct HrrnScheduler<ThreadType: Clone + Eq> {
    /// 当前时间，单位为 `get_next()` 调用次数
    current_time: usize,
    /// 带有调度信息的线程池
    pool: LinkedList<HrrnThread<ThreadType>>,
}

/// `Default` 创建一个空的调度器
impl<ThreadType: Clone + Eq> Default for HrrnScheduler<ThreadType> {
    fn default() -> Self {
        Self {
            current_time: 0,
            pool: LinkedList::new(),
        }
    }
}

impl<ThreadType: Clone + Eq> Scheduler<ThreadType> for HrrnScheduler<ThreadType> {
    type Priority = ();

    fn add_thread(&mut self, thread: ThreadType) {
        self.pool.push_back(HrrnThread {
            birth_time: self.current_time,
            service_count: 0,
            thread,
        })
    }
    fn get_next(&mut self) -> Option<ThreadType> {
        // 计时
        self.current_time += 1;

        // 遍历线程池，返回响应比最高者
        let current_time = self.current_time; // borrow-check
        if let Some(best) = self.pool.iter_mut().max_by(|x, y| {
            ((current_time - x.birth_time) * y.service_count)
                .cmp(&((current_time - y.birth_time) * x.service_count))
        }) {
            best.service_count += 1;
            Some(best.thread.clone())
        } else {
            None
        }
    }
    fn remove_thread(&mut self, thread: &ThreadType) {
        // 移除相应的线程并且确认恰移除一个线程
        let mut removed = self.pool.drain_filter(|t| t.thread == *thread);
        assert!(removed.next().is_some() && removed.next().is_none());
    }
    fn set_priority(&mut self, _thread: ThreadType, _priority: ()) {}
}
```

> ## 高响应比优先(HRRN, Highest Response Ratio Next)算法
>
> 在每次调度时先计算各个进程的响应比，选择响应比最高的进程为其服务
>
> 响应比RP公式
> $$
> R_p = \frac{T_w + T_R}{T_w} = 1 + \frac{T_w}{T_R}
> $$
> Tw为等待时间，TR为服务时间。
>
> 从上式可以看出:
>
> 1. 等待时间相同，则短作业优先权高，有利于短作业。
> 2. 服务时间相同，等待时间越长，其优先权越高，相当于先来先服务。
> 3. 服务时间相对较长的作业，当其等待足够长时，便可获得处理机运行。

声明：

```rust
//os/src/algorithm/src/scheduler/mod.rs
//! 线程调度算法

mod fifo_scheduler;
mod hrrn_scheduler;

/// 线程调度器
///
/// `ThreadType` 应为 `Arc<Thread>`
///
/// ### 使用方法
/// - 在每一个时间片结束后，调用 [`Scheduler::get_next()`] 来获取下一个时间片应当执行的线程。
///   这个线程可能是上一个时间片所执行的线程。
/// - 当一个线程结束时，需要调用 [`Scheduler::remove_thread()`] 来将其移除。这个方法必须在
///   [`Scheduler::get_next()`] 之前调用。
pub trait Scheduler<ThreadType: Clone + Eq>: Default {
    /// 优先级的类型
    type Priority;
    /// 向线程池中添加一个线程
    fn add_thread(&mut self, thread: ThreadType);
    /// 获取下一个时间段应当执行的线程
    fn get_next(&mut self) -> Option<ThreadType>;
    /// 移除一个线程
    fn remove_thread(&mut self, thread: &ThreadType);
    /// 设置线程的优先级
    fn set_priority(&mut self, thread: ThreadType, priority: Self::Priority);
}

pub use fifo_scheduler::FifoScheduler;
pub use hrrn_scheduler::HrrnScheduler;

pub type SchedulerImpl<T> = HrrnScheduler<T>;
```

在lib中添加：

```rust
//os/src/algorithm/src/lib.rs
//! 一些可能用到，而又不好找库的数据结构
//!
//! 以及有多种实现，会留作业的数据结构
#![no_std]
#![feature(drain_filter)]

extern crate alloc;

mod allocator;
mod scheduler;

pub use allocator::*;
pub use scheduler::*;
```

### 运行！

修改 `main.rs`，我们就可以跑起来多线程了。

```rust
//os/src/main.rs
//在前面添加使用
use alloc::sync::Arc;
use memory::PhysicalAddress;
use process::*;


pub extern "C" fn rust_main() -> ! {
    memory::init();
    interrupt::init();

    {
        let mut processor = PROCESSOR.lock();
        // 创建一个内核进程
        let kernel_process = Process::new_kernel().unwrap();
        // 为这个进程创建多个线程，并设置入口均为 sample_process，而参数不同
        for i in 1..9usize {
            processor.add_thread(create_kernel_thread(
                kernel_process.clone(),
                sample_process as usize,
                Some(&[i]),
            ));
        }
    }

    extern "C" {
        fn __restore(context: usize);
    }
    // 获取第一个线程的 Context
    let context = PROCESSOR.lock().prepare_next_thread();
    // 启动第一个线程
    unsafe { __restore(context as usize) };
    unreachable!()
}

//线程入口程序
fn sample_process(id: usize) {
    println!("hello from kernel thread {}", id);
}
```

运行一下，我们会得到类似的输出：

```
hello from kernel thread 7
thread 7 exit
hello from kernel thread 6
thread 6 exit
hello from kernel thread 5
thread 5 exit
hello from kernel thread 4
thread 4 exit
hello from kernel thread 3
thread 3 exit
hello from kernel thread 2
thread 2 exit
hello from kernel thread 1
thread 1 exit
hello from kernel thread 8
thread 8 exit
src/process/processor.rs:87: 'all threads terminated, shutting down'
```

本章我们的工作有：

- 理清线程和进程的概念
- 通过设置 `Context`，可以构造一个线程的初始状态
- 通过 `__restore` 标签，直接进入第一个线程之中
- 用 `Context` 来保存进程的状态，从而实现在时钟中断时切换线程
- 实现内核栈，提供安全的中断处理空间
- 实现调度器，完成线程的调度

同时，可以发现我们这一章的内容集中在内核线程上面，对用户进程还没有过多的提及。而为了让用户进程可以在我们的系统上运行起来，一个优美的做法将会是隔开用户程序和内核。需要注意到现在的内核还直接放在了内存上，在下一个章节，我们暂时跳过用户进程，实现可以放置用户数据的文件系统。

本章的完整代码你可以在仓库`代码\lab4`中找到