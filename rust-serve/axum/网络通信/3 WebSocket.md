websocket通信的核心在于利用`axum::extract::ws::WebSocketUpgrade`来升级Http连接，以及处理升级后的websocket流（类似于socket流可以分割为读取流和写入流）。
### 1 处理路由与升级协议
```rust
use axum::{
	extract::ws::{Message, WebSocket, WebSocketUpgrade}, 
	response::IntoResponse, 
	routing::get, 
	Router, 
};
let app = Router::new() 
	.route("/", get(|| async { "Hello, World!" }))
	.route("/ws", get(ws_handler)); // 绑定 WebSocket 路由
// websocket连接升级
async fn ws_handler(ws: WebSocketUpgrade) {
	// on_upgrade 会接管 socket，并在升级成功后调用handle_socket处理流
	ws.on_upgrade(handler_socket)
}
```
### 2 处理WebSocket连接
这是一个简单的回声处理
```rust
use axum::{
	extract::ws::{Message, WebSocket}
}
async fn handler_socker(mut socket: WebSocket) {
	while let Some(msg) = socker.recv().await {
		let msg = match msg {
			Ok(m) => m,
			Err(e) => {
				println!("接收消息出错: {:?}", e); 
				return; // 客户端断开连接
			}
		}
	}
	// 判断客户端消息类型
	match msg {
		// 文本消息
		Message::Text(t) => {
			let res = socket.send(Message::Text(t)).await;
			if res.is_err() {
				println!("发送消息失败，客户端可能已断开");
				return
			}
		},
		// 处理二进制数据... 
		Message::Binary(data) => { 
			println!("收到二进制数据: {:?}", data); 
		},
		// 客户端主动关闭连接
		Message::Close(_) => { 
			return; 
		},
		// 其他情况
		_ => {}
	}
}
```
### 3 拆分读写流
`WebSocket`流是实现了`Reader`和`Writer`的`Stream`。可以进行分割处理。
```rust
use futures_util::{StreamExt};
async fn handler_socker(mut socket: WebSocket) {
	let (mut tx, mut rx) = socket.split();
	while let Some(Ok(msg)) = rx.next().await {
		match msg {
			Message::Text(text) => {
				tx.send(Message::Text(text.to_string().into)).await.ok();
			},
			Message::Close(_) => {
				break;
			},
			_ => {}
		}
	}
}
```
注意，拆分出来的写入流`tx`是不能复制多处使用的。使用`Arc<Mutex>`锁包裹也是不推荐的做法，这存在严重的性能瓶颈和稳定性风险：一个慢客户端将会拖垮所有发送端。
如果一个写入流需要处理多个场景，推荐的做法是创建一个`mpsc`或`broadcast`通道，在通道的接收端进行处理：
```rust
let (mut tx, mut rx) = ws.split();
// 创建mpsc发送端和接收端
let (mpsc_tx, mut mpsc_rx) = tokio::sync::mpsc::channel::<String>(32);

// 用于发送心跳的发送端
let headerbears_tx = mpsc_tx.clone();

// 处理websocket接收信息
let recv_task = tokio::spawn(async move {
	while let Some(Ok(msg)) = rx.next().await {
		match msg {
			Message::Text(text) => {
				mpsc_tx.send(text.to_string()).await.ok();
			},
			Message::Close(_) => {
				break;
			},
			_ => {}
		}
	}
});

// 在这里集中进行所有websocket发送
let send_task = tokio::spawn(async move {
	while let Some(msg) = mpsc_rx.recv().await {
		match tx.send(Message::Text(msg.into())).await {
			Ok(_) => {},
			Err(_) => {}
		}
	}
});

// 心跳保活发送
let keep_alive = tokio::spawn(async move {
	loop {
		 tokio::time::sleep(std::time::Duration::from_secs(10)).await;
		// 模拟发送心跳，当错误时关闭
		if headerbears_tx.send("Ping from Server".to_string()).await.is_err() {
			break;
		};
	}
});

tokio::select! {
	_ = recv_task => {},
	_ = send_task => {},
	_ = keep_alive => {}
};
```