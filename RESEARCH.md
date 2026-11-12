# 跨语言代码迁移开源项目调研报告

- 调研日期：2026-11-12
- 调研范围：Java↔Python 转换工具、仓库级代码翻译研究、解析与重构基础设施、评测数据集、成熟跨语言 transpiler、模型候选
- 信息来源：项目官方仓库、官方文档、论文和模型卡
- 维护状态说明：下文“活跃/归档/最后提交”等指调研时观察到的仓库状态，不代表发布质量或生产可用性

## 1. 执行摘要

### 1.1 核心结论

1. **未发现成熟、持续维护、能够完成 Java↔Python 企业全项目双向迁移的开源方案。** 现有项目分别属于函数级研究模型、停滞的语法转换器、运行时互操作或仓库级研究代理。
2. 直接相关的 `java2python` 和 `p2j` 都已长期停滞，只能参考规则、样例和类型推断思路，不能作为现代企业迁移引擎。
3. TransCoder 证明了 Java、Python、C++ 函数级神经翻译的研究价值，但它不理解完整仓库，且许可证含非商业限制。
4. RepoTransBench 最接近真实仓库场景，但论文所报告的最佳总体成功率仅 32.8%，Python→Java 在其研究设置下约为 0.6%–7.0%，说明仓库级迁移仍是研究前沿。
5. j2objc、J2CL、Transcrypt、C2Rust 等成熟或较成熟 transpiler 表明：真正的产品需要**翻译器、目标运行时/兼容层、API 映射、构建集成、测试和明确子集边界**，不是单独一个模型。
6. 本项目合理的技术路线是：**本地语义分析＋规则/模型混合＋版本化依赖映射＋沙箱验证＋人工决策**。不应把通用 LLM 包装成“一键企业迁移器”。

### 1.2 对本项目的直接判断

- 限定参考栈和特性矩阵的辅助迁移具有工程可行性。
- 双向、两套全栈、前端跨语言族和零像素验收会显著增加 V1 风险。
- 任意框架保证不可行；未知框架只能实验处理，经过基准后再提升支持等级。
- “能生成”“能构建”“测试通过”“行为已验证”“可维护”是五个不同层次，不能混用。
- 编译率、BLEU 或 CodeBLEU 不能证明行为等价。

## 2. 调研方法与项目分类

本次将相关项目分为六类：

1. **直接源码转换器**：尝试 Java→Python 或 Python→Java。
2. **神经/LLM 代码翻译研究**：函数或仓库级模型、代理和评测。
3. **运行时互操作**：让 Python 与 Java 在同一进程或运行时协作，不生成独立迁移源码。
4. **解析与重构基础设施**：可成为本项目的本地语义前端。
5. **数据集与基准**：用于训练和执行评测，本身不是迁移器。
6. **其他语言 transpiler**：用于分析成熟产品的架构和长期维护成本。

许可证需分别审查代码、模型权重、训练数据、API 映射和生成输出。仓库代码许可证不自动覆盖模型和数据。

## 3. 直接相关工具与研究

### 3.1 TransCoder

- 仓库：[facebookresearch/TransCoder](https://github.com/facebookresearch/TransCoder)
- 论文：[Unsupervised Translation of Programming Languages](https://arxiv.org/abs/2006.03511)
- 实现：Python/PyTorch。
- 方法：XLM、去噪自编码、回译，支持 C++、Java、Python 两两函数级翻译。
- 许可证：CC BY-NC 4.0，含非商业限制。
- 状态：仓库已归档，调研时代码最后 push 为 2021 年。

**优点**

- 直接覆盖 Java 与 Python，证明无平行语料训练代码翻译的研究路径。
- 提供预处理、训练和执行评测思路。
- 可借鉴其函数执行、错误分类和回译评估方法。

**缺点**

- 主要处理独立函数，不理解仓库结构、构建、配置和第三方依赖。
- 输出正确性依赖测试，不能承诺企业项目行为等价。
- 非商业许可证不适合直接并入商业产品。
- 已归档，不适合作为现代核心引擎。

**可复用点**：错误分类、函数级冒烟基准、执行评测，而非核心代码或生产引擎。

### 3.2 java2python / j2py

- 仓库：[natural/java2python](https://github.com/natural/java2python)
- 实现：Python、ANTLR 3、AST 和规则模板。
- 方向：Java→Python；`j2py` 主要是其命令名，不是另一个成熟独立项目。
- 许可证：LGPL-2.1。
- 状态：默认分支最后提交约为 2016-05。

**优点**

- 规则结构简单直观，适合理解基础语法映射。
- 可配置模板对确定性转换有参考价值。
- 可作为历史错误案例和简单语法样例来源。

**缺点**

- 缺少现代 Java 特性、完整类型求解和项目级语义。
- 官方不保证生成有效 Python。
- 长期停止维护，无法覆盖现代 Spring 项目。
- LGPL 代码不能在不审查义务的情况下直接复制。

**可复用点**：少量规则样例和测试用例；不建议作为运行时依赖。

### 3.3 p2j

- 仓库：[chrishumphreys/p2j](https://github.com/chrishumphreys/p2j)
- 实现：Python，使用 Python AST、规则以及运行时参数跟踪辅助类型推断。
- 方向：Python→Java。
- 许可证：GPL-3.0。
- 状态：最后 push 约为 2013-06。

**优点**

- 认识到 Python→Java 的关键是类型恢复。
- 通过动态执行补充类型证据的思路至今仍有参考价值。
- 适合研究早期源码到源码转换器结构。

**缺点**

- 支持的语法子集小，复杂库和高级特性需要大量人工处理。
- 长期停滞，无法支持现代 Python 和 Java。
- GPL-3.0 对直接复现代码和分发方式有显著约束。

**可复用点**：动态执行与静态证据结合的类型推断思想，不建议直接复用代码。

### 3.4 TransCoder-IR / CodeGen

- 文档：[TransCoder-IR](https://github.com/facebookresearch/CodeGen/blob/main/docs/TransCoder-IR.md)
- 论文：[Code Translation with Compiler Representations](https://ai.meta.com/research/publications/code-translation-with-compiler-representations/)
- 实现：Python 研究代码。
- 方法：将神经翻译与编译器 IR 信号结合。
- 覆盖：C++、Java、Rust、Go，**不包含 Python**。
- 许可证：CodeGen 代码为 MIT，模型和数据需另核。
- 状态：调研时 CodeGen 仓库 2026-03 有 push。

**优点**

- 证明编译器表示能为代码翻译提供额外语义信号。
- 可借鉴“源码之外增加结构化语义特征”的思路。

**缺点**

- 不覆盖 Java↔Python 目标语言对。
- 仍偏研究和函数级任务，不是仓库迁移产品。
- LLVM IR 会丢失或不适合表达部分高级语言、UI 和框架语义。

**可复用点**：IR 作为辅助验证或语义特征，不作为本项目唯一通用中枢。

## 4. 运行时互操作：有用但不是源码迁移

### 4.1 Jython

- 仓库：[jython/jython](https://github.com/jython/jython)
- 实现：Java/Python。
- 许可证：以 PSF-2.0 为主。
- 状态：调研时仍有提交，稳定版为 2.7.4。

**优点**：Python 可在 JVM 上运行并调用 Java，适合混合系统、渐进迁移桥梁和某些差分实验。

**缺点**：稳定版仍是 Python 2.7；CPython C 扩展兼容受限；不会生成可独立维护的 Java 源码。

**对本项目的价值**：过渡互操作或测试辅助，不是迁移引擎。

### 4.2 GraalPy

- 仓库：[oracle/graalpython](https://github.com/oracle/graalpython)
- 官方说明：[Modern Python on JVM](https://www.graalvm.org/latest/reference-manual/python/Modern-Python-on-JVM/)
- 实现：Java/Python/Truffle。
- 许可证：UPL-1.0 及第三方许可。
- 状态：活跃。

**优点**：提供现代 Python 3 运行时、Java 嵌入和双向互操作，比 Jython 更适合现代混合系统。

**缺点**：仍不是源码转换；原生扩展兼容存在限制，并引入特定 GraalVM 运行时。

**对本项目的价值**：渐进迁移、混合运行和互操作验证的可选后端，不应计入“源码已迁移”。

## 5. 本地解析与重构基础设施

| 项目 | 方法与许可 | 优点 | 缺点 | 建议用途 |
| --- | --- | --- | --- | --- |
| [OpenRewrite](https://github.com/openrewrite/rewrite) | Java；类型感知无损 LST、Visitor/Recipe；核心 Apache-2.0，部分解析器/Recipe 为 Moderne Source Available 或商业许可；活跃 | 格式保留、类型信息、批量 Recipe、构建集成成熟 | 不是跨语言翻译；模块许可不统一；复杂 Recipe 有学习成本 | 迁移前后同语言清理、框架升级、审计 diff；逐模块核许可 |
| [Spoon](https://github.com/INRIA/spoon) | Java AST、类型模型、分析和代码生成；CeCILL-C 或 MIT 双许可；活跃 | Java 语义模型和转换 API 强，适合分析和代码生成 | 仅支持 Java；格式保持和无 classpath 场景需要验证 | 复杂 Java 分析和目标 Java 生成的备选组件 |
| [JavaParser](https://github.com/javaparser/javaparser) | Java AST、词法保留、注释、Symbol Solver；Apache-2.0 或 LGPL；活跃 | 轻量、易嵌入、适合符号图和本地解析 | 符号求解依赖正确 classpath；本身不提供跨语言语义 | V1 Java 主分析器，配合类型求解和构建工具 |
| [tree-sitter](https://github.com/tree-sitter/tree-sitter) | Rust 核心/C 运行时，多语言 CST、增量解析、错误恢复；MIT；活跃 | 快速索引、增量处理、多语言和容错能力强 | 没有完整类型和名称解析 | 快速索引、上下文切片、未知框架实验前端，不作唯一语义来源 |
| [LibCST](https://github.com/Instagram/LibCST) | Python 无损 CST、作用域元数据、Matcher/codemod；MIT；调研时 2026-08 有 push | 保留注释、空白和格式，适合安全修改 Python | 类型推断需要外部工具，不能解决跨语言映射 | Python 源/目标代码解析与补丁生成 |

### 5.1 基础设施选择结论

推荐组合：**JavaParser/Symbol Solver＋LibCST＋tree-sitter**。Spoon 用于补充复杂 Java 分析，OpenRewrite 用于迁移前后同语言重构。

不要让 tree-sitter 承担完整类型语义，也不要让 LLM 代替本地解析器完成本可确定性完成的工作。

## 6. 数据集与基准

这些项目可以提供语料、指标或执行环境，但本身不是企业迁移工具。

| 项目 | 覆盖与许可 | 优点 | 缺点与使用建议 |
| --- | --- | --- | --- |
| [CodeXGLUE](https://github.com/microsoft/CodeXGLUE) | CodeTrans 为真实项目挖掘的 Java↔C# 方法对，含 CodeBLEU；代码 MIT；调研时 2024-04 有 push | 评测脚本和代码理解任务有参考价值 | 没有 Java↔Python CodeTrans 翻译集，缺仓库上下文；CodeBLEU 不能证明行为等价 |
| [Project CodeNet](https://github.com/IBM/Project_CodeNet) | 约 1400 万提交、55 种语言、4000 道竞赛题，含 Java/Python 和执行元数据；工具 Apache-2.0，数据条款另核 | 规模大，可构造同题弱配对和执行验证 | 同题提交不是人工对齐翻译；算法代码与企业框架差异大 |
| [AVATAR](https://github.com/wasiahmad/AVATAR) | Java↔Python 平行语料；README 列 9,515 份题解、3,391 个平行独立函数；CC BY-SA 4.0；已归档 | 最直接的 Java/Python 初始执行基准之一 | 以竞赛/算法为主，曾修复换行和缩进数据问题，不能代表企业项目 |
| [CodeTransOcean](https://github.com/WeixiangYAN/CodeTransOcean) | MultilingualTrans、NicheTrans、LLMTrans、DLTrans，含 Java/Python 和执行测试；Apache-2.0；最后 push 约 2023-11 | 多语言、框架和 LLM 翻译压力测试较丰富 | 主要来自 Rosetta Code 等，仍非真实企业仓库 |
| [RepoTransBench](https://github.com/DeepSoftwareAnalytics/RepoTransBench) | 1,897 个真实仓库、13 个语言对、自动测试及 ReAct 代理，含 Python→Java；论文与仓库有更新 | 最接近仓库级目标，能测试代理、构建和自动修复 | 调研时 GitHub 未识别许可证，采用前必须另核；成功率说明任务仍很困难 |

相关论文：[RepoTransBench](https://arxiv.org/abs/2412.17744)、[CodeTransOcean](https://arxiv.org/abs/2310.04951)。

### 6.1 RepoTransBench 的警示意义

论文所报告的研究设置中，最佳总体成功率约为 32.8%，Python→Java 约为 0.6%–7.0%。这些数字不能外推为所有模型和系统的固定上限，但足以说明：

- 仓库级迁移远难于函数级演示。
- Python→Java 需要单独评估，不能假设与 Java→Python 对称。
- 构建、测试和修复代理是基础能力，不是附加功能。
- 本项目必须建立自己的两套全栈、双向基准，并披露完整失败数据。

## 7. 其他 transpiler 的优缺点与类比价值

| 项目 | 方向/方法/许可 | 优点 | 缺点与边界 | 对本项目的启示 |
| --- | --- | --- | --- | --- |
| [j2objc](https://github.com/google/j2objc) | Java→Objective-C；AST/规则＋JRE 模拟运行时＋构建/JUnit；Apache-2.0 为主；活跃 | 生产实践证明翻译器、运行时、构建和测试可形成完整工具链 | 明确只做非 UI 代码；依赖 JRE 模拟和平台边界 | 运行时/兼容层与构建测试是核心产品，范围必须明确 |
| [J2CL](https://github.com/google/j2cl) | Java→Closure JavaScript；语义降级、JRE 子集、Closure 优化；Apache-2.0；活跃 | 系统化语义降级和目标构建集成成熟 | 只支持 Java/JRE 子集，目标生态特定 | 建立明确语言子集、标准库映射和语义降级规则 |
| [JSweet](https://github.com/cincheo/jsweet) | Java→TypeScript/JavaScript；通过 candies 描述目标 API；Apache-2.0；最后 push 约 2023-12 | 版本化 API 映射包思路清晰，支持目标库互操作 | 映射覆盖与维护是长期成本，仓库活跃度需关注 | 依赖映射应独立版本化、可测试、可回滚 |
| [Transcrypt](https://github.com/TranscryptOrg/Transcrypt) | Python 3.9→JavaScript；Python parser/AST＋运行时；Apache-2.0；调研时 2026-06 有 push | 可读输出、source map、目标库互操作和确定性路线成熟 | 必须限制 Python 子集和标准库范围，不能支持任意动态行为 | source map、兼容运行时和子集边界是必要条件 |
| [Sharpen](https://github.com/mono/sharpen) | Java 项目→C#；Eclipse/JDT＋可配置规则；GPL-2.0；已归档 | 历史项目级规则设计有参考价值 | 陈旧、GPL 约束、大量人工映射和维护负担 | 规则系统要版本化，但不能假设规则可覆盖所有语义 |
| [C2Rust](https://github.com/immunant/c2rust) | C99→unsafe Rust；Clang AST＋规则＋后续重构/交叉检查；BSD-3-Clause；活跃 | 强调功能保真、交叉检查和后续重构 | 自动输出不天然安全、地道或可维护 | 采用“先功能保真，再逐步地道化”；生成可运行不是终点 |

### 7.1 成熟 transpiler 的共同规律

1. 都有明确源语言和目标平台边界，而非“任意代码任意转”。
2. 都需要标准库/API 映射和运行时兼容策略。
3. 都深度集成构建和测试。
4. 都存在不支持子集和长期维护成本。
5. 生成代码是否地道、安全和可维护是独立问题。

## 8. 为什么没有成熟的 Java↔Python 企业全项目工具

### 8.1 语法翻译远小于生态迁移

Java 的泛型、重载、异常、反射、动态代理和并发，与 Python 的动态类型、元类、猴子补丁和 C 扩展没有一一对应。Spring、Java EE、Django、FastAPI、React 和 Python UI 框架也没有通用映射。

### 8.2 仓库语义难以恢复

跨文件符号、依赖注入、构建脚本、代码生成、资源文件、数据库、模板和第三方 API 必须共同处理。Python→Java 还需要从调用点和运行时行为恢复静态类型。

### 8.3 数据和指标容易失真

主流平行数据多为竞赛函数或方法对，无法覆盖企业框架和集成行为。BLEU/CodeBLEU 衡量文本相似度，不能证明异常、边界和副作用等价。

### 8.4 正确输出不等于可维护输出

直译式“Java 味 Python”或“Python 味 Java”可能形成新的遗留系统。C2Rust 的 unsafe 输出也说明功能保真与安全、惯用和可维护是不同阶段。

### 8.5 存在更经济的替代方案

对很多系统，运行时互操作、保留服务并增加 API、局部重写或只迁移热点模块，可能比一次性全项目转换更经济。因此，本项目必须披露人工工作量，不能只展示自动生成比例。

## 9. 模型候选调研

### 9.1 使用边界

- 模型是迁移系统的一个组件，不替代解析器、项目图谱、规则和测试。
- 上下文窗口是接口上限，不代表输入整个仓库后仍能可靠迁移。
- 公开编码榜单使用不同工具、脚手架和任务，不等于 Java↔Python 全栈行为保持能力。
- 正式选型必须冻结仓库、工具链、模型快照、提示词、推理档位和尝试预算，使用保留集多次运行。

### 9.2 云端候选

| 模型 | 官方能力与形态 | 优点 | 局限与隐私注意 |
| --- | --- | --- | --- |
| [GPT-6.1 Sol](https://developers.openai.com/api/docs/models/gpt-6.1-sol) | 官方页列 1,050,000 上下文、128K 最大输出，Responses API 和工具调用 | 适合作为复杂编码和智能体任务的默认候选 | 闭源；行为受快照和推理配置影响；Codex 产品窗口不能与 API 数字混写 |
| [GPT-6 Astra](https://developers.openai.com/api/docs/models/gpt-6-astra) | 面向更困难任务的旗舰候选 | 可用于多轮失败和跨模块困难任务升级 | 成本与调用预算需实测，不应作为所有任务默认 |
| [Claude Sonnet 5.5](https://platform.claude.com/docs/en/models/sonnet-5-5) | API ID `claude-sonnet-5-5`，1M 上下文、128K 最大输出，多种云渠道 | 速度和能力平衡，适合默认云端对比 | 无公开权重；不同 Claude 版本存在工具和思考块协议差异，适配层不能假设完全兼容 |
| [Claude Fable 5.2](https://platform.claude.com/docs/zh-CN/models/fable-5-2/overview) | 困难任务升级候选 | 可与 Sonnet 分档路由 | 闭源，保留、区域和 ZDR 需按组织与功能核查 |
| [Gemini 3.8 Flash](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash) | 已 GA；输入 1,048,576、输出 65,536 | 较低时延档、长上下文和工具生态 | 闭源；免费与付费数据政策不能混用 |
| [Gemini 4 Argon](https://ai.google.dev/gemini-api/docs/models/gemini-4-argon) | 已 GA，面向复杂软件工程和多步智能体；输入/思考/输出共享 1,048,576 窗口 | 困难任务升级候选 | 共享窗口需单独预算；长输出不等于无限输入 |
| [qwen3-coder-plus](https://help.aliyun.com/zh/model-studio/qwen3-coder-plus) | Coding Agent API；1,000,000 上下文、最大输出 65,536，支持 Function Calling | 适合工具调用、构建修复和国内云候选 | 无公开权重；需核查地域、日志、工作区和合同配置 |
| [DeepSeek API 模型](https://api-docs.deepseek.com/zh-cn/quick_start/pricing) | 调研时接口表列 V4.1 Flash/V4 Pro、1M 上下文和最大 384K 输出，支持多种协议 | 协议适配方便，可进入主备基准 | 别名和能力应动态发现；训练、保留、区域和 ZDR 条款需合同确认 |
| [GLM-5.3](https://docs.z.ai/guides/llm/glm-5.3) | 1M 上下文、128K 输出，支持 OpenAI、Responses 和 Anthropic 协议 | 便于统一适配层接入 | 云端隐私和本地许可证条件需分别核查 |

### 9.3 本地与开放权重候选

| 模型 | 参数、上下文和许可 | 大小估算/部署 | 优点与局限 |
| --- | --- | --- | --- |
| [Qwen3-Coder-Next](https://huggingface.co/Qwen/Qwen3-Coder-Next) | 80B 总参数、3B 激活、原生 262,144、Apache-2.0 | BF16 理论约 160GB；理想 4-bit 权重约 40GB，实际另需量化元数据、运行时和 KV cache | 本地质量优先候选，强调长时工具使用和执行失败恢复；YaRN 扩展不能写成原生上下文 |
| [gpt-oss](https://openai.com/zh-Hans-CN/index/introducing-gpt-oss/) | 20B/120B，分别约 21B/117B 总参数、3.6B/5.1B 激活，128K，Apache-2.0 | 官方 MXFP4 的 20B 示例约 13.8G；120B 面向约 80GB 内存级，后者不是磁盘承诺 | 20B 可作资源受限候选；必须使用 harmony 格式，质量需项目基准验证 |
| [DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash) | 552B、1M、MIT | BF16 理论约 1.104TB；理想 4-bit 约 276GB | 更像服务器/集群方案，需要兼容专用 prompt encoding/recipe，不适合普通本地工作站 |
| [GLM-5.3](https://huggingface.co/zai-org/GLM-5.3) | 753B、约 40B 激活 | BF16 理论约 1.506TB；理想 4-bit 约 376.5GB | 集群级；许可证含特定大规模 Model-as-a-Service 经营者条件，需法务复核 |
| [GLM-5.3-Flash](https://huggingface.co/zai-org/GLM-5.3-Flash) | 320B、18B 激活、1M、MIT | BF16 理论约 640GB；理想 4-bit 约 160GB | 比旗舰小，但仍不是普通单机轻量部署 |

上述理论大小按十进制单位计算：BF16=`总参数量(B)×2GB`，理想 4-bit=`总参数量(B)×0.5GB`。它们不含量化元数据、KV cache、运行时、临时文件和容器镜像，也不等于可运行显存。

### 9.4 云端数据政策参考

- [OpenAI 商业数据说明](https://openai.com/business-data/)：商业 API 默认不使用输入输出训练；保留期和 ZDR 取决于组织资格及配置。
- [Gemini API 条款](https://ai.google.dev/gemini-api/terms)：付费 API 与免费层的数据使用和保留政策不同，安全、反滥用及法律义务日志仍可能存在。
- 其他 Provider 也应逐项核查训练、日志、地域、保留、ZDR 和 DPA，不能用“企业 API”四个字代替合同审查。

### 9.5 推荐选型路径

1. 默认档让 GPT-6.1 Sol、Claude Sonnet 5.5、Gemini 3.8 Flash、qwen3-coder-plus 等进入同一基准，选主备各一家。
2. 困难任务升级到 GPT-6 Astra、Claude Fable 5.2、Gemini 4 Argon 或提高同模型推理档位，并设置尝试预算。
3. 本地优先实测 Qwen3-Coder-Next；资源受限时比较 gpt-oss-20b 和较小 Qwen Coder。
4. DeepSeek、GLM 大模型只在具备服务器集群时考虑本地部署。
5. 最终按构建通过率、行为已验证率、人工修改量、修复轮数、时延和 token 成本定版，而不是按公开榜单名次。

## 10. 对本项目架构与路线的启示

1. **定位为人机协同迁移工作台。** 不承诺一键双向任意项目。
2. **采用双原生前端和统一项目图谱。** JavaParser/Spoon 处理 Java 语义，LibCST 修改 Python，tree-sitter 负责索引。
3. **IR 保留语言专属信息和未知语义。** 不强行压入 LLVM IR 或最低公分母模型。
4. **规则优先、模型可插拔。** 确定性规则处理结构、类型和构建；模型生成惯用表达、映射候选和修复建议。
5. **版本化 API 映射和兼容运行时是核心产品。** 需要记录版本、证据、许可证和测试。
6. **验证闭环先于规模化生成。** 解析→构建/类型检查→单元/集成→差分→UI/E2E→最小修复。
7. **围绕依赖、diff 和审核任务设计 UI。** 支持模块级迁移、接受/拒绝、置信度、审计、回滚和幂等。
8. **建立企业私有基准。** AVATAR 等只做冒烟测试；真实验收使用两套参考栈和外部保留项目。
9. **许可证治理从第一天开始。** 模型、数据、映射、运行时和生成代码分别记录。
10. **公开人工成本。** 如果目标代码仍需大量重写，必须如实反映，不以“代码生成率”代替迁移价值。

## 11. 建议的基准组合

| 层级 | 建议来源 | 用途 | 不能证明的内容 |
| --- | --- | --- | --- |
| 函数冒烟 | AVATAR、Project CodeNet、CodeTransOcean 的许可合规子集 | 基础语法、执行和错误分类 | 企业项目级能力 |
| 仓库研究对照 | RepoTransBench 的可合法使用子集 | 对比仓库代理、构建和修复能力 | 本项目两套全栈保证 |
| 自有符合性套件 | 针对 IR、类型、异常、依赖、ORM、模板、UI 组件设计 | 验证每个支持矩阵声明 | 未覆盖真实项目表现 |
| 官方金样 | 两套参考栈×两个方向 | V1 发布阻断测试 | 外部项目成功率 |
| 外部保留项目 | 至少 8 个未参与调参的项目 | 披露真实构建率、已验证率、人工量和成本 | 任意框架普遍保证 |

模型和系统对比时，应固定工具权限和尝试预算，多次运行并保留失败结果。测试不足的项目只能标记“迁移完成”，不得并入“行为已验证率”。

## 12. 最终调研结论

本项目不是空白赛道中的简单工程复刻，而是位于**代码语义分析、 transpiler、依赖生态迁移、仓库级智能体和自动化验证**交叉处的高风险研发项目。

现有开源项目足以提供解析器、重构框架、基准和架构经验，但没有一个可以直接解决目标问题。最合理的路线不是寻找“更强的单一模型”，而是建设一个可审计的工程系统：

> 原生源码理解 → 项目语义图 → 版本化映射 → 规则/模型混合生成 → 沙箱构建与测试 → 差分和 UI 验证 → 人工审核 → 可追踪交付。

只要坚持支持矩阵、验证分级和失败披露，限定范围内的产品值得立项；如果恢复“任意框架保证、无测试也算行为等价、模型可擅自改契约”的承诺，则项目不可行。
