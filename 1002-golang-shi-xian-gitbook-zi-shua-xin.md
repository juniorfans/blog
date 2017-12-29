#golang 实现 gitbook 自刷新

##概述
gitbook 是安装在本地的一个服务，它可以将本地的 md 文件组织，构建一系列的 html 文件，索引并可以直接在浏览器访问，一般是 4000 端口。
本文分享一个使用 golang 语言写的小工具：用来帮助本地的 gitbook 监测文件变化，重新构建电子书。做这件事有两个动机：
- 1.目前使用的 gitbook 版本信息如下：CLI version: 2.3.2，GitBook version: 3.2.3。 nodejs 版本是v8.9.3
本地的 gitbook 运行良好，只是有一个问题：当 md 文件发生变动后，gitbook watch 到了变动，但是处理失败，自动退出：[git serve can't restart when file changes #1379][1] 。然而按这个文章中的解决办法：使用 gulp 脚本，写入一段逻辑协助 gitbook 刷新。一番尝试未果，因为 gulp 依赖的其它模块安装失败。
- 2.这是一个借 golang 练手的机会。
所以， let's get start!

##原理
逻辑比较简单：
- 首先，我们需要有一个模块用于监控书籍变化。这个好办，我们可以对比前后两次统计书籍目录的统计数据，可以很容易知道是否发生了变化。我们简单的实现为：统计每个文件的字节数，使用一个 map 关联起来。
- 其次，需要结合 gitbook 命令，完成重新构建和重启服务。
  md 书籍的命令是 gitbook build。开启服务命令是 gitbook serve。然而我们缺少一个停止命令。这里需要研究一下了。
  gitbook 由 npm 完成安装，由 nodejs 去启动。一般我们可以在类似这个路径下找到 gitbook.cmd：
  C:\Users\edgarlli.WEBANK\AppData\Roaming\npm。gitbook.cmd 描述了 nodejs 去启动 gitbook.js。一个比较粗鲁的办法是，直接关闭 node.exe 这个进程，那么显然 gitbook 服务也将停止。但这个方法可能不太好，原因是，如果有其它服务也依托于 nodejs，则会有多个 node.exe 进程，全部关闭。较有一定的风险。我们可以通过 gitbook 监听 4000 端口，准确找到其对应的进程再进行关闭。
  
##实现
程序接收两个输入：
- gitbook 书籍目录
- 要监听的文件的后缀名（一般是 md, jpg, png, bmp 等图片后缀）

将下面代码生成的 exe 置于任意一个目录下，使用命令行：F:\dev\monitor>gitbookMonitor.exe ../blog
可以开启监控服务（实际使用时将 ../blog 换成真正的书籍目录）。

话不多说，上代码

```
package main
import (
    "path/filepath"
    "os"
    "fmt"
    "flag"
    "os/exec"
    "time"
    "strings"
)

var cmfFileMap map[string]string

func initCmd(){
    var rebuild_restart_cmd = `@echo off
set dir=%1%

if not exist %dir% (
        echo dir not exsits: %dir%
) else (
        echo dir: %dir%
)

call kill_by_port 4000
call kill_by_port 35729

cd %dir%

call gitbook build
call gitbook serve`

    var kill_by_port_cmd = `@echo off
setlocal enabledelayedexpansion

echo param is : %1%

for /f "delims=  tokens=1" %%i in ('netstat -aon ^| findstr %1%') do (
set a=%%i
goto js
)
:js
taskkill /F /pid "!a:~71,5!"`

    cmfFileMap = make(map[string]string)
    cmfFileMap["rebuild_restart.cmd"]=rebuild_restart_cmd;
    cmfFileMap["kill_by_port.cmd"]=kill_by_port_cmd
}

func createCmdFile(dir string) bool{
    for filename,content := range  cmfFileMap{
        var filePath = dir + "\\" + filename;
        stringToFile(filePath, content)
        fmt.Println("finish create file: ", filePath)
        //check file exsit
        cmdOk,_ := PathExists(filePath);
        if(!cmdOk){
            fmt.Println("create cmd file error")
            return false;
        }
    }
    return true;
}


var lastVersion map[string] int64

func getDirStat(path string, ftypes map[string]bool) map[string]int64 {
    var currentVersion map[string]int64 = make(map[string]int64)
    err := filepath.Walk(path, func(path string, f os.FileInfo, err error) error {
        if f == nil {
            return err
        }
        if f.IsDir() {
            return nil
        }
        fileName := f.Name()
        if (0 == strings.Compare(fileName, "rebuild_restart.cmd") ||
                0 == strings.Compare(fileName, "kill_by_port.cmd")){
            return nil
        }
        fileType := fileName[strings.Index(fileName,".")+1 : len(fileName)]
        if(0 != len(ftypes) && ftypes[fileType]){
            currentVersion[path]=f.Size()
        }

        return nil
    })
    if err != nil {
        fmt.Printf("filepath.Walk() returned %v\n", err)
    }
    return currentVersion
}

func monitor(path string, ftypes map[string]bool) bool{
    changed := false
    current := getDirStat(path, ftypes)
    if(len(current) != len(lastVersion)){
        changed = true
    }

    for key,value := range current{
        if(value != lastVersion[key]){
            changed = true
        }

    }
    lastVersion = current
    return changed;
}

func check(e error) {
    if e != nil {
        panic(e)
    }
}

func stringToFile(filename string, str_content string)  {
    fd,_:=os.OpenFile(filename,os.O_RDWR|os.O_CREATE|os.O_TRUNC,0644)

    fd_content:=str_content
    buf:=[]byte(fd_content)
    fd.Write(buf)
    fd.Close()
}

func PathExists(path string) (bool, error) {
    _, err := os.Stat(path)
    if err == nil {
        return true, nil
    }
    if os.IsNotExist(err) {
        return false, nil
    }
    return false, err
}

func gitbookRebuildAndStart(root string){

    //create file
    if(!createCmdFile("./")){
        fmt.Println("create file error")
        return;
    }
    cmdFile,_ := filepath.Abs("rebuild_restart.cmd")
    execCommand(cmdFile,[]string{root})
}

func execCommand(commandName string, params []string) bool {

    cmd := exec.Command(commandName, params...)

    cmd.Start()

    cmd.Wait()
    return true
}

func main(){
    flag.Parse()
    root := flag.Arg(0)
    ftype := flag.Arg(1)
    if(0 == len(root)){
        root = "./"
    }

    if(0 == len(ftype)){
        ftype = "md,jpg,png,bmp"
    }else{
        ftype = ftype + ",md"//保证至少有 md 文件被监视
    }

    ftypes := make(map[string]bool)
    var types []string = strings.Split(ftype,",")
    for _,ct := range types{
        if(0 != len(ct)) {
            ftypes[ct] = true
        }
    }

    fmt.Println("root directory is: ", root, ", fileType: ", ftypes)

    initCmd()
    lastVersion = getDirStat(root, ftypes)

    for {
        if (monitor(root, ftypes)) {
            fmt.Println("detect file changed ")
            gitbookRebuildAndStart(root)
        }

        time.Sleep(5 * time.Second)
    }

}
```








[1]: https://github.com/GitbookIO/gitbook/issues/1379