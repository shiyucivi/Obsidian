axum规定所有handler函数的返回值类型必须实现了`IntoResponse` trait。axum会自动调用 `into_response` 方法将其转换为最终的 `Response` 类型。所有响应默认的状态码为200。
### 1 基础类型
axum默认为单元类型`()`、`String`(`&str`)、`Vec<u8>`、`Bytes`实现了`IntoResponse`，无需手动转换。
String的默认`content-type`为`text/plain; charset=utf-8`，`Vec<u8>`和`Bytes`默认的`content-type`为`application/octet-stream`。
```rust
use axum::{
	response::IntoResponse,
}
use tokio::{
	fs::File,
	io::{AsyncReadExt, AsyncWriteExt},
}
fn handler() -> String {
	"get a message".to_string()
}
fn file_handler() -> Vec<u8> {
	let file = File::open("hello.txt").await.unwrap();
	let mut contents = Vec::new();
	file.read_to_end(&mut contents).await.unwrap();
	contents
}
```
### 2 格式化包装器
#### 2.1 Json
`axum::Json<T>` 是同时实现了 `FromRequest`（提取请求体）和 `IntoResponse`（返回 JSON 响应），可以用于包装结构体为响应体。只需要对应结构体派生`serde::Serialize`。
Axum 会自动设置 `Content-Type: application/json`。序列化失败返回500。
```rust
use axum::{
	http::StatusCode,
	Json, Router,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
#[derive(Serialize)]
struct User {
	id: u64,
	username: String,
}
#[derive(Deserialize)]
struct CreateUser {
	username: String,
}
async fn create_user(Json(payload): Json<CreateUser>,) -> impl IntoResponse {
	let user = User {
		id: 1337,
		username: payload.username,
	};
	Json(user)
}
```
#### 2.2 Html
`Html`包装器可以为`String`、`&str`、`File`等多种类型进行包装。返回Html文件。
#### 2.3 元组
`(http::StatusCode, impl IntoResponse)`类型是非常常用的类型，用于快速返回指定状态码和响应体。
#### 2.4 Result
`Result<T, E>`只要泛型T、E均实现`IntoResponse`，`Result`也实现`IntoResponse`。
==**`Result<impl IntoResponse, (Status::Code, impl IntoResponse)>`是非常常用的handler返回类型**==，搭配`map_err`错误转换方法与`?`运算符可以方便地处理handler中的各种Result。
### 3 header、Http状态码与组合式响应
`(Status::Code, HeadersMap, impl IntoResponse)`是常用的组合式响应体。可以指定状态码、头部和字符串内容。`axum::http::header` 模块提供了常用的标准头部常量。
```rust
use axum::{
    http::{HeaderMap, header},
    response::IntoResponse,
};
async fn handler() -> impl IntoResponse {
    let mut headers = HeaderMap::new();
    // 标准响应头
    headers.insert(header::CONTENT_TYPE, "text/plain".parse().unwrap());
    // 自定义响应头
    headers.insert("X-Custom-Header", "Hello".parse().unwrap());
    (headers, "Hello, world!")
}
```
### 4 自定义响应: `Response`和`Body`
这两种类型均来自于`hyper` crate。是所有响应和响应体的最终转换类型。实现了`IntoResponse`的类型会被axum自动封装为`Body`，最后封装为`Response`发送给客户端。
`axum::body::Body`既是请求体的基础数据类型，同时也是响应体的最终数据类型。
```rust
// 一个自定义body的文件下载示例
use axum::{
	body::{Body},
	response::{IntoResponse, Response},
	http::{StatusCode, header},
};

use tokio::{
	fs::{self, File},
	io::{AsyncReadExt, AsyncWriteExt},
};
async fn file_handler() -> Result<impl IntoResponse, (StatusCode, String)> {
	let mut file = File::open("hello.txt").await
		.map_err(|e| 
			(StatusCode::INTERNAL_SERVER_ERROR, e.to_string()
		))?;
	let mut contents = Vec::new();
	file.read_to_end(&mut contents).await.unwrap();
	// 从`Vec<u8>`创建Body
	let body = Body::from(contents);
	let mut response = Response::new(body);
	response.headers_mut().insert(
		header::CONTENT_TYPE,
		header::HeaderValue::from_static("application/octet-stream"),
	);
	response.headers_mut().insert(
		header::CONTENT_DISPOSITION,
		header::HeaderValue::from_str(&format!("attachment; filename=\"{}\"", "hello.txt"))
			.map_err(|_| (
				StatusCode::INTERNAL_SERVER_ERROR, "Invalid filename".to_string()
			))?,
	);
	Ok(response)
}
```

`Body`可以作为异步Stream流使用，使用`from_stream`方法从Stream构建`Body`，用于批量分块返回响应体等场景：
```rust
// 流式文件下载示例
use axum::{
    body::Body,
    http::{StatusCode, HeaderMap, header},
    response::IntoResponse,
};
use tokio_util::{
    io::{ReaderStream}
};
use tokio::{io::AsyncWriteExt, fs::File};

async fn download_stream_handler(path: String) -> Result<impl IntoResponse, (StatusCode, String)> {
	tracing::debug!("start download");
	let file = File::open("hello.txt").await
		.map_err(|_| (StatusCode::NOT_FOUND, "File not found".to_string()))?;
	// 将文件转为异步流
	let stream = ReaderStream::new(file);
	// 将异步流转为Body
	let body = Body::from_stream(stream);
	// 添加文件下载所需的Header
	let mut headers = HeaderMap::new();
	headers.insert(
		header::CONTENT_TYPE, 
		header::HeaderValue::from_static("application/octet-stream")
	);
	headers.insert(
		header::CONTENT_DISPOSITION,
		header::HeaderValue::from_str(&format!("attachment; filename=\"{}\"", "hello.txt"))
		.map_err(|_| {
			(StatusCode::INTERNAL_SERVER_ERROR, "Invalid filename".to_string())
	    })?,
    );
    Ok((StatusCode::OK, headers, body))
}
```