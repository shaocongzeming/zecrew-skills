# shaocong-skills

韶聪的 Claude Code 自定义 Skills 仓库。

四个 skill 围绕「用户想法 → AI 工具执行 → 每日复盘」的完整链路:

| 阶段 | 用户状态 | 对应 Skill |
|------|---------|-----------|
| 1. 想清楚 | 有想法但说不清,需要先聊明白 | **idea-to-prompt** |
| 2. 搭骨架 | 想法明确,要建项目 | **project-init** |
| 3. 评估外部工具 | 想了解 GitHub 上的开源工具 | **github-analyzer** |
| 4. 每日复盘 | 想记录今天做了什么、想法和项目进展 | **daily-log** |

## Skills 列表

| Skill | 说明 | 触发方式 |
|-------|------|----------|
| [idea-to-prompt](./idea-to-prompt/) | 把模糊想法变成可粘贴给任意 AI 工具(Claude Code / Cursor / Midjourney / Sora / Gamma…)的精确 Prompt。六心法追问 + 四类项目适配 + 三档深度 | "我想做个东西"、"帮我理一下思路"、"我有个想法" |
| [project-init](./project-init/) | 新项目「上下文入口」搭建。基于 ai-os 落地实践,产物是一份 fresh agent 读完就能上手的「项目开发日志.md」。支持新建/补建/刷新三种模式 | "开始新项目"、"搭项目骨架"、"按这个 prompt 建项目" |
| [github-analyzer](./github-analyzer/) | GitHub 开源项目分析与评估。多来源识别 + 意图驱动报告 + 量化评分 + 风险横幅 + 批量汇总 + 持久化知识库 | 分享 GitHub 链接/截图,或"帮我看看这个项目" |
| [daily-log](./daily-log/) | 每日工作日志。按授权项目索引采集当天 git 概要、项目日志链接和 Downloads 顶层新增文件,生成可修改的 Markdown 日志草稿 | "写今天日志"、"今日复盘"、"补写昨天日志" |

## 如何使用

### 方式一:直接安装到 Claude Code

一次装齐四个 skill:

```bash
for s in idea-to-prompt project-init github-analyzer daily-log; do
  mkdir -p ~/.claude/skills/$s
  curl -o ~/.claude/skills/$s/SKILL.md \
    https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/$s/SKILL.md
done
```

只装单个:

```bash
mkdir -p ~/.claude/skills/idea-to-prompt
curl -o ~/.claude/skills/idea-to-prompt/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/idea-to-prompt/SKILL.md
```

### 方式二:Claude Code 自动识别

在你的 Claude Code 会话中,直接告诉 Claude:

> "帮我安装 shaocong-skills 仓库里的 idea-to-prompt skill"

Claude 会自动下载并配置。

## 设计原则

三个 skill 之间互相**让位不撞触发**:

- `idea-to-prompt` 让位给 `brainstorming`(git repo + 软件 feature 改动场景)
- `idea-to-prompt` 让位给 `project-init`(已有 Prompt 文件 + 开始建项目)
- `project-init` 自动接力 `idea-to-prompt`(检测到 `Prompt-*.md` 自动只问差量)
- `project-init` 跟内置 `/init` 划清(`/init` 只建 CLAUDE.md,本 skill 建整套骨架)

详见每个 skill 的 SKILL.md description 里的 `MUST 不触发` 段落。

## 贡献

这是个人技能仓库,如有建议欢迎提 Issue。

## License

MIT
