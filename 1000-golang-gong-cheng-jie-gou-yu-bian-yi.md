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

. 折叠
真的吗

#试验准备
现在有一个 go 工程目录结构如下图所示

![](/assets/go-lang-dir.png)

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

![编译时如何找到单个文件](/assets/go-build-single-file.png)

需要注意还有两点：
编译成功后生成的文件会放置在当前位置。
若 go build 不指定参数则生成文件名为 main.exe

- ####编译不含main的文件
当我们要编译的 go 文件的包名不含 main 函数，会是怎样的呢？继续试验。qsort.go **没有 main 函数，编译成功后它没有在任何地方生成任何文件**。而 sorter.go 中有 main 函数，它在当前目录下生成了 exe 文件。


- ####main 包下没有main函数
如果一个 go 文件 package 的是 main 包，但没有 main 函数，会有怎样的编译结果？我们更改 qsort.go 文件，将包名改为 main 但不增加 main 函数，编译结果如下:



- ####两个或以上文件都有 main




