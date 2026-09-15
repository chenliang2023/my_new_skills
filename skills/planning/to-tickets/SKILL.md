---
name: to-tickets
description: 将 spec 拆成一组 tracer-bullet ticket，每个 ticket 声明自己的阻塞边，写入 .workflow/tickets/。供服务器端 CodeG 并发分派。
disable-model-invocation: true
---

# To Tickets

把 spec 拆成一组可以在 CodeG 上并发执行的 ticket。每个 ticket 是自包含的：服务器端 agent 拿到单个 ticket 就能干活，不需要读取整个 spec。

## 前提

- 已有一份 spec 在 `.workflow/specs/` 下。
- `.workflow/config.json` 存在（读取分支模板等配置）。

## 流程

1. **读 spec**：通读 spec，识别出实现这个功能需要的工作单元。

2. **拆分原则**：
   - 每个 ticket 是一个 **tracer bullet**：端到端贯穿一个行为，不是横向切片（不是"先写所有 model，再写所有 controller"）。
   - 每个 ticket 自包含：包含足够上下文让一个新 agent session 能直接干活。
   - 每个 ticket 声明 **阻塞边**：它依赖哪些其它 ticket 先完成。
   - 互不依赖的 ticket 可以并发分派给不同 agent。

3. **agent 分派建议**：根据 `.workflow/config.json` 的 `agent_routing`，为每个 ticket 标注建议的 agent（Claude / PI / DeepSeek / AntiGravity）。如果配置为空，根据 ticket 内容自动判断。

4. **写 ticket 文件**：每个 ticket 一个文件，写入 `.workflow/tickets/`。

## ticket 模板

```markdown
# [<ticket-id>] <标题>

## Spec 引用
来源：`.workflow/specs/<spec-name>.md` 的 <相关章节>

## 任务
<这个 ticket 要做什么，agent 视角的描述。自包含，不需要 agent 去读 spec。>

## 验收标准
1. <可测试的条件>
2. ...

## 测试 seam
<在这个 seam 上测试，只测外部行为>

## 阻塞
- 依赖：[<其它 ticket-id>] <标题>（必须先完成）
- 或：无（可立即开始）

## 建议 Agent
<Claude | PI | DeepSeek | AntiGravity>
理由：<一句话>

## 分支
<按 config 的 branch_template，如 ticket/001-user-auth>

## 上下文
<agent 需要知道的代码结构、术语、ADR 引用。复制到这里，不要让 agent 自己去找。>
```

## 路径

ticket 文件存入 `.workflow/tickets/<id>-<slug>.md`，id 用三位数字序号，slug 用 kebab-case。

## 拓扑排序输出

拆完后，输出一个执行顺序图，让用户看到哪些 ticket 可以并发、哪些有依赖：

```
可立即开始：
  [001] init-db-schema (PI)
  [003] research-auth-lib (DeepSeek)

依赖 [001]：
  [002] user-model (Claude)
  [004] user-api (Claude)

依赖 [002] [004]：
  [005] login-ui (AntiGravity)
```

## 下一步

告知用户：
- 将 `.workflow/` 目录提交到 git 并推送到服务器仓库
- 在服务器 CodeG 上运行 `/dispatch` 开始并发执行