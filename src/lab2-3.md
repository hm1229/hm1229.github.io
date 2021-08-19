# 方案解析

![image-20210531142737165](F:\rCore\rcore-tutorial-detail\资源文件\实验二.assets\image-20210531142737165.png)

如果不想被物理内存限制软件的大小，那么必须要实现内存的动态分配。要想实现动态分配，物理内存首先需要选择一种算法来对选择使用伙伴系统来进行内存分配。

在开发的过程中引用了关于伙伴系统的相关依赖。来配合进行内存的分配。主要进行相关的初始化，并为其分配内存空间，就可以进行相关调用。

实际上分配物理内存并不以字节为单位，而是以物理页作为单位。有了分配系统，就可以对物理页的概念进行封装，对每一个物理页帧进行标识。

有了物理页这一概念，就可以实现一个分配器来对其进行分配和回收的操作。这个分配器会以一个PhysicalPageNumber的Range初始化，然后把起始地址记录下来，把整个区间的长度告诉具体的分配器算法，当分配的时候就从算法中取得一个可用的位置作为offset，再加上起始地址返回回去。