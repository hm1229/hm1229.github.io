# 实验步骤

对于用户程序而言，中断的处理应当是不留任何痕迹的：只要中断处理改动了一个寄存器，都可能导致原本正在运行的线程出现错误。因此，在处理中断之前，必须要保存所有可能被修改的寄存器，并且在处理完成后恢复。因此，我们需要保存所有通用寄存器，`sepc`、`scause` 和 `stval` 这三个会被硬件自动写入的 CSR 寄存器，以及 `sstatus`。因为中断可能会涉及到权限的切换，以及中断的开关，这些都会修改 `sstatus`。

## 保存进程上下文

从执行中的程序的角度来讲，该程序不应该感知到操作系统发生了中断。这需要在中断前后维持系统的上下文不变。

创造一个结构体变量Context，使其来存储进程上下文所用到的寄存器信息。但只存储信息还不够，还需要对信息进行保存与恢复。由于涉及到寄存器的操作，由于不能用Rust语言直接完成，在这里将使用汇编语言，将寄存器的值存储在Context中，如图。

![image-20210530231136855](file://F:\rCore\rcore-tutorial-detail\资源文件\实验一.assets\image-20210531141510157.png?lastModify=1629443459)

上文已经成功存储了中断时的上下文信息，但为了能够主动的处理中断，还应编写一个负责识别和处理中断的函数。使用汇编将Context当部分信息，取代的是将堆栈指针指向由进程处理程序所使用的临时堆栈

3）对进程中断所进行的操作均由进程处理程序负责处理，作为操作系统的服务例程，可以为所有的进程中断使用

4）例程结束后，调用特定过程完成剩下的操作。

同时中断分为异常、陷阱、硬件中断。异常是执行指令时产生的无法预料的错误，如无效的内存访问或者非法的指令等。陷阱是一种强行导致中断的指令，如系统调用。硬件中断是由CPU之外的硬件产生的异步中断，比如时钟中断，外设中断等。

如何判断中断的类型，并对不同的中断进行相应的响应是目前需要解决的问题。

## 恢复进程上下文

现在已经完成了中断的处理，需要恢复上下文为中断前的样子。

如果进程中断返回后，处于就绪状态，则调用进程调度程序，决定下阶段调度哪个进程运行；如果调度该进程处于运行状态，则计算机将控制转交给一段例程，由该例程为当前可以运行的进程装入寄存器值，并实现对内存的映射，启动该进程运行；如果进程中断返回后，仍需要等待其他底层服务，则执行到该服务调用处，继续重复中断过程。

这里还是使用汇编语言，将上下文中的信息全部恢复至计算器上，并转跳至consent中的位置。

### Context-实现进程上下文的保存

我们把在中断时保存了各种寄存器的结构体叫做 `Context`，他表示原来程序正在执行所在的上下文（这个概念在后面线程的部分还会用到），这里我们和 `scause` 以及 `stval` 作为一个区分，后两者将不会放在 `Context` 而仅仅被看做一个临时的变量（在后面会被用到），`Context` 的定义如下：

```rust
// os/src/interrupt/context.rs
use riscv::register::{sstatus::Sstatus, scause::Scause};

#[repr(C)]				// repr(C)来确保在 Rust 中，该结构体的内存布局与在 C 中相同。
#[derive(Debug)]		    //使结构体可以打印
pub struct Context {
    pub x: [usize; 32],     // 32 个通用寄存器
    pub sstatus: Sstatus,	// 系统状态
    pub sepc: usize			//触发中断的指令地址
}
```

> **repr(Rust)**
> 首先，每种类型都有一个数据对齐属性 (alignment)。一种类型的对齐属性决定了哪些内存地址可以合法地存储该类型的值。如果对齐属性是 n，那么它的值的存储地址必须是 n 的倍数。所以，对齐属性 2 表示值只能存储在偶数地址里，1 表示值可以存储在任何的地方。对齐属性最小为 1，并且永远是 2 的整数次幂。虽然不同平台的行为可能会不同，但大部分情况下基础类型都是按照它的类型大小对齐的。特别的是，在 x86 平台上 u64 和 f64 都是按照 32 位对齐的。
>
> 一种类型的大小都是它对齐属性的整数倍，这保证了这种类型的值在数组中的偏移量都是其类型尺寸的整数倍，可以按照偏移量进行索引。需要注意的是，动态尺寸类型的大小和对齐可能无法静态获取。
>
> **repr(C)**
> 这是最重要的一种 repr。它的目的很简单，就是和 C 保持一致。数据的顺序、大小、对齐方式都和你在 C 或 C++ 中见到的一摸一样。所有你需要通过 FFI 交互的类型都应该有 repr(C)，因为 C 是程序设计领域的世界语。而且如果我们要在数据布局方面玩一些花活的话，比如把数据重新解析成另一种类型，repr(C) 也是很有必要的。

这里我们使用了 rCore 中的库 riscv 封装的一些寄存器操作，需要在 `os/Cargo.toml` 中添加依赖。

```toml
# /os/Cargo.toml
[dependencies]
riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] }
```

> 如果这个依赖下载缓慢的话，尝试在`.cargo/config`中添加：
>
> ```toml
> [source.crates-io]
> registry = "https://github.com/rust-lang/crates.io-index"
> replace-with = 'ustc'
> [source.ustc]
> registry = "git://mirrors.ustc.edu.cn/crates.io-index"
> ```
>
> 

## 状态的保存与恢复

### 操作流程

为了状态的保存与恢复，我们可以先用栈上的一小段空间来把需要保存的全部通用寄存器和 CSR 寄存器保存在栈上，保存完之后在跳转到 Rust 编写的中断处理函数；而对于恢复，则直接把备份在栈上的内容写回寄存器。由于涉及到了寄存器级别的操作，我们需要用汇编来实现。

而对于如何保存在栈上，我们可以直接令 `sp` 栈寄存器直接减去相应需要开辟的大小，然后依次放在栈上。需要注意的是，`sp` 寄存器又名 `x2`，我们需要不断用到这个寄存器告诉 CPU 其他寄存器放在哪个地址，所以处理这个 `sp` 寄存器本身的保存时也需要格外小心。



### 编写汇编

因为汇编代码较长，这里我们新建一个 `os/src/interrupt/interrupt.asm` 文件来编写这段操作：

```asm
# os/src/interrupt/interrupt.asm
# 我们将会用一个宏来用循环保存寄存器。这是必要的设置
.altmacro
# 寄存器宽度对应的字节数
.set    REG_SIZE, 8
# Context 的大小
.set    CONTEXT_SIZE, 34

# 宏：将寄存器存到栈上
.macro SAVE reg, offset
    sd  \reg, \offset*8(sp)
.endm

.macro SAVE_N n
    SAVE  x\n, \n		#\n使将n转义为n的值，其它同理
.endm


# 宏：将寄存器从栈中取出
.macro LOAD reg, offset
    ld  \reg, \offset*8(sp)
.endm

.macro LOAD_N n
    LOAD  x\n, \n     
.endm

    .section .text
    .globl __interrupt
# 进入中断
# 保存 Context 并且进入 Rust 中的中断处理函数 interrupt::handler::handle_interrupt()
__interrupt:
    # 在栈上开辟 Context 所需的空间
    addi    sp, sp, -34*8

    # 保存通用寄存器，除了 x0（固定为 0）
    SAVE    x1, 1
    # 将原来的 sp（sp 又名 x2）写入 2 位置
    addi    x1, sp, 34*8
    SAVE    x1, 2
    # 保存 x3 至 x31
    .set    n, 3
    .rept   29
        SAVE_N  %n
        .set    n, n + 1
    .endr

    # 取出 CSR 并保存
    csrr    s1, sstatus
    csrr    s2, sepc
    SAVE    s1, 32
    SAVE    s2, 33

    # 调用 handle_interrupt，传入参数
    # context: &mut Context
    mv      a0, sp
    # scause: Scause
    csrr    a1, scause
    # stval: usize
    csrr    a2, stval
    jal  handle_interrupt

    .globl __restore
# 离开中断
# 从 Context 中恢复所有寄存器，并跳转至 Context 中 sepc 的位置
__restore:
    # 恢复 CSR
    LOAD    s1, 32
    LOAD    s2, 33
    csrw    sstatus, s1
    csrw    sepc, s2

    # 恢复通用寄存器
    LOAD    x1, 1
    # 恢复 x3 至 x31
    .set    n, 3
    .rept   29
        LOAD_N  %n
        .set    n, n + 1
    .endr

    # 恢复 sp（又名 x2）这里最后恢复是为了上面可以正常使用 LOAD 宏
    LOAD    x2, 2
    sret
```

这样的话我们就完成了对当前执行现场保存，我们把 `Context` 以及 `scause` 和 `stval` 作为参数传入了 `handle_interrupt` 函数中，这是一个 Rust 编写的函数，后面我们将会实现它。

[汇编宏的定义以及相关语法参考](http://c.biancheng.net/view/3714.html)

RISC-V 和 GNU 使用`.`作为保留字前缀

[`.section`的作用]

> ### 汇编语法**ld** 
>
> ld rd, offset(rs1) 
>
> x[rd] = M[x[rs1] + (sext(offset)] [63:0]) 双字加载 (Load Doubleword). I-type, RV32I and RV64I. 从地址 x[rs1] + sign-extend(offset)读取八个字节，写入 x[rd]。
>
> ### 汇编语法sd
>
> sd rs2, offset(rs1) 
>
> x[rs1] + sext(offset) = x [rs2] [63: 0] 存双字(Store Doubleword). S-type, RV64I only. 将 x[rs2]中的 8 字节存入内存地址 x[rs1]+sign-extend(offset)。

这样的话我们就完成了对当前执行现场保存，我们把 `Context` 以及 `scause` 和 `stval` 作为参数传入了 `handle_interrupt` 函数中，这是一个 Rust 编写的函数，后面我们将会实现它。

> [汇编宏的定义以及相关语法参考](http://c.biancheng.net/view/3714.html)
>
> RISC-V 和 GNU 使用`.`作为保留字前缀
>
> [`.section`的作用](http://blog.chinaunix.net/uid-28458801-id-3554757.html)



## 实现一个陷入处理流程

接下来，我们将要手动触发一个 Trap（`ebreak`），并且进入中断处理流程。



### 初始化相关寄存器

为了让硬件能够找到我们编写的 `__interrupt` 入口，在操作系统初始化时，需要将其写入 `stvec` 寄存器中：

> #### 设置内核态中断响应的入口寄存器
>
> - `stvec`
>   设置内核态中断处理流程的入口地址。存储了一个基址 BASE 和模式 MODE：
>   - MODE 为 0 表示 Direct 模式，即遇到中断便跳转至 BASE 进行执行。
>   - MODE 为 1 表示 Vectored 模式，此时 BASE 应当指向一个向量，存有不同处理流程的地址，遇到中断会跳转至 `BASE + 4 * cause` 进行处理流程。

```rust
/// os/src/interrupt/handler.rs
use super::context::Context;
use riscv::register::{
    scause::{Exception, Interrupt, Scause, Trap},
    stvec,
};

global_asm!(include_str!("./interrupt.asm"));

/// 初始化中断处理
///
/// 把中断入口 `__interrupt` 写入 `stvec` 中，并且开启中断使能
pub fn init() {
    unsafe {
        extern "C" {
            /// `interrupt.asm` 中的中断入口
            fn __interrupt();
        }
        // 使用 Direct 模式，将中断入口设置为 `__interrupt`
        stvec::write(__interrupt as usize, stvec::TrapMode::Direct);
    }
}
```

> ## **`unsafe`**
>
> 目前为止讨论过的代码都有 Rust 在编译时会强制执行的内存安全保证。然而，Rust 还隐藏有第二种语言，它不会强制执行这类内存安全保证：这被称为 **不安全 Rust**（*unsafe Rust*）。它与常规 Rust 代码无异，但是会提供额外的超级力量。
>
> 不安全 Rust 之所以存在，是因为静态分析本质上是保守的。当编译器尝试确定一段代码是否支持某个保证时，拒绝一些有效的程序比接受无效程序要好一些。这必然意味着有时代码可能是合法的，但是 Rust 不这么认为！在这种情况下，可以使用不安全代码告诉编译器，“相信我，我知道我在干什么。”这么做的缺点就是你只能靠自己了：如果不安全代码出错了，比如解引用空指针，可能会导致不安全的内存使用。
>
> 另一个 Rust 存在不安全一面的原因是：底层计算机硬件固有的不安全性。如果 Rust 不允许进行不安全操作，那么有些任务则根本完成不了。Rust 需要能够进行像直接与操作系统交互，甚至于编写你自己的操作系统这样的底层系统编程！这也是 Rust 语言的目标之一。

### 处理中断

然后，我们再补上 `__interrupt` 后跳转的中断处理流程 `handle_interrupt()`：

```rust
/// os/src/interrupt/handler.rs
/// 中断的处理入口
/// 
/// `interrupt.asm` 首先保存寄存器至 Context，其作为参数和 scause 以及 stval 一并传入此函数
/// 具体的中断类型需要根据 scause 来推断，然后分别处理
/// 目前我们能做的只有panic！
#[no_mangle]
pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) {
    panic!("Interrupted: {:?}", scause.cause());
}
```

> ### 完整代码
>
> ```rust
> /// os/src/interrupt/handler.rs
> use super::context::Context;
> use riscv::register::{
> 	scause::{Exception, Interrupt, Scause, Trap},
> 	stvec,
> };
> 
> global_asm!(include_str!("./interrupt.asm"));
> 
> /// 初始化中断处理
> ///
> /// 把中断入口 `__interrupt` 写入 `stvec` 中，并且开启中断使能
> pub fn init() {
>   unsafe {
>    extern "C" {
>           /// `interrupt.asm` 中的中断入口
>        fn __interrupt();
>   }
>    // 使用 Direct 模式，将中断入口设置为 `__interrupt`
>   stvec::write(__interrupt as usize, stvec::TrapMode::Direct);
> 	}
> }
> 
> /// 中断的处理入口
> ///
> /// `interrupt.asm` 首先保存寄存器至 Context，其作为参数和 scause 以及 stval 一并传入此函数
> /// 具体的中断类型需要根据 scause 来推断，然后分别处理
> /// 目前我们能做的只有panic！
> #[no_mangle]
> pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) {
> 	panic!("Interrupted: {:?}", scause.cause());
> }
> ```



### 触发中断

最后，我们把刚刚写的函数声明一下：

```rust
//! os/src/interrupt/mod.rs
//! 中断模块
//! 
//! 

mod handler;
mod context;

pub use context::Context;

/// 初始化中断相关的子模块
/// 
/// - [`handler::init`]
/// - [`timer::init`]
pub fn init() {
    handler::init();    //调用中断处理初始化函数
    println!("中断处理初始化成功");
}
```

同时，我们在 main 函数中主动使用 `ebreak` 来触发一个中断。

```rust
// /os/src/main.rs
...
mod interrupt;
...

/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
pub extern "C" fn rust_main() -> ! {
    // 初始化各种模块
    interrupt::init();          //初始化中断处理模块

    unsafe {
        llvm_asm!("ebreak"::::"volatile");   //触发中断
    };

    println!("Hello rCore-Tutorial!");
    
    panic!("end of rust_main")
}
```

运行一下，可以看到 `ebreak` 导致程序进入了中断处理并退出，而没有执行到后面的 `unreachable!()`：

运行输出

```
Hello rCore-Tutorial!
mod interrupt initialized
panic: 'Interrupted: Exception(Breakpoint)
```



## 时钟中断

本章的最后，我们来实现操作系统中极其重要的时钟中断。时钟中断可以把一些例行的及需要定时执行的程序放在时钟中断中，还可以利用时钟中断协助主程序完成定时、延时等操作，是操作系统能够进行线程调度的基础，操作系统会在每次时钟中断时被唤醒，暂停正在执行的线程，并根据调度算法选择下一个应当运行的线程。

>  **RISC-V 中断寄存器的细分**
>
>  在前面提到，`sie` 和 `sip` 寄存器分别保存不同中断种类的使能和触发记录。例如，软件中断的使能是 `sie` 中的 SSIE 位，触发记录是 `sip` 中的 SSIP 位。
>
>  RISC-V 中将中断分为三种：
>
>  - 软件中断（Software Interrupt），对应 SSIE 和 SSIP
>  - 时钟中断（Timer Interrupt），对应 STIE 和 STIP
>  - 外部中断（External Interrupt），对应 SEIE 和 SEIP

> `sie` 即 Supervisor Interrupt Enable，用来控制具体类型中断的使能，例如其中的 STIE 控制时钟中断使能。
>
> `sip` 即 Supervisor Interrupt Pending，和 `sie` 相对应，记录每种中断是否被触发。仅当 `sie` 和 `sip` 的对应位都为 1 时，意味着开中断且已发生中断，这时中断最终触发。



### 开启时钟中断

时钟中断也需要我们在初始化操作系统时开启，我们同样只需使用 riscv 库中提供的接口即可。

```rust
//! os/src/interrupt/timer.rs
//! 预约和处理时钟中断

use crate::sbi::set_timer;
use riscv::register::{time, sie, sstatus};

/// 初始化时钟中断
/// 
/// 开启时钟中断使能，并且预约第一次时钟中断
pub fn init() {
    unsafe {
        // 开启 STIE，允许时钟中断
        sie::set_stimer(); 
        // 开启 SIE（不是 sie 寄存器），允许内核态被中断打断
        sstatus::set_sie();
    }
    // 设置下一次时钟中断
    set_next_timeout();
}
```

这里可能引起误解的是 `sstatus::set_sie()`，它的作用是开启 `sstatus` 寄存器中的 SIE 位，与 `sie` 寄存器无关。SIE 位决定中断是否能够打断 **supervisor 线程**。在这里我们需要允许时钟中断打断**内核态线程**，因此置 SIE 位为 1。
另外，无论 SIE 位为什么值，中断都可以打断**用户态的线程**。

### 设置时钟中断

每一次的时钟中断都需要操作系统设置一个下一次中断的时间，这样硬件会在指定的时间发出时钟中断。为简化操作系统实现，操作系统可请求（`sbi_call` 调用 `ecall` 指令）SBI 服务来完成时钟中断的设置。OpenSBI 固件在接到 SBI 服务请求后，会帮助 OS 设置下一次要触发时钟中断的时间，CPU 在执行过程中会检查当前的时间间隔是否已经超过设置的时钟中断时间间隔，如果超时则会触发时钟中断。

```rust
/// os/src/sbi.rs
/// 设置下一次时钟中断的时间
pub fn set_timer(time: usize) {
    sbi_call(SBI_SET_TIMER, time, 0, 0);
}
```



为了便于后续处理，我们设置时钟间隔为 100000 个 CPU 周期。越短的间隔可以让 CPU 调度资源更加细致，但同时也会导致更多资源浪费在操作系统上。

```rust
/// os/src/interrupt/timer.rs
/// 时钟中断的间隔，单位是 CPU 指令
static INTERVAL: usize = 100000;

/// 设置下一次时钟中断
/// 
/// 获取当前时间，加上中断间隔，通过 SBI 调用预约下一次中断
fn set_next_timeout() {
    set_timer(time::read() + INTERVAL);
}
```

> ### CPU周期
>
> 机器周期也称为CPU周期。在计算机中，为了便于管理，常把一条指令的执行过程划分为若干个阶段（如取指、译码、执行等），每一阶段完成一个基本操作。完成一个基本操作所需要的时间称为机器周期

由于没有一个接口来设置固定重复的时间中断间隔，因此我们需要在每一次时钟中断时，设置再下一次的时钟中断。

```rust
/// os/src/interrupt/timer.rs
/// 触发时钟中断计数
pub static mut TICKS: usize = 0;

/// 每一次时钟中断时调用
/// 
/// 设置下一次时钟中断，同时计数 +1
pub fn tick() {
    set_next_timeout();
    unsafe {
        TICKS += 1;
        if TICKS % 100 == 0 {
            println!("{} ticks", TICKS);
        }
    }
}
```



完整的 timer.rs 

```rust
//! os/src/interrupt/timer.rs
//! 预约和处理时钟中断

use crate::sbi::set_timer;
use riscv::register::{time, sie, sstatus};

/// 初始化时钟中断
/// 
/// 开启时钟中断使能，并且预约第一次时钟中断
pub fn init() {
    unsafe {
        // 开启 STIE，允许时钟中断
        sie::set_stimer();
        // 开启 SIE（不是 sie 寄存器），允许内核态被中断打断
        sstatus::set_sie();
    }
    // 设置下一次时钟中断
    set_next_timeout();
}

/// 时钟中断的间隔，单位是 CPU 指令
static INTERVAL: usize = 100000;

/// 设置下一次时钟中断
/// 
/// 获取当前时间，加上中断间隔，通过 SBI 调用预约下一次中断
fn set_next_timeout() {
    set_timer(time::read() + INTERVAL);
}

/// 触发时钟中断计数
pub static mut TICKS: usize = 0;

/// 每一次时钟中断时调用
///
/// 设置下一次时钟中断，同时计数 +1
pub fn tick() {
    set_next_timeout();
    unsafe {
        TICKS += 1;
        if TICKS % 100 == 0 {
            println!("{} ticks", TICKS);
        }
    }
}
```



完整的sbi.rs

```rust
//os/src/sbi.rs
//! 调用 Machine 层的操作
// 目前还不会用到全部的 SBI 调用，暂时允许未使用的变量或函数
#![allow(unused)]

/// SBI 调用
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let ret;
    unsafe {
        llvm_asm!("ecall"
            : "={x10}" (ret)
            : "{x10}" (arg0), "{x11}" (arg1), "{x12}" (arg2), "{x17}" (which)
            : "memory"      // 如果汇编可能改变内存，则需要加入 memory 选项
            : "volatile");  // 防止编译器做激进的优化（如调换指令顺序等破坏 SBI 调用行为的优化）
    }
    ret
}


const SBI_SET_TIMER: usize = 0;
const SBI_CONSOLE_PUTCHAR: usize = 1;
const SBI_CONSOLE_GETCHAR: usize = 2;
const SBI_CLEAR_IPI: usize = 3;
const SBI_SEND_IPI: usize = 4;
const SBI_REMOTE_FENCE_I: usize = 5;
const SBI_REMOTE_SFENCE_VMA: usize = 6;
const SBI_REMOTE_SFENCE_VMA_ASID: usize = 7;
const SBI_SHUTDOWN: usize = 8;

/// 向控制台输出一个字符
///
/// 需要注意我们不能直接使用 Rust 中的 char 类型
pub fn console_putchar(c: usize) {
    sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}

/// 从控制台中读取一个字符
///
/// 没有读取到字符则返回 -1
pub fn console_getchar() -> usize {
    sbi_call(SBI_CONSOLE_GETCHAR, 0, 0, 0)
}

/// 调用 SBI_SHUTDOWN 来关闭操作系统（直接退出 QEMU）
pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    unreachable!()
}

/// 设置下一次时钟中断的时间
pub fn set_timer(time: usize) {
    sbi_call(SBI_SET_TIMER, time, 0, 0);
}
```



### 实现时钟中断的处理流程

接下来，我们在 `handle_interrupt()` 根据不同中断种类进行不同的处理流程。（

```rust
/// os/src/interrupt/handler.rs
use super::context::Context;
use super::timer;
use riscv::register::{
    scause::{Exception, Interrupt, Scause, Trap},
    sie, stvec,
};
global_asm!(include_str!("./interrupt.asm"));

/// 初始化中断处理
///
/// 把中断入口 `__interrupt` 写入 `stvec` 中，并且开启中断使能
pub fn init() {
    unsafe {
        extern "C" {
            /// `interrupt.asm` 中的中断入口
            fn __interrupt();
        }
        // 使用 Direct 模式，将中断入口设置为 `__interrupt`
        stvec::write(__interrupt as usize, stvec::TrapMode::Direct);
    }
}

/// 中断的处理入口
/// 
/// `interrupt.asm` 首先保存寄存器至 Context，其作为参数和 scause 以及 stval 一并传入此函数
/// 具体的中断类型需要根据 scause 来推断，然后分别处理
#[no_mangle]
pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) {
    // 可以通过 Debug 来查看发生了什么中断
    // println!("{:x?}", context.scause.cause());
    //对不同中断分开处理
    match scause.cause() {
        // 断点中断（ebreak）
        Trap::Exception(Exception::Breakpoint) => breakpoint(context),
        // 时钟中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => supervisor_timer(context),
        // 其他情况，终止当前线程
        _ => fault(context, scause, stval),
    }
}


/// 处理 ebreak 断点
/// 
/// 继续执行，其中 `sepc` 增加 2 字节，以跳过当前这条 `ebreak` 指令
fn breakpoint(context: &mut Context) {
    println!("Breakpoint at 0x{:x}", context.sepc);
    context.sepc += 2;
}

/// 处理时钟中断
/// 
/// 目前只会在 [`timer`] 模块中进行计数
fn supervisor_timer(_: &Context) {
    timer::tick();
}

/// 出现未能解决的异常，打印出触发异常的数据
fn fault(context: &mut Context, scause: Scause, stval: usize) {
    panic!(
        "Unresolved interrupt: {:?}\n{:x?}\nstval: {:x}",
        scause.cause(),
        context,
        stval
    );
}
```

> ### `match` 控制流运算符
>
> Rust 有一个叫做 `match` 的极为强大的控制流运算符，它允许我们将一个值与一系列的模式相比较，并根据相匹配的模式执行相应代码。模式可由字面值、变量、通配符和许多其他内容构成；
>
> ### 强制类型转换 `as` ：
>
> 只能转换基本数据类型 ，结构体类型只能转换指针，不能转换结构体引用

在我们测试时钟中断之前，我们要将新写的`timer.rs`  声明一下

```rust
//! os/src/interrupt/mod.rs
//! 中断模块
//! 
//! 

mod handler;
mod context;
mod timer;

pub use context::Context;

/// 初始化中断相关的子模块
/// 
/// - [`handler::init`]
/// - [`timer::init`]
pub fn init() {
    handler::init();    //调用中断处理初始化函数
    timer::init();      //调用时钟中断
    println!("中断处理初始化成功");
}
```

最后，我们来测试时钟中断：

只需将结尾的panic！改为loop{},目的是让操作系统不关机，不断地死循环，因此我们就可以看到系统不断地在触发时钟中断了。

运行输出

```
Finished dev [unoptimized + debuginfo] target(s) in 0.20s

OpenSBI v0.6
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name          : QEMU Virt Machine
Platform HART Features : RV64ACDFIMSU
Platform Max HARTs     : 8
Current Hart           : 0
Firmware Base          : 0x80000000
Firmware Size          : 120 KB
Runtime SBI Version    : 0.2

MIDELEG : 0x0000000000000222
MEDELEG : 0x000000000000b109
PMP0    : 0x0000000080000000-0x000000008001ffff (A)
PMP1    : 0x0000000000000000-0xffffffffffffffff (A,R,W,X)
中断处理初始化成功
Breakpoint at 0x80201134
Hello rCore-Tutorial!
100 ticks
200 ticks
300 ticks
400 ticks
...
```

同样可以使用 `ctrl+a` （macOS 为 `control+a`） 再按下 `x` 键退出。