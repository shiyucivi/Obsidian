`thiserror`是用于自定义错误类型和实现错误转换的库，通过宏可以快速为错误枚举实现`Error`、`Display`和`Debug` trait。
### 1 实现错误类型
使用`#[error("errormsg")]`可以为错误类型实现`Display`输出格式，并使用类似`format!`模板语法打印所携带的数据信息：
```rust
use thiserror::Error;
#[derive(Error, Debug)]
enum MyError {
	#[error("无效输入: {input}")]
	InvalidInput { input: String },
	#[error("文件未找到: {0}")]
	FileNotFoune(String)
}
```
### 2 包装错误类型
使用`#[from]`可以快速实现`From<OtherError>` trait。
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Parse error: {0}")]
    Parse(#[from] serde_json::Error),
}
```