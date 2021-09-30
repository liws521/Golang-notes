chan
===

## 1.前言
- channel是Golang在语言层面提高的goroutine间的通信方式, 比Unix管道更易用也更轻便.
- channel主要用于进程内各goroutine间的通信, 如果需要跨进程通信, 建议使用分布式系统的方法来解决.

## 2.chan数据结构
- src/runtime/chan.go, hchan定义了channel的数据结构
- 可以看出channel由队列, 类型消息, goroutine等待队列组成
```go
type hchan struct {
	qcount   uint           // total data in the queue, 当前队列中剩余元素的个数
	dataqsiz uint           // size of the circular queue, 环形队列长度, 即可以存放的元素个数
	buf      unsafe.Pointer // points to an array of dataqsiz elements, 环形队列指针
	elemsize uint16         // 每个元素的大小
	closed   uint32         // 标识关闭状态
	elemtype *_type // element type, 元素类型
	sendx    uint   // send index, 队列下标, 指示元素写入时存放到队列中的位置
	recvx    uint   // receive index, 队列下标, 指示元素从队列的该位置读出
	recvq    waitq  // list of recv waiters, 等待读消息的goroutine队列
	sendq    waitq  // list of send waiters, 等待写消息的goroutine队列

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex // 互斥锁, chan不允许并发读写
}
```
### 2.1 环形队列
- chan内部实现了一个环形队列作为其缓冲区, 队列的长度是创建chan时指定的
- 下图展示了一个可缓存6各元素的channel示意图
![](..\images\channel环形队列.png)
- dataqsiz说明队列长度为6, 即可缓存6个元素
- buf指向队列的内存, 队列中还剩余2个元素
- qcount表示队列中还有2个元素
- sendx指向下一个数据往哪写, 是个下标, 取值[0, 6)
- recvx指向下一个数据从哪读, 是个下标, 取值[0, 6)

### 2.2 等待队列
- 从channel读数据, 如果channel缓冲区为空或者没有缓冲区, 当前goroutine会被阻塞
- 想channel写数据, 如果channel缓冲区已满或者没有缓冲区, 当前goroutine会被阻塞
- 被阻塞的goroutine会挂在channel的等待队列中
	- 因读阻塞的会被向channel写入数据的goroutine唤醒, 反之同理
- 一般情况下, recvq和sendq至少有一个为空, 只有一个例外, 那就是同一个goroutine使用select语句向channel一边写一边读

### 2.3 类型信息
- 一个channel只能传递一种类型的值, 类型信息存储在hchan数据结构中
	- elemtype代表类型, 用于数据传递过程中的赋值
	- elemsize代表类型大小, 用于在buf中定位元素位置

### 2.4 锁
- 一个channel同时仅允许被同一个goroutine读写

## 3. channel读写
### 3.1 创建channel
- 创建channel的过程实际上是初始化hchan结构, 其中类型信息和缓冲区长度由make语句传入, buf的大小由元素大小和缓冲区长度共同决定.
- 创建channel的伪代码如下
```go
func makechan(t *chantype, size int) *hchan {
	var c *hchan
    c = new(hchan)
    c.buf = malloc(元素类型大小*size)
    c.elemsize = 元素类型大小
    c.elemtype = 元素类型
    c.dataqsiz = size
    return c 
}
```
### 3.2 向channel写数据
- 向一个channel中写数据简单过程如下
	- 如果等待接收队列recvq不为空, 说明缓冲区中没有数据或者没有缓冲区, 此时之间从recvq取出G, 并把数据写入, 最后把该G唤醒, 接收发送过程
	- 如果缓冲区中有空余位置, 将数据写入缓冲区, 结束发送过程
	- 如果缓冲区中没有空余位置, 将待发送数据写入G, 将当前G加入sendq, 进入睡眠, 等待被读goroutine唤醒
![](..\images\channel写数据流程图.png)
### 3.3 从channel读数据
- 如果等待发送队列sendq不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程；
- 如果等待发送队列sendq不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；
- 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
- 将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒；
![](..\images\channel读数据流程图.png)
### 3.4 关闭channel
- 关闭channel时会把recvq中的G全部唤醒, 本该写入G的位置为nil, 把sendq中的G全部唤醒, 但是这些G会panic
- 除此之外, panic出现的常见场景还有
	- 关闭值为nil的channel
	- 关闭以及被关闭的channel
	- 向以及关闭的channel写数据
## 4. 常见用法
## 4.1 单向channel
- 顾名思义, 单向channel指只能用于发送或接收数据, 实际上也没有单向channel
- 我们知道channel可以通过参数传递，所谓单向channel只是对channel的一种使用限制，这跟C语言使用const修饰函数参数为只读是一个道理。
	- func readChan(chanName <-chan int)： 通过形参限定函数内部只能从channel中读取数据
	- func writeChan(chanName chan<- int)： 通过形参限定函数内部只能向channel中写入数据
### 4.2 select
- 使用select可以监控多channel，比如监控多个channel，当其中某一个channel有数据时，就从其读出数据。
```go
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
```
- select语句的多个case执行顺序时随机的, select的实现原理后面专门谈
- select语句的case语句读channel不会阻塞, 尽管channel中没有数据, 这是由于case语句编译后调用读channel时会明确传入不阻塞的参数, 此时读不到数据时不会将对当前goroutine加入到等待队列, 而是直接返回
### 4.3 range
- 通过range可以持续从channel中读出数据，好像在遍历一个数组一样，当channel中没有数据时会阻塞当前goroutine，与读channel时阻塞处理机制一样。
- 注意：如果向此channel写数据的goroutine退出时，系统检测到这种情况后会panic，否则range将会永久阻塞。

## resource
[Golang RoadMap](https://www.golangroadmap.com/books/goexpert/chan.html#_1-%E5%89%8D%E8%A8%80)