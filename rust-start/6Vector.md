## 1 创建vector
vector代表一种在heap内存中占据连续空间的同类型元素的不定长度的集合.
创建一个空的vector必须指定元素类型
```rust
let v:Vec<i32> = Vec::new(); //创建空的vector
let mut v = vec![1, 2, 3]; // v为Vec<i32>类型
v.push(4);
v.push(5);
```
## 2 获取vector的元素
```rust
let v = vec![1, 2, 3];
let second_el = &v[1];
let third_el = v.get(2); //get返回的是一个Option枚举，其Some为Some<&i32>
```
如果通过`[index]`根据索引获取元素时，index超出vec最大索引例如`&v[100]`，那么会引发一个panic导致程序崩溃。get则不会。
## 3 vector的引用与所有权
对vec中一个元素的引用视为对vec整体的引用。当一个vec存在引用时，不能修改这个vec，因为vec是连续存储的，修改vec会导致vec占据的内存大小发生变化，导致内存地址的转移。
```rust
let mut v = vec![1, 2, 3];
let third = &v[2];
v.push(4); //报错
println!("{third}");
```
通过for in循环对vec的引用进行遍历时，遍历时的每个vec的元素也都是引用类型，其值为heap中的地址而不是真实的数据。如果想对这个元素进行类似于加减之类的操作需要解引用：
```rust
let mut vec1 = vec![1, 2, 3];
for i in &mut vec1 {
  *i += 50;
}
```
通过push方法向vec中推入元素时，如果元素是个非引用类型，那么元素的所有权被转移给vec。
## 4 迭代器
所有可迭代的数据类型（vec、Array）都有iter()方法，可以返回一个迭代器。
迭代器位于stack中，被创建时是一个指向第一个元素的heap的一个指针，迭代器的next方法会让这个指针移动，指向下一个元素的heap。
```rust
let mut v = vec![1, 2, 3];
let mut iter = v.iter();
// iter为Some<&i32>
let n1 = iter.next().unwrap(); //next返回的Some是前一个元素而不是移动后的元素，因此为&v[0]而不是&v[2]
```
next方法返回的是一个Option，其Some为前一个元素的引用。
如果调用next时iter已经指向最后一个元素，next后iter指针将不指向任何heap。如果再调用next，那么会发生未定义行为。
可以使用for in循环对一个迭代器进行遍历。在遍历时，同样不能修改迭代器指向的vec的长度，只能修改vec里的元素。
```rust
let mut vec3 = vec![1, 2, 3];
let mut_iter = &mut vec3.iter_mut();
for i in mut_iter {
  *i += 10;
  vec3.push(10); //报错
}
```
rust在的Range范围表达式也具有iter的trait，指向heap中的一些连续的数字。