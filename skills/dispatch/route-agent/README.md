# Route Agent

完整规则见同目录 `SKILL.md`。

核心：读 ticket 判断档位（关键 / 复杂 / 常规 / 机械），为本机和服务器各给一条推荐。
关键和复杂档**优先交给 `strength: high`**；本侧没有 high 就用最强者顶上并在理由里注明降级，不写 `manual`。
用判断，不用计分排序。数据只来自 `.workflow/agents.json`。
