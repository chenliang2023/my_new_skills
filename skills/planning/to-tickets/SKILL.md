---
name: to-tickets
description: 将 spec 拆成自包含的 tracer-bullet ticket，声明阻塞边，并给出本机和服务器各自的最佳 agent 推荐，写入 .workflow/tickets/。
disable-model-invocation: true
---

# To Tickets

把 spec 拆成可以直接开工的 ticket。每个 ticket 是自包含的：拿到单个 ticket 的 agent 或当前会话能直接开始，不需要猜测环境，也不需要重读整份 spec。

每个 ticket 给出**两条 agent 推荐**：本机一条，服务器一条。哪台机器执行就取哪条。

## 前提

- 已有一份 spec 在 `.workflow/specs/` 下
- `.workflow/config.json` 存在
- `.workflow/agents.json` 存在（推荐全部为 `manual` 时可缺）
- 拆完后运行 `/route-agent` 校验推荐

## 流程

1. **读 spec**：识别实现功能需要的工作单元和外部行为
2. **拆分**：每个 ticket 是一个 tracer bullet，端到端贯穿一个行为，不做横向切片
3. **声明阻塞边**：列出必须先完成的 ticket，无依赖就写「无」
4. **路由**：判断这个 ticket 的档位（关键 / 复杂 / 常规 / 机械），再为它选本机和服务器各自的推荐 agent。拿不准就交给 `/route-agent`，不要凭感觉填
5. **写文件**：feature ticket 写入 `.workflow/tickets/`，bug fix ticket 写入 `.workflow/bugs/tickets/`

## route 块

所有新 ticket 在状态行后用统一格式：

```markdown
<!-- status: todo -->
<!-- route:
 phase: execution
 local: claude-domestic
 server: oh-my-pi-gpt
-->
<!-- released: <version> -->
```

| 字段 | 含义 |
|------|------|
| `phase` | `planning`、`research`、`execution`、`verification` 或 `review`，多数 ticket 是 `execution` |
| `local` | 本机上的 agent id，或 `manual` |
| `server` | 服务器上的 agent id，或 `manual` |

规则：

- 两条独立写，互不参照。服务器选不出来时写 `manual`，不影响本机那条
- 值必须是 `.workflow/agents.json` 里存在的 id，或 `manual`
- 不写 `display_name`，不写机器名、IP、路径
- **关键和复杂档优先选 `strength: high` 的 agent。** 本侧没有 high 时选最强者，并在理由里注明降级，不要写 `manual` 把 ticket 挂起来
- 拿不准时先写 `manual`，再运行 `/route-agent` 补齐
- 写 `manual` 表示由当前会话或人工完成，不需要注册 agent

## ticket 模板

```markdown
<!-- status: todo -->
<!-- route:
 phase: execution
 local: claude-domestic
 server: oh-my-pi-gpt
-->
<!-- released: <version> -->

# [<ticket-id>] <标题>

## 📌 Spec 引用
来源：`.workflow/specs/<spec-name>.md` 的 <相关章节>

## 🎯 任务
<一句话说清完成后的可观察行为，再补上现在的行为。不读 spec 也能理解目标。>

## ✅ 验收标准
1. <可观察条件，能对应一个命令、请求或操作>
2. ...

## 🧪 测试 seam
<只写外部行为的测试边界>

## 🚧 阻塞
- 依赖：[<其它 ticket-id>] <标题>（必须先完成）
- 或：无（可立即开始）

## 🧭 路由
- Phase：`execution`
- 档位：`常规`
- 本机：`claude-domestic`
- 服务器：`oh-my-pi-gpt`
- 理由：<为什么它适合这件事，一句话>

## 🌿 工作树
- 目录：`.worktrees/<ticket-id>/`
- 分支：按 `config.json` 的 `branch_template`，如 `ticket/001-user-auth`

## 🧩 上下文
<agent 需要知道的代码结构、术语和约束。能从代码里读出来的不要抄。>

<跨模块调用、状态迁移或数据流可放一张不超过九个节点的 Mermaid 图。>
```

## 三个字段的写法

**`## 🎯 任务`** 必须描述起点和终点。不要只写「实现用户认证模块」，要写清输入、输出、错误和当前 stub。

**`## ✅ 验收标准`** 写成可观察行为，不写「功能正常」或「代码质量良好」。每条都要能对应命令、请求或点击。

**`## 🧩 上下文`** 只写 agent 在现场拿不到的东西。技术方案在 spec 里已经论证过，这里给落点，不重新论证。

**`## 🧭 路由`** 的理由只写一句，要说清「为什么它适合这件事」，不是数 tags。关键和复杂档要说明为什么这个 ticket 属于该档。

## 路径

- feature ticket：`.workflow/tickets/<id>-<slug>.md`
- release ticket：`.workflow/version/tickets/v<version>.md`，由 `/version` 创建
- bug fix ticket：`.workflow/bugs/tickets/<id>-fix-<slug>.md`

## 拓扑排序输出

拆完后输出依赖关系和每个 ticket 的推荐：

````markdown
```mermaid
flowchart LR
  subgraph W1["第 1 批 · 可立即开始"]
    T001["[001] init-db-schema<br/>local / server"]
    T003["[003] research-auth-lib<br/>phase: research"]
  end
  subgraph W2["第 2 批"]
    T002["[002] user-model<br/>local / server"]
  end
  T001 --> T002
  T003 --> T002
```
````

同时给出可复制的分派清单：

```markdown
🟢 可立即开始
- [001] init-db-schema → local: claude-code-opus / server: codex-high

🟡 等待 [001]
- [002] user-model → local: claude-code-opus / server: codex-high
```

不要把示例里的 agent id 当成项目默认值，实际值必须来自当前 agents.json。

## 回报给用户

按 `/readable-docs` 回复：给出拆出几个、谁阻塞谁、哪些推荐是明确算出来的、哪些还是 `manual` 待补。不要复述整批 ticket。

## 下一步

- 提交 `.workflow/` 到 git
- 推荐不确定或有 `manual` 时运行 `/route-agent` 补齐
- 运行 `/dispatch` 开始执行

## 反模式

- 🚫 只写一条推荐
- 🚫 服务器那条直接抄本机的值，不单独计算
- 🚫 写机器名、IP 或 `display_name` 当 agent 引用
- 🚫 为了看起来具体就编一个不存在的 agent id
- 🚫 跳过阻塞边声明
