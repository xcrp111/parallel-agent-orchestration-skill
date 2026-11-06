# Parallel Agent Orchestration

**A host-agnostic Agent Skill for parallel background workers and shared Markdown memory.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent-Skill-blue)](https://agentskills.io)

将长任务拆分为多个可独立验收的后台工作流，由 Lead 汇总结果，并把关键结论写入跨窗口、跨宿主可读的 Markdown 记忆。

## 设计目标

- **宿主无关**：同一份 `SKILL.md` 可用于 Claude Code、DeepSeek Harness、Cindy，以及兼容 Agent Skills 的其他宿主。
- **写作用域隔离**：每个 Worker 只写自己的输出文件，避免并行编辑冲突。
- **结论可追溯**：每个工作流产出一份 shard，不依赖聊天记录。
- **可降级**：宿主没有后台能力时回到串行执行，不伪造并行。

## 工作模式

| 模式 | 适用场景 | 行为 |
|---|---|---|
| **Split** | 多个独立模块、素材或问题 | 2–4 个 Worker 并行，Lead 负责集成与验收 |
| **Offload** | 单个长任务，但需要释放当前会话 | 1 个后台 Worker，完成后写回结论 |
| **Sequential** | 单文件修改、强依赖链、需要持续人工判断 | 明确拒绝并行，保持单线程 |

## 安装

### Skills CLI

```bash
npx skills add xcrp111/parallel-agent-orchestration-skill
```

### 手动安装

```bash
git clone https://github.com/xcrp111/parallel-agent-orchestration-skill.git
mkdir -p ~/.agents/skills
cp -r parallel-agent-orchestration-skill/skills/parallel-orchestration ~/.agents/skills/
```

Claude Code 与 DSH 共用同一份 Skill：

```bash
ln -s ~/.agents/skills/parallel-orchestration ~/.claude/skills/parallel-orchestration
```

Windows PowerShell：

```powershell
New-Item -ItemType Junction `
  -Path "$env:USERPROFILE\.claude\skills\parallel-orchestration" `
  -Target "$env:USERPROFILE\.agents\skills\parallel-orchestration"
```

安装后无需额外配置。首次在某个项目中使用 Claude Code 后台会话时，需要在该目录交互式运行一次 `claude` 并接受 workspace trust。

## 使用

直接用自然语言描述任务即可。

### 拆分并行

```text
这个重构比较久，拆成 auth / billing / notifications 三块并行做，
写作用域不要重叠，最后你负责集成和测试，关键结论写到共享记忆。
```

### 后台执行

```text
我先去处理别的事，这个任务放到后台继续跑；
结束后写共享记忆，回来给我一条总结。
```

### 不应并行

```text
就改这一行，别开子任务，直接改完告诉我。
```

Skill 会先给出计划：Worker 数量、各自负责的范围、输出位置和预计时间；确认后再派发。

## 工作流程

1. **评估**：判断任务适合 Split、Offload 还是 Sequential。
2. **计划**：拆分工作流，定义写作用域、验收标准和时间盒。
3. **派发**：调用当前宿主的后台能力，记录句柄和状态。
4. **集成**：Lead 独立复核产物，处理冲突和失败。
5. **写回**：每个 Worker 写一份 shard，Lead 更新索引和决策记录。

### 共享记忆

```text
.agents/memory/parallel/runs/<run-id>/
├── plan.md          # 派发前的计划
├── index.md         # Worker 状态、产物链接、集成进度
├── decisions.md     # 影响后续工作的决策
└── <worker>.md      # 每个 Worker 的结论（schema: 1）
```

shard 包含 `status`、`files`、`Result`、`Verification` 和未决问题。它是跨窗口、跨宿主可读的事实记录，而不是聊天记录。

### 验证一次运行

运行结束后检查：

- `.agents/memory/parallel/runs/<run-id>/` 目录存在；
- 目录中有 `plan.md`、`index.md` 和每个 Worker 的 shard；
- shard 的 `status` 与 `index.md` 一致；
- 产物文件与 shard 中声明的路径一致；
- Lead 做了独立验收，而不是只采信 Worker 的“完成”。

## 兼容性

| 宿主 | 后台派发 | 状态查询 |
|---|---|---|
| Claude Code 2.1.x | `claude --bg "<task>"` | `claude agents --json` |
| DeepSeek Harness | `subagent` | 完成通知 |
| Cindy | Worker / 页签；Claude Code 内核可用 `--bg` | 宿主 UI |
| 其他 CLI Agent | 后台进程 + pid/log | 进程状态 + 日志 |

无后台能力的宿主会自动回退到串行执行。

## 目录结构

```text
.
├── README.md
├── LICENSE
├── .gitignore
├── .gitattributes
└── skills/
    └── parallel-orchestration/
        ├── SKILL.md
        └── references/
            ├── worker-brief.md
            ├── memory-shard.md
            └── harness-adapters.md
```

- `SKILL.md`：编排规则、决策闸门、派发与写回流程。
- `references/worker-brief.md`：Worker 派发模板。
- `references/memory-shard.md`：共享记忆 schema 与索引模板。
- `references/harness-adapters.md`：各宿主的能力映射与已知限制。

## 状态

- 版本：`v0.1`
- DSH：已通过双 Worker 流程验证。
- Claude Code 2.1.x：安装与命令层验证完成，后台任务待端到端验证。
- Cindy：派发接口以宿主实际能力为准。

## 贡献

Issue 和 Pull Request 均可。修改 `SKILL.md` 或 `references/` 时，请保持 `schema: 1` 的向后兼容；如需变更字段，请同时更新 `memory-shard.md` 和 `harness-adapters.md`。

## License

[MIT](LICENSE)
