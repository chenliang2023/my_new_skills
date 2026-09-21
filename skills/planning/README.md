# Planning Skills

规划、调研和交付物准备阶段使用的 skills。本机和服务器都能跑，用哪台就从 agents.json 里取那台的 agent。

## Skills

- **[grill-me](./grill-me/SKILL.md)**：通过 relentless interview 打磨一个想法、计划或设计，直到它足够清晰可以写 spec。
- **[research](./research/SKILL.md)**：派发后台 agent 调研技术问题，产出 Markdown 存入 `.workflow/research/`。
- **[to-spec](./to-spec/SKILL.md)**：将对话或 grill 摘要综合成正式 spec，存入 `.workflow/specs/`。
- **[to-tickets](./to-tickets/SKILL.md)**：将 spec 拆成自包含的 tracer-bullet ticket，声明阻塞边和本机/服务器两条 agent 推荐，写入 `.workflow/tickets/`。
- **[version](./version/SKILL.md)**：管理版本、CHANGELOG 和 release 归档。

这些 skill 产出的文档都要遵守 **[readable-docs](../engineering/readable-docs/SKILL.md)**：材料不够就不写，写了就要让人和 agent 都能读懂。
