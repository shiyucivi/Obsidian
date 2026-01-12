stream就用于在异步任务之间传递的数据流。是`Iterator`的异步版本。Iterator有一个同步的`next`方法用于逐个地遍历每个元素。stream也有一些异步方法来逐个遍历。
```rust
trpl::run(async {
  let vals = vec![1, 2, 3, 4, 5, 6, 7, 8];
  let iter = vals.iter().map(move |n| n * 2);
  let mut stream = trpl::stream_from_iter(iter);
  while let Some(value) = stream.next().await {
    println!("{value}");
  }
})
```
## 1 消息流
Stream的本质上就是Futures的集合。常用于大文件读取、网络消息队列等场景。
```rust
let (tx, rx) = trpl::channel();
 trpl::spawn_task(async move {
  let messages: [&str; 8] = ["a", "b", "c", "d", "E", "f", "g", "H"];
  for (index, message)  in messages.into_iter().enumerate() {
    trpl::sleep(Duration::from_millis(if index%3 == 0 {200} else {100})).await;
    tx.send(String::from(message)).unwrap();
  }
});
let reciver = ReceiverStream::new(rx); //需要转化为流接收器，可以使用next方法异步接收消息
while let Some(val) = message.next().await {
  println!("{val}");
}
```
## 2 Stream合并

