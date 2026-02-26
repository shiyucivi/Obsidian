`Stream`是异步版本的迭代器。其核心trait定义在`futures-core`中。任何实现了Stream的类型都依赖这个trait。
```rust
pub trait Stream {
	type Item;
	fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```
在tokio生态中，通常使用`tokio-stream`和`tokio-util`中的trait来创建`Stream`和赋予`Stream`各种方法。
### 1 `tokio-stream`
`tokio-stream`支持将实现了`Iter`的类型转为`Stream`，目前`Stream`不支持for循环遍历。最常用的方法是`while let`循环和`loop`循环遍历。
```rust
use tokio_stream::StreamExt; 
#[tokio::main] 
async fn main() { 
	let mut stream = tokio_stream::iter(&[1, 2, 3]); 
	while let Some(v) = stream.next().await { 
		println!("GOT = {:?}", v); 
	} 
}
```
#### 1.1 包装器
`tokio_stream::wrapper`模块提供了将一些`stream`包装器，将一些tokio中的类型转化为`Stream`。主要包括：
- `IntervalStream` 将`tokio::time::Interval`包装为`Stream`
- `BroadcastStream` 将 `tokio::sync::broadcast::Receiver<T>` 转为 `Stream<Item = Result<T, RecvError>>`
- `ReceiverStream` 将有界 `mpsc::Receiver<T>` 转为 `Stream`|
```rust
//一个包装broadcasr::Receiver为Stream的例子，普通的Receiver只能通过revc()接收消息
use tokio::sync::broadcast;
use tokio_stream::{wrappers::BroadcastStream, StreamExt};

let (tx, _rx1) = broadcast::channel::<i32>(16);
let rx2 = tx.subscribe();
// 创建第二个接收者并包装为 Stream，并用filter_map适配器进行过滤，返回一个Stream<Item=Option<i32>>
let mut stream = BroadcastStream::new(rx2).filter_map(|res| async move {
	match res {
		Ok(msg) => Some(msg),
		Err(_) => None,
	}
});
tokio::spawn(async move {
	for i in 1..=5 {
        tx.send(i).unwrap();
        tokio::time::sleep(tokio::time::Duration::from_millis(500)).await;
    }
})
while let Some(msg) = stream.next().await {
	println!("{msg}");
}
```
使用`Stream`包装`Receiver`的好处在于可以将多个消息接收流合并，即使每个`Stream`所处理的数据类型不同。
#### 1.2 `StreamExt`
`tokio-stream`的`StreamExt` trait为处理`Stream`提供了多种迭代器方法的异步版本：`next`、`map`、`filter`、`collect`、`merge`等。
### 2 `tokio-util`
`tokio_util::io`提供了`ReaderStream`，用于将`tokio::fs`中提供的读取器转为`Stream`，方便对文件进行流式处理。
```rust
// 通过stream将一个hello.txt的内容复制到foo.txt中
use tokio::{
	fs::File,
	io::{AsyncReadExt, AsyncWriteExt}
};
use tokio_util::io::ReaderStream;
use tokio_stream::StreamExt;

let file = File::open("hello.txt").await.unwrap();
let mut writer = File::create("foo.txt").await.unwrap();
let mut stream = ReaderStream::new(file);
while let Some(chunk) = stream.next().await {
	writer.write_all(&chunk.unwrap()).await.unwrap();
}
```