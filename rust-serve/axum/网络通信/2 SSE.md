SSE是一种HTTP长连接通信。浏览器通过 JavaScript 的 `EventSource` API 向服务器发起一个普通的 HTTP GET 请求。服务端收到后会一直保持连接状态，并可以推送时间给客户端。
服务器在响应客户端的初始请求时，必须包含以下 HTTP 响应头：
- **`Content-Type: text/event-stream`**: 这是最关键的头，它告诉浏览器这是一个 SSE 数据流，需要使用 `EventSource` API 来解析。
- **`Cache-Control: no-cache`**: 禁止缓存，确保客户端接收到的都是服务器的最新数据。
- **`Connection: keep-alive`**: 明确指示保持连接活跃。
服务器推送的消息报文必须遵循固定格式；
- **`data:`** (必需): 消息的主体内容。只能是文本类型。
- **`event:`** (可选): 自定义事件类型。如果指定，客户端会触发对应名称的事件监听器，而不是默认的 `message` 事件。
- **`id:`** (可选): 消息的唯一 ID。浏览器会记住这个 ID，并在自动重连时通过 `Last-Event-ID` 请求头发送给服务器，用于实现断点续传。
- **`retry:`** (可选): 建议客户端在连接断开后，等待多少毫秒再尝试重连。
一个典型的SSE推送：
```text
id: 101 
event: notification 
data: {"message": "您有一条新消息"}
```
### `Axum sse handler`
对于SSE接口的handler，需要在函数中生成一个持续产生数据的流Stream。然后包裹在`axum::response::sse::Sse`中作为返回值。它会自动设置SSE所需的响应头。
这个流中的每一项（`Stream::Item`类型）通常是`axum::response::sse::Event`类型。这个类型中定义要推送的消息：
```rust
use axum::{
    response::sse::{Event, Sse},
};
use axum_extra::TypedHeader;
use futures_util::stream::{self, Stream};
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
	let router = Router::new()
		.route("/sse", get(sse_handler))
}

async fn sse_handler(TypedHeader(user_agent):TypeHeader<headers::UserAgent>) 
	-> Sse(impl Stream<Item=Result<Event, Infallible>>)
{
	let stream = stream::repear_with(|| Event::default().data("hi"))
		.map(Ok)
		.throttle(Duration::from_secs(1));
	Sse::new(stream).keep_alive(
		sse::KeepAlive::new()
			.interval(Dutayion::from_secs(1))
			.text("keep-alive-text")
	)
}
```
注意`Sse`类型需要设置心跳保活，它会定期向客户端发送指定文本，防止代理服务器或浏览器因连接长时间无数据而将其关闭。