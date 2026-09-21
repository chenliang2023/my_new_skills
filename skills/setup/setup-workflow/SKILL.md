---
name: setup-workflow
description: 在被管理的项目里首次启用本工作流。只创建该项目 .workflow/ 下的目录与最小 config.json（路径类字段）。Agent 清单由 setup-agents 管理。
disable-model-invocation: true
---

# Setup Workflow

在**被管理的项目**里首次启用本工作流时运行。**本 skill 只做一件事：在该项目创建 `.workflow/` 目录结构与最小路径配置。**

它不登记 agent，也不配置任何任务系统。执行方式在整条工作流里已经固定：每个 ticket 一个 git worktree。

## 在哪里运行

在要启用这套工作流的那个项目里运行，不是在本 skills 仓库里。

skills 仓库本身没有 `.workflow/`。`.workflow/` 属于每个被管理的项目，随项目提交到 git，让本机和服务器读到同一份 spec、ticket 和报告。

## 采集的配置

向用户逐一询问以下问题，将答案写入 `.workflow/config.json`：

1. **文档目录**：ticket 文件存放的目录。默认 `.workflow/tickets/`。
2. **研究文档目录**：research 产出存放的目录。默认 `.workflow/research/`。
3. **spec 文档目录**：to-spec 产出存放的目录。默认 `.workflow/specs/`。
4. **交接文档目录**：跨机器交接文件存放的目录。默认 `.workflow/handoffs/`。
5. **版本目录**：版本管理根目录。默认 `.workflow/version/`。
6. **bug 目录**：bug 修复根目录。默认 `.workflow/bugs/`。
7. **归档目录**：历史版本归档。默认 `.workflow/version/archive/`。
8. **worktree 目录**：每个 ticket 的工作树放在哪里。默认 `.worktrees/`，位于项目根。
9. **Git 分支策略**：ticket 开发的分支命名模板。默认 `ticket/<id>-<slug>`。

## 不采集的配置

只有一件事不在本 skill 范围内：

- agent 清单（harness、模型强弱、速度、所在机器）：运行 `/setup-agents`，它自带基线名单

不要往 `config.json` 里写机器名、服务器地址、任务系统设置或 agent 信息。这些要么属于 agents.json，要么属于那台机器自己的配置。

## 生成目录结构

```
.workflow/
  config.json          # 路径类最小配置
  agents.json          # 可选，由 /setup-agents 创建并提交
  tickets/             # 当前批次的 feature ticket
  research/            # research 产出的调研文档
  specs/               # 当前批次的 spec 文件
  handoffs/            # 跨机器交接文件

  version/
    tickets/
    specs/
    decisions/
    tags/
    archive/
    history.md

  bugs/
    repros/
    postmortems/
    tickets/

.worktrees/            # 每个 ticket 一个工作树，不进 git
```

`.workflow/` 是共享上下文载体，必须提交到 git，这样本机和服务器能读到同一份 spec、ticket 和报告。

`.worktrees/` 是本地工作目录，必须加入 `.gitignore`。工作树里的内容通过 ticket 分支进入 git，不通过目录本身。

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
  "worktree_dir": ".worktrees",
  "branch_template": "ticket/<id>-<slug>"
}
```

字段都是路径模板，没有任何机器、agent 或服务端配置。

## 检查清单

- [ ] `.workflow/` 目录已创建
- [ ] `config.json` 已写入路径类字段
- [ ] `.worktrees/` 已加入 `.gitignore`
- [ ] 需要登记 agent 时运行 `/setup-agents`
- [ ] `.workflow/` 已提交到 git

## 后续

- 想改路径模板：修改 `config.json`，再处理存量文档路径。
- 想改 agent 清单：运行 `/setup-agents`。
- 想同步多台机器的上下文和 skills：运行 `/context-sync`。
- 不确定下一步：运行 `/ask`。
