# Skills 仓库说明

本仓库是一套开发研究工作流 skill 集合，参考 [mattpocock/skills](https://github.com/mattpocock/skills) 的结构。

模型很简单：**任务内容** 和 **谁来执行**。

## 本仓库没有 `.workflow/`

本仓库只是 skill 集合，自身不跑这套工作流，也不包含 `.workflow/`。

| 东西 | 放哪 | 谁提交 |
|------|------|--------|
| skills | 本仓库的 `skills/` | 本仓库 |
| 基线 agent 名单 | 本仓库 `skills/setup/setup-agents/agents.baseline.json` | 本仓库 |
| `.workflow/config.json`、`agents.json`、spec、ticket、报告 | **被管理项目的** `.workflow/` | 那个项目自己 |

`/setup-workflow`、`/setup-agents` 等 skill 都用在别的项目里，不在本仓库里跑。

基线名单之所以放在本仓库，是因为它要跳项目复用；`.workflow/agents.json` 属于每个项目自己，随项目提交。

## 目录组织

Skills 按 bucket 文件夹组织：

- `ask/`：根技能，所有 skill 的路由器
- `planning/`：规划、调研、spec、ticket 和版本
- `dispatch/`：推荐、分派和验证
- `engineering/`：工程实践规范
- `setup/`：路径配置和 agent 清单

每个 skill 文件夹内含一个 `SKILL.md`，使用 YAML frontmatter 声明 `name`、`description`、是否 `disable-model-invocation`。

- `disable-model-invocation: true`：用户调用，负责编排和明确选择
- 无此字段：模型调用，承载可复用纪律

## 执行模型

唯一的执行方式：**每个 ticket 一个 git worktree**。

- 工作树放在 `config.json` 的 `worktree_dir`（默认 `.worktrees/`），一个 ticket 一个目录
- 分支名按 `branch_template`（默认 `ticket/<id>-<slug>`）
- 完成后在自己的工作树里 commit，合回主分支，再删掉工作树和分支
- 合并是 `/dispatch` 流程的自然结尾，没有独立的集成阶段

不区分任务系统、队列或面板。`/dispatch` 读完 ticket 就能知道该做什么，不需要先判断「这个项目用的是哪种机制」。

## Agent

`.workflow/agents.json` 是唯一注册表，同时包含两台机器的记录。它属于被管理的项目，不在本仓库里。

每个 agent 只记录：

| 字段 | 含义 |
|------|------|
| `id` | 稳定的 kebab-case 引用名 |
| `harness` | 运行外壳，如 `claude-code`、`codex` |
| `host` | `local` 或 `server` |
| `strength` | `high`、`medium`、`low`。决定它能接什么档位的活 |
| `speed` | `fast`、`medium`、`slow` |
| `tags` | 擅长领域 |

`strength`、`speed`、`tags` 是主观判断，登记时向用户确认，不要凭 harness 名字猜。

`strength` 是**优先顺序**：关键和复杂任务优先交给 `high`，`medium` 接常规，`low` 接机械。每台机器尽量至少有一条 `high`；本侧没有 high 时路由会用最强者顶上并注明降级，不会把 ticket 挂起来。

本仓库的 `skills/setup/setup-agents/agents.baseline.json` 里有一份固定下来的基线名单，在项目里首次运行 `/setup-agents` 时复制成该项目的 `.workflow/agents.json`，再按实际情况增改。

基线有一处可以补强：**服务器侧没有 `strength: high` 的 agent**，关键和复杂档 ticket 在服务器上会降级给 `oh-my-pi-gpt`（medium），路由会注明降级。流程不会卡住，服务器上真的接了重要活时再补一条 high 即可。

不要记录任务系统、容量、并发数或安装路径。执行方式已经固定，这些维度不存在。

## 本机和服务器

两台机器**能力完全相同**，都能规划、调研、执行、review 和验证。差别只有各自装了哪些 agent。

- 没有 runtime 注册表，也没有 adapter 概念
- 不从机器名字推断能力，也不把某个阶段绑到某台机器
- 两台机器读同一份 `agents.json`，各自只执行属于自己 `host` 的记录

## 两条推荐

每个 ticket 的 route 块给两条，独立判断：

```yaml
phase: execution
local: codex-gpt
server: oh-my-pi-gpt
```

`/route-agent` 先给 ticket 定档位，再判断这一侧谁适合：

| 档位 | 优先交给 |
|------|----------|
| 关键（认证、迁移、并发、疑难 bug） | `high` |
| 复杂（跨模块重构、新架构、性能定位） | `high` |
| 常规（单模块实现、加接口、补测试） | `medium` 及以上 |
| 机械（重命名、格式化、文档） | 不限，优先 `fast` |

这是优先顺序，不是通行证。本侧没有 `high` 时用本侧最强者顶上，并在报告里注明降级，不写 `manual` 把 ticket 挂起来。

它是判断，不是计分：读 ticket 后说清「为什么它适合这件事」，而不是数 tags 命中数或拿 strength 和 speed 算总分。`speed` 只在机械档位和同档位内部起作用。

`manual` 表示由当前会话或人工完成，不需要注册 agent。

`/dispatch` 只取当前机器那一侧的值。在服务器上跑时 `local:` 的值无关。

## 主工作流

1. `/grill-me` 澄清需求，需要事实时 `/research`
2. `/to-spec` 产出 spec
3. `/to-tickets` 拆成自包含 ticket，写入阻塞边和两条 agent 推荐
4. `/route-agent` 预检，补齐 `manual` 或因注册表变化失效的推荐
5. `/dispatch` 排批次、建 worktree、选 agent、给工具建议，完成后合并
6. `/verify` 合并后全量验证，产出报告
7. `/code-review` 双轴复审
8. `/version` 验证通过后处理 release

不通过时走 `/diagnosing-bugs`，写 bug fix ticket 回到第 3 步。

## 约定

- `.workflow/` 是**被管理项目**里的共享上下文载体，必须提交到那个项目的 git
- `.worktrees/` 必须进被管理项目的 `.gitignore`
- ticket 必须自包含，拿到单个 ticket 就能开工
- agent 引用只写 `id`，不写 `display_name`，不写机器名或路径
- `.workflow/` 下的文档遵守 `readable-docs`：先对人可读，再对 agent 可解析
- `CONTEXT.md` 让不同 agent 使用一致术语
- 不改写本仓库里 skills 的结构去适配某个具体项目
- 不使用 em-dash

## 在一个新项目里启用

以下步骤都在**那个项目**里跑，不在本仓库里：

1. `/setup-workflow` 创建 `.workflow/`、路径配置，并把 `.worktrees/` 加进 `.gitignore`
2. `/setup-agents` 复制基线名单，再按实际情况增改
3. `/context-sync` 确认两台机器的共享文档和 skills 一致
4. `/ask` 或直接 `/grill-me` 开始工作
