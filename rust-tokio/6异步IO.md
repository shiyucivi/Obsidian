### 1 AsyncRead和AsyncWrite
`AsyncReadExt`和`AsyncWriteExt`两个trait提供了文件读写方法。
`AsyncRead`和`AsyncWrite`两个trait是最小化的底层核心trait，提供了异步读写的操作原语。
实现了`AsyncRead`的类型是读取器，实现了`AsyncWrite`的类型是写入器。
`AsyncReadExt`和`AsyncWriteExt`是对这两个trait的扩展，为读取器和写入器提供了多种读写方法。通常只使用扩展trait即可。
#### 1.1 read
`AsyncReadExt::read`用于将读取器（`reader`，`File`和`Stream`等都是读取器）读取到一个缓冲区中，这个缓冲区可以是一个`u8`数组也可以是固定大小的vec。
```rust
use tokio::File;
use tokio::io::AsyncReadExt;

let mut f = File::Open("foo.txt").await.unwrap(); //一个读取器
let mut buffer: [u8; 10] = [0; 10];
// 读取前10个字节的数据，返回读取到的字节数即n=10
let n = f.read(&mut buffer[..]).await.unwrap();
println!("bytes: {}", &buffer[..n]);
```
read方法类似于迭代器的next方法，可以重复调用以读取后面的内容：
如果read方法返回数字小于buf的长度，说明读取器内的数据已全部读取。
如果read方法返回0，说明读取已达到数据末尾（`EOF`），数据应结束读取。
读取器内部维护了一个读取位置（`cursor`），每次成功 `read` 后，这个位置会自动向前推进，直到到达`EOF`。通常使用loop循环读取一个读取器。
如果要读取文件中的所有字节，可以使用`AsyncReadExt::read_to_end`:
```rust
use tokio::io::{self, AsyncReadExt}; 
use tokio::fs::File;

let mut f = File::Open("foo.txt").await.unwrap();
let mut bytes: Vec<u8> = Vec::new();
// 读取文件中所有数据
f.read_to_end(&mut buffer).await.unwrap();
```
#### 1.2 write
`AsyncWrite::write`可以将缓冲区内的内容写入到写入器（`writer`）中，同时返回写入的字节数：
```rust
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

let mut file = File::create("foo.txt").await.unwrap(); //写入器
//返回写入的字节数
let n = file.write(b"some bytes").await.unwrap();
```
write方法的参数是一个数组切片`&[u8]`，在`&str`前面加上b可以将`&str`转换为一个字节数组引用`&[u8;length]`。write方法再隐式转换为数组切片`&[u8]`。
`AsyncWriteExt::write_all`用于将缓冲区内的内容全部写入数据：
```rust
use tokio::io::{self, AsyncWriteExt}; 
use tokio::fs::File; 

let mut file = File::create("foo.txt").await;.unwrap() 
file.write_all(b"some bytes").await?.unwrap();
```
### 2 服务器回声
将客户端发送给服务器的内容原封不动地返回去，有两种方式：
#### 2.1 使用`io::copy`
copy方法会将读取器`reader`中的内容拷贝到写入器`writer`当中。
`TcpStream`类型是一个实现了`Reader + Writter`的读写器，这种类型可以使用`io::split`分离为读取器和写入器。
```rust
use tokio::io;
use tokio::net::TcpListener;

#[tokio::main]
fn main() -> io::Result<()> {
	let listener = TcpListener::bind("127.0.0.1:5673").await.unwrap();
	loop {
		let (mut socket, address) = listener.accept().await.unwrap();
		tokio::spawn(async move {
			let (mut read_socket, mut write_socket) = socket.split();
			io::copy(&mut read_socket, &mut write_socket).await.unwrap();
		})
	}
	Ok(())
}
```
#### 2.2 自定义实现
将接收到的信息读取到一个buf中，然后在写入器中写入。
```rust
let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
loop {
	let (mut socket, _) = listener.accept().await.unwrap();
	task::spawn(async move {
		let (mut reader, mut writer) = socket.split();
		let mut buf = vec![0; 1024];
		loop {
			match reader.read(&mut buf).await {
				Ok(0) => return, //读取到达末尾
				Ok(n) => {
					writer.write_all(&buf[..n]).await.unwrap();
				}
				Err(e) => panic!("connection error = {:?}", e),
			}
		}
	});
}
```
