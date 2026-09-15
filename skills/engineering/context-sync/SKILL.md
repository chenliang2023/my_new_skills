---
name: context-sync
description: 在本机与服务器 CodeG 之间同步上下文文档和 skills。确保两端读取同一套 spec、ticket、research 和术语表。
---

# Context Sync

本机与服务器之间通过 git 仓库中的 `.workflow/` 目录同步上下文文档，通过 CodeG 的 Skill 管理同步 skills。

## 同步的内容

### 1. `.workflow/` 目录（通过 git）

```
.workflow/
  config.json          # 项目配置（两端共享）
  tickets/             # ticket 文件（本机产出，服务器消费）
  specs/               # spec 文件（本机产出，服务器消费）
  research/            # 调研文档（本机产出，服务器消费）
  handoffs/            # 交接文件（双向）
    grill-summary-*.md
    verify-report-*.md
  CONTEXT.md           # 项目术语表（双向，持续更新）
  ADR/                 # 架构决策记录（双向）
```

### 2. Skills（通过 CodeG 的 Skill 管理）

CodeG 有两种 skill 管理方式：

**方式 A：共享存储（推荐）**
- CodeG 的 Settings → Skill Packs → Custom 标签页
- skill 存放在 `~/.codeg/skills`
- 写一次，通过 skill-and-agent 矩阵启用到多个 agent
- 对应本仓库的 `skills/` 目录：把本仓库的 skill 文件夹复制或 symlink 到 `~/.codeg/skills/`

**方式 B：直接写入 agent 目录**
- CodeG 的 Settings → Skills 页面
- 直接写入某个 agent 的 skills 目录（如 Claude Code 的 `~/.claude/skills/`）
- 适合 agent 专属 skill

**同步策略**：
- 把本仓库 `skills/` 下的所有 skill 放入服务器的 `~/.codeg/skills/`
- 在 CodeG 的 Skill Packs → Custom 矩阵中，把 planning 和 setup 类 skill 只启用给 Claude Code（因为本机 VS Code 端用的也是 Claude Code）
- engineering 类 skill 启用给所有三个 agent（Claude / Pi / AntiGravity）
- dispatch 类 skill 只启用给 Claude Code（由用户在服务器端手动调用）

## 同步规则

### 本机 → 服务器（规划结果交接）

当本机完成规划（grill / spec / tickets）后：

1. 确保 `.workflow/` 下所有文档已保存
2. 如果项目有 `CONTEXT.md`，确认术语已更新
3. git add + commit + push
4. 在服务器 CodeG 中：
   - git pull
   - 在 To-dos 面板中根据 ticket 创建任务（运行 `/dispatch`）

### 服务器 → 本机（执行结果回报）

当服务器端完成 ticket 执行后：

1. CodeG Merge 后代码已在主分支
2. `/verify` 产出验证报告到 `.workflow/handoffs/`
3. git add + commit + push
4. 本机 git pull 后运行 `/code-review`

## CONTEXT.md：项目术语表

`CONTEXT.md` 是项目共享术语表，两端共用。作用：
- 让 agent 理解项目 jargon，减少 token 消耗
- 让命名在多个 agent 之间一致
- 让 spec 和 ticket 中的术语有明确定义

格式：

```markdown
# Context

## 术语表
- **材料化（Materialization）**：给一节课在文件系统中分配位置，使其"真实"的过程
- **课程段（Section）**：课程的章，一组课的容器

## ADR 索引
- [ADR-001](ADR/001-auth-strategy.md)：认证策略选择 JWT
```

## CodeG 的 Skill 格式

CodeG 使用的 SKILL.md 格式与本仓库一致：

```markdown
---
name: skill-id
description: 一句话描述
---

正文：技能的步骤和规则。
```

CodeG 通过 `name` 字段作为 `/` 菜单中的标识，`description` 用于在菜单中显示。

## 检查清单

交接前确认：
- [ ] `.workflow/` 已提交到 git
- [ ] CONTEXT.md 如有变更已更新
- [ ] push 成功
- [ ] 服务器 git pull 成功
- [ ] 服务器 CodeG 的 Skill 管理中 skill 已启用给对应 agent