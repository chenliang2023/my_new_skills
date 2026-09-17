# Planning Skills

本机（VS Code）规划阶段使用的 skills。用户调用，产出文档供服务器端消费。

## Skills

- **[grill-me](./grill-me/SKILL.md)**：通过 relentless interview 打磨一个想法、计划或设计，直到它足够清晰可以交由服务器端 agent 实现。多轮提问，每轮推进 frontier。规划入口。
- **[research](./research/SKILL.md)**：派发后台 agent 调研技术问题，产出 Markdown 存入 `.workflow/research/`。查 primary source，每个结论附来源链接和章节号，查证过程单独成节，涉及并发或状态迁移时画 Mermaid 图。
- **[to-spec](./to-spec/SKILL.md)**：将对话或 grill 摘要综合成正式 spec，存入 `.workflow/specs/`。不重新 interview，只做综合。
- **[to-tickets](./to-tickets/SKILL.md)**：将 spec 拆成 tracer-bullet ticket，每个声明阻塞边和建议 agent，写入 `.workflow/tickets/`。供服务器端 CodeG 并发分派。

这些 skill 产出的文档都要遵守 **[readable-docs](../engineering/readable-docs/SKILL.md)**：材料不够就不写，写了就要让人能读懂。模板给结构，`readable-docs` 给填充质量。