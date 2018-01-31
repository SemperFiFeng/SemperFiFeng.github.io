
- - - -
layout: post
title: "`golang`中管道的使用"
date: 2018-01-30 23:32:54.000000000 +09:00
tags: 技术篇
- - - -
#区块链/区块链博客

* 管道是什么？
管道(`channel`)是Go语言在语言级别上提供的**线程间**的**通讯方式**，我们可以使用channel在多个线程之间传递消息。`channel`是**进程内**的通讯方式，是不支持跨进程通信的，如果需要进程间通讯的话，可以使用`Socket`等网络方式。
* 管道的创建方式
```golang
c := make(chan int, 5) //缓冲管道 可以放入多个数据  
c := make(chan int) 	  //非缓冲管道 放入一个数未读取 再放入阻塞
```
其中`make`为生成语句，`chan`为管道类型关键字（`channel`的前四个字母），`chan`后的`int`是在制定管道中存放数据的类型。
在缓冲管道声明中`,`后面的参数为`capacity`指定的是管道的数据容量。这个参数可以通过下行代码来查看数值
```golang
fmt.Println(cap(c))
```
而实际管道中放入的数值个数可以通过以下代码来查看
```golang
fmt.Println(len(c))
```
* 向管道中写入数据
```golang
c <- 10  //注意数据类型的匹配，与声明中保持一致
```
* 取出管道中写入的数据
```golang
<-c  		//取数据
a := <-c //将管道里面取出来的值赋给a
```
* 管道数据读取与写入的阻塞。
管道需要读取写入操作依次进行，否则会产生阻塞。例如一个非缓冲管道，从未写入的管道中读取数据会阻塞，直到真正向其中写入数据才可以读取。向已经写入数据的管道中写入数据也会阻塞，等待一个读取操作完成后可以继续进行写入操作。如下代码块
```golang
	c := make(chan int)
	wg.Add(2)
	go func() { //子线程
		for i := 0; i < 10; i++ {
			c <- i //写入数据
			fmt.Println("写入数据")
		}
		wg.Done()
	}()
	go func() {
		for i := 0; i < 10; i++ {
			a := <-c
			fmt.Println(a) //读取数据
			fmt.Println("读取数据")
		}
		wg.Done()
	}()
	wg.Wait()
```
两个线程会交替完成读写的操作，此时实际上是两个子线程间的数据通信。
```终端执行结果
localhost:0130 Fong$ go run 009_demo.go 
0
读取数据
写入数据
写入数据
1
读取数据
2
读取数据
写入数据
写入数据
3
读取数据
4
读取数据
写入数据
写入数据
5
读取数据
6
读取数据
写入数据
写入数据
7
读取数据
8
读取数据
写入数据
写入数据
9
读取数据
localhost:0130 Fong$ 
```
* `close`与`range`结合实现类似队列的功能，例如 实现在主线程结束之前完成其它子线程的功能。如下代码
```golang
	c := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			c <- i
		}
		close(c)
	}()
	for n := range c {
		fmt.Println(n)
	}
	// num, ok := <-c   测试管道是否被关闭，如果ok返回的false则关闭
```
输出的结果：
```运行终端输出
localhost:0130 Fong$ go run 011-demo.go 
0
1
2
3
4
5
6
7
8
9
localhost:0130 Fong$ 
```

在`range`遍历中，`n`能够不断的读取`c`里面的数据，直到`for`执行完毕，通过下一步的`close(c)`关闭管道。这个过程可以看到数据实际上是从`go`开启的子线程中到达了主线程的`range`处，这个便是主线程和子线程间数据的传递过程。


* 通过队列和管道的方式分别实现多个分线程对管道进行读取操作后并关闭管道
如下代码在队列中加入两个子线程，在第三个子线程中等待结束队列，关闭管道。
```golang
	c := make(chan int)
	var wg sync.WaitGroup //队列
	wg.Add(2)
	go func() {
		fmt.Println("111111111")
		for i := 0; i < 10; i++ {
			c <- i
		}
		wg.Done()
	}()

	go func() {
		fmt.Println("22222222")
		for i := 10; i < 20; i++ {
			c <- i
		}
		wg.Done()
	}()
	go func() {
		fmt.Println("33333333")
		wg.Wait()
		close(c)
	}()

	for n := range c {
		// <-c  有返回值
		fmt.Println(n)
	}
```
如下代码，`<-done`的执行时在等待前面任意一个子线程`for`执行完成后进行写入操作，继而可以读取。当`<-done`两个语句都执行完毕后，两个子线程都执行完毕。
```golang
	c := make(chan int)
	done := make(chan bool)

	go func() {
		for i := 0; i < 10; i++ {
			c <- i
		}
		done <- true
	}()
	go func() {
		for i := 0; i < 10; i++ {
			c <- i
		}
		done <- true
	}()
	<-done //死锁
	<-done
	close(c)
	for n := range c {
		fmt.Println(n)
	}
```