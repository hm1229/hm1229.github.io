# 小结

本章完成了动态分配内存的管理和物理内存的管理，我们通过划分出一段静态内存为操作系统实现了动态内存的分配；通过页的管理模式，实现了物理页的分配器。

本章还只是物理内存的管理，后面为了进一步支持多线程的内存管理，我们将在下一章实现内存的虚拟化。

本章的完整代码你可以在仓库`代码\lab2`中找到