# Chan

# 1. 前言

channel是Golang语言提供 goroutine通信方式

# 2.chan数据结构

~~~go
type hchan struct{
    qcount uint 		//当前队列剩余元素个数
    datasqiz uint 		//环形队列长度,即可以存放元素个数
    buf  unsafe.Pointer //环形队列指针
    elemsize uint16 	//每个元素的大小
    closed uint32 		// 标志关闭状态
    elemtype *_type 	// 元素类型
    sendx uint 			// 队列下标，指示元素写入存到队列的位置 
    recvx uint 			//队列下标，指示元素从队列该位置读出
    recvq waitq 		//等待读goroutine队列
    sendq waitq 		//等待写goroutine队列
    lock mutex 			//互斥锁
}
~~~

## 2.1 环形队列

chan内部实现环形队列做缓冲区，队列的长度是创建chan指定的

  ![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_f1b42d200c5d94d02eeacef7c99aa81b_r.png)

- dataqsiz指示了队列长度为6，即可缓存6个元素；
- buf指向队列的内存，队列中还剩余两个元素；
- qcount表示队列中还有两个元素；
- sendx指示后续写入的数据存储的位置，取值[0, 6)；
- recvx指示从该位置读取数据, 取值[0, 6)；

## 2.2 等待队列

从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞。
向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_f48c37e012c38de53aeb532c993b6d2d_r.png)

一般情况下recvq和sendq 至少有一个为空，那就是同一个goroutine使用select语句

## 2.3 类型信息

一个channel只能传递一个值，类型信息存储在hchan数据结构

- elemtype 代表类型，用于数据传递过程中赋值
- elemsize代表大小，用于进行元素定位

## 2.4 锁

一个`channel`仅允许一个`goroutine`读写

# 3. channel读写 （复习没够好）

## 3.1 创建channel

```go
func makechan(t *chantype, size int) *hchan {
    var c *chan
    c = new(hchan)
    c.buf = malloc(元素类型大小*size)
	c.elemsize = 元素类型大小
    c.elemtype = 元素类型
    c.dataqsiz = size
    return c
}
```

## 3.2 向channel写数据

向channel写数据简单过程如下：

- 1.**等待接收队列recvq不为空**，说明没有数据或者没有缓冲区，(没有需要接受的数据,有需要写入的数据)此时从recvq取出G并把数据写出，最后把数据唤醒
- 2.**数据有空余位置，将数据写入缓冲区**,结束发送过程
- 3.如果缓冲区没有空闲位置，将发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_b235ef1f2c6ac1b5d63ec5660da97bd2_r.png)

## 3.3 从channel读数据

从channel读数据的过程如下：

1.如果等**待发送队列sendq不为空**，而且没有缓冲区，直接把sendq取出G，把G中数据读出

2.**如果等待发送队列sendq不为空，说明缓冲区满**，在缓冲区首部读出数据，把G中数据写入缓冲区尾部，把G环形

3.如果缓冲区中有数据，从缓冲区中取数据

4.将goroutine加入recq 进入睡眠，等待被写goroutine唤醒

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_933ca9af4c3ec1db0b94b8b4ec208d4b_r.png)

## 3.4 关闭channel

关闭channel会把recvq的G唤醒，本该**写入G的数据位置为nil.** 

把sendq的G全部唤醒，但这些G会panic

panic出现场景：

1.关闭值为nil的 channel

2.关闭被关闭的通道

3.向关闭的channel写数据

# 4. 常见用法

## 4.1 单向channel

单向channel只能用于发送和接收数据

- func readChan(chanName <-chan int)： 通过形参限定函数内部只能从channel中读取数据
- func writeChan(chanName chan<- int)： 通过形参限定函数内部只能向channel中写入数据

```go
func readChan(chanName <-chan int) {
    <- chanName
}

func writeChan(chanName chan<- int) {
    chanName <- 1
}

func main() {
    var mychan = make(chan int, 10)

    writeChan(mychan)
    readChan(mychan)
}

//mychan是个正常的channel，而readChan()参数限制了传入的channel只能用来读，writeChan()参数限制了传入的channel只能用来写
```

## 4.2 select

使用select可以监视多个channel，当有一个channel有数据，就读出数据

```go
package main

import (
    "fmt"
    "time"
)

func addNumberToChan(chanName chan int) {
    for {
        chanName <- 1
        time.Sleep(1 * time.Second)
    }
}

func main() {
    var chan1 = make(chan int, 10)
    var chan2 = make(chan int, 10)

    go addNumberToChan(chan1)
    go addNumberToChan(chan2)

    for {
        select {
        case e := <- chan1 :
            fmt.Printf("Get element from chan1: %d\n", e)
        case e := <- chan2 :
            fmt.Printf("Get element from chan2: %d\n", e)
        default:
            fmt.Printf("No element in chan1 and chan2.\n")
            time.Sleep(1 * time.Second)
        }
    }
}
```

程序创建两个channel,函数addNumberToChan函数会给两个channel周期性写入数据，channel读入数据是随机的

## 4.3 range

通过range持续从channel中读出数据，像遍历数组 一样,当channel没有数据时会阻塞当前goroutine

~~~go
func chanRange(chanName chan int) {
    for e := range chanName {
        fmt.Printf("Get element from chan: %d\n", e)
    }
}
~~~



# Slice

# 1. 前言

Slice是动态数组，依托数组实现，可以方便**扩容，传递等等**，实际用起来比数组更灵活

# 2. 热身环节

## 2.1 题目一

~~~go
package main

import (
    "fmt"
)

func main() {
    var array [10]int

    var slice = array[5:6]

    fmt.Println("lenth of slice: ", len(slice))
    fmt.Println("capacity of slice: ", cap(slice))
    fmt.Println(&slice[0] == &array[5])
}
~~~

题目：slice根据数组array创建，和数组共享存储空间，slice的起始位置是array[5],长度为1，容量为5,slice[0]和array[5]地址相同

## 2.2 题目二

```go
package main

import (
    "fmt"
)

func AddElement(slice []int, e int) []int {
    return append(slice, e)
}

func main() {
    var slice []int
    slice = append(slice, 1, 2, 3)

    newSlice := AddElement(slice, 4)
    fmt.Println(&slice[0] == &newSlice[0])
}

//网页的答案 2 4
//我的答案 0 3 6
```

![image-20220312090441461](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220312090441461.png)

append函数的执行会判断切片能否新增，刚开始slice容量为空，所以根据`oldCap * 2< cap`，所以会扩容到3，然后根据翻倍扩容到6

## 2.3 题目三

```go
package main

import (
    "fmt"
)

func main() {
    orderLen := 5
    order := make([]uint16, 2 * orderLen)

    pollorder := order[:orderLen:orderLen]
    lockorder := order[orderLen:][:orderLen:orderLen]

    fmt.Println("len(pollorder) = ", len(pollorder))
    fmt.Println("cap(pollorder) = ", cap(pollorder))
    fmt.Println("len(lockorder) = ", len(lockorder))
    fmt.Println("cap(lockorder) = ", cap(lockorder))
}

//order[low:high:max] 对order进行切片操作, 新切片范围是[low,high),  order长度为2倍len， 
pollorder是前半部分
lockorder是后半部分 [orderLen:]是5开始 [:orderLen:orderLen] 是找五个最大的是五个
```

# 3. Slice实现原理

Slice依托底层数组，底层数组容量不足可以实现自动分配新的Slice

## 3.1 Slice数据结构

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
//array指向底层数组
//len表示切片商都
//cap表示底层数组容量
```

## 3.2 使用make创建Slice

使用make来创建Slice时，可以同时**指定长度和容量，创建时底层会分配一个数组，数组的长度即容量**

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_332a02ff2dc338bb2cce150a23d37b1c_r.png)

## 3.3 使用数组创建Slice

语句`slice := array[5:7]`所创建的Slice，结构如下图所示

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_c6aff21b79ce0b735065a702cb84c684_r.png)

## 3.4 Slice 扩容

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_a87b8e2fb06bff1ea78f6096b7e81325_r.png)

扩容操作只关心容量，会把**原Slice数据拷贝到新Slice**，追加数据由append在扩容结束后完成

扩容遵循如下规则：

- 如果满足oldCap*2 < cap, 那么newCap = cap

- 不满足上述情况，如果oldCap <1024字节，newCap = oldCap * 2

- 如果大于1024字节,newCap = oldCap * 1.25

append()向Slice实现元素步骤如下：

- Slice够用，新元素追加，Slice.len++,返回原slcie
- Slice不够，Slice扩容，扩容得到新slice
- 新元素追加到新Slice,Slice.len++,返回新Slice

## 3.5 Slice Copy

使用copy()内置函数拷贝切片，会将源切片拷贝到目的切片指向的数组，拷贝数量取两个切片长度最小值

例如长度为10的切片，拷贝长度为5的切片，会拷贝5哥元素，copy过程不会发生扩容

## 3.6 特殊切片

~~~go
sliceA := make([]int, 5, 10)
sliceB := sliceA[0:5]

//切片，长度，容量一致
~~~

数组，切片，生成切片是

~~~go
//slice[start:end:cap], 其中cap即为新切片的容量
sliceA := make([]int, 5, 10)  //length = 5; capacity = 10
sliceB := sliceA[0:5]         //length = 5; capacity = 10
sliceC := sliceA[0:5:5]       //length = 5; capacity = 5

//不太常见，在Golang源码可以见到，利于切片理解
~~~

# 4. 编程Tips

- 创建切片需要根据实际预分配容量，尽量避免追加过程扩容操作，有利于性能
- 切片拷贝需要判断实际拷贝的元素个数
- 谨慎用多个切片操作同一个数组，防止读写冲突

# 5. Slice总结

- 每个切片都指向一个底层数组
- 每个切片都保存了当前切片，底层数组的容量
- len()计算切片长度时间复杂度为O(1),不需要遍历切片
- cap()计算切片长度时间按复杂度为O(1)，不需要遍历切片
- 通过函数传递切片，不会拷贝整个切片，因为切片本身是结构体
- 使用append()向切片追加元素时有可能触发扩容，扩容后将会生成新的切片



# Map

# 1. map数据结构

Golang的map利用**哈希表底层实现**，哈希表可以有多个哈希表及诶单，即Bucket

map数据结构由`runtime/map.go:hmap`定义:

```go
type hmap struct {
    count     int //当前保存元素的个数
    ...
    B         uint8
    ...
    buckets    unsafe.Pointer //bucket数组指针，数组大小为2^B
    ...
}
```

~~~go
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
~~~

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_897a05f6373f7f966d00d1bfea6274d2_r.png)

在该例子中，**hmap.B=2 **,而hmap.buckets长度为2^B为4，元素经过哈希运算后落到某个bucket存储，`bucket`很多时候被翻译为桶，所谓的`哈希桶`实际上就是bucket

# 2. bucket数据结构

bucket数据结构由`runtime/map.go:bmap`定义

```go
type bmap struct {
    tophash [8]uint8 //存储哈希值的高8位
    data    byte[1]  //key value数据:key/key/key/.../value/value/value...
    overflow *bmap   //溢出bucket的地址
}
```

- tophash是长度为8的数组，哈希键相同的键（准确的说是哈希值低位相同的键），存入当前bucket会将哈希值高位存储在数组
- data区存放key-value,存放顺序是key/key/key/…value/value/value，如此存放是为了**节省字节对齐带来的空间浪费**。
- overflow指针指向的是下一个bucket,将所有冲突的键连在一起

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_7f0ba5a124641b1413279892581513c4_r.png)

# 3. 哈希冲突

当两个或两个以上的键被哈希存到了一个bucket中，我们称这些键发生冲突，Go中用**链地址法**解决冲突

产生冲突后的map：

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_a8b9e5919d9951a71c1c36445dd68521_r.png)

bucket数据结构指示下一个bucket称为overflow bucket,以为bucket盛不下溢出的部分，hash冲突不是好事，好的hash算法可以保证hash值随机性

# 4. 负载因子

负载因子用于衡量哈希表情况

~~~go
负载因子 = 键数量/bucket数量
~~~

哈希表需要将负载因子控制到合适的大小，超过阈值需要rehash,即为对键值重新组织

- 哈希因子过小，说明空间利用率低
- 哈希因子过大，说明冲突，存取效率低

哈希表实现对复杂因子容忍不同，Redis复杂因子>1触发rehash,Redis bucket只存在1个键值对，Go的bucket可以存8个键值对，所以Go可以容忍更高的负载因子

# 5. 渐进式扩容

## 5.1 扩容的前提条件

触发扩容条件有两个：

1.负载因子 >6.5,平均每个Bucket存储6.5

2.overflow的数量>2^15,overflow数量超过32768

## 5.2 增量扩容

当负载因子过大，就新建一个bucket,**新的bucket是原来的2倍，然后旧数据搬迁到新的bucket**,考虑到如果**map存储了数以亿计的key-value**，一次性搬迁将会造成比较大的**延时**，Go采用**逐步搬迁策略，即每次访问map时都会触发一次搬迁，每次搬迁2个键值对**

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_2d622a6bc19ca1b5bcb225f77869f9c2_r.png)

当前map存储了7个键值对，只有1个bucket。此地负载因子为7**。再次插入数据时将会触发扩容操作**，扩容之后再将新插入键写入新的bucket。

当第8个键值对插入时，将会触发扩容

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_2f0122f26e5d66ca91e6820ace6b379b_r.png)

hmap数据结构的oldbuckets成员指身原bucket.新的键值对被插入新的bucket中，

后续对map访问会迁移，将oldbuckets值逐步迁移过来，Oldbuckets迁移完毕，删除oldbuckets

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_b1178e0a3cea02c9386e5f5eaa6f99a6_r.png)

## 5.3 等量扩容

等量扩容，不是扩大容量，而是buckets数量不变，然后再做一遍搬迁动作，把**松散的键值重新排列一遍**，以使bucket的使用率更高，进而保证更快的存取。

在极端场景下，比如不断地增删，而键值对正好集中在一小部分的bucket，这样会造成overflow的bucket数量增多，但负载因子又不高，从而无法执行增量搬迁的情况，如下图所示

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_f3a5989c90204df9304d5ae246f3db72_r.png)

overflow的bucket中大部分是空的，访问效率会很差。**此时进行一次等量扩容，即buckets数量不变**，经过重新组织后overflow的bucket数量会减少，即节省了空间又会提高访问效率。

# 6. 查找过程

查找过程如下：

- 1.根据Key值算出hash值
- 2.取hash值地位和**hmap.B确定bucket位置**
- 3取哈希值高位在**tophash数组中查询**
- 4.如果**tophash[i]存储值和hash值相等**，则去找bucket的key进行比较
- 5.如果bucket没有找到，**需要找下个overflow的bucket查找**
- 6.当前处于搬迁过程，优先从oldbuckets查找

如果查询不到，不会返回控制，只是会返回相应的零值

# 7. 插入过程

新元素插入过程：

1.根据key求hash值

2.取哈希值低位和hmap.B确定bucket位置

3.查找该key是否存在，存在则更新值

4.如果不存在key，将key插入

# Struct

# 1.前言

Go的Struct附带了Tag的标记，该Tag不仅是字符串，主要用于发射，所以`Tag`遵循一定的规则

# 2. Tag的本质

## 2.1 Tag规则

`Tag`本身是字符串， 字符串是：`以空格分隔的key:value对`

`key`：非空字符号串， 不能包括**控制字符**，**引号**，**冒号**，**空格**

`value`: **双引号标记的字符串**

```go
type Server struct {
    ServerName string `key1: "value1" key11:"value11"`
    ServerIP   string `key2: "value2"`
}
```

## 2.2 Tag是Struct的一部分

`Tag`只有反射场景有用，反射包提供了`Tag`方法，我们需要看reflect包的声明

```go
// A StructField describes a single field in a struct.
type StructField struct {
    // Name is the field name.
    Name string
    ...
    Type      Type      // field type
    Tag       StructTag // field tag string
    ...
}

type StructTag string
```

一个结构体成员包含了 `StructTag`，本身是一个`Tag`

## 2.3 获取Tag

```go
import (
    "reflect"
    "fmt"
)

type Server struct {
    ServerName string `key1:"value1" key11:"value11"`
    ServerIP   string `key2:"value2"`
}

func main() {
    s := Server{}
    st := reflect.TypeOf(s)

    field1 := st.Field(0)
    fmt.Printf("key1:%v\n", field1.Tag.Get("key1"))
    fmt.Printf("key11:%v\n", field1.Tag.Get("key11"))

    filed2 := st.Field(1)
    fmt.Printf("key2:%v\n", filed2.Tag.Get("key2"))
}
```

- 1.创建Struct结构体
- 2.创建结构体实例
- 3.对于结构体实例进行反射，然后通过filed方法进一步获取
- 4.然后通过.Tag.Get方法获取

```go
key1:value1
key11:value11
key2:value2
```

# 3. Tag存在的意义



反射可以动态给结构体成员赋值， 因为有tag，所以在tag来决定赋值动作

# iota

# 1. 前言



**iota用于const表达式，值开始是零值，const声明块没增加一行iota自增1**

# 2. 热身

## 2.1 题目一

```go
type Priority int
const (
    LOG_EMERG Priority = iota
    LOG_ALERT
    LOG_CRIT
    LOG_ERR
    LOG_WARNING
    LOG_NOTICE
    LOG_INFO
    LOG_DEBUG
)
//1~7
```

## 2.2 题目二

```go
const (
   mutexLocked = 1 << iota // mutex is locked
   mutexWoken
   mutexStarving= iota
   mutexWaiterShift
   starvationThresholdNs = 1e6
)

//1左移iota位
//分别是 1 2 2 3 
```

## 2.3 题目三

```go
const (
    bit0, mask0 = 1 << iota, 1<<iota - 1
    bit1, mask1
    _, _
    bit3, mask3
)
// 1 0 2 1 8 7
```

# 3. 规则

1.iota在 const关键字出现重置为0

2.每新增一行iota值自增1

- iota代表了const声明块的行索引（下标从0开始）

```GO
const (
    bit0, mask0 = 1 << iota, 1<<iota - 1   //const声明第0行，即iota==0
    bit1, mask1                            //const声明第1行，即iota==1, 表达式继承上面的语句
    _, _                                   //const声明第2行，即iota==2
    bit3, mask3                            //const声明第3行，即iota==3
)
```

# 4. 编译原理

```GO
    ValueSpec struct {
        Doc     *CommentGroup // associated documentation; or nil
        Names   []*Ident      // value names (len(Names) > 0)
        Type    Expr          // value type; or nil
        Values  []Expr        // initial values; or nil
        Comment *CommentGroup // line comments; or nil
    }
```

我们关注ValueSpec.Names   这个切片保存一行定义的常量，如果一行定义N个常量，那么ValueSpec.Names切片长度即为N

```GO
    for iota, spec := range ValueSpecs {
        for i, name := range spec.Names {
            obj := NewConst(name, iota...) //此处将iota传入，用于构造常量
            ...
        }
    }
```

# string

## string标准概念

Go标准库builtin给出了内置类型

源代码位于`src/builtin/builtin.go`，其中关于string的描述如下

```go
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```

string构成：

**8比特字节集合**，通常但不一定是UTF-8编码文本

- string可以为空(长度为0)，但是不会是nil
- string对象不能修改

## string 数据结构

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

- stringStruct.str：字符串的首地址；
- stringStruct.len：字符串的长度；

## string操作

### 声明

对string变量进行赋值：

字符串的构建过程实际是先字符串构建StringStruct，转成string类型

（string在runtime包中是stringstruct,对外展现是string）

```go
 var str string
    str = "Hello World"
```

```go
func gostringnocopy(str *byte) string { // 根据字符串地址构建string
    ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)} // 先构造stringStruct
    s := *(*string)(unsafe.Pointer(&ss))                             // 再将stringStruct转换成string
    return s
}
```

### []byte转string

[]byte可以转换string

~~~go
func GetStringBySlice([]byte s) string{
    return string(s)
}
~~~

但是需要经历一次内存拷贝：

1.根据切片容量申请空间，内存地址为p，申请的长度为len

2.对结构体的str和len进行赋值(构建String)，string.str = p , string.len = len

3.拷贝数据（拷贝切片的数据到新申请的地址空间）

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_0317d71784cf0c9b1a00cee014429c40_r.png)

### string转为[]byte

```go
func GetSliceByString(str string) []byte {
    return []byte(str)
}


```

内存拷贝：

- 申请切片内存空间
- 将string拷贝到切片

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_5f500d01a01d45f69ea523a1789f9748_r.png)

```go
func concatstrings(a []string) string { // 字符串拼接
   length := 0        // 拼接后总的字符串长度

   for _, str := range a {
      length += len(str)
   }

   s, b := rawstring(length) // 生成指定大小的字符串，返回一个string和切片，二者共享内存空间

   for _, str := range a {
      copy(b, str)    // string无法修改，只能通过切片修改
      b = b[len(str):]
   }

   return s
}
```

string无法直接修改，使用rawstring()方法指定大小的string,同时返回一个切片，二者共享内存，后面向切片中拷贝数据，也就间接修改了string

```go
func rawstring(size int) (s string, b []byte) { // 生成一个新的string，返回的string和切片共享相同的空间
    p := mallocgc(uintptr(size), nil, false)

    stringStructOf(&s).str = p
    stringStructOf(&s).len = size

    *(*slice)(unsafe.Pointer(&b)) = slice{p, size, size}

    return
}
```

## 为什么字符串不允许修改？

在C++的string,其本身的内存空间，对于修改string是支持的，但go中实现，**string没有内存空间，只有一个内存指针**，方便进行传递而不用担心拷贝

string指向字符串字面量，字符串字面量存储是已读段，所以string不可修改

## []byte转换成string一定会拷贝内存吗？

byte切片转换为string场景有很多，有的只是临时需要字符串，这时候byte转换为string不会拷贝内存，只是返回一个string

- 使用m[string(b)]来查找map（map是string为key，临时把切片b转成string）；
- 字符串拼接，如”<” + “string(b)” + “>”；
- 字符串比较：string(b) == “foo”

临时把byte切片转换成string，也就避免了因byte切片同容改成而导致string引用失败的情况，所以此时可以不必拷贝内存新建一个string

## string和[]byte如何取舍

string和[]byte都可以表示字符串，但因数据结构不同，其衍生出来的方法也不同

string 擅长的场景

- 需要字符串比较的场景
- 不需要nil字符串的场景

[]byte擅长的场景:

- 修改字符串的场景，修改粒度为1字节
- 函数返回值，需要nil表示含义
- 需要切片操作

虽然看起来string没有[]byte多，但是因为**string直观，实际应用大量存在**，所以底层用[]byte更多

