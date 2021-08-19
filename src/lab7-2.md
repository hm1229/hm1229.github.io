# 知识储备

## sys_read的实现原理

rCore-Tutorial的用户库已经帮助我们将sys_read系统调用封装为read()函数，我们可以直接在用户态程序中使用：

```rust
read()
```

## 利用json传输数据

`sys_read`系统调用通过一个由`&[u8]`数组组成的缓冲区来传递数据。我们很容易想到一种方便且安全的传输方案：内核将收集到的数据转换成json文本，在缓冲区中放入json文本，用户态得到json文本并进行解析。

当然，我们也不是非得用json。自己定义一种传输格式或者干脆直接传输结构体也是可以的。

## 什么是json

`json`全名叫`JavaScript Object Notation`即`javascript`对象标记。它最早是用来描述`javascript`对象用的（`javascript`中一切皆对象），比如：

```json
{
    "name": "John Doe",
    "age": 43,
    "address": {
        "street": "10 Downing Street",
        "city": "London"
    },
    "phones": [
        "+44 1234567",
        "+44 2345678"
    ]
}
```

我们可以用同样的方式来表达rust的结构体。

rust已经有功能完善，资源占用较低且支持`no_std`的`serde_json`库，因此我们不用劳神写json的生成和解析函数了，用现成的即可。

## 应用json来传输数据

### 添加`serde_json`库

在Cargo.toml中

```
[dependencies]

...

serde_json = { version = "1.0", default-features = false, features = ["alloc"] }

serde = { version = "1.0",default-features = false,  features = ["derive"] }


```

## 解析

用户态从缓冲区中取出文本形式的`json`数据之后，使用`serde_json`库，将缓冲区内的字节变回`utf8`文本，再从`utf8`文本还原回`json`文本，再将`json`文本变回`rust`结构体。

