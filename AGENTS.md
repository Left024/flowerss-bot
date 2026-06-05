# AGENTS.md — flowerss-bot

## 构建与测试

```bash
# 运行所有测试（CI 还会加 -race）
make test            # go test ./... -v

# 构建二进制（通过 ldflags 注入版本信息）
make build           # 输出 flowerss-bot

# 本地运行
make run             # go run .
```

CI 矩阵覆盖 Go 1.18/1.19 + ubuntu/macOS，提交前至少确保 `go test ./...` 通过。
项目没有 golangci-lint 配置，无需运行 lint。

## 配置

项目需要 `config.yml`（仓库外，不可提交）。参考 `config.yml.sample` 创建。
- 默认使用 **SQLite**（路径 `./data.db`），也支持 MySQL
- `bot_token` 为必填项
- `telegraph_token` 等 Telegraph 相关字段用于 Instant View 功能（可选）
- SOCKS5 代理通过 `socks5` 字段配置

## 架构概览

```
main.go → core → bot + scheduler
               ↓
         storage (GORM) ↔ model
               ↓
         SQLite / MySQL
```

| 层 | 目录 | 说明 |
|---|---|---|
| 入口 | `main.go` | 组装 core/bot/scheduler，处理 OS 信号 |
| Bot | `internal/bot/` | telebot v3 命令注册、中间件（用户过滤、管理员检查）、按钮回调 |
| Core | `internal/core/` | 订阅/退订 CRUD、错误计数、核心业务编排 |
| 调度器 | `internal/scheduler/` | Observer 模式轮询 RSS 源，通知 bot 推送新内容 |
| 存储 | `internal/storage/` | GORM 实现，接口定义在 `storage.go`，测试 mock 在 `mock/` |
| 模型 | `internal/model/` | Source、Subscribe、Content、User、Option |
| 配置 | `internal/config/` | viper 加载 YAML，ldflags 注入版本号 |
| 日志 | `internal/log/` | 基于 `go.uber.org/zap` |
| Feed | `internal/feed/` | 基于 `gofeed` 解析 RSS/Atom |
| Preview | `internal/preview/` | `telegraph-go` 发布到 Telegraph 获取 Instant View |
| HTTP | `pkg/client/` | 可复用的 HTTP 客户端（超时、UA、SOCKS5 代理） |

## 关键实现细节

- **Bot 框架**：`gopkg.in/telebot.v3`，使用 **long polling**（非 webhook）
- **调度器模式**：`RssTask` 是发布者，`Bot` 是 observer——新增 bot 需调用 `task.Register()`
- **按钮回调编码**：inline keyboard 的 callback data 使用 **protobuf** 序列化 `userId:sourceId` 对（`internal/bot/session/`）
- **消息模板**：订阅推送消息使用 Go template 渲染，支持 `HTML` 和 `MarkdownV2` 两种解析模式
- **去重**：抓取的条目按 `hash_id`（`internal/model/id.go`）去重存储
- **OPML**：支持通过 `/import` `/export` 命令导入/导出 RSS 订阅列表
- **数据库迁移**：通过 GORM `AutoMigrate` 在 `core.Init()` 中自动执行
