---
name: setup-workflow
description: 首次在本项目上启用本工作流。只需要创建 .workflow/ 下的目录与最小 config.json（路径类字段）。服务端相关配置（CodeG Task settings、Skill 矩阵、agent 安装）由用户自行在 CodeG 里配，不归本 skill 管。每个项目运行一次。
disable-model-invocation: true
---

# Setup Workflow

首次在某个项目上启用本工作流时运行。**本 skill 只做一件事：创建 `.workflow/` 目录结构与最小 config.json。**

服务端相关配置（CodeG To-dos Task settings、Skill Packs 矩阵、agent 安装与认证、preflight command 等）由用户在 CodeG UI 里自行配置，**不在本 skill 范围内**。

## 采集的配置

向用户逐一询问以下问题，将答案写入 `.workflow/config.json`：

1. **文档目录**：ticket 文件存放的目录。默认 `.workflow/tickets/`。
2. **研究文档目录**：research 产出存放的目录。默认 `.workflow/research/`。
3. **spec 文档目录**：to-spec 产出存放的目录。默认 `.workflow/specs/`。
4. **交接文档目录**：本机↔服务器交接文件存放的目录。默认 `.workflow/handoffs/`。
5. **版本目录**：版本管理根目录。默认 `.workflow/version/`。
6. **bug 目录**：bug 修复根目录。默认 `.workflow/bugs/`。
7. **归档目录**：历史版本归档（每个版本号一个子目录）。默认 `.workflow/version/archive/`。
8. **Git 分支策略**：ticket 开发的分支命名模板。默认 `ticket/<id>-<slug>`。

**不采集**（详见"不在本 skill 范围内"段）：

- 服务器仓库路径、本机仓库路径——用户自己心里有数，写进 `~/.bashrc` / `~/.zshrc` 的环境变量更合适
- agent 路由偏好——`/route-agent` 内嵌规则，不读 config.json
- CodeG Task settings / Skill 矩阵 / agent 安装——属于服务端 UI 配置

## 生成目录结构

```
.workflow/
  config.json          # 路径类最小配置
  tickets/             # 当前批次的 feature ticket
  research/            # research 产出的调研文档
  specs/               # 当前批次的 spec 文件
  handoffs/            # 交接文件（本机↔服务器）

  version/             # 版本管理根目录
    tickets/           #   当前 release 的 release ticket
    specs/             #   当前 release 的 release notes 草稿、bump 计划
    decisions/         #   当前 release 的 bump 决策记录
    tags/              #   当前 release 的 tag 镜像
    archive/           #   历史版本归档（每个版本号一个子目录）
      v1.0.0/
        specs/         #     该版本发布涉及的 spec 文件
        tickets/       #     该版本涉及的所有 feature ticket
        bug-tickets.md     #     该版本涉及的所有 bug fix ticket 索引（指针清单）
        bug-repros.md     #     该版本涉及的所有复现命令索引（指针清单）
        bug-postmortems.md #     该版本涉及的所有 postmortem 索引（指针清单）
      v1.1.0/
        ...
    history.md               #   版本流水台账（按版本号顺序追加）

  bugs/                # bug 修复根目录
    repros/            #   diagnosing-bugs 阶段 1：复现命令
    postmortems/       #   阶段 5：根因 + 防同类再次发生
    tickets/           #   bug fix ticket（永久留存，按时间索引；按 release 范围做索引式归档，不实体搬迁）
```

## config.json 模板

```json
{
  "tickets_dir": ".workflow/tickets",
  "research_dir": ".workflow/research",
  "specs_dir": ".workflow/specs",
  "handoffs_dir": ".workflow/handoffs",
  "version_dir": ".workflow/version",
  "bugs_dir": ".workflow/bugs",
  "archive_dir": ".workflow/version/archive",
  "branch_template": "ticket/<id>-<slug>"
}
```

字段都是路径模板，没有服务端配置。如果用户后续要加字段，按"路径类 vs 服务端 UI"区分后再决定是否进 config.json。

## 不在本 skill 范围内

下面这些由用户在 CodeG 端自己配，不归本 skill 管：

- **CodeG 项目文件夹**：在 CodeG 中打开（不是 worktree）。
- **CodeG Task settings**：Default agent、Max concurrent tasks、Process automatically、preflight command 等。
- **CodeG Skill Packs 矩阵**：哪些 skill 启用给哪个 agent。详见 `engineering/context-sync/SKILL.md` 的同步原则。
- **agent 安装与认证**：`.workflow/agents.json` 里每个 agent 都在 CodeG 里安装并认证。改 agent 表请跑 `/setup-agents`。
- **agent 注册表**：放 `.workflow/agents.json`（不是 config.json）。由 `/setup-agents` 管理，route-agent / dispatch / verify 自动按新表路由。

## 检查清单

- [ ] `.workflow/` 目录已创建（含 version/、bugs/、version/archive/ 子目录）
- [ ] `config.json` 已写入（7 个路径类字段）
- [ ] 告知用户下一步：在 CodeG 端自行配置 Task settings / Skill 矩阵 / agent 安装，然后运行 `/grill-me` 开始规划

## 后续

- 想加字段 → 先想清楚"是路径类配置还是服务端 UI 配置"。前者加进 config.json；后者别加，到 CodeG 那边配
- 想换路径模板 → 改 config.json + 跑一次迁移脚本（自行处理存量 ticket 路径）
- 想清掉 config.json → 不推荐，会让所有依赖路径的 skill 退回默认值（详见各 skill 的"前提"段）