---
name: setup-workflow
description: 首次配置双环境工作流。设置文档目录、确认 CodeG 服务器与本机的路径映射、配置 CodeG 的 Task settings 和 Skill 启用。每个项目运行一次。
disable-model-invocation: true
---

# Setup Workflow

首次在某个项目上启用本工作流时运行。它收集该项目的配置信息并写入配置文件，后续 skill 读取该配置。

## 采集的配置

向用户逐一询问以下问题，将答案写入 `.workflow/config.json`：

1. **文档目录**：spec 和 ticket 文件存放的目录。默认 `.workflow/tickets/`。
2. **研究文档目录**：research 产出存放的目录。默认 `.workflow/research/`。
3. **服务器仓库路径**：CodeG 服务器上该项目的仓库路径。
4. **本机仓库路径**：本机上该项目的仓库路径。
5. **Agent 路由偏好**：用户对三个 agent（Claude / PI / AntiGravity）的分派偏好。可选填一个默认分派规则表，也可留空由 `/route-agent` 自动判断。
6. **Git 分支策略**：ticket 开发使用的分支命名模板。默认 `ticket/<id>-<slug>`。
7. **CodeG Task settings**：并发限制、preflight command、worktree location 等 CodeG 特定配置。

## 生成目录结构

```
.workflow/
  config.json          # 上述配置
  tickets/             # to-tickets 产出的 ticket 文件
  research/            # research 产出的调研文档
  specs/               # to-spec 产出的 spec 文件
  handoffs/            # 交接文件（本机→服务器、服务器→本机）
```

## config.json 模板

```json
{
  "docs_dir": ".workflow/tickets",
  "research_dir": ".workflow/research",
  "specs_dir": ".workflow/specs",
  "handoffs_dir": ".workflow/handoffs",
  "server_repo_path": "",
  "local_repo_path": "",
  "branch_template": "ticket/<id>-<slug>",
  "agent_routing": {
    "claude": ["complex-logic", "architecture", "core-domain", "security"],
    "pi": ["crud", "service-layer", "api-endpoint", "scaffold", "config", "migration"],
    "antigravity": ["ui", "interaction", "styling"]
  },
  "codeg": {
    "max_concurrent_tasks": 3,
    "process_automatically": true,
    "preflight_command": "npm test",
    "worktree_location": "",
    "merge_strategy": "squash",
    "delete_worktree_after_merge": true
  }
}
```

## CodeG 端配置步骤

除了写入 config.json，还需要在 CodeG 中做以下配置：

### 1. 打开项目文件夹

在 CodeG 中打开项目文件夹（project root，不是 worktree）。CodeG 需要 project root 才能创建 task worktree。

### 2. Task settings

在 To-dos 面板的 Task settings 中：

- **General tab**：
  - Default agent：Claude Code（主力 agent，可按 ticket 覆盖）
  - Process automatically：开启，让任务自动领取
  - Max concurrent tasks：3（或按需调整）

- **Merge tab**：
  - Default merge strategy：Squash（每个 ticket 合并为一个 commit）
  - Merge automatically：关闭（你想逐个 review）
  - Delete the worktree after merging：开启

- **Worktree tab**：
  - Worktree location：留空（默认在项目旁边）或指定集中存放位置
  - Worktree init command：如 `pnpm install`
  - Preflight command：如 `npm test`（任务进入 review 时自动运行）

- **Prompts tab**（可选）：
  - All stages：`遵循 AGENTS.md / CLAUDE.md 中的约定`
  - Task run：`参考 .workflow/tickets/ 下的对应 ticket 文件`
  - Merge：`用 Conventional Commits 格式写 commit message`

### 3. Skills 启用

在 Settings → Skill Packs → Custom 中：

1. 把本仓库 `skills/` 下的 skill 文件夹复制到服务器的 `~/.codeg/skills/`
2. 在 skill-and-agent 矩阵中启用：

| Skill | Claude Code | PI | AntiGravity |
|-------|-------------|-----|-------------|
| tdd | ✓ | ✓ | ✓ |
| diagnosing-bugs | ✓ | ✓ | ✓ |
| code-review | ✓ | ✓ | ✓ |
| context-sync | ✓ | ✓ | ✓ |
| grill-me | ✓ | - | - |
| research | ✓ | - | - |
| to-spec | ✓ | - | - |
| to-tickets | ✓ | - | - |
| dispatch | ✓ | - | - |
| route-agent | ✓ | - | - |
| integrate | ✓ | - | - |
| verify | ✓ | - | - |
| setup-workflow | ✓ | - | - |

原则：
- engineering 类 skill 给所有 agent（它们需要遵守工程规范）
- planning 和 dispatch 类 skill 只给 Claude Code（用户手动调用的编排 skill）

### 4. Agent 安装确认

在 CodeG 中确认三个 agent 已安装并认证：
- Claude Code
- Pi
- Google Antigravity

## 检查清单

- [ ] `.workflow/` 目录已创建
- [ ] `config.json` 已写入且字段完整
- [ ] 用户确认了 agent 路由偏好（或选择留空）
- [ ] CodeG 中项目文件夹已打开
- [ ] CodeG 的 Task settings 已配置
- [ ] CodeG 的 Skills 已启用给对应 agent
- [ ] 三个 agent 已安装并认证
- [ ] 告知用户下一步运行 `/grill-me` 开始规划