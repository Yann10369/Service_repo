# Skill: 自动上传代码并申请 PR

> 当智能体（Claude Code / Cursor / 其他 Agent）读到本文档时，应能按照下面的步骤，**自动完成本地代码提交并通过 `gh` CLI 创建 Pull Request**。

---

## 1. 触发条件

满足以下任意一条时触发本技能：

- 用户明确说："提交代码"、"上传代码"、"push"、"创建 PR"、"提 PR"。
- 用户给出修改但未指明提交方式，并使用 `/pr`、 `/push` 等指令。
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

任何一项缺失：先**询问用户**而不是猜测。

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

### Step 2. 确认改动范围

```bash
git diff --stat
git diff
```

- 如果改动是用户**明确要求**的，直接进入 Step 3。
- 如果改动是用户**没明确要求**的（例如上次会话遗留、合并冲突等），**先停下**，列出差异让用户确认。

### Step 3. 暂存与提交

```bash
git add <files...>          # 推荐按文件粒度 add，避免误提交
git status                  # 再确认一次
git commit -m "<type>: <subject>

<body 说明本次改动的目的、影响范围、关联 issue>

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

提交信息规范（Conventional Commits）：

- `feat:` 新功能
- `fix:` 修复 bug
- `docs:` 仅文档改动
- `refactor:` 重构（不改变行为）
- `test:` 测试相关
- `chore:` 构建、工具、依赖等杂项

### Step 4. 推送到远端

```bash
git push origin <current-branch>
```

如果推送失败：

- **非快进拒绝 (non-fast-forward)**：先 `git fetch`，再与用户确认是 `rebase` 还是 `merge`。
- **鉴权失败**：提示用户配置凭证或执行 `gh auth login`。
- **分支受保护**：提示用户走 PR 流程。

### Step 5. 创建 Pull Request

仅在推送到非默认分支（或用户明确要求）时执行：

```bash
gh pr create \
  --base main \
  --head <current-branch> \
  --title "<type>: <short title>" \
  --body "<PR body>"
```

PR body 模板（中文）：

```markdown
## 改动说明
- <本次主要改动点 1>
- <本次主要改动点 2>

## 改动类型
- [ ] 新功能 (feat)
- [ ] 修复 (fix)
- [ ] 重构 (refactor)
- [ ] 文档 (docs)
- [ ] 其他

## 测试
- [ ] 已通过本地单元测试
- [ ] 已通过本地 lint / type-check
- [ ] 已人工验证关键路径

## 关联 Issue
Closes #<issue-id>  （如有）

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Step 6. 回报结果

向用户输出：

1. 提交哈希 (commit SHA)
2. 远端分支名
3. PR 链接（如已创建）
4. 任何告警或遗留事项

---

## 4. 常见场景

### 4.1 在默认分支直接提交

如果当前在 `main` / `master`：

- **优先策略**：从默认分支切出 feature 分支再提交与推送，避免污染主干。
- 命令：`git checkout -b feat/<short-desc>`

### 4.2 没有改动

```bash
git status --porcelain
```

输出为空时，**不要**创建空提交，直接告知用户"没有需要提交的改动"。

### 4.3 存在未跟踪文件

- 如果是新文件且属于本次改动：`git add <file>`。
- 如果是构建产物、依赖、日志等：先检查 `.gitignore`，必要时**追加**进 `.gitignore` 而不是提交。

### 4.4 需要合并上游最新代码

```bash
git fetch origin
git rebase origin/main     # 或 merge，看团队规范
```

rebase 冲突时停下来，**不要** 自动 `--force` 或 `-X theirs`。

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

> **用户**：帮我把 `frontend/` 下新增的登录页面提交并提个 PR。
>
> **智能体**：
> 1. 执行 `git status` → 确认当前分支 `feat/login-page`，新增 `frontend/src/pages/Login.vue`。
> 2. 执行 `git diff frontend/src/pages/Login.vue` → 确认改动符合用户描述。
> 3. `git add frontend/src/pages/Login.vue`
> 4. `git commit -m "feat(frontend): 新增登录页面"`
> 5. `git push origin feat/login-page`
> 6. `gh pr create --base main --head feat/login-page --title "feat(frontend): 新增登录页面" --body "..."`
> 7. 向用户回报：commit `abc1234`、分支 `feat/login-page`、PR 链接 `https://github.com/.../pull/12`。