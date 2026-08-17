# dsh-trajectory-control

轨迹控制（Trajectory Control）——把 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的轨迹视图从被动回放改造成主动控制台的设计与实现。

## 这是什么

DSH 的 Trajectory 视图能逐事件回看一个 agent 干了什么，但只能看。你看着 agent 走错方向时，没法在正看着的那一步截住它，也没法回到更早的决策点换个说法重新往下走。

这个项目要加的能力：

- **暂停** —— 运行中实时暂停；跑完后也能在完整轨迹上任选一点截住
- **分支回滚** —— 从某一点岔出一条新会话，原轨迹保留（git branch 的心智）
- **原地回滚** —— 同一条会话退回某点，遮蔽后续，插入新对话续写（重写历史的心智）
- **插话续跑** —— 在回滚点插入指令，让 harness 带着它继续跑

目的是通过控制模型看到的每一个中间步骤，来控制它的最终输出。

## 设计文档

[docs/DESIGN.zh.md](docs/DESIGN.zh.md) 是主文档（[English](docs/DESIGN.md)）。它记录了对 DSH 运行时的核实结论——包括三处需要纠正的判断，两个方向都有：一个并不存在的核心约束曾被当成最高风险，而两个真实的阻塞项被完全漏掉。

文档同时是一份 DSH Agent Note，可直接放回 `.agents/notes/proposed/architecture/`。

## 上游补丁

轨迹控制依赖两项 DSH 本体改动。它们各自独立、与本功能解耦，且都已在上游通过完整门禁（typecheck / lint / 13410 项测试 / 28 项文档门禁）。

| 补丁 | 内容 |
|---|---|
| `0001` | 设计文档本身，作为 Agent Note 放入 `.agents/notes/proposed/architecture/` |
| `0002` | 修复：无人认领的 surface replacement 在对话视图中渲染为空白 |
| `0003` | 新增：`Session.append` 可声明 `ignorable`，补上该字段缺失的写入方 |

`0002` 是当前就存在的真实缺陷，与本功能无关也值得修：`ui-conversation` 靠硬编码插件名识别 replacement，只认压缩功能，而通用兜底只认 append 来源。因此除压缩之外任何生产者的 `user/message` replacement 都无人认领、无人兜底——**页面上什么都不显示**。原地回滚产生的正是这种事件形状。

`0003` 补上一个上游已预留的扩展点：`ignorable` 标记决定"读取方遇到不认识的事件类型时可以跳过、还是必须拒绝整条日志"。该字段早已被 seed 校验、两个持久化后端与 wire schema 认可，但没有任何写入方能设置它。上游的[版本机制 Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/main/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md) 写明这个口子留给"第一个需要它的使用者"。

`0002` 与 `0003` 都改动了 `0001` 引入的 Agent Note，所以三个补丁必须按序应用。

### 应用补丁

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
git am /path/to/dsh-trajectory-control/patches/*.patch
```

补丁基于 `0.1.0-rc.5` 时期的源码，已验证可干净应用于该版本。上游改动较快，冲突时用 `git am -3` 三方合并。

## 进度

| 阶段 | 内容 | 状态 |
|---|---|---|
| T-1 | replacement 兜底渲染 | 已完成（补丁 `0002`） |
| T-1 | `ignorable` 写入面 | 已完成（补丁 `0003`） |
| T0 | 实时暂停/恢复、轨迹行操作、插话表单、暂停横幅 | 未开始 |
| T1 | 标注、血缘树、原地回滚、审批改指令 | 未开始 |
| T2 | 断点、错误自动暂停、副作用清单、Diff、分支探索 | 未开始 |
| T3 | 导出/重放、统计、子 agent 内联、goal 联动 | 未开始 |

T-1 两项是地基：没有它们，新事件会成为日志定时炸弹，且原地回滚的界面是空白的。

## 许可

MIT。`patches/` 下的补丁针对 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（MIT, Copyright (c) 2026 DeepSeek），其内容版权归原项目所有。
