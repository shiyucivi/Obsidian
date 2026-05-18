### 1 fs
rust中fs库主要用于读取文件和写入文件。
#### 1.1 文件操作
```rust
let file = fs::File::create(path);  //创建文件并返回File结构体
let file = fs::File::open(path); //读取文件并返回File结构体
fs::read(path); //读取文件为Vec<u8>
fs::read_to_string(path); //读取文件为UFT-8格式的String
file.write(&String.as_bytes()); //追加写入字符串，注意参数为&[u8]
file.write_all(&String.as_bytes()); //覆盖写入字符串
```
#### 1.2 目录操作
```rust
fs::create_dir(path); //创建单层目录
fs::create_dir_all(path);  //创建多层目录
fs::read_dir(path);  //读取目录，返回一个ReadDir迭代器
fs::remove_dir(path); //删除目录
fs::remove_dir_all(path); //递归删除目录
```
### 2 文件io
io库中为File、TcpStream、内存缓冲区等数据类型封装了读取、写入、移动读写位置等的trait。
#### 2.1 核心trait
io提供了几个核心的trait用于对文件进行各种操作
1. Read。从源读取字节。
Read是对所有可读对象（文件、网络流、内存缓冲区）进行操作的抽象。
2. Write。向目标中写入字节。
3. Seek。在支持随机访问的源中移动读写位置。
4. BufRead。带缓冲区的读取，支持按行读取。

#### 2.2 文件读取
`BufReader` 是标准库中用于**带缓冲读取**的核心类型，它包装任意实现了 `Read`和`BufRead` trait 的类型（如 `File`、`TcpStream` 等），通过内部缓冲区减少系统调用次数，显著提升 I/O 性能。
`BufReader`从`Read`和`BufReader`中继承了一些常用的读取方法
##### 2.2.1 按行读取
可使用`read_line()`方法逐行读取（读取返回每行字节数），或使用`lines()`返回迭代器进行迭代处理
```rust
use std::io::{BufReader, Read, BufRead};

let file = std::fs::File::open("data.txt")?;
let mut reader = BufReader::new(file);
let mut line = String::new();
// 将逐行读取到line中
while reader.read_line(&mut line)? > 0 {
    process(&mut line);
    line.clear(); // 重用 String 避免重复分配
}
// lines方法会返回一个Lines迭代器，每次迭代返回一行的Result
for line in buf_reader.lines() {
    let line = line?;
    process(line);
}
```
##### 2.2.2 按块读取
如果不需要按行读取，可以手动指定一个缓冲区进行读取：
```rust
use std::io::{BufReader, Read, BufRead};

let file = std::fs::File::open("data.txt")?;
let mut reader = BufReader::new(file);

//创建一个1024字节大小的缓冲区
let mut buf = [0; 1024];
buf_reader.read(&mut buf)?; // 将前1024个字节读取到buf中
```
#### 2.3 文件写入
`BufWriter`是一个带缓冲的写入器，包装了实现`Write` trait的类型（`File`，`TcpStream`等）。
`BufWriter`在写入时需要将写入类型转化为字节类型。
##### 2.3.1 `write()`和`write_all()`追加写入
`write()`和`write_all()`都是写入数据的方法，区别是`write()`受缓冲区或磁盘性能等影响不保证全部写入内容，返回实际写入的字节数。而`write_all()`会尝试全部写入。
```rust
use std::io::{BufReader, BufWriter, Read, BufRead, Write};
use std::file::File;
let mut input_file = File::open("output.txt");
let mut output_file = File::create("output.txt");
let mut reader = BufReader::new(input_file);
let mut line = String::new();
let mut writer = BufWriter::new(output_file);
while reader.read_line(&mut line)?>0 {
	writer.write_all(&mut line.as_bytes())?;
	writer.write_all(b"/n")?;
	line.clear();
}
```
##### 2.3.2 格式化写入
`write!`和`writeln!`是用来格式化写入内容的宏，用法类似于`format!`宏，区别在于后者会在末尾自动追加换行符。
```rust
writeln!(writer, "Time: {}, Status: {}", now(), "OK")?;
```
#### 2.4 错误处理
所有的IO操作均返回`Result<T, std::io::Error>`类型。推荐使用?进行错误传播或thiserror自定义错误类型。
### 3 路径：Path和PathBuf
`std::path::Path`是一种不可变的路径引用，类似于`&str`，通常以引用形式出现。而`PathBuf`是可变的拥有所有权的路径。`PathBuf`进行解引用操作可以得到`&Path`。因此两者很多方法可以共享使用。
```rust
// 从字符串字面量创建 
let path = Path::new("/home/user/documents/file.txt");
// 拼接path
let base_path = Path::new("/home/user");
// join返回一个PathBuf
let new_path = base_path.join("document").join("file.txt");
// 检查路径是否存在，会调用操作系统的文件系统接口
if path.exists() { 
	println!("路径存在"); 
	// 检查是否是文件 
	if path.is_file() { 
		println!("它是一个文件"); 
	} 
	// 检查是否是目录 
	if path.is_dir() { 
		println!("它是一个目录"); 
	}
}
```
大多数情况下使用更灵活的`PathBuf`即可。
```rust
// 1. 创建：拥有所有权 
let mut file_path = PathBuf::from("/var/log/my_app"); 
// 2. 修改：动态拼接 (Path 做不到这点) 
file_path.push("2023"); 
file_path.push("error.log");
```
### 4 tokio中的文件异步操作
Tokio的设计哲学是“统一异步接口”。无论是文件还是 TCP 流，都使用相同的 `AsyncRead/Write` 接口。
核心trait：
- `AsyncRead` / `AsyncWrite`：异步读写的基础 trait，所有异步 I/O 类型（如 `TcpStream`, `File`）都实现了它们 。
- `AsyncSeek`：支持异步文件定位（如 `seek(SeekFrom::Start(100))`）。
- `AsyncBufRead`：支持按行读取（`read_line`）等缓冲操作。
#### 4.1 异步文件操作
常用函数（位于 `tokio::fs` 模块）
- `File::open(path)` / `File::create(path)`：异步打开或创建文件。
- `read_to_string`, `read_to_end`一次性读取整个文件内容（适用于小文件）
- `copy(src, dst)`：高效复制文件内容
```rust
use tokio::fs::File; 
use tokio::io::AsyncReadExt; 
#[tokio::main] 
async fn main() -> std::io::Result<()> { 
	let mut file = File::open("example.txt").await?; 
	let mut contents = String::new(); file.read_to_string(&mut contents).await?;
	println!("File contents: {}", contents); 
	Ok(()) 
}
```
或者使用分片读取操作：
```rust
const CHUNK_SIZE: usize = 8 * 1024; 
let mut buf = vec![0u8; CHUNK_SIZE]; 
loop { 
	let n = file.read(&mut buf).await?; 
	if n == 0 { break; } 
	// 分片处理逻辑
	process_chunk(&buf[..n]).await; 
}
```
#### 4.2 BufReader缓冲读取
tokio也提供了BufReader进行缓冲读取的方式。
```rust
let file = tokio::fs::File::open(path).await?;
let buf_reader = tokio::io::BufReader::new(file);
let lines = buf_reader.lines(); //返回一个lines迭代器。
while let Ok(line) = lines.next_line().await {
	println!("line: {:?}", line);
}
```
#### 4.3 网络Stream流操作
```rust
//必须要引入AsyncBufReadExt这个trait，这个traut中定义了BufReader的异步方法。
use tokio::{
	io::{AsyncBufReadExt, BufReader};
	net::TcpListener,	
}

let listener = TcpListener::bind("localhost:8080")
    .await.unwrap();
loop {
	//返回tcp套接字和地址信息
    let (mut socket, _address) = listener.accept().await.unwrap();
    //将tcp套接字切割为读取流和写入流
    let (stream_reader, mut stream_writter) = socket.split();
    //使用bufReader作为流读取工具
    let mut buffer_reader = BufReader::new(stream_reader);
    //line作为BufReader的保存载体
    let mut line = String::new();
	while let result = buffer_reader.read_line(&mut line).await {
		//处理result
		line.clear();
	}
}
```