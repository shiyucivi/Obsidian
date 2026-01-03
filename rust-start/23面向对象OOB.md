面向对象的三大特性：封装、继承、多态。rust实现了其中的封装（结构体）与多态（trait）。
## 1 Trait Object
trait object是一个抽象类型，不是具体的对象，通常用于在运行时将同一批不同类型的值视为一个统一的抽象类型进行处理，只要这些类型都实现了某个trait（需要少量运行时开销）。这允许我们以非常灵活的方式处理数据的类型。
用法：通常通过引用类型或智能指针，加上dyn关键字，在指定trait名称的方式来显式声明trait对象。即`Box<dyn Trait>`或`&dyn Trait`。
注意trait对象是抽象类型，不能添加添加数据字段。
```rust
trait Draw {
  fn draw(&self);
}
struct Circle;
impl Circle for Draw {
  fn draw(&self) {
  //具体实现
  }
}
struct Rectangle;
impl Rectangle for Draw {
  fn draw(&self) {
  //具体实现
  }
}
// 将拥有相同Draw trait的结构体视为同一类型，必须显式声明否则编译不通过
let shapes: Vec<Box<dyn Draw>> = vec![
  Box::new(Circle),
  Box::new(Square),
];
for shape in shapes {
  shape.draw();
}
```
## 2 状态设计模式
利用trait object可以实现OOB设计模式中的状态模式。