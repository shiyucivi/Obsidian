mpsc是多生产者、单消费者的通信通道。通道有两种类型：
异步通道：发送操作是异步的，拥有无限容量的缓冲区；
同步通道：发生操作是同步的，缓冲区的大小是预先分配的固定大小，如果缓冲区溢出就会阻塞。
通道的send和reveive都会返回一个Result。如果Result为Err，通常代表通道的另一端被挂起或者终止，那么之后的操作将无法进行，通常使用unwrap来引发错误传播。
```rust
import std::{mpsc, thread};

fn main() {
	let (tx, rx) = mpsc::channel::<Msg>();
	let handle = thread::spawn(move || {
		while let Ok(msg) = rx.reveive() {
			match msg {
				Msg::Call(task) => task(),
				Msg::Quit => {
					println!("thread quit");
					break
				}
			}
		}
	});
	rx.send(Msg::Call(Box::new(|| {
		println!("hello from main thread")
	}))).unwrap();
	rx.send(Msg::Quit).unwrap();
	handle.join().unwrap();
}
type Task = Box<dyn FnOnce() + Send + 'static>;
enum Msg {
	Call(Task),
	Quit,
};
```