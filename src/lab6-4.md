# 实验步骤

## 构建用户程序框架

接下来我们要做的工作，和实验准备中为操作系统「去除依赖」的工作十分类似：我们需要为用户程序提供一个类似的没有Rust std标准运行时依赖的极简运行时环境。这里我们会快速梳理一遍我们为用户程序进行的流程。

### 建立 crate

首先，我们先把上一个实验用来测验的user删除，并着手做一个测验的user

我们在 `os` 的旁边建立一个 `user` crate。此时，我们移除默认的 `main.rs`，而是在 `src` 目录下建立 `lib` 和 `bin` 子目录， 在 `lib` 中存放的是极简运行时环境，在 `bin` 中存放的源文件会被编译成多个单独的执行文件。

运行命令

```bash
cargo new --bin user
```

### 基础框架搭建

和操作系统一样，我们需要为用户程序移除 std 依赖，并且补充一些必要的功能。

#### 其他文件

- `.cargo/config` 设置编译目标为 RISC-V 64
- `console.rs` 实现 `print!` `println!` 宏
- 等

`.cargo/config` 设置编译目标为 RISC-V 64:

```toml
#os/.cargo/config
# 编译的目标平台
[build]
target = "riscv64gc-unknown-none-elf"
```

系统调用：

```rust
//user/src/syscall.rs
//! 系统调用

pub const STDIN: usize = 0;
pub const STDOUT: usize = 1;

const SYSCALL_READ: usize = 63;
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

/// 将参数放在对应寄存器中，并执行 `ecall`
fn syscall(id: usize, arg0: usize, arg1: usize, arg2: usize) -> isize {
    // 返回值
    let mut ret;
    unsafe {
        llvm_asm!("ecall"
            : "={x10}" (ret)
            : "{x10}" (arg0), "{x11}" (arg1), "{x12}" (arg2), "{x17}" (id)
            : "memory"      // 如果汇编可能改变内存，则需要加入 memory 选项
            : "volatile"); // 防止编译器做激进的优化（如调换指令顺序等破坏 SBI 调用行为的优化）
    }
    ret
}

/// 读取字符
pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize {
    loop {
        let ret = syscall(
            SYSCALL_READ,
            fd,
            buffer as *const [u8] as *const u8 as usize,
            buffer.len(),
        );
        if ret > 0 {
            return ret;
        }
    }
}

/// 打印字符串
pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(
        SYSCALL_WRITE,
        fd,
        buffer as *const [u8] as *const u8 as usize,
        buffer.len(),
    )
}

/// 退出并返回数值
pub fn sys_exit(code: isize) -> ! {
    syscall(SYSCALL_EXIT, code as usize, 0, 0);
    unreachable!()
}

```

`print!`与`println!`的实现：

```rust
//user/src/console.rs
//! 在系统调用基础上实现 `print!` `println!`
//!
//! 代码与 `os` crate 中的 `console.rs` 基本相同

use crate::syscall::*;
use alloc::string::String;
use core::fmt::{self, Write};

/// 实现 [`core::fmt::Write`] trait 来进行格式化输出
struct Stdout;

impl Write for Stdout {
    /// 打印一个字符串
    fn write_str(&mut self, s: &str) -> fmt::Result {
        sys_write(STDOUT, s.as_bytes());
        Ok(())
    }
}

/// 打印由 [`core::format_args!`] 格式化后的数据
pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

/// 实现类似于标准库中的 `print!` 宏
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

/// 实现类似于标准库中的 `println!` 宏
#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}

/// 从控制台读取一个字符（阻塞）
pub fn getchar() -> u8 {
    let mut c = [0u8; 1];
    sys_read(STDIN, &mut c);
    c[0]
}

/// 从控制台读取一个或多个字符（阻塞）
pub fn getchars() -> String {
    let mut buffer = [0u8; 64];
    loop {
        let size = sys_read(STDIN, &mut buffer);
        if let Ok(string) = String::from_utf8(buffer.iter().copied().take(size as usize).collect())
        {
            return string;
        }
    }
}

```

用户进程大小：

```rust
//user/src/config.rs
/// 每个用户进程所用的堆大小（1M）
pub const USER_HEAP_SIZE: usize = 0x10_0000;
```

#### `lib.rs`

- `#![no_std]` 移除标准库
- `#![feature(...)]` 开启一些不稳定的功能
- `#[global_allocator]` 使用库来实现动态内存分配
- `#[panic_handler]` panic 时终止

```rust
//user/src/lib.rs
//! 为各种用户程序提供依赖
//!
//! - 动态内存分配（允许使用 alloc，但总大小固定）
//! - 错误处理（打印信息并退出程序）

#![no_std]
#![feature(llvm_asm)]
#![feature(lang_items)]
#![feature(panic_info_message)]
#![feature(linkage)]

pub mod config;
pub mod syscall;

#[macro_use]
pub mod console;

extern crate alloc;

pub use crate::syscall::*;
use buddy_system_allocator::LockedHeap;
use config::USER_HEAP_SIZE;
use core::alloc::Layout;
use core::panic::PanicInfo;

/// 大小为 [`USER_HEAP_SIZE`] 的堆空间
static mut HEAP_SPACE: [u8; USER_HEAP_SIZE] = [0; USER_HEAP_SIZE];

/// 使用 `buddy_system_allocator` 中的堆
#[global_allocator]
static HEAP: LockedHeap = LockedHeap::empty();

/// 打印 panic 信息并退出用户程序
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!(
            "\x1b[1;31m{}:{}: '{}'\x1b[0m",
            location.file(),
            location.line(),
            info.message().unwrap()
        );
    } else {
        println!("\x1b[1;31mpanic: '{}'\x1b[0m", info.message().unwrap());
    }
    sys_exit(-1);
}

/// 程序入口
#[no_mangle]
pub extern "C" fn _start(_args: isize, _argv: *const u8) -> ! {
    unsafe {
        HEAP.lock()
            .init(HEAP_SPACE.as_ptr() as usize, USER_HEAP_SIZE);
    }
    sys_exit(main())
}

/// 默认的 main 函数
///
/// 设置了弱的 linkage，会被 `bin` 中文件的 `main` 函数取代
#[linkage = "weak"]
#[no_mangle]
fn main() -> isize {
    panic!("no main() linked");
}

/// 终止程序
#[no_mangle]
pub extern "C" fn abort() {
    panic!("abort");
}

/// 内存不足时终止程序
#[lang = "oom"]
fn oom(_: Layout) -> ! {
    panic!("out of memory");
}

```

`helloworld.rs`：

```rust
//user/src/bin/hello_world.rs
#![no_std]
#![no_main]

#[macro_use]
extern crate user;

#[no_mangle]
pub fn main() -> usize {
    println!("Hello world from user mode program!");
    0
}

```

`notebook.rs`：

```rust
//user/src/bin/notebook.rs
#![no_std]
#![no_main]

#[macro_use]
extern crate user;

use user::console::*;

#[no_mangle]
pub fn main() -> usize {
    println!("\x1b[2J<notebook>");
    loop {
        let char = getchars();
        if char == "`"{
            break;			//当输入`时退出记事本
        }
        print!("{}", string);
    }
    0
}
```

## 打包为磁盘镜像

在上一章我们已经实现了文件系统，并且可以让操作系统加载磁盘镜像。现在，我们只需要利用工具将编译后的用户程序打包为镜像，就可以使用了。

### 打包

这个工具可以将一个目录打包成 SimpleFileSystem 格式的磁盘镜像。为此，我们需要将编译得到的 ELF 文件单独放在一个导出目录中，即 `user/build/disk`。

```makefile
#user/Makefile
TARGET      := riscv64gc-unknown-none-elf
MODE        := debug

# 用户程序目录
SRC_DIR		:= src/bin
# 编译后执行文件目录
TARGET_DIR	:= target/$(TARGET)/$(MODE)
# 用户程序源文件
SRC_FILES	:= $(wildcard $(SRC_DIR)/*.rs)
# 根据源文件取得编译后的执行文件
BIN_FILES	:= $(patsubst $(SRC_DIR)/%.rs, $(TARGET_DIR)/%, $(SRC_FILES))

OUT_DIR		:= build/disk
IMG_FILE	:= build/raw.img
QCOW_FILE	:= build/disk.img

.PHONY: dependency build clean

# 安装 rcore-fs-fuse 工具
dependency:
ifeq ($(shell which rcore-fs-fuse),)
	@echo Installing rcore-fs-fuse
	@cargo install rcore-fs-fuse --git https://github.com/rcore-os/rcore-fs
endif

# 编译、打包、格式转换、预留空间
build: dependency
    # 编译
    @cargo build
    @echo Targets: $(patsubst $(SRC_DIR)/%.rs, %, $(SRC_FILES))
    # 移除原有的所有文件
    @rm -rf $(OUT_DIR)
    @mkdir -p $(OUT_DIR)
    # 复制编译生成的 ELF 至目标目录
    @cp $(BIN_FILES) $(OUT_DIR)
    # 使用 rcore-fs-fuse 工具进行打包
    @rcore-fs-fuse --fs sfs $(IMG_FILE) $(OUT_DIR) zip
    # 将镜像文件的格式转换为 QEMU 使用的高级格式
    @qemu-img convert -f raw $(IMG_FILE) -O qcow2 $(QCOW_FILE)
    # 提升镜像文件的容量（并非实际大小），来允许更多数据写入
    @qemu-img resize $(QCOW_FILE) +1G

clean:
	@cargo clean
	@rm -rf $(OUT_DIR) $(IMG_FILE) $(QCOW_FILE)
```

在 `os/Makefile` 中指定我们新生成的 `QCOW_FILE` 为加载镜像，就可以在操作系统中看到打包好的目录了。

添加依赖

```toml
#user/Cargo.toml
[dependencies]
buddy_system_allocator = "0.6.0"
```

最后`make build`生成镜像

## 解析 ELF 文件并创建线程

在之前实现内核线程时，我们只需要为线程指定一个起始位置就够了，因为所有的代码都在操作系统之中。但是现在，我们需要从 ELF 文件中加载用户程序的代码和数据信息，并且映射到内存中。

当然，我们不需要自己实现 ELF 文件解析器，因为有 `xmas-elf` 这个 crate 替我们实现了 ELF 的解析。

> ## ELF文件
>
> 在计算机科学中，是一种用于[二进制文件](https://baike.baidu.com/item/二进制文件/996661)、[可执行文件](https://baike.baidu.com/item/可执行文件)、[目标代码](https://baike.baidu.com/item/目标代码/9407934)、共享库和核心转储格式文件
>
> 组成部分：ELF文件由4部分组成，分别是ELF头（ELF header）、程序头表（Program header table）、节（Section）和节头表（Section header table）。实际上，一个文件中不一定包含全部内容，而且它们的位置也未必如同所示这样安排，只有ELF头的位置是固定的，其余各部分的位置、大小等信息由ELF头中的各项值来决定。

### 读取文件内容

`xmas-elf` 需要将 ELF 文件首先读取到内存中。在上一章文件系统的基础上，我们很容易为 `INode` 添加一个将整个文件作为 `[u8]` 读取出来的方法：

```rust
//os/src/fs/inode_ext.rs
//! 为 [`INode`] 实现 trait [`INodeExt`] 以扩展功能

use super::*;

/// 为 [`INode`] 类型添加的扩展功能
pub trait INodeExt {
    /// 打印当前目录的文件
    fn ls(&self);

    /// 读取文件内容
    fn readall(&self) -> Result<Vec<u8>>;
}

impl INodeExt for dyn INode {
    fn ls(&self) {
        let mut id = 0;
        while let Ok(name) = self.get_entry(id) {
            println!("{}", name);
            id += 1;
        }
    }

    fn readall(&self) -> Result<Vec<u8>> {
        // 从文件头读取长度
        let size = self.metadata()?.size;
        // 构建 Vec 并读取
        let mut buffer = Vec::with_capacity(size);
        unsafe { buffer.set_len(size) };
        self.read_at(0, buffer.as_mut_slice())?;
        Ok(buffer)
    }
}
```

同时，将mod中的ls删去：

```rust
//os/src/fs/mod.rs
//删去
/*pub fn ls(path: &str) {
    let mut id = 0;
    let dir = ROOT_INODE.lookup(path).unwrap();
    print!("files in {}: \n  ", path);
    while let Ok(name) = dir.get_entry(id) {
        id += 1;
        print!("{} ", name);
    }
    print!("\n");
}*/

/// 触发 [`static@ROOT_INODE`] 的初始化并打印根目录内容
pub fn init() {
    ROOT_INODE.ls();
    println!("文件系统初始化成功");
}
```

### 解析各个字段

对于 ELF 中的不同字段，其存放的地址通常是不连续的，同时其权限也会有所不同。我们利用 `xmas-elf` 库中的接口，便可以从读出的 ELF 文件中对应建立 `MemorySet`。

注意到，用户程序也会首先映射所有内核态的空间，否则将无法进行中断处理。

```rust
//os/src/memory/mapping/memory_set.rs
/// 通过 elf 文件创建内存映射（不包括栈）
pub fn from_elf(file: &ElfFile, is_user: bool) -> MemoryResult<MemorySet> {
    // 建立带有内核映射的 MemorySet
    let mut memory_set = MemorySet::new_kernel()?;

    // 遍历 elf 文件的所有部分
    for program_header in file.program_iter() {
        if program_header.get_type() != Ok(Type::Load) {
            continue;
        }
        // 从每个字段读取「起始地址」「大小」和「数据」
        let start = VirtualAddress(program_header.virtual_addr() as usize);
        let size = program_header.mem_size() as usize;
        let data: &[u8] =
            if let SegmentData::Undefined(data) = program_header.get_data(file).unwrap() {
                data
            } else {
                return Err("unsupported elf format");
            };

        // 将每一部分作为 Segment 进行映射
        let segment = Segment {
            map_type: MapType::Framed,
            range: Range::from(start..(start + size)),
            flags: Flags::user(is_user)
                | Flags::readable(program_header.flags().is_read())
                | Flags::writable(program_header.flags().is_write())
                | Flags::executable(program_header.flags().is_execute()),
        };

        // 建立映射并复制数据
        memory_set.add_segment(segment, Some(data))?;
    }

    Ok(memory_set)
}
```

在`process.rs`中添加从文件中读取代码方法,：

```rust
//os/src/process/process.rs 	

use crate::fs::*;
use xmas_elf::ElfFile;

pub struct ProcessInner {
    /// 进程中的线程公用页表 / 内存映射
    pub memory_set: MemorySet,
    /// 打开的文件描述符
    pub descriptors: Vec<Arc<dyn INode>>,
}
impl Process{
     pub fn new_kernel() -> MemoryResult<Arc<Self>> {
        Ok(Arc::new(Self {
            is_user: false,
            inner: Mutex::new(ProcessInner {
                memory_set: MemorySet::new_kernel()?,
                descriptors: vec![STDIN.clone(), STDOUT.clone()],
            }),
        }))
    }
    
    /// 创建进程，从文件中读取代码
    pub fn from_elf(file: &ElfFile, is_user: bool) -> MemoryResult<Arc<Self>> {
        Ok(Arc::new(Self {
            is_user,
            inner: Mutex::new(ProcessInner {
                memory_set: MemorySet::from_elf(file, is_user)?,
                descriptors: vec![STDIN.clone(), STDOUT.clone()],
            }),
        }))
    }
}

```

### 加载数据到内存中

思考：我们在为用户程序建立映射时，虚拟地址是 ELF 文件中写明的，那物理地址是程序在磁盘中存储的地址吗？这样做有什么问题吗？

> 我们在模拟器上运行可能不觉得，但是如果直接映射磁盘空间，使用时会带来巨大的延迟，所以需要在程序准备运行时，将其磁盘中的数据复制到内存中。如果程序较大，操作系统可能只会复制少量数据，而更多的则在需要时再加载。当然，我们实现的简单操作系统就一次性全都加载到内存中了。
>
> 而且，就算是想要直接映射磁盘空间，也不一定可行。这是因为虚实地址转换时，页内偏移是不变的。这时就无法保证在 ELF 中指定的地址和其在磁盘中的地址满足这样的关系。

我们将修改 `Mapping::map` 函数，为其增加一个参数表示用于初始化的数据。在实现时，有一些重要的细节需要考虑。

- 因为用户程序的内存分配是动态的，其分配到的物理页面不一定连续，所以必须单独考虑每一个页面
- 每一个字段的长度不一定是页大小的倍数，所以需要考虑不足一个页时的复制情况
- 程序有一个 bss 段，它在 ELF 中不保存数据，而其在加载到内存是需要零初始化
- 对于一个页面，有其**物理地址**、**虚拟地址**和**待加载数据的地址**。此时，是不是直接从**待加载数据的地址**拷贝到页面的**虚拟地址**，如同 `memcpy` 一样就可以呢？

具体的实现，可以查看 `os/src/memory/mapping/mapping.rs` 中的 `Mapping::map` 函数。

### 运行 Hello World？

可惜的是，我们不能像内核线程一样在用户程序中直接使用 `print`。前者是基于 OpenSBI 的机器态 SBI 调用，而为了让用户程序能够打印字符，我们还需要在操作系统中实现系统调用来给用户进程提供服务。

## 实现系统调用

目前，我们实现 `sys_read` `sys_write` 和 `sys_exit` 三个简单的系统调用。通过学习它们的实现，更多的系统调用也并没有多难。

### 用户程序中调用系统调用

在用户程序中实现系统调用比较容易，就像我们之前在操作系统中使用 `sbi_call` 一样，只需要符合规则传递参数即可。而且这一次我们甚至不需要参考任何标准，每个人都可以为自己的操作系统实现自己的标准。

例如，在实验指导中，系统调用的编号使用了 musl 中的编码和参数格式。但实际上，在实现操作系统的时候，编码和参数格式都可以随意调整，只要在用户程序中的调用和操作系统中的解释相符即可。

代码示例

```rust
// musl 中的 sys_read 调用格式
llvm_asm!("ecall" :
    "={x10}" (/* 返回读取长度 */) :
    "{x10}" (/* 文件描述符 */),
    "{x11}" (/* 读取缓冲区 */),
    "{x12}" (/* 缓冲区长度 */),
    "{x17}" (/* sys_read 编号 63 */) ::
);
// 一种可能的 sys_read 调用格式
llvm_asm!("ecall" :
    "={x10}" (/* 现在的时间 */),
    "={x11}" (/* 今天的天气 */),
    "={x12}" (/* 读取一个字符 */) :
    "{x20}" (/* sys_read 编号 0x595_7ead */) ::
);
```

实验指导提供了第一种无趣的系统调用格式。

> ## musl
>
> musl 是一个全新为 Linux 基本系统实现的标准库。特点是轻量级、快速、简单、免费、标准兼容和安全。

### 避免忙等待

在常见操作系统中，一些延迟非常大的操作，例如文件读写、网络通讯，都可以使用异步接口来进行。但是为了实现更加简便，我们的读写系统调用都是阻塞的。在 `sys_read` 中，使用了 `loop` 来保证仅当成功读取字符时才返回。

此时，如果用户程序需要获取从控制台输入的字符，但是此时并没有任何字符到来。那么，程序将被阻塞，而操作系统的职责就是尽量减少线程执行无用阻塞占用 CPU 的时间，而是将这段时间分配给其他可以执行的线程。具体的做法，将会在后面**条件变量**的章节讲述。

### 操作系统中实现系统调用

在操作系统中，系统调用的实现和中断处理一样，有同样的入口，而针对不同的参数设置不同的处理流程。为了简化流程，我们不妨把系统调用的处理结果分为三类：

- 返回一个数值，程序继续执行
- 程序进入等待
- 程序将被终止

### 系统调用的处理流程

- 首先，从相应的寄存器中取出调用代号和参数
- 根据调用代号，进入不同的处理流程，得到处理结果
  - 返回数值并继续执行：
    - 返回值存放在 `x10` 寄存器，`sepc += 4`，继续此 `context` 的执行
  - 程序进入等待
    - 同样需要更新 `x10` 和 `sepc`，但是需要将当前线程标记为等待，切换其他线程来执行
  - 程序终止
    - 不需要考虑系统调用的返回，直接删除线程

### 具体的调用实现

那么具体该如何实现读 / 写系统调用呢？这里我们会利用文件的统一接口 `INode`，使用其中的 `read_at()` 和 `write_at()` 接口即可。下一节就将讲解如何处理文件描述符。

## 处理文件描述符

尽管很不像，但是在大多操作系统中，标准输入输出流 `stdin` 和 `stdout` 虽然叫做「流」，但它们都有文件的接口。我们同样也会将它们实现成为文件。

但是不用担心，作为文件的许多功能，`stdin` 和 `stdout` 都不会支持。我们只需要为其实现最简单的读写接口。

### 进程打开的文件

操作系统需要为进程维护一个进程打开的文件清单。其中，一定存在的是 `stdin` `stdout` 和 `stderr`。为了简便，我们只实现 `stdin` 和 `stdout`，它们的文件描述符数值分别为 0 和 1。

### `stdout`

输出流最为简单：每当遇到系统调用时，直接将缓冲区中的字符通过 SBI 调用打印出去。

```rust
//os/src/fs/stdout.rs
//! 控制台输出 [`Stdout`]

use super::*;

lazy_static! {
    pub static ref STDOUT: Arc<Stdout> = Default::default();
}

/// 控制台输出
#[derive(Default)]
pub struct Stdout;

impl INode for Stdout {
    //将缓冲区中的字符打印出去
    fn write_at(&self, offset: usize, buf: &[u8]) -> Result<usize> {
        if offset != 0 {
            Err(FsError::NotSupported)
        } else if let Ok(string) = core::str::from_utf8(buf) {
            print!("{}", string);
            Ok(buf.len())
        } else {
            Err(FsError::InvalidParam)
        }
    }

    /// Read bytes at `offset` into `buf`, return the number of bytes read.
    fn read_at(&self, _offset: usize, _buf: &mut [u8]) -> Result<usize> {
        Err(FsError::NotSupported)
    }

    fn poll(&self) -> Result<PollStatus> {
        Err(FsError::NotSupported)
    }

    /// This is used to implement dynamics cast.
    /// Simply return self in the implement of the function.
    fn as_any_ref(&self) -> &dyn Any {
        self
    }
}
```

### `stdin`

输入流较为复杂：每当遇到系统调用时，通过中断或轮询方式获取字符：如果有，就进一步获取；如果没有就等待。直到收到约定长度的字符串才返回。

```rust
//os/src/fs/stdin.rs
//! 键盘输入 [`Stdin`]

use super::*;
use alloc::collections::VecDeque;

lazy_static! {
    pub static ref STDIN: Arc<Stdin> = Default::default();
}

/// 控制台键盘输入，实现 [`INode`] 接口
#[derive(Default)]
pub struct Stdin {
    /// 从后插入，前段弹出
    buffer: Mutex<VecDeque<u8>>,
    /// 条件变量用于使等待输入的线程休眠
    condvar: Condvar,
}

impl INode for Stdin {
    /// Read bytes at `offset` into `buf`, return the number of bytes read.
    fn read_at(&self, offset: usize, buf: &mut [u8]) -> Result<usize> {
        if offset != 0 {
            // 不支持 offset
            Err(FsError::NotSupported)
        } else if self.buffer.lock().len() == 0 {
            // 缓冲区没有数据，将当前线程休眠
            self.condvar.wait();
            Ok(0)
        } else {
            let mut stdin_buffer = self.buffer.lock();
            for (i, byte) in buf.iter_mut().enumerate() {
                if let Some(b) = stdin_buffer.pop_front() {
                    *byte = b;
                } else {
                    return Ok(i);
                }
            }
            Ok(buf.len())
        }
    }

    /// Write bytes at `offset` from `buf`, return the number of bytes written.
    fn write_at(&self, _offset: usize, _buf: &[u8]) -> Result<usize> {
        Err(FsError::NotSupported)
    }

    fn poll(&self) -> Result<PollStatus> {
        Err(FsError::NotSupported)
    }

    /// This is used to implement dynamics cast.
    /// Simply return self in the implement of the function.
    fn as_any_ref(&self) -> &dyn Any {
        self
    }
}

impl Stdin {
    /// 向缓冲区插入一个字符，然后唤起一个线程
    pub fn push(&self, c: u8) {
        self.buffer.lock().push_back(c);
        self.condvar.notify_one();
    }
}
```

添加声明：

```rust
//os/src/fs/mod.rs
mod stdin;
mod stdout;
mod inode_ext;

use crate::kernel::Condvar;
use core::any::Any;

pub use inode_ext::INodeExt;
pub use stdin::STDIN;
pub use stdout::STDOUT;
```

### 外部中断

对于用户程序而言，外部输入是随时主动读取的数据。但是事实上外部输入通常时间短暂且不会等待，需要操作系统立即处理并缓冲下来，再等待程序进行读取。所以，每一个键盘按键对于操作系统而言都是一次短暂的中断。

而在之前的实验中操作系统不会因为一个按键就崩溃，是因为 OpenSBI 默认会关闭各种外部中断。但是现在我们需要将其打开，来接受按键信息，并且对外部中断进行响应：

```rust
//os/src/interrupt/handler.rs
use super::context::Context;
use super::timer;
use crate::fs::STDIN;
use crate::kernel::syscall_handler;
use crate::memory::*;
use crate::process::PROCESSOR;
use crate::sbi::console_getchar;
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

        // 开启外部中断使能
        sie::set_sext();

        // 在 OpenSBI 中开启外部中断
        *PhysicalAddress(0x0c00_2080).deref_kernel() = 1u32 << 10;
        // 在 OpenSBI 中开启串口
        *PhysicalAddress(0x1000_0004).deref_kernel() = 0x0bu8;
        *PhysicalAddress(0x1000_0001).deref_kernel() = 0x01u8;
        // 其他一些外部中断相关魔数
        *PhysicalAddress(0x0C00_0028).deref_kernel() = 0x07u32;
        *PhysicalAddress(0x0C20_1000).deref_kernel() = 0u32;
    }
}

/// 中断的处理入口
/// 
/// `interrupt.asm` 首先保存寄存器至 Context，其作为参数和 scause 以及 stval 一并传入此函数
/// 具体的中断类型需要根据 scause 来推断，然后分别处理
#[no_mangle]
pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) -> *mut Context{
    // 可以通过 Debug 来查看发生了什么中断
    // println!("{:x?}", context.scause.cause());
    {

        let mut processor = PROCESSOR.lock();
        let current_thread = processor.current_thread();
        if current_thread.as_ref().inner().dead {
            println!("thread {} exit", current_thread.id);
            processor.kill_current_thread();
            return processor.prepare_next_thread();
        }
    }
    // 根据中断类型来处理，返回的 Context 必须位于放在内核栈顶
    match scause.cause() {
        // 断点中断（ebreak）
        Trap::Exception(Exception::Breakpoint) => breakpoint(context),
        // 系统调用
        Trap::Exception(Exception::UserEnvCall) => syscall_handler(context),
        // 时钟中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => supervisor_timer(context),
        // 外部中断（键盘输入）
        Trap::Interrupt(Interrupt::SupervisorExternal) => supervisor_external(context),
        // 其他情况，无法处理
        _ => fault("unimplemented interrupt type", scause, stval),
    }
}

/// 处理 ebreak 断点
/// 
/// 继续执行，其中 `sepc` 增加 2 字节，以跳过当前这条 `ebreak` 指令
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

/// 处理外部中断，只实现了键盘输入
fn supervisor_external(context: &mut Context) -> *mut Context {
    let mut c = console_getchar();
    if c <= 255 {
        if c == '\r' as usize {
            c = '\n' as usize;
        }
        STDIN.push(c as u8);
    }
    context
}

/// 出现未能解决的异常
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

这里，我们需要按照 OpenSBI 的接口在指定的地址进行配置。好在这些地址都在文件系统映射的空间内，就不需要再为其单独建立内存映射了。开启中断使能后，任何一个按键都会导致程序进入 `unimplemented!` 的区域。

### 实现输入流

输入流则需要配有一个缓冲区，我们可以用 `alloc::collections::VecDeque` 来实现。在遇到键盘中断时，调用 `sbi_call` 来获取字符并加入到缓冲区中。当遇到系统调用 `sys_read` 时，再相应从缓冲区中取出一定数量的字符。

那么，如果遇到了 `sys_read` 系统调用，而缓冲区并没有数据可以读取，应该如何让线程进行等待，而又不浪费 CPU 资源呢？

## 条件变量

> 条件变量是线程的另外一种同步机制，这些同步对象为线程提供了会合的场所，理解起来就是两个（或者多个）线程需要碰头（或者说进行交互-一个线程给另外的一个或者多个线程发送消息），我们指定在条件变量这个地方发生，一个线程用于修改这个变量使其满足其它线程继续往下执行的条件，其它线程则接收条件已经发生改变的信号。
>
> 条件变量同锁一起使用使得线程可以以一种**无竞争**的方式等待任意条件的发生。所谓无竞争就是，条件改变这个信号会发送到所有等待这个信号的线程。而不是说一个线程接受到这个消息而其它线程就接收不到了。

条件变量（conditional variable）的常见接口是这样的：

- wait：当前线程开始等待这个条件变量
- notify_one：让某一个等待此条件变量的线程继续运行
- notify_all：让所有等待此变量的线程继续运行

条件变量和互斥锁的区别在于，互斥锁解铃还须系铃人，但条件变量可以由任何来源发出 notify 信号。同时，互斥锁的一次 lock 一定对应一次 unlock，但条件变量多次 notify 只能保证 wait 的线程执行次数不超过 notify 次数。

为输入流加入条件变量后，就可以使得调用 `sys_read` 的线程在等待期间保持休眠，不被调度器选中，消耗 CPU 资源。

### 实现条件变量

条件变量会被包含在输入流等涉及等待和唤起的结构中，而一个条件变量保存的就是所有等待它的线程。

```rust
//os/src/kernel/condvar.rs
//! 条件变量

use super::*;
use alloc::collections::VecDeque;

#[derive(Default)]
pub struct Condvar {
    /// 所有等待此条件变量的线程
    watchers: Mutex<VecDeque<Arc<Thread>>>,
}

impl Condvar {
    /// 令当前线程休眠，等待此条件变量
    pub fn wait(&self) {
        self.watchers
            .lock()
            .push_back(PROCESSOR.lock().current_thread());
        PROCESSOR.lock().sleep_current_thread();
    }

    /// 唤起一个等待此条件变量的线程
    pub fn notify_one(&self) {
        if let Some(thread) = self.watchers.lock().pop_front() {
            PROCESSOR.lock().wake_thread(thread);
        }
    }
}
```

当一个线程调用 `sys_read` 而缓冲区为空时，就会将其加入条件变量的 `watcher` 中，同时在 `Processor` 中移出活跃线程。而当键盘中断到来，读取到字符时，就会将线程重新放回调度器中，准备下一次调用。

实现系统调用：

```rust
//os/src/kernel/syscall.rs
//! 实现各种系统调用

use super::*;

pub const SYS_READ: usize = 63;
pub const SYS_WRITE: usize = 64;
pub const SYS_EXIT: usize = 93;

/// 系统调用在内核之内的返回值
pub(super) enum SyscallResult {
    /// 继续执行，带返回值
    Proceed(isize),
    /// 记录返回值，但暂存当前线程
    Park(isize),
    /// 丢弃当前 context，调度下一个线程继续执行
    Kill,
}

/// 系统调用的总入口
pub fn syscall_handler(context: &mut Context) -> *mut Context {
    // 无论如何处理，一定会跳过当前的 ecall 指令
    context.sepc += 4;

    let syscall_id = context.x[17];
    let args = [context.x[10], context.x[11], context.x[12]];

    let result = match syscall_id {
        SYS_READ => sys_read(args[0], args[1] as *mut u8, args[2]),
        SYS_WRITE => sys_write(args[0], args[1] as *mut u8, args[2]),
        SYS_EXIT => sys_exit(args[0]),
        _ => {
            println!("unimplemented syscall: {}", syscall_id);
            SyscallResult::Kill
        }
    };

    match result {
        SyscallResult::Proceed(ret) => {
            // 将返回值放入 context 中
            context.x[10] = ret as usize;
            context
        }
        SyscallResult::Park(ret) => {
            // 将返回值放入 context 中
            context.x[10] = ret as usize;
            // 保存 context，准备下一个线程
            PROCESSOR.lock().park_current_thread(context);
            PROCESSOR.lock().prepare_next_thread()
        }
        SyscallResult::Kill => {
            // 终止，跳转到 PROCESSOR 调度的下一个线程
            PROCESSOR.lock().kill_current_thread();
            PROCESSOR.lock().prepare_next_thread()
        }
    }
}
```

实现文件相关的内核功能：

```rust
//os/src/kernel/fs.rs
//! 文件相关的内核功能

use super::*;
use core::slice::from_raw_parts_mut;

/// 从指定的文件中读取字符
///
/// 如果缓冲区暂无数据，返回 0；出现错误返回 -1
pub(super) fn sys_read(fd: usize, buffer: *mut u8, size: usize) -> SyscallResult {
    // 从进程中获取 inode
    let process = PROCESSOR.lock().current_thread().process.clone();
    if let Some(inode) = process.inner().descriptors.get(fd) {
        // 从系统调用传入的参数生成缓冲区
        let buffer = unsafe { from_raw_parts_mut(buffer, size) };
        // 尝试读取
        if let Ok(ret) = inode.read_at(0, buffer) {
            let ret = ret as isize;
            if ret > 0 {
                return SyscallResult::Proceed(ret);
            }
            if ret == 0 {
                return SyscallResult::Park(ret);
            }
        }
    }
    SyscallResult::Proceed(-1)
}

/// 将字符写入指定的文件
pub(super) fn sys_write(fd: usize, buffer: *mut u8, size: usize) -> SyscallResult {
    // 从进程中获取 inode
    let process = PROCESSOR.lock().current_thread().process.clone();
    if let Some(inode) = process.inner().descriptors.get(fd) {
        // 从系统调用传入的参数生成缓冲区
        let buffer = unsafe { from_raw_parts_mut(buffer, size) };
        // 尝试写入
        if let Ok(ret) = inode.write_at(0, buffer) {
            let ret = ret as isize;
            if ret >= 0 {
                return SyscallResult::Proceed(ret);
            }
        }
    }
    SyscallResult::Proceed(-1)
}
```

实现进程相关的内核功能：

```rust
//os/src/kernel/process.rs
//! 进程相关的内核功能

use super::*;

pub(super) fn sys_exit(code: usize) -> SyscallResult {
    println!(
        "thread {} exit with code {}",
        PROCESSOR.lock().current_thread().id,
        code
    );
    SyscallResult::Kill
}
```

进行声明：

```rust
//os/src/kernel/mod.rs
//! 为进程提供系统调用等内核功能

mod condvar;
mod fs;
mod process;
mod syscall;

use crate::interrupt::*;
use crate::process::*;
use alloc::sync::Arc;
pub(self) use fs::*;
pub(self) use process::*;
use spin::Mutex;
pub(self) use syscall::*;

pub use condvar::Condvar;
pub use syscall::syscall_handler;

```

到这里，我们已经实现了一个小却功能还算完整的内核了，我们尝试在操作系统里运行我们写好的`notebook`和`hello_world`程序

```rust
//os/src/main.rs
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
//!
//! - `#![feature(naked_functions)]`
//!   允许使用 naked 函数，即编译器不在函数前后添加出入栈操作。
//!   这允许我们在函数中间内联汇编使用 `ret` 提前结束，而不会导致栈出现异常
#![feature(naked_functions)]

#[macro_use]
mod console;
mod drivers;
mod fs;
mod panic;
mod sbi;
mod interrupt;
mod memory;
mod process;
mod kernel;
extern crate alloc;

use alloc::sync::Arc;
use memory::PhysicalAddress;
use process::*;
use fs::{INodeExt, ROOT_INODE};
use xmas_elf::ElfFile;
// 汇编编写的程序入口，具体见该文件
global_asm!(include_str!("entry.asm"));

/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
#[no_mangle]
pub extern "C" fn rust_main(_hart_id: usize, dtb_pa: PhysicalAddress) -> ! {
    memory::init();
    interrupt::init();
    drivers::init(dtb_pa);
    fs::init();

    {
        // 创建一个内核进程
        let mut processor = PROCESSOR.lock();
	    processor.add_thread(create_user_process("notebook"));    
        processor.add_thread(create_user_process("hello_world"));    
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

fn sample_process(id: usize) {
    println!("hello from kernel thread {}", id);
}
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
/// 内核线程需要调用这个函数来退出
fn kernel_thread_exit() {
    // 当前线程标记为结束
    PROCESSOR.lock().current_thread().as_ref().inner().dead = true;
    // 制造一个中断来交给操作系统处理
    unsafe { llvm_asm!("ebreak" :::: "volatile") };
}

/// 测试任何内核线程都可以操作文件系统和驱动
fn simple(id: usize) {
    println!("hello from thread id {}", id);
    // 新建一个目录
    ROOT_INODE
        .create("tmp", rcore_fs::vfs::FileType::Dir, 0o666)
        .expect("failed to mkdir /tmp");
    println!("create finish");
    // 输出根文件目录内容S
    ROOT_INODE.ls();
    panic!()
}

// 创建一个用户进程，从指定的文件名读取 ELF
pub fn create_user_process(name: &str) -> Arc<Thread> {
    // 从文件系统中找到程序
    let app = ROOT_INODE.find(name).unwrap();
    // 读取数据
    let data = app.readall().unwrap();
    // 解析 ELF 文件
    let elf = ElfFile::new(data.as_slice()).unwrap();
    // 利用 ELF 文件创建线程，映射空间并加载数据
    let process = Process::from_elf(&elf, true).unwrap();
    // 再从 ELF 中读出程序入口地址
    Thread::new(process, elf.header.pt2.entry_point() as usize, None).unwrap()
}
```

在`make run`之前，将时钟中断中的`println!`语句删除，否则会影响我们在使用程序的体验：

```rust
//os/src/interrupt/timer.rs
/// 每一次时钟中断时调用
///
/// 设置下一次时钟中断，同时计数 +1
pub fn tick() {
    set_next_timeout();
    //删除时钟中断引发的输出
    /*unsafe {
        TICKS += 1;
        if TICKS % 100 == 0 {
            println!("{} ticks", TICKS);
        }
    }*/
}
```

然后我们`make run`

```
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
内存初始化成功
中断处理初始化成功
PhysicalAddress(2183135232)
驱动模块初始化成功
.
..
hello_world
notebook
文件系统初始化成功

<notebook>
Hello world from user mode program!
thread 2 exit with code 0

```

可以看到我们已经进入了用户态的`notebook`程序，并且看到了`hello_world`程序的结果，后面我们就可以往里面打各种字符最后用  `  结束程序。
