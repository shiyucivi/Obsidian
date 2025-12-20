trait是rust为结构体实现多态的一种机制（一个接口拥有多种实现方式）。
孤儿原则：只要trait或struct之一属于当前crate，就可以实现trait。（不能为外部struct实现一个外部的trait）。
## 1 为结构体实现方法规范
trait可以提供方法的前面。来规范结构体某个方法的参数和返回值类型。在实现时必须实现所有在trait中定义的方法规范。
```rust
struct Point {
  x: f32,
  y: f32
}
struct LatLng {
  lat: f32,
  lng: f32
}
// 定义trait
pub trait Format {
  fn to_arr(&self) -> [i32; 2];
}
// 实现trait
impl Format for Point {
  fn to_arr(&self) -> [i32; 2] {
    [self.x, self.y]
  }
}
impl Format for LatLng {
  fn to_arr(&self) -> [i32; 2] {
    [self.lat, self.lng]
  }
}
```
## 2 为结构体实现默认方法实现
trait中定义的方法也可以调用trait中定义的其他方法，不管有没有实现。有默认实现的方法可以不在结构体中实现。
```rust
struct Point {
  x: f32, y: f32
}
struct LatLng {
  lat: f32, lng: f32
}
// 默认的to_arr实现
pub trait Format {
  fn to_arr(&self) -> [i32; 2] {
    [0, 0]
  };
}
// 覆盖了默认的to_array实现，覆盖方法的签名必须与默认的实现保持一致
impl Format for Point {
  fn to_arr(&self) -> [i32; 2] {
    [self.x, self.y]
  }
}
```

## 2 参数和泛型约束
泛型约束有本质上相同的两种形式：
1 规定参数必须实现某个trait
```rust
fn get_array_from_point(point: &impl Format) {
  point.to_arr()
}
```
2 泛型约束
```rust
fn get_array_from_point<T: Format>(point: &T) {
  point.to_arr()
}
// 多个泛型约束
fn print_array_from_point<T: Format + Dispaly >(point: &T) {
  println!("{{point}}:{{point.to_arr()}}")
}
// 使用where关键字约束多个泛型
fn some_func<T, U>(t: &T, u: &U) -> i32
where 
  T: Display + Clone,
  U: Clone + Debug {
  //具体实现
}
```
函数的返回类型同样可以用于返回值的类型约束。
### 2.1 Trait Bound
可以使用trait的泛型约束来对结构体的某些特定类型的实现来实现特定的方法。
```rust
pub trait Point<T> {
  x: T, y: T
}
// 只对实现了Display + PartialOrd trait的泛型约束的Point实现cmp_display方法
impl<T: Display + PartialOrd> Point <T> {
  fn cmp_display(&self) {
    //具体实现
  }
}
```
## 3 trait实现另一个trait
可以令一个trait来实现另一个trait中的方法，这样做的好处是可以在trait中调用属于其他trait的方法。需要使用泛型来实现：
```rust
//for后面只能跟类型和结构体，不能把trait的名称放在for后
impl<T: Display> ToString for T {
  //
}
//这样实现了Display trait的类型也自动实现了ToString Trait
```

## 4 带有泛型T的Trait
### 4.1 带有泛型的Trait
trait的泛型T为某类型实现 trait + 类型T参数，常用于针对不同类型的非self参数，实现同一个名称的方法，效果类似于方法重载。
```rust
struct Container;
pub trait MyTrait<T> {
  fn do_something(&self, input:T) -> T;
}
impl MyTrait<i32> for Container {
    fn do_something(&self, input: i32) -> i32 {
        input * 2
    }
}
impl MyTrait<String> for Container {
    fn do_something(&self, input: String) -> String {
        input.to_uppercase()
    }
}
let c1 = Container{};
let num = c1.do_something(2); //4
```
### 4.2 为带有泛型`<T>`的结构体实现带有泛型的trait`<T>`
为带有泛型T的结构体实现泛型T的trait。==只有泛型类型需要在impl后加上`<T>`==。如果是具体类型，impl后不要加泛型参数。
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