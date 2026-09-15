# 我的开发研究工作流 Skills

参考 [mattpocock/skills](https://github.com/mattpocock/skills) 的结构，适配本人的双环境工作流。服务器端使用 [CodeG](https://github.com/xintaofei/codeg) 作为多 agent 编码工作台。

## 双环境架构

| 环境 | 角色 | 工具 | 职责 |
|------|------|------|------|
| **本机 (Windows + VS Code)** | 规划端 | Claude Code (VS Code 插件) | 研究、探索、规划、文档产出 |
| **服务器** | 执行端 | [CodeG](https://github.com/xintaofei/codeg) | 多 agent 并发开发、代码实现 |

核心思路：本机负责"想清楚"，服务器负责"做出来"。两端通过项目仓库中的 `.workflow/` 目录同步上下文文档，通过 CodeG 的 Skill 管理同步 skills。

## 服务器端 Agent 阵列 (CodeG)

在 CodeG 上配置三个并发 agent，按任务类型分派：

| Agent | 定位 | 适用任务 |
|-------|------|----------|
| **Claude Code** | 主力实现（重推理） | 核心业务逻辑、架构敏感代码、安全敏感代码、需要强推理的实现 |
| **Pi** | 主力实现（重产出） | 标准业务代码、CRUD、服务层、API 端点、脚手架、配置文件 |
| **Google Antigravity** | 前端/交互 | UI 组件、交互逻辑、样式实现 |

Claude 和 Pi 都是主力实现 agent，分界在推理密度：需要架构决策给 Claude，spec 明确照做给 Pi。三个 agent 可并发执行互不依赖的 ticket；有依赖关系的 ticket 按阻塞边顺序执行。

### CodeG 的并发机制

CodeG 自带完整的任务管理和并发控制，你不需要重复造轮子：

- **To-dos 面板**：任务队列，每个任务在独立 git worktree 中运行
- **并发限制**：per folder 控制（默认 2，可调）
- **Process automatically**：开启后任务自动领取
- **Review 流程**：任务完成后进入 review 列，你决定 Merge / Follow up / Complete / Abandon
- **Merge 由 agent 执行**：agent 在自己的 session 中合并，自动解决冲突，CodeG 用 git 验证
- **`@` 委托**：在一个会话中用 `@agent` 让主 agent 委托子任务给其它 agent，子 agent 并行运行
- **Preflight command**：任务进入 review 时自动运行验证命令

## 主工作流：idea → ship

```
本机 (VS Code)                          服务器 (CodeG)
─────────────────                       ─────────────────
/setup-workflow (首次)
    │
/grill-me  (打磨想法)
    │
    ├── /research (如需调研)
    │
/to-spec   (产出 spec)
    │
/to-tickets (拆成 ticket 文件夹)
    │
    └── git push ──────────────────►  git pull
                                          │
                                     /dispatch (在 To-dos 面板创建任务)
                                          │
                                    ┌─────┴────┬─────────────┐
                                   Claude    PI    Antigravity
                                    (各自独立 worktree 并发执行)
                                          │
                                     CodeG Review (逐个 review diff)
                                          │
                                     CodeG Merge (agent 执行合并+冲突解决)
                                          │
                                     /verify (全量验证)
                                          │
    ┌── git pull ◄──────────────────────┘
    │
/code-review (本机最终复审)
```

## Skills 目录结构

```
skills/
  ask/             # 根技能：场景路由器
  planning/        # 本机规划阶段（用户调用）
    grill-me/
    research/
    to-spec/
    to-tickets/
  dispatch/         # 服务器分派阶段（用户调用）
    dispatch/       # 在 CodeG To-dos 面板创建任务
    route-agent/    # 自动判断 ticket → 哪个 agent
    integrate/      # 跨 ticket 协调（补充用，CodeG 自带 Merge）
    verify/         # 全量验证
  engineering/      # 工程实践（模型调用）
    tdd/
    diagnosing-bugs/
    code-review/
    context-sync/   # 本机↔CodeG 上下文同步
  setup/            # 初始化
    setup-workflow/
```

每个 skill 文件夹内含一个 `SKILL.md`，使用 YAML frontmatter 声明 `name`、`description`、是否 `disable-model-invocation`。

## 安装

### 本机 (VS Code)

```bash
# 将 skills/ 目录放入项目根目录，或 symlink 到 ~/.claude/skills/
```

### 服务器 (CodeG)

CodeG 使用共享 skill 存储（`~/.codeg/skills/`），通过 skill-and-agent 矩阵启用：

```bash
# 1. 把本仓库的 skills/ 下所有 skill 文件夹复制到服务器 ~/.codeg/skills/
# 2. 在 CodeG Settings → Skill Packs → Custom 矩阵中启用给对应 agent
#    - engineering 类 skill 给所有三个 agent
#    - planning 和 dispatch 类 skill 只给 Claude Code
```

详见 `/setup-workflow` skill。

运行 `/setup-workflow` 进行首次配置。不知道从哪开始？运行 `/ask`，它会根据你的当前场景告诉你该用哪个 skill。

## 约定

- `.workflow/` 目录是两端共享的上下文载体，必须提交到 git
- ticket 文件是 agent 的工作单元，必须自包含
- 术语表 `CONTEXT.md` 让两端命名一致
- CodeG 的 Merge 流程处理单任务级别的冲突解决和 git 验证，`/integrate` 只在跨 ticket 协调时补充使用
- 不使用 em-dash