# DeepSeek Harness 系统架构深度解析与技术调研

> 从开源代码理解一个可组合、可恢复、可扩展的编码 Agent 运行时

**调研日期：2026 年 11 月 1 日**  
**官方仓库：** https://github.com/deepseek-ai/deepseek-harness  
**源码基线：** `ce7073d51254c26cd63b22a60835b8f948deb447`，提交时间 2026-10-30 00:13:15 +08:00。  
**仓库根包版本：** `0.2.2-alpha.2`；这只是本次快照中的根包版本，不代表所有发布渠道的最新版本。  
**许可证：** MIT。  
**阅读对象：** Agent 平台研发、编码助手研发、技术架构师，以及希望理解编码 Agent 系统工程实现的读者。

本文依据实际克隆的官方仓库，结合架构说明、默认部署组合、关键 TypeScript 实现、包级协议和部分测试用例进行静态研究。不是只根据 README 总结，也不是对 DeepSeek 内部未公开系统的推测。

文中的“源码事实”指本次固定提交中可定位的实现或配置；“设计解读”解释其工程意义；“建议”是本文提出的落地判断。未安装全量依赖、启动产品、调用付费模型或运行完整测试，因此不把代码路径分析写成性能实测、安全认证或测试通过结论。

## 01 执行摘要：DeepSeek Harness 到底是什么

DeepSeek Harness，代码中通常缩写为 DSH，是围绕模型构建编码 Agent 的开源运行时与产品基础设施。它将模型、工具、会话、文件系统、子进程、权限、客户端及扩展能力装配在同一个插件体系中。

“Harness”可以理解成连接模型与真实环境的整套执行装置。模型负责生成文本、推理和工具调用意图；Harness 负责决定哪些输入进入模型、哪些工具可见、工具如何获准执行、结果如何进入历史、断线和重启如何恢复，以及界面如何看到过程。

它解决的核心问题不是“如何发一次聊天请求”，而是“如何把反复调用模型、执行带副作用工具的过程组织成可持续维护的软件系统”。

### 1.1 最值得关注的六个设计

| 设计 | 本次源码中的表现 | 主要价值 | 对应代价 |
|---|---|---|---|
| 插件化组合 | Cordis 服务、事件和受生命周期管理的 effect；Agent Loop 也作为插件提供 | 同一领域接口可替换实现，支持不同产品组合 | 装配顺序、作用域和插件协议必须严格治理 |
| 日志与模型表面分离 | Session 保存事件，surface 决定当前模型可见内容 | 压缩不必删除审计历史，可重建请求上下文 | 历史回放、替换范围和事件版本更复杂 |
| 请求准备形成稳定边界 | step 固定模型选择、解析工具集合、准备调用，再提交实际输入 | 减少模型切换、工具卸载、取消之间的不一致 | 配置变更必须遵守明确时机 |
| 工具统一执行管线 | pre → approval → guards → body → projection → post → finalize | 扩展策略与工具实现解耦，结果形态统一 | 插件错误和异步取消也要纳入协议 |
| 语义持久化检查点 | 模型请求和顶层工具实际执行前 flush | 降低“已经发生外部效果，但没有对应调用记录”的风险 | 存储故障会阻止请求和工具推进 |
| 恢复承认不确定性 | 区分工具尚未开始与可能已经产生效果 | 避免恢复时盲目重复执行写操作 | 不能提供任意外部系统上的 exactly-once |

### 1.2 一句话架构判断

DSH 的主体是一个**以插件装配为基础，以 Session 事件记录和模型可见 surface 为状态边界，以 Agent Loop 和工具管线驱动执行的 Agent 平台**。CLI、Web、Desktop、SDK 等是它的不同宿主和交互入口。

这不是性能排名。仅凭静态源码，不能得出它比其他编码助手更聪明、更快、更安全的结论；可以比较的是架构分工、接口机制及已经公开的实现边界。

来源：[总体架构][S01]、[默认组合][S02]、[Agent Loop 实现][S03]、[工具执行协议][S04]。

## 02 调研范围、证据等级与仓库规模

### 2.1 固定版本比“看过 GitHub”更重要

仓库开发速度较快。把分析固定到具体提交，可以让读者在数周后仍然打开同一份实现。本文所有 `[Sxx]` 源码引用均指向上述提交，而不是持续变化的 `master`。

研究采用三条线交叉检查：

1. **架构线：** `docs/architecture.md`、Agent 生命周期、工具执行管线和子系统说明。
2. **装配线：** `packages/bundle/base/cordis.patch.yml` 及其声明的默认服务、策略、工具与执行器。
3. **实现线：** Agent Loop、Session、工具调度、检查点、压缩、流式历史等关键代码；辅助检查测试目录及具体用例。

包级 README 中详细的行为说明可解释服务契约，但不等于本文逐条验证了它的所有分支。对存储锁、MCP 重连、PTC 运行时等复杂行为，本文明确按“协议及实现设计”介绍，不宣称在本机做过故障注入。

### 2.2 本次快照的静态体量

| 指标 | 结果 | 统计口径 |
|---|---:|---|
| 仓库文件数 | 20,233 | 克隆后遍历文件，排除 `.git` |
| 两层 packages 包清单 | 372 | `packages/*/*/package.json` |
| 非 vendor 的 src TS/TSX 文件 | 3,064 | 路径包含 `src`，扩展名 `.ts` / `.tsx`，排除 vendor 和 `.git` |
| 上述文件总行数 | 539,125 | Python 按 `splitlines()` 计数，包含注释、空行和部分生成/辅助代码 |

这些数字用于说明工程规模，**不是核心算法代码量，也不是逐文件精读覆盖率**。本文重点检查关键路径和可替换边界，没有逐行审计整个仓库。

顶层工作区还包括 `apps/*`、`vendor/*`、`native/system` 及其包、`website`。源码根包声明 ESM、Node `>=24.9.0`、`pnpm@11.28.5`。本次这些是复现环境要求的来源，不是建议用户无条件升级已有项目环境。

### 2.3 如何阅读本文

初次理解系统，可先读第 3、5、7、9 章；关注可靠性，读第 10—12、16 章；关注平台扩展，读第 4、6、13—15、18 章；关注自研选型，读第 21—24 章。附录提供源码导航和核验方法。

来源：[根包清单][S00]、[架构入口][S01]。

## 03 全局架构：入口、控制、模型、执行和状态

### 3.1 分层视图

```text
用户 / IDE / SDK 调用方
          │
          ▼
CLI · Web · Desktop · SDK · ACP 配置入口
          │ 配置组合、会话控制、交互请求
          ▼
Cordis Loader / Context / Service / Events / Effects
          │
          ├── Agent Factory / Agent Loop / Inbox / Scope
          ├── System Prompt / Instructions / Skills
          ├── LLM Runtime / Provider Adapter / Request Recovery
          ├── Tool Registry / Native 或 PTC Executor
          ├── Approval / Permission / Guards / Sandbox Policy
          └── Subagent / Goal / Workflow 等扩展
          │
          ▼
Session 事件日志 ──► Surface / Projection ──► 模型请求与界面历史
          │
          ├── JSONL + zstd 持久化 / 格式迁移 / 写入者协调
          ├── Attachment / Spill 等外置内容
          └── FS / Subprocess / Sandbox / 网络服务
```

上图是按职责重新归纳的概念图，不代表源码存在一个同名的统一中心类。实际调用通过 Cordis 服务和事件连接。

### 3.2 四条需要分开理解的数据通路

**控制通路：** 客户端启动或恢复 Session，向 Agent 输入消息，发出取消、模型选择或权限设置。控制指令改变执行状态，但不一定都作为自然语言消息交给模型。

**模型通路：** 当前模型选择、系统提示、工具 schema 与 Session surface 组合成冻结的请求；适配器输出异步流；成功或失败的尝试在收束时写入对应事件。

**工具通路：** 模型完整响应中的调用被解析、调度、审批与执行；结果按模型原始调用顺序回填；工具带来的 additional context 排在批次结果之后进入后续输入。

**观察通路：** 已提交 Session 事件和实时 assistant 流分别推送给客户端。观察系统需要处理重连、重复和游标，而不能反向任意改写已确定的工具结果。

分开这些通路后，才能理解为何“界面已显示几段 token”不意味着它们已经成为可恢复日志，也能理解为何“收到取消请求”不等于所有进程已经退出。

### 3.3 与常见简单 Agent 的差别

一个教学 Agent 常见实现是：`while → model → tool → append messages`。DSH 保留这一基本结构，但把周围大量隐含假设变成显式协议：准备快照、持久化时机、作用域、重试决策、工具调用与结果配对、子 Agent 生命周期、流式显示和输出投影。

复杂度主要来自这些真实运行条件，而不是循环语句本身。

来源：[总体架构][S01]、[生命周期][S05]、[API 历史监听][S17]。

## 04 Cordis 插件体系：系统如何被装配出来

### 4.1 插件是能力与生命周期的基本单位

Cordis Context 可以提供服务、注册事件监听、挂载子插件及注册可撤销的 effect。DSH 中，Session 服务、LLM 服务、工具注册表、具体执行器和 Agent Loop 都在这一机制内工作。

从使用者角度，`ctx.llm`、`ctx.tools`、`ctx.sessions` 是领域接口；对应插件负责提供实现。依赖注入使消费者在所需服务可用时激活，并在依赖卸载时随生命周期收束。

其工程意义是：更换持久化后端，不应要求改写 Agent 的每一段业务逻辑；更换执行器，不应复制所有工具定义；为某类 Agent 添加策略，应优先在有明确定义的扩展点完成。

但插件体系本身不会消除耦合。若扩展私自读取其他插件内部对象、遗漏 disposer、绕过作用域，仍可能产生生命周期错误。代码库要求注册与撤销成对，就是为控制这些问题。

### 4.2 服务、事件和策略的不同角色

| 机制 | 典型用途 | 应避免的误用 |
|---|---|---|
| 服务 | 提供稳定领域能力，如创建 Session、解析工具集合 | 把全部业务状态堆进无边界全局对象 |
| 普通事件 | 通知状态变化或已确定结果 | 让观察者异常改变已经收束的结果 |
| waterfall / around | 包裹下游过程、拦截决策、调整输入或输出 | 忘记调用 `next()` 而意外截断后续插件 |
| effect / disposer | 将注册、资源和异步清理绑定插件生命周期 | 卸载仅删引用，却留下后台进程或监听 |
| 类型声明合并 | 声明可注入服务和事件参数 | 把编译期类型当作运行时安全隔离 |

waterfall 的返回值可参与 Promise 或 AsyncIterable 等协议，应按照具体事件签名使用，不能一概理解成同步广播。

### 4.3 Profile 与 patch：部署本身也是代码结构

架构文档列出 Web、headless、SDK、SDK minimal、ACP 等组合方式。通常先加载有序 bundle，再加载 profile patch、用户目录 patch 和命令行 `--patch` 覆盖。`sdk-minimal` 使用明确的完整树，不能简单套用“所有 profile 都继承同一 base”的说法。

一个特别容易出错的细节是：**按 id 替换插件配置时，替换的是整块 config，不是任意深度合并。** 如果原来配置有多个字段，后续 patch 只写一个字段，不能假定其余字段继续来自原对象。

配置还允许 `!!js` 等表达式。因此 profile 和 patch 应被视为受信任的可执行装配输入，不应把不可信用户提供的任意配置当成普通静态数据加载。

HMR 也属于组合选择：架构文档说明 base 中的开发热加载与部分部署模式的关闭策略；不能由“支持热加载”推导出所有宿主都默认开启。

来源：[Cordis 入门][S06]、[总体架构的组合规则][S01]、[base patch][S02]。

## 05 关键对象：Agent、Session、Turn、Step、Attempt

### 5.1 对象关系

| 概念 | 含义 | 关键边界 |
|---|---|---|
| Agent Factory | 创建、恢复、管理 Agent 的服务 | 持有启动与销毁过程，不只是 `new` 的封装 |
| Agent | 活跃执行对象 | 有模型选择、收件箱、取消信号、作用域和运行状态 |
| Session | 事件与历史容器 | 可在 Agent 停止后持久存在，并被重新装载 |
| Turn | 一段输入驱动的工作回合 | 可含零个或多个 step，必须有结束原因 |
| Step | 一次模型请求及其工具阶段 | 成功响应如有工具调用，结果供下一 step 使用 |
| Attempt | 同一 step 的一次模型尝试 | 失败恢复可能在同一 step 内重新请求 |
| Scope | 注册和事件路由边界 | 不是操作系统或进程安全边界 |

区分 step 和 attempt 很重要。一次请求因网络或上下文问题重试，并不必然创建新的 step，也不应把同一条用户消息重复追加多次。

### 5.2 零 step 的 turn 为什么合理

Agent 先开启 turn，再运行 pre-step。如果 pre-step 策略拒绝，或初始准备后没有实际工作，可以结束一个没有模型调用的 turn。这仍值得记录：系统曾收到工作、策略作出决定、执行结束，并不是无事发生。

若监控系统把“每个 turn 必须有一条模型响应”作为假设，就会把正常阻断误报为丢失请求。

### 5.3 输入何时被接收，何时被模型看到

Inbox 承接输入，但进入 Inbox 不等于立即写进当前模型请求。在步骤准备期间，系统会领取输入、解析工具和提示，再决定接受或拒绝。正在执行的工具和模型请求也不会因新的普通输入到达而随意重写其既有快照。

这使“用户追加一句话”“子 Agent 发来消息”“工具附带补充上下文”等情况可以在有序边界被处理。

来源：[Agent Loop 主实现][S03]、[Agent Loop 说明][S07]。

## 06 Scope：注册可见性和事件路由如何隔离

Scope 使用对象身份作为不透明 key。子 scope 可以绑定父 scope，绑定关系会做循环检查；重绑定需要持有专门返回的权限句柄。

它的两个方向很容易混淆：

1. **能力可见性向下继承：** 子 scope 的注册视图可以看见祖先层，再叠加自己的层。
2. **事件观察向上开放：** 发给子 scope 的事件，可以由祖先 scope 的监听者接收；子级监听者不会因为其存在而自动收到父级所有事件。

不带 scope 标签的监听者属于全局观察者，还会受原有 Cordis filter 约束。`scopeTarget()` 生成的是路由载体，真正的 Agent 等主题对象通过事件参数传递，避免把路由对象误作领域对象。

### 6.1 为什么对子 Agent 有价值

父产品组合可以统一观察后代 Agent；子 Agent 可拥有自己的工具注册与策略；释放 scope 时，可以撤销该 scope 下注册的资源。这样不必把所有 Agent 都挂在同一个无差别工具列表里。

### 6.2 边界解释

Scope 处理的是应用内的注册和路由。它不能阻止同进程插件直接调用 Node API、读取进程可访问的内存或绕过正常服务。若第三方插件不可信，需要进程、容器或其他独立信任边界，不能单靠 scope 命名。

源码还显式等待 Fiber 的异步清理与 inertia；因此“调用 dispose”与“资源真正静止”是两个需要连接起来的阶段。

来源：[Scope 实现][S08]。

## 07 Agent 主循环：从输入到最终答复逐步拆解

### 7.1 正常路径

```text
输入进入 Inbox
  → turn/start
  → preStep：领取输入、固定模型、resolve tools、assemble prompt
  → agent/pre-step：允许、改写或拒绝
  → step/start
  → prepareRequest：请求配置策略 + llm.prepareCall
  → 提交 system/user 等实际模型输入
  → buildRequest：记录 envelope/context，deriveMessages，冻结
  → 流式调用模型，向界面广播临时 chunk
  → 收束 assistant/message
  → 有工具调用：执行工具批次、写入结果、收集补充上下文
  → step/end
  → 下一 step，或 turn-stopping，或 turn/end
```

### 7.2 为什么模型选择要先固定

preStep 读取当前模型选择并以此解析工具集合。有的工具可见性、图像能力、schema 或上下文预算依赖模型能力。若后续 `agent/request` 中途换成另一模型，前面的解析结果可能已不再成立。

因此实现会验证 request 策略没有把 provider、model 或 reasoning effort 改成不同选择；需要变更时，应在 step 准备前通过模型选择接口完成。

这是一种“先确定执行契约，再构造实际请求”的设计，减少运行中重配置造成的隐式竞态。

### 7.3 为什么 prepareCall 放在输入提交之前

LLM 的 `prepareCall()` 可绑定适配器、解析默认配置和能力。若准备失败或被取消，当前输入不应提前成为“模型已经看到的历史”。代码在准备阶段检查取消，随后才提交实际 system/user 输入。

这使日志更接近实际请求事实，避免出现历史里已经记录用户输入，但该次准备根本没有进入模型调用的混乱。不过真正对外请求是否已经到达服务端，仍需由请求边界、传输错误和持久化检查点共同判断，不能只靠一个状态字段推断。

### 7.4 为什么工具不在流式参数出现时立即执行

本次实现先消费完整 assistant stream，完成消息收束，再提取并执行 tool calls。中途看到的工具名或 JSON 片段仅是流式展示状态。

这避免在参数尚未完整、响应最终失败、或供应商改变终止原因时提前触发外部副作用。代价是少了“模型生成下一个工具参数时，上一个工具已开始”的潜在延迟优化。

### 7.5 结束条件不只有“没有工具”

实现会处理自然完成、策略阻断、max-tokens、取消和错误，并在 finally 路径写入 step/turn 的结束状态。自然停止前还会发出 `agent/turn-stopping`，扩展有机会追加工作，使本来将结束的回合继续。

注意 max-tokens 的分支位于工具执行之前；不能把被截断响应中的调用视为普通完成调用执行。若还有待处理输入，驱动器会按新的运行信号继续调度，而不是无限复用已经 aborted 的控制器。

来源：[agent.ts：preStep / turn / step / prepareRequest / buildRequest][S03]、[生命周期时序][S05]。

## 08 LLM 适配层、请求快照与错误恢复

### 8.1 模型接口不是业务循环本身

LLM Runtime 管理适配器注册、调用准备和流式分发。调用者使用统一契约，供应商差异放在适配器。base 注册原生 DeepSeek 相关适配器；也有其他模型集成入口，但是否实际可用取决于 profile、凭据与相应配置。

`llm/stream` 使用 AsyncIterable，适合逐块传递文本、推理、工具调用和结束状态。适配器存在不等于模型已配置，也不等于某个包中提到的模型名称可以跨版本直接使用。

### 8.2 请求 envelope 有自己的历史

Session 不只记录聊天消息，还记录 `request/header`、`request/context` 等请求事实。header 包含实际配置与工具 schema 等信息；context 记录模型和上下文窗口等相关状态，只有相关内容变化时才写新事件。

构建请求时从已提交 surface 派生消息，并冻结边界消息和请求对象。冻结的作用是防止下游在不留痕情况下改写这一轮输入，不是加密，也不能防止整个受信插件进程做其他事情。

“模型可见就应有日志依据”是设计约束。对复盘而言，重点是能重建逻辑请求表面；它不承诺供应商内部隐藏提示、实际 KV Cache 或随机采样状态也可被复现。

### 8.3 重试属于恢复决策，不是无限重复 HTTP

失败 assistant 尝试会记录为 `assistant/attempt`，再进入 `agent/request-error` 扩展链。允许重试时，循环在同一 step 的下一 attempt 重建请求；已经提交的用户输入不会再次追加。

LLM Retry 插件支持 provider 路由的 normal / always 策略。normal 根据可重试错误集合和次数判断；always 允许持续恢复，但仍受取消、插件生命周期与下游恢复决定约束。不能把“always”理解成永远不可停止。

退避前记录 `llm/retry`，真正开始下一尝试前记录 `llm/retry-started`。状态投影依据 provider 和策略身份计数，在 step/start 或 turn/end 重置。等待期间融合调用取消和插件卸载信号。

### 8.4 失败尝试与有效回答不同

失败 attempt 可以用于诊断，但不应自动成为正常回答加入后续模型对话。取消过程中已有可用文本的流式前缀可能以 interrupted 形式收束；无内容和错误尝试则按 attempt 等路径记录。不同终止情况必须按实现分别处理。

来源：[LLM Runtime][S09]、[LLM Retry][S10]、[Agent 请求构建及流收束][S03]。

## 09 工具系统：定义、策略、执行与结果投影

### 9.1 工具不只是 name + function

工具定义包含模型可见 schema、执行函数、规范输出及呈现规则，还可以声明动态可见性、并发安全性、内容投影、最终内容处理等。

最重要的分离是：

- **canonical value：** 工具产生的无损 JSON 结果，供程序化调用和结构化处理。
- **model content：** 模型实际看到的内容块，可以被渲染、压缩或改写。
- **presentation metadata：** 界面使用的呈现辅助数据，不应偷偷改写模型内容或 canonical value。

这样，模型可以看到紧凑摘要，界面展示结构化卡片，PTC 拿到完整结构值，而不用把所有用途都压成一串文本。

### 9.2 完整管线

| 顺序 | 阶段 | 主要职责 |
|---:|---|---|
| 1 | 创建 execution | 固定调用身份、工具定义相关快照与信号 |
| 2 | `tools/pre-execute` | 返回 allow / deny / ask 等前置决策 |
| 3 | 审批服务 | 将 ask 转成一次性许可或明确拒绝 |
| 4 | guards | 在审批后检查不可被下游放宽的约束 |
| 5 | `tools/execute` 包裹链 | 检查点、超时等围绕实际执行的扩展 |
| 6 | 工具 body | 在合作式取消约束下产生规范结果 |
| 7 | `projectContent` | 将规范值投影为当前模型内容 |
| 8 | `tools/post-execute` | 接受、替换内容、阻断或添加纠正反馈 |
| 9 | 归一化与 `finalizeContent` | 最终内容收束，形成权威结果 |
| 10 | `tools/result` 通知 | 观察已确定结果，不提供再修改结果通道 |
| 11 | Session `tool/result` | 按批次顺序进入可恢复历史 |

不同拒绝或异常路径会跳过不适用的阶段，不能把表格理解成任意失败都必定运行工具 body。最终处理和观察通知有专门的兜底路径。

### 9.3 审批不是字符串提示

当策略返回 ask，执行管线通过 approval 服务发送请求，并区分 `allowed-once`、`rejected`、`cancelled` 和 `unavailable`。没有 approval 服务，或无 Agent 无法路由审批时，降级为拒绝，而不是自动允许。

这与“在系统提示里要求模型先问用户”不同：前者在执行路径上有明确决策，后者只是模型行为约束。两者可以配合，但不能混为同一强度的控制。

### 9.4 后置阻断不等于撤销

post-execute 可以把结果改为错误并提供纠正反馈，但此时工具 body 可能已经修改文件、发起网络调用或创建进程。后置阻断控制的是结果如何被接受，不是事务回滚。

因此带高影响副作用的规则应尽可能在前置策略、guard 和实际执行环境中落实。需要补偿事务时，应由具体工具或业务集成定义，不能指望统一管线替所有工具撤销世界。

### 9.5 观察者错误受到隔离

`tools/result` 的同步异常和异步拒绝被记录为警告，不会反向篡改已确定结果。工具执行对象和最终结果的处理体现了“决策路径”和“观察路径”分离。

来源：[工具契约][S11]、[管线实现][S12]、[工具执行时序][S04]。

## 10 并发调度、超时和取消：怎样控制副作用

### 10.1 默认独占，明确声明才并发

工具的 `isConcurrencySafe(args)` 是并发判定入口。返回 true 才能进入并行安全组；默认独占，参数非法等情况也不应冒险并发。

原生批次调度使用滚动池，并行执行安全工具；独占调用形成屏障。在启动后续工具前重新判断分类，避免仅依赖很早之前的判断。

例如模型一次给出以下调用，假设读取工具声明安全、写入工具声明独占：

```text
read A ─┐
read B ─┴─ 可并行，等待二者结清
write C ─── 独占执行
read D ──── 后续安全组
```

这只是契约示例，不代表所有名字含 read 的工具都自动安全。是否可并发由工具自己的实现、参数与声明决定。

### 10.2 完成顺序和日志顺序分开

并发读 B 可能先于 A 返回，但工具结果按模型调用顺序提交。这能保持 tool_call 与 tool_result 对应关系稳定，让重放与请求重建不依赖机器调度偶然性。

不应因此以为整个批次都是原子事务：A、B 的真实外部效果仍按实际执行时刻发生，日志顺序不能抹掉真实时间差。

### 10.3 取消是一个收敛过程

收到 abort 后，调度器停止启动新工作，等待已启动工作收束，为未开始调用生成对应错误结果。内部调度错误也保留部分结果，而不是把全部已完成工作当作从未发生。

同进程工具依赖协作式 AbortSignal，必须在返回前停止自身工作。工具注册表无法强杀同进程任意 JavaScript；对失控同步计算、外部进程和沙箱代码，需要更低层的进程管理能力。

### 10.4 超时不等于回滚

工具契约可声明超时，超时策略通过执行包裹链落实。超时可以阻止继续等待、触发取消和清理，但先前已经产生的文件或网络效果仍可能存在。

应分别监测“发出取消”“工具返回”“进程确认退出”“资源释放完成”。只用一个 `cancelled=true` 字段代表所有阶段，会掩盖泄漏和重复执行风险。

来源：[原生批次调度][S13]、[工具取消契约][S11]、[管线信号融合][S12]。

## 11 Session：事件日志、Surface 与 Projection

### 11.1 三者解决不同问题

| 结构 | 回答的问题 | 示例 |
|---|---|---|
| 事件日志 | 历史上发生过什么 | turn/start、assistant/message、tool/result、压缩事件 |
| Surface | 下一次模型应该看到什么 | 当前消息节点及替换后的摘要 |
| Projection | 某个领域的当前派生状态是什么 | 子 Agent 目录、重试计数、会话列表元数据 |

事件日志可以保留原始大量历史，surface 则只保留当前有效上下文。因此“日志越来越长”和“模型上下文被压缩”可以同时成立。

### 11.2 append 的职责

Session.append 对事件数据做可序列化的无损 JSON 快照，分配顺序和时间、验证、冻结，再提交。BigInt、非有限数字、Map 等不属于其任意接受的数据类型。

提交前会验证 surface 变化；提交后观察者异常受到隔离，并防止不安全的 append 重入。这里要特别区分：**内存事件提交不是磁盘 fsync。** 持久化由存储监听、flush 和检查点边界完成。

### 11.3 deriveMessages 为什么有缓存

模型请求反复需要从 surface 派生消息。实现根据 content generation 与已有节点缓存结果：只有追加时可增量派生；发生替换则重新计算相关视图。

返回数组是新的，但其中已冻结消息可以共享。这样既减少大历史复制，也避免下游直接修改缓存消息。空 assistant 内容、失败 attempt 等也不会不加区分地进入有效对话。

### 11.4 事件序号与显示帧不同

Session seq 标识已提交事件顺序；实时流式 ordinal 描述未收束 attempt 的展示进度。二者解决不同时间尺度的问题，不能用 token 帧计数代替持久化事件游标。

来源：[Session 实现][S14]、[Assistant 流桥接][S18]、[历史监听][S17]。

## 12 持久化与崩溃恢复：可靠性如何落地

### 12.1 JSONL 存储的结构

官方提供的持久化实现是 JSONL Session 存储，默认使用 zstd，也可配置无压缩。Session 按版本 generation 文件保存；本次逻辑格式常量为 **V4**。包文档中的旧文件名示例不能覆盖实际常量。

存储会区分当前、旧版和未来版本。旧数据可在内存读取时迁移；要写入新格式时生成并验证新 generation，再发布，不直接覆盖源 generation。未来版本拒绝以旧程序随意处理，避免不可逆降级。

### 12.2 语义检查点是关键补充

base 启用 `session-checkpoint-policy`。其重要动作包括：

- 模型流真正向下游构造并读取之前，flush 对应 Session 的完整已记录请求前缀。
- 顶层工具 body 之前，flush 已记录的调用；嵌套调用复用外层持久化边界。
- 在步骤准备等边界推动前一步内容持久化。

检查点失败会阻止下游模型或工具执行。这是一种明确选择：宁可暂时停下来，也不默默在没有持久记录的条件下继续产生效果。

### 12.3 为什么仍然不能 exactly-once

考虑一次写文件工具：

```text
调用已记录并 flush
  → 工具实际写入完成
  → 进程崩溃
  → tool/result 尚未持久化
```

恢复后能够知道该调用已派发，但无法仅靠本地日志证明外部写入到底完成了多少。若对象是第三方 API 或数据库，这一问题更明显。

因此修复逻辑区分：

| 恢复判断 | 含义 | 合理处理 |
|---|---|---|
| `TOOL_NOT_STARTED` | 没有观察到开始执行的派发记录 | 按修复提示判断后续动作；fork 场景还要注意原父会话之后可能继续执行 |
| `TOOL_OUTCOME_UNKNOWN` | 已有派发证据，但没有对应结果 | 先检查实际状态；只对只读或明确幂等操作谨慎重试 |

修复会补齐必要 tool results、step/end 和 turn/end，以恢复协议配对；不会自动把未知写操作重做一遍，也不会声称原工具失败等于原效果不存在。

### 12.4 文件损坏与进程中断是两类恢复

存储层可以处理末尾撕裂的记录或帧，保留可确定的完整前缀；完整但损坏的帧应报错，不能随意跳过。语义层再依据该前缀关闭未完成 turn、补齐工具配对。

这等于先回答“哪些字节可信”，再回答“这些事件构成什么状态”。把两者混在一起，很容易在修复时制造虚假的成功记录。

### 12.5 写入者协调和运维代价

包协议描述了进程内占用与系统锁协调，包括 POSIX 锁及 Windows 对应机制。跨机器共享文件系统仍要关注底层锁语义。日志不会因上下文压缩自动删除；附件和 spill 也需要各自的备份与保留策略。

存储包关于 fsync、回滚到先前文件长度、迁移验证和锁的设计值得进一步故障测试；本文没有在本机验证断电、NFS 或多写入者竞争。

来源：[持久化包协议][S15]、[检查点策略实现][S16]、[恢复修复实现][S19]、[Session 格式常量][S20]。

## 13 上下文工程：提示、指令、预算与压缩

### 13.1 输入不是单一系统提示字符串

系统提示服务可以组合 persona 与插件提供的部分；项目 instructions、Skills、工具 schema、运行时上下文也有各自入口。部署 persona 是组合选择，而不是工具或压缩器私自决定的全局角色。

base 的 agent-instructions 配置有 65,536 bytes 限制；这不等于模型窗口有 64K tokens，也不等于所有仓库说明文件一定全部塞进请求。byte 限额、token 估算和模型窗口是三个不同量纲。

### 13.2 压缩是独立服务，不是循环内不可替换魔法

compaction-basic 通过 pre-step、模型切换和请求错误事件参与执行。这样可以更换摘要器或策略，而不必重写整个 Agent Loop。

本次默认配置包括：自动压缩开启、thresholdRatio 0.8、retainRatio 0.16、headroomTokens 65,536、摘要重试与溢出恢复各有明确上限。实际阈值并不是简单的“窗口到 80% 就压缩”。

设上下文窗口为 `W`、保留给输出的预算为 `R`、额外余量为 `H`，比例阈值为 `a`，保留比例为 `b`，相应配置分支近似写成：

```text
trigger = floor(min(W × a, W - R - H))
retain  = floor((W - R) × b)
并要求 retain < trigger
```

显式 token 配置和按模型策略会改变取值，详细以 resolver 为准。这里的公式用于解释为什么同样 80% 配置，在小窗口模型和大输出预算下可能更早触发，甚至配置无效。

### 13.3 “替换模型表面”而不是“删除历史”

压缩事件包含开始、摘要及结束记录；真正改变模型可见内容的是带 replace 语义的消息，对选定 surface 区间进行替换。原始事件继续留在日志中。

选区需要保持工具调用和结果配对，不应截出一个只有 tool_result 没有 tool_call 的上下文。它处理的是合法 surface 区间，不一定以完整 turn 为唯一最小单位。

压缩期间又有新输入到达时，需要校验源区间，避免把较新的消息错误替换掉。未完成压缩也有相应状态和恢复处理，不能把仅有 start 事件当作摘要已经生效。

### 13.4 四种不同减负手段

| 手段 | 处理对象 | 作用 |
|---|---|---|
| Spill | 单次过大的内容 | 外置内容，模型收到可管理的引用或投影 |
| Tool-result pruner | 已进入历史的超大工具结果 | 保留头尾或紧凑版本，先减少低密度内容 |
| Conversation compaction | 较大范围的对话 | 用摘要替代较旧上下文 |
| Image offload | 超出图像预算的旧图片 | 将对应表面位置替换为占位信息 |

base 的 spill 内联阈值为 12,500 tokens，工具结果裁剪配置为 8,192 chars 阈值、4,096 chars 头部、1,024 chars 尾部。这些属于不同层次，不应相加后当作统一窗口限制。

### 13.5 上下文溢出恢复必须证明有进展

当 provider 报出规范的上下文溢出错误，压缩插件检查 surface replace generation 是否实际推进。只有确有替换进展、没有取消且重试预算允许时，才返回 retry。

即使后续摘要失败，只要前面的无模型裁剪已经留下持久的表面缩减，也可能成为一次重试依据。反之，不能无变化地重复同一个超窗请求。

源码显示该恢复进入同一 step 的请求错误决策；不要将其简化成“必定结束旧 step 再起新 step”。这是应以实际循环实现校正概述性文档的例子。

来源：[压缩子系统][S21]、[压缩配置计算][S22]、[压缩触发与恢复][S23]、[默认 spill / pruner 配置][S02]。

## 14 PTC：让模型通过程序组合工具

### 14.1 PTC 与原生工具调用的区别

原生模式中，模型直接输出多个工具调用。PTC 模式中，模型调用 `run_code`，提交程序，程序通过生成的 SDK binding 调用工具、处理结构值、实现分支和循环，再将重要结果返回。

```text
原生：模型 → read / search / read / ... → 每批结果回模型
PTC ：模型 → run_code(program)
                  ├─ await tools.search(...)
                  ├─ 循环筛选 / 并行读取 / 结构化计算
                  └─ return 精简结果
```

潜在价值是把机械遍历和中间数据处理交给程序，减少每一步都让模型阅读大量结果。是否降低成本或延迟，取决于任务和模型编程正确率，本文没有做基准验证。

### 14.2 默认状态必须说清楚

base 同时登记 native executor 和 PTC executor 条目，但 **PTC executor 被 `disabled: true` 显式关闭，默认使用 native**。存在 `ptc-runtime` 和 workflow 的配置，不等于所有普通工具调用已切换到 PTC。

一个 scope 只应选择一个执行器。启用 PTC 却没有运行时，会阻止启动；运行时卸载不应静默退回另一执行方式。

### 14.3 PTC 仍然经过工具策略

嵌套 SDK 调用复用工具的策略、审批、并发和结果协议。内部调用记录为 PTC dispatch 等专门日志，外层 `run_code` 才是模型工具结果；不能把所有内层结果直接插成模型的顶层工具消息，否则会破坏调用配对。

根 `run_code` 独占执行；嵌套并行安全调用有独立池，文档默认上限 10，独占调用仍形成屏障。PTC 模式下绕过 `run_code` 直接调用原生工具名会被拒绝，保持“宣告的可调用表面”和“实际调用方式”一致。

### 14.4 Node 运行时的三类模式

| 模式 | 语法及生命周期 | 内存是否跨调用 |
|---|---|---|
| `per-call` | 新进程执行 async function body，支持 await / return | 否 |
| `vm-script` | Script 求值，可顶层 await，动态 import | 默认否；`scriptPersistence` 可显式开启 |
| `persistent-in-memory` | 每个 owner 的保留 REPL，采用对应声明规则 | 是，但仅运行时内存 |

默认 `per-call`。文档列出 120 秒默认执行期限、600 秒上限、512 MiB V8 old-generation 限额等。期限包括嵌套工具和审批等待，实际返回还可能需要额外清理时间；V8 old-generation 也不是整个进程总内存上限。

### 14.5 持久内存不是 Session 持久化

保留变量可能在取消、超时、进程退出、权限变更、Agent 释放或运行时卸载时清空。恢复 Session 或 fork Agent 不会恢复 JS 堆对象。

因此程序必须能重新读取文件、重建句柄。关键业务结果应进入可恢复外部状态，不能只留在 REPL 变量中。

`node:vm` 上下文本身不是安全沙箱。Node 全局、文件、网络和子进程访问由实际进程执行环境与所选 sandbox 约束。程序失败也不代表已调用的工具没有副作用，更不应自动原样重放整个程序。

来源：[PTC 执行器][S24]、[Node PTC Runtime][S25]、[默认执行器选择][S02]。

## 15 Subagent 与 Workflow：如何组织多 Agent 工作

### 15.1 领域管理器与执行后端分开

Subagent 服务负责父子关系、创建/恢复、收件箱、消息、生命周期和结果交付。provider 负责子任务如何启动，可提供一次性 start 或可继续的 prepareContinuable 等能力。

base 挂载 in-process 的 spawn 与 fork 后端，以及委派和控制工具。in-process 是实现选择，意味着共享宿主进程资源；不同 Session 和 scope 仍不等于进程级隔离。

### 15.2 Spawn 与 Fork

| 维度 | Spawn | Fork |
|---|---|---|
| 历史起点 | 新的任务上下文 | 父会话已完成 turn 的平衡前缀 |
| 当前未完成父 turn | 不自动继承 | 明确排除 |
| 后续父会话变化 | 不自动同步 | 不自动同步；fork 是一次性快照 |
| 适用任务 | 独立检索、隔离分析主题 | 接续已有讨论、带背景的审查 |
| 权限理解 | 按子 Agent 组合决定 | 继承历史不等于复制父 Agent 的全部权力 |

fork 在父会话还没有完成任何 turn 时，其历史种子可以为空。父 Agent 局部限制也不能想当然认为已经原样复制到子 Agent；需要检查子 preset、scope 和策略组合。

### 15.3 消息不是全局聊天室

子系统约束可消息关系，例如直接父与可继续子 Agent；消息在运行中的合适 step 边界 steer，在 idle 时可唤醒，必要时 cold resume。它并非允许任意 session id 互发消息。

interrupt 可以发起取消并保留待处理工作，但不是等待所有后代立即静止的证明。释放父子资源时需要按生命周期关系收敛，避免父级销毁后留下无人管理子任务。

### 15.4 结果和历史应分别处理

子 Agent 有自己的完整历史。父 Agent 通常接收结果或完成通知，而不是自动吸收全部子历史，否则多 Agent 很快变成上下文膨胀器。

Fork 可通过相同历史前缀提高潜在 KV Cache 复用机会，但是否命中仍取决于供应商、模型、提示和 schema 的实际一致性，不能承诺“fork 必然零成本”。

### 15.5 Workflow 的位置

workflow-ptc 允许程序组织子 Agent 调用和结果处理。它与直接调用 subagent 工具不同：脚本可以持有中间结果并决定如何汇总，父模型只收到最终值和相关状态。

这适合“先并行若干独立研究，再按结构化结果决策”的任务。若工作间存在共享文件写入冲突，仍需显式协调，不能因为叫 workflow 就自动具有分布式事务语义。

仓库还有 experimental agent-team 等包；包存在不等于默认启用，本文没有把实验团队协作描绘成所有部署的基础能力。

来源：[Subagent 子系统][S26]、[Fork 后端][S27]、[Workflow PTC][S28]、[默认子 Agent 配置][S02]。

## 16 权限与沙箱：从意图约束到实际执行

### 16.1 至少四种控制不能混成一个开关

| 层次 | 解决的问题 | 不能单独保证什么 |
|---|---|---|
| 提示和 plan-mode 说明 | 告诉模型当前应该怎么做 | 不等同于 OS 拒绝写文件 |
| 工具策略与 guard | 某个调用是否可以进入 body | 无法约束绕过工具的受信插件任意代码 |
| Approval | 当前用户是否授权一次动作 | 不能撤销过去效果，也不能证明命令内容无风险 |
| Sandbox / FS / Subprocess 后端 | 实际文件、进程等执行能力边界 | 不应被视为覆盖所有风险的完整安全保证 |

### 16.2 默认权限组合

base 在未显式覆盖环境时采用 workspace-write，并通过 approval ask 参与执行控制；danger-full-access 对应 never。permission-presets 还定义 read-only 等预设。

预设服务处理用户可选组合、会话固定事实和配置关系，实际限制由底层服务执行。已有会话恢复时也不能简单套用新的全局默认：权限需要依据会话记录与明确变更处理。

可选 auto 模式有单独的集成和准入要求，不能因为发现相关实验包，就描述成默认自主批准所有高权限动作。

### 16.3 文件系统与子进程应属于同一个执行世界

架构说明强调 fs 与 subprocess provider 的一致性。如果读文件来自远端容器，而 shell 在本地宿主运行，模型看到和实际修改的对象就可能完全不同。

因此替换后端时，应一起检查路径、cwd、进程、沙箱、附件及权限策略。只替换一个服务名称不等于完成远程执行适配。

### 16.4 Windows 与 Unix 是不同后端路径

base 对 Bash 与 PowerShell 工具和沙箱有平台条件。不要把 Linux 上的命令行为、信号和锁机制直接外推到 Windows。跨平台支持需要相应测试和实际部署验证，而不是只看 TypeScript 可编译。

### 16.5 项目的官方成熟度声明

官方 `SAFETY.md` 明确将其定义为 experimental developer-preview，未经过安全审计，不能视为生产就绪或安全保证。这个状态应成为选型判断的一部分。

对企业部署，合理的后续工作是验证所需平台的沙箱、权限、凭据和故障恢复，再决定隔离方式。本文不据源码组织良好就推断安全已被证明。

来源：[权限预设协议][S29]、[base 平台与权限配置][S02]、[官方安全说明][S30]、[执行世界说明][S01]。

## 17 Skills、Instructions 与 MCP：三类扩展知识入口

### 17.1 Instructions 与 Skill 的差异

Instructions 更偏向当前仓库或目录应遵循的工作约定；Skill 是可发现、可按需加载并用于某项工作的能力说明。二者都可能影响模型行为，但不等同于新增底层权限。

Skill 服务将目录摘要与正文读取分开。模型可先看到名称和简介，再在需要时加载正文，这样减少把全部技能说明永久放进上下文的成本。

### 17.2 Skill 可见性是有规则的

全局层和 scope 层叠加，近层可覆盖远层；同层还有 provider 和注册顺序等排序规则。`modelInvocable` 与 `userInvocable` 分开，存在四种组合，并不是“Skill 在磁盘上就一定可以被模型自动调用”。

发现失败会标记目录不完整，避免把“暂时读失败”错误解释成“技能不存在”。缓存按 cwd、scope 和 revision 等维度组织，并有容量限制与失效规则。

### 17.3 MCP 是外部能力适配，不是内部全部架构

MCP Client 支持 stdio 和 Streamable HTTP 等连接；外部工具经过命名空间映射进入 DSH 工具系统。没有默认开启的 MCP server，用户需要明确配置。

同名工具、服务器重连、工具列表变化、输出类型和服务端 instructions 都需要处理。因此 MCP 集成不只是“把 JSON schema 转发一下”。

### 17.4 工具目录更新要避免半旧半新

包协议描述了 server generation 的整组替换，避免同一服务部分工具已更新而另一部分仍旧。工具名规范化还要防止不同原始名字映射成冲突名称。

断线时目录可暂时保留，但调用会失败；重连采用退避与次数上限。目录存在不等于连接健康，产品 UI 和模型错误信息都应区分二者。

### 17.5 输出与信任边界

MCP 的文本、图像、资源链接等输出需要投影成当前模型和附件系统可接受的内容。不是所有 MCP 类型都能无损以原生多模态形式送进任何模型。

服务端 instructions 会参与提示构建，应把它视为来自该服务器的受信程度相关输入，而非自动成为高优先级安全规则。stdio 环境变量有敏感项过滤和显式配置合并，但这不等于第三方 server 已被完全隔离。

来源：[Skill 协议][S31]、[MCP Client 协议][S32]、[instructions 默认配置][S02]。

## 18 客户端、Gateway 与流式重连

### 18.1 为什么需要两套历史处理

界面既要看到刚生成的 token，也要能在刷新后恢复已完成消息。前者来自实时 assistant stream；后者来自已提交 Session 日志。

若只保存每个 token，日志体积和高频持久化成本会很大；若只等完整消息再显示，交互延迟会变差。DSH 使用临时流帧加收束日志，接受硬崩溃可能丢失尚未收束 attempt 的显示前缀这一边界。

### 18.2 重连的竞争窗口

典型问题是：先取历史、再订阅事件，中间可能漏消息；先订阅、再取历史，则可能重复。

历史监听实现采用先监听并缓冲、再建立游标和基线的方式。已在基线前的事件丢弃重复，后续事件按预期 seq 校验；检测到缺口会报出，而不是无声拼接一段不完整历史。

实时 assistant 流还维护 ordinal baseline，利用 startedAfterSeq 等信息关联 durable log 和未结算 attempt。两者共同避免“历史已经包含完整消息，界面又把先前 token 再拼一次”的显示错误。

### 18.3 Gateway 的流式复用

源码存在 `/api/remote.mux` WebSocket 多路复用协议，以流 id、generation 和 next/result/rejected/cancel 等消息组织远端调用。

它的意义是让多个长流和控制动作共享连接，并明确处理取消及最终结果。它不是对公网暴露服务的自动授权依据；是否暴露、如何认证与部署，需要继续检查宿主配置和接入层。

### 18.4 Desktop 与 SDK 的复用

架构文档描述 Desktop 用 Electron 宿主启动 CLI runner 的 Web 组合并提供打包资产；SDK 也通过同版本 CLI 运行相应 profile，而不是重新实现一个缩小版 Agent Loop。

这种复用减少行为漂移，但也带来版本耦合：SDK、CLI、协议和配置组合需要匹配。SDK 的 resumeSession 恢复历史与中断状态，不恢复所有外部 shell、浏览器进程或 PTC 内存。

来源：[历史监听实现][S17]、[Assistant 流桥接][S18]、[Mux 协议][S33]、[SDK 客户端][S34]、[宿主架构][S01]。

## 19 默认能力清单：不要把仓库目录等同于产品开关

下表以本次 base 组合为主，具体 Web 或其他 profile 还可以覆盖它。

| 能力 | 本次 base 状态 | 解释 |
|---|---|---|
| Agent Loop / Session / LLM | 挂载 | base 的启动 agents 数组为空，实际会话可由入口按需创建 |
| JSONL + zstd 持久化 | 配置 | home sessions 路径，配合检查点策略 |
| 原生工具执行器 | 启用 | 普通工具调用默认路径 |
| PTC 执行器 | 显式关闭 | 需切换配置，不能由 runtime 存在推断已启用 |
| PTC Node Runtime / Workflow | 挂载相关能力 | 与普通执行器选择是两件事 |
| 文件编辑与搜索 | 挂载 | `apply_patch` 额外工具默认 false，不代表所有编辑能力关闭 |
| Shell | 按平台 | Unix Bash、Windows PowerShell 对应路径 |
| 自动上下文压缩 | 挂载 | 由 compaction-basic 默认配置决定实际触发 |
| Spill / Tool-result pruner / Image offload | 挂载 | 在不同层次减少模型输入压力 |
| Spawn / Fork 子 Agent | 挂载 | 独立 Session 与 scope，后端在同一进程 |
| Skills | 挂载 | 发现目录不等于所有 Skill 正文常驻上下文 |
| MCP | 有服务入口 | 默认无实际配置的外部 server |
| Web search / fetch | base 有默认组合 | Web 可按 Agent preset 重组，凭据和网络仍决定可用性 |
| Session query SQLite | 配置为内存、openAt never | 不能据目录推断默认全量全文索引已经持续打开 |
| Goal / Plan / Todo | 挂载相关域和工具 | 与普通自然语言提示及执行许可分开 |
| Experimental agent teams / auto review 等 | 仓库存在 | 需核对具体组合；不作为默认能力承诺 |

### 19.1 遥测配置的实际含义

base 对会话遥测明确配置反馈触发模式及禁用环境变量，注释说明普通活动不会自动触发 Session 前缀采集。它还定义匿名用户标识、批次大小和关闭期限。

这只能支持对该配置与链路的说明，不能扩大成“整个产品没有任何其他网络请求”或“所有 profile 永远一样”。上线评估还应检查实际生效配置和端点行为。

来源：[完整 base patch][S02]。

## 20 五个场景，把抽象机制串起来

### 20.1 场景 A：读取代码、修改文件、运行测试

用户输入后开启 turn。preStep 决定当前模型、工具和提示；请求准备成功后提交输入；模型完整返回读取工具调用；安全读取可以并发，但结果按原调用序列写回。

下一 step 模型决定编辑。调用先过策略、审批和 guard，在 body 前持久化调用边界。工具修改文件并返回 canonical value，随后投影成模型文本、经后置策略收束。再下一 step 模型调用 shell 测试，最终给出解释。

这期间模型调用和工具调用并不是一个原子操作。若编辑后测试失败，历史应保留失败事实，Agent 再决定修复，而不是让系统隐藏中间错误。

### 20.2 场景 B：模型流输出一半网络断开

界面可能已经显示临时 token。流收束为失败 attempt 后，request-error 扩展链决定是否重试。重试仍是该 step 的新 attempt，用户输入不会再提交一次。

如果进程直接崩溃，未收束流帧可能丢失。产品应从 durable history 重建，再建立新执行，而不是承诺恢复每个已显示字符。

### 20.3 场景 C：工具完成写入但结果未落盘

持久化前的调用记录让系统知道存在待完成调用；恢复时补一个 outcome unknown 结果并关闭未完成结构。Agent 应检查文件或远端状态，再决定继续。

这比盲目再次执行更适合有副作用的场景，但仍需要工具设计提供幂等键、状态查询或补偿机制。

### 20.4 场景 D：任务太长触发压缩

pre-step 估算压力，先后可能裁剪大结果、外置内容和生成摘要。摘要替换选定模型 surface，旧事件保留。后续请求从新的 surface 构建，header/context 保留实际配置。

若 provider 仍报超窗，只有压缩产生实际表面进展且预算允许才重试。这避免死循环地提交同一份超大上下文。

### 20.5 场景 E：派生三个子 Agent 并取消其中一个

管理器创建独立子 Session 和 scope，选择 spawn 或 fork。父模型接收启动或结果通知，子历史不自动全部回流。

取消某个 child 会发起其当前执行收敛，消息是否保留、是否可继续由具体接口决定。父级释放也要管理后代资源；共享工作区写冲突仍需业务协调。

以上为根据源码机制编排的解释性场景，不是本次实际运行记录。它们的价值在于帮助读者检查自己的实现是否遗漏了某个边界。

## 21 架构收益、代价与评估边界

### 21.1 最值得借鉴的设计

**第一，组合能力从运行时逻辑中显式抽离。** 产品差异可以通过 profile 和插件树表达，减少在核心执行路径中堆积宿主条件分支。

**第二，把审计历史与当前模型输入分开。** 长任务既需要保留过程，也需要压缩上下文。用一个 messages 数组同时承担两者，会产生保留历史与节省窗口之间的冲突。

**第三，把请求、工具和持久化边界连接起来。** 仅有事件总线或数据库还不够，必须说明外部副作用前哪些记录已经可靠落盘。

**第四，原生调用与程序化调用共享工具协议。** PTC 提供额外调度方式，也需要遵守审批、权限和日志规则。

**第五，明确承认恢复的不确定结果。** 面对外部副作用，unknown 比编造成功或失败更可靠。

### 21.2 可组合性带来的工程成本

插件数量增加后，团队需要维护依赖关系、装配顺序、作用域和卸载协议。事件与 surface 分离后，事件版本、投影缓存和替换范围也需要持续治理。可替换接口只有在契约测试和真实部署中保持一致，才能转化为可靠扩展能力。

### 21.3 静态研究不能替代评测

本文没有测量代码生成质量、真实任务成功率、token 成本、长任务稳定性、并发吞吐和安全强度。这些指标需要在固定任务集、模型配置、预算和运行环境下验证，不能依据包数量或架构图推断。

对自研编码助手，最实际的用法是逐项核对是否已有等效能力、失败边界是否明确，再按证据确定改进优先级。

## 22 工程质量：已有信号与未验证部分

### 22.1 仓库可见的工程投入

根包脚本包含 typecheck、lint、覆盖率分区、单元、e2e、snapshot、Web、性能和跨平台相关检查入口。Agent Loop 测试目录也覆盖取消、请求冻结、请求重建、资源生命周期、错误恢复等主题。

例如 cancel.spec.ts 的用例名称直接涉及：取消发生在唤醒窗口、保留 Inbox、dispose 时的新输入、流式前缀收束、assistant/message 观察者触发取消等。这说明项目把真实异步边界作为测试对象，而不是只测试 happy path。

但“有测试”与“本次通过所有测试”是不同结论。本文只检查了结构及部分用例，没有运行该测试套件，也没有拿静态覆盖率配置当作真实覆盖结果。

### 22.2 最值得补做的验证

| 维度 | 推荐实验 | 观察证据 |
|---|---|---|
| 请求可重建 | 记录供应商实际 request，再从日志重建 | system、tools、messages、参数的一致性 |
| 工具顺序 | 让并发工具随机延迟并混入独占工具 | 屏障、日志顺序与 call/result 配对 |
| 崩溃恢复 | 在调用 flush 后、body 后、result flush 前强制退出 | NOT_STARTED / UNKNOWN 的区分是否合理 |
| 存储故障 | 注入写满、写失败、末尾截断、损坏完整帧 | 是否 fail-closed，是否只修复可信前缀 |
| 压缩 | 长对话、大结果、图像预算和模型切换 | 配对保留、surface 区间、摘要失败恢复 |
| 取消 | 同步循环、子进程树、审批等待、卸载 | 收敛时延、残留进程、后续任务能否恢复 |
| 多 Agent | 子级写冲突、父级退出、冷恢复 | 目录一致性、消息权限与资源释放 |
| 跨平台 | 同任务运行于 Windows / Linux | 路径、shell、锁、沙箱实际表现 |
| MCP | 重连中调用、工具变更、命名冲突 | 整组更新、错误可解释性和权限保留 |
| 成本 | 原生与 PTC 对照相同任务 | 成功率、总 token、重试次数、端到端耗时 |

### 22.3 静态研究中的主要不确定性

一是不同 profile 和 preset 可以覆盖 base，本文不能代表用户未来安装时的全部生效配置。二是包协议比代码摘要详细，但复杂平台后端需要实测确认。三是 developer preview 接口可能变化，本文锚定提交而不是承诺兼容未来版本。

来源：[根包测试脚本][S00]、[取消测试][S35]、[请求冻结测试][S36]。

## 23 自研平台如何吸收这些设计

### 23.1 优先补可靠性边界，再扩展功能数量

如果已有编码助手，不必立即改成 300 多个包。优先确认下面几件事是否已经有清楚答案：

1. 每次模型实际看到的输入能否从记录重建？
2. 工具派发前是否已有可靠调用记录？
3. 已执行但结果未知时是否会盲目重做？
4. 取消后能否确认子进程和后台任务已清理？
5. 压缩是否保留审计历史与工具配对？
6. 客户端刷新是否可能漏帧、重复消息或误认为任务完成？

这些往往比再加一个搜索工具更影响长任务可信度。

### 23.2 分阶段实施建议

| 阶段 | 建议交付 | 验收标准 |
|---|---|---|
| A：统一事件与请求边界 | 版本化事件、调用 id、请求快照、配对校验 | 随机工具完成顺序下可重建相同逻辑历史 |
| B：建立恢复闭环 | flush 检查点、未知结果语义、末尾修复 | 故障注入后无伪造成功、无盲目重复副作用 |
| C：治理扩展 | 工具契约、策略管线、scope、生命周期 | 卸载扩展不残留注册和执行任务 |
| D：改进上下文 | surface 替换、spill、分层压缩 | 长任务窗口可控且调用配对合法 |
| E：程序化与多 Agent | PTC 或 workflow、子任务管理 | 同任务评测证明收益，取消与权限不退化 |

这是本文的实施建议，不是 DSH 官方路线图。阶段先后可按现有系统情况调整。

### 23.3 何时适合直接基于 DSH 扩展

如果团队希望开放组合、愿意采用其 Cordis/TypeScript 技术栈、需要多入口复用和可替换执行器，可以通过固定版本原型验证它作为基础设施的适配度。

如果已有成熟状态模型、协议和部署平台，先抽取事件与工具边界思想可能比整体替换更经济。迁移成本主要在既有会话格式、权限、远程执行世界和 UI 流协议，而不是把一个 model SDK 换成另一个。

## 24 最终架构结论

DSH 最有研究价值的地方，是将编码 Agent 的“不确定执行过程”拆成有明确协议的准备、请求、工具、记录、投影、恢复和观察环节。

它的核心循环仍然直观，但周围工程边界相当丰富：插件装配提供可替换性，Session surface 支持长任务上下文管理，工具管线统一策略和副作用，检查点与 repair 让恢复有证据，临时流与持久历史分离让实时交互和重连可以各自优化。

同时，其成熟度与安全性需要按 developer preview 对待。插件化不是隔离证明，日志不是任意副作用的 exactly-once，取消不是回滚，fork 不是完整权限继承，持久 REPL 不是可恢复状态，存在某个包也不是默认功能承诺。

对架构学习，建议把“每个边界记录了什么、何时生效、失败后能证明什么”作为阅读主线。对实际选型，应在固定版本上完成目标平台、任务集和故障路径验证。

## 25 附录 A：源码导航与证据索引

下面链接均固定到本次提交。表内符号用于快速定位；部分复杂子系统主要参考其包级协议，已经在正文说明实测边界。

| 编号 | 路径 / 主题 | 建议重点 |
|---|---|---|
| S00 | [package.json][S00] | 工作区、环境、构建与测试入口 |
| S01 | [docs/architecture.md][S01] | 插件、组合、Session 和执行世界 |
| S02 | [bundle/base/cordis.patch.yml][S02] | 默认能力、关闭项、阈值与平台条件 |
| S03 | [agent-loop/src/agent.ts][S03] | `ReactLoopAgent`、`preStep`、`turn`、`step`、`prepareRequest`、`buildRequest` |
| S04 | [tool-execution-pipeline.md][S04] | 工具管线次序和 PTC 对应关系 |
| S05 | [agent-lifecycle.md][S05] | 生命周期时序 |
| S06 | [cordis-primer.md][S06] | service / event / effect |
| S07 | [agent-loop/README.md][S07] | 循环服务协议 |
| S08 | [scope/src/index.ts][S08] | `createScope`、`scopeTarget`、父链及异步释放 |
| S09 | [llm/src/index.ts][S09] | 适配器与准备调用 |
| S10 | [llm-retry/src/index.ts][S10] | 重试投影、退避和取消 |
| S11 | [tools/src/contracts.ts][S11] | 工具输出、并发、取消、可见性契约 |
| S12 | [tools/src/pipeline.ts][S12] | prepare / dispatch / complete / finish / serviceAsk |
| S13 | [tools/src/batch.ts][S13] | 滚动并发池、屏障和有序提交 |
| S14 | [session/src/index.ts][S14] | append、commit、deriveMessages、flush |
| S15 | [session-persistence-jsonl/README.md][S15] | 日志格式、损坏边界、锁和迁移协议 |
| S16 | [session-checkpoint-policy/src/index.ts][S16] | 请求和工具副作用前的持久化检查点 |
| S17 | [history/listener.ts][S17] | 先订阅再建基线、seq 校验 |
| S18 | [history/assistant-stream.ts][S18] | 实时流和持久历史衔接 |
| S19 | [session/src/repair.ts][S19] | 恢复未知结果与结构补齐 |
| S20 | [session/src/types.ts][S20] | 当前格式 V4 |
| S21 | [subsystems/compaction.md][S21] | surface 替换与配对 |
| S22 | [compaction-basic/src/config.ts][S22] | 阈值公式和默认策略 |
| S23 | [compaction-basic/src/index.ts][S23] | 压力、切换和溢出恢复 |
| S24 | [tool-executor-ptc/README.md][S24] | 程序化调用与嵌套工具协议 |
| S25 | [ptc-runtime-node/README.md][S25] | 进程执行模式、限额与内存边界 |
| S26 | [subsystems/subagent.md][S26] | 子 Agent 管理器契约 |
| S27 | [subagent-fork-in-process/README.md][S27] | completed-turn 种子边界 |
| S28 | [workflow-ptc/README.md][S28] | 程序组织子任务 |
| S29 | [permission-presets/README.md][S29] | 权限配置与会话固定 |
| S30 | [SAFETY.md][S30] | 官方成熟度与安全边界声明 |
| S31 | [skill/README.md][S31] | 分层发现、调用属性和缓存 |
| S32 | [mcp-client/README.md][S32] | 连接、重连、工具目录与输出 |
| S33 | [gateway/stream-protocol.ts][S33] | WebSocket mux 协议 |
| S34 | [sdk/client/README.md][S34] | SDK 启动、恢复和外部状态边界 |
| S35 | [cancel.spec.ts][S35] | 取消窗口、Inbox 与收敛测试 |
| S36 | [request-freeze.spec.ts][S36] | 请求冻结相关测试 |

## 26 附录 B：复核方法与后续阅读顺序

### 26.1 复现调研基线

```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
git checkout ce7073d51254c26cd63b22a60835b8f948deb447
git show -s --format=fuller HEAD
```

以上仅克隆和查看源码；不需要提供 API Key。准备启动或测试前，再按该提交的官方文档核对 Node、pnpm、native 构建和平台依赖。不要把本文的静态研究当作安装成功记录。

### 26.2 推荐源码阅读路线

```text
docs/architecture.md
  → packages/bundle/base/cordis.patch.yml
  → packages/core/agent-loop/src/agent.ts
  → packages/core/tools/src/contracts.ts + pipeline.ts + batch.ts
  → packages/core/session/src/index.ts + repair.ts
  → packages/session/session-checkpoint-policy/src/index.ts
  → compaction-basic/src/config.ts + index.ts
  → subagent / PTC / MCP / API history 的领域协议与实现
  → 对应测试，最后才做目标场景运行验证
```

### 26.3 统计口径示意

```python
from pathlib import Path
root = Path('deepseek-harness')
files = [p for p in root.rglob('*') if p.is_file()
         and '.git' not in p.relative_to(root).parts]
packages = list(root.glob('packages/*/*/package.json'))
sources = [p for p in files
           if 'vendor' not in p.relative_to(root).parts
           and 'src' in p.relative_to(root).parts
           and p.suffix in ('.ts', '.tsx')]
lines = sum(len(p.read_text(encoding='utf-8').splitlines()) for p in sources)
print(len(files), len(packages), len(sources), lines)
```

该统计会随提交、生成文件和本地构建产物变化。本次在未安装/构建的克隆快照上统计。

**交付边界：** 本文是一份固定源码快照的技术研究与架构讲解，包含事实、解释和建议；不是官方设计文档的全文复制，不是性能评测报告，也不是安全审计。读者可以用固定源码链接逐项复查关键结论。

[S00]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/package.json
[S01]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/architecture.md
[S02]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/bundle/base/cordis.patch.yml
[S03]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/agent-loop/src/agent.ts
[S04]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/tool-execution-pipeline.md
[S05]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/agent-lifecycle.md
[S06]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/cordis-primer.md
[S07]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/agent-loop/README.md
[S08]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/scope/src/index.ts
[S09]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/llm/llm/src/index.ts
[S10]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/llm/llm-retry/src/index.ts
[S11]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/tools/src/contracts.ts
[S12]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/tools/src/pipeline.ts
[S13]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/tools/src/batch.ts
[S14]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/session/src/index.ts
[S15]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/session/session-persistence-jsonl/README.md
[S16]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/session/session-checkpoint-policy/src/index.ts
[S17]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/api/session-controller/src/history/listener.ts
[S18]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/api/session-controller/src/history/assistant-stream.ts
[S19]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/session/src/repair.ts
[S20]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/session/src/types.ts
[S21]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/subsystems/compaction.md
[S22]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/compaction/compaction-basic/src/config.ts
[S23]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/compaction/compaction-basic/src/index.ts
[S24]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/tool-executor-ptc/README.md
[S25]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/ptc-runtime/ptc-runtime-node/README.md
[S26]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/docs/subsystems/subagent.md
[S27]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/subagent/subagent-fork-in-process/README.md
[S28]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/workflow/workflow-ptc/README.md
[S29]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/interaction/permission-presets/README.md
[S30]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/SAFETY.md
[S31]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/skill/skill/README.md
[S32]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/mcp/mcp-client/README.md
[S33]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/api/gateway/src/stream-protocol.ts
[S34]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/sdk/client/README.md
[S35]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/agent-loop/tests/cancel.spec.ts
[S36]: https://github.com/deepseek-ai/deepseek-harness/blob/ce7073d51254c26cd63b22a60835b8f948deb447/packages/core/agent-loop/tests/request-freeze.spec.ts
