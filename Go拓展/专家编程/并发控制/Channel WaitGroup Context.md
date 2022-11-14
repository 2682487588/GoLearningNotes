# WaitGroup

# 1 前言

```go
package main

import (
    "fmt"
    "time"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    wg.Add(2) //设置计数器，数值即为goroutine的个数
    go func() {
        //Do some work
        time.Sleep(1*time.Second)

        fmt.Println("Goroutine 1 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    go func() {
        //Do some work
        time.Sleep(2*time.Second)

        fmt.Println("Goroutine 2 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    wg.Wait() //主goroutine阻塞等待计数器变为0
    fmt.Printf("All Goroutine finished!")
}
```

# 2 基础知识

## 2.1 信号量

Unix系统中保护共享资源的一个机制

简单理解为信号量为一个数值：

- 当信号量>0时，表示资源可用，获取信号量时系统自动将信号量减1；
- 当信号量==0时，表示资源暂不可用，获取信号量时，当前线程会进入睡眠，当信号量为正时被唤醒；

# 3 WaitGroup组成

## 3.1 数据结构

```go
type WaitGroup struct {
    state1 [3]uint32
}
```

state1是长度为3的数组，其中包含了state和一个信号量，state实际上是两个计数器

- counter : 还未执行结束的goroutine计数器
- waiter count : 等待goroutine-group结束的goroutine数量，有多少等待者
- semaphore:信号量

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_b68f98a52c940a8c94a7c39f1f56a901_r.png)

WaitGroup对外提供了三个接口：

- Add(delta int) 将delta值将入到counter中
- Wait():waiter递增1，并且阻塞等待信号量semaphore
- Done(): counter递减1，并且按照waiter数值释放相对应次数信号量

## 3.2 Add(delta int)

Add做了两件事

- delta值累加到counter,因为delta的值可以为负,counter有可能变为0或负值
- counter变为0，根据waiter释放等量信号量，counter变为负值则发生panic

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state() //获取state和semaphore地址指针

    state := atomic.AddUint64(statep, uint64(delta)<<32) //把delta左移32位累加到state，即累加到counter中
    v := int32(state >> 32) //获取counter值
    w := uint32(state)      //获取waiter值

    if v < 0 {              //经过累加后counter值变为负值，panic
        panic("sync: negative WaitGroup counter")
    }

    //经过累加后，此时，counter >= 0
    //如果counter为正，说明不需要释放信号量，直接退出
    //如果waiter为零，说明没有等待者，也不需要释放信号量，直接退出
    if v > 0 || w == 0 {
        return
    }

    //此时，counter一定等于0，而waiter一定大于0（内部维护waiter，不会出现小于0的情况），
    //先把counter置为0，再释放waiter个数的信号量
    *statep = 0
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false) //释放信号量，执行一次释放一个，唤醒一个等待者
    }
}
```

## 3.3 Wait()

Wait方法做了两件事，一个是累加waiter,一个是阻塞信号量

```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state() //获取state和semaphore地址指针
    for {
        state := atomic.LoadUint64(statep) //获取state值
        v := int32(state >> 32)            //获取counter值
        w := uint32(state)                 //获取waiter值
        if v == 0 {                        //如果counter值为0，说明所有goroutine都退出了，不需要待待，直接返回
            return
        }

        // 使用CAS（比较交换算法）累加waiter，累加可能会失败，失败后通过for loop下次重试
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            runtime_Semacquire(semap) //累加成功后，等待信号量唤醒自己
            return
        }
    }
}

//这里调用CAS算法保证goroutine同时调用Wait（）也可以正确累加到Wait
```

## 3.4 Done()

Done制作一件事，Add()可以接受负值，所以wg.Done就是wg.Add(-1)

```go
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

## 4 小结

`WaitGroup`通常用于一组 “工作协程”结束的场景，内部维护着两个计算器，我们称之为 “工作协程”和“坐等协程”

- `Add(delta int)` ,增加 "工作协程"计算器，启动“新的工作协程”前调用
- `Done()` 方法用于减少工作协程计数， 每次调用递减`1`,  在`工作协程`内部临近返回的时候调用
- `Wait()` 用于"坐等协程"计数，在所有"工作协程"启动之后调用

`Done()`方法递减“工作协程”计数后，如果“工作协程”计数变成负数时，将会触发`panic`，这就要求`Add()`方法调用要早于`Done()`方法

`Add()`方法累加的“工作协程”计数要与实际需要等待的“工作协程”数量一致，否则也会`panic`

“工作协程”计数多于实际需要等待的“工作协程”数量时, "坐等协程"会无法环形而发生死锁

“工作协程”计数小于实际需要等待的“工作协程”数量时， `Done()`会在 "工作协程"计数变为负数触发` panic`

# Context

# 1. 前言

Golang context 是Golang应用开发的并发控制技术，**他和WaitGroup最大的不同点是context对派生的goroutine有更强控制力**

context翻译成中文是”上下文”，即它可以控制一组呈树状结构的goroutine

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_fbdcf78fa8ee5b280dae750ff631c3d6_r.png)

# 2. Context实现原理

## 2.1 接口定义

```go
type Context
```

# 反射机制

# 1. 前言

# 2. 反射概念

官方介绍：

1. 反射提供一种让程序检查自身的能力
2. 反射是困惑的源泉

第一条，精确点，**反射是检查interface变量的底层类型和值的机制**

## 2.1 关于静态类型

Go是静态类型语言， "int float32 [] byte"等等，都是在编译的时候确定的

~~~go
type MyInt int
var a int
var b MyInt
~~~

## 2.2 特殊的静态类型interface

~~~go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
~~~

实现Read()方法就认为是Reader接口，只要实现Writer方法就被认为实现了Writer接口，**接口类型的变量可以存储任何实现该接口类型的值**

## 2.3 特殊的interface类型

特殊的interface类型是空interface类型，interface表示一组方法集合，所有实现该方法集合被认为实现该接口

**一个类型实现空interface并不重要，重要的是一个空interface类型变量可以存放所有值**

## 2.4 interface类型是如何表示的

nterface类型的变**量可以存放任何实现了该接口的值**。还是以上面的`io.Reader`为例进行说明，`io.Reader`是一个接口类型，`os.OpenFile()`方法返回一个`File`结构体类型变量，**该结构体类型实现了`io.Reader`的方法，那么`io.Reader`类型变量就可以用来接收该返回值**

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

# 3. 反射三定律

这里说interface是说interface类型有个(value,type)对，而反射就是检查这个value,type,具体一点说就是Go提供一组方法提取interface的value，提供另一组方法提取interface的type

反射包里有两个接口类型要先了解一下.

- `reflect.Type` 提供一组接口处理interface的类型，即（value, type）中的type
- `reflect.Value`提供一组接口处理interface的值,即(value, type)中的value

## 3.1 反射第一定律：反射可以将interface类型转为反射对象

~~~go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4

    v := reflect.ValueOf(x) //v is reflect.Value

    var y float64 = v.Interface().(float64)
    fmt.Println("value:", y)
}
~~~

## 3.2 反射第二定律: 反射可以将反射对象还原为interface()对象

~~~go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4

    v := reflect.ValueOf(x) //v is reflect.Value

    var y float64 = v.Interface().(float64)
    fmt.Println("value:", y)
}
~~~

## 3.3 反射第三定律：反射对象可以修改,value值必须是可以设置的

```go
package main

import (
    "reflect"
)

func main() {
    var x float64 = 3.4
    v := reflect.ValueOf(x)
    v.SetFloat(7.1) // Error: will panic.
}

//panic
//因为reflect.ValueOf(x)传递的是值，
//但是ELem方法可以给指针指向Value

```

~~~go
func main() {
	var x  float64 = 3.4
	v:=reflect.ValueOf(&x)
	v.Elem().SetFloat(7.1)
	fmt.Println("x : ",v.Elem().Interface())
}

//改为Elem.SetFloatOf进行设置Value，不会报panic
~~~



# Context

Golang context是Golang应用开发常用的并发控制技术，和WaitGroup不同的是Context对派生的goroutine有更强的控制力

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_fbdcf78fa8ee5b280dae750ff631c3d6_r.png)

# 2. Context实现原理

## 2.1 接口定义

源码包中 `src/context/context.go:Context`

~~~go
type Context interface{
    Deadline()(deadline time.Time, ok bool)
    Done() <-chan struct(){}
    
    Err() error
    Value(key interface{}) interface{}
}
~~~

### 2.1.1 Deadline()

返回一个deadline和标识是否设置deadline的bool值，**没有设置deadline的值，ok== false,deadline作为一个初始化的time.Time值**

### 2.1.2 Done()

该方法返回一个channel需要在select-case使用  "case <- context.Done()"

当context**关闭后**，Done()返回一个被关闭的通道，这时候的关闭的管道是可以读的,**goroutine可以收到关闭请求**

**当context未关闭，Done()返回一个nil**

### 2.1.3Err()

描述context关闭的原因，

- 因为deadline关闭, "context deadline exceeded"
- 主动关闭 "context conceled"

### 2.1.4 	Value()

有一种context,用于树状分数goroutine之间传递信息

## 2.2 空context

context包定义一个空的context，为emptyCtx,简单实现了Context，**本身不包括任何值，仅用于其他context的父节点**

空的context代码定义如下

~~~go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}
~~~

context包定义一个emptyCtx的全局变量，交background，可以用context.Background获取

~~~go
var background = new(emptyCtx)
func Background() Context {
    return background
}
~~~

![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_f41adc85deafd7243fc1eb3e6c553ced_r.png)



## 2.3 cancelCtx

源码包 `src/context/context.go:cancelCtx`定义了该类型的context

~~~go
type cancelCtx struct{
    Context
    mu sync.Mutex
    done chan struct{}
    children map[canceler]struct{}
    err error
}
~~~

**children 记录了这个context派生的所有child,此context被cancel会把所有child都cancel**

cancel和Deadline和Value无关 只需把Done()和Err()外露接口

### 2.3.1 Done()接口实现

~~~go

func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
~~~

> cancelCtx没有初始化函数，所以cancel.Done可能未分配
>
> 分配过程:nil -> chan struct{}  -> closed chan

### 2.3.2 Err()接口实现

~~~go
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
~~~

> cancelCtx.err 默认是nil ,在Context被cancel返回一个变量
>
> var Canceled = errors.New("Context canceled")

### 2.3.3 Cancel()接口实现

~~~go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled  已经取消
	}
	c.err = err		//设置err说明关闭原因
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.  遍历所有children逐个cancel
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {	//需要将自己从parent删除
		removeChild(c.Context, c)
	}
}
~~~

WithCancel() 返回的第二个用于cancel context 正是这个concel

### 2.3.4 WithCancel()方法实现

~~~go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)  //将自身加入父节点
	return &c, func() { c.cancel(true, Canceled) }
}
~~~

自身加入父节点

1.父节点支持cancel，父节点肯定有children成员，将新context加入children

2.父节点不支持cancel，继续查询找到一个支持cancel,将新context加入children

3.所有父节点不支持cancel，启动一个协程等待父节点结束，然后把当前context结束

### 2.3.5 典型使用案例

```go
package main

import (
    "fmt"
    "time"
    "context"
)

func HandelRequest(ctx context.Context) {
    go WriteRedis(ctx)
    go WriteDatabase(ctx)
    for {
        select {
        case <-ctx.Done():
            fmt.Println("HandelRequest Done.")
            return
        default:
            fmt.Println("HandelRequest running")
            time.Sleep(2 * time.Second)
        }
    }
}

func WriteRedis(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("WriteRedis Done.")
            return
        default:
            fmt.Println("WriteRedis running")
            time.Sleep(2 * time.Second)
        }
    }
}

func WriteDatabase(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("WriteDatabase Done.")
            return
        default:
            fmt.Println("WriteDatabase running")
            time.Sleep(2 * time.Second)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go HandelRequest(ctx)

    time.Sleep(5 * time.Second)
    fmt.Println("It's time to stop all sub goroutines!")
    cancel()

    //Just for test whether sub goroutines exit or not
    time.Sleep(5 * time.Second)
}
```

```bash
HandelRequest running
WriteDatabase running
WriteRedis running
HandelRequest running
WriteDatabase running
WriteRedis running
HandelRequest running
WriteDatabase running
WriteRedis running
It's time to stop all sub goroutines!
WriteDatabase Done.
HandelRequest Done.
WriteRedis Done.
```

## 2.4 timerCtx

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}
```

- deadline: 指定最后期限，比如context将2018.10.20 00:00:00之时自动结束
- timeout: 指定最长存活时间，比如context将在30s后结束。

### 2.4.1 Deadline()接口实现

Deadline()方法仅仅是返回**timerCtx.deadline而矣**。而timerCtx.deadline是WithDeadline()或WithTimeout()方法设置的

### 2.4.2 cancel()接口实现

cancel()方法基本继承cancelCtx，只需额外把timer关闭

- 如果deadline到来之前手动关闭，则关闭原因与cancelCtx显示一致；
- 如果deadline到来时自动关闭，则原因为：”context deadline exceeded”

### 2.4.3 WithDeadline()方法实现

- 初始化一个timeCtx实例
- 将timerCtx添加父节点的children
- 启动定时器，定时器到期会自动cancel本context
- 返回timerCtx实例和cancel()方法

### 2.4.4 WithTimeout()方法实现

WithTimeOut()调用了WithDeadLine，实际原理一样

~~~go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
~~~

### 2.4.5 典型案例

```bash
HandelRequest running
WriteRedis running
WriteDatabase running
HandelRequest running
WriteRedis running
WriteDatabase running
HandelRequest running
WriteRedis running
WriteDatabase running
HandelRequest Done.
WriteDatabase Done.
WriteRedis Done.
```

## 2.5 valueCtx

源码包中`src/context/context.go:valueCtx`：定义了类型context

```go
type valueCtx struct {
    Context
    key, val interface{}
}
```

### 2.5.1 Value()接口的实现

~~~go
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
~~~

### 2.5.2 WithValue（）方法实现

```go
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    return &valueCtx{parent, key, val}
}
```

### 2.5.3 典型使用案例

```go
package main

import (
    "fmt"
    "time"
    "context"
)

func HandelRequest(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("HandelRequest Done.")
            return
        default:
            fmt.Println("HandelRequest running, parameter: ", ctx.Value("parameter"))
            time.Sleep(2 * time.Second)
        }
    }
}

func main() {
    ctx := context.WithValue(context.Background(), "parameter", "1")
    go HandelRequest(ctx)

    time.Sleep(10 * time.Second)
}
```

- Context仅仅是一个接口定义，根据实现的不同，可以衍生出不同的context类型；
- cancelCtx实现了Context接口，通过WithCancel()创建cancelCtx实例；
- timerCtx实现了Context接口，通过WithDeadline()和WithTimeout()创建timerCtx实例；
- valueCtx实现了Context接口，通过WithValue()创建valueCtx实例；
- 三种context实例可互为父节点，从而可以组合成不同的应用形式；
