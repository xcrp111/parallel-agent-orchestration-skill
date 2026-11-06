# 使用说明（给不写代码的人）

## 已经装好了

- 技能本体：`~/.agents/skills/parallel-orchestration`
- Claude Code 入口：`~/.claude/skills/parallel-orchestration`（指向同一份，不是副本）
- 所以 DSH、Claude Code、以及读 `~/.agents/skills` 的 Cindy 用的是**同一份**规则；改一处，两边生效。

## 你可以直接说的三句话

**1. 拆开做（split）**

> 这个任务比较久，拆成 auth / billing / notifications 三块并行做，写作用域别重叠，最后你负责集成和跑测试。关键结论写到共享记忆。

**2. 后台跑，我去忙别的（offload）**

> 我先去处理别的事，这个任务你放到后台继续跑；结束后把结论写到共享记忆，回来给我一条总结。

**3. 不要并行（该拒绝时）**

> 就改这一行，别开子任务，直接改完告诉我。

第三种同样重要：Skill 的效果不只是“会开并行”，还包括“该不开时不开”。

## 会发生什么

1. **先说计划**：几个 Worker、各自做什么、你（Lead）做什么、大概多久。不会闷头开五个。
2. **再派发**：DSH 用 `subagent`；本地 Claude Code 用 `claude --bg`。
3. **各自写回**：每个 Worker 在 `.agents/memory/parallel/runs/<run-id>/` 下写一份 Markdown 分片；Lead 写 `index.md` 和 `decisions.md`。
4. **一条总结**：完成了什么、还在跑什么、需要你决定什么。

窗口是宿主的事：DSH/Cindy 的页签、Claude Code 的 `claude agents`。Skill 不会假装帮你开了一个系统窗口。

## 怎么判断有没有效果（2 分钟）

1. 打开工作区里的 `.agents/memory/parallel/runs/`。
2. 能看到本次 run 的目录，而不是只有聊天记录。
3. 目录里有 `plan.md`、`index.md`，以及每个 Worker 一份 `.md`。
4. 打开 Worker 的分片：有 `status`、`files`、`Result`、`Verification`，不是一句“完成了”。
5. `index.md` 的表格状态与分片对得上；最终回复能说清“谁做了什么、还在跑什么”。

五项都满足，说明流程真的跑通了；只满足聊天部分，说明 Skill 被跳过或没生效。

## 怎么判断没效果

- 没有 run 目录，结论只停在聊天里。
- 说“已经开了新窗口”，但 `claude agents` 或宿主里没有对应任务。
- 两个 Worker 改了同一个文件，或者 Lead 又把 Worker 的活重做一遍。
- 全程在“每 30 秒查一下”，没有实际推进。
- 分片只有“完成”，没有验证命令/结果。

## 本地 Claude Code 的两个注意点

- Claude Code 2.1.284 原生是 `claude --bg "<任务>"`；`/subtask` 不是原生命令，只有 Cindy 或插件提供时才有。
- 第一次在某个目录用 `claude --bg`，要先在该目录交互式运行一次 `claude`，接受 workspace trust。后台会话若显示 `blocked`，用 `claude attach <id>` 回答，不要重复派发。

## 只想验证一次

直接说：

> 用并行编排演示一下：两个互相独立的小任务，各写一个文件、各写一份共享记忆，最后你来验收。

然后按上面的“2 分钟”步骤看目录。
