`Rc<T>`智能指针可以拥有多个所有者。编译器根据计数器strong_count来跟踪一个值的引用数量，每当一个引用离开作用域时减一，根据数量是否为0来判断是否清理指针。
RC智能指针通常用于一个程序中多个地方共同使用Heap上的一个数据。但是编译时无法判断哪个部分完成最后的调用。
多线程不能共享RC指针。
改造链表结构，使得一个节点可能链接多个节点（图数据结构）：
```rust
use std::rc::Rc
enum List {
  Nil,
  Cons(i32, Rc<List>)
}
use List:: {Nil, Cons};
let node1: Rc<List> = Rc::new(Cons(1, Rc::new(Nil)));
let node2: List = Cons(2, node1.clone());
let node3: List = Cons(3, node1.clone());
//node1.clone()相当于Rc::clone(&node)，参数为&Self类型
//node3、node2都有指向node1的指针
```
Rc的clone trait与普通的clone不同，clone方法会返回一个新的Rc指针指向同一块heap，==不会深拷贝数据，只增加引用计数==。
注意`Rc::new`仍然会转移所有权。所以不能对node1使用`Rc::new`。
只有当最后一个引用离开作用域时，才会对Rc调用drop方法。
==通常情况下Rc内部的数据是不可变的。除非Rc内部是RefCell==

## `Rc::downgrade`弱引用计数
`Rc<T>` 内部维护两个计数器：
1. **`strong_count`**：强引用数量（决定是否释放数据）
2. **`weak_count`**：弱引用数量（仅用于管理弱引用自身）
`Rc::downgrade`方法可以对一个Rc指针创建一个弱引用，增加weak_count。这种引用不会增加引用计数，当strong_count耗尽时数据内存被释放（即弱引用不阻止释放）。
这个弱引用不能直接访问数据，要通过upgrade方法来访问，这个方法返回一个`Option<Rc<T>>`。
```rust
use std::rc::{Rc, Weak};
let strong = Rc::new(42);
println!("Strong count: {}", Rc::strong_count(&strong)); // 1
let weak: Weak<i32> = Rc::downgrade(&strong);
println!("Strong count: {}", Rc::strong_count(&strong)); // 1（未变）
println!("Weak count: {}", Rc::weak_count(&strong));     // 1

if let Some(strong_again) = weak.upgrade() {
    println!("Value: {}", *strong_again); // 42
} else {
    println!("Value has been dropped!");
}
```
弱引用相当于一个只知道数据的地址，但是不能直接读取也不能阻止释放的指针。