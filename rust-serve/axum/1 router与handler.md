### 1 hello world
```rust
use axum::{
	Router,
	routing::{get},
	response::{Html},
}
#[tokio::main]
async fn main() {
	let router = Router::new()
		.route("/", get(handler))
	let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();	
	println!("listening on {}", listener.local_addr().unwrap());
	axum::serve(listener, rouuter).await.unwrap();
}
async fn handler() -> Html<'static str> {
	Html("<h1>hello world</h1>")
}
```
axum需要异步main函数作为启动函数。
### 2 Router
#### 2.1 Router
`Router` 是 Axum 提供的核心类型，用于构建路由表。可以通过链式调用的方式添加路径和对应的 handler。
```rust
let app = Router::new()
    .route("/", get(root))
    .route("/foo", get(get_foo).post(post_foo))
    .route("/foo/bar", get(foo_bar));

// handlers
async fn root() {}
async fn get_foo() {}
async fn post_foo() {}
async fn foo_bar() {}
```
#### 2.2 Router常用方法
##### 2.2.1 `merge(Router)`
接收一个`Router`类型作为参数，用于将两个Router中的路由表进行合并：
```rust
let router1 = Router::new("/foo", get(foo));
let router2 = Router::new("bar", get(bar));
router1.merge(router2);
```
##### 2.2.2 `nest(Path, Router)`
将另一个 `Router` 挂载到指定前缀路径下，常用于挂载子路由的嵌套路由：
```rust
let api_router = Router::new()
	.route("/foo", get(foo));  //处理/api/foo
	.route("/bar", get(bar));  //处理/api/bar
let root = Router::new().nest("/api", api_router);
```
##### 2.2.3 `fallback(handler)`
用于指定无法匹配任何Path情况下的handler。
##### 2.2.4 `layer(middleware)`
添加中间件（日志、报文压缩、加密等）
##### 2.2.5 `with_state(state)`
注入共享状态

### 3 routing方法与handler
`axum::routing`提供了多个针对不同Http方法的函数，这些函数接收一个async函数（`Handler`）作为参数，来构造一个`MethodRouter`：
- `get(handler)`：处理 GET 请求
- `post(handler)`
- `put(handler)`
- `delete(handler)`
- `patch(handler)`
- `head(handler)`
- `options(handler)`
这些方法均实现了链式调用，用于应对同一route的多种请求方法。
#### 3.1 handler
作为Handler的函数要实现`handler` trait约束：
- 必须是async函数
- 参数必须实现 Axum 的 **extractors**（如 `Path`, `Query`, `Json`, `State` 等）
- 函数返回值必须实现`IntoResponse`。常用的`String`、`str`、`Json`、`Html`、`(StatusCode, T: IntoResponse)`等类型均已实现了这个trait。