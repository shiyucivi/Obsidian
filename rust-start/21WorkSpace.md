workspace允许在项目中创建多个独立的子库package。即每个库拥有自己的名称和toml。但这些库共享同一个顶层的Cargo.lock配置。
配置步骤：
1 创建目录
```shell
mkdir my_project
cd my_project
```
2 创建并配置Cargo.html
```toml
[workspace]
members = [
    "my-utils",   # 子库
    "my-main",    # 主程序（可选）
]
```
3 创建子库，包含一个执行库和一个lib库
```shell
cargo new --lib my-utils
cargo new --bin my-main
```
4 在主程序所在库中添加依赖：
```toml
# my-project/my-main/Cargo.toml
[dependencies]
my-utils = { path = "../my-utils" }
```
5 在lib库的lib.rs声明要公开的函数等
```rust
// my-project/my-utils/src/lib.rs
pub fn hello() {
    println!("Hello from sub-library!");
}
```
6 在执行库的主程序中使用子库中公开的api
```rust
// my-project/my-main/src/main.rs
use my_utils::hello;
fn main() {
    hello();
}
```
7 编译运行执行库
```rust
cargo run -p my-main
```