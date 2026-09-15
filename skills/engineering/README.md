# Engineering Skills

工程实践规范。model-invoked，agent 实现代码时自动遵守，不需要用户手动调用。

## Skills

- **[tdd](./tdd/SKILL.md)**：测试驱动开发。红绿循环、好测试的标准、seam 选择、反模式。服务器端 agent 实现 ticket 时遵守。
- **[diagnosing-bugs](./diagnosing-bugs/SKILL.md)**：系统性调试。先建紧凑反馈循环再修复，分阶段定位根因，最后写 regression 测试。
- **[code-review](./code-review/SKILL.md)**：双轴代码复审（规范符合 + 工程标准）。本机拉取服务器结果后运行，作为交付前最终检查。
- **[context-sync](./context-sync/SKILL.md)**：本机与服务器 CodeG 之间的上下文同步规范。通过 `.workflow/` 目录同步文档，通过 CodeG 的 Skill 管理同步 skills。