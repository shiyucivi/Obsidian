### 1 Service
tower生态中的`Service`是一种强大的中间件trait。`Service`的定义如下：
```rust
trait Service<Request> { 
	type Response; 
	type Error; 
	type Future: Future<Output = Result<Self::Response, Self::Error>>; 
	// 1. 准备阶段：检查服务是否准备好接收请求（背压机制） 
	fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>; 
	// 2. 调用阶段：处理请求并返回一个 Future 
	fn call(&mut self, req: Request) -> Self::Future; 
}
```
`Service`与普通handler的区别：
- **有状态**: `&mut self` 意味着 Service 可以持有可变状态（如数据库连接池、限流计数器、缓存）。
- **背压 (Backpressure)**: `poll_ready` 允许服务控制输出的速率。
- **通用性**: 它操作的是原始的 `http::Request<B>` 和 `http::Response<P>`，不关心具体的业务参数提取。
`tower_http`所提供的中间件本质上就是`Service`。`ServiceBuilder` 内部会将这些 Layer 堆叠起来，最后在`Router.layer`中生成具体的 Service 实例。
### 1 `TraceLayer`日志中间件

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/hello", get(hello_handler))
    .layer(TraceLayer::new_for_http());
```
这个中间件会字段为每个http请求的handler创建一个span。这个span的默认名称是`request`，还包括`method`、`url`、`status`等字段信息。
这个中间件可以使用`make_span_with`来自定义每个http请求对应span的名称、字段等信息。
```rust
use tower_http::trace::{TraceLayer, DefaultMakeSpan};
use tracing::Level;

let trace_layer = TraceLayer::new_for_http()
    .make_span_with(|req: &Request<_>| {
        // 自定义 Span 名称和字段
        tracing::info_span!(
            "http_request", // 这是 Span 的名称
            method = %req.method(),
            uri = %req.uri(),
            version = ?req.version(),
            route = "/hello", // 自定义字段
        )
    });
```
### 2 请求id中间件
为每个请求生成独一无二的id并记录在span中
```rust
use tower::ServiceBuilder;
use tower_http::{
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},
    trace::TraceLayer,
};
// 设置自定义header
static REQUEST_ID_HEADER: &'static str = "x-request-id";
let request_id_header = HeaderName::from_static(REQUEST_ID_HEADER);
let middleware = ServiceBuilder::new()
	// 生成Id的layer
	.layer(SetRequestIdLayer::new(request_id_header.clone(), MakeRequestUuid))
	.layer(
		TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
			// 记录生成的request id
			let request_id = request.headers().get(REQUEST_ID_HEADER);
			match request_id {
				Some(request_id) => info_span!(
					"http_request",
					request_id = ?request_id,
				),
				None => {
					error!("could not extract request_id");
					info_span!("http_request")
				}
			}
		})
	)
	// 将与request相同的header设置给response的header
	.layer(PropagateRequestIdLayer::new(request_id_header));;
```
### 3 压缩
使用gzip进行压缩和解压：
```rust
use tower::ServiceBuilder;
use tower_http::{
	compression::CompressionLayer,
	decompression::RequestDecompressionLayer
};
Router::new()
	.route("/", post(root))
	.layer(
		ServiceBuilder::new()
			// 请求进入进行解压
			.layer(RequestDecompressionLayer::new())  
			// 响应发送进行压缩
			.layer(CompressionLayer::new())
	)
```
### 4 CORS跨域
```rust
use axum::{
    http::{HeaderValue, Method},
    response::{Html, IntoResponse},
    routing::get,
    Json, Router,
};
use tower_http::cors::CorsLayer;
let back_end = {
	let router = Router::new()
		.route("/json", get(json))
		.layer(
			CorsLayer::new()
				.allow_origin("http://localhost:3000".parse::<HeaderValue>().unwrap())
				.allow_methods([Method::GET])
		);
	serve(tokio::net::TcpListener::bind("127.0.0.1:4000").await.unwrap(), router)

};
async fn json() -> impl IntoResponse {
    Json(vec!["one", "two", "three"])
}
```
