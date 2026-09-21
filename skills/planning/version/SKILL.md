---
name: version
description: 管理项目的版本号、CHANGELOG、release 流程。决定何时 bump major/minor/patch、如何生成 changelog、如何打 git tag。可在 /verify 之后、/code-review 之前或之后手动调用。
disable-model-invocation: true
---

# Version

把本机和服务器上多个 agent 产生的变更归一到版本管理上：每次 release 都有明确的版本号、自洽的 CHANGELOG、可回溯的 git tag。**前提是项目已经经过 `/verify` 全量验证**。

## 何时调用

- 一批 ticket 都合并完、想切一个版本出来（feature release）
- 修了 critical bug，要紧急 patch release
- 项目首次发版（0.1.0）
- 想看一下当前版本距离上次 release 累积了多少 breaking changes

不在主流程上每次跑，只在 release 节点跑。CI/CD pipeline 里通常也会自动跑，本 skill 是手动调用的口子。在哪台机器上跑都行，先确认工作树都已合并且 `.worktrees/` 里没有未合并的分支。

## 产物落点

跑完本 skill，文件落在这些位置：

```
项目根/
├── CHANGELOG.md                                 # 由 release 工具生成
├── package.json / pyproject.toml / ...          # version 字段已 bump
└── .git/refs/tags/v<version>                    # annotated tag

.workflow/
├── specs/                                       # 当前批次的 spec（归档后会被搬走）
├── tickets/                                     # 当前批次的 feature ticket（同上）
└── version/
    ├── tickets/v<version>.md                    # release ticket
    ├── specs/release-notes-draft.md             # release notes 草稿
    ├── decisions/<date>-bump.md                 # bump 决策记录
    ├── tags/v<version>.md                       # tag 信息镜像
    ├── archive/
    │   └── v<version>/                          # 归档目录
    │       ├── README.md                        # 该版本总览
    │       ├── specs/                           # 该版本的 spec 文件
    │       └── tickets/                         # 该版本的 feature ticket
    └── history.md                               # 版本流水台账（按版本号追加）
```

## 设计原则

- **SemVer 是底线**：major.minor.patch（v2.1.0）。不用日期版本号、不用 0.x 当永久状态
- **Conventional Commits 是 commit message 的约束**，由 agent 在 commit 时遵守，本 skill 只消费它的输出
- **CHANGELOG 由 commit 历史生成**，不手写。手写一定漏、一定错位
- **一次 release = 一个 tag**，tag 指向 merge commit。tag message 用 release notes 摘要
- **pre-release 用语义后缀**：alpha / beta / rc（v2.0.0-rc.1）
- **不把"开发版本"打 tag**：develop 分支的 commit 不要 tag，只在 main / release 分支切版本

## bump 规则

### patch（+0.0.X）

- 任何 `fix:` commit
- 任何 `docs:`、`chore:`、`refactor:` 中**不改变行为的**提交
- 内部实现调整、性能优化、测试补全

### minor（+0.X.0）

- 任何 `feat:` commit
- 新 API、新配置项、新可选依赖
- **必须兼容旧版本**（不删旧字段、不改默认值）

### major（+X.0.0）

- 任何带 `BREAKING CHANGE:` footer 的 commit
- 删字段、改默认值、改接口签名、改配置项语义
- 同一批 commit 同时有 feat + BREAKING → **major**，不是 minor

### pre-release

- alpha：早期内部测试，不稳定，可能丢数据。tag：`v2.0.0-alpha.1`
- beta：功能冻结，只修 bug。tag：`v2.0.0-beta.1`
- rc：候选发布，与正式版一致。tag：`v2.0.0-rc.1`

升正式版 = 去掉后缀重新 tag（`v2.0.0`），不改 commit history。

## 工具选型

按"项目已有 → 直接用 / 项目没有 → 装一个轻量工具"原则：

| 项目类型 | 推荐 | 备注 |
|----------|------|------|
| Node.js / TS | changesets | 多包仓库首选；monorepo 友好 |
| 单包 JS / Python / Go | release-please / standard-version | 单 git tag 友好 |
| Java / JVM | jreleaser | 多语言打包发布 |
| 任意项目想快速起步 | standard-version | 零配置，按 commit 生成 |

**不要每种工具都试一遍**。已经在用哪个就用哪个；想换只换一次，不要中途换。

## 工作流

### 0. 确认没有未合并的工作树

每个 ticket 在自己的 worktree 里完成，发版前必须先把它们收干净：

```bash
git worktree list
```

对每个列出的工作树确认三件事：

- 改动已 commit
- 分支已合并回主分支
- 工作树和分支可以删除

还有未合并的分支时停止发版，先回去合并。带着未合并的工作树打 tag，tag 指向的 commit 里不会有那部分代码，但 ticket 状态已经是 `done`，后面很难查。

合并完成后清理：

```bash
git worktree remove <path>
git branch -d <branch>
```

### 1. 扫描 commit

```bash
# 自上次 tag 以来的所有 commit
git log $(git describe --tags --abbrev=0)..HEAD --oneline
```

### 2. 决定 bump 级别

按上面规则扫 commit type：

- 出现 `BREAKING CHANGE:` → major
- 出现 `feat:` → minor
- 否则 → patch
- 同一批里同时有 feat + fix → minor + patch 累积，**版本只 bump 一次**（按最高级别走）

### 3. 生成 CHANGELOG

工具会自动生成（release-please / changesets / standard-version）。如果你要手写，按下面骨架：

```markdown
# Changelog

## [2.1.0] - 2026-09-18

### Features
- 新增 xxx（[#123]）
- 支持 xxx 配置（[#125]）

### Bug Fixes
- 修复 xxx 在 yyy 场景下的 zzz（[#127]）

### BREAKING CHANGES
- xxx 字段移除，请改用 yyy
```

每条 commit 一行，不要凑整段话。**链接到 PR / commit hash**，方便追溯。

### 4. 更新版本号

- `package.json` / `pyproject.toml` / `Cargo.toml` / `pom.xml` 的 version 字段
- 任何 constants / config 里硬编码的版本号
- API 文档里的版本示例
- OpenAPI / Swagger 的 `info.version`

**不要漏掉硬编码**。漏一处，运行时用户就会拿到旧版本号。

### 5. 打 tag

```bash
git tag -a v2.1.0 -m "Release 2.1.0

- feat: 新增 xxx
- fix: 修复 yyy
"
git push origin v2.1.0
```

annotated tag，message 里塞 release notes 摘要。**不要轻量 tag**（`-a` 必带），否则 git 里看不到 tagger 信息和签名。

### 6. 跑一遍 `/verify`

tag 完不要立刻走，跑一遍 verify 确认版本号相关的：

- build 出来的产物 version 字段是对的
- 启动后 `GET /version` 或类似端点返回新版本号
- 文档站点的版本选择器对得上

### 7. 归档 + 追加历史台账

发版完成后，**把当前批次的 specs 和 tickets 归档**，并把这次 release 写进 `.workflow/version/history.md`。

#### 7.1 归档当前批次

```bash
VERSION="v2.1.0"   # 带 v 前缀的版本号（与 git tag 一致）
ARCHIVE=".workflow/version/archive/${VERSION#v}"

mkdir -p "$ARCHIVE/specs" "$ARCHIVE/tickets"

# 把当前批次的 spec 搬进去
mv .workflow/specs/*.md "$ARCHIVE/specs/"

# 把所有已 done 的 feature ticket 搬进去（未 done 的不动）
for f in .workflow/tickets/*.md; do
  if grep -q 'status: done' "$f"; then
    mv "$f" "$ARCHIVE/tickets/"
  fi
done
```

归档结果：

```
.workflow/version/archive/v2.1.0/
├── specs/                  # 该版本发布涉及的所有 spec 文件
├── tickets/                # 该版本涉及的所有 feature ticket
├── bug-tickets.md          # 该版本涉及的所有 bug fix ticket 索引（指针清单）
├── bug-repros.md           # 该版本涉及的所有复现命令索引（指针清单）
└── bug-postmortems.md      # 该版本涉及的所有 postmortem 索引（指针清单）
```

归档原则：

- spec：当前批次在 `.workflow/specs/` 下的全部文件都搬（spec 是一次性的，发布完就过时）
- feature ticket：只搬 `status: done` 的，未 done 的（如被 block、还在 dispatch 中）**不搬**，留到下个版本
- bug fix ticket：**索引式归档，不实体搬迁**，，见 7.4

#### 7.2 找出本 release 涉及的 bug fix ticket

bug fix ticket 的"是否属于本 release"靠 commit 时间范围判断：

```bash
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -z "$LAST_TAG" ]; then
  RANGE="HEAD"
else
  RANGE="$LAST_TAG..HEAD"
fi

# 找出本 release commit 范围内触及 .workflow/bugs/ 的 commit
git log "$RANGE" --pretty=format:"%H" -- .workflow/bugs/ | sort -u > /tmp/release-commits.txt

# 从 commit 信息里抓 ticket id（约定 bug fix ticket id 形如 042-fix-xxx）
BUG_TICKETS=()
for hash in $(cat /tmp/release-commits.txt); do
  msg=$(git log -1 --pretty=format:"%s" "$hash")
  for id in $(echo "$msg" | grep -oE '\[[0-9]{3}\]' | tr -d '[]'); do
    [ -f ".workflow/bugs/tickets/${id}-fix-"*.md ] && BUG_TICKETS+=("$id")
  fi
done
printf '%s\n' "${BUG_TICKETS[@]}" | sort -u > /tmp/this-release-bug-tickets.txt
```

或者更直接：**让 agent 在跑 `/version` 时询问用户**"本次 release 包含哪些 bug fix"，把用户给定的 ticket id 列表作为输入。自动化脚本只是兜底。

#### 7.3 给 bug ticket 打 released 标签

对每个识别出的 bug fix ticket，在文件顶部的状态行下方加一行：

```markdown
<!-- status: done -->
<!-- released: v2.1.0 -->
```

不要改 status，不要删内容，只在 frontmatter 区加一行。这样从 `.workflow/bugs/tickets/` 全量查的时候，每条 ticket 自报家门。

#### 7.4 写 bug-tickets 索引文件

在归档目录里生成 `bug-tickets.md`（不复制 ticket 实体）：

```markdown
# v2.1.0 包含的 bug fix

本 release 涉及到的 bug fix ticket 清单。实体仍在 `.workflow/bugs/tickets/`。

## 清单

- [042] search-flake（[.workflow/bugs/tickets/042-fix-search-flake.md](../../bugs/tickets/042-fix-search-flake.md)）
  - 复现：[.workflow/bugs/repros/2026-08-12-search-flake.md](../../bugs/repros/2026-08-12-search-flake.md)
  - postmortem：[.workflow/bugs/postmortems/2026-08-15-search-flake.md](../../bugs/postmortems/2026-08-15-search-flake.md)
- [089] token-expire（[.workflow/bugs/tickets/089-fix-token-expire.md](../../bugs/tickets/089-fix-token-expire.md)）
  - 复现：[.workflow/bugs/repros/2026-08-30-token-expire.md](../../bugs/repros/2026-08-30-token-expire.md)
  - postmortem：[.workflow/bugs/postmortems/2026-09-02-token-expire.md](../../bugs/postmortems/2026-09-02-token-expire.md)
```

**为什么不复制 ticket 实体？**

- bug ticket 是按时间索引的（`.workflow/bugs/tickets/` 是全量历史），不能搬走
- 跨路径"按 release 查"和"按时间全量查"都能用，是两条独立检索路径
- 索引文件足够小，Git 友好，跨平台无差异
- 比起 feature ticket 的 `mv` 是有意的差别：feature ticket 发完版就没用了（下次开发是新的），bug ticket 全程保留（统计趋势、追责需要）

#### 7.5 写 bug-repros 索引文件

复现命令与 bug ticket 是同一个 bug 的两个切片，bug fix ticket 索引里已经引用了 repros，但 repros 单独看也有价值（"v2.1.0 集中修了哪些复现命令"），所以也起一个索引：

```markdown
# v2.1.0 包含的 bug 复现命令

本 release 涉及到的复现命令清单。实体仍在 `.workflow/bugs/repros/`。

## 清单

- [042] search-flake 复现（[.workflow/bugs/repros/2026-08-12-search-flake.md](../../bugs/repros/2026-08-12-search-flake.md)）
  - 对应 ticket：[.workflow/bugs/tickets/042-fix-search-flake.md](../../bugs/tickets/042-fix-search-flake.md)
- [089] token-expire 复现（[.workflow/bugs/repros/2026-08-30-token-expire.md](../../bugs/repros/2026-08-30-token-expire.md)）
  - 对应 ticket：[.workflow/bugs/tickets/089-fix-token-expire.md](../../bugs/tickets/089-fix-token-expire.md)
```

#### 7.6 写 bug-postmortems 索引文件

postmortem 是 bug 修复的根因分析，是追溯事故最有价值的资产，必须独立索引：

```markdown
# v2.1.0 包含的 bug postmortem

本 release 涉及到的 postmortem 清单。实体仍在 `.workflow/bugs/postmortems/`。

## 清单

- [042] search-flake postmortem（[.workflow/bugs/postmortems/2026-08-15-search-flake.md](../../bugs/postmortems/2026-08-15-search-flake.md)）
  - 对应 ticket：[.workflow/bugs/tickets/042-fix-search-flake.md](../../bugs/tickets/042-fix-search-flake.md)
- [089] token-expire postmortem（[.workflow/bugs/postmortems/2026-09-02-token-expire.md](../../bugs/postmortems/2026-09-02-token-expire.md)）
  - 对应 ticket：[.workflow/bugs/tickets/089-fix-token-expire.md](../../bugs/tickets/089-fix-token-expire.md)
```

#### 7.7 写归档目录里的版本说明

在归档目录下加一个 `README.md`，作为该版本的总览入口：

```markdown
# v2.1.0 归档（2026-09-18）

## 包含内容

- 3 个 spec（见 `specs/`）
- 12 个 feature ticket（见 `tickets/`）
- 2 个 bug fix（见 `bug-tickets.md`，含 repros / postmortems 指针）

## 索引

- [bug-tickets](./bug-tickets.md) ， bug fix ticket 清单
- [bug-repros](./bug-repros.md) ， 复现命令清单
- [bug-postmortems](./bug-postmortems.md) ， postmortem 清单

## git tag

- v2.1.0 → commit abc1234

## CHANGELOG 摘要

- 新增 xxx
- 修复 yyy
```

#### 8. 追加版本流水台账

在 `.workflow/version/history.md` **末尾**追加一段（**不覆盖既有内容**）：

```markdown
## v2.1.0 ， 2026-09-18

- **类型**：minor
- **变更**：feat 3 / fix 7 / chore 12 / BREAKING 0
- **归档**：[archive/v2.1.0/](./archive/v2.1.0/)
- **git tag**：v2.1.0 → abc1234
- **CHANGELOG**：见 ./CHANGELOG.md
```

每次发版都追加一段，不删不改历史。`history.md` 是 release 历史的唯一真相源，，倒着追加，避免并发发版时的行号竞态。

### 9. 通知

release 完成后告知用户：

- 🏷️ 已发版 v2.1.0
- 📝 CHANGELOG 已生成（路径）
- 🏷️ git tag 已推送
- 📦 当前批次已归档到 `.workflow/version/archive/v2.1.0/`
- 📜 版本台账已追加（`.workflow/version/history.md`）
- ⏭️ 下一步：CI/CD 自动发布到 npm / PyPI / Docker Hub / Maven Central，或手动推包

## 与其它 skill 的衔接

- **上游**：`/verify` 通过 → 本 skill
- **下游**：`/code-review` 可以 review release commit 本身（CHANGELOG 措辞、版本号是否漏改）
- **commit message 约束**：在 `engineering/tdd` 和 `engineering/code-review` 里要求 agent 写 `feat:` / `fix:` / `chore:` / `docs:` / `refactor:` / `test:` 前缀
- **monorepo 拆分版本**：单包项目不需要考虑；monorepo 项目用 changesets，每个子包独立 version，本 skill 加一条"按子包逐个切版本"

## 反模式

- 🚫 **每次 merge 都 bump**：版本是 release 节点的事，不是 commit 节点的事
- 🚫 **major 升级了但 CHANGELOG 没写 BREAKING CHANGES**：合规事故
- 🚫 **手写 CHANGELOG 不引用 commit hash**：失去追溯能力
- 🚫 **tag 指向 squash merge 的中间 commit 而不是 main 分支**：tag 找不到代码
- 🚫 **CI/CD 没跑过就 tag**：版本号与实际产物可能对不上
- 🚫 **0.x 当永久状态**：永远不升 1.0，导致 SemVer 信号失效

## 输出格式

```markdown
## 🏷️ 已发版 v2.1.0

### 📊 本次变更统计
- feat: 3
- fix: 7
- chore: 12
- BREAKING: 0

### 📝 CHANGELOG 已更新
- 路径：./CHANGELOG.md
- 本节：## [2.1.0] - 2026-09-18

### 🏷️ Git tag
- v2.1.0 → commit abc1234
- 已推送到 origin

### ✅ 已 verify
- 类型检查：pass
- 测试：pass
- 构建：pass
- 产物 version：2.1.0 ✓

### 📦 归档
- 当前批次已迁至 `.workflow/version/archive/v2.1.0/`
- spec：3 个
- feature ticket：12 个
- bug fix ticket：2 个（如有）
- `.workflow/version/history.md` 已追加 v2.1.0 段

### ⏭️ 下一步
- 📤 推送到 npm / PyPI / Docker Hub
- 📢 通知相关方
```