`serde`提供了`Serialize`和`Deserialize`两大trait，赋予了一个类型可以被序列化为某种数据结构的能力。同时还提供`Serializer`和`Deserializer`两个trait用于序列化为数据结构的具体实现。`serde_json`、`axum`中提供的序列化与反序列化功能都是对`serde`中提供的`Serializer`和`Deserializer`trait的具体实现。
### 1 基本用法
#### 1.1 添加依赖
```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"  # 或 serde_yaml, toml 等
```
#### 1.2 为结构体增加序列化与反序列化trait
`serde`提供了`Serialize`、`Deserialize`派生宏为结构体实现这两个trait。`Serialize`宏允许类型序列化为JSON、YAML等字符串。`Deserialize`宏允许JSON等反序列化为Rust类型。
```rust
use serde::{Serialize, Deserialize};
#[derive(Serialize, Deserialize)]
struct Person {
	name: String,
	age: u32,
}
// 序列化为josn
let p = Person { name: "Alice".to_string(), age: 22 };
let json = serde_json::to_string(&p).unwrap();
/// 反序列化
let json = r#"{"name":"Bob","age":25}"#;
let p: Person = serde_json::from_str(json).unwrap();
```
#### 1.3 trait bound
一个结构体要实现`Serialize`和`Deserialize`两大trait，其中的所有字段也都必须满足这两个trait。`serde`对rust中的大部分类型（`String`、`Vec<T>`等）都实现了这两个trait。
如果某个字段的类型未实现这两个trait。有这些处理方法：
- 手动为这个字段的类型实现trait
- 使用`#[serde(skip)]`或`#[serde(default)]`属性宏跳过这个字段或提供默认值。
- 提供自定义序列化和反序列化函数
### 2 常用属性宏
`serde`提供了多种属性宏来自定义序列化和反序列化中的行为
#### 2.1 字段重命名
```rust
#[derive(Serialize, Deserialize)]
struct User {
    #[serde(rename = "userId")]
    user_id: u32,
    #[serde(rename = "userName")]
    name: String,
}
// JSON: { "userId": 42, "userName": "Alice" }
```
#### 2.2 忽略字段
```rust
#[derive(Serialize, Deserialize)]
struct User {
    id: u32,
    #[serde(skip)]
    temp_cache: String, // 不参与序列化/反序列化
}
```
#### 2.3 `#[serde(default)]`默认值
当JSON中缺少该字段时，序列化时使用默认值，注意需要该类型实现`Default trait` 
```rust
#[derive(Serialize, Deserialize, Default)]
struct Config {
	#[serde(default)]
	timeout: u32, //默认为0
	#[serde(default = "default_retries")] //使用函数作为默认值
    retries: u32,
}
fn default_retires -> u32 {
	3
}
```
#### 2.4 处理不必要字段
默认的反序列化是忽略JSON中存在而类型中不存在的字段的。无需额外处理。
#### 2.5 结构体嵌套
`#[serde(flattern)]`宏可以将另一个类型中的字段扁平化地嵌入到本类型中。
```rust
#[derive(Serialize, Deserialize)]
struct User {
    id: u32,
    #[serde(flatten)]
    metadata: Metadata,
}
#[derive(Serialize, Deserialize)]
struct Metadata {
    created_at: String,
    updated_at: String,
}
// JSON:
// {
//   "id": 42,
//   "created_at": "2024-01-01",
//   "updated_at": "2024-01-02"
// }
```
### 3 自定义序列化与反序列化
#### 3.1 自定义序列化
使用`serde(serialize_with) = "serialize_fn"`的宏为结构体中某个字段指定序列化函数，当进行序列化时自动调用函数进行：
```rust
#[derive(Serialize)]
struct Event {
	#[serde(serialize_with = "to_timestamp")]
	time: chrono::NavieDateTime
}
// 序列化时会调用to_timestamp函数
fn to_timestamp<S: serde::Serializer>(date: &chrono::NaiveDateTime, serializer: S) -> Result<S::OK, S:ERROR> {
	serializer.serialize_i64(date.timestamp())
}
```
这个自定义序列化函数必须满足要求：参数一是字段类型的引用，参数二的类型是实现了 `serde::Serializer` 的泛型S。
`serde::Serializer`是`serde`序列化功能的核心trait，其中具有`OK`和`ERROR`两个关联类型。这个trait中定义了很多类型的序列化方法。这些方法的返回值都是`RESULT<Self::OK, Self::ERRO>`。
实现该trait的类型被称为序列化器，由其他crate提供，例如`serde_json::ser::Serializer<W>`。
#### 3.2 自定义反序列化
使用`#[serde(deserialize_with = "deserialize_fn")]`宏为字段指定反序列化函数，当进行反序列化时调用：
```rust
// 将字符串"true"、"false"反序列化为bool类型
use serde::{Deserialize, Deserializer};
fn deserialize_bool_from_str<'de, D>(deserializer: D) -> Result<bool, D::Error> where D: Deserializer<'de> {
    let s = String::deserialize(deserializer)?;
    match s.as_str() {
        "true" => Ok(true),
        "false" => Ok(false),
        _ => Err(serde::de::Error::custom("expected 'true' or 'false'")),
    }
}
#[derive(Deserialize, Debug)]
struct Config {
    #[serde(deserialize_with = "deserialize_bool_from_str")]
    enabled: bool,
}
```
`deserializer`反序列化器同样具有`OK`和`ERROR`两个关联类型。
### 4 错误处理
`serde_json::to_string()`、`serde_json::from_str()` 等函数返回 `Result<T, Error>`。当出现 `Err` 时，应妥善处理。
`axum`中的提取器在遇到序列化错误时，会自动捕获提取失败，并返回 400 Bad Request。
#### 4.1 容忍部分字段出现序列化错误
想要避免部分字段序列化错误导致整体错误，可以将可能出错的字段为包装为`Option`或`Result`，当`None`时或`Err`时跳过这个字段
```rust
#[derive(Serialize)]
struct Response {
    data: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    debug_info: Option<String>, // 如果生成失败，设为 None，JSON 中不出现
}
```
#### 4.2 容忍部分字段出现反序列化错误
##### 4.2.1 处理空字段：将字段设置为`Option`或提供默认值
将字段设置为`Option`，当没有字段时字段值为`None`
```rust
#[derive(Deserialize)]
struct SearchParams {
    name: Option<String>,
    age: Option<u32>,
}
```
或提供默认值
```rust
#[derive(Deserialize)]
struct SearchParams {
    #[serde(default = "default_name")]
    name: String,
    #[serde(default)]
    age: u32, // 默认为 0
}
fn default_name() -> String {
    "Anonymous".to_string()
}
```
##### 4.2.2 用 `Result<T, serde_json::Error>` 包装字段
这种方式需要自定义反序列化函数
```rust
use serde::{Deserialize, Deserializer};
use serde_json;

#[derive(Deserialize)]
struct Response {
    id: u32,
    #[serde(deserialize_with = "deserialize_ok_or_err")]
    data: Result<i32, serde_json::Error>,
}

fn deserialize_ok_or_err<'de, D>(deserializer: D) -> Result<Result<i32, serde_json::Error>, D::Error>
where
    D: Deserializer<'de>,
{
    // 先反序列化为 serde_json::Value
    let value = serde_json::Value::deserialize(deserializer)?;
    // 再尝试解析为 i32，失败则保留错误
    Ok(i32::deserialize(value).map_err(|e| e))
}
```
### 5 `Vec`的序列化与反序列化
只要`Vec<T>`中T实现了序列化或反序列化，就可以轻松使用`serde_json`等工具进行序列化与反序列化：
```rust
#[derive(Serialize, Deserialize, Debug)]
struct Person {
    name: String,
    age: u32,
}
fn main() {
    let people = vec![
        Person {
            name: "Alice".to_string(),
            age: 30,
        },
        Person {
            name: "Bob".to_string(),
            age: 25,
        },
    ];
    //序列化：Rust Vec -> JSON 数组
    let json_str = serde_json::to_string_pretty(&people).unwrap();
    //反序列化
	let deserialized: Vec<Person> = serde_json::from_str(&json_str).unwrap();
```
### 6 嵌套结构
对于嵌套类型的结构体`struct<T>`，只要对T派生了`Serialize` 和 `Deserialize` trait 即可，serde会自动处理嵌套字段。
对于这样一个Json：
```json
{
  "id": "order-123",
  "customer": {
    "name": "Alice",
  },
  "items": [
    {
      "product": "Book",
      "price": 19.99
    },
    {
      "product": "Pen",
      "price": 1.50
    }
  ]
}
```
针对每个子结构单独派生`Serialize` 和 `Deserialize`
```rust
use serde::{Deserialize, Serialize};
#[derive(Serialize, Deserialize, Debug)]
struct Order {
    id: String,
    customer: Customer,
    items: Vec<Item>,
}
#[derive(Serialize, Deserialize, Debug)]
struct Customer {
	name: String,
}
#[derive(Serialize, Deserialize, Debug)]
struct Item {
	product: String,
	price: f64
}
// 反序列化：
let order: Order = serde_json::from_str(json_body).unwrap();
```
### 7 隐式类型转换
`serde_json`所提供的序列化与反序列化是严格类型匹配的，不进行任何隐式类型转换。如果JSON中存在与结构体定义类型不同的字段，会导致反序列化失败，需要进行额外处理。
对于可能数字和字符串、布尔值混传的字段，可以用`serde_with`进行宽松化处理：
```rust
use serde::{Deserialize, Serialize};
use serde_with::{serde_as, DisplayFromStr};

#[serde_as]
#[derive(Deserialize, Serialize)]
struct Payload {
    #[serde_as(as = "DisplayFromStr")]
    id: i32,
    #[serde_as(as = "DisplayFromStr")]
    age: u8,
    #[serde_as(as = "DisplayFromStr")]
    active: bool,
}
```
 `DisplayFromStr` 表示：**接受任意类型，先转成字符串，再用 `FromStr` trait 解析为目标类型**。
