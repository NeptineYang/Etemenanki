# Worker 与 runner 协议

- 状态：V1 设计基线
- 日期：2026-11-17
- 依据：DESIGN.md §5.1、§5.2、§11，Q21、Q32、Q33、Q39、Q41、Q56
- 上位文档：[DESIGN.md](../../DESIGN.md)

## 1. 角色与链路

| 链路 | 进程位置 | 启动方式 | 用途 |
| --- | --- | --- | --- |
| Core ↔ Java Worker | Windows 本机子进程 | Core 拉起，随包的精简 JRE | Java AST、符号与类型解析、classpath 证据 |
| Core ↔ Python Worker | Windows 本机子进程 | Core 拉起，随包的 embeddable Python | Python CST、作用域与类型证据、目标代码修改 |
| Core ↔ runner | WSL2 发行版内 | `wsl.exe` 拉起，标准输入输出即通道 | 依赖获取、构建、测试、截图、沙箱内文件操作 |

三条链路使用同一套消息格式与 schema，仅方法集不同。所有链路都不监听端口，不使用 token。

## 2. 分帧

- 连接是单工 stdin/stdout 对：Core 写子进程 stdin，读子进程 stdout。
- 帧格式为 `1 字节类型 + 4 字节大端长度 + 载荷`。类型 `0x01` 为 JSON 消息，类型 `0x02` 为二进制载荷（tar 流、补丁包、截图）。
- JSON 消息是单行 UTF-8 编码的 JSON-RPC 2.0 对象；纯 JSON 场景下等价于 NDJSON。
- 单帧上限默认 8 MiB，超过时改用二进制帧分块传输，块大小 4 MiB；上限可在 `initialize` 中协商。接收方必须流式处理二进制帧，不得整块读入内存后再落盘。
- stdout 只承载协议帧。日志、进度提示、第三方工具的原始输出一律写 stderr，并带 trace ID 前缀，防止污染协议。
- 子进程退出码非 0 时，Core 以 stderr 尾部与最后一条协议消息作为失败证据。

## 3. 消息集（初版）

方法名在 V1-A 冻结。初版范围：

| 方法 | 方向 | 说明 |
| --- | --- | --- |
| `initialize` | Core → Worker | 交换协议版本、能力位、工具链清单、并发上限 |
| `workspace.import` / `workspace.export` | 双向 | 以二进制帧传输 tar 流，导入源项目快照或导出补丁与产物 |
| `project.scan` | Core → Worker | 建立文件清单、构建系统识别、配置与资源清点 |
| `parse.file` / `parse.project` | Core → Worker | 返回 AST 摘要与源位置，不返回内部节点句柄 |
| `symbol.resolve` | Core → Worker | 类型、签名、继承与实现关系 |
| `types.infer` | Core → Worker | 类型证据与置信度，供迁移 IR 使用 |
| `deps.resolve` | Core → runner | 两阶段依赖解析与下载，返回坐标、哈希与缺口 |
| `build.run` / `test.run` | Core → runner | 返回结构化结果、原始日志引用与产物路径 |
| `screenshot.capture` | Core → runner | 受控环境截图，写入工作区 |
| `fs.read` / `fs.write` / `fs.list` | Core → 任一方 | 仅限工作区根目录内，路径经规范化校验 |
| `cancel` | Core → 任一方 | 通知，按 `request id` 取消未完成调用 |
| `progress` / `log` | 任一方 → Core | 通知，携带 trace ID 与阶段 |
| `shutdown` | Core → 任一方 | 优雅退出；超时后强制结束进程 |

## 4. 版本握手

- 应用版本全仓统一（见 [versioning.md](../process/versioning.md)），但协议另有独立的 schema 版本 `major.minor`。
- `initialize` 交换 `app_version`、`schema_version`、能力位与工具链清单。
- `schema_version` 主版本不一致时拒绝启动并提示升级；次版本不一致时按“新方兼容旧方”降级，缺少的能力位必须显式上报，不得静默忽略。
- Core 升级后自动把匹配版本的 runner 推送进发行版，避免应用与 runner 版本错配。

## 5. 错误分类

| 类别 | 含义 | 是否可重试 |
| --- | --- | --- |
| `protocol` | 帧损坏、版本不匹配、未知方法 | 否 |
| `environment` | 缺少 WSL2、工具链下载失败、磁盘不足 | 视情况，需用户处理 |
| `project` | 源项目无法构建、依赖缺失、语法不支持 | 否，转缺口报告 |
| `budget` | 超出时间、轮数、token 或成本预算 | 否，输出人工任务 |
| `internal` | 断言失败、未捕获异常 | 否，附 trace ID 与最小复现信息 |

每条错误包含 `code`、`message`、`retryable`、`evidence` 与 `trace_id`。错误不得吞掉原始原因。

## 6. 取消、超时与心跳

- 每个请求带唯一 `id`；取消以通知形式发送，接收方必须在下一个检查点响应。
- 心跳间隔 5 秒，连续 15 秒未收到心跳即判定进程失联并记录诊断信息。
- 默认超时：握手 30 秒；单文件解析 300 秒；依赖解析 1800 秒；构建与测试 3600 秒；截图 120 秒。超时值按方法配置，不设全局统一值。
- 长任务每 10 秒至少上报一次 `progress`，否则视为挂起。超时与挂起都产生 `environment` 或 `internal` 错误，并保留最后一条进度。

## 7. 任务状态机

```
queued → preparing → baseline → analyzing → planning
       → generating → verifying → awaiting_review → completed
                                             ↘ failed / cancelled
```

- 状态持久化在 SQLite，每个批次有幂等 ID 与检查点。
- 崩溃后从最近检查点恢复，不重复覆盖已审核结果。
- `awaiting_review` 是人工闸门，不是自动完成状态。
- 并发上限默认值为：解析 `min(4, CPU 物理核数 / 2)`，构建与测试 1，截图 1；构建串行是为了避免缓存与端口争用。上限可在设置中下调，不允许超过默认值的两倍。

## 8. 类型定义

- Worker 协议、迁移 IR、报告结构以 JSON Schema 为唯一来源，存放于 `schemas/`。
- Rust、TypeScript、Java、Python 的类型由 schema 生成，不手写重复定义。
- schema 变更必须同步契约测试；破坏性变更按中间位版本处理。

## 9. 缓存与存储

### 9.1 模型回放缓存

- 键为 `SHA256(provider + model_snapshot + prompt_version + messages + tool_schema)`，值为原始响应与结构化结果，存 SQLite。
- 默认关闭；基准测试、回归与问题复现时开启。命中缓存必须在审计中记录，不得伪装成新的模型调用。
- 容量上限 2 GiB，按最近最少使用淘汰；缓存不跨项目共享，删除项目时一并清除。

### 9.2 SQLite 迁移

- 只做前向迁移，不提供 down migration。
- 启动时比对 `schema_version`；迁移前把数据库复制为 `app.db.bak-<from_version>`，迁移失败即回滚备份并拒绝启动。
- 迁移脚本随应用版本发布，禁止在运行时拼接 DDL。

## 10. V1-A 冻结项

- 方法名与字段的最终命名（当前为初版，允许在 V1-A 调整一次）。
- `initialize` 能力位的完整清单。
