### 1 spawn
`tokio::spawn`用于创建一个异步任务并随机放入到线程池中的一个线程中。会返回一个`JoinHandler<T>`类型。这个类型可以通过.await来获取结果（结果是个`Option<T>`类型）。
```rust
async fn add(a: i32, b: i32) -> i32 {
    println!("{}", a+b);
    a + b
}
#[tokio::main]
async fn main() {
    let join_handler1 = tokio::spawn(add(2, 3));
    let result = join_handler1.await;
    println!("Hello, world!");
}
//先打印出5，后打印出hello world。这段代码实际上是串行地执行异步任务。因为在一个异步任务后立即await会挂起当前异步线程直到一步完成。如果没用await那就先打印出hello
```
如果一个spawn创建的异步任务没用跟await，那么它就会在另一个线程中独立运行，不会挂起当前线程。这一用法适用于一些不需要异步任务返回结果的情况。
### 2 Join
多个异步任务并行执行，并且可以将所有任务都执行完成后返回的结果赋值给一个集合：
```rust
async fn add(a: i32, b: i32) -> i32 {
	a + b
}
let result = tokio::join!(add(1, 2), add(3, 4));
//result = (3, 4);
let result2 = tokio::join!(tokio::spawn(add(1, 2)), tokio::spawn(add(3, 4)));
```
result和result2的区别：
前者是单线程内协作式并发，即在一个任务线程内交替执行（如果传入的Future中都不存在await那就会变成顺序运行，因为谁都不会主动让出线程）。
后者是真正的并发+并行，每个异步函数有一个单独的任务线程来执行。
tokio提供了joinSet类型，这是一个集合类型，用于管理多个并发的异步任务。可以按照完成顺序获取结果或等待完成后获取全部结果。
```rust
let mut set = JoinSet::new();
for i in 1..10 {
	set.spawn(add(i, i+1));
}
while let Some(val) = set.join_next().await {
	println!("{val:?}");
}
```

### 3 `yield_now`
yield_now方法返回一个空的future。异步任务会在遇到await尝试让出异步运行时。通常使用`yield_now().await`来在程序没有future可供await的情况下主动让出运行时，让其他异步任务优先执行。
### 4 sleep
`tokio::time::sleep(time).await`可以让当前异步任务进入休眠状态，并将线程交给其他异步任务。
### 5 `spawn_blocking`
如果在tokio异步运行时的一个线程中执行一个需要占据大量时间来完成的同步操作，例如没有实现Future的文件IO、CPU密集计算等无法await的任务，那么该线程会完全阻塞。这被成为线程饥饿。`spawn_blocking`就是为了解决这一问题而生的。
被放入`spawn_blocking`的任务会被丢入一个独立的阻塞线程池中运行，这个线程是一个系统线程。并返回一个JoinHandler，可以通过.await实现异步获取结果。
```rust
fn delay(taskid: u32, time: u32) {
	println!("{taskid} start");
	tokio::task::spawn_blocking(move || {
		thread::sleep(Duration::from_secs(time));
	}).await;
	println!("{taskid} end");
}
#[tokio::main]
fn main() {
	join!(delay(1, 1), delay(2, 1));
}
//两个delay分别占用一个线程
//打印结果:
//1 start
//2 start
//1 end
//2 end
```
由于系统线程开销较大，我们可以指定阻塞线程池的线程数。避免创建太多系统线程。
tokio的spawn和spawn_blocking都必须在异步运行时中执行。