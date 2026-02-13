### 1 标准库中的TcpListener与网络流
可以使用`std::net::TcpListener`对本地某个端口进行监听，创建一个标准Tcp服务器。
loop循环对每个新建立的网络流进行处理，每个连接是一个由TcpStream请求流和SocketAddr地址组成的元组，可以使用BufReader读取请求流的内容。并使用`TcpStream::write_all`返回字节流信息。
```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    loop {
        match listener.accept() {
            Ok((mut stream, address)) => {
	            let buf_reader = BufReader::new(&mut stream);
	            // BufReader.lines返回一个迭代器，此处获取请求第一行的文本
	            let request_line = buf_reader.lines().next().unwrap().unwrap();
	            if request_line == "GET / HTTP/1.1" {
		            let status_line = "HTTP/1.1 200 OK";
			        let contents = fs::read_to_string("hello.html").unwrap();
			        let length = contents.len();
			        stream.write_all(format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}").as_bytes()).unwrap();
	            }
            },
            Err(err) => {
                eprintln!("Fatal accept error: {}", err);
                break;
            }
        }
    }
}
```
### 2 tokio中的网络服务器
通常会将tokio的tcpstream分割为读取流和发送流分开操作。
```rust
use tokio::{
    io::{AsyncBufReadExt, AsyncWriteExt, BufReader},
    net::TcpListener,
    fs
};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").await.unwrap();
    loop {
        let (mut stream, _address) = listener.accept().await.unwrap();
        tokio::spawn(async move {
            let (stream_reader, mut stream_writter) = stream.split();
            let reader = BufReader::new(stream_reader);
            let request_line = reader.lines().next_line().await.unwrap().unwrap();
            if request_line == "GET / HTTP/1.1" {
                let status_line = "HTTP/1.1 200 OK";
                let contents = fs::read_to_string("hello.html").await.unwrap();
                let length = contents.len();
                stream_writter.write_all(format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}").as_bytes()).await.unwrap();
            }
        });
    }
}
```