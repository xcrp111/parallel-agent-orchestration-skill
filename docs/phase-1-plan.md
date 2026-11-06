# Phase 1 实施计划：Skill v0.1

- 目标：用一份可安装、可验证的 `SKILL.md`，把“长任务并行 + 共享记忆”从个人习惯变成可复用流程。
- 周期：3 个工作日（含 1 天基线测试与 1 天迭代）。
- 前置决策：[ADR-0001](./0001-skill-vs-plugin.md)
- 产物：[SKILL.md](../skills/parallel-orchestration/SKILL.md) + references + 可发布仓库。

## 1. 本阶段不做什么

- 不写 Plugin、不注册工具、不做 UI、不打开系统窗口。
- 不实现后台任务调度；只调用宿主已有的原语。
- 不定义新的记忆存储引擎；只定义 Markdown 分片的写回协议。
- 不追求“自动判断所有任务”；先覆盖四类高频场景。

## 2. 工作分解

| # | 任务 | 产出 | 预计 |
|---|---|---|---|
| 1 | 基线测试（RED） | 4 个场景在“无 Skill”下的行为记录，逐字记录模型的借口/错误 | 0.5 天 |
| 2 | 写 `SKILL.md` | frontmatter + 编排正文 + 快速检查表 | 0.5 天 |
| 3 | 写 references | `worker-brief.md`、`memory-shard.md`、`harness-adapters.md` | 0.5 天 |
| 4 | 场景测试（GREEN） | 4 个场景 + 跨宿主互读，记录与修订 | 0.5 天 |
| 5 | 补漏洞（REFACTOR） | 把新出现的合理化借口写进 red flags / 反模式 | 0.5 天 |
| 6 | 仓库包装 | README、LICENSE、安装说明、路线图 | 0.5 天 |

### 2.1 基线测试怎么做

按 `writing-skills` 的 RED 阶段，在**不加载 Skill**的会话里跑下列 prompt，记录模型是否：

- 该并行时没并行（把本可并行的 3 个模块串行做）；
- 不该并行时乱并行（同一个文件开两个 Worker）；
- 派发后空等/轮询；
- 完成后只在聊天里总结，没有落盘；
- 声称“已开新窗口”但实际没有。

记录要逐字保留，后面写进 `## Red flags`。

### 2.2 验收场景

| 场景 | Prompt 要点 | 期望行为 |
|---|---|---|
| A. 多模块重构 | “把 `auth/`、`billing/`、`notifications/` 三个模块分别升级到新接口，最后统一跑测试” | 识别 3 个独立写作用域；派发 2–3 个后台 Worker；Lead 做集成与测试；全部写 shard |
| B. 单文件修 bug | “这个 parser 在空输入时崩溃，修一下” | 明确拒绝并行；单线程完成；说明理由 |
| C. 调研 + 实现 | “先调研两个候选库，再选一个接进项目” | 调研可并行（只读），实现由 Lead/单 Worker 串行；决策写进 `decisions.md` |
| D. 只想开窗 | “这个任务太久了，帮我开另一个窗口做点别的” | 不假装能开窗；能拆则 split，不能拆则整单 offload 给 1 个后台 Worker；不可交接则说明并保持单线程 |
| E. 跨宿主互读 | Claude Code 跑 A 的一个 Worker，DSH 读同一份 shard 继续集成 | 同一 `PARALLEL_MEMORY_ROOT`；字段完整；无宿主专属字段 |

E 是“不是只针对 Cindy”的硬验证：同一个 Skill、同一份记忆，两个宿主都能读懂并接着做。

### 2.3 度量

| 指标 | 目标 |
|---|---|
| 准入判断正确率 | 4/4 场景 |
| shard 写回率 | 100%（每个 Worker 一份，含 partial/failed） |
| 同文件并发写冲突 | 0 |
| 重复劳动 | 0（Lead 不重做 Worker 的作用域） |
| 用户澄清次数 | ≤1 次/场景 |
| 首次可用结果时间 | 相比串行基线不劣化 |
| 跨宿主互读 | E 场景 1/1 通过 |

## 3. 交付物清单

```
parallel-agent-orchestration/
├─ README.md
├─ LICENSE
├─ .gitignore
├─ docs/
│  ├─ 0001-skill-vs-plugin.md
│  └─ phase-1-plan.md
└─ skills/
   └─ parallel-orchestration/
      ├─ SKILL.md
      └─ references/
         ├─ worker-brief.md
         ├─ memory-shard.md
         └─ harness-adapters.md
```

`README.md` 必须包含：这是什么、什么时候用、安装（目录复制 / `npx skills add`）、30 秒示例、已验证/未验证清单、路线图。

## 4. 分发与安装

### Skill（本阶段）

| 方式 | 命令 | 状态 |
|---|---|---|
| 一份源（DSH 用户级） | `cp -r skills/parallel-orchestration ~/.agents/skills/` | 可用（本机路径已验证） |
| Claude Code 链接到同一份 | Windows `New-Item -ItemType Junction`；Unix `ln -s` | 可用（本机 `.claude/skills` 已是此模式） |
| 项目级随仓库 | 放进项目的 `.agents/skills/` 与 `.claude/skills/` | 可用 |
| Skills CLI | `npx skills add <owner>/<repo>` | 文档存在；仓库布局需按 CLI 实际行为验证 |
| Cindy SkillHub | 以 Cindy 文档为准 | **需要验证** |
| `agr add` | 以对应 CLI 文档为准 | **需要验证**（本机无此命令） |

> 结论：即使 SkillHub / `agr` 不可用，仓库也能靠“复制目录”和 `npx skills add` 分发。不要把发布阻塞在某个私有市场上。

### Plugin（Stage 2，暂不做）

| 方式 | 命令 | 状态 |
|---|---|---|
| 会话内安装 | `plugin_manager` → `install_bundle`，target 为工作区包目录 | 已验证（内置技能文档） |
| CLI | `dsh plugin --profile <name> add <package>` | 命令存在于安装包文档；本机 PATH 无 `dsh` |
| Cindy 插件市场 | 以 Cindy 文档为准 | **需要验证** |

## 5. Stage 2 预告（只做设计，不写代码）

只有在 Stage 1 的 friction log 证明“知识解决不了”时，才启动。第一版只做 DSH 侧三件事：

1. **`parallel_dispatch`**：入参 `{ goal, brief, write_scope[], done_criteria, timebox_min }`，返回 `{ run_id, worker_id, status }`。内部调用现有 `subagent` 能力，不新造调度器。
2. **`parallel_status`**：入参 `{ run_id? }`，返回各 Worker 的状态、耗时、shard 路径；状态从会话日志/存储派生，不另建事实源。
3. **`parallel_collect`**：入参 `{ run_id }`，返回结构化结果 + shard 校验结果；缺 shard 时明确报错，不静默成功。

生命周期（已核对的 API）：

- 对象式插件 `apply(ctx)`；`ctx.tools.register(defineTool({...}))` 注册工具。
- `agent/created` 事件里拿 `agent.ctx` 注册 per-agent 行为，并在插件 effect 里保存 disposer。
- `turn/end`、`tools/result` 等持久事件上更新索引；不要轮询 `agent/status`。
- `ctx.sessionProjections` 派生状态；`agent.followup()` 唤醒空闲 Lead。
- 安装用 `plugin_manager.install_bundle`；配置写进 bundle 的 `cordis.patch.yml`。

**明确不做**：不在 Plugin 里写“何时并行”的长篇策略；不试图控制 Claude Code 的 `--bg` 会话；不做跨 Harness 的伪统一，除非先验证宿主 API。

## 6. 时间线

| 日 | 内容 | 检查点 |
|---|---|---|
| D1 上午 | 基线测试 4 个场景，记录 | 有逐字失败记录 |
| D1 下午 | 写 `SKILL.md` 骨架 + frontmatter | 能被 Skill 发现并加载 |
| D2 | references + 场景 A/B 复测 | 准入判断正确，shard 完整 |
| D3 | 场景 C/D 复测、补 red flags、README/LICENSE | 4/4 通过，仓库可发布 |

## 7. 完成定义（DoD）

- [ ] 4 个场景全部达到期望行为，且有记录。
- [ ] 同一份 `SKILL.md` 在 Claude Code 与 DSH 中都能加载；E 场景互读通过。
- [ ] `SKILL.md` frontmatter 的 `name`/`description`/`whenToUse` 完整，description 只写触发条件。
- [ ] 每个 Worker 都有 shard；`index.md` 能还原一次 run 的全貌。
- [ ] README 的安装命令在干净环境里可执行（至少“一份源 + 链接”路径）。
- [ ] 明确列出“需要验证”的项，不用推测冒充实测。
- [ ] 仓库可直接 `git init` 后推到 GitHub，无本机绝对路径、无密钥。
