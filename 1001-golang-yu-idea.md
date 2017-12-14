#golang 与 idea

#概述
本文更像是一篇记录文，记录使用 idea 作为编辑器写 golang 踩过的坑。

#安装
idea 安装网上可找的资料很多，不阐述。
go lang sdk 安装也比较容易，直接去官网下载最新版即可。安装完成后需要设置两个环境变量: GOPATH, GOROOT，golang 编译时会去这两个目录下找目标，GOROOT 即是 golang 的安装目录，GOPATH 是项目目录。
为了在 idea 里面联动 golang 编译器，代码自动完成，工程管理，需要安装 idea 的 golang-plugin，这个我们可以直接在 iead 的插件管理里面在线安装，或是下载 zip 包手动安装，问题不大。

#编译
不使用 idea 的编译方式，上一篇文章已经详细说明了。
使用 idea 编译，容易碰到文件找不到的问题，比如 import 自己工程的某个目录报找不到，这个错是 idea 报出的，我们需要让 idea 找到它：我们需要将本工程起始目录设置到 GOPATH 中。注意设置成功后需要重启 idea 生效.
本想像 java 的 classpath 一样，通过在 GOPATH 中增加 ./ 来完成设置，但是发现设置这个之后，在 idea 中会认为这个 ./ 是 C:\Windows\System32\，并不是当前工程的目录（可能这个 ./ 被解析为 cmd.exe 的所在的目录了），无法达到我们的目的。
偶然间看到 idea 有一个 environment 设置，设置之后，即使重启也无法达到我们的目的，后来想到，这个环境变量可能是在调试时设置给应用程序使用的，并不是 idea 使用的。经过验证，确实如此。
看来目前只能老老实实地设置 GOPATH。这样有一个担忧，如果有不同的 GOPATH ，而这些目录下有相同的工程名，而各个工程目录都大概一致，src/main，会不会编译时发生错乱。实际上不用担忧这个问题，由于 idea 是清楚知道当前正打开的工程/文件的，所以不会发生错误。
我们来做一个有意思的试验:
- 1.GOPATH 下有一个名为 sorter 的 golang 工程（即上一篇文章中的 sorter），现在复制 sorter 放到任何一个非 GOPATH 目录下。将这两个 sorter 工程都在 idea 中打开，这两个工程都可以正确地在 idea 中编译和运行。
- 2.再在非 GOPATH 下建一个工程，只有一个非常简单的 golang 文件 main.go 如下:

```
package main

import "fmt"


func main(){
	fmt.Println("success in BGo")
}
```
在 idea 中可以正确编译且运行。这很容易理解，因为在编译这个文件时，idea 不需要找其它依赖，所以可以顺畅编译运行。
- 3.做一点更改，在 2 中的工程中增加一个文件 tool.go，并且让 main.go 依赖它，即：

```
//main.go
package main

import "fmt"
import "tools"


func main(){
	fmt.Println("success in BGo")
	tools.Say()
}


//tool.go
package tools

import "fmt"

func Say(){
	fmt.Println("in tools.say: ok ")
}
```
- 4.现在发现 main.go 编译错误，报错信息如下：

```
cannot find package "tools" in any of:
	C:\Program Files\go\src\tools (from $GOROOT)
	E:\Go\src\tools (from $GOPATH)
	E:\Go\src\sorter\src\tools
```

- 5.第4步没有通过编译这符合预期，因为 import "tools" 是无效的， idea /golang 编译器找不到此文件。再次强调，golang 中的 import 与 c/c++ 中的 include 是不同的，它更像是 java 中的 import 的概念：全局地从 GOPATH 中引入那个路径下的所有 golang 文件(一个文件算一个模块)。

- 6.与第 1 点相呼应，为什么位于非 GOPATH 下的 sorter 可编译成功呢，它也引用了自己目录下的其它 golang 文件。原因不那么明显：这是因为它依赖的是位于 GOPATH 目录下的那个 sorter 的模块！所以我们要尽量避免在同一 GOPATH 中有相同的工程名字，这有可能导致依赖的混乱。






