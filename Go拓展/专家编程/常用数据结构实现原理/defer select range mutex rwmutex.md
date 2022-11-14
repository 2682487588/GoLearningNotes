# defer

# 1. 前言

关于defer的调用，每个defer都会把一个函数压入栈中，然后在函数返回之前把延迟函数取出来

 延迟函数可能有输入参数，延迟函数也可能引用主函数用于返回的变量，也就是说延迟函数可能会影响主函数的一些行为，这些场景下，如果不了解defer的规则很容易出错。

# 2. 热身

## 2.1 题目一

```go
func deferFuncParameter() {
    var aInt = 1

    defer fmt.Println(aInt)

    aInt = 2
    return
}

//输出1
//延迟函数在defer语句的注册的时候已经将变量填入栈中，后面就无法被修改了
```

## 2.2 题目二

```go
package main

import "fmt"

func printArray(array *[3]int) {
   for i := range array {
      fmt.Println(array[i])
   }
}

func deferFuncParameter() {
   var aArray = [3]int{1, 2, 3}

   defer printArray(&aArray)

   aArray[0] = 10
   return
}

func main() {
   deferFuncParameter()
}
```

输出10,2,3 三个值 ， 延迟函数printArray在defer注册的时候确定了，即数组地址， 延迟函数执行时在return赋值之前

## 2.3 题目三

```go
func deferFuncReturn() (result int) {
    i := 1
	
    defer func() {
       result++
    }()

    return i
}

// 2
```

首先有一个局部变量赋值，然后返回局部变量值用result接收1，然后result++就为2

# 3. defer规则

## 3.1 规则一：延迟函数的参数在defer语句出现时就已经确定下来了

```go
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```

defer语句的fmp.Prinln()参数i在defer的时候已经确定，实际是拷贝了一份

注意：指针类型，规则仍然使用，只不过延迟函数的**参数是一个地址**，**defer后面的语句对变量修改可能会影响延迟函数**

## 3.2 规则二：延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

defer类似于入栈，执行defer类似于出栈

资源顺序，申请 A->B->C   释放 C->B->A

没申请一个用完需要释放的资源，立即定义一个defer释放资源

## 3.3 规则三：延迟函数可能操作主函数的具名返回值

### 3.3.1 函数返回过程

```go
func deferFuncReturn() (result int) {
    i := 1

    defer func() {
       result++
    }()

    return i
}

//1.i值存入栈中作为返回值
//2.defer的执行时机正是跳转前,所以说defer执行时还是有机会操作返回值的
//3.执行跳转
```

### 3.3.2 主函数拥有匿名返回值，返回字面值

```go
func foo() int {
    var i int

    defer func() {
        i++
    }()

    return 1
}
```

return语句，直**接把1写入栈中作为返回值**，延迟函数无法操作该返回值，所以就无法影响返回值

### 3.3.3 主函数拥有匿名返回值，返回变量

```go
func foo() int {
    var i int

    defer func() {
        i++
    }()

    return i
}
```

上面的函数，返回一个局部变量，同时defer函数也会操作这个局部变量。对于匿名返回值来说，可以假定仍然有一个变量存储返回值，假定返回值变量为”anony”，上面的返回语句可以拆分成以下过程：

```go
anony = i
i++
return
```

### 3.3.4 主函数拥有具名返回值

一个影响函返回值的例子：

```go
func foo() (ret int) {
    defer func() {
        ret++
    }()

    return 0
}
```

上面的函数拆解出来，如下所示：

```go
ret = 0
ret++
return
```

# 4. defer实现原理

## 4.1 defer数据结构

源码包`src/src/runtime/runtime2.go:_defer`定义了defer的数据结构：

```go
type _defer struct {
    sp      uintptr   //函数栈指针
    pc      uintptr   //程序计数器
    fn      *funcval  //函数地址
    link    *_defer   //指向自身结构的指针，用于链接多个defer
}
```

defer后面跟一个函数，所以defer的数据结构和一般函数类似

和函数不同的他有一个指针，指向另外一个defer,goroutine的数据结构也有一个defer指针

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_226214a05ea08033680d03d624a60de3_r.png)

新声明的defer总是添加到链表头部，一个goroutine可以调用多个函数，进入函数时添加defer，离开函数时取出defer，所以即便调用多个函数，也总是能保证defer是按LIFO方式执行的。

## 4.2 defer的创建和执行

- deferproc()： 在声明defer处调用，其将defer函数存入goroutine的链表中；
- deferreturn()：在return指令，准确的讲是在ret指令前调用，其将defer从goroutine链表中取出并执行。

可以简单这么理解，在编译阶段，声明defer处插入了函数deferproc()，在函数**return前插入了函数deferreturn()**

# 5. 总结

- defer延迟函数在defer时候确定
- defer**定义和实际执行顺序相反**
- return不是原子操作，执行过程是: 保存返回值(若有)–>执行defer（若有）–>执行ret跳转
- 申请资源后立即使用**defer关闭资源是好习惯**



## recover 失效的条件

上面代码`IsPanic()`失效了，其原因是违反了recover的一个限制，导致recover()失效（永远返回`nil`）。

以下三个条件会让recover()返回`nil`:

1. panic时指定的参数为`nil`；（一般panic语句如`panic("xxx failed...")`）
2. 当前协程没有发生panic；
3. recover没有被defer方法直接调用；

# select

# 1. 前言

select是Golang在语言层面提供的多路IO复用的机制，可以检测多个channel是否ready，使用起来方便

# 2. 热身环节

## 2.1 题目1

select中各个case是随机的，如果某个case的channel已经ready，所有case未ready,执行default语句退出select流程

可能的输出一：

```go
chan1 ready.
main exit.
```

可能的输出二：

```go
chan2 ready.
main exit.
```

可能的输出三：

```go
default
main exit.
```

## 2.2 题目2

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    chan1 := make(chan int)
    chan2 := make(chan int)

    writeFlag := false
    go func() {
        for {
            if writeFlag {
                chan1 <- 1
            }
            time.Sleep(time.Second)
        }
    }()

    go func() {
        for {
            if writeFlag {
                chan2 <- 1
            }
            time.Sleep(time.Second)
        }
    }()

    select {
    case <-chan1:
        fmt.Println("chan1 ready.")
    case <-chan2:
        fmt.Println("chan2 ready.")
    }

    fmt.Println("main exit.")
}
```

答案：select会随机的顺序检测各case语句中channel是否ready，如果所有的channel都没有ready就会阻塞

## 2.3 题目3

```go
func main() {
    chan1 := make(chan int)
    chan2 := make(chan int)

    go func() {
        close(chan1)
    }()

    go func() {
        close(chan2)
    }()

    select {
    case <-chan1:
        fmt.Println("chan1 ready.")
    case <-chan2:
        fmt.Println("chan2 ready.")
    }

    fmt.Println("main exit.")
}
```

select随机熟悉怒检测channel是否ready，考虑到已经关闭的channel也是可读的，所以上述程序select不会阻塞

## 2.4 题目4

```go
package main

func main() {
    select {
    }
}
```

当前协程会阻塞，同时Golang自带死锁机制，发现协程阻塞的时候会报panic

# 3. 实现原理

## 3.1 case数据结构

~~~go
type scase struct{
    c *hchan 				//chan
    kind uint16	
    elem unsafe.Pointer		//data element
}
~~~

scase.c 为当前case语句操作的channel指针 , 一个语句只能操作一个channel

scase.kind 表示 case的类型分别为 读channel 写channel 和default

- caseRecv: case语句从scase.c 获取数据
- caseSend: case语句从scase.c 发送数据
- caseDefault: default语句

scase.elem表示缓冲区地址

- scase.kind == caseRecv   scase.elem 表示读出channel的存放位置
- scase.kind == caseSend  scase.elem 表示写入channel的存放位置

## 3.2 select实现逻辑

定义select选择case函数：  func selectgo(cas0  *scase, order0 * uint16, ncases int) (int, bool)

函数参数：

- cas0 为 scase数组的首地址，selectgo()从这些scase找一个返回
- order0为二倍cas0数组长度的buffer,保证scase乱序的pollorder和scase的channel地址序列lockorder
- 1. pollorder:每次selectgo把scase打乱，达到随机检测case目的
  2. lockorder ：所有case中channel序列，防止给channel加锁重复加锁

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    //1. 锁定scase语句中所有的channel
    //2. 按照随机顺序检测scase中的channel是否ready
    //   2.1 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index, true)
    //   2.2 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index, false)
    //   2.3 所有case都未ready，则解锁所有的channel，然后返回（default index, false）
    //3. 所有case都未ready，且没有default语句
    //   3.1 将当前协程加入到所有channel的等待队列
    //   3.2 当将协程转入阻塞，等待被唤醒
    //4. 唤醒后返回channel对应的case index
    //   4.1 如果是读操作，解锁所有的channel，然后返回(case index, true)
    //   4.2 如果是写操作，解锁所有的channel，然后返回(case index, false)
}
```

特别说明：对于读channel的case,如`case elem, ok := <-chan1:`, **如果channel有可能被其他协程关闭的情况下，一定要检测读取是否成功**，因为close的channel也有可能返回，此时ok == false

# 4. 总结

- select语句除了default ,每个case操作一个channel ,要么读要么写
- select除了default ,各个case是随机的
- select如果没有default,则会阻塞等待一个case
- select语句中读操作要判断是否成功读取，关闭的channel也可以读取

# range

# 1. 前言

range是Golang迭代遍历手段，可操作性的类型 `数组`,`切片`，`Map`，`Channel`

# 2. 热身

## 2.1 题目一：切片遍历

~~~go
func RangeSlice(slice []int) {
    for index, value := range slice {
        _, _ = index, value
    }
}
~~~

> 遍历过程每次迭代都会给index和value赋值，
>
> 数据量大或者value类型是string,对value的赋值是多余的
>
> for-range 忽略value值，可以用slice[index]去引用这个值

## 2.2 题目二：Map遍历

~~~go
func RangeMap(myMap map[int]string) {
    for key, _ := range myMap {
        _, _ = key, myMap[key]
    }
}
~~~

函数的for-range值只获取key值，然后根据key找到value,虽然看起来少了一次赋值，但是key值找value值消耗高于  赋值消耗

## 2.3 题目三：动态遍历

```go
func main() {
    v := []int{1, 2, 3}
    for i:= range v {
        v = append(v, i)
    }
}

//可以正常结束 
//循环次数在循环开始的时候就已经决定了
```

# 3. 实现原理

## 3.1 range for slice

~~~go
// The loop we generate:
//   for_temp := range
//   len_temp := len(for_temp)
//   for index_temp = 0; index_temp < len_temp; index_temp++ {
//           value_temp = for_temp[index_temp]
//           index = index_temp
//           value = value_temp
//           original body
//   }
~~~

遍历slice:

1.获取slice长度

2.每次循环对index,value进行一次赋值，如果for-range接受index-value  会对 index和value进行一次赋值

## 3.2 range for map

```go
// The loop we generate:
//   var hiter map_iteration_struct
//   for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
//           index_temp = *hiter.key
//           value_temp = *hiter.val
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

map没有指定循环次数，循环体和slice遍历相似，但是map的底层是hash，插入数据位置随机，所以不能保证新插入的数据可以遍历到

## 3.3 range for channel

```go
// The loop we generate:
//   for {
//           index_temp, ok_temp = <-range
//           if !ok_temp {
//                   break
//           }
//           index = index_temp
//           original body
//   }
```

channel遍历是从channel中读取数据，如果channel没有元素 则会阻塞等待，如果channel被关闭，则会解除阻塞退出循环

- 上述注释中index_temp实际上描述是有误的，应该为value_temp，因为index对于channel是没有意义的
- 使用for -range 遍历channel直接获取一个返回值

# 4. 编程Tips

- 遍历过程中可以选择不遍历index和value从此提高性能
- 遍历channel，如果channel没有数据 可能会阻塞
- 尽量避免过程中修改数据

# 5. 总结

- for-range实际实现的是C风格的for循环
- 使用index,value接收range值会进行一次数据拷贝

# mutex

# 1. 前言

互斥锁是实现并发访问共享资源的主要手段，对此Go语言提供了Mutex,对外暴露了两个方法 Lock()和 Unlock() 用于加锁和解锁

目的：掌握Mutex的集中状态 

探究Mutex重复解锁引起的Panic的原因

# 2. Mutex数据结构

```go
type Mutex struct{
    state int32
    sema uint32
}
```

- Mutex.state: 表示互斥锁的状态， 是否解锁等等
- Mutex.sema : 表示信号量， 协程阻塞等待信号量， 解锁的协程释放信号量从而唤醒等待信号量的协程

Mutex.state是**32位的整形变量**，内部实现把该变量分为4份，记录Mutex的四种状态

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_45a91868c2c9d5dc2617e9fda0e46049_r.png)

- Locked: 表示Mutex是否被锁定， 0 ：没有锁定， 1：已经被锁定
- Woken: 表示Mutex是否被唤醒 ， 0 ：没有协程唤醒， 1：已有协程唤醒，**正在加锁过程中**
- Starving: 表示Mutex是否饥饿， 0 ：没有饥饿 ，1：饥饿，协程阻塞超过1ms
- Waiter: 表示阻塞等待锁的协程个数，  协程解锁根据这个值是否需要释放信号量

协程之间实际是抢给Locket的权利，一旦能给Locket置位1，说明抢锁成功，抢抢不到就等待Mutex.sema信号量，一旦持有锁的线程解锁，等待的线程就会被唤醒

## 2.2 Mutex方法

Mutex对外提供了两个方法：

- Lock() : 加锁方法
- UnLock():  解锁方法

加锁分成功和失败：成功直接获取锁，失败则协程阻塞

# 3. 加解锁过程

## 3.1 简单加锁

只有一个协程在加锁，没有其他干扰

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_4b43555a5440890c948680aab58e982f_r.png)

加锁过程中会判断Locked是否为0，如果为0就将Locket置位1

## 3.2 加锁被阻塞

假定加锁的时候，锁已经被其他写成占用了

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_7009a47d6e8acb7e7b9c421ad1fece22_r.png)

当协程B对已经被加锁的协程基础加锁，Waiter计数器增加1，协程B被阻塞，知道Locket变为0

## 3.3 简单解锁

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_6f4885510e5659f0615f17e6a5b89d2f_r.png)

没有其他线程等待加锁，此时解锁只需把Locket位置变为0

## 3.4 解锁并唤醒协程

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_f45d385c3f7bacc272878bbfd6d48182_r.png)

协程A解锁两个不走：一是看到Locket位置为0，而是看到Waiter>0,就释放一个信号量，唤醒一个阻塞的线程，被唤醒的协程B把Locket位置换位1，协程B得到锁

# 4. 自旋过程

加锁的时候，如果Locket结果为1，说明这个锁为其他协程所持有，尝试加锁的过程不是马上阻塞，而是持续探测Locket为0，这个过程为自旋过程

自旋的过程很短，但是如果自旋的过程中发现锁已经被释放，那么协程会立即获得锁，其他被唤醒的协程也只能继续阻塞 

好处：加锁失败不需要立即转为阻塞，而是有一定的几率来获取锁，避免线程切换

## 4.1 什么是自旋

自旋对应CPU的"PAUSE"指令，CPU对该指令什么都不做相当于空转，相当于程序sleep一小段时间自旋过程中会持续探测Locket是否为0

## 4.1 自旋条件

加锁的时候自动判断是否可以自旋，无限制的自旋给CPU巨大压力

自旋必须满足以下所有条件：

- 自旋次数够小，通常为4，一般自旋四次
- CPU核数大于1，否则自旋没有意义，此时不可能有其他协程释放锁
- 协程调度机制Process > 1 ，比如高使用GOMAXPROCS（）将处理器设置为1，不能用自旋
- 协程调度机制可运行队列为空，否则延迟协程调度

## 4.2 自旋的优势

自旋的优势是更充分利用CPU，避免协程切换，因为当前申请加锁的协程拥有CPU，经过短时间的自旋获取锁，当前协程可以继续运行，不必阻塞

## 4.3 自旋的问题

如果自旋过程获取锁，那么自旋阻塞的线程无法获取锁，如果加锁的协程特别多，每次都自旋获取锁，那么阻塞的进程就很难获取锁

# 5. Mutex模式

我们前面只关注了Waite Locket , 现在我们看Starving作用

## 4.1 normal模式

默认情况，Mutex模式为Normal

该模式下，协程加锁的不成功立即转入阻塞排队，而是判断是否满足自旋的条偶见，如果满足就会自旋，尝试抢锁

## 4.2 starvation模式

自旋过程中抢到锁， 一定意味着同一时刻协程释放了锁，我们知道释放锁时如果发现有**阻塞等待的协程**，还会**释放一个信号量来唤醒一个等待协程**，被唤醒的协程得到CPU后开始运行 ，如果超过1ms的话，会将Mutex标记为”饥饿”模式，然后再阻塞

处于饥饿模式下，**不会启动自旋过程，也即一旦有协程释放了锁**，那么一定会**唤醒协程**，被唤醒的协程将会成功获取锁，同时也会把等待计数减1

# 5. Woken状态

 Woken用于加锁和解锁，同一时刻，**两个协程一个在加锁，一个在解锁**,在加锁的协程**可能在自旋过程中，此时把Woken标记为1**,不必释放信号量

# 6. 为什么重复解锁要panic

为什么Go不能实现得更健壮些，多次执行Unlock()也不要panic？

。Unlock过程分为将Locked置为0，然后判断Waiter值，如果值>0，则释放信号量。

如果多次Unlock()，那么可能每次都释放一个信号量，这样会唤醒多个协程，**多个协程唤醒后会继续在Lock()的逻辑里抢锁，势必会增加Lock()实现的复杂度**，也会引起不必要的协程切换。

# 7. 编程Tips

## 7.1 使用defer避免死锁

加锁后立即使用defer对其解锁，可以有效的避免死锁

## 7.2 加锁和解锁应该成对出现

加锁和解锁最好出现在同一个层次的代码块中，比如同一个函数。

**重复解锁会引起panic，应避免这种操作的可能性**

# rwmutex

# 1. 前言

读写互斥锁RWMutex,在某些场景可以发挥灵活的控制能力

如果程序写操作少，读操作多，如果程序一次执行写，多次读，使用Mutex，这个过程是串行的，即使N次读操作，使用Mutex, 过程是串行的，如果使用读写锁，多个读操作可以同时持有锁，并发能力将大大提升

实现读写锁需要解决如下几个问题：

1. 写锁需要阻塞写锁：一个协程拥有写锁时，其他协程写锁定需要阻塞
2. 写锁需要阻塞读锁：一个协程拥有写锁时，其他协程读锁定需要阻塞
3. 读锁需要阻塞写锁：一个协程拥有读锁时，其他协程写锁定需要阻塞
4. 读锁不能阻塞读锁：一个协程拥有读锁时，其他协程也可以拥有读锁

# 2. 读写锁数据结构

## 2.1 类型定义

`src/sync/rwmutex.go:RWMutex`定义了读写锁数据结构

~~~go
type RWMutex struct{
    w Mutex //控制多个血锁，获得写锁要先获得该锁，如果有一个写锁在运行，那么其他写锁会阻塞
    writerSem 	uint32 //写阻塞等待信号量，最后一个读者释放锁的时候会释放这个信号量
    readerSem 	uint32 //读阻塞信号量，写锁释放的时候会释放这个信号量
    readerCount int32 //读者的人数
    readerWait  int32 //写阻塞的读者人数
}
~~~

## 2.2 接口定义

RWMutex提供了4个简单的接口提供服务

- RLock():读锁定
- RUlock():解除读锁定
- WLock():写锁定
- WUlock():解除写锁定

## 2.2.1 Lock()实现逻辑

写锁定需要两件事：

- 获取互斥锁
- 阻塞等待所有读操作结束（如果有的话）

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_b31533f6f5d070460b2194aa486ee040_r.png)

## 2.2.2 Unlock()实现逻辑

解除锁定需要做的两件事：

- 唤醒因读锁定被阻塞的协程
- 解除互斥锁

所以`func (rw *RWMutex)Unlock()`  接口实现流程如图所示：

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_15371303a4b9903ccf8df1a1d3d6c761_r.png)

## 2.2.3 RLock()实现逻辑

读锁定需要的两个事情：

- 增加读计数操作，readerCount++
- 阻塞等待写操作结束

func(rw *RWMutex) RLock()

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_970f3e966f918951584d728879e613c4_r.png)

## 2.2.4 RUnlock()实现逻辑

决出读锁定两件事：

- 减少读计数的操作,readerCount--
- 唤醒写操作进程(如果有需要)

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_0a707690c6ed961a03c503a85f679225_r.png)

# 3. 场景分析

## 3.1 写操作是如何阻止写操作的

读写锁包括一个互斥锁(Mutex)，写锁定必定先获取互斥锁，如果互斥锁被协程A获取，意味着协程B只能阻塞等待互斥锁

## 3.2 写操作是如何阻止读操作的

RWMutex.readerCount是整值型，**实际上可以支持 2^30个读者**，写锁定的时候会先减2^30的读者，这是判断为负，如果读操作来了就知道有写操作，但是读者并没有丢失，只需要加上2^30的读者就可以了

## 3.3 读操作是如何阻止写操作的

读操作会先把readerCount+1,让readerCount不为0，这样写者就判断有读者进行操作了

## 3.4 为什么写锁定不会被饿死

#### 问题提出：

在写操作等待读操作的过程，这个过程仍然可能会有读操作不断到来， 如果写操作等待所有读操作结束。

写操作到来，会把RWMutex.readerCount拷贝RWMutex.readerWait，标记排在写操作前面的读者个数，**读操作结束后，递减RWMutex.readerWait值**，当RWMutex.readerWait变为0唤醒写操作

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_4cc8b286df29eeb2d3f3809977df015c_r.png)

# 协程调度

# 前言

# 1. 线程池的缺陷

在高并发应用中**频繁创建线程减少不必要的开销**，所以我们可以创建线程池，将一定数量的线程放入线程池中， 然后任务发布在任务队列，**线程池的线程不断从任务队列取出任务执行**，可以减少线程创建和销毁所需要的消耗

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_b499f154d0135854b725ce27a6b7a009_r.png)

如果worker上的线程进行了系统调用，操作系统会把该线程置为阻塞状态，**这样的话线程消费任务队列的能力就会下降**，如果任务队列**大部分任务都会系统调用**，**让该状态恶化**

# 2. Goroutine调度器

线程过多，操作系统不断的处理线程，造成了性能瓶颈，G**o中提供了一种机制，可以线程自己调度，上下文切换轻量**，从而线程数少，并发不变

Goroutine概念：

- G(goroutine):Go协程，每个go关键字都会创建一个Go协程
- M(machine):工作线程
- P(processor):处理器，包括Go代码需要的资源，也有调度goroutine的能力

M只能拥有P才能调度G，P可以包括多个G的队列

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_274ee3af62bab4ad8f74a6753d6969cf_r.png)

P的个数是在程序启动的时候决定的，默认情况为CPU的核数， M必须有一个P才能有代码，所以在运行M的同时，线程数就等于CPU数

程序可以通过`runtime.GOMAXPROCS`,设置P的个数

# 3. Goroutine调度策略

## 3.1 队列轮转

每个P维护一个G的队列**，P周期性将G调度到M中，执行一段时间，将上下文保存**，然后将G放在队列尾部

还有一个全局G队列，P会周期性查看全局G有是否待运行的调度到M里面

## 3.2 系统调用

P的个数默认为CPU核数，每个M持有一个P才能执行G，**一般M个数略大于P个数**，多出来的**M会在G产生系统调用**发挥作用

当G0进入系统调用，M0释放P,M1会接收P并且维护P里面的队列，继续执行P剩下的G

M1的来源有可能是M的缓存池，也可能是新建的。当G0系统调用结束后，根据M0是否能获取到P，将会将G0做不同的处理：

1. 如果有空闲的P，则获取一个P，继续执行G0。
2. 如果没有空闲的P，则将G0放入全局队列，等待被其他的P调度。然后M0将进入缓存池睡眠

## 3.3 工作量窃取

多个P中维护的G队列有可能是不均衡的

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_511f878a3328cf036e73368eaf864177_r.png)



然后去查询全局队列，全局队列中也没有G，而另一个M中除了正在运行的G外，队列中还有3个G待运行。此时，空闲的P会将其他P中的G偷取一部分过来，一般每次偷取一半

**（这个上面讲的是先从全局拿）**

## 4. GOMAXPROCS设置对性能的影响

一般来讲，程序运行时就将GOMAXPROCS大小设置为CPU核数，可让Go程序充分利用CPU。
在某些IO密集型的应用里，这个值可能并不意味着性能最好。
**理论上当某个Goroutine进入系统调用时，会有一个新的M被启用或创建，继续占满CPU。**
但由于Go调度器检测到M被阻塞是有一定延迟的，也即旧的M被阻塞和新的M得到运行之间是有一定间隔的，所以在IO密集型应用中不妨把GOMAXPROCS设置的大一些，或许会有好的效果

