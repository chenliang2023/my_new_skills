---
name: diagnosing-bugs
description: 系统性调试。当遇到 bug、测试失败或意外行为时使用，在提出修复前先建立紧凑反馈循环。任何 runtime 中的 agent 都应遵守。
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

这些文件通过 git 在 runtime 之间共享，不要求修复一定发生在某个环境。

## 核心原则：先有反馈循环再修复

紧凑反馈循环是一个现在就能在这个 bug 上失败的命令。如果没有命令能失败，先写复现测试或脚本，再开始修复。

## 流程

1. 建立反馈循环，连续运行 3 次确认稳定失败
2. 用测试或命令定位范围
3. 写清根因，不只写症状
4. 先写 regression 测试，再做最小修复
5. 确认反馈循环、regression 和全量测试都通过
6. 写 post-mortem，记录如何防止同类 bug

## 与 ticket 和 route 的关系

修复必须走 bug fix ticket。ticket 的 `## 🧭 路由` 可以指定运行时、adapter、agent 和所需能力。没有指定时，按 `/route-agent` 动态匹配，不默认某个环境或 agent。

## 禁止

- 没有红测试就改代码
- 用临时 print 或 console.log 猜测修复
- 没定位根因就提出方案
- 跳过 regression 测试或 post-mortem
