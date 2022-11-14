# 协程

 线程并发执行并且共享进程的内存，

## 线程和协程

Go中,协程被认为是轻量级的线程。和线程不同，操作系统感知不到协程存在，**协程管理依赖Go语言自身提供的调度器**。

为什么Go语言不直接操作线程

- 调度方式
- 上下文切换速度
- 调度策略
- 栈的大小

### 1.调度方式

协程是用户态的，Go的协程从属于一个线程，协程和线程的关系为M:N

![image-20220610155622764](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610155622764.png)

### 2.上下文切换的速度

协程的切换快于线程，协程的切换不需要经过内核态和用户态的转换

Go中协程切换只需要少量状态和寄存器变量值

线程会保留额外的寄存器变量值

### 3.调度策略

线程调度是抢占式的，操作系统调度器为了平衡各个线程的执行周期，会定时发送中断信号，强制执行上下文切换

协程调度是协作式的，当一个协程处理完自己任务，会自动过渡到其他协程，当一个协程运行时间过长，Go处理器才会干预

### 4.栈的大小

线程的栈一般是创建的时候指定的，为了避免栈溢出，栈的空间默认分配的比较大(为2MB)，Go协程默认大小为2KB,所以可以看到大量的协程出现

## 协程示例

```go
nc main() {

   links := []string{
      "http://www.baidu.com",
      "http://www.jd.com",
      "http://www.taobao.com",
      "http://www.163.com",
      "http://www.sohu.com",
   }
   for _, address := range links {
      checkAddress(address)
   }

}

func checkAddress(address string) {
   _, err := http.Get(address)
   if err != nil {
      fmt.Println("visit address failed",err)
   }
   fmt.Println("visit "+address+" success")
}
```

~~~bash
visit http://www.baidu.com success
visit http://www.jd.com success
visit http://www.taobao.com success
visit http://www.163.com success
visit http://www.sohu.com success
# 但是这样是顺序进行，比较浪费空间
~~~

![image-20220610162609846](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610162609846.png)

## 主协程和子协程

![image-20220610163025402](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610163025402.png)

改成这样的形式发现程序退出了，因为主协程（main Goroutine）挂了,所以程序直接退出了，子协程(child Goroutine)也就挂了

![image-20220610163139896](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610163139896.png)

~~~bash
visit http://www.163.com success
visit http://www.baidu.com success
visit http://www.jd.com success
visit http://www.sohu.com success
visit http://www.taobao.com success

# 这个访问顺序是乱序的，这个是并发特性导致的，无法确定哪个协程先执行
~~~

# 深入协程设计调度原理

## 协程生命周期和状态转移

![image-20220610164819697](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610164819697.png)

- _Gidle协程刚创建的状态
- _Gdead协程被销毁的状态
- _Grunnable协程在运行队列中的状态，等待运行
- _Grunning协程正在被运行
- _Gwaiting协程运行被锁定，不能执行用户代码
- _Gsyscall 协程在系统调用
- _Gpreempted代表协程G被强制抢占后的状态
- _Gcopystack 协程栈扫描发现需要扩容或缩小栈空间

![image-20220610170709026](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610170709026.png)

协程记过g0->g->g0的过程，完成了一次调度循环，协程切换的过程叫做上下文切换。  协程执行现场存储在g.gobuf结构体，保存着几个寄存器值，分别是rsp,rip,rbp

- **rsp**:指向函数调用栈栈顶
- **rip**:指向程序执行的下一条指令
- **rbp:**存储函数栈帧起始位置

特殊协程g0和执行用户代码的流程有所不同，g0为了避免栈溢出进而可以重复使用

![image-20220610171332713](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610171332713.png)

## 线程本地存储和线程绑定

![image-20220610173850397](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610173850397.png)

## 调度策略

调度策略的核心在schedule中

Go调度器把运行队列分为局部运行队列和全局运行队列

局部队列是每个P特有的长度为256的数组，模拟一个循环队列，其中runqhead标识循环队列开头，runqtail标识循环队列队尾，  **每次放G都从循环队列末尾插入**

![image-20220610201413352](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610201413352.png)

![image-20220610201421843](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610201421843.png)

一般情况是先执行局部队列，然后执行全局队列的G，这样有一个问题，**全局队列的G可能不会运行**

Go语言调度器策略：P每执行61次，就要从全局队列获取一个G到当前P，作为下一次执行的G

![image-20220610201845113](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610201845113.png)

每个P在执行，都会先从runnext中获取下一个G

如果runnext为空，就从P局部队列runq找需要执行的G

局部队列为空，尝试从全局队列取G

如果全局队列没有要执行的G，尝试从其他的P中偷取可用G

如果获取不到，当前的P和M解除绑定，P会放在P的空闲队列

![image-20220610202356042](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610202356042.png)

### 1.获取本地运行队列

调度器(P)首先查看runnext是否为空，不为空就返回G

为空就在局部运行队列查找，当循环队列的头(runqhead)和尾(runqtail)相同，以为没有运行的协程，否则，循环队列从头部获取一个协程返回

存在部分P窃取任务导致访问的情况，因此需要加锁

### 2.获取全局运行队列

**当P执行61次**或者 **局部运行队列不存在可用的协程**

这时候每个P都共享全局队列，因此为了保证公平，先把P的数量平均分全局运行队列的G，然后转移数量不超过局部队列的一半（256/2=128），

然后通过循环调用runqput将全局队列G放入P的局部队列

![image-20220610213029678](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610213029678.png)

**如果本地队列满了，会把本地队列的一半放到全局队列**

![image-20220610213053798](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220610213053798.png)



### 3.获取准备就绪的网络协程

如果全局、本地都找不到，调度器会找当前准备好的网络协程，会找是否准备号的网络协程

### 4.协程窃取

局部运行队列，全局运行队列以及准备就绪的网络列表都找不到可以用的协程，需要从其他P的本地队列获取

所有的P存储在 allp []*p,Go采取findrunnable函数

