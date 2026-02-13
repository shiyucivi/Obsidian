### 1 Future
一个简单的Future定义：
```rust
trait SimpleFuture { 
	type Output; 
	fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>
} 
	
enum Poll<T> { 
	Ready(T),
	Pending, 
}
```
其中，cx则由异步运行时提供。
可以看出一个Future包含两个状态枚举Ready和Pending，由执行器通过轮询的方式不断调用poll，如果未完成返回Pending，如果完成则返回Ready。通常异步运行时中为了提高效率，执行器不会反复轮询，而是等待Future通知运行时完成，才修改Future的状态。


### 2 `Pin`指针

Box::pin是一种特殊的Box智能指针，类型是`Pin<Box<T>>`。而Pin本身是一个指针的包装器。保证了被包裹的指针指向的地址不会被move。常用于存在自身引用的结构体中。
```rust
struct Test { 
	a: String, 
	b: *const String, 
}
impl Test { 
	fn new(txt: &str) -> Self { 
		Test { 
			a: String::from(txt), 
			b: std::ptr::null(), 
		} 
	}
}
```


为什么要pin：
因为async块本质上是一个状态机的结构体，状态机中包含了记录await点的状态的字段。其内部存在自引用（状态字段指向内部其他字段），一旦结构体被移动，那么其内部的自引用就会变成悬垂指针。