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
spawn创建的任务不会立即执行，而是会进入执行队列等待调度执行。
如果一个spawn创建的异步任务没用跟await，那么它就会在另一个线程中独立运行，不会挂起当前线程。这一用法适用于一些不需要异步任务返回结果的情况。
注意：如果spawn创建的异步线程的handler后面没用跟.await，那么主线程结束后。这个异步线程也会结束，不管有没有完成。
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
#### 2.1 `try_join`宏
并发执行多个返回 `Result` 的 Future，任一出错立即返回错误。
#### 2.2 `join_all`和`try_join_all`函数
这两个函数用于并发执行动态数量的并发任务。唯一区别是`try_join_all`会短路执行。
```rust
let urls = vec!["a.com", "b.com", "c.com"];
let futures:Vec<_> = urls
	.into_iter()
	.map(|url| task::spawn(fetch_url(url))).collect();
let results = futures::future::join_all(futures).await;
```
### 3 `yield_now`
yield_now方法返回一个空的future。异步任务会在遇到await尝试让出异步运行时。通常使用`yield_now().await`来在程序没有future可供await的情况下主动让出运行时，让其他异步任务优先执行。
### 4 sleep
`tokio::time::sleep(time).await`可以让当前异步任务进入休眠状态，并将线程交给其他异步任务。
### 5 `spawn_blocking`
如果在tokio异步运行时的一个任务中执行一个需要占据大量时间来完成的同步操作，例如没有实现Future的文件IO、CPU密集计算等无法await的任务，那么该任务的线程会长时间无法释放。`spawn_blocking`就是为了解决这一问题而生的。
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
### 6 select
`tokio::select!` 宏是 Tokio中用于并发等待多个异步操作的核心工具，它的核心作用是：
同时匹配多个异步分支（Future），一旦其中任意一个完成，就立即执行对应的处理逻辑，并取消其余未完成的分支。
`select!`模式匹配的语法：
`<结果> = <Future 分支表达式> => {结果处理},`
这是一个tcp客户端并发同时处理接收到消息和发送消息的示例：
```rust
let mut stream = TcpStream::connect("127.0.0.1:8080").await?; 
let (mut reader, mut writer) = stream.split(); 
let mut buf = [0u8; 1024];
loop { 
	tokio::select! { 
		//分支1 收到消息
		n = reader.read(&mut buf) => { 
			let n = n?; 
			if n == 0 { 
				break; 
			} 
			println!("Received: {:?}", &buf[..n]); 
		}
		// 分支2 发送消息
		_ = writer.write_all(b"ping\n") => { 
			println!("Sent ping"); 
			tokio::time::sleep(tokio::time::Duration::from_secs(1)).await; 
		} 
	} 
}
```
所有分支的 `Future` 同时被随机轮询检查是否完成。哪个先完成执行哪个。
作为分支条件的Future表达式后面不需要跟await。宏会自动解包Future。
#### 6.1 `select_biased`宏
与select宏类似。但`select——biased`会每次轮询时**从上到下依次检查**分支是否就绪而不是随机检查。用于处理多个权重不同的异步并发任务。
### 7 Race
race与select类似。race等待多个Future中的第一个完成，其他未完成的取消任务。这个函数来自于futures库。
race与select区别在于race不用额外处理每个Future完成的情况。只需要提供一个通用解。
```rust
use futures::future::race; 
async fn fast_service() -> String { 
	"fast".into()
} 
async fn slow_service() -> String {
	tokio::time::sleep(Duration::from_secs(5)).await;
	"slow".into()
} 
let result = race(fast_service(), slow_service()).await;
// result == "fast"
```

### 8 服务器高并发
对于并发服务器的场景，推荐的做法是使用loop循环作为进程主循环，循环中等待连接建立并使用await挂起。针对每个连接使用spawn创建一个异步线程。在异步线程中再使用loop+select的组合。针对多种不同类型的请求进行并发处理。
需要注意这些问题：
1. 资源耗尽问题：考虑连接池或限制 `spawn` 数量（用 `Semaphore` 控制并发）。
2. 连接异常：在 `select!` 中加入定时心跳发送来检测连接是否存活。
3. 共享状态：使用 `Arc<Mutex<T>>` 或更高效的 `tokio::sync::RwLock` / `watch` / `broadcast`。
4. 死锁问题：优先避免共享可变状态，使用消息传递（`mpsc`）而非Mutex等。优先使用异步锁而不是线程锁。
5. 避免主进程panic：在 `spawn` 内部加 `panic` 防护。
6. 不要在异步线程中执行大量会产生同步阻塞的代码。