# github-analyzer

## 简介

GitHub 开源项目分析与评估 Skill。当你在社交媒体上看到一个有趣的开源项目，不确定值不值得花时间研究时，用它快速得到一份实用评估报告。

## 解决什么问题

刷 Twitter、公众号、V2EX 时经常看到别人推荐 GitHub 项目，但：
- 不知道这个项目适不适合自己
- 不清楚社区真实评价如何
- 懒得自己查 Star 数、活跃度、竞品对比

这个 Skill 帮你一次性搞定，给出"装不装"的明确建议。

## 功能

1. **多来源识别** — 支持 GitHub 链接、截图（自动 OCR）、文字描述三种输入
2. **并行调研** — 同时启动多个子 Agent 收集项目基本面、社区评价、技术深度
3. **评估报告** — 一句话结论 + 有用程度评级 + 竞品对比 + 安装建议
4. **针对性强** — 结合你的实际使用场景（Claude Code 用户、已有 skill 情况等）

## 触发条件

- 用户发来 GitHub 链接（`github.com/owner/repo` 格式）
- 用户发来包含 GitHub 页面的截图
- 用户说"帮我看看这个项目"、"这个开源项目怎么样"、"值不值得装"等

## 不适用场景

- 纯代码审查（review 自己的代码）
- 创建新项目（这是开发任务）
- 简单搜索查询

## 报告示例

```markdown
# claude-code-action 分析报告

## 一句话结论
GitHub Actions 里跑 Claude，CI/CD 自动化利器

## 项目概览
- **定位**：在 GitHub Actions 中运行 Claude Code 的官方 Action
- **Stars**：1.2k | **语言**：TypeScript
- **活跃度**：活跃 — 2 天前更新
- **License**：MIT

## 评估结果
### 有用程度：高

### 建议
**推荐** — 如果你用 GitHub 做自动化 PR review，值得装
```

## 安装

```bash
mkdir -p ~/.claude/skills/github-analyzer
curl -o ~/.claude/skills/github-analyzer/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/github-analyzer/SKILL.md
```

安装后在 Claude Code 中输入 `/github-analyzer` 或直接发 GitHub 链接即可触发。
