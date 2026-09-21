---
name: diagnosing-bugs
description: 系统性调试。当遇到 bug、测试失败或意外行为时使用，在提出修复前先建立紧凑反馈循环。所有 agent 都应遵守。
---

# Diagnosing Bugs

遇到难搞的 bug：第一眼看不出来、间歇性 flake，或两个已知良好状态之间潜入的 regression。本 skill 把调试纪律封装成分阶段循环。

## 产物落点

```text
.workflow/bugs/
├── repros/<date>-<slug>.md
├── postmortems/<date>-<slug>.md
└── tickets/<id>-fix-<slug>.md
```

这些文件通过 git 在本机和服务器之间共享，不要求修复一定发生在某台特定的机器上。

## 核心原则：先有反馈循环再修复

紧凑反馈循环是一个现在就能在这个 bug 上失败的命令。如果没有命令能失败，先写复现测试或脚本，再开始修复。

## 流程

1. 建立反馈循环，连续运行 3 次确认稳定失败
2. 用测试或命令定位范围
3. 写清根因，不只写症状
4. 先写 regression 测试，再做最小修复
5. 确认反馈循环、regression 和全量测试都通过
6. 写 post-mortem，记录如何防止同类 bug

## 与 ticket 和推荐 agent 的关系

修复必须走 bug fix ticket。ticket 的 route 块给出本机和服务器两条推荐 agent，按当前机器取一条；还没写时运行 `/route-agent`。

疑难 bug 的根因定位属于关键档，优先给本侧 `strength: high` 的 agent。毛病看不出来往往就是因为它难，拿快模型硬试只会浪费时间；本侧没有 high 时用最强者顶上并注明降级。

## 禁止

- 没有红测试就改代码
- 用临时 print 或 console.log 猜测修复
- 没定位根因就提出方案
- 跳过 regression 测试或 post-mortem
