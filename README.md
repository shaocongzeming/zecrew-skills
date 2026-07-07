# shaocong-skills

韶聪(shaocong)的 Claude Code 自定义 Skills 仓库。

**本仓库只收录我自己原创、或我主导的二次创作(标注出处)的 skill。** 第三方 skill 一律不 rehost,只在推荐清单里署原作者 + 给出处。

## Skills 列表

| Skill | 说明 | 归属 |
|-------|------|------|
| [idea-to-prompt](./idea-to-prompt/) | 把模糊想法追问成可直接粘贴给任意 AI 工具的精确 Prompt。六心法追问 + 四类项目适配 + 三档深度 | 原创 |
| [project-init](./project-init/) | 新项目「上下文入口」搭建。产物是一份 fresh agent 读完就能上手的「项目开发日志.md」。新建/补建/刷新三模式 | 原创 |
| [github-analyzer](./github-analyzer/) | GitHub 开源项目分析与评估。多来源识别 + 意图驱动报告 + 量化评分 + 持久化知识库 | 原创 |
| [daily-log](./daily-log/) | 每日工作日志。采集当天 git 概要/文件痕迹生成日志草稿,支持周/月回顾找规律 | 原创 |
| [vault-archive](./vault-archive/) | 结束对话时归档进 Obsidian:对话日志 + 次日日记 + 状态真相源 + 用户画像学习闭环 | 原创 |
| [digital-employee-forge](./digital-employee-forge/) | 把真实员工蒸馏成"数字分身"skill set,带 L1-L4 证据分级 | **二次创作**(见下) |

### 附带资源
- [`Skill创作SOP.md`](./Skill创作SOP.md) — 《怎么写一个 Claude Code Skill》。基于对多个高星开源 skill 的逆向研究 + Anthropic 官方 Agent Skills 规范整理。
- [`ai-os-template/`](./ai-os-template/) — 个人 AI 操作系统骨架的**脱敏模板**:一套让 Claude Code + Codex 长期协作的路由 + 协议 + skill 治理 + 失败复盘 + 项目日志机制。占位符填成你自己的即可用。
- [`推荐给覃朝瑞.md`](./推荐给覃朝瑞.md) — 我常用 skill 的对外推荐清单(含我原创的 + 我推荐的第三方,后者标了原作者出处)。

## 安装

一键装单个 skill:
```bash
s=project-init
mkdir -p ~/.claude/skills/$s
curl -sL "https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/$s/SKILL.md" \
  -o ~/.claude/skills/$s/SKILL.md
```

> 部分 skill 顶部有「配置」块,用 `$WORKSPACE_ROOT` / `$LOG_ROOT` / `$AI_OS_HOME` 等变量表示路径,默认值在 `~/` 下。装完按你自己的目录改一次即可。

## 归属与致谢

- 标「原创」的 skill 由我(shaocong)独立编写。
- **digital-employee-forge 是二次创作**:它把 `cangjie` / `nuwa`(花叔) / `anyone` / `darwin·EvoSkill` / `skill-eval` 等社区 skill 的核心机制,编排成一条面向真实企业员工的蒸馏产线。底层机制的功劳属于这些原作者,本 skill 仅原创"产线化编排"这一层。部分来源仓库仍在核实,详见各自 README 与 `推荐给覃朝瑞.md`。**欢迎原作者认领补全出处。**
- `ai-os-template` 中的编码准则源自 [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills)。

## License

MIT(见 [LICENSE](./LICENSE))。原创部分随便用随便改;二次创作部分请一并尊重上游原作者。
