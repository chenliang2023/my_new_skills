# Planning Skills

本机（VS Code）规划阶段使用的 skills。用户调用，产出文档供服务器端消费。

## Skills

- **[grill-me](./grill-me/SKILL.md)**：通过 relentless interview 打磨一个想法或计划，直到它足够清晰可以交由服务器端 agent 实现。规划入口。
- **[research](./research/SKILL.md)**：派发后台 agent 调研技术问题，产出 Markdown 存入 `.workflow/research/`。
- **[to-spec](./to-spec/SKILL.md)**：将对话或 grill 摘要综合成正式 spec，存入 `.workflow/specs/`。
- **[to-tickets](./to-tickets/SKILL.md)**：将 spec 拆成 tracer-bullet ticket，每个声明阻塞边，写入 `.workflow/tickets/`。