## 1 String是什么
String是标准库中队`vec<u8>`的一种实现，本质上是utf8码（8位字节码）的集合，而不是`vec<char>`。因此不能用索引来取到其中的char（因为一个char可能占据多个字节）。
```rust
let mut empty_str = String::new(); //创建一个空字符串
let sl = "hello"; //&str
let s2 = String::from("hello");
empty_str = sl.to_string();
```
实现了Display trait的数据都有to_string方法，to_string方法会创建一个新的String，不涉及所有权。

字符串的最终渲染形式是字形簇的叠加，一个字形簇由一个或多个字符char组成（例如一些小语种的复杂文字）。一个字符则由一个或多个8位字节码组成。
## 2 String的方法
### 2.1 追加字符串
push_str可以为字符串追加字符传，push_str的参数只能是&str。
push可以为字符串追加字符，参数只能是char。
```rust
let mut s1 = String::new();
s1.push_str("lo");
s1.push('l');
```
### 2.2 拼接字符串
字符串支持使用+进行拼接，但必须是String+&str(+&str)的格式。同时+会导致所有权转移。
+操作符本质上是add trait。
```rust
let s1 = String::from("lo");
let s2: &str = "l";
let s3 = s1 + s2; //s1所有权转移到s3
```
多个字符串拼接：
```rust
let s1 = String::from("league");
let s2 = s1 + " " + "of" + " " + "legends"
//或者
let s1 = String::from("league");
let s2 = "legends"
let s3 = format!("{s1} of {s2}");
```
## 2.3 切片
使用`[Range]`切片时Range的范围必须位于有效字节的范围内。
### 2.4 字符串遍历
String.chars()方法可以对字符串中每个字符进行遍历，返回一个Chars类型的可迭代结构体。
bytes方法遍历其中每个字节。
```rust
let s1 = String::from("hello");
let chars1 = s1.chars();
for chara in chars1 {
  println!("{chara}");
}
```