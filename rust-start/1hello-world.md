#### 1 helloworld
rust的文件命名规范为hello_word.rs
```rust
fn main() {
	println!("hello, world!");
}
```
main函数是一个rust程序最先执行的函数。像println!这种带有感叹号的是宏（macro）而不是函数。==**每行语句以; 结尾**。==

编译一个rust程序：
```bash
rustc hello_world.rs
```
会生成一个可执行二进制文件hello_world.exe和调试文件hello_world.pbd文件。

像rust这种编译语言存在编译时(compile-time)和运行时(run-time)两个阶段。编译时会将rust源文件编译为可执行的二进制文件（exe），同时检查源代码中的语法、类型、内存安全等，将宏替换为实际代码等。rust几乎不需要运行时。
#### 2 使用Cargo创建一个rust项目
Cargo 是 Rust 的构建系统和包管理器。
```bash
cargo new hello_cargo 
cd hello_cargo
cargo build
cargo run
cargo build --release
```
`cargo new`创建的项目中会有一个`Cargo.toml`的配置文件。
`cargo build`会在`target/debug`下编译生成一个可执行文件，还有一个`cargo.lock`文件用于记录依赖版本。
`cargo run`会编译源文件并自动运行所生成的可执行文件。
`cargo build --release`会在`target/release`下编译生成可执行文件，用于正式发布而不是debug。
一个完整的rust包被成为一个crate
#### 3 一个猜数字游戏
```rust
use std::io;
fn main() {
    println!("Please input your guess: ");

    let mut guess = String::new(); //mut表示一个可变变量
    io::stdin().read_line(&mut guess).expect("Failed to read line");
    println!("You guessed: {}", guess);
}
```
`let`表示声明一个变量，`mut`表示这个变量是可变类型的变量。

`String`是标准库提供的字符串类型。双引号`::`表示后面要执行的`new`方法是`String`的关联函数（类似于静态方法），返回一个空字符串。

`use std::io`表示引入`std`库中的`io`库，`std`库是`rust`的标准库，rust程序预导入了一些库例如`String`，但io库需要显式导入。

`io::stdin()`返回了一个`std::io::Stdin`的实例，这是一个引用终端输入的句柄（==句柄是一种特殊指针，用于间接地引用特殊资源如文件、用户输入、设备等，只能调用系统提供的接口而不能直接修改==。句柄与一般指针的区别在于普通的指针是内存地址，而句柄是操作系统提供的标识符）。
`.read_line(&mut guess)`表示调用`Stdin`句柄的`read_line`方法。

`&`表示参数是一个引用，并将`guess`作为一个引用传入这个方法的参数。

`read_line`方法会返回一个`result`类型的值，这个值是一个枚举类型。`result`的`expect`方法会在`result`属于`Ok`枚举类型时返回`result`本身，而`expect`为`Err`时返回`expect`方法的参数并直接终止进程，除非继续进行错误处理。
如果一个方法返回`result`类型，但没有继续调用`expect`方法，那么编译时会进行警告。

##### 3.1 生成随机数并比较
##### 3.1.1 引入一个crate
在Cargo.toml中写入：
```toml
[dependencies]
rand = "0.8.5"
```
或在命令行中运行：`cargo add rand@0.8.5`
###### 3.1.2 导入Rng
```rust
use rand::Rng;
```
Rng是rand库中的一个trait。trait相当于一个interface。生成随机数的方法在Rng中定义。Rust规定必须首先引入这个trait才能使用trait上的方法。
###### 3.1.3 生成随机数
```rust
let secret_num = rand::thread_rng().gen_range(1..=101)
```
`thread_rng`是rand库中的一个模块函数（注意不是关联函数，关联函数通常是一个类型impl上的类静态方法），用于生成一个线程本地的随机数生成器，他会从操作系统中获取一个随机数种子。这个生成器属于一个具体的Rng trait的实现。这个Rng存在一个gen_range方法，`1..=101`是一个范围表达式，指>=1并<101的范围。
###### 3.1.4 类型转换和比较
```rust
use std::cmd::Ordering

let gusee: i32 = guess.trim().parse().expect("Please type a number!");
match guess.cmp(&target_number){
	Ordering::Less => { println!("Too small!") },
	Ordering::Greater => { println!("Too big") },
	Ordering::Equal => { println!("Thats right") }
}
```
rust允许声明同一名称的变量即变量遮蔽。由于字符串的parse方法返回类型需要推断后确定，因此必须显式声明第二个guess的类型为i32.
Ordering是一个枚举，有Less、Greater、Equal三个成员。