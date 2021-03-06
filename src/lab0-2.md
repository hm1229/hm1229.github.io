# 知识储备

## 应用程序执行环境与平台



### 环境栈

![../_images/app-software-stack.png](资源文件\实验零.assets\百度网盘.lnk)

*应用程序执行环境栈：图中的白色块自上而下（越往下则越靠近底层，下层作为上层的执行环境支持上层代码的运行）表示各级**执行环境**， 黑色块则表示相邻两层执行环境之间的**接口**。



#### Hello, world! 用到的系统调用

从之前的 `cargo run` 的输出可以看出之前构建的可执行文件是在 target/debug 目录下的 os 。 在 Ubuntu 系统上，可以通过 `strace` 工具来运行一个程序并输出程序运行过程当中向内核请求的所有的系统调用及其返回值。 我们只需输入 `strace target/debug/os` 即可看到一长串的系统调用。

其中，真正容易看出与 `Hello, world!` 相关的只有一个系统调用：

```
write(1, "Hello, world!\n", 14)         = 14
```

其参数的具体含义我们暂且不进行解释。

其余的系统调用基本上分别用于函数库和内核两层执行环境的初始化工作和对于上层的运行期监控和管理。之后，随着应用场景的复杂化，我们需要更强的抽象能力，也会实现这里面的一些系统调用。



#### 从应用程序到标准库

我们的应用位于最上层，它可以通过调用编程语言提供的**标准库**或者其他**三方库**对外提供的功能强大的函数接口，使得仅需少量的源代码就能完成复杂的功能。

但是这些库的功能不仅限于此，事实上它们属于应用程序的 **执行环境** (Execution Environment)，在我们通常不会注意到的地方，它们还会在执行应用之前完成一些初始化工作，并在应用程序执行的时候对它进行监控。

我们在打印 `Hello, world!` 时使用的 `println!` 宏正是由 **Rust 标准库 std** 和**GNU Libc库**等提供的。



#### 从标准库到内核/操作系统

将线性地址空间分为**用户空间**和**内核空间**，低地址空间被操作系统占用，称为内核空间，高地址空间则称为用户空间。从内核/操作系统的角度看来，，它上面的一切都属于**用户态**，而它自身属于**内核态**。无论用户态应用如何编写，是手写汇编代码，还是基于某种编程语言利用其标准库或三方库，某些功能总要直接或间接的通过内核/操作系统提供的 **系统调用** (System Call) 来实现。因此系统调用充当了**用户和内核之间的边界**。内核作为用户态的**执行环境**，它不仅要**提供系统调用接口**，还需要对用户态应用的执行进行**监控和管理**。



#### 从内核/操作系统到硬件平台

从硬件的角度来看，它上面的一切都属于软件。硬件可以分为三种： 

* 处理器 (Processor) ——它更常见的名字是中央处理单元 (CPU, Central Processing Unit)
* 内存 (Memory) 
* I/O 设备

其中处理器无疑是其中最复杂同时也最关键的一个。它与软件约定一套 **指令集体系结构** (ISA, Instruction Set Architecture)， 使得软件可以通过 ISA 中提供的汇编指令来访问各种硬件资源。软件当然也需要知道处理器会如何执行这些指令：最简单的话就是一条一条执行位于内存中的指令。当然，实际的情况远比这个要复杂得多，为了适应现代应用程序的场景，处理器还需要提供很多额外的机制，而不仅仅是让数据在 CPU 寄存器、内存和 I/O 设备三者之间流动。



#### 多层执行的必要性

除了最上层的应用程序和最下层的硬件平台必须存在之外，作为中间层的函数库和内核并不是必须存在的：它们都是对下层资源进行了 **抽象** (Abstraction)， 并为上层提供了一套执行环境。抽象的优点在于它让上层以较小的代价获得所需的功能，并同时可以提供一些保护。但抽象同时也是一种限制，会丧失一些应有的灵活性。比如，当你在考虑在项目中应该使用哪个函数库的时候，就常常需要这方面的权衡：过多的抽象和过少的抽象自然都是不合适的。

实际上，我们通过应用程序的特征来判断它需要什么程度的抽象。

- 如果函数库和内核都不存在，那么我们就是在手写汇编代码，这种方式具有最高的灵活性，抽象能力则最低，基本等同于硬件。我们通常用这种方式来 实现一些架构相关且仅通过编程语言无法描述的小模块或者代码片段。
- 如果仅存在函数库而不存在内核，意味着我们不需要内核提供的抽象。在嵌入式场景就常常会出现这种情况。嵌入式中设备虽然也包含 CPU、内存和 I/O 设备，但是它上面通常只会同时运行一个或几个功能非常简单的小应用程序，其定位就是那种功能单一的场景，比如人脸识别打卡系统等。我们常用的操作系统如 Windows/Linux/macOS 等的抽象都支持同时运行很多应用程序，在嵌入式场景是过抽象的。因此，常见的解决方案是仅使用函数库构建单独的应用程序或是用专为应用场景特别裁减过的轻量级内核管理少数应用程序。

### 平台

对于一份用某种编程语言实现的源代码而言，编译器在将其通过编译、链接得到目标文件的时候需要知道程序要在哪个 **平台** (Platform) 上运行。这里 **平台** 主要是指CPU类型、操作系统类型和标准运行时库的组合。 从上面给出的应用程序执行环境栈可以看出：

- 如果用户态基于的内核不同，会导致系统调用接口不同或者语义不一致；
- 如果底层硬件不同，对于硬件资源的访问方式会有差异。特别是 ISA 不同的话，对上提供的指令集和寄存器都不同。

它们都会导致最终生成的目标文件有很大不同。需要指出的是，某些编译器支持同一份源代码无需修改就可编译到多个不同的目标平台并在上面运行。这种 情况下，源代码是 **跨平台** 的。而另一些编译器则已经预设好了一个固定的目标平台。

我们希望能够在另一个体系结构上运行 `Hello, world!`，而与之前的默认平台不同的地方在于，我们将 CPU 架构从 x86_64 换成 RISC-V。

> #### Why？
>
> **为何基于 RISC-V 架构而非 x86 系列架构？**
>
> x86 架构为了在升级换代的同时保持对基于旧版架构应用程序/内核的兼容性，存在大量的历史包袱，现代x86架构的文档长达数千页，且版本众多，且其架构的发展的过程也伴随了现代处理器架构技术的不断发展成熟。，也就是一些对于目前的应用场景没有任何意义，但又必须花大量时间正确设置才能正常使用 CPU 的奇怪设定。为了建立并维护架构的应用生态，这确实是必不可少的，但站在教学的角度几乎完全是在浪费时间。而新生的 RISC-V 架构十分简洁，并且没有背负向后兼容的历史包袱架构文档，需要阅读的核心部分不足百页，且这些功能已经足以用来构造一个具有相当抽象能力的内核了。

## 生成内核镜像

### 编译流程

![image-20210817121751032](资源文件\实验零.assets\image-20210817121751032.png)

我们可以将常说的编译流程细化为多个阶段（虽然输入一条命令便可将它们全部完成）：

1. **编译器** (Compiler) 将每个源文件从某门高级编程语言转化为汇编语言，注意此时源文件仍然是一个 ASCII 或其他编码的文本文件；
2. **汇编器** (Assembler) 将上一步的每个源文件中的文本格式的指令转化为机器码，得到一个二进制的 **目标文件** (Object File)；
3. **链接器** (Linker) 将上一步得到的所有目标文件以及一些可能的外部目标文件链接在一起形成一个完整的可执行文件。

每个目标文件都有着自己局部的内存布局，里面含有若干个段。在链接的时候，链接器会将这些内存布局合并起来形成一个整体的内存布局。此外，每个目标文件都有一个符号表，里面记录着它需要从其他文件中寻找的外部符号和能够提供给其他文件的符号，通常是一些函数和全局变量等。在链接的时候汇编器会将外部符号替换为实际的地址。

我们可以通过 **链接脚本** (Linker Script) 调整链接器的行为，使得最终生成的可执行文件的内存布局符合我们的预期。

> **链接脚本**用于描述链接器处理目标文件和库文件的方式
> 1.合并各个目标文件中的段
> 2.重定位各个段的起始地址
> 3.重定位各个符号的最终地址
>
> 每个链接都由一个链接脚本控制。 该脚本使用链接器命令语言编写。 链接脚本的主要目的是描述如何将输入文件中的各个部分映射到输出文件中，并控制输出文件的内存布局。 大多数链接描述文件仅此而已。

## 加载内核镜像执行

### 可执行文件进入内存并执行

![image-20210818115711049](资源文件\实验零.assets\image-20210818115711049.png)

操作系统内核程序不同于应用程序，由于我们之前移除了标准库依赖和运行时的环境依赖，所以没有相应的系统调用接口（应用程序的执行如下图），内核程序只能选择SBI进行加载。

![image-20210818180850429](资源文件\实验零.assets\image-20210818180850429.png)

对于 OS 内核，一般都将其地址空间放在高地址上。并且在 QEMU 模拟的 RISC-V 中，DRAM 内存的物理地址是从 0x80000000 开始，有 128MB 大小（如想进一步了解，可参看 `qemu/hw/riscv/virt.c` 中的 `VIRT_DRAM` 的赋值，以及[实验指导二中物理内存探测](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/lab-2/guide/part-2.html)小节）。因此接下来我们需要调整程序的内存布局，改变它的链接地址。

>  **程序的内存布局**
>
>  一般来说，一个程序按照功能不同会分为下面这些段：
>
>  - .text 段：代码段，存放汇编代码
>  - .rodata 段：只读数据段，顾名思义里面存放只读数据，通常是程序中的常量
>  - .data 段：存放被初始化的可读写数据，通常保存程序中的全局变量
>  - .bss 段：存放被初始化为 0 的可读写数据，与 .data 段的不同之处在于我们知道它要被初始化为 0，因此在可执行文件中只需记录这个段的大小以及所在位置即可，而不用记录里面的数据，也不会实际占用二进制文件的空间
>  - Stack：栈，用来存储程序运行过程中的局部变量，以及负责函数调用时的各种机制。它从高地址向低地址增长
>  - Heap：堆，用来支持程序**运行过程中**内存的**动态分配**，比如说你要读进来一个字符串，在你写程序的时候你也不知道它的长度究竟为多少，于是你只能在运行过程中，知道了字符串的长度之后，再在堆中给这个字符串分配内存
>
>  内存布局，也就是指这些段各自所放的位置。一种典型的内存布局如下：
>
>  ![../_images/MemoryLayout.png](资源文件\实验零.assets\MemoryLayout.png)
>
>  代码部分只有代码段 `.text` 一个段，存放程序的所有汇编代码。
>
>  数据部分则还可以继续细化：
>
>  - 已初始化数据段保存程序中那些已初始化的全局数据，分为 `.rodata` (read only data) 和 `.data` 两部分。前者存放只读的全局数据，通常是一些常数或者是常量字符串等；而后者存放可修改的全局数据。
>  - .bss 段：存放被初始化为 0 的可读写数据，与 .data 段的不同之处在于我们知道它要被初始化为 0，因此在可执行文件中只需记录这个段的大小以及所在位置即可，而不用记录里面的数据，也不会实际占用二进制文件的空间；
>  - **堆** (heap) 用来支持程序运行过程中内存的动态分配，比如说你要读进来一个字符串，在你写程序的时候你也不知道它的长度究竟为多少，于是你只能在运行过程中，知道了字符串的长度之后，再在堆中给这个字符串分配内存内存布局；
>  - 栈区域 stack 不仅用作函数调用上下文的保存与恢复，每个函数作用域内的局部变量也被编译器放在它的栈帧内。来存储程序运行过程中的局部变量，以及负责函数调用时的各种机制。它从高地址向低地址增长。
>
>  > **局部变量与全局变量**
>  >
>  > 在一个函数的视角中，它能够访问的变量包括以下几种：
>  >
>  > - 函数的输入参数和局部变量：保存在一些寄存器或是该函数的栈帧里面，如果是在栈帧里面的话是基于当前 sp 加上一个偏移量来访问的；
>  > - 全局变量：保存在数据段 `.data` 和 `.bss` 中，某些情况下 gp(x3) 寄存器保存两个数据段中间的一个位置，于是全局变量是基于 gp 加上一个偏移量来访问的。
>  > - 堆上的动态变量：本体被保存在堆上，大小在运行时才能确定。而我们只能 *直接* 访问栈上或者全局数据段中的 **编译期确定大小** 的变量。 因此我们需要通过一个运行时分配内存得到的一个指向堆上数据的指针来访问它，指针的位宽确实在编译期就能够确定。该指针即可以作为局部变量 放在栈帧里面，也可以作为全局变量放在全局数据段中。

### 使用 OpenSBI 提供的服务

OpenSBI 实际上不仅起到了 bootloader 的作用，还为我们提供了一些底层系统服务供我们在编写内核时使用，以简化内核实现并提高内核跨硬件细节的能力。这层底层系统服务接口称为 SBI（Supervisor Binary Interface），是 S Mode 的 OS 和 M Mode 执行环境之间的标准接口约定。

参考 [OpenSBI 文档](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#legacy-sbi-extension-extension-ids-0x00-through-0x0f) ，我们会发现里面包含了一些以 C 函数格式给出的我们可以调用的接口。

上一节中我们的 `console_putchar` 函数类似于调用下面的接口来实现的：

```c++
void sbi_console_putchar(int ch)
```

而实际的过程是这样的：运行在 S 态的 OS 通过 ecall 发起 SBI 调用请求，RISC-V CPU 会从 S 态跳转到 M 态的 OpenSBI 固件，OpenSBI 会检查 OS 发起的 SBI 调用的编号，如果编号在 0-8 之间，则进行处理，否则交由我们自己的中断处理程序处理（暂未实现）。想进一步了解编号在 0-8 之间的系统调用，请参考看 [OpenSBI 文档](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#function-listing-1)。

> SBI调用编号0：
>
> ```c++
> void sbi_set_timer(uint64_t stime_value)
> ```
>
> 为`stime_value`时间后的下一个事件安排时钟。此功能还清除待处理的定时器中断位。
>
> SBI调用编号1：
>
> ```c++
> void sbi_console_putchar(int ch)
> ```
>
> 将当前的数据写入`ch`传入调试控制台
>
> SBI调用编号2：
>
> ```c++
> int sbi_console_getchar(void)
> ```
>
> 从调试控制台中读取一个字节，成功时返回字节，失败返回-1.
>
> SBI调用编号3：
>
> ```c++
> void sbi_clear_ipi(void)
> ```
>
> 清除等待的处理器间中断，处理器间中断只有在SBI援引的硬件线程中被清除
>
> SBI调用编号4：
>
> ```c++
> void sbi_send_ipi(const unsigned long *hart_mask)
> ```
>
> 把一个处理器间中断发送到所有被定义在`hart_mask`中的硬件线程.像软中断那样，处理器间中断在接受到的硬件线程中显示.
>
> SBI调用编号5：
>
> ```c++
> void sbi_remote_fence_i(const unsigned long *hart_mask)
> ```
>
> 指示远程的硬件线程执行指令
>
> SBI调用编号6：
>
> ```c++
> void sbi_remote_sfence_vma(const unsigned long *hart_mask,
>                         unsigned long start,
>                         unsigned long size)
> ```
>
> 指示远程的硬件线程执行一个或多个指令，包含从开始到结尾范围内的虚拟地址
>
> SBI调用编号7：
>
> ```c++
> void sbi_remote_sfence_vma_asid(const unsigned long *hart_mask,
>                              unsigned long start,
>                              unsigned long size,
>                              unsigned long asid)
> ```
>
> 指示远程的硬件线程执行一个或多个指令，包含从开始到结尾范围内的虚拟地址。这一次只包括给出的线程
>
> SBI调用编号8：
>
> ```c++
> void sbi_shutdown(void)
> ```
>
> 关机

执行 `ecall` 前需要指定 SBI 调用的编号，传递参数。一般而言，`a7(x17)` 为 SBI 调用编号，`a0(x10)`、`a1(x11)` 和 `a2(x12)` 寄存器为 SBI 调用参数：

对于参数比较少且是基本数据类型的时候，我们从左到右使用寄存器 `a0` 到 `a7` 就可以完成参数的传递。具体规范可参考 [RISC-V Calling Convention](https://riscv.org/wp-content/uploads/2015/01/riscv-calling.pdf)。

对于设置寄存器并执行汇编指令的代码编写，已经超出了 Rust 语言的基本描述能力。之前采用的 `global_asm!` 方式在Rust代码中插入汇编代码，还不太方便实现Rust代码与汇编代码的互操作。为有效编写 Rust代码与汇编代码的互操作，我们还有另外一种**内联汇编（Inline Assembly）**方式， 可相对简单地完成诸如把 `u8` 类型的单个字符传给 `a0` 作为输入参数的编码需求。**内联汇编（Inline Assembly）**的具体规范请参考书籍[《Rust 编程》](https://kaisery.gitbooks.io/rust-book-chinese/content/content/Inline Assembly 内联汇编.html)。

输出部分，我们将结果保存到变量 `ret` 中，限制条件 `{x10}` 告诉编译器使用寄存器 `x10`（即 `a0` 寄存器），前面的 `=` 表明汇编代码会修改该寄存器并作为最后的返回值。

输入部分，我们分别通过寄存器 `x10`、`x11`、`x12` 和 `x17`（这四个寄存器又名 `a0`、`a1`、`a2` 和 `a7`） 传入参数 `arg0`、`arg1`、`arg2` 和 `which` ，其中前三个参数分别代表接口可能所需的三个输入参数，最后一个 `which` 用来区分我们调用的是哪个接口（SBI Extension ID）。这里之所以提供三个输入参数是为了将所有接口囊括进去，对于某些接口有的输入参数是冗余的，比如 `sbi_console_putchar` 由于只需一个输入参数，它就只关心寄存器 `a0` 的值。