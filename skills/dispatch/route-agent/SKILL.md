---
name: route-agent
description: 读取 .workflow/agents.json，为每个 ticket 算出本机和服务器各自的最佳 agent，输出两条推荐。不硬编码机器名或 agent 名称。
---

# Route Agent

ticket 需要两条推荐：**本机最佳** 和 **服务器最佳**。本 skill 负责算出来。

推荐只在两台机器各选一个 agent，不涉及任务系统、队列或 adapter。执行方式在整条工作流里已经固定为 git worktree，所以路由只需要回答一个问题：这台 ticket 交给谁。

## 数据来源

只有一份注册表：`.workflow/agents.json`。每个 agent 记录 `harness`、`model`、`host`、`strength`、`speed`、`tags`。

- `host` 决定它属于哪台机器的候选池
- `strength`、`speed`、`tags` 决定在池子里的排序
- 本机和服务器能力相同，不存在「这台机器不能做这个阶段」

没有 runtime 注册表，也没有 adapter 概念。不要去找它们。

## 输出格式

所有 ticket 统一给两条：

```yaml
phase: execution
local: claude-code-opus
server: codex-high
```

- `local`、`server` 各写一个 agent id，或写 `manual`
- 两条独立计算，互不参照。服务器选不出来时写 `manual`，不影响本机那条
- `manual` 表示由当前会话或人工在这台机器上完成，不需要注册 agent

`/to-tickets` 把这两条写进 ticket。`/dispatch` 按当前所在机器取其中一条。

## 算法

对 `local` 和 `server` 各跑一遍同一套流程，只是候选池按 `host` 过滤。

### 1. 读 ticket

拿到 `phase` 和任务内容。phase 决定评分权重，任务内容决定 tags 匹配。

| phase | 主要看 | 说明 |
|-------|--------|------|
| `planning` | `strength` | 需求澄清和拆分，判断力比速度重要 |
| `research` | `strength` + `tags` | 需要读文档、读源码、下判断 |
| `execution` | `tags` + `speed` | 领域匹配优先，其次看吞吐 |
| `verification` | `speed` + `tags` | 跑命令为主，快一点有明显收益 |
| `review` | `strength` | 挑错需要更强推理，且希望换一个 agent 来看 |

### 2. 过滤候选池

按 `host` 取出这一侧的 agent。**不做任何淘汰**，`strength: low` 也留在池子里，只是排后面。

如果这一侧一个 agent 都没有，直接写 `manual`，说明原因。

### 3. 排序

依次比较：

1. **tags 命中数**：ticket 涉及的领域与 agent `tags` 的交集大小
2. **strength**：`high` > `medium` > `low`
3. **speed**：`fast` > `medium` > `slow`

phase 会调整先后。`execution` 和 `verification` 先比 tags 再比 speed，strength 随后；`planning`、`research`、`review` 先比 tags 再比 strength，speed 随后。

排序结果唯一时直接采用。前两名在 tags 命中数和 strength 上都相同时，也是并列，取 speed 更快者；speed 也相同时向用户报告并列，不要随便挑一个。

### 4. 关键任务提高强度要求

任务涉及以下任一情况时，`strength: low` 的 agent 在排序中降到所有 `high` 和 `medium` 之后，即使它的 tags 命中更多：

- 认证、加密、权限、支付
- 数据迁移、破坏性变更、schema 调整
- 并发、分布式一致性
- 疑难 bug 的根因定位

这仍然是排序规则，不是淘汰规则。这一侧只有 `low` 的 agent 时，可用它将就，并在输出里说明。

### 5. review 阶段换人

`phase: review` 时，优先选与 `phase: execution` 推荐不同的 agent。同一台机器上如果只有那一个 agent，就照用，并注明「同 agent 自审」。

## 预检报告

对一批 ticket 一次性输出：

```text
[001] init-db-schema
  phase: execution
  local:  claude-code-opus      （tags 命中 schema、migration）
  server: codex-high            （tags 命中 schema、migration）

[002] user-model
  phase: execution
  local:  claude-code-opus      （tags 命中 backend）
  server: codex-high

[003] update-readme
  phase: execution
  local:  copilot-fast          （tags 命中 docs，fast）
  server: manual                （服务器上没有 tags 命中 docs 的 agent）
```

每行给一句理由，理由必须来自注册表里真实存在的字段，不要编。

## 并列和缺失怎么报告

并列：

```text
[007] design-auth-interface
  phase: planning
  local:  并列 claude-code-opus / other-high
  server: 并列 codex-high / other-high
  原因: 两组 tags 命中数和 strength 都相同，speed 也相同
  需要用户决定: 选一个，或修改 agents.json 里的 tags 让它唯一
```

这一侧没有 agent：

```text
[010] update-docs
  phase: execution
  local:  manual
  server: manual
  原因: agents.json 里没有任何 agent 的 tags 命中 docs
  影响: 由当前会话或人工完成，没有 agent 承担
```

## ticket 里的值失效时

发现 `local:` 或 `server:` 指向的 id 在 agents.json 里不存在：

1. 视为失效，在预检报告里标出来
2. 重新算这一侧
3. 把新值写回 ticket 的 route 块
4. 在报告里说明改了什么

不要留着死引用，也不要静默跳过。

## 升级路径

执行中发现 ticket 需要更强的 agent：

1. 更新 ticket 的 `local:` 或 `server:`
2. 重新运行本 skill 确认
3. 如果这一侧没有更合适的 agent，如实说明，不要把 `manual` 说成「最优解」

## 反模式

- 🚫 输出 runtime、adapter 或任务系统信息
- 🚫 只给一条推荐
- 🚫 因为 `strength: low` 就把 agent 从候选池里删掉
- 🚫 两台机器用同一条推荐互相复制，不各自计算
- 🚫 编造理由，写注册表里没有的 tags
- 🚫 在本 skill 里写死任何机器名、IP 或具体 agent id

## 与其它 skill 的衔接

- **`/setup-agents`** 维护 agents.json，提供本 skill 需要的全部字段
- **`/to-tickets`** 把两条推荐写进 ticket 的 route 块
- **`/dispatch`** 按当前所在机器取 `local:` 或 `server:`
- **`/verify`、`/code-review`** 用各自 phase 重新走一遍本 skill
