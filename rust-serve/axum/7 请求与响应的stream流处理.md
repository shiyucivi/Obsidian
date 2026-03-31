### 1 处理FormData中的二进制文件
对于`FormData`请求体，首先使用`axum::extractor::Multipart`提取出`Multipart`类型的FormData请求体。然后使用`next_field`方法遍历`Field`并使用`field_name`方法判断字段是否是文件类型。
对于文件类型的`Field`。其本身就是一个实现了`Stream<Item=Result<Bytes, E>>`的Stream。有两种方式对其进行读取并存储
#### 1.1 使用`futures_util::StreamExt`的`try_next`方法进行流处理
```rust
use axum::{
	extract::{Multipart, Path, Request},
    http::StatusCode,
}
use futures_util::{Stream, TryStreamExt};
use tokio::{fs::File, io::{AsyncWriteExt, BufWriter}};

static UPLOAD_DIR: &str = "uploads";
async fn accept_form(Path(path): Path<String>, mut multipart: Multipart) -> Result<StatusCode, (StatusCode, String)> {
    while let Ok(Some(mut field)) = multipart.next_field().await {
	    // 判断当前字段是文件类型并提取文件名
        let file_name = if let Some(file_name) = field.file_name() {
            file_name.to_owned()
        } else {
            continue;
        };
        // 使用try_next方法进行遍历
        while let Some(chunk_result) = field.try_next().await.unwrap() {
			// chunk_result 是 Bytes
			// 手动写入文件或其他处理逻辑
			let path = std::path::Path::new(UPLOAD_DIR).join(&file_name);
			// 创建文件读取器
			let mut writer = BufWriter::new(File::create(path).await.unwrap());
			// 写入流
			writer.write_all(&chunk_result).await.map_err(|e| {
				(StatusCode::INTERNAL_SERVER_ERROR, "".to_string())
			})?
        }
    }
    Ok(StatusCode::Ok)
}
```
#### 1.2 将`Stream`转为`AsyncReader`进行处理
最方便的做法是将`Stream`转为`AsyncReader`读取器，再创建一个`AsyncWriter`写入器。使用`tokio::io::copy`进行流赋值。注意`copy`需要`AsyncReader`被`pin`住。
```rust
use futures_util::{Stream, TryStreamExt};
use std::{io, pin::pin};
use tokio::{fs::File, io};
use tokio_util::io::{StreamReader};
// 省略提取field的代码
// 从field创建一个异步读取器
let body_with_io_error = field.map_err(io::Error::other);
let mut body_reader = pin!(StreamReader::new(body_with_io_error));

// 创建一个异步写入流                                      
let path = std::path::Path::new(UPLOAD_DIR).join(path);
let mut writer = BufWriter::new(File::create(path).await?);

tokio::io::copy(&mut body_reader, &mut writer).await?;
```
### 2 处理原始请求中的二进制流
使用`Request`原始请求提取器，使用`into_body`方法提取`Request`中的`body`，再使用`into_data_stream()`转为`Stream<Item=Result<Bytes, E>>`转为`Stream`即可。
```rust
async fn save_request_body(Path(file_name): Path<String>, request: Request) {
    let mut stream = request.into_body().into_data_stream();
    // 参考第一节中的代码
}
```

