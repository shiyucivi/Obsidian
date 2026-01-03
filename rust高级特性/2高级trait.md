## 1 关联类型
关联类型会将一个类型占位符绑定到trait中。实现trait时必须提供这个占位符的类型。
```rust
trait Iterator {
  type Item;
  fn next(&mut self) -> Option<Self::Item>;
}
```
### 1.1 关联类型和trait泛型
关联类型trait与泛型trait的区别在于，关联类型trait对每个类型只能实现一次，而泛型trait可以使用不同的泛型参数对同一类型进行多次实现。
关联类型trait的定义更清晰、约定更严格，在trait内部使用时更方便，实际当中应该优先使用关联类型。
## 2 运算符重载
rust中可以通过实现`std::ops`中定义的trait来重载运算符：
```rust
use std::ops::Add;
struct Point {
  x: i32,
  y: i32
}
impl Add for Point {
  type Output = Point;
  fn add(self, other: Point) -> Point {
    Point {
      x: self.x + Other.x,
      y: self.y + Other.y
    }
  }
}
```
## 3 trait泛型默认类型
定义trait时，可以为trait的泛型参数指定一个默认的类型：
```rust
trait Add<Rhs=Slef> {
  type Output;
  fn add(self, rhs:Rhs) -> Self;
}
```
实现Add trait时，如果不指定Rhs，那么Rhs默认为Self类型也就是要实现Add trait的类型。也可以指定Rhs为其他类型，这样就实现了不同类型的加法运算：
```rust
struct Centimeter(u32);
struct Meter(u32);
impl Add<Centimeter> for Meter {
  type Output = Meter;
  fn add(self, other: Centimeter) {
    Meter(self.0 + (other.0/100))
  }
}
```
默认类型参数可以让我们方便地实现默认定义。并且允许其他自定义情况而不破坏现有定义。
## 4 完全限定语法
一个类型可以实现多个拥有相同名方法的trait，也可以自己再实现同名的方法。但是调用时默认调用的是自己实现的方法。如果想调用trait中的同名方法。需要显式声明：
```rust
trait Bird {
  fn run (self: &Self) -> Self {/*具体实现*/}
}
trait Wizard {
  fn run (self: &Self) -> Self {/*具体实现*/}
}
struct Human {}
impl Bird for Human {
  fn run(self) -> Human {/*具体实现*/}
}
impl Wizard for Human {
  fn run(self) -> Human {/*具体实现*/}
}
impl Human {
  fn run {/*具体实现*/}
}
let person = Human;
person.run(&person); //调用自己实现的run方法
Bird::run(&person); //调用Bird中的run方法
```
### 4.1 调用trait中的静态方法
如果一个trait中存在一个没有self参数的静态方法，不能通过`trait::fn_name`的方式来直接调用trait中的静态方法。而是要通过这种方式：
`<Type as Trait>::function(args)`
## 5 Super trait
如果想在一个trait中实现另一个trait（Super trait）中的方法。但是这要求类型必须同时实现两个trait。
```rust
use std::fmt::Display;
//Display是OutlineDisplay的Super trait
trait OutlinePrint: Dispaly {
  fn outline_print(&self) {
    let output = self.to_string();
    let len = output.len();
    println("{}", "*".repeat(len+4));
    println!("*{}*", " ".repeat(len+2));
    println!("* {output} *");
    println!("*{}*", " ".repeat(len+2));
    println("{}", "*".repeat(len+4));
  }
}
struct Point {
  x: i32,
  y: i32
}
//必须先为Point实现Display，再实现OutlineDisplay
impl Display for Point {
  fn fmt(&slef, f: &mut fmt::Formatter) -> fmt::Result {
    write!(f, "({},{})", self.x, self.y)
  }
}
impl Outline for Point {}
```
## 6 Newtype模式（包装器）
一般情况下，实现trait时，trait和类型至少要有一个是当前项目的crate。但可以使用newtype模式绕过这个限制。简单说就是将一个类型使用元组包裹，变成一个包装类型。这种类型被成为newtype。
例如我们想为一个`vec<String>`类型实现一个Display方法：
```rust
struct VecWrapper (Vec<String>); //
impl Display for VecWrapper {
  fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
    write!(f, "[{}]", self.0.join(", "))
  }
}
```
