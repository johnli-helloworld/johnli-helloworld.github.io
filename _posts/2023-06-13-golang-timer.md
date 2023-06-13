---
layout:     post
title:      golang Timer应用
subtitle:   Timer
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Go
    - filecoin
---

这里可以看看我在filecoin提的一个pr;
> 地址：[Reuse timers in sealing batch logic](Reuse timers in sealing batch logic)

围绕这个pr我们来说说Timer的应用


 重复触发：

[func NewTicker(d Duration) *Ticker](https://pkg.go.dev/time#NewTicker) ， 返回一个会发送timer的channel的Ticker.调用它的Stop可以释放相关的资源。

[func Tick(d Duration) <-chan Time](https://pkg.go.dev/time#Tick)  ， Tick 只是对于NewTicker 的一个简单包装，提供了一个获取ticking的channel.当调用者不需要关闭这个Ticker的时候使用它，而且它不能被GC回收， 所以会"泄漏",小心使用。(有点不明白golang 为啥还要提供这个函数)

一次性触发：

[func NewTimer(d Duration) *Timer](https://pkg.go.dev/time#NewTimer), 创建一个timer , 当过了duration以后会往它自己的channel 中发送当前时间。

[func After(d Duration) <-chan Time](https://pkg.go.dev/time#After), 当过了duration 时间段后会往channel 中发送当前时间，只有当它到了指定的duration 后， 才会被GC掉。  如果为了保持高效，建议用NewTimer来配合Stop 来做。(这个After又是一个对newtimer 的包装的感觉，不需要你显示的调用Stop)

func AfterFunc(d Duration, f func()) *Timer, 在等待duration 时间后，在一个goroutine 当中调用f函数。返回值是一个Timer, 可用Timer的stop 来取消调用f函数。

 下面总结几个time 函数调用的例子：

 例子一：
 ```golang
 go demo(input)
 
func demo(input chan interface{}) {
    for {
        select {
        case msg <- input:
            println(msg)
 
        case <-time.After(time.Second * 5):
            println("5s timer")
 
        case <-time.After(time.Second * 10):
            println("10s timer")
        }
    }
}
 ```
上面是摘抄过来的一个case， 本意在间隔5秒或者10秒后，触发一次性动作。 但是当input的msg 进入速度足够快的时候，5秒和10秒的动作无法触发。 因为time.after是一次性的动作，当下一次for 循环时又重新产生新的after。 同时这样还会造成很多time.After生成的timer无法回收，直到这些timer达到了触发条件才会回收掉。  这样在一个for 循环场景中就达不到目的。

 改进办法一， 是用newtimer ,或者newticker 来操作。 
 ```golang
func demo(input chan interface{}) {
    t1 := time.NewTimer(time.Second * 5)
    t2 := time.NewTimer(time.Second * 10)
 
    for {
        select {
        case msg <- input:
            println(msg)
 
        case <-t1.C:
            println("5s timer")
            t1.Reset(time.Second * 5)
 
        case <-t2.C:
            println("10s timer")
            t2.Reset(time.Second * 10)
        }
    }
    //  如果for 循环会break 的话，需要调用t1, t2 的Stop() 方法
    t1.Stop()
    t2.Stop()
}
 ```
上面的例子OK了, 但是会引申另外一个问题。关于timer的reset 的问题。

 以前用timer 的reset ,就简单认为在任何时候都可以直接做timer的reset就是重置timer了. 但是看了golang 的reset的文档，有这样的描述：
```golang
 // Reset changes the timer to expire after duration d.
// It returns true if the timer had been active, false if the timer had
// expired or been stopped.
//
// Resetting a timer must take care not to race with the send into t.C
// that happens when the current timer expires.
// If a program has already received a value from t.C, the timer is known
// to have expired, and t.Reset can be used directly.
// If a program has not yet received a value from t.C, however,
// the timer must be stopped and—if Stop reports that the timer expired
// before being stopped—the channel explicitly drained:
//
// if !t.Stop() {
// <-t.C
// }
// t.Reset(d)
//
// This should not be done concurrent to other receives from the Timer's
// channel.
//
// Note that it is not possible to use Reset's return value correctly, as there
// is a race condition between draining the channel and the new timer expiring.
// Reset should always be invoked on stopped or expired channels, as described above.
// The return value exists to preserve compatibility with existing programs.
```
第二段的大意是说：如果一个程序已经接收过了t.C的数据，那么这个timer就是已知过期的了，那么可以直接用t.Reset来重置时间。但是， 如果一个程序还未接收过t.C的数据，那么就要先调用timer的Stop 方法来停止这个timer, 如果Stop方法返回false, 表明这个timer 在我们之前就已经过期了，那么要显式的消费掉这个t.C里面的数据，再去做reset重置。

下面代码可以证明如果不显式的消费掉t.C的内容， 会有什么样的结果：
```golang
func main() {
       t1 := time.NewTimer(1 * time.Second)
       fmt.Printf("newtimer , now is :%v \n",time.Now())
       time.Sleep(3 * time.Second)
       fmt.Printf("after sleep 3 sec, now is :%v , and we let t1 expire , and reset 2 sec \n",time.Now())
       t1.Reset(2 * time.Second)
       t1C := <- t1.C
       fmt.Printf("after reset 2 sec , we get t1C :%v , and now is :%v \n",t1C, time.Now())
       t2C := <- t1.C
       fmt.Printf("after reset , we get t2C :%v , and now is :%v \n",t2C,time.Now())
}
```
结果如下：

```
newtimer , now is :2018-04-19 23:09:03
after sleep 3 sec, now is :2018-04-19 23:09:06 , and we let t1 expire , and reset 2 sec
after reset 2 sec , we get t1C :2018-04-19 23:09:04 , and now is :2018-04-19 23:09:06
after reset , we get t2C :2018-04-19 23:09:08 , and now is :2018-04-19 23:09:08 
```

1. 首先做了一个1秒后触发的timer， 但是不马上消费，而是去sleep 3秒，就是为了让这个timer 过期。

2. t1.Reset 重置2秒， 马上去消费t1.C. 结果就是马上得到数据, 从time.Now()的23:09:06 知道是跟上一句的时间一样的。

    而且消费到的数据是23:09:04的数据，这时间点就是第一步设置的reset 1秒的结果。

3.  再次消费t1.C给t2C, 这时候是等了2秒才拿到数据，time.Now()的时间为23:09:08. 

所以这样证明了，golang 的注释中所说的， 每次reset要调用Stop方法来判断是否这个time 已经过期。如果为false , 就表明这个已经过期，要显式的消费掉t.C的数据。再去设置t.Reset，不然会造成timer时间与自己预期不符合。

但是事情还没结束，如果简单的按照注释来的话，在某种场景下，会有下面这样的结果，代码如下：

```golang
func main() {
       t1 := time.NewTimer(2 * time.Second)
       <- t1.C
       fmt.Printf("newtimer , now is :%v \n",time.Now().Format("2006-01-02 15:04:05"))
       //time.Sleep(3 * time.Second)
       // 如果stop 为false , 表明timer已经过期，或者已经停止。要消费掉t1.C的数据。
       // 但是这个时候t1.C已经没有数据往里发送了。
       if !t1.Stop() {
              <- t1.C
       }
       t1.Reset(1 * time.Second)
       fmt.Printf("finish \n ")
}
```

结果如下：

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
timer3.go:17 +0x182
newtimer , now is :2018-04-19 23:29:31
exit status 2
```

死锁了， 为什么， 因为刚开始已经消费了t1.C的数据， 在判断 t1.Stop 的时候就为false. 如果又去消费t1.C, 但是悲剧的没有timer给t1.C发送了。所以deadlock.

可以用以下方法来避开：
```golang
func main() {
       t1 := time.NewTimer(2 * time.Second)
       <- t1.C
       fmt.Printf("newtimer , now is :%v \n",time.Now().Format("2006-01-02 15:04:05"))
       if !t1.Stop() {
              select {
                     case <- t1.C:
                     default:
              }
       }
       t1.Reset(1 * time.Second)
       fmt.Printf("finish \n ")
}
```

到这里，应该算是可以用好timer了，后面来分析timer的生成方法。