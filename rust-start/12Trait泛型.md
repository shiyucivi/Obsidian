为带泛型的结构体实现trait时，什么时候要用impl后面要加泛型`<T>`，什么时候不需要：
==**如果是为泛型为具体类型的结构体实现trait时，不需要impl后加泛型，如果是为整个结构体实现trait，则需要`impl<T>`**==。可以分以下几种类型讨论。
trait的泛型配合结构体的泛型，可以实现方法重载（trait与结构体一对一、一对多、多对多）。
## 1 trait不带有泛型，结构体有泛型
### 1.1 为结构体整体实现trait
需要在impl后面加上泛型参数`<T>`。这里的泛型参数T可以加上Trait bounds约束。
```rust
use std::fmt::Display;
struct Wrapper<T> {
  value: T,
}
//为满足Display trait泛型约束的Wrapper结构体实现fmt方法
impl <T:Display> Display for Wrapper<T> {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    write!(f, "Wrapper({:?})", self.value)
  }
}
```
如果T是不同的泛型约束呢，还能否实现？：不能
### 1.2 为具体泛型的结构体实现trait
```rust
// 只为 Wrapper<i32> 实现 Display的fmt方法
impl Display for Wrapper<i32> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Integer wrapper: {}", self.value)
    }
}
//注意，会与上面的fmt方法冲突
```
注意：如果已经为结构体整体实现了Display trait的fmt方法，那么就不能为特定类型约束的结构体实现同名方法。反之亦然。
## 2 trait带有泛型
trait带有泛型分为三种情况，1、trait本身带有泛型参数`<T>`；2、trait中方法带有泛型参数；3、trait带有关联类型。
### 2.1 trait本身带有泛型参数`<T>`
From trait就是一个带有泛型参数的trait。From trait在对结构体进行实现时trait的泛型参数必须为具体类型或trait bounds约束。
```rust
trait From<T> {
    fn from(value: T) -> Self;
}
impl <T> From<i32> for Wrapper<T> {
    fn from(value) -> Wrapper<T> {
      //具体实现
    }
}
```
也存在不需要具体类型泛型参数的trait：
```rust
pub trait MyTrait<T> {
  fn do_something(&self) -> T;
}
struct Container<T> {
  value: T
}
impl<T> MyTrait<T> for Container<T>
where T: Clone,
{
  fn do_something(&self) -> T {
    self.value.clone()
  }
}
```
这种带有泛型的结构体，通常为了给一种结构体实现多种类型的trait实现（一对多）
### 2.2 trait中的方法拥有泛型参数
这种情况下结构体和方法各自维护自己的泛型参数即可。
```rust
trait MyTrait {
    fn process<T>(&self, input: T);
}
struct MyStruct<T> {
	  value: T
}
impl MyTrait for MyStruct<i32> {
    fn process<U>(&self, input: U) {  // 方法内的泛型 U，与 impl 无关
        // ...
    }
}
```
### 2.3 带有关联类型的trait
关联类型不需要指定泛型参数，只需要在实现trait时指定具体关联类型。
例如Iterator就是一个带有关联类型的trait：
```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
struct Counter {
    count: u32,
}
impl Iterator for Counter {
    type Item = u32;  // 指定关联类型，关联类型用::表示
    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        Some(self.count)
    }
}
```

## 4 什么情况下trait本身增加泛型参数
核心原则：**“一个类型是否需要对同一个 trait 提供多个实现？”**
 使用 **泛型 trait**（`trait Trait<T>`）当：

> **同一个结构体需要针对不同的 `T` 实现多次该 trait。**