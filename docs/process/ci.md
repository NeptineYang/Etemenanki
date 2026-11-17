# CI 与仓库自动化

- 状态：V1 设计基线
- 日期：2026-11-17
- 依据：Q8、Q27、Q44、Q61–Q62
- 上位文档：[DESIGN.md](../../DESIGN.md) §12.3

## 1. 现在就有

仓库还没有代码，先落地四类不依赖构建的自动化：

| 项 | 文件 | 作用 |
| --- | --- | --- |
| PR 规范检查 | `.github/workflows/pr-title.yml` | 标题必须匹配 `类型: 描述`，类型限于 `feat`、`fix`、`refactor`、`chore`、`test`、`docs`、`build`、`data` |
| 安全基线 | `.github/workflows/security.yml` | 对 push 与 PR 运行密钥扫描；同时开启 GitHub 原生 secret scanning 与 push protection |
| 协作模板 | `.github/pull_request_template.md`、`.github/ISSUE_TEMPLATE/`、`.github/SECURITY.md` | 统一 PR 正文、Issue 分类与漏洞上报入口 |
| 文档链接检查 | `.github/workflows/docs-links.yml` | 检查 Markdown 中的相对链接与锚点，防止 `RESEARCH.MD` 这类坏链再次出现 |

各组件的 CI 随该组件第一个 PR 一起加入，不预先搭空流水线。CodeQL、许可证检查、SBOM 与发版流水线等有实际代码后再加。

### 1.1 仓库规则

- `main` 必须经 PR 合入，至少 1 个 Approve；只允许 Squash merge。
- 必需检查：PR 标题、文档链接、密钥扫描；组件 CI 建立后逐项追加。
- 保护 `v*` 标签：禁止删除与强制移动。
- 暂不启用“必须由 code owner 审核”。CODEOWNERS 只登记维护者，等有第二位有写权限的成员再开启。
- 开启 Discussions 与私密漏洞报告，后者用于 SECURITY.md 的报送渠道。
- 上述规则在 GitHub 仓库设置中配置，不在版本控制内；变更需在 PR 或 Issue 中记录原因。

## 2. 组件 CI

| 组件 | 位置 | 检查内容 |
| --- | --- | --- |
| Rust Core | `crates/` | `cargo fmt --check`、`cargo clippy -D warnings`、`cargo test` |
| Java Worker | `workers/java/` | Maven 构建与单测，基于 Temurin 生成 jlink 精简 JRE |
| Python Worker | `workers/python/` | uv 同步、ruff、pytest |
| Web | `web/` | pnpm 安装、类型检查、构建 |
| Tauri 应用 | `apps/desktop/` | `pnpm tauri build` 产物可生成，NSIS 打包通过 |
| runner | `runner/` | Rust 构建、协议契约测试、bubblewrap 与断网行为测试 |
| 环境包 | `packaging/wsl/` | rootfs 可复现构建、`.wsl` 包可安装、体积记录 |

## 3. 执行环境拆分

GitHub 托管的 Windows runner 不支持 WSL2（runner-images #10563 已以 not_planned 关闭，`wsl --update` 变通常因 403 失败），因此：

- **Windows runner**：Core、Java Worker、Python Worker、Web、Tauri 打包、界面关键流程。
- **Ubuntu runner**：runner、环境包构建、bubblewrap 沙箱、断网与白名单行为、Linux 侧契约测试。
- **不在 CI 里做**：Windows 与 WSL2 的完整联调，改由手工清单覆盖（见 §4）。

## 4. 手工联调

- 触发条件：每个 X.Y.0 的 PR；触及 runner 或桥接层的 patch PR。
- 时机：PR 合并前完成，结果附在该 PR。合并即正式发布，没有发布后补验的窗口。
- 内容：安装包安装与卸载、便携版解压即用、环境包导入、`wsl.exe` 拉起 runner、受控环境截图、断网构建、密钥与数据位置、Tauri 配置快照（CSP、capability、禁远程内容、devtools 关闭）。
- 清单随版本演进，模板放在 `docs/process/` 下，与本文同目录维护。

## 5. 工具链固定

- Rust：`rust-toolchain.toml` 固定 channel 与组件。
- Node：`package.json` 的 `packageManager` 字段固定 pnpm 版本。
- Python：`.python-version` 固定 3.12 小版本。
- Java：Maven Wrapper + 固定 JDK 发行版与版本。
- 所有固定值同时写入环境包定义，保证本机与沙箱一致。

## 6. 权限与供应链

- 工作流默认 `permissions: read-all`，按需最小授权。
- 第三方 Action 应固定到提交 SHA。当前三个工作流使用版本标签并留有 TODO，在首个代码 PR 中一并固定。
- 为 Actions 开启 Dependabot，定期升级固定版本。
- 发布物附构建来源证明，用户可用 `gh attestation verify` 核对。

## 7. 待定事项

- [待定] 密钥扫描工具选型：GitHub 原生、gitleaks（需核查许可证）或两者并用。
- [待定] 文档链接检查工具（lychee 或其他）与是否检查外链。
- [待定] 环境包构建是否完全在 CI 内完成，还是允许本地可复现脚本。
- [待定] X.Y.0 手工联调清单的正式模板内容。
