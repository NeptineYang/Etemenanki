# 交付形态与分发

- 状态：V1 设计基线
- 日期：2026-11-18
- 依据：DESIGN.md §9、Q17、Q20、Q32、Q34–Q37、Q39、Q40、Q43、Q50–Q55、Q61–Q64、Q71
- 上位文档：[DESIGN.md](../../DESIGN.md)

## 1. 平台与形态

V1 只发布 Windows 11 x64，不发布 Linux 与 macOS 桌面版，也不要求用户安装 Docker/Podman。

| 形态 | 产物 | 说明 |
| --- | --- | --- |
| 安装版 | NSIS `-setup.exe`，`installMode=currentUser` | 无需管理员权限，固定安装到 `%LOCALAPPDATA%` 下（具体子目录在 V1-A 实测确认） |
| 便携版 | zip | 解压即用，数据写在 exe 旁 `data\`，可放在任意可写目录 |
| CLI | 独立 exe | 直接链接 Core 库，与界面共享同一套配置与数据库 |
| 沙箱环境 | `.wsl` 环境包 | 转写必需；未安装时只能扫描与分析，见 §3 |

界面为 Tauri 2 桌面窗口，使用系统 WebView2。安装包不提供路径选择：安装目录固定，需要自定义位置时使用便携版。这一口径取代早期“安装到指定位置”的表述。

- WebView2 使用 `embedBootstrapper`（约 1.8 MB）：Win11 自带时跳过，缺失时由安装包内置的引导器联网安装；全程离线安装不在 V1 范围。
- CLI 默认加入用户级 `PATH`，安装界面提供可取消的勾选项并说明改动；卸载时移除该项。便携版不修改 `PATH`，由用户在需要时自行配置。

## 2. 目录、数据与凭据

| 项目 | 安装版 | 便携版 |
| --- | --- | --- |
| 程序目录 | `%LOCALAPPDATA%` 下的固定子目录（V1-A 实测确认） | 解压目录 |
| 数据目录 | `%LOCALAPPDATA%\Etemenanki` | `<exe 所在目录>\data` |
| API key | Windows 凭据管理器（`keyring` crate） | DPAPI 加密文件，位于 `data\` |
| 升级 | 保留数据目录 | 保留 `data\`；替换程序文件 |

便携版的 DPAPI 加密绑定当前 Windows 用户账户：目录拷到其他机器或其他用户下无法解密，教程中明确说明。

卸载时提供两个独立勾选项：删除应用数据、注销 WSL2 发行版，默认都勾选。勾选时显示不可逆警告，并列出将删除的具体内容（工作区、报告、凭据条目、发行版及其缓存）；取消勾选则保留对应内容。若曾把 CLI 加入 `PATH`，卸载时移除用户级条目。此项满足 SignPath 对“系统改动需可卸载”的要求。

## 3. 首次使用

1. 启动 exe，界面默认跟随系统语言，可在设置页切换。
2. 选择模型接口（OpenAI 兼容 / Anthropic / Gemini）并填写 base URL、key 与模型名。
3. 按教程安装 WSL2 环境包。这是转写的必要步骤：文件读写、命令执行、构建与测试都在发行版内执行，未安装时只能导入、扫描与分析，不能进入阶段计划与生成。
4. 导入项目，源项目以只读方式导入。

## 4. 更新

- 只检查更新，不收集任何使用数据；bug 与问题依靠社区反馈。
- 检查默认开启，可在设置页关闭；每次启动最多检查一次，只向 GitHub 请求版本信息，除连接本身携带的 IP 外不发送任何内容。
- 0.x 版本全部标记为 pre-release，因此更新检查使用 releases 列表接口，不使用 `/releases/latest`。
- 检查失败时显示“检查失败”，不影响正常使用。
- 新版本流程：用户确认下载 → 下载到临时目录 → 校验 SHA256 → 校验通过后由程序启动安装器，用户仍需在安装界面确认。不做静默更新。
- 便携版：提示用户下载 zip 并手动解压，保留原 `data\` 目录。
- 程序记录安装位置，更新后沿用原安装路径。

## 5. 发布物清单

每个版本都在 GitHub Releases 发布完整清单，不做增量发布：

| 文件 | 说明 |
| --- | --- |
| `Etemenanki_<version>_x64-setup.exe` | NSIS 安装包 |
| `Etemenanki_<version>_x64-portable.zip` | 便携版 |
| `wsl-env-<content-hash>.wsl` | 环境包，内容寻址、独立版本线 |
| `SHA256SUMS` | 全部文件的校验和 |
| GitHub 构建来源证明 | 由 CI 生成，用户可用 `gh attestation verify` 核对 |

环境包按内容寻址：只有基础系统、runner 或预置配置变化时才重建，文件名随内容变化；未变化时该版本 Release 直接引用同一个不可变文件。这样既满足“每版全量列出”，又不必每次重打包几百 MB。

GitHub Releases 的约束：每个 release 最多 1000 个文件，单个文件小于 2 GiB，无总大小与带宽限制。当前发布物远低于这两个上限，不需要拆分。

Release 说明取该版本对应 PR 的标题与链接，不维护 `CHANGELOG.md`，避免每次发版都要改 `main`。

## 6. WSL2 环境包

- 基础系统：Debian 13 slim，规避再分发修改版 Ubuntu rootfs 的商标限制。
- 内容：精简 rootfs + Rust runner，不含 JDK、Maven、Node 与 Playwright 浏览器。
- 构建：CI 用 mmdebstrap 从 Debian 官方镜像源构建，不依赖手工环境；产物目标 `≤ 450 MB`（gzip 后的 `.wsl`），超出时先削减文档、locale 与缓存，不裁剪工具链。
- 安装：双击安装，或在 PowerShell 执行 `wsl --install --from-file <file>`；发行版默认名称在 `oobe.defaultName` 中固定。
- 配置：`/etc/wsl.conf` 设置 `[automount] enabled=false`、`[interop] enabled=false`、`appendWindowsPath=false`。
- 工具链：首次使用时按需下载，版本与 SHA256 由 Core 下发的清单固定，缓存在发行版内。下载优先由 runner 在 WSL 内直连（走代理与镜像）；直连失败时回退为 Windows 侧下载后推送进发行版。
- 升级：应用升级时自动推送匹配版本的 runner；只有基础系统变化才需要用户重新导入环境包。

打包约束（来自微软的 `.wsl` 规范）：需要 WSL 2.4.4 或更高版本；产物是根文件系统 tar（建议 gzip 压缩）改名为 `.wsl`；`/etc/wsl.conf` 与 `/etc/wsl-distribution.conf` 属主为 root:root、权限 0644；不包含 `/etc/resolv.conf`、内核与 initramfs；`/etc/shadow` 中不保留密码哈希；启用 systemd 时屏蔽 systemd-resolved、systemd-networkd、NetworkManager 与 tmp.mount。

## 7. 语言

- 界面、HTML 报告、用户教程（含 WSL2 教程）从第一版起中英双语，默认跟随系统语言，可在设置页切换。
- CLI 输出与运行日志统一使用英文，便于粘贴到 Issue 与脚本处理。

## 8. 代码签名与分发渠道

- 1.0.0 之前不做代码签名。教程说明 SmartScreen 的放行步骤，并提供 SHA256 核对方法。
- 外部试点之后申请 SignPath Foundation：免费、面向 OSI 许可证且无商业双重许可的项目，要求安装时展示隐私说明并提供关闭选项。
- 证书由 SignPath Foundation 署名，只能签本仓库产出的二进制；每次签名发布需人工批准；主页需链接代码签名政策，并声明“Free code signing provided by SignPath.io, certificate by SignPath Foundation”。
- 1.0.0 之前只通过 GitHub Releases 分发。国内下载渠道在 1.0.0 与 GEO 一并评估；Gitee 附件上限放不下安装包，不作为候选。
- GHCR 不再使用：项目不产出容器镜像。

## 9. V1-A 实测确认项

以下数值无法在设计阶段确定，由首个垂直切片记录并回填本文：

- 安装目录在 `%LOCALAPPDATA%` 下的确切子目录，以及 NSIS 实际默认值。
- 环境包压缩后的实际体积与解压占用。
- 目标机器上 WebView2 的缺失比例，用于校准安装提示文案。
- 便携版在中文路径、长路径、只读目录下的实际行为。
