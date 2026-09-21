---
name: setup-agents
description: 管理 .workflow/agents.json。登记本机和服务器上有哪些 agent，每个 agent 只记录 harness、模型强弱、响应速度快慢、所在机器和擅长领域。属性不确定时向用户询问。
disable-model-invocation: true
---

# Setup Agents

`.workflow/agents.json` 是本工作流唯一的注册表。它同时列出本机和服务器上的 agent，`/route-agent` 和 `/dispatch` 都只从这里读数据。

## `.workflow/` 属于每个项目，不属于 skills 仓库

本仓库只是 skill 集合，自身没有、也不应该有 `.workflow/`。

- `.workflow/agents.json` 在**使用这套 skill 的那个项目**里创建，由那个项目自己提交到 git
- 本 skill 自带一份基线名单 `agents.baseline.json`，首次使用时复制过去作起点
- 在 skills 仓库里跑 `/setup-workflow` 或本 skill 都是错位的，不要这么做

## Agent 是什么

一个 agent 只由三样东西定义：

- **Harness**：用什么外壳跑它，例如 `claude-code`、`codex`、`copilot-cli`、`cursor`
- **模型强弱** `strength`：`high`、`medium`、`low`。它告诉路由这个 agent 适合接多难的活
- **响应速度快慢** `speed`：`fast`、`medium`、`slow`

再补上 `host` 说明它装在哪台机器上，`tags` 说明它擅长什么。

### `strength` 决定优先交给谁

`/route-agent` 会给每个 ticket 定一个档位，然后按 `strength` 决定优先交给谁：

| 档位 | 优先 |
|------|----------|
| 关键（认证、迁移、并发、疑难 bug） | `high` |
| 复杂（跨模块重构、新架构、性能定位） | `high` |
| 常规（单模块实现、加接口、补测试） | `medium` 及以上 |
| 机械（重命名、格式化、文档） | 不限 |

这是**优先顺序，不是通行证**。本侧没有 `high` 时，路由会用本侧最强的那个顶上，把降级写进报告。不会因为缺 high 就把 ticket 挂起来。

所以 `strength` 填错的代价是「强的活没人优先做」或「小事占用最强的慢模型」，而不是流程卡死。即便如此也不要凭 harness 名字猜。

不记录任务系统、队列、面板、adapter 或其它运行机制。执行方式在整条工作流里只有一种：git worktree。所以 agent 记录里没有这些维度的位置。

## 本机和服务器是平等的

本机和服务器具备**完全相同的能力**，都能规划、调研、执行、review 和验证。差别只有一个：各自装了哪些 agent。

所以本 skill 不需要问「哪台机器负责哪个阶段」，只问「这台机器上有哪些 agent」。

## 何时调用

- 第一次在被管理的项目上启用工作流
- 换了机器、装了新 harness 或换了模型
- 想改某个 agent 的强弱、速度或擅长领域
- 想确认某台机器上现在有哪些 agent

平时不用跑，只在 agent 清单变化时跑。

## 属性从哪里来

`harness`、`host`、`model` 是可以直接确认的事实，看一下装了什么就知道。

`strength`、`speed`、`tags` 是主观判断，**不确定就问用户，不要照着本文件的模板抄，也不要凭 harness 的名字猜**。提问时给选项，不要抛开放式问题。

### 问卷模板

```text
这台机器上装了哪些 harness？
1. claude-code
2. codex
3. copilot-cli
4. cursor
5. 其它（直接写名字，例如 oh-my-pi、antigravity）

<harness> 你主要用哪个模型？
它的定位是：
- 强弱：high（复杂设计、疑难 bug） / medium（常规实现） / low（重命名、格式化这类机械活）
- 速度：fast（几十秒内出结果） / medium / slow（数分钟以上）

它擅长哪几类工作？（可多选）
- frontend / backend / architecture / security / research / test / docs / refactor
```

`harness` 是自由字符串，上面那份列表只是提示，项目有自己的壳子就照填。

问卷只在基线与实际不符时逐条走完。基线已经对上的部分不用重新问。

### 一台一台登记

不需要一次把两台机器都填完。在**哪台机器上跑，就登记哪台**：

- 在本机上跑：只问本机装了哪些 harness，新记录写 `host: local`
- 在服务器上跑：只问服务器，新记录写 `host: server`

已有记录直接复用，只追加这台机器缺的那几条。两台机器问出完全相同的答案也是正常的，照实写。

## 设计原则

- **只描述 agent 本身**：harness、模型强弱、速度、所在机器、擅长领域。不描述任务机制。
- **一台机器一条记录**：同一个 harness 加同一个模型装在两台机器上，写两条，`host` 不同，`id` 也不同。
- **稳定 ID**：kebab-case，全表唯一。ticket 用 `id` 引用，不用 `display_name`。
- **`strength` 是优先顺序**：`high` 优先接关键和复杂任务，`medium` 接常规，`low` 接机械。填错不会卡流程，但会让活配得不合适。
- **每台机器尽量有一条 `high`**：不是硬要求，但没有的话，关键 ticket 只能降级给本侧最强的 medium，效果打折扣。
- **空表合法**：没有登记任何 agent 时，ticket 推荐 `manual`，由当前会话完成。

## 基线名单

同目录的 `agents.baseline.json` 是固定下来的基础名单，覆盖两台机器上实际在用的四个 agent：

| id | harness | model | host | strength | speed | 定位 |
|----|---------|-------|------|----------|-------|------|
| `codex-gpt` | codex | gpt | local | high | slow | 本机唯一 high，关键/复杂档优先给它 |
| `claude-domestic` | claude | deepseek 等国模 | local | medium | fast | 常规与机械档主力 |
| `oh-my-pi-gpt` | oh-my-pi | gpt | server | medium | medium | 服务器常规档，也是服务器上的降级况 |
| `antigravity-gemini` | antigravity | gemini | server | medium | fast | 服务器前端专项 |

首次在某个项目上运行时：

1. 把 `agents.baseline.json` 去掉 `_comment` 字段后写入项目的 `.workflow/agents.json`
2. 问用户这份基线跟这台机器的实际情况是否一致
3. 不一致就按问卷逐条改，一致就直接用

之后可以随时增改。基线只是一份起点，不是强制值。

### 基线可以补强的地方

**服务器上没有 `strength: high` 的 agent。** 认证、数据迁移、并发、跨模块重构、性能定位这类 ticket 在服务器上会降级给 medium（默认是 `oh-my-pi-gpt`），路由会在理由里注明降级。

流程不会卡住，但如果服务器上真的接了重要活，建议补一条 high。两种情况：

- 服务器上有更强的模型但没登记 → 补一条记录
- 实际就 `oh-my-pi-gpt` 最强 → 把它改成 `strength: high`

不补也能跑。先看实际用下来什么样再说。

## 字段说明

| 字段 | 必填 | 含义 |
|------|------|------|
| `schema_version` | 是 | 当前 schema 版本，写 `1` |
| `agents` | 是 | agent 数组，顺序无意义 |
| `agents[].id` | 是 | 稳定的 kebab-case ID，全表唯一 |
| `agents[].display_name` | 否 | 给人看的名字，可随时改 |
| `agents[].harness` | 是 | 运行外壳，例如 `claude-code`、`codex`、`copilot-cli` |
| `agents[].model` | 否 | 具体模型标识，自由字符串 |
| `agents[].host` | 是 | `local` 或 `server` |
| `agents[].strength` | 是 | `high`、`medium` 或 `low` |
| `agents[].speed` | 是 | `fast`、`medium` 或 `slow` |
| `agents[].tags` | 是 | 擅长领域的标签数组 |
| `agents[].description` | 否 | 一句话说明 |

约束：

- `id` 全表唯一。改 id 要迁移 ticket 里的引用。
- 每台机器尽量有一条 `strength: high`。没有也能运行，关键 ticket 会降级给本侧最强者。
- `tags` 可以重叠。重叠表示多个 agent 都能做，由 `/route-agent` 再按适合性判断选哪个。
- ticket 里的推荐必须引用这里存在的 `id`，或写 `manual`。
- 空数组是合法的。这时这类 ticket 只能推荐 `manual`，或从那台机器手动跑。

## 流程

### 首次登记

1. 在**被管理的项目**里运行（不是在 skills 仓库里）。如果该项目还没有 `.workflow/`，先跑 `/setup-workflow`。
2. 把 `agents.baseline.json` 写成该项目的 `.workflow/agents.json`：去掉 `_comment`，保留 `schema_version` 和 `agents`。
3. 问用户这份基线跟这台机器的实际情况是否一致，不一致就按问卷逐条改。
4. 在这台机器上跑，新记录就写对应那一侧的 `host`。
5. 运行 `/route-agent` 预检：这台机器上有没有能满足常见 phase 的 agent。

另一台机器下次在那台上跑一遍同样的流程即可。不需要一次填齐。

### 新增 agent

1. 追加唯一的 `id`。
2. 按问卷补全字段。
3. 重新预检受影响的 ticket。

### 修改 agent

- 改 `display_name`、`description`：安全，不影响任何 ticket。
- 改 `tags`：影响后续推荐，已写死 agent 的 ticket 不变。
- 改 `strength`：改变优先顺序。调低后关键 ticket 会转给别的 agent，或在本侧降级，先跑 `/route-agent` 预检。
- 改 `speed`：只影响机械档位和同档位内部的选择。
- 改 `host`：等于把 agent 搬到另一台机器，两台机器的推荐都会变。
- 改 `id`：破坏性操作，要同步改所有 ticket 里的 `local:` 和 `server:` 值。

### 删除 agent

1. 扫描 `.workflow/` 里的 `local:`、`server:` 和 agent id 引用。
2. 先把未完成 ticket 改推荐到别的 agent 或改成 `manual`。
3. 从 `agents` 数组移除。

## 反模式

- 🚫 凭 harness 名字猜强弱和速度，不向用户确认
- 🚫 在 agent 记录里写任务系统、队列、面板或 adapter 信息
- 🚫 只登记本机，忘记服务器
- 🚫 把 `strength` 说成硬关卡，它实际是优先顺序，缺 high 只降级不阻塞
- 🚫 把 agents.json 放进 .gitignore，它必须提交到 git

## 输出格式

```markdown
## 🤖 agents.json 已更新

### 🖥️ 本机（local）
- `claude-code-opus`：claude-code / opus，high / slow，擅长 架构、安全
- `copilot-fast`：copilot-cli / flash，low / fast，擅长 文档、重命名

### ☁️ 服务器（server）
- `codex-high`：codex / gpt-5-high，high / medium，擅长 后端、重构、测试

### 🔀 影响
- `<ticket-id>` 的推荐值已失效，需要重新运行 `/route-agent`

### ⚠️ 提醒
- <哪台机器缺少合适强度的 agent，或哪类工作没有对应的 tags>
```

## 与其它 skill 的衔接

- `/setup-workflow` 只管 `.workflow/` 的目录和路径配置，不碰 agents.json
- `/route-agent` 读取 agents.json，为 ticket 算出本机和服务器各自的最佳 agent
- `/to-tickets` 把推荐写进 ticket 的 route 块
- `/dispatch` 按当前所在机器，从两条推荐里选一条执行
