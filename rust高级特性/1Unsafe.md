## 1 裸指针
unsafe模式下可以使用两种原始指针（裸指针），`*const T`不可变原始指针与`mut T`可变原始指针。这里的`*`是原始指针类型的一部分，不是解引用。
原始指针的特点
1 绕过借用检查器制度
2 不保证不为悬垂指针、野指针，可以赋值为空指针null
3 不自动清理，需要手动drop
```rust
let x = 42;
let p: *const i32 = &raw const x; 
let q: *mut i32 = &raw mut x;
```
更推荐使用`std::ptr::addr_of!` 和 `addr_of_mut!`来获取原始指针
```rust
use std::ptr;
let x = 42;
let p = ptr::addr_of!(x);          // *const i32
let q = ptr::addr_of_mut!(x);      // ❌！x 不是 mut
```
## 2 Unsafe函数
可以在一个函数中声明一个unsafe块。并且可以保证函数本身是safe的。
如果一个函数或方法声明了unsafe，那么只能在safe块里使用。
```rust
use std::slice;
//对数组进行切片，返回切片索引的函数
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
  let len = values.len();
  let ptr: *mut i32 = values.as_mut_ptr(); //as_mut_ptr可以获得可变原始指针
  assert!(mid <= len);
  unsafe {
    (
      slice::from_raw_parts_mut(ptr, mid),
      slice::from_raw_parts_mut(ptr.add(mid), len - mid),//对原始指针进行移位
    )
  }
}
```
这里的`from_raw_parts_mut`就是一个unsafe函数。
## 3 接入外部代码

## 4 静态变量
静态变量是具有'static声明周期，并且内存地址固定的变量。
同时静态变量是可变的。但只能在unsafe中修改。
```rust
static mut COUNT: u32 = 0;
unsafe {
  COUNT += 1;
}
println!("count is {}", *(&raw const COUNT)); //通过原始指针获取到COUNT的内存地址，再通过解引用得到真实数据。safe中不能直接对静态变量进行引用和解引用
```