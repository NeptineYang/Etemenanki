# 格式与数据模型

- 状态：V1 设计基线
- 日期：2026-11-18
- 依据：DESIGN.md §5.3、§5.5、§9.1、§11，Q21、Q25、Q73、Q81、Q97
- 上位文档：[DESIGN.md](../../DESIGN.md)

## 1. 总原则

- 人写与可版本控制的文件用 TOML：支持矩阵、依赖映射、应用配置。
- 机器交换与跨语言消费的文件用 JSON，并由 JSON Schema 校验：Worker 协议、迁移 IR、报告、审计记录。
- 不使用 YAML 作为新格式（`serde_yaml` 已停止维护）。
- 所有格式带显式 `schema_version`，破坏性变更按中间位版本处理。
- “配置文件也可以转写”指迁移源项目的配置文件（`application.yml`、`settings.py` 等），不是指本平台自身的配置格式。

## 2. 支持矩阵

位置：`mappings/matrix/*.toml`。一个参考栈一个文件，按框架与构建工具分节。

必备字段：

| 字段 | 说明 |
| --- | --- |
| `stack` | 参考栈标识，如 `A-api-fullstack` |
| `source` / `target` | 语言、框架、构建工具、运行时版本区间 |
| `features` | 特性白名单，逐项列出受支持的语法与 API |
| `exclusions` | 明确不支持的用法 |
| `evidence` | 对应符合性测试的路径或基准 ID |
| `status` | `guaranteed` / `experimental` / `blocked` |

判定规则：命中 `guaranteed` 走保证路径；命中 `experimental` 走实验路径并单独统计；命中 `blocked` 直接阻断并输出缺口。

示例（节选）：

```toml
schema_version = "1.0"

[stack]
id = "a-api-fullstack"
status = "guaranteed"
evidence = "tests/conformance/a-api"

[[stack.source]]
language = "java"
version = "17"
framework = "spring-boot"
framework_range = ">=3.0,<4.0"
build = "maven"

[[stack.target]]
language = "python"
version = "3.12"
framework = "fastapi"
build = "uv"

[stack.features]
supported = ["rest-controller", "jpa-repository", "bean-validation", "scheduled-task"]
excluded = ["reflection-proxy", "custom-classloader"]
```

## 3. 依赖映射库

位置：`mappings/deps/*.toml`，按生态分文件。

必备字段：

| 字段 | 说明 |
| --- | --- |
| `source` / `target` | 坐标与版本范围 |
| `license` | 源与目标的许可证标识（SPDX） |
| `maintenance` | 维护状态与最后核验日期 |
| `behavior_diff` | 默认值、异常、边界与副作用差异 |
| `evidence` | 测试或文档依据 |
| `reviewer` | 人工审核人 |
| `status` | `guaranteed` / `experimental` / `blocked` |

依赖平替不能只比较名称：功能、默认值、异常语义、维护状态与许可证都要记录。许可证受限的实现不得复制进本仓库。

示例（节选）：

```toml
schema_version = "1.0"

[[mapping]]
source = "org.springframework.boot:spring-boot-starter-web"
target = "fastapi"
license = { source = "Apache-2.0", target = "MIT" }
maintenance = { source = "active", target = "active", checked = "2026-11-17" }
behavior_diff = "默认 JSON 序列化字段命名与错误响应结构不同，需显式配置"
evidence = "mappings/tests/starter-web.md"
reviewer = "@NeptineYang"
status = "guaranteed"
```

## 4. source map

- 采用 Source Map v3，记录目标片段到源位置的映射。
- 另附一份旁注文件（JSON），记录每个映射的规则或模型、依赖决策、置信度与验证状态。
- 生成代码中的每个文件都要可追溯到源文件；无法映射的生成内容必须显式标记为 `generated`。
- 目录约定：`sourcemap/` 下按目标路径镜像存放 `<target>.map` 与同名的 `<target>.map.meta.json`（旁注）；`sourcemap/manifest.json` 统一登记 `target`、`source_map`、`sidecar`、`generated` 四个字段，作为唯一入口。
- 旁注与 map 必须同时存在；只有 map 没有旁注视为构建产物不完整，报告标为缺陷。
- 目标文件由模型整文写出（DESIGN §5.5），映射来源因此分两层：模型在写出文件时按 `stage.report` 附带符号级映射声明（目标符号 → 源节点 id 与 span），后置校正器再按 AST 校对一遍。校对不通过、或声明缺失的符号，映射粒度退为文件级并标 `generated`，报告计入“未映射符号数”。
- 文件级映射是下限而不是目标：`generated` 占比超过项目阈值时，该阶段不满足“收口”判据。

## 5. 报告

| 报告 | 格式 | 用途 |
| --- | --- | --- |
| 迁移报告 | JSON + HTML | 范围、模块状态、缺口、人工任务、映射决策 |
| 阶段计划 | JSON | 阶段契约、判据、预算、需批准事项，批准后冻结 |
| 阶段报告 | JSON | 单阶段改动、缺口、删除清单、映射声明、成本 |
| 测试报告 | JSON + HTML | 单元、集成、API、差分、E2E 结果 |
| 视觉报告 | HTML + 截图 | 零像素差对比与遮罩披露 |
| 性能报告 | JSON | 时延、吞吐、内存、启动时间 |
| 审计日志 | JSON Lines | 授权、模型调用、工具与命令、文件变更、人工批准 |

JSON 是唯一事实来源，HTML 由 JSON 渲染，不手工维护。报告必须包含生成器版本、支持矩阵版本、映射库版本与模型快照，保证旧报告可复现。

迁移报告的 JSON 顶层字段固定为：`schema_version`、`run`（时间、工具版本、模型快照）、`project`（源/目标栈、许可证）、`modules[]`（状态、有效代码行、缺口）、`gaps[]`、`manual_tasks[]`、`metrics`、`artifacts[]`、`audit[]`。新增字段只能追加，不改变既有字段语义。

### 5.1 阶段计划与阶段报告

两者均为独立 JSON 文件，进入工作区 `reports/`，并按阶段 id 归档。阶段计划一经批准即冻结，修订产生新版本而非覆盖。

`stage_plan` 顶层字段：`schema_version`、`plan_id`、`revision`、`approved_at`、`approved_by`、`stages[]`。每个 `stages[]` 元素对应 DESIGN §5.5.3 的字段表，另加 `status`（`pending` / `active` / `passed` / `failed` / `rolled_back`）与 `wave_index`。

`stage_report` 顶层字段固定为：

| 字段 | 说明 |
| --- | --- |
| `schema_version` / `stage_id` / `plan_revision` | 归属与版本 |
| `criteria_results[]` | 每条判据的结果、证据与校验器版本 |
| `changes[]` | 文件变更：路径、`added`/`modified`/`deleted`、增删行数、`origin`（`model`/`rule`） |
| `deletions[]` | 删除清单：路径、理由、被替代项，供人工逐条确认 |
| `symbol_map[]` | 符号级映射声明：目标符号、源节点 id、span、置信度 |
| `gaps[]` | 缺口：类别（`capability`/`dependency`/`contract`/`policy`/`scope`）、影响模块、已采取的近似方案 |
| `unverified[]` | 未验证项：原因、影响范围、建议补测方式 |
| `dependency_decisions[]` | 依赖替换或自研实现的决策与许可证影响 |
| `budget` | 本阶段消耗的 token、工具轮数、命令数与人工唤醒次数 |
| `next_prerequisites` | 后续阶段的前置条件是否仍成立 |
| `notes` | 自由叙述，仅作附注，不参与任何判定 |

`criteria_results[]` 与 `changes[]` 是必填项；`gaps[]` 为空数组表示显式声明“无缺口”，与字段缺失含义不同，字段缺失视为未披露。

## 6. 应用配置

- 用户配置使用 TOML，位于数据目录；界面只读写这一份文件。
- API key 不写入 TOML，按 [delivery.md](delivery.md) §2 存放。
- 配置项带默认值与作用域说明；未知字段在读取时告警但不阻断。
- 机器可读的导出（如诊断包）使用 JSON，并在导出前做密钥扫描。

## 7. 迁移 IR 信封（初版）

```json
{
  "schema_version": "1.0",
  "nodes": [
    {
      "id": "java:com.example.OrderService#create",
      "kind": "method",
      "lang": "java",
      "span": { "file": "src/main/java/com/example/OrderService.java", "start": 42, "end": 88 },
      "evidence": [{ "type": "signature", "ref": "OrderService.java:42" }],
      "confidence": "high",
      "attrs": {}
    }
  ],
  "edges": [
    { "from": "java:com.example.OrderController#post", "to": "java:com.example.OrderService#create", "kind": "calls", "confidence": "high" }
  ]
}
```

- `kind` 取值：`package`、`module`、`class`、`interface`、`function`、`method`、`type`、`config`、`route`、`entity`、`template`、`component`、`asset`、`test`。
- `attrs` 保存语言专属信息（修饰符、异常、注解、装饰器、类型参数等），禁止把语言专属语义压平到通用字段。
- 边类型：`calls`、`inherits`、`implements`、`reads`、`writes`、`injects`、`routes`、`renders`、`depends`、`flows`、`tests`。
- 每个节点必须带 `span` 与至少一条 `evidence`；缺失即视为解析失败，不得静默生成。
- 在转写 agent 架构下，IR 的定位是**计划与查询的权威视图**，不是代码生成的输入模板：主 agent 依据它划阶段与依赖闭包，子 agent 通过 `graph.query` 取节点邻域与证据。生成本身由模型读写文件完成，IR 不承载目标代码形状。

## 8. V1-A 冻结项

- 字段命名与枚举值的最终冻结（当前为初版）。
- `stage_plan` 与 `stage_report` 的字段清单与必填项。
- 报告 HTML 模板的章节顺序与中英文案。
- `audit[]` 的完整字段清单。
