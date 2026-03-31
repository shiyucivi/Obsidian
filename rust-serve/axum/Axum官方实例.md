
### 1 基础路由与处理器
1. **hello-world** - 最简单的 Axum 应用，了解基本结构
2. **readme** - README 中介绍的示例，了解核心概念
3. **routes-and-handlers-close-together** - 学习如何组织路由和处理器
4. **global-404-handler** - 全局 404 处理器
### 2 提取器与响应
1. **form** - 处理表单数据
2. **multipart-form** - 处理FormData表单与文件上传
3. **stream-to-file** - 流式处理文件上传
4. **handle-head-request** - 处理 HEAD 请求
5. **parse-body-based-on-content-type** - 根据内容类型解析请求体
6. **consume-body-in-extractor-or-middleware** - 在提取器或中间件中消费请求体
### 3 自定义提取器与错误处理
1. **customize-extractor-error** - 自定义提取器错误
2. **customize-path-rejection** - 自定义路径提取器错误
3. **anyhow-error-response** - 使用 anyhow 进行错误类型归一化处理
4. **error-handling** ⭐ - 错误处理综合+扩展类型
### 4 状态管理与依赖注入
1. **dependency-injection** ⭐ 状态管理和依赖注入
2. **key-value-store** - 实际的键值存储示例
### 5 中间件Layer与tower-http生态
1. **print-request-response** - 使用中间件打印请求和响应，`from_fn`快速构建中间件
2. **request-id** - 请求 id 中间件，为每个请求和响应生成唯一id
3. **tracing-aka-logging** - 日志记录
4. **compression** - 响应压缩
5. **cors** - 跨域资源共享
6. **key-value-store** - 键值存储状态，多种中间件结合使用（压缩，错误处理，负载脱落，并发限制，超时，打印）
### 6 模板与静态文件
1. **templates** - 模板引擎使用
2. **templates-minijinja** - Minijinja 模板
3. **static-file-server** - 静态文件资源服务

### 第七阶段：实时通信

24. **websockets** ⭐ - WebSocket 基础
25. **testing-websockets** - WebSocket 测试
26. **sse** - Server-Sent Events
27. **chat** - 聊天室应用（WebSocket 实战）

### 第八阶段：数据库集成

28. **sqlx-postgres** - SQLx + PostgreSQL
29. **tokio-postgres** - PostgreSQL 异步驱动
30. **tokio-redis** - Redis 集成
31. **diesel-postgres** - Diesel ORM + PostgreSQL
32. **diesel-async-postgres** - Diesel 异步版本
33. **mongodb** - MongoDB 集成

### 第九阶段：认证与安全

34. **jwt** ⭐ - JWT 认证
35. **oauth** - OAuth 认证
36. **validator** - 数据验证
37. **tls-rustls** - TLS/HTTPS（使用 Rustls）
38. **tls-graceful-shutdown** - TLS + 优雅关闭

### 第十阶段：高级特性

39. **graceful-shutdown** ⭐ - 优雅关闭
40. **testing** ⭐ - 测试技巧
41. **reqwest-response** - 使用 reqwest 客户端
42. **serve-with-hyper** - 与 Hyper 集成
43. **reverse-proxy** - 反向代理
44. **http-proxy** - HTTP 代理

### 第十一阶段：其他特性

46. **prometheus-metrics** - Prometheus 监控指标
47. **auto-reload** - 自动重载（开发时）
48. **unix-domain-socket** - Unix 域套接字
49. **stream-to-file** - 流式写入文件
50. **async-graphql** - GraphQL 集成
51. **todos** - TODO 应用（完整示例）
52. **versioning** - API 版本控制
53. **simple-router-wasm** - WASM 支持

### 第十二阶段：底层与进阶

54. **low-level-rustls** - 底层 TLS（Rustls）
55. **low-level-native-tls** - 底层 TLS（Native TLS）
56. **low-level-openssl** - 底层 TLS（OpenSSL）