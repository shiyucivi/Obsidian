### 1 Crate和package
通常使用cargo new创建的一个项目被称为一个package。crate则是package中的一个编译单元（**通常称为包**）。一个package中可以有若干个binary crate和一个library crate。
`src/main.rs`是默认的可执行编译单元（main binary crate），此外还可以在bin目录下创建多个子执行单元。可以通过`cargo run --bin <crate_name>`来运行子执行单元。
一个binary package通常的结构如下：
```
├── Cargo.toml 
├── Cargo.lock 
├── src 
	├── main.rs  -默认二进制包
	├── lib.rs  -唯一库包
	└── bin  -其余二进制包
		└── main1.rs 
		└── main2.rs 
├── tests  -集成测试文件
	└── some_integration_tests.rs 
├── benches  -基准性能测试
	└── simple_bench.rs 
└── examples  -示例
	└── simple_example.rs
```
### 2 workspace管理多个package
一个工作空间是由多个 `package` 组成的集合，它们共享同一个 `Cargo.lock` 文件、输出目录和一些设置。每个package拥有自己的名称（toml中的`[package]`内容）和Cargo.toml。但这些package共享同一个顶层的Cargo.lock配置，并可以在顶层Cargo.toml的workspace管理。
工作空间有两种模式：root package和虚拟清单。
#### 2.1 rootpackage模式
一个 `package` 的 `Cargo.toml` 包含了`[package]` 的同时又包含了 `[workspace]` 部分，则该 `package` 被称为工作空间的根 `package`。
配置步骤：
1 创建根package
```shell
cargo new root_package
```
2 创建子package，包含一个binary package和一个library package
```shell
cargo new --lib my_lib
cargo new --bin my_bin
```
3 创建并配置Cargo.html
```toml
# root_package/Cargo.toml
[workspace]
members = [
    "my_bin",   # library package
    "my_lib",    # binary package
]
```
4 在lib库的lib.rs声明要公开的函数等
```rust
//root_project/my_lib/src/lib.rs
pub fn hello() {
    println!("Hello from sub-library!");
}
```
5 在binary package中添加library package的依赖
```toml
#root_package/my_bin/Cargo.toml
[dependencies]
my_lib = { path = "../my_lib" }
```
6 在binary package中使用library package中公开的内容
```rust
//root_package/my_bin/src/main.rs
use my_lib::hello;
fn main() {
    hello();
}
```
7 添加第三方依赖
可以在顶层toml中添加第三方依赖，然后所有package都能共享相同版本的依赖：
```toml
#root_package/Cargo.toml 
[workspace.dependencies] 
serde = { version = "1.0", features = ["derive"] }

#root_package/my_bin/Cargo.toml
[dependencies]
serde = { workspace = true }
```
8 运行与产出
可以使用cargo run执行src/main.rs这个主入口文件，也可以执行`cargo run -p <member>`运行某个子应用。
在 root package 模式下，cargo build 会“全量构建”整个工作区，产出多个二进制文件到target目录下
#### 2.2 vitrual manifest虚拟清单
若一个 `Cargo.toml` 有 `[workspace]` 但是没有 `[package]` 部分，则它是虚拟清单类型的工作空间。根目录下不应该有src/main.rs。
在这种模式下，不能够执行cargo run执行因为没有主入口文件，需要执行`cargo run -p <member>`来执行某个可执行crate。
