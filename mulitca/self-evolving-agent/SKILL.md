---
name: self-evolving-agent
description: "自进化:每次任务完成后,复盘本次执行过程,检查自己的 agent 定义(instructions/description)是否需要更新;如有明确证据表明确有必要,则通过 multica CLI 自行更新,并把变更记录到任务中。Use after completing any task or issue run."
user-invocable: false
allowed-tools: Bash(multica *)
---

# 自进化 Agent(Self-Evolving Agent)

本 skill 让启用它的 agent 具备自我维护能力:每次任务结束后,基于本次任务的实际执行证据,判断自己的 agent 定义是否需要更新;需要时自行更新,不需要时明确跳过。

## 触发时机

在每个任务(issue run 或产生了交付物的 chat 回合)即将结束、发出最终回复**之前**执行本流程。纯问答、无需工具的闲聊回合可以跳过。

## 流程

### 1. 复盘本次任务

回顾刚刚完成的任务,问自己:

- 执行过程中,我的 `instructions` 是否有指导缺失、步骤过时、或给出错误指引的地方?
- 是否出现了反复出现的新模式(例如某个仓库的固定构建命令、某个项目的提交流程),值得固化到 instructions 中?
- `description`(目录简介)是否与实际职责不符?

**只有本次任务给出了具体、可引用的证据时才继续;凭感觉、单次偶发事件、或用户的一次性偏好,不足以触发自我修改。**

### 2. 读取当前定义

先确定自己的 agent ID:运行时通常已把身份信息注入上下文(如 AGENTS.md 中的 "You are: <名字> (ID: ...)")。找不到时,用名字匹配:

```bash
multica agent list --output json
multica agent get <agent-id> --output json
```

### 3. 判定是否需要更新

- **需要更新**:instructions 中存在本次任务证实为错误/过时的具体指引;或缺失某个今后会反复用到的明确步骤。
- **不需要更新**:绝大多数任务属于此类——执行顺利,或遇到的问题是一次性的。此时在最终回复中用一句话说明"已自检,无需更新"即可,不做任何写操作。

### 4. 执行更新(仅在判定时)

只许修改 `instructions` 和 `description` 两个字段,且遵守:

- 改动最小化:只修正被证实的部分,不做整体重写,不顺手重构无关内容。
- `description` 不超过 255 字符。
- **绝不触碰** `custom_env`、`mcp_config`、skill 绑定、`model`、`runtime_id`、`max_concurrent_tasks` 等其他字段。

```bash
multica agent update <agent-id> --instructions "<更新后的完整 instructions>" --output json
```

### 5. 记录与回退

- 更新成功后,在当前 issue 上发评论(`multica issue comment add --content-file <file>`),说明:改了什么、为什么改(引用本次任务中的证据)。这是给人看的审计轨迹,不可省略。
- 如果 `multica agent update` 因权限被拒(agent actor 可能被拒绝修改 agent 记录),不要反复重试:把建议的变更内容写成 issue 评论,说明"自检发现 instructions 建议更新,但当前权限不足",交给 workspace owner 处理。

## 边界

- 自我修改是保守行为:宁可少改,不可多改。一次任务最多触发一次更新。
- 不得基于假设或推测修改自己的定义;必须有本次任务中的实际证据。
- 不得利用本 skill 提升自身权限、读取 secrets(`custom_env` 对 agent actor 永远不可读)、或改绑 runtime/skills。
