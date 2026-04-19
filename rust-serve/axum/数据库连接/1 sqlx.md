### 1 创建数据库连接池
sqlx连接数据库需要创建并配置数据库连接池，创建连接池首先需要指定一个连接url。url标准格式为：
```
postgres://[user][:password]@[host][:port][/database][?parameters]
```
例如用户名为postgre，密码为`a12345`，数据库地址为`localhost:5432`，数据库名称为mydb，那么url为：
```
postgres://postgres:a12345@localhost:5432/mydb
```
实际中通常不硬编码连接url，而是在启动环境变量或者`.env`环境变量（需要`dotenv`）中配置。
#### 1.1 配置数据库连接池
使用`sqlx::postgres::PgPoolOptions`创建连接池。并使用共享状态注入。
```rust
use sqlx::postgres::PgPoolOptions;
let db_url = std::env::var("DB_URL").unwrap();
let pool = PgPoolOption::new() 
	.max_connection(5) //最大连接数
	.acquire_timeout(std::Time::Duration::from_secs(15))
	.connect(&db_url)
	.expect("connect db failed");

let app = axum::Router::new()
	.router("/", get(handler))
	.with_state(pool);
```
### 2 SQL查询语句
#### 2.1 宏
宏会在编译时连接数据库，解析SQL语句、验证语法、表/列是否存在。并推断返回的字段类型。只能使用字面量SQL语句不能动态拼接（例如`format!`等是不被允许使用的，会引发SQL注入问题）。
##### 2.1.1 `query!`
`query!`查询宏会返回一个匿名结构体，其中包含了`SELECT`语句中的字段。
```rust
use axum::{extractor::{State, Path};
use sqlx::{PgPool};

async fn get_city_handler(
	State(pool): State<PgPool>, 
	Path(city_name): Path<String>
) -> Result<String, (StatusCode, String)> {
	let row = sqlx::query!("SELECT name FROM cities WHERE name = $1", city_name)
		.fetch_one(&pool)
		.await
		.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
	Ok(format!("city: {}!", row.name))
}
```
`fetch_one`是SQL语句的执行方法，表示只查找一行，如果SQL返回多行或0行都会返回`Err`。
##### 2.1.2 `query_as!`
`query_as`用来将SQL查询结果映射到一个类型上，需要对该类型派生`slqx::FromRow` trait。
```rust
use axum::{Json};

#[derive(serde::Deserialize, sqlx::FromRow)]
struct City {
	name: String
}
async fn get_city_handler(
	State(pool): State<PgPool>, 
	Path(city_name): Path<String>
) -> Result<Json(city), (StatusCode, String)> {
	let city = sqlx::query_as!(City, "SELECT name FROM cities WHERE name = $1", city_name)
		.fetch_one(&pool)
		.await
		.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
	Ok(Json(city))
}
```
`query_as!`是最推荐的`SELECT`查询宏，类型安全且性能好。
#### 2.1.3 `query_scalar!`
`query_scalar`用于查询单个标量值（如 `COUNT(*)`, `MAX(id)` 等），需要声明该值的类型。
```rust
let count: u32 = slqx::query_scalar!("SELECT count(*) from cities")
	.fetch_one(&pool)
	.await?;
```
#### 2.2 查询函数
查询方法都是运行时动态查询的，适用于动态SQL或无法在编译时确定结构的场景。使用`bind`方法传入占位符填充值。（多个占位符多次使用`bind`）。
##### 2.2.1 `query()`
类似于`query!`
```rust
let row = sqlx::query("SELECT name FROM cities WHERE id = $1")
    .bind(1i32)
    .fetch_one(&pool)
    .await?;
let name: String = row.get("name");
```
##### 2.2.2 `query_as::<_, T>()`
映射到实现了`sqlx::FromRow`的类型
```rust
#[derive(sqlx::FromRow)]
struct City {
	name: String
}
let user = sqlx::query_as::<_, User>("SELECT * FROM cities WHERE id = $1")
    .bind(1i32)
    .fetch_one(&pool)
    .await?;
```
##### 2.2.3 `query_scalar`
用于查询单个标量值
### 3 数据库增删改
#### 3.1 增改
推荐在SQL语句中使用`INSERT/UPDATE ... RETURNING ...`同时完成插入/更新与返回新数据行的操作，并使用`fetch_one`或`fetch_optional`执行：
```rust
async fn create_city(
	State(pool): State<sqlx::PgPool>,
	Json(city): Json<City>
) -> Result<Json(city), (StatusCode, String)> {
	let city = sqlx::query_as!(
		City,
		r#"INSERT INTO cities (id, name) VALUES ($1, $2)
		RETURNING id name"#,
		city.id, city.name
	).fetch_one(&pool)
	.await
	.map(|err| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
	Ok(Json(city))
}
```
#### 3.2 删除
删除通常不返回行，需要使用`execute`方法执行操作并判断操作结果是否成功：
```rust
let result = sqlx::query!("DELETE FROM cities WHERE id = $1", id)
	.execute(&pool)
	.await
	.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

if result.rows_affected() == 0 {
	Err((StatusCode::NOT_FOUND, "City not found".to_string()))
} else {
	Ok(())
}
```
### 4 语句执行方法
所有查询语句都支持以下方式：
- `.fetch_one(&db_pool)`：获取恰好一行的数据，0行或多行时返回`Err`。
- `.fetch_optional(&db_pool)`：获取0行或1行时返回`Option<T>`，否则返回`Err`。
- `.fetch_all(&db_pool)`：获取所有行，返回`vec<T>`。
- `.fetch(&db_pool)`：返回`Stream<Item=Row>`，适合流式处理处理大量数据。
- `.execute(&db_pool)`：不返回查询行，返回`Result<PgQueryResult>`，不能用于`SELECT`操作语句，适合执行修改删除行或对表的操作。
**Example：一个分页列表的示例：**
```rust
use axum::{
	extractor::{Query, State},
	http::StatusCode
};
use sqlx::{PgPool};
#[derive(serde::Deserialize)]
struct Pagination {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_size")]
    size: u32,
}
// 推荐将分页查询参数做成提取器方便复用
async fn city_list_pagination(
	Query(pagination): Query<Pagination>,
	State(pool): State<PgPool>
) -> Result<Json<Vec<City>>, (StatusCode, String)> {
	let offset = (pagination.page - 1) * pagination.size;
    let limit = pagination.size;
	let cities = sqlx::query_as!(
		City,
		"SELECT * FROM cities ORDER BY id LIMIT $1 OFFSET $2",
		limit,
		offset
	)
	.fetch_all(&pool)
	.await
	.map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
	
	Ok(Json(cities))
}
```
### 5 批量操作
PostgreSQL 支持将整个`Vec`作为参数传入，配合 `= ANY($1)` 可以安全、高效地处理成千上万条记录。
```rust
use sqlx::PgPool;
async fn batch_delete(pool: PgPool, ids: Vec<String>) {
	sqlx::query("DELETE FROM users WHERE id = ANY($1))
		.bind(ids)
		.execute(pool)
		.await.unwrap();
}
```
### 6 `QueryBuilder`动态查询
如果需要在查询语句中动态拼接（例如子句、列名等），需要使用`sqlx::QueryBuilder`构建查询对象。
例如一个查询用户列表的接口中，存在动态数量的筛选条件（用户姓名、年龄范围、邮箱）：
```rust
use sqlx::{PgPool, Row};
use sqlx::postgres::PgRow;

struct UserFilter {
	name: Option<String>,
	email: Option<String>,
	min_age: Option<u32>,
}
async fn user_list(pool: PgPool, filter: &UserFilter) ->
sqlx::Result<PgRow>
{
	let mut query_builder: sqlx::QueryBuilder<sqlx::Postgres> = 
	    sqlx::QueryBuilder::new("SELECT id, name, email, age FROM users");
	// 用于WHERE子句中的AND分隔符
	let mut separated = query_builder.separated(" AND ");
	
	if filter.name.is_some() && filter.email.is_some() && filter.min_age.is_some() 
	{
		query_builder.push(" WHERE ");
		separated = query_builder.separated(" AND ");
	}
	if Some(name) = filter.name {
		separated.push(" name = ");
		separated.bind(name);
	}
	if Some(email) = filter.email {
		separated.push(" email = ");
		separated.bind(email);
	}
	if Some(min_age) = filter.min_age {
		separated.push(" age >= ");
		separated.bind(min_age);
	}
	let query = query_builder.build();
	let rows = pool.fetch_all(pool).await?;
	Ok(rows)
}

let filter = UserFilter {
    name: Some("Alice".to_string()),
    email: None,
    min_age: Some(18),
};
let users = find_users(&pool, &filter).await?;
for row in users {
	println!(
		"id: {}, name: {}, email: {}, age: {}",
		row.get::<i32, _>("id"),
		row.get::<String, _>("name"),
		row.get::<String, _>("email"),
		row.get::<i32, _>("age")
	);
}
```
### 7 事务
使用`pool.begin`即可开启事务。这个方法返回一个事务对象`tx`，后续所有事务的操作都要在这个对象上进行。推荐使用`?`让作用域在事务出错时结束并`drop tx`，此时会自动`ROLLBACK`。
```rust
use sqlx::{Acquire, PgPool}; // 提供 .begin() 方法

async fn example(pool: &PgPool) -> Result<(), sqlx::Error> {
	// 开启事务
	let tx = pool.begin();
	// 事务操作1
	sqlx::query("INSERT INTO users (name) VALUES $1")
		.bind("Alice")
		.execute(tx).await?;
	// 创建保存点
	sqlx::query("SAVEPOINT sp1").execute(&mut *tx).await?;
	// 事务操作2
	let result = 
		sqlx::query("UPDATE accounts SET balance = balance - 100 WHERE user_id = $1")
	    .bind(1)
	    .execute(&mut *tx).await;
	if let Err(e) = result {
		// 事务2出错回到存档点
		sqlx::query("ROLLBACK TO SAVEPOINT sp1")
			.execute(tx).await?;
	} else {
		// 成功则释放存档点
		sqlx::query("RELEASE SAVEPOINT sp1")
			.execute(tx).await?;
	}
	// 事务完成并提交
	tx.commit();
	Ok(())
}
```
如果想手动让事务`ROLLBACK`，调用`tx.rollback().await`即可。
### 8 错误处理
`sqlx`提供了统一的错误类型`sqlx::Error`枚举。包括了大多数数据库错误类型如连接失败、SQL语法错误、行列不存在、数据类型匹配错误等。使用时可以按需转化为自定义错误类型（`impl From<Sqlx::Error> for MyError`）或实现`IntoResponse`。