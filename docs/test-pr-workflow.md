# Test PR Workflow

> 本文件用于验证 `pr_git.md` 工作流的端到端可用性，验证通过后可删除。

## 验证项

- [x] 模块归属识别（docs）
- [x] 模块白名单过滤（仅 `docs/` 下文件可提交）
- [x] 分支命名 `test/pr-workflow-validation`
- [x] 提交信息格式 `<type>(<module>): <subject>`
- [x] 仅 `git add docs/<file>`，未使用 `git add . / -A`
- [x] 按 docs 提交并推送
- [x] gh pr create 流程

## 验证时间

2026/06/15