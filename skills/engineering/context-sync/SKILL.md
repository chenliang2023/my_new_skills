---
name: context-sync
description: 在本机和服务器之间同步 .workflow 上下文和 skills，确保 spec、ticket、research、agent 清单和验证报告使用同一份内容。
---

# Context Sync

`.workflow/` 是跨机器的共享上下文载体。通过 git 同步文档和 agent 清单，通过各机器的 skill 安装机制同步 skills。

两台机器能力相同，没有主从之分。不要写「某台机器负责某个阶段」。

## 同步内容

```text
.workflow/
  config.json          # 路径配置
  agents.json          # 本机和服务器上的 agent 清单
  tickets/             # ticket
  specs/               # spec
  research/            # 调研文档
  handoffs/            # 交接文件和 verify 报告
  CONTEXT.md           # 术语表
  ADR/                 # 架构决策记录
```

这些文件必须提交到 git。ticket、spec、research 和报告不能依赖某台机器上的未提交状态。

不进 git 的东西：

- `.worktrees/`：每台机器自己的工作树，必须加入 `.gitignore`
- 各机器自己的 harness 配置、凭证和环境变量

## agent 清单在两台机器上的关系

`.workflow/agents.json` 同时包含 `host: local` 和 `host: server` 的记录。两台机器读同一份文件。

- 每台机器只执行属于自己 `host` 的记录
- 在 A 机器上新增 agent，提交后 B 机器也看得到，但 B 不会去跑它
- `host` 是记录属性，不是机器自报身份。哪台机器是 local 由使用者在执行时确定

在 B 机器上跑 `/dispatch` 时，ticket 的 `local:` 值对 B 没意义。B 用 `server:` 那条。

## 同步 skills

把本仓库的 `skills/` 安装到两台机器各自能读到的地方。安装位置由该机器自己的 harness 决定。

同步原则：

- 两台机器装同一份 skills，不要各自维护改动的副本
- `engineering/` 下的 skill 在所有会实现、验证或 review 的 agent 上都要可用
- agent 清单从 `.workflow/agents.json` 读取，不存在单独的可用性注册表

## 交接流程

### 机器 A → 机器 B

1. 确认 `.workflow/` 文档已保存，工作树里的改动已 commit
2. 确认 ticket 的 route 块里两台机器的推荐都有值
3. 在 A 上合并完 worktree，让主分支处于一致状态
4. `git add`、`commit`、`push`
5. B 拉取同一个 commit
6. B 运行 `/context-sync` 或至少检查 agents.json、spec、ticket 和 CONTEXT.md
7. 按 `/dispatch`、`/verify` 或 `/code-review` 继续，不需要重新解释上下文

### 在一台机器上改，在另一台上验

1. 改的机器合并完并推送
2. 验的机器拉取同一个 commit
3. 在验的机器上按 `phase: verification` 取推荐并执行
4. 报告写入 `.workflow/handoffs/` 并提交

同一个 commit 在两台机器上验出不同结果时，先查环境差异（依赖版本、环境变量、外部服务），不要先怀疑代码。

### 并行开发

两台机器可以同时推进**不同批次**的 ticket，因为每台在自己的 worktree 里工作。

规则：

- 同一台机器内部的并行由 `/dispatch` 按批次控制
- 跨机器并行时，两台不要同时往主分支合并。约定一方合并并推送后，另一方拉取再继续
- 同一批次的 ticket 尽量放在同一台机器上，减少合并交错

## 检查清单

- [ ] `.workflow/` 已提交到 git
- [ ] `.worktrees/` 已在 `.gitignore` 中
- [ ] agents.json 里 ticket 引用的 id 都存在
- [ ] ticket 的 `local:` 和 `server:` 都有值，没有失效引用
- [ ] 本轮新增或改动的文档按 `/readable-docs` 自检
- [ ] 目标机器已拉取同一 commit
- [ ] 没有 worktree 分支忘记合并或删除

## 常见故障

**ticket 里的 agent id 在某台机器上不存在**

不是同步问题，是 agents.json 缺少对应 `host` 的记录。运行 `/setup-agents` 补上，再 `/route-agent` 重算。

**两台机器看到的 spec 不同**

检查是否有一方改了 `.workflow/` 但没提交，或提交了但没推。

**merge 时冲突集中在 ticket 文件**

正常。两台机器都可能更新 ticket 的 status 和 route 行。合并时保留双方的有效信息，status 取更新的那个，route 取完整的那个。

## 反模式

- 🚫 把某台机器写成唯一生产者或唯一消费者
- 🚫 把 `.worktrees/` 提交进 git
- 🚫 两台机器各自维护一份 skills 副本并分别改动
- 🚫 只同步代码，不同步 `.workflow/`
- 🚫 两台机器同时往主分支合并
