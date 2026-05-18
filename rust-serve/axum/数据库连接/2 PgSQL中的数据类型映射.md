### 1 日期时间
`chrono`中的日期时间类型与PgSQL中的时间类型有明确的映射关系：
`chrono::NaiveDate`对应Pg中的`DATE`类型（无时区）
`chrono::NaiveDateTime`对应Pg中的`TIMESTAMP`类型（无时区）
`chrono::DateTime<Utc>`对应Pg中的`timestamptz`类型（有时区）
时间类型的列可以映射到Rust `chrono`中的时间类型，也可以映射为字符串。
```rust
use chrono::{NaiveDateTime, DateTime, Utc};
use sqlx::FromRow;
#[derive(FromRow, Debug)]
struct Order {
	id: i32,
	create_time: DateTime<Utc>
}
let orders = sqlx::query_as!(
	Order,
	"SELECT (id, create_time) FROM orders"
).fetch_all(&pool).await?;
```
#### 1.1 映射到字符串
如果SQL语句中使用了`to_char`函数格式化时间，那么一把需要映射到字符串
```rust
use sqlx::FromRow;
#[derive(FromRow, Debug)]
struct Order {
	id: i32,
	create_time: String
}
let orders = sqlx::query_as!(
	Order,
	r#"
	SELECT (id, to_char(create_time, 'YYYY-MM-DD HH24:MI:SS') as create_time) 
		FROM orders"#
).fetch_all(&pool).await?;
```
### 2 Uuid
`uuid::Uuid`与Pg中的Uuid可以直接映射
### 3 枚举类型
假设PgSQL中创建了一个枚举类型：
```sql
CREATE TYPE user_role AS ENUM ('admin', 'normal', 'vistor');
CREATE TABLE person (
    id SERIAL PRIMARY KEY,
    name TEXT,
    user_role user_role,
);
```
然后使用`sqlx::Type`宏指定一个枚举与PgSQL中的枚举对应，其中`type_name`必须与SQL中的枚举名称一致，`rename`必须与枚举中的变体名称一致（或使用`rename_all`指定所有重命名写法）。枚举不能携带值：
```rust
use sqx::{FromRow, Row};
#[derive(sqlx::Type, Debug, Clone, PartialEq)]
#[sqlx(type_name = "roles")] // 必须与 PostgreSQL 中的枚举名一致
// #[sqlx(rename_all = "camelCase")] 全部重命名
pub enum UserRole {
    #[sqlx(rename = "admin")]
    Admin,
    #[sqlx(rename = "normal")]
    Normal,
    #[sqlx(rename = "visitor")]
    Visitor,
}
// 在结构体中使用
#[derive(FromRow, Debug)]
pub struct Person {
    pub id: i32,
    pub name: String,
    pub user_role: UserRole,
}
let person = sqlx::query_as<_, Person>("SELECT * FROM person WHERE id = 1")
	.fetch_one(&pool)
	.await?;
```

### 4 几何类型
