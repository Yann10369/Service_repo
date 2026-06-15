# Test PR Workflow

> 本文件用于验证 `pr_git.md` 工作流的端到端可用性，验证通过后可删除。

## 验证项

- [ ] 模块归属识别（docs）
- [ ] 模块白名单过滤（仅 `docs/` 下文件可提交）
- [ ] 分支命名 `test/pr-workflow-validation`
- [ ] 提交信息格式 `<type>(<module>): <subject>`
- [ ] 仅 `git add docs/<file>`，未使用 `git add . / -A`
- [ ] 按 docs 提交并推送
- [ ] 创建 PR（默认走 GitHub REST API，无需 gh CLI）

## 验证时间

2026/06/15