# Golang 之 interface

## 概述

本文由浅入深, 讲述 golang 的 interface 的实现原理

## interface 为何物

interface 本质上是行为的集体: 实现出一个 interface 所有行为的具体类型即是 implement 了这个 interface.

## interface 的作用

代码中所有接纳一个 interface 的地方可以换成任意的实现了它行为集体的具体类型, 这称之为动态绑定, 也即多态, 赋予了程序强大的灵活性.举例:

```
type Animal interface{


Say();


}


​


type Cat struct{


name string


id int


weigth double


}


​


func (c Cat) Say(){


fmt.println("i am a cat")


}


​


func (c Cat) Eat(){


fmt.println("i eat fish")


}
```

此时 Cat 就是一个具体类型, 它实现了 Animal 的所有行为, 所以它实现了 Animal 的接口.

```


c := Cat{name:"fatcat", id:3, weigth:10}


var a Animal = c


a.Say()
//调用 Cat.Say(c)
```

## inside the interface

### 把戏

多态在底层均是函数指针替换的把戏.C++/Java 的多态实现机制在于, 于编译期安插代码, 在构造函数中设置好函数表, 析构函数中删除函数表.smalltalk, python, javascript 的多态实现机制在于, 运行时动态地去查找相关函数\(带缓存的\).golang 的原理介于这两者之间。interface 包括两个隐藏的字段, 长度均是 uintptr, 即指针大小:receiver: 简单类型变量的值/或变量的地址itablePtr: itable 地址\(接口表, 内含具体类型的类型信息和实现出的方法的地址\).

这两个字段在运行时会被设置, 见注释:c := Cat{}var a Animal = c //此时 a 中的 receiver 被设置为 &c, itablePtr 被设置 Cat 的方法表的地址a.Say\(\) //此时调用 a.receiver.itablePtr.Say\(\)

### 谁的接口表

itablePtr 所指向的 itable 是对应于 interface 的, 而非具体类型的, itable 中只含 interface 中声明的方法. 因为一个 interface 只有在被具体类型赋值时才会有多态行为, 所以编译器理应只为该场景下的 interface 去关联具体类型生成 itable. 故, itable 从属主上来说, 应该是\(interface, 具体类型\). 上例中 itable 是 \(Animal, Cat\).

### 如何生成 itable

若程序中出现多次, 每一次 var a Animal = c 被执行时, 都会去生成 \(Animal, Cat\) 的 itable 吗? 从存储及运行效率来讲, 显然不会. 直觉上我们认为应该只会有一份 itable. 同时直觉上我们可能会认为, 如果该代码在高并发情况下运行, 是否可能为 \(Animal, Cat\) 生成两个 itable. 打开iface.go 源码, 我们发现几个事实:1.itable 是全局的, 并且是以 \(interface+具体类型\) 计算的 hash 值去关联的.2.itabsinit 和 getitab 中均加锁了, 所以不会有并发问题.

### inside the itable

itable 包含以下成员:接口信息 InterfaceInfo \(即字符串化的方法签名数组, 用于运行时反射得到接口信息\)类型信息 TypeInfo \(字符串化的方法签名与方法地址对应关系之数组, 用于运行时反射得到类型信息\)方法地址列表\(用于直接在 itablePtr 上调用方法\)如下图:



golang interface 的优势在于, 它是一个松耦合且灵活的规范:1.golang 实现类不需要引入接口所在的包. 更深层次来讲, 实现和接口解耦: 相同的行为可以来自于接口 A, 也可以是 B: 实现类可以认为实现了 A, 也

```


可以认为实现了 B, 还可以认为实现了未来定义相同方法集合的接口 C, 在 java/C++ 中这是不具备的. 从另一个角度讲, 实现类添加行为的实现而不


需要像 java/C++ 那样前置式声明要实现某接口, 这极具灵活性, 同时还避免了接口具体类实现的接口爆炸.
```

2.golang 实现类只有当它真正使用多态时, 才会与接口产生关联且产生性能的代价, 否则完全不会有副作用3.golang 一个具体类可以实现多了个 interface 的行为, 具体类在运行时灵活选择具现出某一个 interface 的行为, 可以屏蔽其它 interface 的影响.反观 java, 具体类必须在一开始就要声明它实现哪些接口, 在多处分别使用时可能体现的是不同接口的行为, 但代价已经存在于所有的这些地方.

补充：a\).将变量赋值给 interface 时, 若变量的直可直接存储在 receiver 中则直接赋值, 否则生成一个变量的拷贝, 以其地址设置为 receiver.b\).itablePtr 所指内容包括具体类型的如下信息:

