## 1.Golang“调度器”的由来

###  1.单进程时代不需要调度器

一切的程序只能串行发生

### 2.多进程/线程时代有了调度器需求

进程拥有太多的资源，进程的**创建、切换、销毁，都会占用很长的时间**，CPU虽然利用起来了，但如果**进程过多，CPU有很大的一部分都被用来进行进程调度了**

### 3.协程来提高CPU利用率

为**每个任务创建线程不现实**,

> #### N:1关系

N个协程绑定1个线程，优点就是**协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速**。但也有很大的缺点，1个进程的所有协程都绑定在1个线程上

缺点：

- 某个程序用不了**硬件的多核加速能力**
- 一旦某协程阻塞，造成**线程阻塞**，本进程的其他协程都无法执行了，根本就**没有并发的能力了。**

> #### 1:1 关系

1个协程绑定1个线程，这种**最容易实现**。协程的调度都由CPU完成了，不存在N:1缺点，

缺点：

- 协程的创建、删除和切换的代价都由CPU完成，有点略显昂贵了

> #### M:N关系

M个协程绑定1个线程，是N:1和1:1类型的结合，**克服了以上2种模型的缺点，但实现起来最为复杂**

协程跟线程是有区别的，线程由CPU调度是抢占式的，**协程由用户态调度是协作式的**，一个协程让出CPU后，才执行下一个协程

### 4.Go语言的协程goroutine

**Go为了提供更容易使用的并发方法，使用了goroutine和channel**。goroutine来自协程的概念，让一组可复用的函数运行在一组线程之上，即使有协程阻塞，该线程的**其他协程也可以被`runtime`调度**，转移到其他可运行的线程上。最关键的是，**程序员看不到这些底层的细节，这就降低了编程的难度，提供了更容易的并发**

Goroutine特点：

- 占用内存更小（几kb）
- 调度更灵活(runtime调度)

### 5.被废弃的goroutine调度器

G: goroutine 协程

M: thread 线程

老调度器是如何执行的：

**M执行，放回G都需要访问全局G队列，并且M有多个**，多线程访问同意资源需要保证互斥/同步，所以全局G队列有互斥锁进行保护

老调度器有**几个缺点**：

1. **创建、销毁、调度G都需要每个M获取锁**，这就形成了**激烈的锁竞争**。
2. M转移G会造成**延迟和额外的系统负载**。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了**很差的局部性**，因为G’和G是相关的，最好放在M上执行，而不是其他M’。
3. 系统调用(CPU在M之间的切换)导致频繁的线程阻塞和取消阻塞操作增加了系统开销。

## 2.Goroutine调度器的GMP模型的设计思想

![image-20220306165218754](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220306165218754.png)

**Processor，它包含了运行goroutine的资源**，如果线程想运行goroutine，必须先获取P，P中还包含了可运行的G队列

### 1.GMP模型

**线程是运行goroutine的实体，调度器的功能是把可运行的goroutine分配到工作线程上**

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/16-GMP-%E8%B0%83%E5%BA%A6.png)

1. **全局队列**（Global Queue）：存放等待运行的G。
2. **P的本地队列**：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G’时，G’优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列。
3. **P列表**：所有的P都在程序启动时创建，并保存在数组中，最多有`GOMAXPROCS`(可配置)个。
4. **M**：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列**拿**一批G放到P的本地队列，或从其他P的本地队列**偷**一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。

#### 有关P和M的个数问题

1.P的数量：

- 由启动时环境变量`$GOMAXPROCS`或者是由`runtime`的方法`GOMAXPROCS()`决定。这意味着在程序执行的任意时刻都只有`$GOMAXPROCS`个goroutine在同时运行。

2.M的数量

- go语言本身的限制：go程序启动时，会设置M的最大数量，默认10000.但是内核很难支持这么多的线程数，所以这个限制可以忽略。
- runtime/debug中的SetMaxThreads函数，**设置M的最大数量**
- 一个M阻塞了，会创建新的M

#### P和M何时会被创建

1.P何时创建：确定p最大数量n后，p就会创建

2.M何时会创建:没有足够的M来关联P并且运行其中可以运行的G的时候，回去找空闲的M,如果没有找到，那么就需要创建

### 2.调度器的设计策略

#### 1.work stealing

本线程没有可以运行的G,允许从其他线程绑定的P来偷取G

### 2.handoff 

本线程因为G系统调用阻塞，线程会释放绑定的P，把P转移到空闲的线程执行

**并行：**  **GOMAXPROCS** 设置P的数量，最多GOMAXPROCS线程分布在多个CPU运行，这限制了并发的程度

**抢占：**  

Go，中一个goroutine最多占用10ms

goroutine和coroutine区别

goroutine:并发执行，多线程下运行，可抢占，channel通信

coroutine:顺序执行，单线程运行，不可抢占,yield和resume操作

****

**全局G队列：**调度器有全局G队列，如果M执行work stealing偷不到其他P的G,可以从全局队列获取G

### 3.go func() 调度流程

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/18-go-func%E8%B0%83%E5%BA%A6%E5%91%A8%E6%9C%9F.jpeg)

- 1.通过go func 创建goroutine

- 2.两个储存G的队列，一个是**局部调度器P的本地队列**，一个是**全局G队列**，新建的G会保存在P的本地队列中
- 3.G在M中运行，一个M持有一个P，**M和P  1:1 关系**，M会从P的本地弹出一个可以运行的G，如果P的本地队列为空，就从其他的MP组合偷取一个G
- 4.M调度G执行的是**循环机制**
- 5.**M执行一个G如果发生了syscall或其他阻塞操作**，M会阻塞，如果当一些G执行，runtime会把这个线程M从P中删除，**然后创建或者找空闲线程服务这个P**
- 6.M调用结束，G会获取空闲的P执行，并放到这个P的本地队列，如果获取不到P，这个M会休眠，加入到空闲线程

### 4.调度器的生命周期

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/17-pic-go%E8%B0%83%E5%BA%A6%E5%99%A8%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

**M0**

M0是启动程序后编号为0的主线程，M对应实例会在全局变量runtime.m0,不需要heap分配，M0负责初始化操作和启动第一个G，之后M0就和其他的M一样

**G0**

G0是每次启动M都会创建goroutine,G0用于**负责调度的G**， G0**不指向任何可执行的函数**，每个M都会有都会有一个自己的G0，在调度或系统调用会**使用G0栈空间**，全局变量的G0是M的G0

**过程：**

1.runtime创建 m0 和 goroutine g0 并把二者关联

2.调度器初始化:初始化m0 , 栈， 垃圾回收和初始化GOMAXPROCS

3.示例代码的main函数是`main.main`，`runtime`中也有1个main函数——`runtime.main`，代码经过编译后，`runtime.main`会调用`main.main`，程序启动时会为`runtime.main`创建goroutine

4.启动m0,m0绑定了P，会从P本地获取G

5.G拥有栈，M根据G的栈信息和调度信息设置运行环境

6.M运行G

7.G退出，再次回到M获取可运行的G，这样重复下去，直到`main.main`退出，`runtime.main`执行Defer和Panic处理，或调用`runtime.exit`退出程序

调度器的生命周期几乎占满了一个Go程序的一生，`runtime.main`的goroutine执行之前都是为调度器做准备工作，`runtime.main`的goroutine运行，才是调度器的真正开始，直到`runtime.main`结束而结束

### 5.可视化GMP编程

## 3.Go调度器调度场景过程全解析

**自旋锁：**自旋锁如果在申请资源的时候，其他资源正在占用，自旋锁会进行循环等待资源，但是自旋锁不会像互斥锁一样引起线程休眠

### 1.场景1

P拥有G1，M1获取P开始运行G1，**G1使用go func()创建G2**，局部性G2加到P1的本地队列

 ![img](https://www.topgoer.cn/uploads/golangxiuyang/images/26-gmp%E5%9C%BA%E6%99%AF1.png)

### 2.场景2

G1运行之后，M上的goroutine切换G9,G0负责调度协程的切换，从P本地获取G2,从G0切换到G2，并运行G2

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/27-gmp%E5%9C%BA%E6%99%AF2.png)

### 3.场景3

假设P的本地队列只能存3个G，G2创建了6个G，前三个G加入了p1本地队列，p1本地队列满了

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/28-gmp%E5%9C%BA%E6%99%AF3.png)

### 4.场景4

G2在创建G7的时候，发现P1的本地队列已满，需要执行**负载均衡**(把P1中本地队列中前一半的G，还有新创建G**转移**到全局队列)

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/29-gmp%E5%9C%BA%E6%99%AF4.png)

### 5.场景5

G2创建G8时，P1的本地队列未满，所以G8会被加入到P1的本地队列。

![null](https://www.topgoer.cn/uploads/golangxiuyang/images/30-gmp%E5%9C%BA%E6%99%AF5.png)

### 6.场景6

规定：**在创建G时，运行的G会尝试唤醒其他空闲的P和M组合去执行**。

![null](https://www.topgoer.cn/uploads/golangxiuyang/images/31-gmp%E5%9C%BA%E6%99%AF6.png)

假定**G2唤醒了M2，M2绑定了P2，并运行G0**，但P2本地队列没有G，M2此时为自旋线程**（没有G但为运行状态的线程，不断寻找G）**。

### 7.场景7

M2尝试从全局队列(GQ) 取G放到P2的本地队列，M2从全局队列取出的G符合下面的公式

~~~bash
n = min(len(GQ)/GOMAXPROCS+1,len(GQ/2))
~~~

至少取一个G，但不要取太多，给其他留点，这是**从全局队列到P本地队列的负载均衡**

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/32-gmp%E5%9C%BA%E6%99%AF7.001.jpeg)

 假定我们场景中一共有4个P（GOMAXPROCS设置为4，那么我们允许最多就能用4个P来供M使用）。所以M2只从能从全局队列取1个G（即G3）移动P2本地队列，然后完成从G0到G3的切换，运行G3

### 8.场景8

假设G2一直在M1上运行，经过2轮后，M2已经把**G7、G4从全局队列获取到了P2的本地队列并完成运行**，全局队列和P2的本地队列都空了,如场景8图的左半部分

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/33-gmp%E5%9C%BA%E6%99%AF8.png)

**全局队列已经没有G，那m就要执行work stealing(偷取)：从其他有G的P哪里偷取一半G过来，放到自己的P本地队列**。P2从P1的本地队列尾部取一半的G，本例中一半则只有1个G8，放到P2的本地队列并执行

### 9.场景9

G1本地队列G5、G6已经被其他M偷走并运行完成，当前M1和M2分别在运行G2和G8，M3和M4没有goroutine可以运行，M3和M4处于**自旋状态**，它们不断寻找goroutine。

![null](https://www.topgoer.cn/uploads/golangxiuyang/images/34-gmp%E5%9C%BA%E6%99%AF9.png)

 为什么要让m3和m4自旋，自旋本质是在运行，线程在运行却没有执行G，就变成了浪费CPU. 为什么不销毁现场，来节约CPU资源。因为创建和销毁CPU也会浪费时间，我们**希望当有新goroutine创建时，立刻能有M运行它**，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费CPU，所以系统中最多有`GOMAXPROCS`个自旋的线程(当前例子中的`GOMAXPROCS`=4，所以一共4个P)，多余的没事做线程会让他们休眠

### 10.场景10

假定当前除了M3和M4为自旋线程，还有M5和M6为空闲的线程(没有得到P的绑定，注意我们这里最多就只能够存在4个P，所以**P的数量应该永远是M>=P, 大部分都是M在抢占需要运行的P**)，G8创建了G9，G8进行了**阻塞的系统调用**，M2和P2立即解绑，P2会执行以下判断：如果P2本地队列有G、全局队列有G或有空闲的M，P2都会立马唤醒1个M和它绑定，否则P2则会加入到空闲P列表，等待M来获取可用的p。本场景中，P2本地队列有G9，可以和其他空闲的线程M5绑定

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/35-gmp%E5%9C%BA%E6%99%AF10.png)

### 11.场景11

G8创建了G9，假如G8进行了**非阻塞系统调用**

![img](https://www.topgoer.cn/uploads/golangxiuyang/images/36-gmp%E5%9C%BA%E6%99%AF11.png)

 M2和P2会解绑，但M2会记住P2，然后G8和M2进入**系统调用**状态。当G8和M2退出系统调用时，会尝试获取P2，如果无法获取，则获取空闲的P，如果依然没有，G8会被记为可运行状态，并加入到全局队列,M2因为没有P的绑定而变成休眠状态(长时间休眠等待GC回收销毁)

**总结**

**Go调度本质是把大量的goroutine分配到少量线程上去执行，并利用多核并行，实现更强大的并发**

