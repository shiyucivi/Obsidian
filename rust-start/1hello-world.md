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
`cargo new`创建的项目中会有一个Cargo.toml的配置文件。
`cargo build`会在target/debug下编译生成一个可执行文件，还有一个cargo.lock文件用于记录依赖版本。
`cargo run`会编译源文件并自动运行所生成的可执行文件。
`cargo build --release`会在target/release下编译生成可执行文件，用于正式发布而不是debug。
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
let表示声明一个变量，mut`示这个变量是可变类型的变量。

`String`是标准库提供的字符串类型。双引号`::`表示后面要执行的`new`方法是`String`的关联函数（类似于静态方法），返回一个空字符串。

`use std::io`表示引入std库中的`io`库，std库是rust的标准库，rust程序预导入了一些库例如String，但io库需要显式导入。

`io::stdin()`返回了一个std::io::Stdin的实例，这是一个引用终端输入的句柄（==句柄是一种特殊指针，用于间接地引用特殊资源如文件、用户输入、设备等，只能调用系统提供的接口而不能直接修改==。句柄与一般指针的区别在于普通的指针是内存地址，而句柄是操作系统提供的标识符）。
`.read_line(&mut guess)`表示调用Stdin句柄的read_line方法。

&表示参数是一个引用，并将guess作为一个引用传入这个方法的参数。

read_line方法会返回一个result类型的值，这个值是一个枚举类型。result的expect方法会在result属于Ok枚举类型时返回result本身，而result为Err时返回expect方法的参数并直接终止进程，除非继续进行错误处理。
如果一个方法返回result类型，但没有继续调用expect方法，那么编译时会进行警告。

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
let secret_num = rand::thread_rng().gen_range(1..101)
```
`thread_rng`是rand库中的一个模块函数（注意不是关联函数，关联函数通常是一个类型impl上的类静态方法），用于生成一个线程本地的随机数生成器，他会从操作系统中获取一个随机数种子。这个生成器属于一个具体的Rng trait的实现。这个Rng存在一个gen_range方法，`1..101`是一个范围表达式，指>=1并<101的范围。
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