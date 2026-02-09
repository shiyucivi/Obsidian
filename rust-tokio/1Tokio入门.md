### 1 runtime
可以在main函数中使用tokio的runtime作为异步运行时。更方便的做法是使用`tokio::main`
```rust
async run() {
	for i in 1..10 {
	    println!("{i}");
	}
}
#[tokio::main]
async fn main() {
    run().await
}
//main宏的本质是在main函数内部执行了一个异步运行时
```
默认情况下，`tokio::main`会为每个cpu核心启动一个工作线程（可修改线程数量），这些线程组成了一个线程池。
tokio运行时会默认为每个线程分配一个执行器（executor，任务队列）和一个反应器（reactor，即事件循环）。当执行器中的一个任务进入阻塞时，就会放入反应器。阻塞停止时再进入执行器。
同时tokio也提供了一些常用工具的异步版本（IO、FS、网络等）。
### 2 绿色线程与系统线程
tokio默认创建的线程是绿色线程。我们可以使用`tokio::spawn`将异步任务调度到tokio的线程池中并发执行：
```rust
async run() {
	for i in 1..10 {
	    println!("{i}");
	}
}
async hi() {
	println!("hi");
}
#[tokio::main]
fn main() {
	tokio::spawn(run()); //进入到tokio线程池中执行
	hi().await; //一个串行异步任务
}
//hi和run这两个异步函数的执行顺序是随机的，因为tokio默认会随机分配线程来执行这两个异步任务
```
绿色线程是区别于系统线程的概念。系统线程由操作系统来协调运行，每个线程的内存占用大，线程切换开销高。绿色线程则由异步运行时来协调，开销更小，但对于多核的利用率较差，更适合执行高并发任务。
Tokio的默认工作线程池（worker threads）是为 **非阻塞、协作式异步任务** 设计的。这些线程通过轮询 `Future` 来实现高并发。
### 3 异步任务
Tokio的默认工作线程池（worker threads）会共享的一个任务队列。spawn发起的任务会被推入这个队列中。线程池会才用抢占的方式执行这些任务，如果任务在执行中预到await，则挂起任务并放回到任务队列中，同时线程也会去抢占其他任务。因此每个异步任务的执行线程是不固定的。
虽然可以在装载了异步运行时的main函数中以`future.await`的方式直接运行一个async函数或async块。但这种方式运行的异步代码没有进入到tokio的异步线程池中，本质上还是以串行方式顺序执行。只有通过`tokio::spawn`才能将异步任务放入线程池中。这两种方法对于Future对象即async函数有着不同要求：
直接.await会将当前执行栈的上下文挂起，可以在async块中通过闭包引用当前作用域中的变量。
`tokio::spawn` 要求 Future 满足 ==`'static + Send`==，`'static`要求这个异步任务必须是`async fn`或者一个move闭包。而Send要求异步任务中所有可能会让出线程的节点（即出现.await的地方）的上下文中不能存在`!Send`的数据。例如RefCell。
```rust
tokio::spawn(async { 
	let rc = Rc::new("hello"); // `rc` 在 `.await` 后还被继续使用，因此它必须被作为任务的状态保存起来 
	yield_now().await; 
	// 事实上，注释掉下面一行代码，依然会报错 
	// 原因是：是否保存，不取决于 `rc` 是否被使用，而是取决于 `.await`在调用时是否仍然处于 `rc` 的作用域中 
	println!("{}", rc); // rc 作用域在这里结束 
});
```