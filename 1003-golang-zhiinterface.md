# golang 之 interface

## 什么是 interface

interface 本质上是行为的集体: 实现出一个 interface 所有行为的具体类型即是 implement 了这个 interface.

## interface 的作用

代码中所有接纳一个 interface 的地方可以换成任意的实现了它行为集体的具体类型, 这称之为动态绑定, 也即多态, 赋予了程序强大的灵活性.  
举例:

```go
type Animal interface{
    Say();
}

type Cat struct{
    name string
    id int
    weigth double
}

func (c Cat) Say(){
    fmt.println("i am a cat")
}

func (c Cat) Eat(){
    fmt.println("i eat fish")
}
```

此时 Cat 就是一个具体类型, 它实现了 Animal 的所有行为, 所以它实现了 Animal 的接口.

```go
c := Cat{name:"fatcat", id:3, weigth:10}
var a Animal = c
a.Say()    //调用 Cat.Say(c)
```

## inside the interface

### 把戏

多态在底层均是函数指针替换的把戏.  
C++/Java 的多态实现机制在于, 于编译期安插代码, 在构造函数中设置好函数表, 析构函数中删除函数表.  
smalltalk, python, javascript 的多态实现机制在于, 运行时动态地去查找相关函数\(带缓存的\).  
golang 的原理介于这两者之间。  
interface 包括两个隐藏的字段, 长度均是 uintptr, 即指针大小:

* receiver: 变量的 **值/或地址**
* itablePtr: itable 地址\(接口表, 内含具体类型的类型信息和实现出的方法的地址\).

这两个字段在运行时会被设置, 见注释:

```go
c := Cat{}
var a Animal = c     //此时 a 中的 receiver 被设置为 &c, itablePtr 被设置 Cat 的方法表的地址
a.Say()             //此时调用 a.receiver.itablePtr.Say()
```

### 谁的接口表?

itablePtr 所指向的 itable 是对应于 interface 的, 而非具体类型的, itable 中只含 interface 中声明的方法. 因为一个 interface 只有在被  
具体类型赋值时才会有多态行为, 所以编译器理应只为该场景下的 interface 去关联具体类型生成 itable. 故, itable 从属主上来说, 应该是  
\(interface, 具体类型\). 上例中 itable 是 \(Animal, Cat\).

### 如何生成 itable

若程序中出现多次, 每一次 `var a Animal = c` 被执行时, 都会去生成 \(Animal, Cat\) 的 itable 吗? 从存储及运行效率来讲, 显然不会. 直觉上我们  
认为应该只会有一份 itable. 同时直觉上我们可能会认为, 如果该代码在高并发情况下运行, 是否可能为 \(Animal, Cat\) 生成多个 itable. 打开  iface.go 源码, 我们发现几个事实:

* 1.itable 是全局的, 并且是以 \(interface+具体类型\) 计算的 hash 值去关联的.
* 2.itabsinit 和 getitab 中均加锁了, 所以不会有并发问题.

在文章的最后给出验证代码, 在并发环境下执行 `var a Animal = c` 确实只有一个 itable 被产生.

### inside the itable

itable 包含以下成员:

* 接口信息 InterfaceInfo \(即字符串化的方法签名数组, 用于运行时反射得到接口信息\)  
* 类型信息 TypeInfo \(字符串化的方法签名与方法地址对应关系之数组, 用于运行时反射得到类型信息\)  
* 方法地址列表\(用于直接在 itablePtr 上调用方法\)  

下图是一张经典的 itable 图示, 环境是 32 位机器字长:

![](/assets/golang/go-interface-itab.png)

```go
type Stringer interface {
    String() string
}
type Binary uint64

func (i Binary) String() string {
    return strconv.Uitob64(i.Get(), 2)
}

b := Binary(200)
s := Stringer(b)
s.String()
```

如前, Stringer 这个 interface 有两个字段, tab 指向 itable\(Stringer, Binary\), data 指向 Binary 类型 变量.  
编译器会将 `s.String()` 替换为 `s.tab.func[0](s.data)`. 至于为什么是 func\[0\] 这一点就类型于 C++ 的虚函数了, 下标均是在编译期就可以确定的, 此处 itable 只有一项所以是 func\[0\]。  
有一点特别重要, 上图中的 itable func\[0\] 的值是 `(*Binary).String`, 表示定义于Binary指针上的方法String\(\)。而显示定义的 String 方法则是定义于 Binary 上的。

```go
//itable 中的方法
func (i *Binary) String() string{
    return strconv.Uitob64(i.Get(), 2)
}

//实际定义的方法
func (i Binary) String() string {
    return strconv.Uitob64(i.Get(), 2)
}
```

这是为何呢？  
先给出以下两个结论:

* 1.当定义 `func (i *Binary) String()` 或 `func(i Binary) String()` 一个时便会暗地里定义另外一个
* 2.`func(i *Binary) String()` 会被编译为 `func String(i *Binary)`
  以下的代码可验证上面的两点:

```go
type Fruit struct {
    name string
}

func (this  Fruit) Eat() {
    fmt.Println("in struct eat a fruit: ", this.name)
}


func wrapCall(f func (this Fruit), data Fruit)  {
    f(data)
}

func wrapCallPtr(f func (this *Fruit), data *Fruit)  {
    f(data)
}

f := Fruit{name:"apple"}
wrapCall((Fruit).Eat, f)    //ok, "in struct eat a fruit:  apple"
wrapCallPtr((*Fruit).Eat, &f)    //ok, "in struct eat a fruit:  apple"
```

再回到问题: 为什么 itable 中的 String 方法是基于 \*Binary 的而不是 Binary 呢？原因是对 interface 调用 String\(\) 方法时, 不知道具体类型 Binary 的大小也不需要知道: 直接使用指针这个定长之物即可: 毕竟任何变量的地址的长度均是一致的. 而再结合上面提到的两点, 即使具体没有显式地定义基于指针的方法, 但暗地里会有一个这样的方法被定义出来，所以可以这样用。

### itable 值语义

有一点可能没有提及: itable 中的 data 成员, 大多数情况它存储的是一个地址. 当有 `var a Animal = c` 这句代码时, data 所指的变量是否是 c 呢? 我们注意到 golang 是纯粹的值语义: 结构体赋值, 数组赋值, map 赋值\(map 内含的hashtable 指针, 也是值语义\), 切片\(内含数组指针, 长度, 容量, 也是值语义\)等，为了保持值语义，将具体类型变量赋值给 interface 时，不违反值语义的直觉，所以要切断具体类型变量与 interface 变量的联系--采用了拷贝具体类型变量并让 data 指向它。

### 例外

当具体类型占空间少于等于指针大小时, data 字段直接存储即可，不需要另外分配空间，再让 data 指向它。对应的，itable 中的方法不再是基于指针的，而是直接基于具体类型的。  
例如当定义 `type Binary32 uint32` 时对应的 itable 如下：

![](/assets/golang/go-interface-itab2.png)

另外一个例外是当接口不含任何方法时，可以不再保留 itable

![](/assets/golang/go-interface-itab3.png)

## golang interface 优势

golang interface 的优势在于, 它是一个松耦合且灵活的规范:

* 1.golang 实现类不需要引入接口所在的包. 更深层次来讲, 实现和接口解耦: 相同的行为可以来自于接口 A, 也可以是 B: 实现类可以认为实现了 A, 也可以认为实现了 B, 还可以认为实现了未来定义相同方法集合的接口 C, 在 java/C++ 中这是不具备的. 从另一个角度讲, 实现类添加行为的实现而不需要像 java/C++ 那样前置式声明要实现某接口, 这极具灵活性, 同时还避免了接口具体类实现的接口爆炸.
* 2.golang 实现类只有当它真正使用多态时, 才会与接口产生关联且产生性能的代价, 否则完全不会有副作用
* 3.golang 一个具体类可以实现多了个 interface 的行为, 具体类在运行时灵活选择具现出某一个 interface 的行为, 可以屏蔽其它 interface 的影响.
  反观 java, 具体类必须在一开始就要声明它实现哪些接口, 在多处分别使用时可能体现的是不同接口的行为, 但代价已经存在于所有的这些地方.    

## 并发环境下的 itable

验证并发环境下始终人有一个 itable 被产生.

```go
type CanEat interface {
    Eat()
}

type Fruit struct {
    name string
}

func (this  Fruit) Eat() {
    fmt.Println("in struct eat a fruit: ", this.name)
}

var flags chan bool

func testItable(threadId int) []CanEat {

    cases := 10
    ret := make([]CanEat, cases)
    for i:=0;i < cases;i ++{

        var caneat CanEat = Fruit{name:"apple"}
        ret[i] = caneat
        //fmt.Println("sizeof caneat: ", unsafe.Sizeof(caneat))    //mine pc prints 16

        var itableAddrAddr *int = (*int)(unsafe.Pointer(&caneat))
        fmt.Println("thread-", threadId,"itable address: ",*itableAddrAddr)

        var dataAddrAddr *int = (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&caneat)) + uintptr(8)))
        fmt.Println("thread-",threadId,"data address: ", *dataAddrAddr)
    }
    flags <- true

    return ret
}

func main()  {
    casees := 10
    flags = make(chan bool, casees)
    for i:=0;i<casees;i ++{
        go testItable(i)
    }

    for i:=0;i < casees;i ++{
        <- flags
    }
    fmt.Println("test finished ~")
}
```

以上使用 10 个协程, 各跑 10 次，输出如下：  

thread- 4 itable address:  5432128  

thread- 4 data address:  825741222048  

thread- 4 itable address:  5432128  

thread- 4 data address:  825741222064  

thread- 4 itable address:  5432128  

thread- 4 data address:  825741222080  

thread- 4 itable address:  5432128  

thread- 4 data address:  825741222096  

thread- 2 itable address:  5432128  

thread- 2 data address:  825741221952  

thread- 2 itable address:  5432128  

thread- 2 data address:  825741279696  

thread- 2 itable address:  5432128  

thread- 2 data address:  825741279712  

thread- 2 itable address:  5432128  

thread- 2 data address:  825741279728  

  
.....  
限于篇幅, 不全部列出。如上很清晰可以看到 itable address 都是一致的，这说明只有一个 itable 产生。而很明显与之对应的是 data 所指变量每次都不一样，这是在预料中的。  
可能有人觉得奇怪，为什么 testItable 方法要返回无用的 CanEat 切片呢，这是为了验证每个 data 均不一样，若不返回此对象， golang 编译器的逃逸分析可能会导致各个线程产生的 Fruit 对象的地址总是一样的。

