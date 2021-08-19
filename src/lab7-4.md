# 实验步骤

## 制造负载`top.rs`

```rust
//user/src/bin/top.rs
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

use user_lib::{exec, exit, fork, get_time, getpid, open,  sleep, OpenFlags};

//mod monitor;
//use monitor::monitor;

#[no_mangle]
pub fn main() {
    let mut pid: isize = 0;
    let runtime = [30, 28, 26, 24, 22, 20, 18, 16, 14, 12, 10, 8, 6, 4, 2];
    let mut child_id: usize = 0;
    for i in 0..5 {
        pid = fork();
        if pid == 0 {
            child_id = i;
            sleep(1000 * (i + 1));
            break;
        }
    }

    if pid == 0 {
        let time_start = get_time();
        let x = 3;
        while true {
            let x = x * x - x * 3;

            if get_time() - time_start > runtime[child_id] * 1000 {
                break;
            }
        }
    } else {
        exec("monitor\0", &[0 as *const u8]);
    }
}
```

- 第18行至第25行程序将自己的进程进行复制，当他是子进程时，跳出循环。当他是父进程的时候，继续fork自己。
- 第27-36行是子进程添加负载的过程
- 第38行父进程调用monitor进行监视

## 监控数据`monitor.rs`

```rust
//user/src/bin/monitor.rs
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;
extern crate alloc;
extern crate serde_json;

use alloc::string::{String, ToString};
use alloc::vec::Vec;
use core::str::from_utf8;
use core::time;
use serde::Deserialize;
use serde_json::from_str;
use user_lib::*;

const PROC_INFO: usize = 3;
const WHITE_SPACE: u8 = 32;
const LOOP_TIMES:usize=30;

#[derive(Copy, Clone, Deserialize)]
pub struct SyscallCount {
    pub syscall_dup: (usize, usize),
    pub syscall_open: (usize, usize),
    pub syscall_close: (usize, usize),
    pub syscall_pipe: (usize, usize),
    pub syscall_read: (usize, usize),
    pub syscall_write: (usize, usize),
    pub syscall_exit: (usize, usize),
    pub syscall_yield: (usize, usize),
    pub syscall_get_time: (usize, usize),
    pub syscall_getpid: (usize, usize),
    pub syscall_fork: (usize, usize),
    pub syscall_exec: (usize, usize),
    pub syscall_waitpid: (usize, usize),
}

#[derive(Deserialize, PartialEq, Eq)]
pub enum TaskStatus {
    Ready,
    Running,
    Zombie,
}

#[derive(Deserialize)]
pub struct ProcInfoDirect {
    // name:
    pub pid: usize,
    pub name: String,
    pub ppid: isize,
    // heap_sz:
    // mem:
    pub cpu_time: usize,
    pub status: TaskStatus,
    pub syscall_cnt: SyscallCount,
}

#[no_mangle]
pub fn main() {
    let mut buf: [u8; 3500] = [WHITE_SPACE; 3500];

    let sart_time = get_time();
    while true {
        sleep(1000);

        read(PROC_INFO, &mut buf);
        let temp = from_utf8(&buf).unwrap().trim();
        let time_sec  = (get_time() - sart_time) / 1000;
        if time_sec > 40{
            exit(0);
        }
        println!("-----------------------------------------------------------------------------------------------------------------------");
        println!("           rCore-Tutorial-v3 Resource Monitor                            Time: {} s", time_sec);
        println!("-----------------------------------------------------------------------------------------------------------------------\n");
        let proc_stat: Vec<ProcInfoDirect> = from_str(&temp).unwrap();
        println!("Process   |   pid   |   ppid   |   status   |   heap   |   mem   |   cpu time  |     syscall    |  times  |  duration  ");
        println!("-----------------------------------------------------------------------------------------------------------------------\n");
        for i in 0..proc_stat.len() {
            if proc_stat[i].status == TaskStatus::Zombie {
                continue;
            }
            //println!("{}   ", proc_stat[i].name);
            print!(
                "{:<15}{:<10}{:<10}",
                proc_stat[i].name, proc_stat[i].pid, proc_stat[i].ppid
            );
            match proc_stat[i].status {
                TaskStatus::Ready=> print!("Ready"),
                TaskStatus::Running => print!("Running"),
                _ => panic!("wrong status"),
            };
            print!("\n{:<82}", "");
            let sycall_cnt = proc_stat[i].syscall_cnt;
            if sycall_cnt.syscall_dup.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_DUP", sycall_cnt.syscall_dup.0, sycall_cnt.syscall_dup.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_open.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_OPEN", sycall_cnt.syscall_open.0, sycall_cnt.syscall_open.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_close.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_CLOSE", sycall_cnt.syscall_close.0, sycall_cnt.syscall_close.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_pipe.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_PIPE", sycall_cnt.syscall_pipe.0, sycall_cnt.syscall_pipe.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_read.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_READ", sycall_cnt.syscall_read.0, sycall_cnt.syscall_read.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_write.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_WRITE", sycall_cnt.syscall_write.0, sycall_cnt.syscall_write.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_exit.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_EXIT", sycall_cnt.syscall_exit.0, sycall_cnt.syscall_exit.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_yield.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_YIELD", sycall_cnt.syscall_yield.0, sycall_cnt.syscall_yield.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_get_time.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_GET_TIME",
                    sycall_cnt.syscall_get_time.0,
                    sycall_cnt.syscall_get_time.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_getpid.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_GETPID", sycall_cnt.syscall_getpid.0, sycall_cnt.syscall_getpid.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_fork.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_FORK", sycall_cnt.syscall_fork.0, sycall_cnt.syscall_fork.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_exec.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_EXEC", sycall_cnt.syscall_exec.0, sycall_cnt.syscall_exec.1
                );
                print!("\n{:<82}", "");
            }
            if sycall_cnt.syscall_waitpid.0 != 0 {
                print!(
                    "{:<18}{:^4}      {}ms",
                    "SYSCALL_WAITPID", sycall_cnt.syscall_waitpid.0, sycall_cnt.syscall_waitpid.1
                );
                print!("\n{:<82}", "");
            }
            println!("");
        }
        print!("-----------------------------------------------------------------------------------------------------------------------\n");
    }
    
}
```

- 第23行定义了一个`syscall`的结构体，里面装有系统调用的次数与时间
- 第47行定义了`ProcInfoDirect`这个结构体，这里面与内核态存储监控数据的结构体是一样的，这样方便后续传输
- 第61行构建了一个由空格组成的缓冲区。
- 第67行调用`syscall_read`系统调用，读取内核中的数据
- 第70-72行当`monitor`运行40s后自动退出
- 第76行将缓冲区里的文本再次转换为结构体数据。
- 第81行如果当前进程是`Zombie`时跳过

## 添加一种新的文件类型`proc.rs`

```rust
//os/src/fs/proc.rs
use super::File;
use alloc::{sync::Arc, vec::Vec};
use alloc::string::{String, ToString};
use crate::task::{SyscallCount, get_info};
use spin::Mutex;
use crate::mm::{
    UserBuffer,
};
use super::TaskStatus;
use super::{Serialize,Deserialize};
use super::to_string;
//#[repr(C)]
pub struct ProcInfo{
    // name:
    pub pid: usize,
    pub name: String,
    pub ppid: isize,
    // heap_sz:
    // mem:
    // pub cpu_time: usize,
    pub status: TaskStatus,
    pub syscall_cnt: Arc<Mutex<SyscallCount>>,
}
#[derive(Serialize)]
pub struct ProcInfoDirect{
    // name:
    pub pid: usize,
    pub name: String,
    pub ppid: isize,
    // heap_sz:
    // mem:
    pub cpu_time: usize,
    pub status:TaskStatus,
    pub syscall_cnt: SyscallCount,
}

pub struct ProcInfoList{
//    pub inner:Vec<ProcInfo>,
}

impl File for ProcInfoList{

    fn readable(&self) -> bool {
        true
    }

    fn writable(&self) -> bool {
        false
    }

    /// 这个函数做的事是把信息放到buffer里面
    fn read(&self, mut buf: UserBuffer) -> usize {
        let mut vec_proc_info:Vec<ProcInfo>= get_info().0;
        let mut vec_proc_info_direct:Vec<ProcInfoDirect>=Vec::new();
        for item in vec_proc_info{
            let direct_syscall_count = item.syscall_cnt.lock().clone();
            vec_proc_info_direct.push(ProcInfoDirect{
                pid:item.pid,
                name:item.name,
                ppid:item.ppid,
                cpu_time:item.cpu_time,
                status:item.status,
                syscall_cnt:direct_syscall_count,
            })
        }
        let json = to_string(&vec_proc_info_direct).unwrap().clone();
        let mut cursor = 0;
        println!("json size: {}",json.as_bytes().len());
        for test in buf.into_iter(){
            unsafe{*test = json.as_bytes()[cursor];}
            cursor+=1;
            if cursor==json.as_bytes().len(){
                break;
            }
        }
        cursor
    }

    fn write(&self, buf: UserBuffer) -> usize {
        println!("never gonna give you up");
        114514
    }
}
```

- 第26行构建一个和用户态相同的结构体文件
- 第38-81行给`ProcInfoList`添加`File`接口，并实现这些功能
- 第54行调用`get_info`获取信息
- 第67行将信息写入`json`文件
- 第70-78行将`json`文件写入缓冲区

## 添加文件描述符

```rust
//os/src/task/task.rs impl TaskControlBlock
pub fn new(elf_data: &[u8]) -> Self {
    // memory_set with elf program headers/trampoline/trap context/user stack
    let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
    let trap_cx_ppn = memory_set
    .translate(VirtAddr::from(TRAP_CONTEXT).into())
    .unwrap()
    .ppn();
    // alloc a pid and a kernel stack in kernel space
    let pid_handle = pid_alloc();
    let kernel_stack = KernelStack::new(&pid_handle);
    let kernel_stack_top = kernel_stack.get_top();
    let name = String::new();
    // push a task context which goes to trap_return to the top of kernel stack
    let task_cx_ptr = kernel_stack.push_on_top(TaskContext::goto_trap_return());
    let task_control_block = Self {
        pid: pid_handle,
        kernel_stack,
        ppid: -1,
        inner: Mutex::new(TaskControlBlockInner {
            trap_cx_ppn,
            base_size: user_sp,
            task_cx_ptr: task_cx_ptr as usize,
            task_status: TaskStatus::Ready,
            name,
            memory_set,
            parent: None,
            children: Vec::new(),
            exit_code: 0,
            fd_table: vec![
                // 0 -> stdin
                Some(Arc::new(Stdin)),
                // 1 -> stdout
                Some(Arc::new(Stdout)),
                // 2 -> stderr
                Some(Arc::new(Stdout)),
                // 3 -> proc_info
                Some(Arc::new(ProcInfoList {})),
            ],
            start_time: 0,
            cpu_time: 0,
            syscall: Arc::new(Mutex::new(SyscallCount{
                syscall_dup: (0, 0),
                syscall_open: (0, 0),
                syscall_close: (0, 0),
                syscall_pipe: (0, 0),
                syscall_read: (0, 0),
                syscall_write: (0, 0),
                syscall_exit: (0, 0),
                syscall_yield: (0, 0),
                syscall_get_time: (0, 0),
                syscall_getpid: (0, 0),
                syscall_fork: (0, 0),
                syscall_exec: (0, 0),
                syscall_waitpid: (0, 0),
            }))
            //start_run_stamp:0,
        }),
    };
    // prepare TrapContext in user space
    let trap_cx = task_control_block.acquire_inner_lock().get_trap_cx();
    *trap_cx = TrapContext::app_init_context(
        entry_point,
        user_sp,
        KERNEL_SPACE.lock().token(),
        kernel_stack_top,
        trap_handler as usize,
    );
    task_control_block
}
```

- 在第38行出添加新的文件描述符

## 获取信息`get_info`

```rust
//os/src/task/task.rs impl TaskControlBlock
pub fn get_info(&self)-> ProcInfo{
    let pid = self.getpid();
    let status = self.inner.lock().get_status();
    let name = self.get_name();
    let ppid = self.get_ppid();
    let cpu_time = self.get_cpu_time();
    let syscall_cnt = self.get_syscall_cnt();
    ProcInfo{
        pid,
        name,
        status,
        ppid,
        cpu_time,
        syscall_cnt,
    }
}
```

传回`TaskControlBlock`系统的数据

## 保存进程名称`set_name`

在程序执行时，把名称存入进程控制块。

## 系统调用计数`syscall_cnt`

构建结构体`SyscallCount`

```RUST
//os/src/syscall/process.rs
#[derive(Copy, Clone, Serialize)]
pub struct SyscallCount{
    pub syscall_dup: (usize, usize),
    pub syscall_open: (usize, usize),
    pub syscall_close: (usize, usize),
    pub syscall_pipe: (usize, usize),
    pub syscall_read: (usize, usize),
    pub syscall_write: (usize, usize),
    pub syscall_exit: (usize, usize),
    pub syscall_yield: (usize, usize),
    pub syscall_get_time: (usize, usize),
    pub syscall_getpid: (usize, usize),
    pub syscall_fork: (usize, usize),
    pub syscall_exec: (usize, usize),
    pub syscall_waitpid: (usize, usize),
}
```

目前实现了13个系统调用的统计，元组内第一个元素存储系统调用次数，第二个元素存储系统调用时间。

示例：

```rust
//os/src/syscall/process.rs
pub fn sys_yield() -> isize {
    let start_time = get_time_ms();
    suspend_current_and_run_next();
    let current_task = current_task().unwrap();
    let inner = current_task.acquire_inner_lock();
    let mut syscall_inner = inner.syscall.lock();
    syscall_inner.syscall_yield.0 += 1;
    syscall_inner.syscall_yield.1 += get_time_ms() - start_time;
    0
}
```

其他12个系统调用仿照当前实现写即可。