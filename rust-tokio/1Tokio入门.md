tokio是rust的标准异步库。
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
默认情况下，`tokio::main`会为每个cpu核心启动一个线程（可修改线程数量）。
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