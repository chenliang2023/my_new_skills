# Skills 仓库说明

本仓库是一套可在多个 runtime 中运行的开发研究工作流 skill 集合，参考 [mattpocock/skills](https://github.com/mattpocock/skills) 的结构。CodeG 是一个可选 runtime，它通过 `codeg-todos` adapter 提供多 agent 任务能力。本机工作区也可以独立承担规划、执行、验证和 review。

## 目录组织

Skills 按 bucket 文件夹组织：

- `ask/`：根技能，所有 skill 的路由器
- `planning/`：规划、调研、spec 和 ticket 产出
- `dispatch/`：按 ticket 分派和执行任务
- `engineering/`：工程实践规范
- `setup/`：路径、agent 和 runtime 注册表初始化

每个 skill 文件夹内含一个 `SKILL.md`，使用 YAML frontmatter 声明 `name`、`description`、是否 `disable-model-invocation`。

- `disable-model-invocation: true`：用户调用，负责编排和明确选择
- 无此字段：模型调用，承载可复用纪律

## Runtime 架构

本仓库不把“规划端”和“执行端”绑定到某个地点。任何已注册且具备相应能力的 runtime 都可以运行 planning、research、dispatch、execute、verify、review 或 integrate。

- `.workflow/agents.json`：Agent 注册表的唯一真相源，只描述稳定 `id`、显示名、可选角色、标签、成本和描述，不描述运行环境
- `.workflow/runtimes.json`：Runtime、adapter、能力、agent policy 和阶段默认路由的唯一真相源
- `.workflow/config.json`：只保存路径类配置
- `.workflow/`：跨 runtime 共享并提交到 git 的上下文载体

CodeG runtime 的 To-dos、git worktree 隔离、并发、Review/Merge、`@` 委托和 preflight 都作为 `codeg-todos` adapter 的可选能力。它们不是所有 runtime 必须提供的工作流前提。本机 adapter 也可以声明其中一部分或全部能力。

## Agent

Agent 的名称、数量、角色和可用 runtime 都由项目配置决定。不要在文档或 skill 中假设某个固定 agent、单一入口或固定模型。

- Agent 用稳定的 kebab-case `id` 引用
- `display_name` 只用于展示，可以修改
- `role` 是可选的人读分类，`tags` 和 `cost_tier` 用于动态路由
- 不再使用单一 `manual_entry`。阶段默认入口写在 `.workflow/runtimes.json`，也可以由 ticket 或本次调用覆盖
- `agent_id: null` 表示当前用户会话、脚本或人工执行，不需要注册 agent

## 主工作流

1. 选择或自动匹配一个具备 `interactive`、`plan` 或 `research` 能力的 runtime
2. `/grill-me` 澄清需求，必要时 `/research`
3. `/to-spec` 产出 spec
4. `/to-tickets` 拆成自包含 ticket，并写入 phase、runtime、adapter、agent 和能力约束
5. 选择具备 `execute` 或 `dispatch` 能力的目标，运行 `/dispatch`
6. 按目标 adapter 提供的能力执行任务、并发、隔离、review 和 merge
7. `/verify` 在具备 `verify` 能力的目标上全量验证
8. `/code-review` 在任意具备 `review` 能力的 runtime 上复审，必要时通过 `.workflow/` 交接
9. `/version` 在验证通过后处理 release

如果路由没有唯一结果，询问用户，不要暗中回退到某个地点或某个 agent。

## 约定

- `.workflow/` 目录是共享上下文载体，必须提交到 git
- ticket 文件是 agent 或当前会话的工作单元，必须自包含
- ticket 的 runtime、adapter 和 agent 是三个独立维度
- `.workflow/` 下的文档遵守 `readable-docs`：先对人可读，再对 agent 可解析
- `CONTEXT.md` 让不同 runtime 和 agent 使用一致术语
- `/integrate` 只在目标 adapter 没有足够的 merge 能力，或需要跨 ticket 协调时补充使用
- 不使用 em-dash

## 首次使用

1. 运行 `/setup-workflow` 创建路径和目录
2. 运行 `/setup-agents` 配置可用 agent，允许为空
3. 运行 `/setup-runtimes` 配置本机、CodeG 或其它 runtime adapter
4. 运行 `/context-sync` 确认共享文档和 skills 已同步
5. 运行 `/ask` 或直接运行 `/grill-me` 开始工作
