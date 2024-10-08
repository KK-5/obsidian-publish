## Context

context 是一个用于在程序各个组件之间传递请求范围数据以及管理请求超时、取消和截止的包。它是一个用于**处理并发、异步操作和请求**的上下文环境。 context 包提供了 Context 类型，它是一个用于传递请求范围数据的对象。通过创建和传递 Context，你可以在应用程序的不同部分之间传递请求相关的元数据，如请求 ID、用户身份验证信息、日志记录器等。

更详细的说明：[https://pkg.go.dev/context](https://pkg.go.dev/context)

经常看到这种写法：

```
	for {
		select {
		case <-ctx.Done():
			return
		case a := <-chanXXX:
			DoSomething(a)
		}
	}
```

select语句用于处理多个通道的并发操作，它可以在多个通道间进行非阻塞的选择操作。比如上面的语句，select位于一个死循环中，如果收到context结束的消息时，就跳出循环，这个context结束的消息一般会在别的goroutine中发出；在收到context结束的消息之前，就一直从chanXXX通道中取值赋给a，然后对a做一些操作，一般会有另一个goroutine在为chanXXX通道赋值。

select会阻塞当前goroutine，也就是说，如果没有收到ctx.Done()或chanXXX通道数据，当前的程序会阻塞，一直到能收到一个信息为止。

## sync

sync 包是 Go 语言标准库中提供的用于同步（Synchronization）的包，主要用于控制并发访问和操作的同步。

使用示例：

```
	wg := sync.WaitGroup{}
	wg.Add(1)
	go func() {
		defer wg.Done()
		listener.Run(workerCtx)
	}()
	wg.Wait()
```

首先，sync.WaitGroup{} 创建了一个 sync.WaitGroup 类型的变量 wg，接下来，wg.Add(1) 将等待组的计数增加了1，这告诉 wg，我们要等待一个 goroutine 完成执行。 然后，go func() { ... }() 启动一个匿名的 goroutine，该 goroutine 中的代码将在后台并发执行。 在匿名函数的开头，defer wg.Done() 语句用于在匿名函数执行完成时调用 wg.Done()，表示一个 goroutine 执行完成，这样，当这个匿名的 goroutine执行完成时，使用 wg.Wait() 的地方会继续执行。 所以这段代码的含义是：阻塞当前goroutine，等待listener.Run(workerCtx)执行完成。