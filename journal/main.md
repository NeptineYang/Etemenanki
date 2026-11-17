# main 分支工作日志

本文件记录直接作用于 `main` 的会话与决策。格式与读写规则见 [JOURNAL.md](../JOURNAL.md)。

## 2026-11-16 设计拷问（Q1–Q49）

- 工具/模型：Qoder Desktop（模型未记录）。分支 `main`，只讨论设计，无代码改动。
- 共五轮：文档与仓库、无需实测的已知缺陷、Actions 与发布、大版本后的 GEO、版本标签机制。

### 决策索引

完整结论已写入 `DESIGN.md` V1.1 与 `docs/`，此处只保留索引，供文档中的 Q 编号回溯。

| 编号 | 结论 |
| --- | --- |
| Q1 | `DESIGN.md` 为总纲，细节拆到 `docs/`；升版并修 `RESEARCH.MD` 坏链 |
| Q2 | 单仓库，布局见 AGENTS.md §1 |
| Q3 | 原生 Core + 容器 Worker（已被 Q33 覆盖） |
| Q4 | 已知缺陷设计四项全做：安全、协议与状态、迁移语义、格式与指标 |
| Q5–Q7 | 版本 `x.y.z` 全仓统一；每个 PR 升版；不兼容变更升中间位 |
| Q8 | 先加 PR 规范、安全基线、协作模板、文档链接检查 |
| Q9 | LICENSE 恢复 Apache 原文，署名移入 NOTICE |
| Q10 | GEO 每个 `X.0.0` 后做，首次 1.0.0，中英双语 |
| Q11 | JOURNAL 是跨会话工作日志 |
| Q13–Q15 | 合并后自动打 `vX.Y.Z`；每版全量发布；0.1.0 起步，1.0.0 手动 |
| Q16 | 只走 GitHub Releases（GHCR 已随 Q17 取消） |
| Q17、Q34、Q35 | 只发 Windows 11 x64；Tauri 2 窗口；安装包 + 便携 zip，不要 Docker |
| Q18 | 容器运行时问题作废 |
| Q19 | 依赖获取严格两阶段，默认不执行项目脚本 |
| Q20、Q43 | 只检查更新，默认开、可关，不收集数据 |
| Q21 | Worker 协议、IR、报告以 JSON Schema 为准 |
| Q22 | 能证明无差异才用地道写法，否则保真实现并标注 |
| Q23 | 模块级判定、项目取最差；两种“行为已验证率”口径 |
| Q24、Q38 | 阻断 Java<17 与 Spring Boot 2，其余非参考版本走实验 |
| Q25 | 自身格式用 TOML/JSON；“配置转写”指迁移源项目配置 |
| Q26 | 界面、报告、教程中英双语；CLI 与日志英文 |
| Q27 | 工作台用 pnpm |
| Q28 | 加 `.gitattributes`，统一 LF |
| Q29 | JOURNAL 按分支分文件 |
| Q30 | CODEOWNERS 只写维护者，不开启强制 code owner 审核 |
| Q31 | README 等 1.0.0 再写 |
| Q32、Q39、Q40 | WSL2 环境包 + runner；`wsl.exe` stdio 不开端口；基础包 + 按需下载 |
| Q33 | Worker 在 Windows 本机运行，随包精简 JRE 与 embeddable Python |
| Q36 | 1.0 前不签名；试点后申请 SignPath |
| Q37 | 国内下载渠道 1.0.0 时评估 |
| Q41 | V1 不提供本地 HTTP API |
| Q42 | 默认官方源，可换镜像 |
| Q44 | Windows/Ubuntu runner 拆开测，完整联调手工做 |
| Q45 | OpenAI 兼容 + Anthropic + Gemini |
| Q46 | 代理跟随系统，可手动填写 |
| Q47 | Windows 侧只下载、不执行依赖 |
| Q48 | 内置 delombok，注解处理器产物从基线构建取回 |
| Q49 | Gradle 联网阶段可执行项目脚本（已批准例外） |

### 已核实事实（细节已迁入文档）

- GitHub 托管的 Windows runner 不支持 WSL2（runner-images #10563，not_planned）。
- `.wsl` 格式、rootfs 打包与 systemd 屏蔽要求见 [delivery.md](../docs/design/delivery.md) §6。
- `GITHUB_TOKEN` 创建的标签不触发其他工作流，见 [versioning.md](../docs/process/versioning.md) §3。
- SignPath Foundation 的条件与署名要求见 [delivery.md](../docs/design/delivery.md) §8。
- 未签名 exe 会触发 SmartScreen；智能应用控制可能直接阻止运行。

## 2026-11-17 设计文档落地（Q50–Q65）

- 工具/模型：WorkBuddy Desktop（模型未记录）。分支 `docs/design-baseline`。

### 做了什么

1. 复核 Q1–Q49，识别被后位决策覆盖的项，记入 `DESIGN.md` 附录 A。
2. 追加 Q50–Q69：环境包内容寻址独立版本线；安装器 `currentUser`，路径定制交给便携版（修正 Q17）；数据与密钥按形态区分；校验后由程序启动安装器；Debian 13 slim；runner 用 Rust 共用协议 crate；白名单代理在 runner 内；bubblewrap + cgroup v2；工作区在 WSL 内、stdio 传 tar；无 WSL2 时 Gradle 阻断迁移保留扫描；合并即正式发布；联调移到 PR 前；1.0 前不回溯；工具链下载双路径；Tauri 安全四项全做；CLI 可选加入 PATH（Q66）；卸载默认删除数据与发行版（Q67）；生成风格按语义分型（Q68）；GEO 用 GitHub Pages（Q69）。
3. 把各文档的 `[待定]` 全部替换为可执行规格：协议超时与并发、SQLite 迁移与回放缓存、Maven 与 Gradle 约束、私有仓库凭据、置信度三级、生成风格、IR 信封、报告字段、source map 关联、模块边界、试点规则、发布工作流结构、手工联调 13 项清单、`llms.txt` 模板。仅保留 V1-A 实测项与 1.0 决策项。
3. `DESIGN.md` 升 V1.1：§0 文档索引、15 处章节修订、附录 A。
4. 新建 `docs/design/` 六篇、`docs/process/` 三篇；`AGENTS.md` 校正为当前形态。
5. `LICENSE` 恢复 Apache-2.0 原文并新建 `NOTICE`；新增 `.gitattributes`、`CODEOWNERS`、PR/Issue 模板、`SECURITY.md` 与三个工作流。
6. JOURNAL 拆分：本文件承接记录，`JOURNAL.md` 只保留规则与索引。

### 验证结果

- 项目内相对链接 33/33 有效；`DESIGN.md` 无 Axum、本地 API、容器沙箱残留。
- 仓库无代码，未运行构建与测试；YAML 未做机器校验。

### 未完成项

- 各文档的开放项只剩 V1-A 实测值与 1.0 决策项（自定义域名、对比页对象、`release/1.x`）。
- `.gitignore` 的 `env/`、`out/`、`build/`、`dist/` 未锚定。
- 仓库设置需在 GitHub 侧开启：分支保护与必需检查、secret scanning 与 push protection、Discussions。

### 下一步

1. 提交并推送 `docs/design-baseline`，开 PR 评审。
2. 修正 `.gitignore` 未锚定规则。
3. V1-A 阶段用实测数据回填各文档的校准项。
