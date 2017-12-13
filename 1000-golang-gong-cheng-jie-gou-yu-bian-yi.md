#golang 工程结构与编译

#概述
本文是一篇对 go build/install 命令以及目录结构试验文，目的是得出 golang 编译规则。我们的手段是不断调整目录结构和编译命令，通过得出的结果进行思考分析。
#结论先行
- 0.go build 与 go install 命令对于文件，目录的规则是一样的。
- 1.go build 后面可以接文件, 此时，从当前目录出发找到指定的文件编译。
- 2.go build 后面可以接目录(看起来像是包名，但其实是目录). 从 GoPath 开始找到这个目录，然后编译下面所有的 go 文件. 再强调一次： go build 后面的目录并 非是相对当前目录的，它是相对 GoPath 的。
- 3.go build  后面接目录，只会编译此目录下的 go 文件，并不会递归处理子目录下的 go 文件。
- 4.go build 后面可以不接参数，表示编译当前目录下的所有 go 文件. 当前目录下若没有 go 文件则会编译报错。
- 5.当一个目录下存在引入不同包的 go 文件，使用 go build 会报错。
- 6.GOPATH 下的目录下的 src 目录可以被自动忽略，这是因为， go 语言假设所有工程的源码都放于 src 文件夹下。看后面的举例即可明白。
- 7.main 包下的 go 源文件必须含有 main 函数

. 折叠
真的吗

#试验准备
现在有一个 go 工程目录结构如下图所示

![](/assets/golang/go-lang-dir.png)

代码内容如下

```go
//sorter.go
package main

import "bufio"
import "flag"
import "fmt"
import "io"
import "os"
import "strconv"
import "time"
import "algorithm/bubblesort"
import "algorithm/qsort"

var infile *string = flag.String("i", "unsorted.dat", "File contains values for sorting")
var outfile *string = flag.String("o", "sorted.dat", "File to receive sorted values")
var algorithm *string = flag.String("a", "qsort", "Sort algorithm")

func readValues(infile string) (values []int, err error) {
	file, err := os.Open(infile)
	if err != nil {
		fmt.Println("Failed to open the input file ", infile)
		return
	}
	defer file.Close()
	br := bufio.NewReader(file)
	values = make([]int, 0)
	for {
		line, isPrefix, err1 := br.ReadLine()
		if err1 != nil {
			if err1 != io.EOF {
				err = err1
			}
			break
		}
		if isPrefix {
			fmt.Println("A too long line, seems unexpected.")
			return
		}
		str := string(line) // 转换字符数组为字符串
		value, err1 := strconv.Atoi(str)
		if err1 != nil {
			err = err1
			return
		}
		values = append(values, value)
	}
	return
}

func writeValues(values []int, outfile string) error {
	file, err := os.Create(outfile)
	if err != nil {
		fmt.Println("Failed to create the output file ", outfile)
		return err
	}

	defer file.Close()
	for _, value := range values {
		str := strconv.Itoa(value)
		file.WriteString(str + "\n")
	}
	return nil
}

func main() {
	flag.Parse()
	if infile != nil {
		fmt.Println("infile =", *infile, "outfile =", *outfile, "algorithm =",
			*algorithm)
	}
	values, err := readValues(*infile)
	if err == nil {
		t1 := time.Now()
		switch *algorithm {
		case "qsort":
			qsort.QuickSort(values)
		case "bubblesort":
			edgarlli.BubbleSort(values)
		default:
			fmt.Println("Sorting algorithm", *algorithm, "is either unknown or unsupported.")
		}
		t2 := time.Now()
		fmt.Println("The sorting process costs", t2.Sub(t1), "to complete.")
		writeValues(values, *outfile)
	} else {
		fmt.Println(err)
	}
}


```


```
//bubblesort.go
package edgarlli


import "fmt"

func BubbleSort(values []int) {
	flag := true
	for i := 0; i < len(values)-1; i++ {
		flag = true
		for j := 0; j < len(values)-i-1; j++ {
			if values[j] > values[j+1] {
				values[j], values[j+1] = values[j+1], values[j]
				flag = false
			} // end if
		} // end for j = ...
		if flag == true {
			break
		}
	} // end for i = ...
}

func main(){
	fmt.Println("a fake main in bubblesort")
}
```


```
//qsort.go
package qsort

func quickSort(values []int, left, right int) {
	temp := values[left]
	p := left
	i, j := left, right
	for i <= j {
		for j >= p && values[j] >= temp {
			j--
		}
		if j >= p {
			values[p] = values[j]
			p = j
		}
		if values[i] <= temp && i <= p {
			i++
		}
		if i <= p {
			values[p] = values[i]
			p = i
		}
	}
	values[p] = temp
	if p-left > 1 {
		quickSort(values, left, p-1)
	}
	if right-p > 1 {
		quickSort(values, p+1, right)
	}
}
func QuickSort(values []int) {
	quickSort(values, 0, len(values)-1)
}

```


注意代码中的几点:
- bubblesort.go 和 bubblesort_test.go 都 package 到 bubblesort 包.
- qsort.go 和 qsort_test.go 都 package 到 qsort 包.
- sorter.go 则 package 到 main 包.

我们将顶层的 sorter 目录设置为 GOPATH，开始试验。

#试验进行时

###go build 单个文件的情况
- ####如何找到文件
使用 go build 编译单个文件，go 编译器是如何找到文件的呢？
如下图的试验，我们可以直接在 GOPATH 下使用相对目录编译 sorter.go，或者到那个文件所在目录，直接使用 go build 编译那个文件，或者直接 go build 带参数，都是可以的。抑或是在非 GOPATH 下使用相对目录编译那个文件，这一点说明了编译单个文件时，查找文件的方式是基于当前目录的，与 GOPATH 无关。

![编译时如何找到单个文件](/assets/golang/go-build-single-file.png)

需要注意还有两点：
编译成功后生成的文件会放置在当前位置。
若 go build 不指定参数则生成文件名为 main.exe。如果我们编译不含 main 的文件会得到什么结果呢？

- ####编译不含main的文件
当我们要编译的 go 文件的包名不含 main 函数，会是怎样的呢？继续试验。qsort.go **没有 main 函数，编译成功后它没有在任何地方生成任何文件**。而 sorter.go 中有 main 函数，它在当前目录下生成了 exe 文件。

![](/assets/golang/go-build-no-main.png)

- ####GOPATH 下的 src 目录被自动加上
我们可以在 sorter 目录下，直接使用 ***go build main*** 来编译所有 main 目录下所有的 go 文件, sorter 下的 src 不用给出。
 
![](/assets/golang/go-build-gopath-auto-src.png)

- ####如何编译整个工程
遗憾，目前没有找到方法。但有一个替代的办法：我们一次性编译 main 目录下所有的文件，它们会将依赖的文件，库，都编译.（再强调: 上面说的不是 main 包，是 main 目录，这是因为 go build 后面的不是包名而是目录名）
如上一个试验，我们使用 go build main 即可。我们进一步，只要设置了 GOPATH，我们可以在任何目录下 go build main 来编译所有GOPATH 下的 main 目录。

![](/assets/golang/go-build-gopath-main-anywhere.png)


- ####main 函数不在 main 包
如果一个 go 文件 package 不是 main 包，但含有 main 函数，会有怎样的编译结果？我们更改 qsort.go 文件，增加 main 函数，如下:

```
//qsort.go 包名不是 main 但含有 main 函数
package qsort

import "fmt"

func quickSort(values []int, left, right int) {
	temp := values[left]
	p := left
	i, j := left, right
	for i <= j {
		for j >= p && values[j] >= temp {
			j--
		}
		if j >= p {
			values[p] = values[j]
			p = j
		}
		if values[i] <= temp && i <= p {
			i++
		}
		if i <= p {
			values[p] = values[i]
			p = i
		}
	}
	values[p] = temp
	if p-left > 1 {
		quickSort(values, left, p-1)
	}
	if right-p > 1 {
		quickSort(values, p+1, right)
	}
}
func QuickSort(values []int) {
	quickSort(values, 0, len(values)-1)
}

func main(){
	fmt.Println("a fake main in qsort")
}
```

编译结果如下图：
![](/assets/golang/go-lang-package-nomain-main_func.png)

编译ok 没有生成可执行程序，目前猜测是，当 main 函数不在 main 包里会被作为普通的函数，不被当成是程序的启动点。


- ####main 包下没有main函数
如果一个 go 文件 package 的是 main 包，但没有 main 函数，会有怎样的编译结果？我们再次更改 qsort.go 文件，将包名改为 main 但不增加 main 函数，如下:

```
//qsort.go 打包到 main 包但不含 main 函数
package main

import "fmt"

func quickSort(values []int, left, right int) {
	temp := values[left]
	p := left
	i, j := left, right
	for i <= j {
		for j >= p && values[j] >= temp {
			j--
		}
		if j >= p {
			values[p] = values[j]
			p = j
		}
		if values[i] <= temp && i <= p {
			i++
		}
		if i <= p {
			values[p] = values[i]
			p = i
		}
	}
	values[p] = temp
	if p-left > 1 {
		quickSort(values, left, p-1)
	}
	if right-p > 1 {
		quickSort(values, p+1, right)
	}
}
func QuickSort(values []int) {
	quickSort(values, 0, len(values)-1)
}

func main(){
	fmt.Println("a real main in qsort")
}
```
编译结果如下:

![](/assets/golang/go-lang-package-without-main_func-.png)


- ####main包下有多个main函数

继续测试，现在 sorter.go 和 qsort.go 都被打包到 main，且有 main 函数。我们现在编译整个工程，结果如下：

![](/assets/golang/go-build-multiple-main.png)

编译的结果显示，sorter.go 中引用 qsort 失败，因为它不是一个可引用的包，而是一个程序！这是因为 qsort 是 main 包且有 main 函数，所以它不能被引入，导致编译出错。

- ####目录名和包名不一致
我们观察 bubblesort.go 这个文件，包名是 edgarlli, 但是它的目录名是 bubblesort,  虽然不一样但是编译成功了，且在 sorter.go 中成功地引用了这个模块：edgarlli.bubblesort，注意在 sorter.go 中 import 的是：

```import "algorithm/bubblesort"``` 

所以结论是，import 时是路径，而代码中使用的是包名.函数名/类 的方式。
