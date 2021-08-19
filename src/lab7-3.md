# 方案解析

<img src="./资源文件/实验七.assets/流程图.png" alt="image-20210531141431138" style="zoom:50%;" />

使用`top.rs`制造负载，并调用`monitor.rs`监控程序，`monitor`程序调用`sys_read`这个系统调用，`sys_read`将从`TaskControlBlock`中获取到的想要监控的内容通过`json`格式传递回`monitor`，`monitor`对这些信息进行解析后输出。成果如下：

<img src="./资源文件/实验七.assets/成果展示.png" alt="image-20210531141431138" style="zoom:100%;" />

监控程序将持续监控40s后退出。

