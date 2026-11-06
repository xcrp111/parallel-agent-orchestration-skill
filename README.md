# parallel-agent-orchestration

让长任务在后台继续跑，主会话去做别的事；Worker 的结论写回一份跨窗口、跨宿主、跨 Harness 可读的共享记忆。

> **状态：v0.1。** DSH 侧已用 2 个 Worker 跑通一次演示（plan → 并行 → shard → index）；Claude Code 侧已安装，待一次真实后台任务验证。

**一个 Skill，三种宿主：**

| 宿主 | 派发方式 | 装到哪 |
|---|---|---|
| 本地 Claude Code | 原生 `claude --bg "<brief>"`，`claude agents --json` 查状态 | `~/.claude/skills/` |
| DSH | `subagent` 工具，完成时通知 | `~/.agents/skills/` |
| Cindy | 宿主 Worker / 页签；CC 内核可用 `claude --bg`，DSH 内核可用 `subagent` | 随内核；若读 `~/.agents/skills/` 则共用一份 |

核心循环不写死任何宿主命令，只要求三件事：**拿得到句柄、查得到状态、收得到结果**。具体命令由 adapter 提供。

## 为什么先做 Skill，而不是 Plugin

并行派发本身已经不是缺口：Claude Code 2.1.x 有 `--bg` 和 in-turn 子代理，DSH 有 `subagent` / `workflow`，Cindy 有 Lead/Worker。真正缺的是：

1. **判断**：什么任务该并行、切几刀、谁合并；
2. **协议**：Worker 怎么拿到上下文、完成后写什么、写到哪、冲突怎么办。

这两件事都是 Markdown 能解决的，不需要 TypeScript。等流程跑稳，再把“持久状态、完成唤醒、状态查询”下沉成 Plugin。

完整论证见 [docs/0001-skill-vs-plugin.md](docs/0001-skill-vs-plugin.md)。

## 安装

Skill 是一个目录，核心只有 `SKILL.md`。**保留一份源，其他宿主用链接指过来**，避免三份副本漂移。

### 1. 放一份到用户级目录

```bash
mkdir -p ~/.agents/skills
cp -r skills/parallel-orchestration ~/.agents/skills/
```

### 2. 让 Claude Code 看到同一份

macOS / Linux：

```bash
mkdir -p ~/.claude/skills
ln -s ~/.agents/skills/parallel-orchestration ~/.claude/skills/parallel-orchestration
```

Windows（本机 `.claude/skills` 已经在用 junction 指向 `.agents/skills`，同一模式）：

```powershell
New-Item -ItemType Junction `
  -Path "$env:USERPROFILE\.claude\skills\parallel-orchestration" `
  -Target "$env:USERPROFILE\.agents\skills\parallel-orchestration"
```

不想用链接就直接复制一份；只是以后更新要记得两边都换。

### 3. 项目级（可选）

把 `skills/parallel-orchestration` 放进项目的 `.agents/skills/` 和 `.claude/skills/`，随仓库走，团队成员 clone 即用。

### 4. 其他分发方式

```bash
npx skills add <owner>/<repo>   # 发布到 GitHub 后
```

**Cindy SkillHub / `agr add`：** 本机未找到可用证据，请以 Cindy 文档为准；上面的目录/链接方式不依赖它们。若 Cindy 读 `~/.agents/skills/`，装一次即可。

## 使用

更细的非程序员说明（怎么判断有没有效果）见 [docs/usage.zh.md](docs/usage.zh.md)。

安装后直接描述任务，Skill 会按触发条件加载。也可以显式要求：

```text
这个重构比较久，用并行编排：auth / billing / notifications 三个模块分开做，
最后你来做集成和测试。关键结论写到共享记忆。
```

期望行为：

1. 先过闸门：可拆就 split，不可拆但能交接就 offload 成 1 个后台 Worker，都不行就明确拒绝；
2. 按宿主能力选择派发层级，记录句柄；
3. Lead 做集成，不空等、不轮询；
4. 每个 Worker 写一份 shard，Lead 合并 `index.md` / `decisions.md`；
5. 一条消息汇报：完成了什么、还在跑什么、需要什么决策。

## 共享记忆

默认写到当前工作区的 `.agents/memory/parallel/`，所以同一个仓库里的 Claude Code、DSH、Cindy 都能读到。跨工作区时，在两个宿主里设同一个绝对路径：

```bash
export PARALLEL_MEMORY_ROOT="/abs/path/to/shared-memory"
```

```powershell
$env:PARALLEL_MEMORY_ROOT = "D:\shared-memory"
```

分片格式见 [memory-shard.md](skills/parallel-orchestration/references/memory-shard.md)。这是 Skill 与未来 Plugin 之间唯一的契约（`schema: 1`）。

## 仓库结构

```
.
├─ README.md
├─ LICENSE
├─ docs/
│  ├─ 0001-skill-vs-plugin.md    # 架构决策（ADR）
│  ├─ phase-1-plan.md            # 第一阶段实施计划
│  └─ usage.zh.md                # 非程序员使用/验收说明
└─ skills/
   └─ parallel-orchestration/
      ├─ SKILL.md                # 编排逻辑（宿主无关）
      └─ references/
         ├─ worker-brief.md      # 派发模板
         ├─ memory-shard.md      # schema: 1
         └─ harness-adapters.md  # Claude Code / DSH / Cindy / 通用 CLI
```

`SKILL.md` 与 references 用英文写，便于公开分发和被模型稳定遵循；决策文档与计划用中文。

## 已验证 / 需要验证

| 结论 | 状态 |
|---|---|
| 用户级 Skill 目录 `~/.agents/skills/` | 已验证（本机） |
| `~/.claude/skills` 用 junction 指向 `~/.agents/skills` | 已验证（本机现有布局） |
| Claude Code 2.1.284 有 `claude --bg`、`claude agents --json` | 已验证（CLI help + 实跑 `agents --json`） |
| `--bg` 与 `--print` 冲突，prompt 用位置参数 | 已验证（CLI 报错原文） |
| 后台会话需要 workspace trust | 已验证（CLI 报错原文） |
| 原生 `/subtask` | **不存在于 stock 2.1.284**；可能是 Cindy/插件命令 |
| DSH 插件对象式 `apply(ctx)`、`ctx.tools.register(defineTool(...))` | 已验证（安装包内类型与实现） |
| DSH `plugin_manager` / `dsh plugin add` | 已验证（安装包文档） |
| Cindy 插件 API / 窗口 API | **需要验证** |
| Cindy SkillHub / `agr add` | **需要验证**（本机无证据） |
| 后台会话是否用 git worktree、如何合并 | **需要验证**（`claude rm` 提到 worktree） |

Skill 里所有不确定的地方都写成“探测 + 回退”，不会假装命令存在。

## 验证记录

- **2026-11-05，DSH**：2 个 Worker 并行写独立文档与 shard；Lead 写 `index.md`/`decisions.md`；无同文件冲突。任务为流程演示，时长低于正式门槛。
- **2026-11-06，Claude Code 2.1.284**：安装与 junction 已验证；`claude --bg` / `claude agents --json` 在参数层面已验证，完整后台任务待接受 workspace trust 后实测。

## 路线图

| 阶段 | 内容 | 状态 |
|---|---|---|
| 1 | Skill v0.1 + 场景验证（Claude Code / DSH 各一遍） | DSH 演示通过；Claude Code 待验证 |
| 1.5 | 冻结 `schema: 1`，跑通双宿主互读 | 待做 |
| 2 | DSH Plugin：`parallel_dispatch` / `status` / `collect` + 完成唤醒 | 待验证后启动 |
| 3 | 宿主状态面板 / 开窗入口 / 跨宿主适配 | 取决于宿主 API |

Stage 2 的启动条件：friction log 证明前三号问题是“没有持久状态 / 没人写回 / 无法取消”，而不是“不知道该不该并行”。

## License

MIT
