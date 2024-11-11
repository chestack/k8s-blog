## Golang应用 遇到的问题

一般中大型项目的build配置都被包在Makefile 或者 Dockerfile里, 之前没有仔细看过 goalng build相关的参数, 直到陆续遇到几个相关问题, 这里借由相关问题, 把golang 关于 CGO_ENABLED，静态编译, 以及两者之间的关系做下整理.


#### 问题1
```
netplugin failed: "/opt/cni/bin/loopback: /usr/lib64/libc.so.6: version `GLIBC_2.34' not found (required by /opt/cni/bin/loopback)
```
这个问题是 cni plugin binary 执行阶段报错, 报错信息相对明确, 依赖的.so(动态库)找不到, 说明这个loopback是 非静态编译的。



#### 问题2 
[net: prefer /etc/hosts over DNS when no /etc/nsswitch.conf is present](https://github.com/golang/go/issues/35305)


CGO_ENABLED=0的情况下, golang 代码访问域名"localhost" 不查看/etc/hosts里配置的 127.0.0.1, 而是直接走DNS查询。

这个问题涉及到: glibc的逻辑，golang net库考虑和glibc兼容性的问题，alpine 使用musl libc的问题, 详细背景记录在 [这里](https://stackoverflow.com/questions/56937331/golangs-handling-of-localhost/74446140#74446140)



#### 以上两个问题, 引申的一些疑问
- CGO具体是什么原理, 为什么默认 CGO_ENABLED=1?
- CGO具体是如何影响 静态/动态编译呢? 两者之间有 因果/等价关系吗？
- Golang默认是静态还是动态编译？ 什么场景应该采用静态编译?


## 源码从编译到运行，关键阶段和产物

- 编译, 输入是预处理之后的代码, 输出是 .o文件(目标文件), 但其中的外部依赖还未解析，即尚未链接.
- 链接, 输入是 .o文件，输出是可执行文件，链接器（Linker）将各个 .o 文件和库文件结合起来，生成可执行文件. 静态库文件.a(archive), 动态库文件 .so(shared object)。
- 运行, ELF文件(Executable and Linkable Format), ELF 文件格式定义了标准化的结构, 使链接器可以从目标文件或库中提取符号信息和代码片段，将它们组合在一起，生成一个最终的程序文件.


## CGO_ENABLED

关于CGO的介绍, 参照[golang官方blog](https://go.dev/blog/cgo)
```
Cgo lets Go packages call C code.

package rand

/*
#include <stdlib.h>
*/
import "C"

func Random() int {
    return int(C.random())
}

func Seed(i int) {
    C.srandom(C.uint(i))
}
```

#### 为什么需要CGO

- C作为老大哥级别的语言, 走过的路更多, 且更贴近底层, 对于一些需要和OS打交道的工作效率更高、支持更好。Golang可以直调用C的实现，运行更快、软件体积更小.
- 业务需要, 可能某些场景 被调用方只有C的实现, golang应用就需要调用C的代码
- 为什么默认 CGO_ENABLED=1， [why CGO_ENABLED=1 in default](https://stackoverflow.com/questions/64531437/why-is-cgo-enabled-1-default)

Golang里 大部分标准库都是go语言实现的, 但有些一些库(库里面的部分功能)在build时可以选择cgo-based实现 或者 pure go实现, 常见的如

- os/users 库
- net库中的 [Name Resolution功能模块](https://pkg.go.dev/net#hdr-Name_Resolution), 这个在cni、kubernetes中比较常见，也是我们遇到的问题.


## 静态编译

上面介绍了CGO以及CGO_ENABLED, 现在让我来想想CGO 和 静态编译的关系, 我们可以分几种情况分析


如果golang应用 不调用C代码, 或者不使用cgo-based的库(os/users、net), 则golang应用不需要
```
package main

import "fmt"

func main() {
 fmt.Println("hello, world")
}
```

```
$go build -o hw main.go

$ldd hw
$    not a dynamic executable
```
可以看到, 上面的hw不依赖任何动态库, 也就是静态编译的，也就是在默认情况下(CGO_ENABLED=1)，Go会优先使用**静态链接**的方式.



如果我们在golang代码中，使用了cgo-based的库(os/users、net), 默认情况下(CGO_ENABLED=1)，build出来的二进制会依赖 glibc, 也就是开头的 loopback的那个例子, 因为loopback是依赖"net"库的。 这种情况 只要显示配置CGO_ENABLED=0, 就可以不依赖glibc, 可以实现静态编译


如果业务上golang必须调用C代码, 那还能做到静态编译吗, 其实也是可以的。

大概原理是
1. 先把C代码编译为 .a静态库
2. build golang代码时，通过flags指定静态编译 ```-extldflags "-static"```


还有一种情况, golang应用 --依赖--> 第三方go module(非golang标准库) --依赖--> C语言, 常见的例子是go-sqlite3, 具体可以参照 [Go编译的几个细节，连专家也要停下来想想](https://mp.weixin.qq.com/s/wKqcAAU7aq-i34ZhW7NH9A)。

解决思路是 要准备go-sqlite3 依赖的C语言静态库, 再做golang应用的静态编译。


### 为什么需要静态编译

静态编译的好处是更好的兼容性, 真正的 build once run anywhere。

云原生场景下, 如果build出来的golang应用是container方式运行, 可以保证docker image包含应用需要的动态库一般不会出问题, 以下两种情况会有问题

- build image 和 runtime image不一致, 常见的 runtime image采用小size的 alpine
- build 出来的二进制 run在HOST上, HOST上因操作系统的差动态库版本无法保障, 例如 cni plugin 和 kubelet都是run在host上的二进制.