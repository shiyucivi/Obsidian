## 1 HashMap的创建与存取
HashMap是一个键值对`<key, value>`的集合，存储在heap上。
使用insert插入键值对，使用get方法获取某个值。==HashMap的所有键值对的键的类型（泛型K）必须都相同，值的类型（泛型V）也必须相同。==
如果K、V其中一个是引用类型，那么需要标注生命周期。
```rust
use std::collections::HashMap;
//HashMap没有被默认导入，需要引入
let mut colors = HashMap::new();
//从数组中创建一个HashMap，数组元素必须是(K, V)类型
let map2 = HashMap::from([
    ("apple", 5),
    ("banana", 3),
    ("orange", 8),
]);
colors.insert(String::from("Red"), (255,0,0));
let red_color = colors.get("Red");
colors.insert(String::from("Blue"), String::from("#0000FF")); //报错，值已经确定为(i32, i32, i32)类型
```
==insert方法的第一个参数没有类型限制（泛型K）。==
==get方法的参数必须是键类型的引用类型（&K）==。返回一个Option。可以衔接copied和unwrap_or来获取到Option里的值：
```rust
let red = colors.get("Red").copied().unwrap_or((255,255,255));
//如果Option里的值是个引用类型，copied方法会对其解引用并copy一个全新的值出来，在返回一个Option。注意Option的值必须是引用类型而是Option<T>或者Option<&mut T>。
//unwrap_or用于在解包一个Option，如果为None则将第一个参数作为默认值返回。
```
### 1.1 HashMap的遍历
HashMap实现了iter trait。可以用for in循环来遍历
```rust
for let (key, value) in &colors {
	println!("{}:{}", key, value);
	//key为&String类型，value为&(i32, i32, i32)类型
}
```
key和value都是引用类型。分别为&K和&V。

## 2 HashMap的所有权
如果将一个变量作为key或者value通过insert存入HashMap。那么如果变量没有实现copy trait，就会发生所有权转移。
```rust
let green_tp = (0, 255, 0);
colors.insert(String::from("green"), green_tp);
//不能再使用green_tp
```
多次insert一个值相同的key，后插入的键值对会覆盖之前的。
entry方法可以检查map中是否存在一个key：
```rust
colors.entry(String::from("pink")).or_insert((255, 192, 203));
//如果不存在"pink"的key，插入一个pink: (255,192,203)的键值对。
```
entry返回一个Entry枚举。or_insert返回插入值或原有值的一个&mut V 引用。
这是一个统计一个字符串中每个单词出现次数的map：
```rust
let text = "hello world how beautiful world";
let mut map = HashMap::new();
//split_whitespace将字符串分解为了一个可迭代的数据
for word in text.split_whitespace() {
  //count为每个单词字符串对应的key的一个可变引用，即&mut i32
  let count = map.entry(word).or_insert(0);
  //count需要*解引用为i32才能使用加法
  *count += 1;
}
```
HashMap实现了index trait，可以使用索引形式的`map[key]`来获取值的引用。但如果没有获取到值，就会触发panic。通过`&Map[key]`这种索引引用会视为对map的一个引用，因此引用存在期间不能修改map。而`&mut Map[key]`进行索引可变引用非常危险，一般不要使用。
```rust
let mut h = HashMap::new();
h.insert("k1", 1);
let hv1 = &h["k1"]; //视为对h的引用
h.insert("k2", 2); //错误，hv1存在时不能修改h
println!("{hv1}")
```
