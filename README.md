# AI Agent Harness 工程化问题集与处理方式

2026-07-05  

## 1. 定义与边界

在 AI Agent 工程中，Harness 不应仅理解为“测试脚手架”。更准确地说，Harness 是围绕模型的控制面、执行面、评测面、安全面、观测面和运维面的工程系统。

本文采用如下边界：

| 层次 | 含义 | 典型职责 |
|---|---|---|
| Agent Harness | 让模型能够作为 Agent 行动的系统 | 指令、工具、循环、状态、编排、审批、观测 |
| Execution Harness | 让 Agent 行动发生在受控环境中的系统 | Sandbox、浏览器、VM、文件系统、命令执行 |
| Evaluation Harness | 端到端运行评测并判断质量的系统 | 任务集、环境初始化、评分、聚合、回归门禁 |
| Operations Harness | 让 Agent 能稳定上线和持续治理的系统 | 发布、灰度、回滚、成本、限流、告警、审计 |

本文将当前 AI Agent 工程化中的 Harness 归纳为 12 类：

1. Agent Runtime Harness
2. Tool / MCP Harness
3. Sandbox / Execution Harness
4. Browser / Computer-Use Harness
5. Orchestration / Multi-Agent Harness
6. State / Memory / Persistence Harness
7. Retrieval / RAG Harness
8. Guardrail / Security Harness
9. Human-in-the-Loop Harness
10. Observability / Replay Harness
11. Evaluation / Benchmark Harness
12. Deployment / Operations Harness

## 2. 总体处理原则

### 2.1 控制面与执行面分离

Harness 控制面负责 Agent loop、模型调用、工具路由、handoff、审批、trace、恢复和状态管理。执行面负责文件、命令、浏览器、网络、API、数据库等真实副作用。

处理方式：

- 控制面不得直接拥有无边界执行能力。
- 所有真实副作用必须经过工具、sandbox、浏览器或 API 代理层。
- 执行面必须具备权限边界、审计、超时、资源限制和错误分类。
- 高风险动作必须通过审批、策略或显式用户确认。

验收标准：

- 任意一次 Agent 动作均能追溯到 run id、tool call id、输入参数、执行结果和审批状态。
- 任意执行失败均能定位到控制面、模型输出、工具参数、执行环境或外部依赖。

### 2.2 外部内容默认不可信

网页、PDF、邮件、知识库片段、工具返回、MCP 资源、用户上传文件和第三方 API 返回均应视为不可信数据，而非指令。

处理方式：

- 将外部内容放入明确的数据边界中。
- 不将外部内容拼接进高优先级开发者指令或系统指令。
- 对外部内容进入工具参数、长期记忆、审批请求前做校验和脱敏。
- 对提示注入、越权请求、数据外泄意图设置检测与拦截。

验收标准：

- 任何外部内容不能直接改变工具权限、审批策略、模型系统指令或租户边界。
- 安全回归集中必须包含间接 prompt injection、工具输出注入、文档注入、网页注入样本。

### 2.3 质量闭环必须基于 trace 和 eval

Agent 的质量问题通常出现在中间步骤，而非最终文本本身。因此必须同时建设 trace、回放和评测。

处理方式：

- 对模型调用、工具调用、检索、记忆读写、handoff、guardrail、审批、异常分别打 span。
- 将线上失败样本转为离线 eval case。
- 对 prompt、模型、工具、检索、guardrail 的变更建立回归门禁。

验收标准：

- 每次重大变更均有基线结果、回归结果和失败样本解释。
- 线上事故复盘后必须更新 eval suite 或安全规则。

## 3. Agent Runtime Harness

### 3.1 定位

Agent Runtime Harness 是最基础的 Harness，负责将一次用户任务转化为模型可执行的运行循环。它决定模型何时思考、何时调用工具、何时停止、如何输出、如何处理异常。

### 3.2 执行流程

1. 接收用户请求，统一转换为 `AgentRunRequest`。
2. 装配系统指令、开发者指令、用户输入、上下文、预算和策略。
3. 初始化 `AgentRunState`、run id、trace root 和终止条件。
4. 调用模型，获得最终输出、工具调用、handoff 请求或审批中断。
5. 若模型请求工具，转入 Tool Harness 执行，并将结构化工具结果回填。
6. 每轮执行后检查 guardrail、预算、轮次、错误阈值和超时。
7. 达到终止条件后生成 `AgentRunResult`。
8. 持久化最终状态、trace、成本、错误和可回放信息。

### 3.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 任务入口不统一 | 定义统一 request envelope，包含 user_id、tenant_id、session_id、intent、input、attachments、policy_context、budget、metadata | 所有入口均能映射到同一 schema；缺失字段有默认值或拒绝策略 |
| 指令层级混乱 | 明确 system、developer、user、tool、external data 的优先级；禁止将用户输入或外部数据注入高优先级指令 | 指令模板版本化；安全测试覆盖“忽略前文”等注入样本 |
| Agent loop 终止条件不清晰 | 设置 final output、max_turns、max_tokens、max_cost、timeout、tool_error_threshold、approval_pending 等终止条件 | 任意运行均能解释为何结束；无无限循环 |
| 模型直接回答与调用工具边界不清晰 | 为每个工具定义使用条件、禁止条件和示例；在 prompt 中声明何时必须查证、何时可直接回答 | trace 中可判断每次工具调用是否符合策略 |
| 工具结果回填上下文失控 | 工具输出做长度裁剪、结构化摘要、敏感字段脱敏；大结果以 artifact 或引用方式传递 | 单次工具返回不会导致上下文爆炸或敏感泄露 |
| 模型输出无效 | 使用结构化输出 schema；无效输出进入重试、修复或失败路径；避免任意 free-form 输出驱动下游动作 | 下游系统只消费 schema 校验通过的数据 |
| 错误恢复策略缺失 | 将错误分为模型错误、工具参数错误、外部 API 错误、权限错误、超时、资源耗尽；分别设置 retry、fallback、abort | 错误码标准化；错误处理路径可测试 |
| 成本和延迟不可控 | 设置每次运行预算；按模型、工具、检索、重试计算成本；低风险阶段使用低成本模型 | 超预算自动中断或降级；成本指标进入监控 |
| 流式输出与最终状态不一致 | 将 streaming event 与最终 RunResult 分离；最终状态以持久化 run state 为准 | UI 流式展示不影响最终可审计结果 |
| run id 与可追踪性缺失 | 每次运行生成全局唯一 run id；所有模型、工具、检索、审批、日志共享该 id | 任意问题可从 run id 定位完整链路 |

### 3.4 推荐实施路径

1. 定义 `AgentRunRequest`、`AgentRunState`、`AgentRunResult` 三个核心对象。
2. 将模型调用封装为可替换接口，固定模型参数和默认超时。
3. 将工具调用纳入统一 dispatcher，不允许模型直接执行副作用。
4. 添加循环终止策略和预算策略。
5. 为每次运行生成 trace 和结构化日志。
6. 将失败样本接入 Evaluation Harness。

## 4. Tool / MCP Harness

### 4.1 定位

Tool / MCP Harness 负责将外部能力暴露给 Agent。其核心问题不是“能否调用工具”，而是工具权限、参数、结果、错误、安全和版本是否可治理。

### 4.2 执行流程

1. 在工具注册中心加载工具定义、schema、权限、版本和风险等级。
2. 根据用户、租户、会话和任务上下文解析可用工具集合。
3. 接收模型生成的 tool call，并绑定 tool call id。
4. 对工具名称、参数 schema、权限、租户边界和业务规则做预校验。
5. 根据风险等级触发 guardrail、审批、拒绝或自动执行。
6. 以统一超时、重试、熔断和审计策略调用外部系统或 MCP server。
7. 对工具返回做结构化、脱敏、裁剪、错误归类和可信度标注。
8. 将工具结果、状态码、耗时、成本和审计信息写入 trace 并返回 Runtime Harness。

### 4.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 工具 schema 不明确 | 使用 JSON Schema、Pydantic、Zod 或 OpenAPI 定义参数、返回、错误码 | 工具参数必须通过 schema 校验 |
| 工具描述诱导错误调用 | 工具描述写明用途、禁止用途、参数语义、示例和副作用 | 对每个工具有正负调用样本 |
| 工具权限过大 | 按最小权限拆分 read、write、admin、external-send、delete 等工具 | 无通用无限制 shell、无限制 SQL、无限制 HTTP 工具 |
| 读写工具混用 | 查询工具和变更工具拆分；写操作要求幂等键或审批 | 写工具不可由纯查询意图触发 |
| 参数未经校验 | 在工具执行前做类型、范围、权限、业务规则和租户边界校验 | 无效参数不会进入外部系统 |
| 工具返回泄露敏感数据 | 对返回字段分级，默认最小返回；敏感字段脱敏或只返回引用 | trace 和模型上下文中不包含不必要敏感数据 |
| 工具输出被模型当指令 | 工具结果包装为数据对象，并标注 untrusted；禁止工具输出改变系统策略 | 注入样本无法提升权限或改写指令 |
| API 限流与重试失控 | 设置指数退避、最大重试、熔断、降级和排队策略 | 限流不会导致 Agent 无限重试 |
| 工具版本不可控 | 工具名、schema、行为和权限版本化；破坏性变更走新版本 | trace 可还原当时使用的工具版本 |
| MCP 鉴权缺失 | 本地 STDIO 与远程 HTTP MCP 分别处理；远程 MCP 使用 OAuth、scope、租户隔离和审计 | 每次 MCP 调用可追踪用户、scope、server、tool |
| 第三方工具供应链风险 | 固定依赖版本；检查工具来源；对高权限 MCP 做安全审查 | 新工具接入前必须通过安全评审 |

### 4.4 推荐实施路径

1. 建立工具注册中心，记录工具名称、版本、schema、权限、风险等级。
2. 为所有工具增加统一 pre-execution middleware。
3. 对写操作和外发操作加入审批或二次确认。
4. 将工具返回结构化，并按字段做脱敏和上下文裁剪。
5. 对 MCP server 做鉴权、scope、租户隔离和调用审计。

## 5. Sandbox / Execution Harness

### 5.1 定位

Sandbox / Execution Harness 负责为 Agent 提供可读写文件、执行代码、安装依赖、运行测试和生成产物的隔离环境。它是代码 Agent、数据分析 Agent、文档生成 Agent 的关键基础设施。

### 5.2 执行流程

1. 根据任务类型选择镜像、资源配额、网络策略和文件系统策略。
2. 创建隔离 sandbox，并挂载只读输入目录、工作目录和输出目录。
3. 注入最小必要凭证、环境变量和依赖缓存。
4. 接收命令、代码执行、文件读写或服务启动请求。
5. 在资源限制、超时和审计约束下执行操作。
6. 采集 stdout、stderr、exit code、文件变更、端口状态和产物。
7. 在关键步骤生成 snapshot 或 checkpoint。
8. 任务完成或失败后导出产物、写入 trace，并清理或销毁执行环境。

### 5.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 执行环境未隔离 | 使用容器、VM、Firecracker、Kubernetes sandbox 或托管 sandbox；禁止直接使用生产宿主机 | Agent 无法访问宿主机敏感路径 |
| 环境变量泄露 | 默认不继承宿主机 env；按任务注入最小凭证；密钥只暴露给需要的工具 | sandbox 内无法读取无关密钥 |
| 文件系统边界不清晰 | 明确只读输入目录、工作目录、输出目录、禁止路径 | 文件访问越界会被拒绝并记录 |
| 网络访问无控制 | 默认关闭或 allowlist；对外请求记录域名、URL、方法、状态码 | 不能任意访问内网或未知外部地址 |
| 资源耗尽 | 限制 CPU、内存、磁盘、进程数、运行时长、网络流量 | 失控任务被自动终止并保留原因 |
| 依赖安装不可复现 | 使用 lockfile、镜像缓存、私有包 allowlist；记录安装日志 | 同一任务可在干净环境中复现 |
| 命令执行不可审计 | 所有 shell/code 执行记录命令、cwd、env 摘要、stdout、stderr、exit code、耗时 | trace 可还原完整执行过程 |
| 长任务中断后无法恢复 | 在关键步骤生成 snapshot；状态和产物分离保存 | 任务可从最近安全 checkpoint 恢复 |
| 恶意代码或后台进程残留 | 任务结束后杀进程、卸载文件系统、销毁容器或重置快照 | 无跨任务污染 |
| 产物不可追踪 | 输出产物写入 artifact store，记录 hash、路径、生成步骤和权限 | 任意产物可追溯到 run id |
| 多租户隔离不足 | 每个租户或任务使用独立 sandbox、独立凭证、独立存储命名空间 | 租户间文件、网络、凭证不可见 |

### 5.4 推荐实施路径

1. 从容器级 sandbox 开始，禁止直接使用宿主机执行。
2. 定义 workspace 结构：`/input`、`/work`、`/output`、`/tmp`。
3. 对命令执行加入 resource limit、timeout 和审计。
4. 对网络采用默认拒绝策略，再按场景配置 allowlist。
5. 产物统一进入 artifact store，并与 trace 绑定。

## 6. Browser / Computer-Use Harness

### 6.1 定位

Browser / Computer-Use Harness 负责让 Agent 操作网页或桌面 UI。其风险高于普通 API 工具，因为网页内容、截图、表单、按钮和弹窗都可能包含不可信指令或造成真实副作用。

### 6.2 执行流程

1. 启动隔离浏览器、远程桌面、容器或 VM，并设置视口、域名策略和权限。
2. 打开目标页面或应用，初始化登录态、测试账号和下载目录。
3. 捕获当前 UI 状态，包括 screenshot、DOM、accessibility tree、URL 和弹窗状态。
4. 将 UI 状态作为不可信观察结果提供给模型或 UI 操作策略。
5. 接收点击、输入、滚动、导航、上传、下载等动作请求。
6. 在执行前校验目标域名、动作风险、元素状态和是否需要人工确认。
7. 执行动作后重新捕获 UI 状态，并处理遮罩、弹窗、跳转和错误。
8. 记录每一步动作、截图、页面状态和审批结果，形成可回放 trajectory。

### 6.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| UI 状态表示不稳定 | 根据任务选择 screenshot、DOM、accessibility tree 或混合模式；每步操作后重新获取状态 | Agent 不依赖过期坐标或过期 element ref |
| 坐标点击脆弱 | 优先使用可访问树或 DOM ref；必须使用坐标时绑定截图尺寸和缩放比例 | 不同视口下仍能稳定执行 |
| 弹窗、遮罩、cookie banner 干扰 | 建立 UI 异常处理策略；检测遮挡元素并重新获取状态 | 点击失败有明确恢复路径 |
| 登录态管理不安全 | 使用专用测试账号或受限账号；凭证不暴露给模型；敏感操作需要审批 | 模型不能读取或导出登录凭证 |
| 页面提示注入 | 将页面文本视为 untrusted data；页面内容不得授予权限或覆盖用户意图 | 网页中的“忽略之前指令”无效 |
| 截图泄露敏感信息 | 截图进入模型前做区域裁剪、模糊或最小化；日志中控制截图保存权限 | trace 不暴露不必要隐私 |
| 文件上传下载风险 | 下载目录隔离；文件类型、大小、来源校验；上传前审批 | Agent 不能任意上传本地敏感文件 |
| 跨域和外部跳转风险 | 域名 allowlist；跳转到未知域时暂停或请求确认 | 不会被钓鱼页面诱导执行 |
| 高影响动作误触发 | 对支付、提交订单、发送消息、删除记录、同意条款等动作设置人工确认 | 高影响动作无审批不得执行 |
| 任务失败不可回放 | 记录每步截图、动作、目标元素、URL、DOM 摘要、时间戳 | 可从 trace 重建失败路径 |

### 6.4 推荐实施路径

1. 初期优先使用 Playwright/Selenium 等可审计自动化框架。
2. 浏览器运行在隔离容器或 VM 中，关闭不必要扩展和本地文件访问。
3. 每一步执行后保存状态快照。
4. 对页面内容进入模型上下文前加 untrusted 边界。
5. 建立高影响 UI 动作审批清单。

## 7. Orchestration / Multi-Agent Harness

### 7.1 定位

Orchestration / Multi-Agent Harness 负责多个 Agent、工具和工作流节点之间的分工、路由、handoff、合并和裁决。其关键是避免过早复杂化，并防止多 Agent 带来的上下文污染、循环和成本失控。

### 7.2 执行流程

1. 接收主任务，并判断是否由单 Agent、manager、specialist 或固定工作流处理。
2. 根据任务意图、策略、工具需求和上下文复杂度选择路由模式。
3. 对交给子 Agent 的上下文做裁剪、脱敏和任务封装。
4. 调用 specialist、agent-as-tool 或 handoff 目标，并记录路由原因。
5. 收集子 Agent 的结构化输出、证据、错误和成本。
6. 对多个结果进行冲突检测、可信度排序、规则裁决或人工升级。
7. 将有效结果合并回全局状态，并由最终 owner 生成输出。
8. 对路由、handoff、合并、冲突和失败全链路打 trace。

### 7.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 过早拆分 Agent | 默认从单 Agent 开始；仅在能力隔离、策略隔离、工具隔离或评测清晰度明显改善时拆分 | 拆分有明确收益说明 |
| 最终回答权不清 | 明确 manager、specialist、handoff 三类角色；声明谁负责最终输出 | 最终输出只有一个 owner |
| handoff 与 agent-as-tool 混用 | 需要专家接管对话时用 handoff；需要经理综合结果时用 agent-as-tool | 路由模式在设计文档中明确 |
| 子 Agent 输入过宽 | 为子 Agent 定义最小输入、上下文过滤和输出 schema | 子 Agent 不接收无关历史或敏感上下文 |
| 多 Agent 结论冲突 | 建立裁决策略：规则优先、可信源优先、投票、复核 Agent、人工确认 | 冲突不会被静默覆盖 |
| 循环委派 | 设置最大 handoff 次数、重复路由检测和环路断路器 | trace 中无无限委派 |
| 共享状态污染 | 状态写入必须带来源、版本、置信度和作用域；关键状态写入需校验 | 子 Agent 不能随意改写全局状态 |
| 局部失败扩大 | 子 Agent 失败返回结构化错误，由 manager 决定重试、降级、转人工 | 单点失败不导致全局不可解释失败 |
| 成本指数增长 | 对每个 Agent 设预算；限制并发、轮次、工具调用次数 | 多 Agent 工作流成本可预测 |
| 子 Agent 不可评测 | 每个 Agent 有独立单元 eval 和端到端 eval | 能定位失败来自路由、子任务或综合阶段 |

### 7.4 推荐实施路径

1. 先实现单 Agent baseline。
2. 识别必须拆分的职责边界。
3. 为每个 Agent 定义输入、输出、工具、权限、风险等级。
4. 对路由决策打 trace，并纳入 eval。
5. 加入环路检测、预算限制和冲突裁决。

## 8. State / Memory / Persistence Harness

### 8.1 定位

State / Memory / Persistence Harness 负责保存会话状态、任务状态、长期记忆、checkpoint 和恢复信息。它既影响 Agent 能力，也直接影响安全和合规。

### 8.2 执行流程

1. 根据 run id、thread id、user id 和 tenant id 加载作用域内状态。
2. 区分当前运行状态、会话历史、长期记忆、任务产物和审计记录。
3. 在模型调用、工具执行、审批中断和产物生成等边界创建 checkpoint。
4. 对状态更新执行 schema 校验、权限校验、来源标注和版本控制。
5. 对长期记忆写入执行可信度判断、冲突检测、敏感信息检测和用户授权检查。
6. 在上下文超限时执行压缩、摘要或检索式记忆加载。
7. 在失败、中断或恢复时读取最近安全 checkpoint，并检查幂等记录。
8. 按留存策略执行归档、删除、迁移和审计导出。

### 8.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 状态范围不清 | 区分 run state、thread state、user memory、tenant memory、global knowledge | 每类状态有独立 schema 和权限 |
| checkpoint 粒度不合理 | 在模型调用前后、工具执行前后、审批中断、产物生成后建立 checkpoint | 能从关键边界恢复 |
| 长短期记忆混淆 | 短期记忆用于当前会话；长期记忆必须经过写入策略、用户授权和安全校验 | 临时错误不会永久污染用户记忆 |
| 记忆投毒 | 外部内容写入前做来源标记、可信度评分、冲突检测和敏感信息检测 | 注入内容不能长期影响高风险决策 |
| 上下文压缩丢失关键信息 | 压缩时保留任务目标、约束、审批状态、已执行动作、未完成事项 | 恢复后 Agent 不重复或遗漏关键动作 |
| 用户删除数据不可执行 | 为记忆和状态记录 subject id、tenant id、retention policy | 可按用户或租户删除 |
| 状态 schema 演进困难 | 状态版本化；提供 migration；旧 trace 只读保留 | 新版本可读取或迁移旧状态 |
| 恢复后重复执行副作用 | 写操作使用幂等键；恢复时检查已执行 tool call id | 不会重复付款、发信、删除 |
| 并发冲突 | 对同一 thread 或资源使用乐观锁、版本号、队列或事务 | 并发运行不会覆盖关键状态 |
| 审计字段不足 | 状态记录来源、时间、操作者、工具、审批人、变更前后摘要 | 可满足复盘与合规要求 |

### 8.4 推荐实施路径

1. 先定义状态分层，而非直接建设“记忆”。
2. 对每个状态字段定义 owner、生命周期、权限和可删除性。
3. 对长期记忆写入采用显式策略，不允许模型任意写入。
4. 在所有副作用工具上实现幂等键。
5. 将 checkpoint 与 replay 机制联动。

## 9. Retrieval / RAG Harness

### 9.1 定位

Retrieval / RAG Harness 负责让 Agent 使用外部知识。其核心不是“能检索”，而是检索内容是否可信、相关、最新、有权限、可引用，并且不会污染指令或记忆。

### 9.2 执行流程

1. 注册数据源，记录来源、权限、可信度、更新频率和保留策略。
2. 对文档进行解析、清洗、切块、去重、元数据抽取和索引。
3. 接收用户查询或 Agent 子查询，并绑定用户、租户和权限上下文。
4. 可选执行 query rewrite、query decomposition 或过滤条件生成。
5. 在权限过滤约束下执行向量检索、关键词检索或混合检索。
6. 对候选片段做 rerank、去重、freshness 检查和可信度排序。
7. 将证据、引用、分数和不可信标注返回给生成阶段。
8. 对最终回答做 groundedness、引用完整性和无答案路径检查，并记录检索指标。

### 9.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 数据源可信度不清 | 按官方文档、内部知识库、用户上传、第三方网页等分级 | 检索结果携带 source trust level |
| 文档解析错误 | 使用结构化 parser；保留页码、标题、表格、图片说明、原文引用 | 可从答案追溯到原文片段 |
| chunk 策略不合理 | 按语义、标题、段落、表格边界切块；控制 chunk size 和 overlap | 检索样本评估召回率与噪声 |
| 索引过期 | 定义增量更新、重新索引、删除同步、版本戳和 freshness | 答案可显示知识版本或更新时间 |
| 权限过滤后置 | 权限过滤必须在检索前或检索过程中完成，不能只在生成后过滤 | 用户无法检索无权限片段 |
| query rewrite 失控 | query rewrite 输出结构化；保留原始 query；限制扩展范围 | rewrite 不改变用户真实意图 |
| rerank 规则不可解释 | 记录召回分、rerank 分、过滤原因、最终选用片段 | 可解释为何使用某片段 |
| 检索不到时幻觉 | 明确无结果路径：询问澄清、声明不足、转人工、扩大检索范围 | 无依据时不生成确定性答案 |
| 引用缺失 | 输出要求带来源、段落、页码或链接；无引用内容标注为推断 | 关键事实均可追溯 |
| 检索内容注入 | 检索片段标记为 untrusted evidence；不得覆盖系统策略或工具权限 | 文档内指令无法操控 Agent |
| 私有数据进入共享上下文 | 按用户、租户、数据集隔离索引；禁止跨租户缓存污染 | 私有片段不会进入他人会话 |

### 9.4 推荐实施路径

1. 建立数据源目录和权限模型。
2. 解析、切块、索引、更新、删除全链路版本化。
3. 检索前执行权限过滤。
4. 对 answer generation 强制引用和 groundedness 检查。
5. 用真实问题集评估召回、精确、引用正确性和无答案处理。

## 10. Guardrail / Security Harness

### 10.1 定位

Guardrail / Security Harness 负责控制 Agent 在不确定输入、工具副作用、敏感数据和高风险任务下的行为边界。它必须覆盖输入、上下文、工具、输出和运行时策略，而不是只做内容审核。

### 10.2 执行流程

1. 在请求进入 Runtime Harness 前执行输入分类、风险识别和策略匹配。
2. 对外部内容、检索片段、工具输出和用户上传文件加不可信边界。
3. 在模型调用前检查上下文是否包含敏感数据、越权内容或注入迹象。
4. 在工具执行前校验权限、参数、租户、风险等级和审批要求。
5. 在工具执行后检查返回数据是否可进入模型上下文或最终输出。
6. 在最终输出前执行 schema、安全、隐私、事实引用和策略一致性检查。
7. 对违规结果执行 block、redact、degrade、ask-human 或 fail-close。
8. 记录策略版本、命中规则、风险分、处理动作，并将绕过样本沉淀到 eval。

### 10.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 威胁模型缺失 | 按 prompt injection、tool misuse、data exfiltration、memory poisoning、excessive agency、DoW、supply chain 建模 | 每个风险有检测、缓解和测试样本 |
| 输入 guardrail 缺失 | 对用户输入做意图分类、敏感任务识别、越权检测、恶意内容检测 | 高风险输入不直接进入高权限 Agent |
| 工具 guardrail 缺失 | 工具执行前校验权限、参数、资源、租户、风险等级；执行后校验输出 | 工具不能被模型绕过策略调用 |
| 输出 guardrail 缺失 | 输出前做 schema、敏感信息、政策、事实引用和格式校验 | 输出不含未授权数据或无效结构 |
| 外部内容提权 | 外部内容只作为 data；模型不得从外部内容获得新权限 | 间接注入测试不能绕过 |
| 结构化约束不足 | 高风险节点使用 enum、fixed schema、required fields；减少 free-form 通道 | 下游动作不消费自由文本命令 |
| 敏感数据处理不足 | 数据分级、最小上下文、脱敏、masking、tokenization、访问审计 | 日志、trace、模型上下文均受控 |
| 策略失败时行为不明确 | 定义 fail-close、fail-open、ask-human、degrade 四类策略；高风险默认 fail-close | 策略异常不会自动放行高风险动作 |
| 误杀率不可评估 | 建立安全 eval，覆盖正常样本、边界样本、攻击样本 | 评估 precision、recall、false positive |
| 规则不可版本化 | 安全策略、正则、分类器、阈值、审批规则版本化 | 任意输出可追溯当时策略版本 |
| 绕过样本未沉淀 | 红队和线上攻击样本进入回归集 | 同类绕过不会重复上线 |

### 10.4 推荐实施路径

1. 先做动作风险分级和工具权限分级。
2. 对输入、工具、输出建立三段式 guardrail。
3. 对高风险工具默认审批或 fail-close。
4. 将安全策略与业务策略版本化。
5. 建立安全回归集，并纳入 CI。

## 11. Human-in-the-Loop Harness

### 11.1 定位

Human-in-the-Loop Harness 负责在 Agent 执行高风险动作前暂停、展示证据、获取人类决策，并从正确状态恢复执行。它是高影响 Agent 的必要控制，而不是附加 UI。

### 11.2 执行流程

1. 在工具调用、UI 动作或工作流节点执行前识别是否命中审批策略。
2. 若需要审批，创建 interruption，并在当前安全边界生成 checkpoint。
3. 组装审批包，包含用户意图、动作、参数、证据、风险和预期副作用。
4. 将审批请求发送给具备权限的审批人或审批系统。
5. 接收 approve、reject、edit、clarify 或 escalate 决策。
6. 将审批结果结构化写入状态、trace 和审计日志。
7. 若批准或修改后批准，从 checkpoint 恢复并继续执行；若拒绝则安全终止或改走替代路径。
8. 对审批等待、超时、撤回、二次确认和争议处理进行持续审计。

### 11.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 哪些动作需要审批不明确 | 建立动作风险矩阵：read、draft、send、write、delete、financial、legal、admin、external-visible | 每个工具有 approval policy |
| 审批证据不足 | 审批页展示用户意图、工具名称、参数、数据来源、预期副作用、风险说明、可选替代 | 审批人能独立判断 |
| 审批人上下文不足 | 提供最小必要 trace、原始用户请求、关键检索证据和历史决策 | 不要求审批人阅读完整日志 |
| 批准、拒绝、修改路径缺失 | 支持 approve、reject、edit parameters、request clarification、escalate | 审批结果结构化写入 state |
| 运行暂停不可恢复 | 在审批前创建 checkpoint；审批后用 RunState 或 graph state 恢复 | 恢复不重跑已完成副作用 |
| 审批超时 | 设置 SLA、超时策略、提醒、自动拒绝或升级 | 长时间 pending 不占用执行资源 |
| 模型伪造用户同意 | 用户同意只能来自可信 UI/API，不接受模型生成的“用户已同意”文本 | 审批事件有独立身份和签名 |
| 高影响动作二次确认缺失 | 对不可逆、外部可见、财务、合规动作增加二次确认或双人审批 | 单一误点不导致重大损失 |
| 审批 UI 泄露敏感参数 | 对审批展示字段做最小化和脱敏；敏感内容只给授权审批人 | 审批链路符合数据权限 |
| 审计不可用 | 记录审批人、时间、决策、理由、参数摘要、前后状态 | 可用于复盘、合规和争议处理 |

### 11.4 推荐实施路径

1. 按工具建立风险等级和审批策略。
2. 在工具执行前插入 interruption。
3. 审批请求写入持久化状态，并通知审批人。
4. 审批后从 checkpoint 恢复，而非重新开始。
5. 将审批结果进入 trace 和审计日志。

## 12. Observability / Replay Harness

### 12.1 定位

Observability / Replay Harness 负责让 Agent 的每个关键决策、动作、失败和成本可见、可查、可复现。没有该 Harness，Agent 质量问题只能靠猜测。

### 12.2 执行流程

1. 在每次 Agent 运行开始时创建 trace root，并绑定 run id、tenant id、user id 和版本信息。
2. 对模型调用、工具调用、检索、记忆、handoff、guardrail、审批、sandbox 操作创建 span。
3. 在每个 span 中采集输入摘要、输出摘要、状态、错误、耗时、token、成本和关联 artifact。
4. 对日志、截图、工具返回和上下文执行脱敏、裁剪和访问控制。
5. 对失败 span 做自动分类，并支持人工标注根因。
6. 将可重放输入、mock、环境快照或外部响应摘要保存为 replay fixture。
7. 将指标汇总到 dashboard，并对异常阈值触发告警。
8. 将高价值失败 trace 转为 Evaluation Harness 的回归样本。

### 12.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| trace 不完整 | span 覆盖模型调用、工具调用、检索、记忆、handoff、guardrail、审批、状态变更 | 能还原完整执行路径 |
| span 结构不统一 | 定义统一 trace schema：run_id、span_id、parent_span_id、type、input、output、status、latency、cost | 所有框架输出可进入同一观测系统 |
| 日志泄露敏感数据 | 日志分级；默认脱敏；敏感 payload 只存加密引用 | 调试不需要暴露完整隐私数据 |
| 失败分类粗糙 | 将失败分为 instruction、planning、tool_choice、tool_args、retrieval、memory、permission、external、model_output、policy | 能统计主要失败模式 |
| 成本不可见 | 记录 token、模型、工具耗时、外部 API 成本、sandbox 资源 | 单任务和租户级成本可计算 |
| 线上样本无法回放 | 保存可重放输入、工具 mock、环境快照或外部依赖响应摘要 | 失败样本可在离线环境复现 |
| 版本对比困难 | trace 记录 prompt version、model version、tool version、policy version、index version | 能对比变更前后行为 |
| 异常不告警 | 设置指标阈值：成功率、失败率、延迟、成本、审批等待、工具错误、拒绝率 | 异常可进入告警系统 |
| trace 与 eval 脱节 | 线上失败 trace 可一键转 eval case；保留输入、期望、评分依据 | 生产问题进入回归体系 |
| 观测系统成为数据风险 | 设置访问控制、留存周期、脱敏、审计、删除能力 | 观测数据符合安全与合规要求 |

### 12.4 推荐实施路径

1. 在 Runtime Harness 层统一注入 trace context。
2. 对所有工具、检索、记忆、审批、sandbox 执行自动打 span。
3. 建立失败标签体系。
4. 提供 replay runner 或至少提供可重放 fixture。
5. 将 trace 样本转入 Evaluation Harness。

## 13. Evaluation / Benchmark Harness

### 13.1 定位

Evaluation / Benchmark Harness 负责判断 Agent 是否真的变好。它必须同时评估最终结果和中间过程，并处理 Agent 非确定性、多步骤、工具副作用和环境依赖。

### 13.2 执行流程

1. 选择评测套件，明确任务集、风险覆盖、基线版本和通过阈值。
2. 为每个 case 初始化干净环境、fixture、mock、数据快照和权限上下文。
3. 按固定配置或实验配置运行 Agent，并根据需要进行多次采样。
4. 收集最终输出、环境状态、工具轨迹、检索证据、审批行为和完整 trace。
5. 使用 deterministic tests、schema check、unit tests、LLM judge 或人工标注评分。
6. 聚合 pass rate、稳定性、成本、延迟、安全命中率和失败类别。
7. 与 baseline 比较，并按发布门禁判断通过、警告或阻断。
8. 输出失败样本、根因标签、回归建议，并同步到开发和安全闭环。

### 13.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 评测任务不代表真实场景 | 从线上任务、失败样本、边界案例、红队样本构建 eval suite | eval 覆盖核心用户路径和高风险路径 |
| 成功标准不明确 | 每个 case 定义 input、environment、expected outcome、allowed variance、scoring rubric | 评分不依赖临场主观判断 |
| 环境不可复现 | 使用容器、fixture、mock server、seed data、数据库快照 | 同一 case 可重复运行 |
| 只评最终文本 | 同时评估工具选择、参数、检索证据、计划、状态变更、审批行为 | 可定位失败发生在哪一步 |
| 评分器单一 | 组合 deterministic tests、unit tests、schema check、LLM judge、human calibration | 关键任务优先使用可验证 outcome |
| 非确定性未处理 | 多次运行，统计 pass@1、pass@k、稳定性、方差；固定必要 seed 和环境 | 不以单次成功代表系统可靠 |
| 模型能力与 Harness 能力混淆 | 记录模型、prompt、工具、环境、策略版本；对比时只改变一个变量 | 能判断提升来自何处 |
| LLM judge 不可靠 | Judge prompt 版本化；使用 golden labels 校准；抽样人工复核 | judge 与人工一致性可度量 |
| 失败样本不可解释 | 保存 transcript、trace、环境状态、评分理由 | 失败可复盘并归因 |
| 未接入 CI/CD | 将核心 eval suite 作为 merge gate；大型 eval 定期运行 | 关键指标下降阻断发布 |
| 安全 eval 缺失 | 增加 prompt injection、data leakage、tool misuse、excessive agency、memory poisoning 样本 | 安全回归与功能回归同等处理 |

### 13.4 推荐实施路径

1. 从 20 到 50 个高价值 case 建立第一版 eval suite。
2. 每个 case 定义环境、输入、成功标准和评分器。
3. 将 trace 与评测结果绑定。
4. 对每次模型、prompt、工具、检索、策略变更运行回归。
5. 将生产失败持续加入 eval suite。

## 14. Deployment / Operations Harness

### 14.1 定位

Deployment / Operations Harness 负责 Agent 的生产化发布、灰度、回滚、成本控制、租户隔离、合规审计和事故处理。其核心目标是让 Agent 具备可运维性。

### 14.2 执行流程

1. 打包发布对象，包括模型、prompt、工具、策略、索引、记忆策略、runtime 和 sandbox 配置。
2. 运行功能 eval、安全 eval、性能检查和合规检查，确认发布门禁。
3. 通过 feature flag、租户白名单、用户分组或流量比例执行灰度发布。
4. 在线监控成功率、延迟、成本、工具错误、审批等待、拒绝率和用户反馈。
5. 执行租户级限流、预算控制、密钥轮换、配额管理和异常熔断。
6. 出现事故时进入降级、暂停高风险工具、转人工、回滚或切换只读模式。
7. 保留必要审计记录、发布记录、审批记录和数据访问记录。
8. 事故复盘后更新 runbook、guardrail、eval suite、监控阈值和发布策略。

### 14.3 问题集与处理方式

| 问题 | 处理方式 | 工程产物与验收标准 |
|---|---|---|
| 模型版本漂移 | 固定模型版本、参数、reasoning effort、temperature、tool choice 策略 | 发布结果可复现 |
| prompt 和工具不可回滚 | prompt、tool schema、policy、index、memory strategy 全部版本化 | 可按版本回滚 |
| 密钥管理混乱 | 使用 secret manager；按工具和租户发放最小权限凭证；定期轮换 | Agent 无法读取无关密钥 |
| 租户隔离不足 | 请求、状态、记忆、索引、trace、sandbox、artifact 全部带 tenant boundary | 跨租户访问被拒绝 |
| 限流和预算缺失 | 设置用户、租户、工具、模型、任务维度的 rate limit 和 budget | 防止滥用和 denial of wallet |
| 灰度发布缺失 | 按租户、用户群、流量比例、任务类型灰度；监控关键指标 | 异常可快速停止扩散 |
| 线上质量指标不清 | 监控成功率、人工接管率、拒绝率、工具错误率、延迟、成本、投诉、回归命中率 | 指标与业务目标绑定 |
| 故障降级缺失 | 定义只读模式、禁用高风险工具、转人工、使用低权限 Agent、返回安全失败 | 外部依赖故障不导致危险动作 |
| 长任务调度混乱 | 使用队列、任务状态机、幂等任务 id、超时、重试和取消机制 | 长任务可暂停、恢复、取消 |
| 队列积压不可见 | 监控队列长度、等待时间、失败重试、worker 健康度 | 积压触发扩容或限流 |
| 合规审计不足 | 保留必要审计日志、审批记录、数据访问记录、输出记录；设置留存和删除策略 | 满足企业审计和监管要求 |
| 事故复盘无闭环 | incident 后更新 guardrail、eval、runbook、监控阈值和工具策略 | 同类事故可被提前检测或阻断 |

### 14.4 推荐实施路径

1. 建立发布对象清单：model、prompt、tool、policy、index、memory、runtime。
2. 将所有发布对象版本化并记录到 trace。
3. 引入灰度、回滚和 feature flag。
4. 设置租户级预算、限流和告警。
5. 建立 incident 到 eval 和 guardrail 的闭环机制。

## 15. 跨 Harness 依赖关系

| 上游 Harness | 依赖的下游能力 | 如果缺失的后果 |
|---|---|---|
| Runtime Harness | Tool、State、Observability | Agent 可运行但不可治理 |
| Tool Harness | Guardrail、HITL、Observability | 工具调用可能越权或不可审计 |
| Sandbox Harness | Deployment、Security、Replay | 执行可用但风险不可控 |
| Browser Harness | Guardrail、HITL、Replay | 网页注入和误触高影响动作风险高 |
| Multi-Agent Harness | State、Observability、Evaluation | 无法定位路由和协作失败 |
| Memory Harness | Security、Deployment | 记忆投毒和隐私删除风险 |
| RAG Harness | Security、Evaluation、Observability | 幻觉、越权检索和引用错误 |
| Guardrail Harness | Evaluation、Observability | 安全策略无法验证 |
| HITL Harness | State、Audit、Replay | 审批无法恢复或不可追责 |
| Observability Harness | Runtime、Tool、State | 无法复盘和量化质量 |
| Evaluation Harness | Sandbox、Replay、Trace | 回归结果不可复现 |
| Deployment Harness | 全部 Harness | 生产 Agent 无法稳定运营 |

## 16. 上线准入清单

上线前必须完成以下检查：

- Runtime：是否有统一 request、state、result、run id、终止条件和预算限制。
- Tool：所有工具是否 schema 化、最小权限化、版本化、可审计。
- Sandbox：是否隔离文件、网络、环境变量、资源和产物。
- Browser：页面内容是否不可信处理，高影响 UI 动作是否审批。
- Orchestration：多 Agent 是否有清晰 owner、路由、冲突处理和环路限制。
- State：checkpoint、长期记忆、恢复、幂等和删除机制是否明确。
- RAG：检索权限、引用、freshness、无答案路径和注入防护是否明确。
- Guardrail：输入、工具、输出三段式防护是否具备。
- HITL：审批动作、证据展示、恢复和审计是否闭环。
- Observability：trace 是否覆盖关键路径，是否支持失败归因和回放。
- Evaluation：是否有功能、安全、回归和生产失败样本集。
- Deployment：是否有版本、灰度、回滚、限流、预算、告警和事故复盘机制。

## 17. 建设优先级

### 17.1 原型阶段

优先建设：

1. Runtime Harness
2. Tool Harness
3. Observability Harness 的最小 trace
4. Evaluation Harness 的小规模 golden set

目标：证明 Agent 可以稳定完成核心任务，并能解释失败。

### 17.2 内测阶段

优先补齐：

1. Guardrail Harness
2. Sandbox 或 Browser Harness 的隔离能力
3. State / Persistence Harness
4. HITL Harness 的高风险动作审批

目标：控制真实副作用和安全风险。

### 17.3 生产阶段

必须完善：

1. Deployment / Operations Harness
2. 完整 Observability / Replay Harness
3. 持续 Evaluation / Benchmark Harness
4. Incident 到 eval、guardrail、runbook 的闭环

目标：实现稳定发布、可观测、可回滚、可持续改进。

## 18. 参考来源

- Anthropic: Demystifying evals for AI agents  
  https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- OpenAI: Agents SDK and orchestration  
  https://developers.openai.com/api/docs/guides/agents  
  https://developers.openai.com/api/docs/guides/agents/orchestration
- OpenAI: Sandbox Agents  
  https://developers.openai.com/api/docs/guides/agents/sandboxes
- OpenAI: Computer Use  
  https://developers.openai.com/api/docs/guides/tools-computer-use
- OpenAI Agents SDK: Tools, Guardrails, Tracing, Human-in-the-loop  
  https://openai.github.io/openai-agents-python/tools/  
  https://openai.github.io/openai-agents-python/guardrails/  
  https://openai.github.io/openai-agents-python/tracing/  
  https://openai.github.io/openai-agents-python/human_in_the_loop/
- LangGraph: Overview, Persistence, Interrupts  
  https://docs.langchain.com/oss/python/langgraph/overview  
  https://docs.langchain.com/oss/python/langgraph/persistence  
  https://docs.langchain.com/oss/python/langgraph/interrupts
- LangSmith Observability  
  https://docs.langchain.com/langsmith/observability
- Model Context Protocol  
  https://modelcontextprotocol.io/docs/getting-started/intro  
  https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- OWASP AI Agent Security Cheat Sheet  
  https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html
- SWE-bench  
  https://github.com/swe-bench/SWE-bench
