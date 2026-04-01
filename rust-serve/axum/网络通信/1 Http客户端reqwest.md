reqwest是一个用于发起Http请求的客户端，可以用于跨服务端通信。
### 1 Http客户端
`reqwest::Client::new()`可以创建一个Http客户端。客户端可以使用`get`方法或`post`方法向url发送数据和请求报文。
推荐将`client`放在全局State中使用。
```rust
let client = reqwest::Client::new();
let app = axum::Router::new().with_state(client);

//发送get请求
let res = client.get("http://127.0.0.1:3000/stream").send().await?;

// 发送json
let response = client
	.post("https://httpbin.org/post")
	.json(&json!({ "key": "value", "language": "Rust" }))
	.send().await?;

// 序列化为json并发送
#[derive(serde::Serialize)] 
struct UserPayload { 
	username: String, 
	age: u8,
}
let payload = UserPayload { 
	username: "alice".to_string(), 
	age: 30
};
let response = client
	.post("https://httpbin.org/post")
	.json(&payload) // 注意：这里传入的是引用 &payload 
	.send().await?;
```
### 2 处理请求结果
请求得到的res是一个`Response`类型。有多种方法可以处理。
#### 2.1 检查状态
使用`response.status`方法可以直接获取状态码，这个状态等同于`axum::http::StautsCode`中提供的状态码。也可以使用`status.is_success()`方法以及`status.is_client_error()`快速检查状态类型。
#### 2.2 反序列化
response的`.json`方法可以进行反序列化处理：
```rust
#[derive(serde::Deserialize)]
struct User { 
	username: String, 
	age: u8,
}
let user: User = response.json();
```
如果具体结构字段未知，可以反序列化为`serde_json::Value`动态类型：
```rust
let json_value: Value = response.json().await?; 
// 通过键名访问数据 
if let Some(title) = json_value["title"].as_str() { 
	println!("动态获取标题: {}", title); 
}
```
#### 2.3 流处理
##### 2.3.1 使用`bytes_stream()`
`response.bytes_stream()` 会返回一个 `impl Stream<Item = Result<Bytes>>`。可以使用`Stream`相关的方法进行遍历等操作：
```rust
use futures::StreamExt;

let response = client.get("https://httpbin.org/stream/5").send().unwrap(); 
// 获取流 
let mut stream = response.bytes_stream(); 
// 循环读取数据块 
while let Some(chunk_result) = stream.next().await { 
	let chunk = chunk_result.unwrap(); // 处理可能的 IO 错误 
	println!("收到数据块 ({} 字节): {:?}", chunk.len(), chunk); 
	// 在这里处理 chunk，例如写入文件或解析 JSON 行 
}
```
##### 2.3.2 流式转发
可以构建为`axum`中的`Body`进行转发处理
```rust
let stream = response.bytes_stream();
use axum::{
    body::{Body, Bytes},
    response::{IntoResponse, Response},
}
// 构建axum的Response
let mut response_builder = Response::builder().status(response.status());
let stream = response.bytes_stream();
let body = Body::from_stream(stream);
response_builder.body(body).unwrap();
```
##### 2.3.3 使用`tokio`的IO流处理
```rust
use futures::TryStreamExt;
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use tokio_stream::wrappers::ReceiverStream;

let mut file = File::create("foo.txt").await.unwrap();
let response = client.get("https://www.rust-lang.org").send().await.unwrap();
let mut stream = response.bytes_stream();

stream.try_for_each(|chunk| async {
	file.write_all(&chunk).await.unwrap();
}).await.unwrap();
```

