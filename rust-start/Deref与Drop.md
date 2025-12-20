Box的内存结构：
```rust
let s1 = String::from("hello");
let b1 = Box::new(s1); //s1的所有权被转移给了b1
```
s1是个胖指针：
```
Stack (s1):
| ptr  → ──────────┼───→ Heap: [h][e][l][l][o][...unused...]
| len  = 5         |
| cap  ≥ 5         |
```
b1是个有两层堆的智能指针，在栈上只占据8字节，在堆上的第一层为一个String结构体，结构体中的ptr指向String在heap上的真实数据：
```
Stack (b1):
+----------+
| b1 →     | 
+----------+
     │
     ▼
Heap (String struct, 24 bytes):
+------------------+
| ptr  → ──────────┼───→ Heap: [h][e][l][l][o]  ← 真实字符串数据
| len  = 5         |
| cap  ≥ 5         |
```

# Deref Trait
## 1 `*`语法糖的本质
`*`可以取到一个Box类型上拥有的数据，
Box实际上类似于存储在heap上的只有一个元素的元组。
Box也可以通过`*`操作符实现解引用，实际上是Deref trait的`derefe`方法的语法糖。
```rust
struct MyBox<T>(T)
impl <T> MyBox<T> {
  fn new(x: T) -> MyBox<T> {
    MyBox(x)
  }
}
impl <T> Deref for MyBox<T> {
  type Target = T;
  derefer(&self) -> &Self::Target {
    &self.0
  }
}

let y = MyBox::new(5);
assert!(*y, 5); //true
//这里的*y相当于*(y.deref())
let x = *y + 1; //x = 6
let z = y; //y的所有权被转移给了z，z所拥有的数据仍然是5
```
注意：在 Rust 中，对任意类型 `T` 执行 `*y`（其中 `y: T`），如果 `T` 实现了 `Deref` trait，那么 `*y` **等价于 `*(y.deref())`**，而不是 `y.deref()` 本身。
当对一个实现了 `Deref` 的类型使用 `*` 操作符时，编译器会自动插入 `.deref()` 调用，并对结果再解引用。
对普通引用`&x`进行解引用和对Box进行`*`解引用的区别在于，普通引用解引用得到的是原始的被引用变量的位置，这个变量可能是个栈上的指向堆上的指针，也可能是堆上的数据，这种解引用不能转移所有权。而Box解引用得到的必然是堆上的数据，如果堆上的数据是没实现Copy的，那么解引用可能会转移所有权消耗掉Box。
```rust
let mut a = String::from("hello");
let c = &a;
let e = *c; //❌编译报错，普通引用解引用不能转移所有权

let b = Box::new(String::from("hello"));
let s = *b; //解引用得到的是堆上的数据，String的所有权被转移给了s
```
## 2 隐式解引用转换Deref coercion
当你把一个实现了 `Deref` trait 的类型（如 `&String`、`Box<T>`、`Rc<T>` 等）传递给一个期望其 **目标引用类型（`&Target`）** 的函数或方法时，编译器会**自动插入 `*` 解引用操作**，将 `&Wrapper<T>` 转换为 `&T`。
例如：
```rust
fn func1 (s: &str) -> String {
	String::from(s)
}
let s1 = String::from("hello");
let s2 = &s1; //s2是&String类型
func1(s2); //将s2从&String类型隐式转换为了&str类型
func1(&&s2); //会多次使用* 直到类型匹配
func1(Box::new(s1)); //❌，Box<String>不是String的引用类型
func1(&Box::new(s1)); //可以通过编译，因为可以通过两次解引用得到&str
func1(&*Box::new(s1)); //可以通过编译
```
存在与Deref trait类似的针对于可变引用的trait：DerefMut。
存在以下三种隐式解引用转换的场景：
```
&T -> &U (T: Deref<Target=U>)
&mut T -> &mut U(T: Deref<Target=U>)
&mut T -> &U(T: Deref<Target=U>)
```

# Drop Trait
drop规定了一个指针离开作用域时的动作。只有一个drop方法。
```rust
struct CustomSmartPointer {
  data: String
}
impl Drop for CustomSmartPointer {
  fn drop(&mut self) {
    println!("Droping data: {}", self.data);
  }
}
{
  let c1 = CustomSmartPointer {
    data: String::from("hello")
  };
  let c2 = CustomSmartPointer {
    data: String::from("world")
  }
}
//先打印c2，后打印c1。服从栈内存后入先出原则。
```
不允许手动调用指针的drop方法来提前释放指针，rust会根据词法作用域在编译过程中自动添加drop调用。可以通过`std::mem::drop`函数来手动释放指针，这个函数会转移指针的所有权。
```rust
let b1 = Box::new(32);
std::mem::drop(b1);
//b1不再有所有权
```