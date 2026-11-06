# ADR-0001：并行 Agent 编排，先 Skill 后 Plugin

- 状态：Accepted（待你确认）
- 日期：2026-11-05
- 决策范围：Cindy、独立 Claude Code、DeepSeek Harness，以及其他加载 `SKILL.md` 的宿主
- 关联文档：[Phase 1 实施计划](./phase-1-plan.md) · [Skill 草稿](../skills/parallel-orchestration/SKILL.md)

## 1. 背景与约束

需求来自一个真实工作方式：任务耗时长时，希望另开一个窗口做别的事，同时让两个窗口共享历史记忆，最后把结论合回来。最终要能公开到 GitHub，供他人安装或参考。

已知能力（来源见 §6 证据表）：

- **Cindy**：Lead/Worker 多 Agent 调度；Worker 是独立模型 + 独立上下文的完整会话；记忆是跨 Harness 共享的 Markdown 分片；侧边栏页签可拖出为独立窗口。
- **Claude Code 2.1.284（本机实测）**：原生 `claude --bg "<task>"` 启动后台会话并返回 id；`claude agents --json` 查状态；`claude logs/attach/stop/rm` 管理。`/subtask` 不在 stock 2.1.284 中，可能是 Cindy 或插件提供的命令。
- **DSH**：`subagent` 工具启动后立即返回 id，完成后通知；`workflow` 可做多 Agent 扇出；插件是 `apply(ctx)` 生命周期 + `ctx.tools.register(defineTool(...))`；技能是 `SKILL.md` 指令集，用户级目录为 `~/.agents/skills/`。

一句话结论：**“并行派发”已经不是缺口，“共享记忆”也已经有载体；真正缺的是两者之间的编排协议，以及协议之上那层可持久化、可自动化的运行时。**

## 2. 核心缺口：能力、知识、体验三层拆开看

| 层 | 需求 | 现状 | 是否缺口 | 缺口类型 |
|---|---|---|---|---|
| 运行时派发 | 启动一个不阻塞主会话的 Worker | Claude Code 2.1.x 有 `--bg` 后台会话；DSH 有 `subagent`/`workflow` | 否 | — |
| 上下文继承 | Worker 知道主会话在做什么 | `--bg` 会话可带 brief，但是新上下文；DSH Worker 独立上下文但可由 Lead 写 brief | 部分 | 知识（怎么给上下文） |
| 共享记忆载体 | 两个窗口/两个 Harness 都能读到 | Cindy 的 Markdown 分片 | 否 | — |
| 共享记忆契约 | 写什么、何时写、写到哪、冲突怎么办 | 只有“分片”这个载体，没有 schema 和写回时机 | **是** | **知识** |
| 编排策略 | 什么任务该并行、切几刀、谁合并 | 没有成文规则；模型默认要么不并行，要么一把开五个 | **是** | **知识** |
| 持久状态 | 关掉窗口后还能查到“跑到哪了” | 后台任务句柄通常只活在会话里 | 是 | 运行时 |
| 完成即写回 | Worker 结束时主会话可能空闲，谁来落盘 | Skill 只在模型被调用时生效 | **是** | 运行时 |
| 一键开窗/看板 | 点一下把并行任务放到另一个窗口 | 宿主有拖拽页签，但没有“并行任务”语义 | 是 | 宿主/UI，优先级最低 |
| 跨 Harness 归一化 | Claude Code 和 DSH 用同一套“派发/查询/收集” | 两套原语，参数和语义不同 | 是 | 运行时（且很难） |

把表拆完后，结论很清楚：

1. **缺少的第一件事是“在什么场景下派发、怎么切、派发后怎么汇报、结论怎么落回共享记忆”的编排知识。** 这正是 Skill 的定义域：一份 Markdown 指令集，不注册新工具、不写 TypeScript，改一版就能验证。
2. **缺少的第二件事是“持久化 + 自动化 + 宿主体验”。** 这是 Plugin 的定义域：注册工具、监听事件、管理生命周期。
3. **“另开一个窗口”不是编排问题，是宿主 UI 问题。** 后台 Worker + 共享 Markdown 已经能做到“主会话继续干别的”；窗口只是呈现方式。把窗口当第一优先级，会把一个两周能验证的流程拖成一个月的 UI 工程。

## 3. 决策

**采用“Skill 先行、Plugin 补齐”的组合路线。**

- **现在做 Skill v0.1**：把编排判断、任务拆解、Worker brief、Harness 适配、共享记忆 schema、汇报与失败处理写成 `SKILL.md` + 参考模板。
- **暂不写 Plugin 代码**：先跑 2–3 周真实任务，记录“哪些摩擦是知识解决不了的”。只有被反复证明属于运行时缺口的点，才进入 Plugin v0.1。
- **最终形态是组合，不是一个包**：Skill 负责“判断与协议”，Plugin 负责“能力与状态”，两者通过一个版本化的记忆契约解耦。

### 3.1 为什么不是 Plugin 先行

1. **Plugin 会把“错误的编排”自动化。** 现在模型对“何时并行”没有稳定判断；先写 `dispatch` 工具，只会让“一把开五个 Worker、互相改同一个文件”的成本更低。
2. **Plugin 天然按 Harness 分裂。** DSH 插件无法控制 Claude Code 的 `--bg` 会话；Cindy 宿主是否暴露插件 API 也待验证。先做 Plugin，等于先选一个 Harness 站队，而用户的真实工作流是 Claude Code / DSH / Cindy 多栈。
3. **Skill 今天就能装、零权限面、可回滚。** 复制一个目录即可；不需要 review TypeScript，不需要重启宿主，不需要处理插件升级兼容。
4. **Plugin 需要一个稳定的契约才能写。** “写回哪个文件、什么字段、谁合并”没定下来之前，Plugin 只能把不确定性焊死在代码里。Skill 阶段正好把这个契约跑出来。
5. **验证成本不对称。** Skill 一次失败只损失一次对话；Plugin 一次失败可能留下孤儿会话、半写文件、错误索引。

### 3.2 分工边界（避免重叠）

| 关注点 | Skill 负责 | Plugin 负责 | 禁止越界 |
|---|---|---|---|
| 触发判断 | 准入/否决条件、任务类型、用户意图识别 | 工具描述里只写“做什么、返回什么”，不写“何时用” | Plugin 不内嵌启发式规则 |
| 任务拆解 | 切分维度、Worker 数量上限、写作用域划分、依赖 DAG | 不做拆解，只接收已结构化的 workstream | Plugin 不替用户决定切几刀 |
| Worker brief | 模板、必填字段、验收标准、输出契约 | 可校验必填字段，但不生成策略 | Plugin 不写“怎么给上下文”的长文 |
| 派发 | 说明在哪个 Harness 用哪个原语、要记录什么 | 提供统一工具 `parallel_dispatch`，归一化句柄 | Skill 不实现重试/轮询 |
| 状态 | 在哪些里程碑查看、如何向用户汇报 | 持久 registry、`parallel_status`、事件驱动唤醒 | Skill 不要求“每 30 秒查一次” |
| 写回 | schema、时机、`index.md` 合并规则、失败降级 | 校验 schema、自动索引、空闲时唤醒 Lead | 两边各写一套字段定义 |
| 冲突 | 一个文件只有一个 writer、Lead owns merge | 可检测写作用域重叠并拒绝派发 | Plugin 不自动合并代码 |
| 窗口/UI | 解释宿主已有页签/窗口能力 | 若宿主 API 允许，提供状态面板/开窗入口 | 第一阶段不做 |
| 分发 | 目录复制 / `npx skills add` | profile bundle 安装 | 不互相依赖 |

**唯一契约**：`skills/parallel-orchestration/references/memory-shard.md` 里的 `schema: 1`。Skill 按它写，Plugin 按它校验；改字段必须升版本，两边同时改。

**一条判定规则**：能用代码机械执行的，归 Plugin；需要判断、取舍、沟通的，归 Skill。两边都想要的能力，先放 Skill，等它稳定成“无争议的机械规则”再下沉到 Plugin。

### 3.3 落地顺序与门槛

| 阶段 | 产物 | 周期 | 退出条件（硬门槛） |
|---|---|---|---|
| 1. Skill v0.1 | `SKILL.md` + 3 个 reference + README | 2–3 天 | 4 类任务；Claude Code / DSH 各跑一遍；准入判断 4/4；写回 100%；文件冲突 0 |
| 1.5 契约加固 | `schema: 1` 冻结、run ledger、失败 playbook | 1 天 | 两个不同 Harness 写出的 shard 可互相读懂 |
| 2. Plugin v0.1（仅 DSH） | `parallel_dispatch/status/collect` + 事件唤醒 + schema 校验 | 4–6 天 | 关掉页面后仍能查到状态；Worker 完成后 30 秒内索引更新 |
| 3. 宿主集成 | 状态面板、开窗入口、跨 Harness 适配 | 待验证 | 取决于 Cindy 是否暴露插件/窗口 API |

**Stage 1 不通过，不进入 Stage 2。** 如果 Skill 跑下来发现“模型根本不愿意并行”或“并行没有更快”，那说明缺的不是工具，而是更上层的任务选择策略；这时候写 Plugin 是浪费。

## 4. 需求映射：用户的三个诉求分别由谁满足

| 诉求 | Stage 1（Skill） | Stage 2（Plugin） |
|---|---|---|
| 长任务时做别的事 | 可拆时 split 成 2–4 个 Worker；不可拆时 offload 成 1 个后台 Worker，主会话保持空闲 | `parallel_dispatch` 返回稳定句柄；状态可查 |
| 共享历史记忆 | Worker 结束前写 shard；Lead 合并 `index.md`；跨 Harness 可读 | 自动校验/索引；空闲唤醒，避免“没人写回” |
| 另开窗口 | 解释窗口只是视图；后台任务已经让主会话空出来 | 若宿主允许，提供“在窗口中打开此 run”的入口（**需要验证**） |

## 5. 风险与未决项

| 项 | 影响 | 处理 |
|---|---|---|
| Claude Code `--bg` 的 worktree/权限/合并行为 | Worker 的改动可能不在主工作树里 | Skill 要求绝对路径 + 显式收集；首次运行验证 worktree 行为；**需要验证** |
| `/subtask` 是否存在于你的 Cindy 环境 | 适配段可能写错 | 已确认为非 stock 命令；存在则当 Tier B/A，不存在则用 `claude --bg` |
| Cindy 是否暴露插件 API、窗口 API | Plugin 可能只能做 DSH 侧 | Stage 2 先只承诺 DSH；Cindy 侧**需要验证** |
| 共享记忆分片的真实路径/格式 | 写回可能写错地方 | Skill 用“解析顺序 + 可配置根目录”，不硬编码；首次运行先探测 |
| SkillHub / `agr add` 是否存在 | 分发说明可能失效 | 本机未找到证据；以目录复制和 `npx skills add` 为保底，**需要验证** |
| 子代理无写权限 | shard 落不了盘 | Worker 返回 shard 正文，Lead 代写（Skill 已含降级路径） |
| Skill 在压力下被跳过 | 又回到“一把开五个” | 按 `writing-skills` 做 RED/GREEN 测试，补 rationalization table |
| Plugin API 漂移 | 升级后崩溃 | 只使用 `cordis_inspect_query` 能查到的 Service/Event；锁定 DSH 版本 |

## 6. 证据表（本次在本机 DSH 安装中核对）

| 结论 | 证据 | 等级 |
|---|---|---|
| 用户级 Skill 目录是 `~/.agents/skills/` | 本机存在 `~/.agents/skills/find-skills/SKILL.md`、`~/.agents/skills/writing-skills/SKILL.md` | 已验证 |
| Skill 生态安装命令是 `npx skills add <package>` | 上述 `find-skills/SKILL.md` 第 21–32、96–104 行 | 已验证（文档） |
| DSH 插件可用对象式 `apply(ctx)` | `app.asar` 内 Cordis 插件类型定义：`Plugin.Object.apply(ctx, config)` | 已验证 |
| 插件注册工具用 `ctx.tools.register(defineTool({...}))` | `app.asar` 内 action-cards 插件实现 | 已验证 |
| 插件安装可用 `plugin_manager` 的 `install_bundle` | 内置 `cordis-plugin-development/SKILL.md` 第 5 行 | 已验证（文档） |
| CLI 存在 `dsh plugin --profile <name> add <package>` | `app.asar` 内的 profile / 插件文档与错误提示 | 已验证（文档） |
| 本机 PATH 上没有 `dsh` | `Get-Command dsh` 返回 CommandNotFound | 已验证（本机） |
| 会话日志是唯一事实源；插件状态应是派生缓存 | 内置 `references/practices.md` 原则 1 | 已验证（文档） |
| Claude Code 2.1.284 原生支持 `claude --bg` | `claude --help` 的 `--bg, --background` 条目 | 已验证 |
| 后台会话可脚本化查询 | `claude agents --json` 实际返回 `id`/`cwd`/`kind`/`startedAt`/`sessionId`/`name`/`state` | 已验证 |
| `--bg` 不能与 `--print` 同用，prompt 用位置参数 | CLI 报错原文：`--bg and --print conflict ... claude --bg '<task>'` | 已验证 |
| 后台会话要求 workspace trust | CLI 报错原文：`Workspace not trusted ...` | 已验证 |
| stock Claude Code 2.1.284 没有 `/subtask` | 安装包 `@anthropic-ai/claude-code` 内全文搜索无 `subtask` 命中 | 已验证（本机） |
| `~/.claude/skills` 与 `~/.agents/skills` 可共用一份 | 本机 `.claude/skills` 下多项为 Junction，Target 指向 `.agents/skills` | 已验证（本机） |
| DSH `subagent` 立即返回并在完成时通知 | 当前会话可用工具说明 | 已验证（运行时） |
| Cindy SkillHub / `agr add` | 本机未找到 `SkillHub` 字符串，也没有 `agr` 命令 | **需要验证** |
| Cindy 插件/窗口 API | 本机无 Cindy 源码/文档 | **需要验证** |

## 7. 结论

- **优先做 Skill**，因为它解决的是当前真正的瓶颈：判断与协议。
- **Plugin 不是不做，而是排第二**，并且第一版只做 DSH 侧、只做被 friction log 证明的三件事：派发、状态、写回。
- **组合的耦合点只有一个**：版本化的记忆分片契约。守住这个边界，两边可以独立演进。

## 8. 修订记录

- **2026-11-05 v1.1：从“Cindy 专用”改为“宿主无关”。** 起因是需求方明确：Skill 必须同时能用于本地 Claude Code 和 DSH，而不是只服务 Cindy。核对后修正三处：
  1. stock Claude Code 2.1.284 **没有** `/subtask`；原生能力是 `claude --bg` + `claude agents --json`。Skill 因此改成“能力层级 A/B/C/D”，不再把某个命令写进主循环。
  2. 本机 `~/.claude/skills` 已用 junction 指向 `~/.agents/skills`，证明“一份源 + 链接”可以同时服务 Claude Code 和 DSH；安装说明按此改写。
  3. 决策不变（Skill 先行），但 Plugin 的目标从“Cindy 插件”收窄为“DSH 侧优先，Cindy 待验证”。
