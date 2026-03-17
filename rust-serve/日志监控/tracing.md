`tracing`和`tracing-subscriber`是构成日志系统的核心crate。其中`tracing`提供了事件追踪的核心trait和宏，决定了何时产生日志与日志的输出结构、内容等。`tracing-subscriber`负责收集日志信息，通常`tracing-subscriber`会将日志输出到控制台当中。
### 1 基础用法
#### 1.1 添加依赖：
```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```
#### 1.2  在main函数中初始化`tracing_subscriber`
```rust
// 初始化 tracing subscriber：启用控制台输出 + 从 RUST_LOG 环境变量过滤日志级别
async fn main() {
	tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();
}
```
还有一种初始化方式：
```rust
tracing_subscriber::registry()
	.with(tracing_subscriber::EnvFilter::from_default_env())
	.with(tracing_subscriber::fmt::layer())
	.init();
```
这两种方式实际上等价。`registry().init()`会创建一个订阅者容器，但不会主动监听events，而是通过`with`方法手动添加layer（处理events的插件，例如`fmt::layer`会创建一个格式化层，将日志转为文本格式）。适合于需要组合多个layer的场景。
#### 1.3 通过环境变量控制日志级别
```bash
RUST_LOG=info cargo run      # 只显示 info 及以上
RUST_LOG=my_web_app=debug cargo run  # 仅 my_web_app crate 的 debug 日志
```
在powershell中通过这些命令来设置环境变量：
```shell
$env:RUST_LOG="debug"; cargo run
```
#### 1.4 使用`tracing`宏记录日志
```rust
use tracing::{info, debug, warn};
async fn get_hello() -> &'static str {
	info!("Handler get_hello");
	debug!("This is a debug message");
	"Hello world!"
}
```
#### 1.5 日志级别与日志过滤
tracing提供了5个标准基本的日志，从低到高分别为：
`TRACING`、`DEBUG`、`INFO`、`WARN`、`ERROR`。每个级别都有对应的宏。例如：
```rust
use tracing::{error, warn, info, debug, trace};
error!("Database connection failed");
info!("User {} logged in", user_id);
```
日志输出的过滤方式则由`tracing-subscriber`提供，最常用的是基于环境变量进行过滤：
```rust
use tracing_subscriber::EnvFilter;
// 从环境变量 RUST_LOG 读取过滤规则
let filter = EnvFilter::from_default_env();
// 创建subscriber
tracing_subscriber::fmt().with_env_filter(filter).init();
```
也可以用一个字符串来指定过滤级别：
```rust
let filter = EnvFilter::new("info");
let filter = EnvFilter::new("my_crate=debug");
let filter = EnvFilter::new("warn,my_crate::db=trace");
```
#### 1.6 结构化输出
`tracing`的`info!`、`debug!`等宏均提供了结构化输出方式
```rust
let user_id = 123;
let email = "238186@g.com";
info!(user_id, emial, "User logged in");
// 输出：INFO ... user_id=123 email="user@example.com" User logged in
```
### 2 Tracing核心概念
#### 2.1 Span
span是日志记录的区间，所有在一个span中的日志都会带上这个span的名称、自定义的字段等信息。可以使我们方便地追踪一个日志所发生的位置。
```rust
use tracing::{info, span, Level};
async fn hello() -> &'static str {
    // 手动创建一个等级为INFO名为 "greet_user" 的 Span
    let _guard = span!(Level::INFO, "greet_user", user_id = 42).entered();
    info!("Preparing greeting message");
    "Hello!"
}
// 作用域结束Span被自动drop
```
如果想要将某个函数的整个函数体都设置为 span 的范围，最简单的方法就是为函数标记上 `#[instrument]`
```rust
use tracing::{info, instrument}; 
use tracing_subscriber::{fmt, layer::SubscriberExt, util::SubscriberInitExt}; 
#[instrument] fn foo(ans: i32) { 
	info!("in foo"); 
} 
fn main() { 
	tracing_subscriber::registry().with(fmt::layer()).init(); 
	foo(42); 
}
```
#### 2.2 Event
Event就是日志发生的底层事件。
```rust
use tracing::{event, span, Level, info};
event!(Level::INFO, "something happened");
//等价于
info!("Something happened");
```
`debug`、`warn`等宏本质上就是`event`的语法糖。
#### 2.3 Collector
`tracing-subscriber`就是日志的收集器，当Span和Event发生后，会被实现了`Collect` trait的收集器所收集。
### 3 输出文件
输出文件需要使用`tracing-appender`。它提供了异步创建与写入日志的方式。
这是一个同时将日志输出到控制台和文件中的例子：
```rust
// 使用color_eyre定义输出文字的颜色
use color_eyre::{eyre::eyre, Result};
use tracing::{error, info, instrument};
use tracing_appender::{non_blocking, rolling};
use tracing_error::ErrorLayer; 
use tracing_subscriber::{ 
	filter::EnvFilter, 
	fmt, 
	layer::SubscriberExt, 
	util::SubscriberInitExt, 
	Registry, 
};
fn main() -> Result<()> {
	// 过滤的layer
	let env_filter = EnvFilter::from_default_env().unwrap_or_else(|| 
		EnvFilter::new("info")
	);
	// 输出到控制台的layer
	let formatting_layer = fmt::layer().pretty().with_writer(std::io::stderr);
	// 追加输出到文件中，并设置滚动机制，never表示从不滚动
	let file_appender = rolling::never("logs", "app.log");
	// 包装为非阻塞模式
	let (non_blocking_appender, _guard) = non_blocking(file_appender);
	
	let file_layer = fmt::layer() 
		//关闭anis颜色码，颜色只用来在终端控制台中显示
		.with_ansi(false)
		//指定输出目标
		.with_writer(non_blocking_appender);
	Registry::default()
		.with(env_filter) 
		// ErrorLayer 可以让 color-eyre 获取到 span 的信息 
		.with(ErrorLayer::default()) 
		.with(formatting_layer) 
		.with(file_layer) 
		.init();
	// 安裝 color-eyre 的 panic 处理句柄 
	color_eyre::install()?;
	Ok(())
}
#[instrument] 
fn call_return_err() { 
	info!("going to log error"); 
	error!(?Err(eyre!("Something went wrong")), "error"); 
}
```
`tracing-appender` crate 提供了日志文件的自动滚动机制，主要基于时间策略（按日、小时、分钟）。例如2026-02-26日滚动后，前一天的文件就会变成app.log.2026-02-25。
`non_blocking`会启动一个专用的日志线程并创建一个内存通道。每当日志事件发生时，日志数据就被序列号并通过通道发送到线程，避免阻塞其他线程。