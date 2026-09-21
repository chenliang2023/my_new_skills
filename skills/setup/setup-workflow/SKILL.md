---
name: setup-workflow
description: 首次在本项目上启用本工作流。只创建 .workflow/ 下的目录与最小 config.json（路径类字段）。Agent 和 runtime 配置分别由 setup-agents、setup-runtimes 管理。
disable-model-invocation: true
---

# Setup Workflow

首次在某个项目上启用本工作流时运行。**本 skill 只做一件事：创建 `.workflow/` 目录结构与最小路径配置。**

它不决定任务在哪个环境运行，也不安装 agent，不配置 CodeG 或其它任务系统。

## 采集的配置

向用户逐一询问以下问题，将答案写入 `.workflow/config.json`：

1. **文档目录**：ticket 文件存放的目录。默认 `.workflow/tickets/`。
2. **研究文档目录**：research 产出存放的目录。默认 `.workflow/research/`。
3. **spec 文档目录**：to-spec 产出存放的目录。默认 `.workflow/specs/`。
4. **交接文档目录**：跨 runtime 交接文件存放的目录。默认 `.workflow/handoffs/`。
5. **版本目录**：版本管理根目录。默认 `.workflow/version/`。
6. **bug 目录**：bug 修复根目录。默认 `.workflow/bugs/`。
7. **归档目录**：历史版本归档。默认 `.workflow/version/archive/`。
8. **Git 分支策略**：ticket 开发的分支命名模板。默认 `ticket/<id>-<slug>`。

## 不采集的配置

以下配置由专门的注册表管理：

- runtime、adapter、能力和阶段默认路由：运行 `/setup-runtimes`
- agent 的 ID、显示名、标签和成本档：运行 `/setup-agents`
- 任务系统的 UI 设置、登录认证和本机服务：由对应 runtime adapter 自己配置

不要因为当前使用了 CodeG，就把 CodeG 路径或任务设置写进 `config.json`。同一个项目可以从多个 runtime 读取这份配置。

## 生成目录结构

```
.workflow/
  config.json          # 路径类最小配置
  agents.json          # 可选，由 /setup-agents 创建并提交
  runtimes.json        # 可选，由 /setup-runtimes 创建并提交
  tickets/             # 当前批次的 feature ticket
  research/            # research 产出的调研文档
  specs/               # 当前批次的 spec 文件
  handoffs/            # 跨 runtime 交接文件

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
```

`.workflow/` 是共享上下文载体，必须提交到 git。目录里的文档不能假设某一个 runtime 一定在线。

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

字段都是路径模板，没有 runtime、agent 或服务端配置。

## 检查清单

- [ ] `.workflow/` 目录已创建
- [ ] `config.json` 已写入路径类字段
- [ ] 需要 agent 时运行 `/setup-agents`
- [ ] 需要本机、CodeG 或其它执行面时运行 `/setup-runtimes`
- [ ] `.workflow/` 已提交到 git

## 后续

- 想改路径模板：修改 `config.json`，再处理存量文档路径。
- 想加 agent：运行 `/setup-agents`。
- 想加运行时或改变默认路由：运行 `/setup-runtimes`。
- 想同步上下文和 skills：运行 `/context-sync`。
