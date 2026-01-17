h实现了Iterator trait的结构体就是迭代器。
```rust
pub trait Iterator {
  type Item;
  pub fn next(&mut &self) -> Option<Self::Item>;
}
```
trait中的type用于指定关联类型。实现trait必须指定Item的类型。
next方法：每次返回Iteratior中的一个Item，包裹在Some中。迭代结束时返回None。
```rust
let v1 = vec![1,2,3];
let mut v1_iter = v1.iter();
assert_eq!(v1_iter.next(), Some(&1)); //初次调用next，返回第一个元素的引用
```
要使用next方法，Iterator必须是mut可变的。因为next方法会修改Iterator中的内容。注意，v1_iter虽然是mut，但是他获得的元素的引用是不可变的。
要在迭代器中获取元素所有权或者获得可变的引用，要使用into_iter()和iter_mut()方法。

对一个元素实现Iterator trait时必须实现next方法。因为Iterator trait中有一些默认实现的方法中调用了next方法。这些调用next方法的方法被称为消耗性适配器。即调用这些消耗性方法后不能再使用迭代器。
```rust
let v1 = vec![1,2,3];
let v1_iter = v1.iter();
let sum = v1_iter.sum(); //sum方法中会不断调用next来进行求和操作
```

## 迭代器适配器
迭代器适配器是定义在Iterator trait上的方法。这种方法接受一个闭包作为参数，返回一个新的迭代器，并且会消耗掉原来的迭代器。
```rust
let v1 = vec![1,2,3];
let v1_iter = v1.iter();
let v1_map_iter = v1_iter.map(|x| x + 1); //v1_iter被消耗掉，不能再使用
let v2: Vec<_> = v1_map_iter.collect(); //vec![2,3,4]
```
注意map是惰性执行的，map后面必须执行collect、sum等方法才会执行，否则不会执行。如果想遍历迭代器并且直接执行，可以使用for_each