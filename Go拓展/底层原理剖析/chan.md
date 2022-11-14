# 通道基本使用方式

通道是Go语言中的一等公民，其操作方式比较简单和形象。如下所示，将箭头（←）作为操作符进行通道的读取和写入。本节将详细介绍通道的基本使用方式。

## 通道声明与初始化

通道声明：

~~~go
var name chan T
~~~

通道形式：

- chan T
- chan <- T
- <-chan T

不带T不限制读写，带T 的限制读写

~~~go
chan int
chan <- float
<- chan string
~~~

未初始化的chan在编译运行不会报错，但是显然不会写入或者读取任何数据

对通道进行赋值，需要make操作符

![image-20220611143223791](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220611143223791.png)



## 2.通道写入数据![image-20220611144431214](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220611144431214.png)

无缓冲通道可以读取数据的前提是，有另外一个协程在读取通道

无缓冲通道的读写应该位于不同的协程

## 3通道读取数据

读取通道有两个返回值，第二个返回值为布尔类型，返回false代表当前通道关闭

~~~go
data,ok:= <-c
~~~

## 4.通道关闭

~~~go
close(c)
~~~

## 5. 通道作为参数和返回值

通道是协程之间的交流方式， 不管是数据读取还是写入数据，都需要将代表通道的变量通过函数传递到所在协程中去



通道作为返回值一般用于创建通道的阶段，createWorker函数创建通道C

~~~go
func createWorker(id int) chan int{
    c := make(chan int)
    go worker(id,c)
    return c
}
~~~

Go中通道是 **引用类型**而不是 **值类型**，传递到其他写成的通道，实际引用了一个通道

## 6. 单方向通道

chan <- float表示只能写入浮点数

<- chan string 表示只能读取而不能写入字符串

第一个协程完成

1. ~~~go
   
   func checkLink(link string, c chan string) {
   	_, err := http.Get(link)
   	if err != nil {
   		fmt.Println(link, "is up!")
   		return
   	}
   	fmt.Println(link, "is up!!")
   	c <- "is up"
   }
   func main(){
       //onetowardschan
   	link := []string{
   		"http://www.baidu.com",
   		"http://www.jd.com",
   		"https://ww.taobao.com",
   	}
   
   	c := make(chan string)
   	for _, l := range link {
   		go checkLink(l, c)
   	}
   	<-c
   }
   ~~~

> main刚开始不会接受数据而阻塞,当第一个子协程完成之后,main接受了数据不会阻塞，进而就完成了进程

![image-20220611165017911](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220611165017911.png)

## 7. 循环检查网址的状态链接

```go
package main

import (
   "fmt"
   "net/http"
)

func ForChan() {
   links := []string{
      "http://www.baidu.com",
      "http://www.jd.com",
      "https://ww.taobao.com",
   }

   c := make(chan string)
   for _, link := range links {
      go checkLinks(link, c)
   }
   for {
      go checkLinks(<-c, c)
   }
}

func checkLinks(link string, c chan string) {
   _, err := http.Get(link)
   if err != nil {
      fmt.Println(link, "might be donw!")
      c <- link
      return
   }
   fmt.Println(link, "is up!!")
   c <- link
}
```

如果将

~~~go
   for {
      go checkLinks(<-c, c)
   }
~~~

写成

~~~go
	for l := range c{
		go func() {
			time.Sleep(time.Second)
			checkLinks(l, c)
		}()
	}
/*http://www.baidu.com is up!!
http://www.jd.com is up!!
https://ww.taobao.com is up!!
https://ww.taobao.com is up!!
https://ww.taobao.com is up!!
https://ww.taobao.com is up!!*/
~~~

l变量是一个固定地址，相当于一个引用，每次读到的字符串都会赋值给l，后面变量会覆盖前面变量的值

# Select多路复用

select用于多个通道和多个协程通信的情况

我们不希望因为一个通道读写陷入阻塞，select可以解决这个问题

![image-20220611174705884](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220611174705884.png)

## select随机选择

~~~go
package Chan16_3

import "fmt"

func SelctManyTimesUse() {
	c := make(chan int, 1)
	c <- 1
	select {
	case <-c:
		fmt.Println("Hello,KuGou")
	case <-c:
		fmt.Println("Hello,QQMusic")

	}
}
//Hello,KuGou
//Hello,QQMusic
//Hello,QQMusic
//Hello,QQMusic
~~~

## select堵塞与控制

为了避免阻塞，select都有default分支

 ![image-20220613144658143](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613144658143.png)

**可以通过case定时来实现定时控制**

![image-20220613144753251](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613144753251.png)

## select和nil

一个nil的通道，不论读取和读入都陷入阻塞

当select的case对nil通道操作，case分支将永远得不到执行

~~~go
package Chan16_3

import "fmt"

func SelectAndNil() {
	a := make(chan int)
	b := make(chan int)
	go func() {
		for i := 0; i < 121; i++ {
			select {
			case a <- 2:
				a = nil
			case b <- 3333:
				b = nil


			}
		}
	}()
	fmt.Println(<-a)
	fmt.Println(<-b)
}
~~~

上述协程，一旦写入管道，就将通道置位nil，导致没有机会执行case,达到交替写入a,b通道

# 通道底层原理

## 通道结构和环形队列

~~~go
typ hchan struct{
    qcount uint		    //当前使用的数量
    dataqsiz uint       //最大容纳空间
    buf unsafe.Pointer	//对于有缓冲通道需要记录缓冲区
    elemsize uint 		//每个空间占用的长度
    closed uint32		//关闭状态
    elemtype *_type     //类型元数据
    sendx uint			//当前发送位置
    recvx uint			//当前接收位置
    recvq waitq			//接收队列
    sendq waitq			//发送队列
    lock mutex			//锁
}
~~~

![image-20220613151237085](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613151237085.png)

循环到队列末尾，send会置为0，防止下一次写入0号位置

这意味着，如果通道满了，再次写入数据陷入等待，知道第0号位移出![image-20220613151925495](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613151925495.png)

## 通道初始化

通道初始化调用makechan函数

第1个代表通道类型，带二个表示通道元素大小

makechan会判断元素大小，对齐，会在内存上分配元素大小

![image-20220613153021417](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613153021417.png)

##  通道写入原理

 c< - 5 调用 chansend执行

![image-20220613153358300](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613153358300.png)



### 有正在等待读取的协程

通道hchan的recvq存储了等待的协程

每个协程对应sudog，包含准备获取协程中的元素指针

![image-20220613153752846](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613153752846.png)

### 缓冲区有空余

直接向缓冲区写入当前元素

![image-20220613154056141](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613154056141.png)

### 缓冲区无空余

通道无缓冲或者当前缓冲区满了，代表sudog结构需要放入sendq链表末尾，党旗前协程陷入睡眠

![image-20220613154220713](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613154220713.png)

## 通道读取原理

![image-20220613155844020](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613155844020.png)

### 有正在写入的协程

直接从写入协程链表获取第1个 协程，将写入的元素写入当前协程，唤醒被阻塞的写入协程

![image-20220613160130882](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613160130882.png)

### 缓冲区有元素

读取缓冲区的数据，写入当前读取协程

### 缓冲区五元素

当前协程的sudog放入recvq链表末尾，当前协程陷入休眠状态

![image-20220613160818959](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613160818959.png)
