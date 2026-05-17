# shaocong-skills

韶聪的 Claude Code 自定义 Skills 仓库。

## Skills 列表

| Skill | 说明 | 触发方式 |
|-------|------|----------|
| [github-analyzer](./github-analyzer/) | GitHub 开源项目分析与评估。多来源识别 + 并行调研 + 量化评分 + 竞品对比 + 批量汇总 | `/github-analyzer` 或分享 GitHub 链接/截图 |

## 如何使用

### 方式一：直接安装到 Claude Code

将目标 skill 的 `SKILL.md` 复制到你的 `~/.claude/skills/` 目录下即可：

```bash
# 示例：安装 github-analyzer
mkdir -p ~/.claude/skills/github-analyzer
curl -o ~/.claude/skills/github-analyzer/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/github-analyzer/SKILL.md
```

### 方式二：Claude Code 自动识别

在你的 Claude Code 会话中，直接告诉 Claude：

> "帮我安装 shaocong-skills 仓库里的 github-analyzer skill"

Claude 会自动下载并配置。

## 贡献

这是个人技能仓库，如有建议欢迎提 Issue。

## License

MIT
