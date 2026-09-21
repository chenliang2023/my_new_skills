---
name: tdd
description: 测试驱动开发。当用户要构建功能或修复 bug 时使用。任何 runtime 中实现 ticket 的 agent 都应遵守。
---

# Test-Driven Development

TDD 是红 → 绿循环。本 skill 说明怎样让这个循环产出值得保留的测试。

## 什么是好测试

测试通过公共接口验证行为，不验证实现细节。代码可以重写，测试不应因此失效。

## Seam

`seam` 是观察行为的公共边界。只在 ticket 的 `## 测试 seam` 上测试，不深入内部。

## 反模式

- mock 内部协作者、测试私有方法、绕过公共接口
- 用代码同样的方式重算期望值
- 先批量写测试再批量写实现，失去反馈

## 循环规则

- 红先于绿
- 一次一个纵向切片
- 重构放在 review 阶段，不混进红绿循环

## 在本工作流中的位置

任何具备 `execute` 能力的 runtime 都可实现 ticket，但 agent 需要遵守同一套 TDD 纪律。`/verify` 负责最终全量检查。
