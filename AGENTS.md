# Repository Guidelines

## Project Structure & Module Organization

This repository preserves reusable notes for autumn recruiting preparation. Keep content organized by learning area rather than by date.

- `README.md` explains the repository scope and current navigation.
- `leetcode/` contains topic templates, numbered in learning order: `00-基础语法与复杂度.md`, `01-哈希表.md`, and so on.
- `leetcode/题目复盘/` contains one Markdown file per completed problem. Its `README.md` defines the required retrospective template.
- `agent/` contains Agent project retrospectives, numbered in learning order, for example `00-第一个旅行助手Agent.md`.

When adding a new LeetCode topic, use the next two-digit prefix and a concise Chinese title, for example `04-二分查找.md`. Put a problem-specific reflection in `题目复盘/`, not in a general topic note. When adding an Agent project note, also use the next two-digit prefix and keep one project per file.

## Build, Test, and Development Commands

There is no build system, package manager, or automated test suite. Work directly with Markdown files and preview them in a Markdown-capable editor before committing. Useful repository checks are:

```powershell
git status          # review changed note files
git diff --check    # detect whitespace errors
git log --oneline   # inspect recent documentation conventions
```

Do not add generated site output, editor metadata, or dependencies unless the repository introduces a documented publishing workflow.

## Git 操作纪律

- 执行 `git add`、`git commit` 或 `git push` 后，优先直接读取命令结果和 `git status`，不要无依据地重复等待或重复执行。
- 如果命令因权限审核、工具超时或进程中断而没有明确结果，先检查 `git log`、`git status` 和远端分支，再决定是否重试。
- 用户要求推送时，完成后必须明确汇报提交号、推送结果和仍未提交的无关改动；不要让用户在不确定状态下等待。
- 只暂存本次任务涉及的文件，保留 `README.md`、`agent/` 等其他未提交改动，不要误提交或覆盖。
- 工具调用异常时，及时向用户说明当前状态和下一步，不要连续发送空等待或重复的无变化更新。

## Writing Style & Naming Conventions

Write notes in clear Chinese with Markdown headings. Start each document with one `#` title and use `##` sections for major ideas. Prefer short explanations, focused code blocks (currently Python), and tables for concise comparisons such as complexity costs. Keep code examples runnable and explain non-obvious tradeoffs or edge cases near the example.

For problem retrospectives, follow the existing sections exactly: `题意`, `我的思路`, `标准模板`, `踩坑点`, and `复习记录`. Use problem titles as filenames, e.g. `移动零.md`; retain standard punctuation and avoid duplicate versions of the same problem.

## Testing Guidelines

Validate documentation changes manually: preview heading hierarchy, verify fenced-code language tags, and confirm internal paths and filenames are accurate. For algorithm notes, mentally check the stated complexity and run supplied snippets against normal and boundary inputs when practical. There is no coverage target.

## Commit & Pull Request Guidelines

Recent commits use imperative Conventional Commit-style documentation messages, for example `docs: add sliding window notes` and `docs: add container problem notes`. Follow the same pattern: `docs: add <topic or problem> notes`.

Keep each commit focused on one topic or problem. Pull requests should summarize the note added or corrected, list affected paths, and call out substantive template or navigation changes. Include screenshots only when rendered Markdown layout is materially relevant.
