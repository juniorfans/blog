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
本想像 java 的 classpath 一样，通过在 GOPATH 中增加 ./ 来完成设置，但是发现设置这个之后，在 idea 中会认为这个 ./ 是 C:/windows/system32，并不是当前工程的目录，无法达到我们的目的。
偶然间看到 idea 有一个 environment 设置，设置之后，即使重启也无法达到我们的目的，后来想到，这个环境变量可能是在调试时设置给应用程序使用的，并不是 idea 使用的。经过验证，确实如此。
看来目前只能老老实实地设置 GOPATH。


