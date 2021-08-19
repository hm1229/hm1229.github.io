# rCore-Tutorial-detail

#### 介绍

本项目是对于清华大学 rCore-Tutorial 项目的知识点补充与引申

本文的目的是使用Rust语言去实现一个基于RISC-V架构的操作系统内核，利用Rust语言的关键特性使得操作系统内核具备线程安全的特性。该操作系统内核是一个具备了操作系统基础的核心机制的微内核，基于负责与硬件通信的SBI，包括引导程序、内核加载器bootloader、中断、内存、进程、线程、文件系统等模块。

本文首先对相关研究现状进行了分析，对于Rust语言、RISC-V架构及其汇编语言进行简要介绍；自上而下对阐述了整个内核架构的设计方案，在技术路线中给出了操作系统内和各个模块之间的关系与实现思路；然后对于操作系统内核进行了自下而上的实现，并给出了关键代码及其解释，最后经过细化的系统测试，验证功能的正确性。

本文基于Ubuntu操作系统和Rust工具链作为开发环境，使用Rust语言以及RISC-V架构的汇编语言的混合编程进行开发，利用QEMU虚拟机进行调试和测试所实现的操作系统内核功能。



目前本文对示例代码进行了修改，调整了复现逻辑，增加了相关知识的拓展，增加了难点的讲解。

力求做到**从零开始实现操作系统内核**，截至目前全部实验**均可复现**。



#### 目录

1. 环境部署
2. 实验零
3. 实验一
4. 实验二
5. 实验三
6. 实验四
7. 实验五
8. 实验六
9. 实验七



#### TO DO

- 添加更为详细的代码注释
- 添加知识点，丰富文档内容
- 继续修改原文档逻辑



### [文档](hm1229.github.io)可见链接

### [性能监控视频](链接：https://pan.baidu.com/s/1NFPMJqs2hDexR-GuFXazWA )可见链接

提取码：jwy1



#### 鸣谢

感谢清华团队对整个 rCore 项目做出的开拓性贡献



#### 安装教程

1. 安装完整 git 环境

2. 加入 rCore学习小组

3. 使用 `git clone https://gitee.com/rCore-Tutorial-detail/r-core-tutorial-detail.git`  

   或 `git clonegit@gitee.com:rCore-Tutorial-detail/r-core-tutorial-detail.git` 克隆代码到本地



#### 使用说明

* 本项目根据[《rCore-Tutorial V3》](https://rcore-os.github.io/rCore-Tutorial-deploy/) 与[《rCore-Tutorial-Book 第三版》](http://wyfcyx.gitee.io/rcore-tutorial-book-v3/)综合而成，如无法复现，请参考原开发文档
* 图片等资源文件以相对路径存储在 `./资源文件/${filename}.assets` 




#### 参与贡献

1.  Fork 本仓库
2.  新建分支
3.  提交代码
4.  新建 Pull Request

#### 相关链接

操作系统网课：https://www.xuetangx.com/learn/thu08091002729/thu08091002729/5883981/video/9244836

笔记仓库：https://codechina.csdn.net/weixin_53305890/rcore-tutorial-detail

官方文档：https://rcore-os.github.io/rCore-Tutorial-deploy/notes/log.htmlrust

语法1（英文版）：https://doc.rust-lang.org/nightly/std/index.htmlrust

语法2（译文版）：https://kaisery.github.io/trpl-zh-cn/ch01-02-hello-world.html

#### 成员

指导教师：赵霞  

学生：彭淳毅 路博雅