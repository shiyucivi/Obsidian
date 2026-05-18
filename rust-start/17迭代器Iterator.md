实现了Iterator trait的结构体就是迭代器。
```rust
pub trait Iterator {
  type Item;
  pub fn next(&mut &self) -> Option<Self::Item>;
}
```
trait中的type用于指定关联类型。实现trait必须指定`Item`的类型。
`next`方法：每次返回Iterator中的一个`Item`，包裹在`Some`中。迭代结束时返回`None`。
```rust
let v1 = vec![1,2,3];
let mut v1_iter = v1.iter();
assert_eq!(v1_iter.next(), Some(&1)); //初次调用next，返回第一个元素的引用
```
要使用`next`方法，Iterator必须是`mut`可变的。因为`next`方法会修改Iterator中的内容。注意，`v1_iter`虽然是mut，但是他获得的元素的引用是不可变的。
要在迭代器中获取元素所有权或者获得可变的引用，要使用`into_iter()`和`iter_mut()`方法。

对一个元素实现Iterator trait时必须实现next方法。因为Iterator trait中有一些默认实现的方法中调用了`next`方法。这些调用`next`方法的方法被称为消耗性适配器。即调用这些消耗性方法后不能再使用迭代器。
```rust
let v1 = vec![1,2,3];
let v1_iter = v1.iter();
let sum = v1_iter.sum(); //sum方法中会不断调用next来进行求和操作
```

### 迭代器适配器
迭代器适配器是定义在Iterator trait上的方法。这种方法接受一个闭包作为参数，返回一个新的迭代器，并且会消耗掉原来的迭代器。
```rust
let v1 = vec![1,2,3];
let v1_iter = v1.iter();
let v1_map_iter = v1_iter.map(|x| x + 1); //v1_iter被消耗掉，不能再使用
let v2: Vec<_> = v1_map_iter.collect(); //vec![2,3,4]
```
注意`map()`是惰性执行的，`map`后面必须执行`collect`、`sum`等方法才会执行，否则不会执行。如果想遍历迭代器并且直接执行，可以使用`for_each`方法