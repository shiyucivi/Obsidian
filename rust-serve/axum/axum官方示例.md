## 📚 Axum 示例学习路线图

### 第一阶段：基础入门（必学）

1. **hello-world** - 最简单的 Axum 应用，了解基本结构
2. **readme** - README 中介绍的示例，了解核心概念
3. **routes-and-handlers-close-together** - 学习如何组织路由和处理器
4. **print-request-response** - 学习如何打印请求和响应，调试基础

### 第二阶段：请求数据处理

5. **form** - 处理表单数据
6. **parse-body-based-on-content-type** - 根据内容类型解析请求体
7. **multipart-form** - 处理文件上传（多部分表单）
8. **consume-body-in-extractor-or-middleware** - 在提取器或中间件中消费请求体

### 第三阶段：错误处理（重要）

9. **error-handling** ⭐ - 学习如何处理和转换错误
10. **customize-extractor-error** - 自定义提取器错误
11. **customize-path-rejection** - 自定义路径拒绝错误
12. **anyhow-error-response** - 使用 anyhow 进行错误处理
13. **global-404-handler** - 全局 404 处理器

### 第四阶段：状态管理与依赖注入

14. **dependency-injection** ⭐ - 学习状态管理和依赖注入
15. **key-value-store** - 实际的键值存储示例
16. **request-id** - 请求 ID 中间件

### 第五阶段：中间件与日志

17. **tracing-aka-logging** - 日志记录
18. **compression** - 响应压缩
19. **cors** - 跨域资源共享
20. **handle-head-request** - 处理 HEAD 请求

### 第六阶段：模板与静态文件

21. **templates** - 模板引擎使用
22. **templates-minijinja** - Minijinja 模板
23. **static-file-server** - 静态文件服务

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
41. **json-error-response** - JSON 错误响应
42. **reqwest-response** - 使用 reqwest 客户端
43. **serve-with-hyper** - 与 Hyper 集成
44. **reverse-proxy** - 反向代理
45. **http-proxy** - HTTP 代理

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

## 🎯 学习建议

1. **标记 ⭐ 的是重点推荐学习的示例**
2. 每个示例都可以通过 `cargo run -p example-<name>` 运行
3. 建议按顺序学习，每个示例运行并修改代码实验
4. 前四个阶段是基础，必须扎实掌握
5. 数据库集成阶段选择你需要的数据库学习即可
6. 实时通信和认证是 Web 开发的重要技能

## 快速开始命令

```bash
# 运行 hello-world 示例
cargo run -p example-hello-world

# 运行后访问 http://127.0.0.1:3000
```

祝你学习愉快！如果对某个示例有疑问，随时问我。