# daily-log

## 简介

每日工作日志 Skill。用于在用户主动触发时,从授权项目和轻量文件痕迹中生成 Markdown 日志草稿。

它解决的是「每天做了很多事,但结束后没精力复盘」的问题。核心定位是**跨项目全局复盘**:项目细节仍然归各项目的 `项目开发日志.md`,daily-log 只记录全局摘要、项目进展链接、当天想法、决策和明天要做什么。

## 触发方式

- "写今天日志"
- "今日复盘"
- "记录一下今天"
- "更新今天日志"
- "补写昨天日志"
- "补 2026-05-17 的日志"
- "daily log"

## 默认输出

```text
/Users/sunshaocong/Desktop/Daily-Logs/
├── projects-index.md
├── 2026/
│   ├── 2026-05-17.md
│   └── ...
└── weekly-reviews/
```

`projects-index.md` 是授权扫描清单,每行一个项目路径:

```markdown
# 活跃项目

- /Users/sunshaocong/Desktop/MacBookpro备份内容/孙韶聪个人skills
- ~/Projects/project-a
```

## 隐私边界

- 不后台监听
- 不读取浏览器历史
- 不扫描全盘
- 不上传外部服务
- 不记录 API key、密码、token 等敏感信息
- 不扫描 `projects-index.md` 列表之外的项目目录

## 安装

```bash
mkdir -p ~/.claude/skills/daily-log
curl -o ~/.claude/skills/daily-log/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/daily-log/SKILL.md
```

Codex 可复制到:

```bash
mkdir -p ~/.agents/skills/daily-log
cp daily-log/SKILL.md ~/.agents/skills/daily-log/SKILL.md
```
