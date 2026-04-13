JWT（JSON Web Token）是一种无状态的安全认证机制。
JWT由三部分组成，格式为`Header.Payload.Signature`：
- `Header`：令牌加密算法
- `Payload`：用户身份信息（Id、角色）与过期时间、签发时间等元数据
- `Signature`：签名，服务器对`Header`和`Payload`的加密，用于校验数据合法性
认证流程：
1. 用户登录后服务器使用私钥生成一个JWT返回给客户端
2. 客户端存储在本地会话中
3. 客户在后续请求中的header中携带`Authorization: Bearer <token>`
4. 服务器收到后先从header中提取JWT，再使用密钥对`Header`和`Signature`计算签名，与JWT中的签名对比校验
5. 如果对比一致且未过期，则认证通过
### 1 rust实践
#### 1.1 加密
jwt加密需要使用`jsonwebtoken` crate。这个crate提供了`encode`方法和`decode`方法来进行生成token和解密token。
1. 首先生成一对非对称密钥分别用于加密和解密：
```rust
use jsonwebtoken::{
	decode, encode, DecodingKey, EncodingKey, Header, Validation
};
struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}
impl Keys {
    fn new(secret: &[u8]) -> Self {
        Keys {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret)
        }
    }
}
// 将密钥对放入全局静态锁中并设置懒初始化
static KEYS: LazyLock<Keys> = LazyLock::new(|| {
    let secret: &'static str = "secret";
    Keys::new(secret.as_bytes())
});
```
2. 在用户登录接口使用加密方法返回token
`jsonwebtoken::encode`方法需要一个元数据结构体的引用和一个密钥作为参数，返回加密字符串。这里省去自定义错误类型`AuthError`的定义
```rust
// claims代表用户信息的元数据
// 登录表单
#[derive(Deserialize)]
struct LoginPayload {
    username: String,
    pwd: String
}
// 登录返回的加密响应体
#[derive(Serialize)]
struct AuthBody {
    access_token: String, // 加密后的jwt
    token_type: String //token类型，通常硬编码为Bearer
}
impl AuthBody {
    fn new(token: String) -> Self {
        Self { access_token: token, token_type: "Bearer".to_string() }
    }
}
#[derive(Debug, Clone, Deserialize, Serialize)]
struct Claims {
    username: String, // 用户名
    role: String, // 身份
    exp: usize // 过期时间，单位秒
}
// 返回带有token的Response
async fn login(Json(payload): Json<LoginPayload>) -> Result<Json<AuthBody>, AuthError> {
    if payload.username.is_empty() || payload.pwd.is_empty() {
        return Err(AuthError::MissingCredentials)
    };
    if payload.username != "foo" || payload.pwd != "bar" {
        return Err(AuthError::WrongCredentials);
    };
    let userdata = Claims {
        username: payload.username,
        role: "Admin".to_string(),
        exp: 200000
    };
    let token = encode(&Header::default(), &userdata, &KEYS.encoding).map_err(|e| {
        AuthError::TokenCreation
    })?;
    Ok(Json(
        AuthBody::new(token.to_string())
    ))
}
```
#### 1.2 解密
`jsonwebtoken::decode`方法接收token字符串和解密密钥作为参数。通常使用一个`token`提取器或者中间件进行解密处理，解密不成功则返回`Err`：
```rust
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
// 为用户元数据结构体实现FromRequestParts实现提取器
impl <S> FromRequestParts<S> for Claims
where S: Send + Sync,
{
    type Rejection = AuthError;
    async fn from_request_parts(
        parts: &mut Parts, state: &S,
    ) -> Result<Claims, Self::Rejection> {
        // 从Header中提取出token
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;
        // 解密token，需要传入泛型参数即加密时的元数据类型
        let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;
        Ok(token_data.claims)
    }
}
```