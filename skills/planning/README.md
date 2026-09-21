# Planning Skills

规划、调研和交付物准备阶段使用的 skills。可以在任何具备相应能力的 runtime 中调用，不固定由本机或 CodeG 承担。

## Skills

- **[grill-me](./grill-me/SKILL.md)**：通过 relentless interview 打磨一个想法、计划或设计，直到它足够清晰可以写 spec。
- **[research](./research/SKILL.md)**：派发后台 agent 调研技术问题，产出 Markdown 存入 `.workflow/research/`。
- **[to-spec](./to-spec/SKILL.md)**：将对话或 grill 摘要综合成正式 spec，存入 `.workflow/specs/`。
- **[to-tickets](./to-tickets/SKILL.md)**：将 spec 拆成自包含的 tracer-bullet ticket，声明阻塞边和 canonical route，写入 `.workflow/tickets/`。
- **[version](./version/SKILL.md)**：管理版本、CHANGELOG 和 release 归档，可在任意适合的 runtime 手动调用。

这些 skill 产出的文档都要遵守 **[readable-docs](../engineering/readable-docs/SKILL.md)**：材料不够就不写，写了就要让人和 agent 都能读懂。
