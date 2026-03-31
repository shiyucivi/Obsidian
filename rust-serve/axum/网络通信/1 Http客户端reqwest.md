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

#### 2.2 反序列化

#### 2.3 流处理
