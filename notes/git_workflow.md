# Git 协作流程要点

- 主分支通常叫 `main`，功能开发从 `main` 切 `feature/xxx`。
- 提交前 `git status` / `git diff` 确认改动范围。
- 用 PR/MR 做代码评审，而不是直接推主分支。
- 合并方式：merge（保留历史）或 rebase（线性历史）。
- 误提交用 `git commit --amend`；已推送的不要随意 `--force`。
