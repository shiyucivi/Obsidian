## 1 RefCell
`RefCell<T>`的用法类似于`Box<T>`，区别在于Box会在编译过程中进行借用检查。而RefCell只有在运行时才会执行所有权规则，一旦违反所有权规则，就会强制退出。
但是`RefCell<T>`的内存结构与`Box<T>`完全不同，`RefCell<T>`的数据并不保存在栈上，而是完全是由`T`决定的，`RefCell<T>`相当于对T的一个特殊的封装。仍然遵循借用检查器制度，**允许多次不可变借用，或一次可变借用**。

与其他智能指针的最大区别在于RefCell支持运行时可变性（内部可变性），即使没有声明为可变类型但仍然可以修改内部数据。实际开发中，`Rc<RefCell<T>>` 是非常常见的组合，用于实现**单线程下的共享可变状态**。可以直接对`Rc<RefCell<T>>`进行borrow_mut得到一个对T的可变引用，也可以使用`&Rc<RefCell<T>>.clone().borrow_mut`得到一个T的可变引用。
```rust
use std::rc::Rc;
use std::cell::RefCell;
// 多个地方共享同一个数据，但需要能修改它
let counter = Rc::new(RefCell::new(0));
let c1 = Rc::clone(&counter);
let c2 = &counter.clone(); //等价于上一句
*c1.borrow_mut() = 2;
*c2.borrow_mut() += 1;
println!("{}", counter.borrow()); // 3
counter.borrow_mut() += 1; //4
*c1.borrow_mut() = 2; //❌ 运行时报错，违反借用检查器制度
```
## 2 循环引用与双向链表
通常借助RefCell与Rc的组合是实现双向链表。双向链表中需要两个链表节点中存在互相引用即循环引用。
这是一个双向链表结构：
```rust
```rust
//循环引用
use std::rc::Rc;
use std::cell::RefCell;
#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
    prev: Option<Rc<RefCell<Node>>>, // ❌ 强引用！
}
fn main() {
    let a = Rc::new(RefCell::new(Node { value: 1, next: None, prev: None }));
    let b = Rc::new(RefCell::new(Node { value: 2, next: None, prev: None }));

    // a.next = b
    a.borrow_mut().next = Some(b.clone());
    // b.prev = a
    b.borrow_mut().prev = Some(a.clone());

    println!("a strong count: {}", Rc::strong_count(&a)); // 2
    println!("b strong count: {}", Rc::strong_count(&b)); // 2
    println!("{:?}", a);// 栈溢出
    drop(a); //不会报错，但a和b互相引用谁都不能释放，出现内存泄漏
}
```
运行到打印a时会报栈溢出错误，因为`*a`的next为`&b`，`*b`的prev又为`&a`，这样打印时会无限循环打印a和b的嵌套。
同时在这个例子中，由于a和b互相引用，谁先释放都会违反借用制度导致panic。因此谁都不会被释放，出现内存泄漏。

利用Rc弱指针只知道数据地址但不能阻止释放的特性，可以打破循环引用：
```rust
//解除循环引用
use std::rc::{Rc, Weak};
use std::cell::RefCell;
#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>, //强引用
    prev: Option<Weak<RefCell<Node>>>, //弱引用
}
fn main() {
    // 创建第一个节点
    let node1 = Rc::new(RefCell::new(Node {
        value: 1,
        next: None,
        prev: None,
    }));
    // 创建第二个节点
    let node2 = Rc::new(RefCell::new(Node {
        value: 2,
        next: None,
        prev: Some(Rc::downgrade(&node1)), // node2.prev → node1的弱引用
    }));
    // 让 node1的next指向node2的强引用
    node1.borrow_mut().next = Some(node2.clone());

    println!("Before drop:");
    println!("node1 strong: {}", Rc::strong_count(&node1)); // 1（只有node1持有）
    println!("node2 strong: {}", Rc::strong_count(&node2)); // 2（node2与node1.next都持有）
    println!("{node1:?}");
    // 现在丢弃 main 中的引用
    drop(node1);
    drop(node2);
    // ✅ 所有节点都被正确释放！
}
```
这个例子中最后会形成这样的结构：
```
node1 = Rc<RefCell<Node1>>  //Node1为value=1的Node结构体
Node1.next = 强引用Node2 
node2: Rc<RefCell<Node2>>
Node2.prev = 弱引用Node1 
```
这个例子中打印node1会出现这样的结构：
```
RefCell { value: Node { value: 1, next: Some(RefCell { value: Node { value: 2, next: None, prev: Some((Weak)) } }), prev: None } }
```
可以看到虽然node1和node2还是存在循环引用，但是debug在遇到Weak时，由于Weak不能直接访问数据，所以会停止进一步对node2的prev的读取。
最后无论先释放node1还是先释放node2都不会报错：
假如先释放node1，node2的prev是node1的弱引用，不会阻止node1与Node1的释放，获取node2.prev会得到`Option::None`。
如果先释放node2，还存在Node1的next是node2的强引用，node2的引用计数没用归零，数据Node2还在heap上存活，有Node1的next仍然可以指向Node2。