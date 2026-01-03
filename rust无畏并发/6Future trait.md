Future trait的定义为：
```rust
pub trait Future {
  type Output;
  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>
}
enum Poll<T> {
  Ready(T),
  Pending
}
```
poll方法用于随时间推进，修改Future内部的状态。表示返回或准备中。cx则由异步运行时提供。self参数的类型注解为`Pin<&mut Self>`，即包装该类型引用的Pin。
可以看出poll中需要用到自身的引用，因此这个self的地址必须被固定。

