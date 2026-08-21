# Codex CLI（codex-rs）三维度深度解读实施计划

> 本计划基于 read-code 技能的"三观递进法"（整体观 → 具体观 → 深刻不忘观），对 /workspace 下的 OpenAI Codex CLI monorepo 产出三篇长篇深度解读文档，全部保存至 `/workspace/ReadCode/` 文件夹。

---

## 一、任务概要（Summary）

对 OpenAI Codex CLI 仓库（以 codex-rs Rust workspace 为主体）进行全方位深度解读，产出三篇层层递进的中文长篇文档：

| 序号 | 文件名 | 认知目标 | 核心维度 |
|------|--------|----------|----------|
| 1 | `ReadCode/01-整体观-Codex全貌解读.md` | 看见（建立认知地图） | 项目定位/生态定位、代码结构、设计理念、原理概述、端到端流程、使用指南与案例 |
| 2 | `ReadCode/02-具体观-算法与实现剖析.md` | 看懂（深入技术细节） | Agent 循环、流式处理、上下文工程、apply_patch 算法、沙箱安全、网络代理、MCP、极致工程——每点配论文原理 + 代码实现 + 具体例子 |
| 3 | `ReadCode/03-深刻观-哲学与升华.md` | 看透（升华本质洞见） | 设计哲学、框架思想、算法框架之思、核心主题升华，最终凝练为两三句话 |

**硬性要求**：
- 每篇采用"总分总"结构，篇内章节层层递进
- 每篇至少 3 张 Mermaid 图表，每图必须配图注说明
- 每篇包含使用指南与相关案例
- 第二篇每个核心技术点至少引用 1 篇学术论文（格式：`[论文标题](URL) — 核心贡献一句话概括`）
- 第三篇以两三句话终极概括收尾
- 所有 Mermaid 图表必须逐一校验语法并修正
- 只新建 `ReadCode/` 下 3 个 md 文件，不修改任何源代码

---

## 二、现状分析（Phase 1 探索结论）

### 2.1 项目身份

- **定位**（[README.md](file:///workspace/README.md) L1-L8）：Codex CLI 是 OpenAI 的**本地运行编码 Agent**，与 IDE 扩展、桌面 App（`codex app`）、云端 Codex Web（chatgpt.com/codex）构成 Codex 产品家族；本仓库是其开源核心实现。
- **技术栈**：Rust workspace（`codex-rs/`，约 100 个 crate）+ Node.js 分发壳（`codex-cli/`）+ Python/TypeScript SDK（`sdk/`）+ Bazel/Cargo 双构建体系。

### 2.2 关键架构事实（已核实，含文件锚点）

**入口与分层**：
- CLI 多路分发入口：[cli/src/main.rs](file:///workspace/codex-rs/cli/src/main.rs)（`MultitoolCli`：无子命令→交互 TUI；`Exec`/`Login`/`Mcp`/`McpServer`/`AppServer` 等子命令）
- TUI 入口：[tui/src/main.rs](file:///workspace/codex-rs/tui/src/main.rs) → `codex_tui::run_main`
- `cli` crate 依赖 `codex-core`、`codex-tui`、`codex-exec`、`codex-mcp-server`、`codex-app-server`、`codex-protocol` 等（[cli/Cargo.toml](file:///workspace/codex-rs/cli/Cargo.toml) L25-L72）

**核心代理循环**（事件驱动 Actor 模型）：
- [core/src/codex_thread.rs](file:///workspace/codex-rs/core/src/codex_thread.rs) L145-L210：`CodexThread` 封装 `Session`/`SessionIo`，`submit(Op)` 委托给 `io.submit(op)`
- [core/src/session/mod.rs](file:///workspace/codex-rs/core/src/session/mod.rs) L781-L805：`SessionIo::submit()` 将 `Op` 包成 `Submission` 写入 `tx_sub`（tokio mpsc 通道）
- [core/src/session/handlers.rs](file:///workspace/codex-rs/core/src/session/handlers.rs) L515-L680：**submission_loop**——主调度器，分派 `Interrupt`/`TurnInput`/`RecoverTurn`/`Compact`/`Shutdown`
- [core/src/session/turn_input.rs](file:///workspace/codex-rs/core/src/session/turn_input.rs) L141-L249：处理 `TurnInput`，创建/复用 `TurnContext`，`spawn_task` 启动模型任务
- [core/src/client.rs](file:///workspace/codex-rs/core/src/client.rs) L1870-L2113：模型调用分发（WebSocket Responses API，可回退 HTTP SSE）；`map_response_events` 处理流式事件
- [core/src/stream_events_utils.rs](file:///workspace/codex-rs/core/src/stream_events_utils.rs) L288-L390：识别 tool call / 普通消息 / fatal error，tool call 进入执行队列
- [protocol/src/protocol.rs](file:///workspace/codex-rs/protocol/src/protocol.rs) L539-L575（`Op` 枚举）、L1282-L1341（`EventMsg`：`TurnStarted`/`TurnComplete`/`TurnAborted`/`TokenCount`/`AgentMessage` 等）

**上下文工程**：
- [core/src/tasks/compact.rs](file:///workspace/codex-rs/core/src/tasks/compact.rs) L28-L85：按 `TokenBudget` 选择手动/远程/本地压缩
- [core/src/compact_token_budget.rs](file:///workspace/codex-rs/core/src/compact_token_budget.rs) L66-L93：pre/post compact hooks、`ContextCompaction` turn item、`start_new_context_window`
- [core/src/context_manager/history.rs](file:///workspace/codex-rs/core/src/context_manager/history.rs) L155-L213：历史规范化与 `ResponseItem` 生成

**工具系统**：
- [core/src/tools/handlers/unified_exec.rs](file:///workspace/codex-rs/core/src/tools/handlers/unified_exec.rs) L97-L142、[shell_spec.rs](file:///workspace/codex-rs/core/src/tools/handlers/shell_spec.rs) L21-L110（`exec_command` 参数：cmd/workdir/tty/yield_time_ms/max_output_tokens 等）
- [core/src/tools/handlers/apply_patch.rs](file:///workspace/codex-rs/core/src/tools/handlers/apply_patch.rs) L72-L89：`StreamingPatchParser` 解析补丁
- [apply-patch/src/parser.rs](file:///workspace/codex-rs/apply-patch/src/parser.rs) L145-L210：`*** Begin Patch`/`*** End Patch` 标记解析 → `ApplyPatchArgs`
- [apply-patch/src/file_update.rs](file:///workspace/codex-rs/apply-patch/src/file_update.rs) L24-L82：`compute_replacements` 生成新文件内容（换行标准化）

**沙箱安全**：
- [sandboxing/src/manager.rs](file:///workspace/codex-rs/sandboxing/src/manager.rs) L36-L76：三平台沙箱选择（macOS Seatbelt / Linux seccomp+Landlock+bubblewrap / Windows restricted token）；L78-L126：`SandboxCommand`/`SandboxExecRequest` 边界参数
- [core/README.md](file:///workspace/codex-rs/core/README.md)：三平台沙箱细节（Seatbelt profile、bwrap 优先选取、WSL1 不支持、Windows split filesystem policy）
- [execpolicy](file:///workspace/codex-rs/execpolicy/src/)：WASM（OPA Rego）命令审批
- [network-proxy/src/mitm.rs](file:///workspace/codex-rs/network-proxy/src/mitm.rs)：MITM 代理拦截过滤网络请求（含 socks5、windows_tcp_attribution 等）

**提示词工程**：
- [core/gpt-5.2-codex_prompt.md](file:///workspace/codex-rs/core/gpt-5.2-codex_prompt.md)、[gpt_5_codex_prompt.md](file:///workspace/codex-rs/core/gpt_5_codex_prompt.md) 等：Agent 身份、编辑约束、git 工作区行为、plan 工具、review 思维、输出格式
- [core/src/realtime_prompt.rs](file:///workspace/codex-rs/core/src/realtime_prompt.rs) L5-L24：prompt 组装优先级管线

**MCP 集成**：
- [core/src/session/mcp.rs](file:///workspace/codex-rs/core/src/session/mcp.rs) L91-L149：会话级 MCP 配置构建
- [codex-mcp/src/connection_manager.rs](file:///workspace/codex-rs/codex-mcp/src/connection_manager.rs) L79-L129：`McpConnectionSet` 连接生命周期（复用/启动/关闭）

**会话持久化**：
- [rollout/src/recorder.rs](file:///workspace/codex-rs/rollout/src/recorder.rs) L77-L122：JSONL 格式，`rollout-<timestamp>-<conversation_id>.jsonl`
- [rollout/src/reverse_jsonl_scanner.rs](file:///workspace/codex-rs/rollout/src/reverse_jsonl_scanner.rs) L19-L43：文件末尾反向扫描

**极致工程**：
- Bazel（MODULE.bazel）+ Cargo 双构建；justfile 主命令（`just codex`/`just test`/`just fmt`）
- insta 快照测试（tui）；core/suite 集成测试（`test_codex` mock Responses API）
- [tui/src/chatwidget.rs](file:///workspace/codex-rs/tui/src/chatwidget.rs)：ratatui 事件消费/历史 cell/transcript overlay 架构

### 2.3 探索结论

信息已足够支撑三篇文档写作，无需向用户追加澄清（任务目标、输出位置、结构、风格、篇幅均已明确）。

---

## 三、产出物详细规划（Phase 3 核心）

### 3.1 文档一：`ReadCode/01-整体观-Codex全貌解读.md`

**篇幅目标**：约 6000–9000 字 + 5 张 Mermaid 图

**结构（总分总 + 层层递进）**：

```
一、总起：一句定位与三个关键事实
   —— Codex CLI 是 OpenAI 官方开源的本地编码 Agent；Rust 单内核多前端；
      安全沙箱纵深防御；事件驱动内核
二、分述
  第1章 生态定位：Codex 家族全景（CLI / IDE / Desktop / Web / Cloud）
         —— 本仓库在家族中的角色：一切形态共享的核心引擎
         【图1】Codex 产品生态图（flowchart：家族成员 + 本仓库位置）
  第2章 仓库宏观结构：monorepo 五大板块
         （codex-rs / codex-cli / sdk / docs / .github+bazel）
         【图2】monorepo 目录结构图（mindmap）
  第3章 codex-rs workspace 解剖：约百个 crate 的分层
         入口层（cli/tui/app-server/exec/mcp-server）
         → 内核层（core/core-api/protocol）
         → 能力层（apply-patch/sandboxing/network-proxy/execpolicy/file-search/rollout/codex-mcp/...）
         → 基础层（http-client/uds/async-utils/...）
         【图3】crate 分层架构图（flowchart + subgraph 四层）
  第4章 设计理念：四大支柱
         ① 事件驱动 Actor 内核（Op/Event 双通道契约）
         ② 单内核多前端（core 不绑定 UI，TUI/CLI/IDE/App-Server 平等）
         ③ 协议即契约（protocol crate 独立于实现）
         ④ 安全纵深（沙箱+审批+策略+网络代理四道闸）
  第5章 端到端流程：一次对话的完整生命周期
         submit(Op) → SessionIo → tx_sub → submission_loop → TurnInput
         → TurnContext → client 流式调用 → ResponseItem → 工具分发
         → 沙箱执行 → 结果回传 → 循环 → TurnComplete
         【图4】端到端时序图（sequenceDiagram：用户→前端→CodexThread→Session→client→模型→工具→沙箱）
  第6章 使用指南与案例
         6.1 安装（curl/npm/brew）与登录（ChatGPT 计划 / API key）
         6.2 构建与开发（just codex / just test / just fmt / Bazel）
         6.3 常用命令速查（codex 交互、codex exec 非交互、resume、/compact、审批模式）
         6.4 典型案例×3：
             案例1：交互式修 bug（TUI 全流程走查）
             案例2：codex exec 一行命令自动改代码（CI 场景）
             案例3：会话断点续传（rollout resume + JSONL）
         【图5】新手使用流程图（flowchart：安装→登录→选择模式→任务→审批→交付）
三、总结收束：一张认知地图收拢全文——"内核很小，边界很硬，契约很清晰"
```

### 3.2 文档二：`ReadCode/02-具体观-算法与实现剖析.md`

**篇幅目标**：约 9000–14000 字 + 7 张 Mermaid 图

**结构（总分总）**：总起（概述 7 个技术点如何共同支撑"可靠的编码 Agent"）→ 分述（每个技术点三段式：**论文原理 → 代码实现 → 具体例子**）→ 总结收束。

| # | 技术点 | 拟引论文（执行时 WebSearch 核实链接） | 代码锚点 | 具体例子 |
|---|--------|----------------------------------------|----------|----------|
| 1 | **Agent 主循环（ReAct 范式）** | ReAct: Synergizing Reasoning and Acting in Language Models（Yao et al., ICLR 2023, arXiv:2210.03629）；SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering（Yang et al., NeurIPS 2024, arXiv:2405.15793） | session/handlers.rs `submission_loop`；turn_input.rs；stream_events_utils.rs | 一次"读文件→改文件→跑测试"三轮工具调用的完整事件流转 |
| 2 | **流式响应与事件系统** | Efficient Streaming Language Models with Attention Sinks（Xiao et al., arXiv:2309.17453）；（补充）OpenAI Responses API 文档 | client.rs L1870-L2113（WebSocket 优先/HTTP SSE 回退、`map_response_events`） | 工具调用增量参数如何在流中被聚合为完整 function call |
| 3 | **上下文工程与压缩（compact）** | Lost in the Middle: How Language Models Use Long Contexts（Liu et al., TACL 2024, arXiv:2307.03172）；（补充）prompt caching 相关公开资料 | tasks/compact.rs；compact_token_budget.rs；context_manager/history.rs | token 超阈值触发压缩：pre-hook → 摘要新窗口 → post-hook 的全流程 |
| 4 | **apply_patch 补丁算法** | （实践源头）Aider 的 search/replace 编辑格式文档；SWE-bench: Can Language Models Resolve Real-World GitHub Issues?（Jimenez et al., ICLR 2024, arXiv:2310.06770）中 patch 验证思想 | apply-patch/src/parser.rs（`*** Begin Patch` 文法）；file_update.rs `compute_replacements` | 给出一段真实 patch 文本，逐行演示解析→匹配→替换→冲突处理 |
| 5 | **沙箱纵深防御** | Landlock LSM 官方论文/文档（Linux 安全模块）；capability-based security 经典论述（可引 Dennis & Van Horn 1966 或现代综述） | sandboxing/src/manager.rs（三平台选择）；core/README.md 平台细节；execpolicy（WASM/OPA） | workspace-write 模式下一条 `rm -rf /` 命令被拦截的全链路 |
| 6 | **网络代理与按请求授权** | （工程实践）MITM TLS 拦截的安全分析文献；MCP 规范（Anthropic, 2024） | network-proxy/src/mitm.rs、connect_policy、socks5、windows_tcp_attribution | 沙箱内命令访问外部 API 被代理捕获→策略判定→放行/阻断 |
| 7 | **极致工程（构建/测试/分发）** | Software Engineering at Google（测试章节）；（或）Google Testing on the Toilet 系列 | Bazel MODULE.bazel + Cargo 双轨；core/suite `test_codex` mock 框架；insta 快照（tui）；rollout 反向 JSONL 扫描 | test_codex 如何用 mock SSE 服务器无网络跑通端到端 Agent 测试 |

**Mermaid 图规划**：
- 【图1】Agent Loop 状态流转图（flowchart：思考→工具调用→执行→回传→再思考→完成/中断）
- 【图2】流式事件管道图（flowchart：SSE/WS → map_response_events → 聚合/分发）
- 【图3】上下文生命周期图（flowchart：token 预算阈值分支 → 三种 compact 路径）
- 【图4】apply_patch 解析状态机（stateDiagram-v2：Begin Patch → 文件头 → hunk → End Patch → 应用）
- 【图5】沙箱四道闸分层图（flowchart subgraph：审批 → execpolicy → OS 沙箱 → 网络代理）
- 【图6】MCP 连接管理架构图（flowchart：Codex ↔ McpConnectionSet ↔ 外部 MCP servers）
- 【图7】测试体系金字塔（flowchart：单测 → core/suite 集成 → insta 快照 → wine/远程矩阵）

**使用指南**（本文档形态）：附"源码阅读路线图"案例——按依赖逆序阅读 protocol → session → client → tools → sandboxing，每站给出建议入口文件与停留时长配比。

### 3.3 文档三：`ReadCode/03-深刻观-哲学与升华.md`

**篇幅目标**：约 5000–8000 字 + 4 张 Mermaid 图

**结构（总分总）**：

```
一、总起：回望两篇——看见了骨架（整体观）、看清了血肉（具体观），
   还差一口气：为什么它是这样设计的？它指向什么未来？
二、分述（三层递进）
  第1章 设计哲学：Agent 内核即操作系统内核
         —— 单内核多前端 = 微内核架构；protocol crate = syscall 表；
            rollout JSONL = 事件溯源日志（Event Sourcing）
         【图1】"Agent 内核"概念图（flowchart：内核/系统调用/用户态三层类比）
  第2章 框架思想：确定性外壳包裹概率性内核
         —— LLM 是概率的，但循环、审批、沙箱、回放是确定的；
            工程的全部努力 = 把不可预测的智能约束在可预测的轨道上
         【图2】思想演进脉络（timeline：function calling → ReAct → 工具生态
            → Agent 框架 → Agent-Computer Interface → Codex 内核化）
  第3章 算法框架之思：Agent = f(LLM, 工具, 循环, 记忆, 护栏)
         —— 五要素缺一不可；SWE-agent 的启示：接口设计（ACI）与模型能力同等重要
         【图3】设计哲学心智图（mindmap：内核化/契约化/可回放/纵深防御）
三、总结收束
   【图4】核心洞见总结图（flowchart：三层认知 → 一句话本质）
   —— 终极概括（两三句话，全文认知结晶）：
   "Codex 的本质，是把软件工程过程编译成一条可回放、可审计、可中断的事件流——
    模型负责思考，事件流负责事实，沙箱负责边界。
    一切成熟的 Agent 系统，终将收敛为同一个形态：
    一个安全的内核、一组显式的契约、一个受控的循环。"
   （执行时允许微调措辞，但必须保持两三句话、可独立传达本质、令人难忘）
```

**使用指南**（本文档形态）：附"何时回看哪一篇"决策小节（新人入门看一 / 深挖实现看二 / 架构决策看三），并以"如果你想造自己的 Agent"给出三步走建议案例。

---

## 四、执行步骤（执行阶段依序进行）

1. **补充精读源码**（只读）：core/src/client.rs 关键段、protocol/src/protocol.rs 枚举全貌、apply-patch/src/parser.rs、sandboxing/src/manager.rs、tui/src/chatwidget.rs 头部注释、core/gpt-5.2-codex_prompt.md 全文，为三篇文档采集直接引语与代码片段。
2. **论文搜索**：用 WebSearch 逐一核实表 3.2 中论文的 arXiv/正式链接与准确标题（ReAct、SWE-agent、Lost in the Middle、Attention Sinks/SSE 流式、SWE-bench、Toolformer（如用于工具节引言）、Landlock、MCP 规范）；补充检索 1-2 篇 Agent 安全综述备选。
3. **创建 `ReadCode/` 目录并撰写文档一**（Write 工具），边写边套用 Mermaid 校验清单。
4. **撰写文档二**，论文引用逐条以 `[标题](URL) — 一句话贡献` 格式落地。
5. **撰写文档三**，收尾两三句话反复打磨。
6. **全量校验**（见第五节清单），发现问题即改。
7. **交付**：向用户返回三篇文档摘要 + 文件链接 + 终极两三句话。

> 注：本任务只新增 ReadCode/ 下 3 个 md 文件，不触碰任何 Rust 源码，因此无需 `just fmt`/`just test`（不适用）；计划文件本身位于 .trae/documents/。

---

## 五、质量校验清单（Verification）

### 5.1 Mermaid 语法校验（每图必查）
- [ ] 节点/边标签含 `()`、`[]`、`{}`、`:`、`/` 等特殊字符时，一律用双引号包裹：`A["文本 (含括号)"]`
- [ ] `subgraph` 中文标题使用 `subgraph id["中文标题"]` 形式；subgraph 必须以 `end` 关闭
- [ ] 边标签使用 `-->|"标签"|` 或 `-- 标签 -->`（不含特殊字符时）
- [ ] `sequenceDiagram`：participant 别名不含空格；`Note over`/`activate`/`deactivate` 配对正确；`->>` 实线箭头、`-->>` 虚线返回
- [ ] `stateDiagram-v2`：状态名含空格用 `state "长名" as s1`；转移标签 `s1 --> s2 : 事件`
- [ ] `mindmap`：严格两级空格缩进，节点根不加引号时不含特殊字符（含则用 `"..."`）
- [ ] `timeline`：`section` 分节 + 缩进事件
- [ ] 不使用 `end`、`graph`、`class` 等保留字作节点 ID
- [ ] 图表代码块统一标注 `mermaid` 语言标签，且与其它代码块隔离正确闭合
- [ ] 如环境允许（npx 可用），用 `npx -y @mermaid-js/mermaid-cli` 批量渲染冒烟验证；不可用则按上述清单人工逐图走查并即时修正

### 5.2 结构与风格校验
- [ ] 每篇均为"总（开篇 2-3 段）→ 分（章节递进）→ 总（收束段）"
- [ ] 三篇之间递进：认知地图 → 技术细节 → 本质升华，前后互相呼应引用
- [ ] 每篇 ≥3 张 Mermaid 图（实际规划：文档一 5 张 / 文档二 7 张 / 文档三 4 张），每图上下配一句图注
- [ ] 每篇含使用指南与案例（文档一第6章 / 文档二源码阅读路线图 / 文档三"何时回看"）
- [ ] 论文引用格式统一：`[论文标题](URL) — 核心贡献一句话概括`，链接经 WebSearch 核实
- [ ] 文档三以两三句话收尾
- [ ] 代码/文件引用均带反引号或可点击 file:/// 链接
- [ ] 全文中文，术语首现附英文原文

---

## 六、假设与决策（Assumptions & Decisions）

1. **语言**：全中文撰写（技术术语保留英文，首现附中文释义）。
2. **保存位置**：`/workspace/ReadCode/`（用户指定），文件名按技能规范编号。
3. **篇幅**：三篇合计约 2-3 万字，允许长篇，深度优先。
4. **不修改源码**：只读分析 + 新建 3 个 md 文档；ReadCode/ 不属于仓库 docs/ 目录，符合 AGENTS.md "不在 docs/ 加产品文档"的约束。
5. **论文时效**：以经典 + 近三年 Agent 论文为主，链接优先 arXiv；检索日期 2026-08-21。
6. **Mermaid 版本**：按 v10+ 语法写作（flowchart/stateDiagram-v2/timeline/mindmap 均可用）。
7. **事实边界**：所有架构描述以 Phase 1 探索到的真实文件与行号为锚，未经证实的推测一律不写。
