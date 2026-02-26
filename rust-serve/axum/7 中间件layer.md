`Router::layer`方法可以让我们在处理全部或部分请求时，为每个请求接口增加公共的处理方法。常用来进行加解密、压缩、校验等。
`layer`方法的参数必须是实现了`tower::Layer` trait的类型。
Axum中的中间件非常多样，这里只介绍基础类型的中间件构造方法。
### 1 中间件执行顺序
```rust
let app = Router::new()
    .route("/", get(handler))
    .layer(layer_one)
    .layer(layer_two)
    .layer(layer_three);
```
执行顺序为`layer_three` -> `layer_two` -> `layer_one` -> `handler` -> `layer_one` -> `layer_two` -> `layer_one`。
中间件中可以提前返回响应，不执行下一步的处理逻辑。
`tower::ServiceBuilder`可以将多个中间件合并为一个：
```rust
use tower::ServiceBuilder;
app.layer(
        ServiceBuilder::new()
            .layer(layer_one)
            .layer(layer_two)
            .layer(layer_three),
    );
```
#### 1.1 为部分路由或子路由设置中间件
注意，`layer`方法会为作用于`Router`中的所有路由，即使只为被嵌套的子路由设置了`layer`。
为部分路由设置中间件，需要`route_layer`方法搭配`nest`方法：
```rust
//用户认证中间件函数
async fn require_auth<B>( req: Request<B>, next: Next<B> ) -> Result<Response, Response> {
    if req.headers().get("Authorization").is_some() {
        Ok(next.run(req).await)
    } else {
        Err((axum::http::StatusCode::UNAUTHORIZED, "Missing auth").into_response())
    }
}
// 需要认证的路由
let api_app = Router::new()
	.router("/profile", get(|| async { "User profile" }))
	.route_layer(middleware::from_fn(require_auth));
// 不需要认证的公共路由
let public_app = Router::new()
	.router("/login", get(|| async { "Login success" }));
let app = Router::new().nest("/api", api_app).nest("/public", public_app);
```
### 1 `from_fn`和`from_extractor`构造中间件
#### 1.1 `from_fn`
使用`axum::middleware::from_fn`可以将一个async函数包装为一个中间件函数。被包装的async函数有两个参数：`Request`和`Next`，`Request`就是请求提取器。`Next`则是下一步处理的包装器，`Next`拥有一个异步方法`run`，代表被所有下一步处理后返回的数据即`Response`。
```rust
use axum::{
    http::{Request},
    middleware::{Next, from_fn},
    response::{IntoResponse, Response},
    routing::post, Router,
};
let app = Router::new().route("/", post(|| async move { "Hello from `POST /`" }))
	.layer(middleware::from_fn(print_request_response));
async fn print_request_response<B>(req: Request<B>, next: Next<B>) -> Response {
	let (parts, body) = req.into_parts();
	// 处理body
	let res = next.run(req).await;
	// 处理res
	res;
}
```
作为参数的async函数有这些要求：
- 可以使用`FromRrequestParts`提取器作为前几个参数（也可以不使用）
- 倒数第二个参数`request`必须是`IntoRequest`，这个参数会消费请求的body
- 最后一个参数必须是`Next`
- 返回值必须是`impl IntoResponse`的类型
`from_fn`适合简单、快速、适合逻辑不复杂的中间件，如日志、认证、数据统计等场景。对于拥有复杂状态和生命周期的中间件需要考虑使用`tower::Service`构建。
##### 1.1.1 在`from_fn`中消费body
作为中间件的函数中倒数第二个`request`参数是拥有对整个请求数据的所有权的，因此调用`next.run`执行下一步操作时，必须传入原本的`req`参数。否则下一步会接收不到。
如果在中间件函数中消费了body，必须对body进行重组：
```rust
use http_body_util::BodyExt;

let (parts, body) = request.into_parts();
//对body进行了消费
let bytes = body
	.collect().await
	.map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()).into_response())?
	.to_bytes();
// 重组body
let req = Request::from_parts(parts, Body::from(bytes));
// 交给下一步处理
next.run(req).await;
```
#### 1.2 `from_extractor`
`from_extractor`允许把一个`extractor`类型（实现了`FromRequest`或`FromRequestParts`）作为中间件。这个中间件只关心提取是否成功，不成功则提前返回响应。不保留提取结果也不关心后续处理。
```rust
// 将一个自定义extractor RequireAuth作为`from_extractor`中间件
use axum::{
    extract::{FromRequestParts, Json},
    middleware::from_extractor,
    routing::{get, post},
    Router,
    http::{header, StatusCode, request::Parts},
};
struct RequireAuth;
impl<S: Send+Sync> FromRequestParts<S> for RequireAuth {
	type Rejection = StatusCode;
	async fn from_request_parts(parts: &mut Parts, state: &s) -> Result<Self, Self::Rejection> {
		let auth_header = parts.headers.get(header:AUTHORIZATION)
			.and_then(|value| value.to_str().ok());
		match auth_header {
			Ok(auth_header) if token_is_valid(auth_header) => Ok(Self)
			_ => Err(StatusCode::UNAUTHORIZED)
		}
	}
}
fn token_is_valid(token: &str) -> bool {
    // 鉴权处理
}
fn handler(Json(params): Json<Params>) {
	//handler
}
let app = Router::new()
    .route("/foo", post(handler))
    .route_layer(from_extractor::<RequireAuth>());
```
### 2 `tower::Service`自定义中间件

