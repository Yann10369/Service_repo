# Skill: 自动上传代码并申请 PR（按模块归属）

> 当智能体（Claude Code / Cursor / 其他 Agent）读到本文档时，应能按照下面的步骤，**自动完成本地代码提交并通过 `gh` CLI 创建 Pull Request**。
>
> 本仓库采用**模块化分工开发**，仓库内每个模块由不同成员负责：
>
> | 模块          | 路径前缀      | 典型负责人        |
> |---------------|---------------|-------------------|
> | 前端          | `frontend/`   | frontend-lead     |
> | 后端          | `backend/`    | backend-lead      |
> | 数据 / 模型   | `model/`      | model-lead        |
> | 文档          | `docs/`       | docs-lead         |
> | 仓库根级      | `根目录`       | repo-maintainer   |
>
> **智能体必须先确认用户负责的模块，再只对该模块目录下的改动执行提交 / 推送 / PR 流程。**

---

## 1. 触发条件

满足以下任意一条时触发本技能：

- 用户明确说："提交我的前端代码"、"上传后端"、"push 我的模块"、"为 backend 提 PR"。
- 用户给出修改但未指明提交方式，并使用 `/pr`、 `/push` 等指令，且**上下文能推断出模块**。
- 用户的修改已经完成代码层面验证（lint / test 通过），需要合入远程分支。

---

## 2. 前置条件

在执行前，智能体应**先确认**以下条件都已满足，否则先停下来询问用户：

| 条件                                          | 检查方式                                    |
|-----------------------------------------------|---------------------------------------------|
| 当前目录是 git 仓库                           | `git rev-parse --is-inside-work-tree`       |
| 已配置 `user.name` 与 `user.email`            | `git config user.name` / `user.email`        |
| 远端仓库已配置且可访问                        | `git remote -v`                             |
| 已登录 `gh` CLI（如需创建 PR）                | `gh auth status`                            |
| 工作树没有未保存的机密（密钥、token 等）       | 智能体自查，不应提交 `.env`、私钥等         |
| **用户身份与目标模块归属匹配**                | 见下文 §2.1                                 |

任何一项缺失：先**询问用户**而不是猜测。

### 2.1 模块归属校验（必须）

智能体**必须**通过以下顺序确认本次提交归属：

1. **用户显式声明**：用户消息中提到具体模块（如"提交我的 frontend 代码"），直接采纳。
2. **本地配置文件**：读取 `.git/config` 的 `[user]` 段，或仓库根 `CODEOWNERS` / `OWNERS` 文件（如果存在），匹配成员与模块的映射。
3. **环境变量**：检查环境变量 `AGENT_OWNED_MODULES`（支持逗号分隔，如 `frontend,docs`）。
4. **询问用户**：以上都没有时，**停下来**向用户提问：

   > "本次改动属于哪个模块？当前模块归属：frontend / backend / model / docs / root（仓库根级）。"

绝对不要在未确认归属的情况下直接 `git add .` 或 `git commit`。

### 2.2 模块白名单

确认归属后，本次提交允许的文件路径必须**全部**落在该模块目录下：

```
frontend 模块  →  frontend/**
backend 模块   →  backend/**
model 模块     →  model/**
docs 模块      →  docs/**
root 模块      →  仅允许修改仓库根级的：README.md, .gitignore, LICENSE, CODEOWNERS, OWNERS
```

不属于白名单的文件：
- **跳过**，不 add、不 commit。
- **停下来**告知用户存在越界文件，列出文件路径，请用户确认如何处理（拆分提交 / 移交他人 / 撤回改动）。

---

## 3. 执行步骤

按顺序执行，每一步失败都应停下来回报，而不是继续往下跑：

### Step 1. 查看当前状态

```bash
git status
git branch --show-current
git log --oneline -5
```

向用户简要汇报：当前在哪个分支、有哪些改动、最近一次提交是什么。

### Step 2. 确认改动范围 & 模块归属

```bash
git diff --stat
git diff
```

1. **确认模块归属**：按 §2.1 的顺序确定本会话的目标模块（如 `frontend`）。如果当前会话**还没有确定模块归属**，**立刻停下来询问用户**。
2. **过滤改动文件**：把 `git status` 列出的文件按 §2.2 白名单分类：

   ```bash
   # 列出"用户模块内"的改动文件
   git status --porcelain | awk '{print $2}' | grep -E '^<module>/'
   ```

3. **越界检查**：白名单外的文件**绝对不能**进入本次提交。向用户回报：

   > "检测到以下文件不属于您负责的 `<module>` 模块，已跳过本次提交：
   > - <file1>
   > - <file2>
   > 请联系对应模块负责人或显式授权我代为提交。"

### Step 3. 按模块暂存 & 提交

```bash
# 只 add 本模块的文件（明确路径，不使用 git add . / -A）
git add <module>/<file1> <module>/<file2> ...

git status   # 再确认一次 staged 的全部位于 <module>/ 下

git commit -m "<type>(<module>): <subject>

<body 说明本次改动的目的、影响范围、关联 issue>

模块: <module>
负责人: <user-name>

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

提交信息规范（Conventional Commits + 模块前缀）：

- `feat(frontend):` 新功能
- `feat(backend):` 新功能
- `fix(frontend):` 修复 bug
- `docs(docs):` 仅文档改动
- `refactor(model):` 重构（不改变行为）
- `test(backend):` 测试相关
- `chore(root):` 构建、工具、依赖等杂项（仅 root 模块使用）

> ⚠️ **强约束**：执行 `git add` 时，路径前缀必须与本次会话归属模块完全匹配。  
> 例：归属为 `frontend` 时，`git add backend/foo.py` **必须被拒绝**。

### Step 4. 推送到远端

```bash
git push origin <current-branch>
```

如果推送失败：

- **非快进拒绝 (non-fast-forward)**：先 `git fetch`，再与用户确认是 `rebase` 还是 `merge`。
- **鉴权失败**：提示用户配置凭证或执行 `gh auth login`。
- **分支受保护**：提示用户走 PR 流程。

### Step 5. 创建 Pull Request

仅在推送到非默认分支（或用户明确要求）时执行。

PR 标题必须带模块前缀（如 `feat(frontend): 新增登录页面`），并设置 reviewer 为该模块的负责人：

```bash
gh pr create \
  --base main \
  --head <current-branch> \
  --title "<type>(<module>): <short title>" \
  --body "<PR body>" \
  --reviewer <module-owner-login>
```

PR body 模板（中文，已包含模块信息）：

```markdown
## 改动模块
- 模块: `<module>`
- 负责人: <user-name>

## 改动说明
- <本次主要改动点 1>
- <本次主要改动点 2>

## 改动文件
- <module>/<file1>
- <module>/<file2>

## 改动类型
- [ ] 新功能 (feat)
- [ ] 修复 (fix)
- [ ] 重构 (refactor)
- [ ] 文档 (docs)
- [ ] 其他

## 测试
- [ ] 已通过本模块本地单元测试
- [ ] 已通过本模块本地 lint / type-check
- [ ] 已人工验证关键路径

## 跨模块影响
- [ ] 无
- [ ] 有（说明：___________）

## 关联 Issue
Closes #<issue-id>  （如有）

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Step 6. 回报结果

向用户输出：

1. 提交哈希 (commit SHA)
2. 远端分支名
3. **模块归属**（再确认一次）
4. PR 链接（如已创建）
5. **被跳过的越界文件**（如有）
6. 任何告警或遗留事项

---

## 4. 常见场景

### 4.1 在默认分支直接提交

如果当前在 `main` / `master`：

- **优先策略**：从默认分支切出 feature 分支再提交与推送，避免污染主干。
- 命令：`git checkout -b <type>/<module>/<short-desc>`  
  示例：`git checkout -b feat/frontend/login-page`

### 4.2 没有改动

```bash
git status --porcelain
```

输出为空时，**不要**创建空提交，直接告知用户"没有需要提交的改动"。

### 4.3 存在未跟踪文件

- 如果是新文件且属于本模块的本次改动：`git add <module>/<file>`。
- 如果是构建产物、依赖、日志等：先检查 `.gitignore`，必要时**追加**进 `.gitignore` 而不是提交。

### 4.4 需要合并上游最新代码

```bash
git fetch origin
git rebase origin/main     # 或 merge，看团队规范
```

rebase 冲突时停下来，**不要** 自动 `--force` 或 `-X theirs`。

### 4.5 改动跨多个模块（重要）

如果 `git status` 显示的改动**同时落在多个模块目录下**：

1. **默认拒绝**：智能体**不能**一次提交多个模块的改动——这意味着 PR 的影响面超出提交人的职责。
2. **拆分提交**：建议用户：

   > "检测到改动跨越多个模块。按模块拆分建议：
   > - frontend 改动 → 请 frontend 负责人执行
   > - backend 改动 → 请 backend 负责人执行
   > 是否要为每个模块各开一个分支、分别提 PR？"

3. **显式授权例外**：用户明确说"我有所有模块权限"或"我是 repo-maintainer"，且能拿出凭证（环境变量 `AGENT_OWNED_MODULES=frontend,backend,model,docs,root` 全集），才允许一次性提交，并在 PR body 的 "跨模块影响" 字段里**逐项说明**。

### 4.6 当前会话还没确定模块

不要做任何 `git add` / `git commit` 操作。直接停下向用户提问：

> "本次改动属于哪个模块？可选：frontend / backend / model / docs / root。"

确定归属后再继续。

### 4.7 用户临时变更模块

会话中途用户说"这次改成提交 backend 的"：

- **重新走 §2.1 校验**：再次确认用户身份与 backend 模块归属是否匹配。
- **撤销之前为 frontend 的暂存**：`git reset HEAD` 清理 staged 区，避免跨模块误混。

---

## 5. 安全与红线

以下行为**禁止**自动执行，必须先得到用户明确授权：

- `git push --force` / `git push -f`
- `git reset --hard`
- `git commit --amend`（已推送过的提交）
- 删除远端分支
- 跳过 PR review（如果仓库有分支保护）
- 提交任何包含密钥、token、密码、个人信息的文件

---

## 6. 失败回报格式

当执行失败时，使用下面的格式回报，方便用户快速定位：

```
❌ Skill: 自动上传代码并申请 PR 失败

阶段: <Step 1 ~ Step 5>
命令: <实际执行的命令>
错误: <关键错误信息>

建议:
- <下一步建议 1>
- <下一步建议 2>
```

---

## 7. 示例：完整一次会话

### 示例 A：模块归属清晰（正常流程）

> **用户**：帮我把 `frontend/` 下新增的登录页面提交并提个 PR。
>
> **智能体**：
> 1. **模块归属识别**：用户提到 `frontend/`，直接锁定归属为 `frontend`。
> 2. 执行 `git status` → 当前分支 `feat/frontend/login-page`，新增 `frontend/src/pages/Login.vue`。
> 3. 执行 `git diff frontend/src/pages/Login.vue` → 确认改动符合用户描述，全部位于 `frontend/` 白名单内。
> 4. `git add frontend/src/pages/Login.vue`
> 5. `git commit -m "feat(frontend): 新增登录页面"`
> 6. `git push origin feat/frontend/login-page`
> 7. `gh pr create --base main --head feat/frontend/login-page --title "feat(frontend): 新增登录页面" --body "..." --reviewer frontend-lead`
> 8. 向用户回报：commit `abc1234`、分支 `feat/frontend/login-page`、PR 链接 `https://github.com/.../pull/12`、reviewer `frontend-lead`。

### 示例 B：模块未声明（智能体主动询问）

> **用户**：把改动提交一下。
>
> **智能体**：
> 1. 执行 `git status` → 显示有改动，但用户没说模块。
> 2. **停下，向用户提问**："本次改动属于哪个模块？可选：frontend / backend / model / docs / root。"
> 3. 用户回答："backend。"
> 4. **过滤改动**：把 `git status --porcelain` 的文件按 `^backend/` 白名单筛选，越界文件列表报告给用户。
> 5. `git add backend/<白名单内文件>`（明确路径，不使用 `git add .`）。
> 6. 后续 commit / push / PR 流程同示例 A，但 PR 标题前缀改为 `feat(backend):`。

### 示例 C：改动跨模块（拆分拒绝）

> **用户**：我刚改了 `frontend/` 和 `backend/` 一起提交吧。
>
> **智能体**：
> 1. 执行 `git status` → 检测到同时存在 `frontend/src/x.vue` 与 `backend/src/y.py` 的改动。
> 2. **触发 §4.5 跨模块场景**，向用户说明：
>    > "检测到改动跨越 frontend 与 backend 两个模块。按模块拆分建议：
>    > - frontend 改动 → 请 frontend 负责人执行
>    > - backend 改动 → 请 backend 负责人执行
>    > 是否要为每个模块各开一个分支、分别提 PR？"
> 3. **默认拒绝**单次多模块提交，等用户给出明确拆分方案后再继续。