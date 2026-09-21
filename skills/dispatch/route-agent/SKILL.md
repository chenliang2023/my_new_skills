---
name: route-agent
description: 读取 .workflow/agents.json，读 ticket 内容后判断谁真正适合，为本机和服务器各给一条推荐。关键是保证强模型承担关键与复杂任务，不用计分排序。
---

# Route Agent

ticket 需要两条推荐：**本机最佳** 和 **服务器最佳**。本 skill 负责判断出来。

推荐只在两台机器各选一个 agent，不涉及任务系统、队列或 adapter。执行方式在整条工作流里已经固定为 git worktree，所以路由只需要回答一个问题：这件事交给谁最合适。

## 这是判断，不是打分

读 ticket，然后回答「这台机器上谁真正适合做这件事」。

**不要用**「tags 命中数」这类计分方式排序，也不要拿 `strength` 和 `speed` 算总分。tags 和 strength 是判断的输入，不是分数。

两条要求：

- **关键和复杂任务优先交给 `strength: high` 的 agent。** 本侧有 high 就不要跳过它。
- **本侧没有 high 时，用本侧最强的可用 agent 顶上去，在理由里注明降级。** 不写 `manual`，不把 ticket 挂起来。

另外：**常规和机械任务要落在适合它的 agent 上。** 不要为了“保险”把小事推给最强的慢模型，也不要让不搭领域的 agent 硬接。

## 数据来源

只有一份注册表：`.workflow/agents.json`。每个 agent 记录 `harness`、`model`、`host`、`strength`、`speed`、`tags`。

- `host` 决定它属于哪台机器的候选池
- `strength` 决定它能接什么档位的活
- `tags` 是领域线索，告诉你它擅长什么
- `speed` 只在机械档位和同档位内部才起作用
- 本机和服务器能力相同，不存在「这台机器不能做这个阶段」

没有 runtime 注册表，也没有 adapter 概念。不要去找它们。

## 输出格式

所有 ticket 统一给两条：

```yaml
phase: execution
local: codex-gpt
server: oh-my-pi-gpt
```

- `local`、`server` 各写一个 agent id，或写 `manual`
- 档位和适合性理由写在预检报告里，不写进 ticket。ticket 只带两个值
- 两条独立判断，互不参照
- `manual` 只用于两种情况：本侧一个 agent 都没有，或用户明确要求由人工完成。**不是「本侧没有 high」的理由**

`/to-tickets` 把这两条写进 ticket。`/dispatch` 按当前所在机器取其中一条。

## 判断分三步

对 `local` 和 `server` 各跑一遍，候选池按 `host` 过滤。

### 第一步：定档位

档位决定**优先交给什么样的人**。看 ticket 的 `## 🎯 任务`、`## ✅ 验收标准`、`## 🧩 上下文`，以及它阻塞了谁。

| 档位 | 判据 | 优先级 |
|------|------|----------|
| **关键** | 认证、加密、权限、支付；数据迁移或破坏性 schema 变更；并发与一致性；无法稳定复现的疑难 bug | 优先 `high` |
| **复杂** | 跨模块重构；新架构或新子系统；性能瓶颈定位；多子系统协调 | 优先 `high` |
| **常规** | 单模块内实现功能、加接口、补测试、按已有模式扩展 | `medium` 及以上 |
| **机械** | 重命名、格式化、改文档、改配置值、样板代码 | 不限，优先 `speed: fast` |

档位跟着**任务的实质**走，不跟 ticket 的体量走。一个只改十行代码的并发 bug 是关键档，一个写三百行样板的前端页面是常规档。

判定拿不准时往高档靠，并在理由里说明你的顾虑。

### 没有 high 时就降级，不要卡住

真实情况里本侧经常没有 high。这时候正常往下走：

1. 在本侧现有 agent 里选**最可能胜任**的那个（`medium` 优先，`low` 最后）
2. 在理由里写清降级：

```text
理由: 这个 ticket 是复杂档，理想是本侧 high；服务器侧只有 medium，
      交给 oh-my-pi-gpt（medium）顶上，它擅长 backend
      风险: 跨模块重构的判断力可能不够，如果发现方向反复就停下来报缺口
```

3. 如果降级后仍不放心，可以**额外**提一句「建议本侧补一条 high」，但不要因此挂起 ticket

**不要因为本侧没有 high 就写 `manual`。** 有人能接就用人，把缺口写在报告里，由用户决定要不要补 agent。

### 第二步：判断适合

过了档位判断后，在候选里选**真正适合这件事**的那个。逐个问自己：

- 它擅长的领域和这个 ticket 对得上吗？前端活不要给只做后端的
- 它的强项正好是这个 ticket 的难处吗？疑难 bug 要的是排查能力，不是出活速度
- 这台机器上还有没有更能胜任的？如果有，为什么不是它
- 一个 ticket 连续做了几步时，下一步是否该换人
- `phase: review` 时是否该换一个眼睛？用 execution 同一个 agent 自审价值有限

`speed` 只在两处起作用：机械档位选谁，以及同档位内部都合格时谁先上。**不要让它盖过适合性判断。**

### 第三步：写理由

理由要说清「为什么它适合这件事」。

好理由：

```text
理由: 这个 ticket 要在 pg_trgm 和 tsvector 之间做取舍，属复杂档；
      codex-gpt 是这一侧唯一的 high，且它的 debugging 标签对应这种需要读文档做判断的活
```

坏理由：

```text
理由: tags 命中 backend、test 两个（计分式，没说清为什么适合）
理由: 它最快（把 speed 当成了决定因素）
```

## 预检报告

对一批 ticket 一次性输出。每行给出档位、两侧推荐、一句适合性理由：

```text
[001] 建搜索索引 + 迁移脚本
  档位: 复杂（涉及 schema 变更和索引选型）
  local:  codex-gpt       理由: 复杂档；它在本机是唯一 high，且擅长迁移类判断
  server: oh-my-pi-gpt    理由: 本侧无 high，用最强者顶上；它擅长 backend
                          ⚠️ 降级：复杂档交给 medium，重构方向反复就报出来

[002] 搜索 API
  档位: 常规（按已有模式加接口）
  local:  claude-domestic 理由: 常规档；它擅长 backend 且出活快，不必占用 codex-gpt
  server: oh-my-pi-gpt    理由: 常规档对口，medium 足够

[003] 搜索 UI
  档位: 常规
  local:  claude-domestic 理由: 本机唯一覆盖前端
  server: antigravity-gemini 理由: 它的前端强项正好对口

[004] 更新 README
  档位: 机械
  local:  claude-domestic 理由: 机械档，但它已覆盖 docs 且是本机最快
  server: oh-my-pi-gpt    理由: 两侧均可，它已覆盖 docs
```

理由必须针对这个 ticket 说话，不要写成通用评价。

## 需要用户决定时

**两个候选真的难分：**

```text
[009] 重构搜索层
  档位: 复杂
  local:  难分 codex-gpt / other-high
  原因: 两者都够强且领域都沾边，说不清谁更合适
  需要用户决定: 选一个，或给 agents.json 补一条更具体的 tags
```

**本侧一个 agent 都没有：**

```text
[010] 修文档链接
  档位: 机械
  local:  copilot-fast
  server: manual          理由: agents.json 里没有任何 host: server 的记录
```

不要为了交差随便挑一个。说不清就说不清。

## ticket 里的值失效时

发现 `local:` 或 `server:` 指向的 id 在 agents.json 里不存在：

1. 视为失效，在预检报告里标出来
2. 重新算这一侧
3. 把新值写回 ticket 的 route 块
4. 在报告里说明改了什么

不要留着死引用，也不要静默跳过。

## 升级路径

执行中发现 ticket 的档位判断错了（实际比预期难）：

1. 把档位调高一档
2. 本侧有 high 就换给它；没有 high 就维持当前 agent，并把这个缺口写进报告
3. 卡住不动且方向反复时，停下来报缺口，不要硬耗

## 反模式

- 🚫 用计分或排序的方式选 agent，而不是读 ticket 后判断谁适合
- 🚫 本侧有 high 却没用上
- 🚫 降级了但不写出来，让人以为这本就是最佳安排
- 🚫 因为本侧没有 high 就写 `manual`，把能做的 ticket 挂起来
- 🚫 为了省事把机械活推给最慢最强的 agent
- 🚫 把 `speed` 当成主要决定因素
- 🚫 把不搭领域的 agent 硬配给 ticket，只因为它闲着
- 🚫 输出 runtime、adapter 或任务系统信息
- 🚫 只给一条推荐
- 🚫 两台机器用同一条推荐互相复制，不各自判断
- 🚫 编造理由，或写注册表里没有的 tags
- 🚫 说不清谁更合适时随便挑一个
- 🚫 在本 skill 里写死任何机器名、IP 或具体 agent id

## 与其它 skill 的衔接

- **`/setup-agents`** 维护 agents.json，提供本 skill 需要的全部字段
- **`/to-tickets`** 把两条推荐写进 ticket 的 route 块
- **`/dispatch`** 按当前所在机器取 `local:` 或 `server:`
- **`/verify`、`/code-review`** 用各自 phase 重新走一遍本 skill
