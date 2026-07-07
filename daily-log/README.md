# daily-log

## 简介

每日工作日志 Skill。用于在用户主动触发时,从授权项目和轻量文件痕迹中生成 Markdown 日志草稿。

它解决的是「每天做了很多事,但结束后没精力复盘」的问题。核心定位是**跨项目全局复盘**:项目细节仍然归各项目的 `项目开发日志.md`,daily-log 只记录全局摘要、项目进展链接、当天想法、决策和明天要做什么。

## 解决什么问题

一天里同时用 Claude Code、Codex、浏览器、文档和多个项目目录工作,最后常见问题是:
- 记不清今天到底推进了哪些项目
- git commit 有记录,但分散在多个仓库
- 很多重要工作没有 commit,只留下文件时间戳或目录变化
- 项目级 `项目开发日志.md` 太细,daily review 又需要全局视角
- 想保留日记和复盘,但不想上后台监控或读隐私数据

这个 skill 的目标是:**用户主动触发 + 授权范围内采集 + 生成可手改的 Markdown 日志**。

## 功能

### 1. 当日/历史日志生成

支持写今天、补昨天、补指定日期,默认输出到:

```text
$LOG_ROOT/YYYY/YYYY-MM-DD.md
```

### 2. 项目索引驱动

通过 `projects-index.md` 控制授权扫描范围,每行一个项目路径。skill 会读取每个项目的:
- 当日 git commit 摘要
- 是否存在 `.git`
- 是否存在 `项目开发日志.md`
- 项目路径和项目日志链接

### 3. 工作区轻量扫描

默认会轻量扫描:

```text
$WORK_ROOT
```

只看顶层/二层近期文件和目录,用于发现:
- 新项目目录
- 公众号文章草稿
- 媒体下载/转录/字幕/摘要
- 全局资产
- skill 文件更新
- AI 网络环境配置等近期工作痕迹

### 4. 全局扫描模式

当用户明确说「全局扫描一下」「全局扫描最近几天再更新日志」时,会额外轻量扫描:

```text
$HOME/Desktop
$HOME/.ai-os
$HOME/.agents/skills
$HOME/.codex/skills
$HOME/.claude/skills
```

全局扫描仍然只看路径、文件名、修改时间、git commit 和 git status 摘要,不读取浏览器历史、聊天记录、系统日志、账号配置或密钥文件。

### 5. 追加更新而非覆盖

如果目标日志已经存在:
- 保留原文和手写内容
- 避免重复追加同一 commit 或路径
- 按重要性把新增线索合并到对应章节
- 只有用户明确要求「整理成最终版」时才做重排

### 6. 周/月回顾(读已写日记找规律)

当用户说「本周复盘」「月度回顾」「看看这周日记有什么规律」「分析下我最近的日记」时进入本模式。

和模式 1-5 的本质区别:**1-5 是写入,6 是读取与发现**。

- 输入是用户**已经写过**的日记原始文本(`Daily-Logs/YYYY/*.md`),不是 git commit、不是文件清单。
- 任务是发现:情绪模式、反复出现的关键词与焦虑触发、自我感知与事实的偏差,而不是补写流水。
- 报告默认存到 `Daily-Logs/weekly-reviews/YYYY-MM-DD_to_YYYY-MM-DD.md`。
- 如果命中日志 < 3 天,会提示原始素材太少,不会用 git commit 拼凑替代 —— 因为「如果连原始材料都是 AI 写的,那 AI 帮你做回顾就成了 AI 在读自己写的东西」。

### 7. 手动补充区保真

skill 在任何写入模式下,都对用户口述/写入的「想法 / 决策 / 明天接着做 / 感悟」等手动区原文保留:

- 不重写、不润色、不「结论先行」重排
- 即使句子不通顺、有错别字、情绪化、前后矛盾也照原样留
- 最多只去掉纯口癖(「嗯嗯啊啊」)
- 结论先行的规范只适用于自动采集区(commit 摘要、文件清单、扫描说明)

日记的价值是真实,不是文笔。AI 润色过的不是录音,是翻唱。

## 触发方式

- "写今天日志"
- "今日复盘"
- "记录一下今天"
- "更新今天日志"
- "补写昨天日志"
- "补 2026-05-17 的日志"
- "全局扫描一下再更新日志"
- "本周复盘"、"月度回顾"、"看看这周日记有什么规律"、"分析下我最近的日记"
- "daily log"、"weekly review"、"monthly review"

## 默认输出

```text
$LOG_ROOT/
├── projects-index.md
├── 2026/
│   ├── 2026-05-17.md
│   └── ...
└── weekly-reviews/      # 周/月回顾报告(模式 6 输出)
    └── 2026-05-12_to_2026-05-19.md
```

`projects-index.md` 是授权扫描清单,每行一个项目路径:

```markdown
# 活跃项目

- $WORK_ROOT/个人skills仓库
- $WORK_ROOT/03_示例项目A
- $WORK_ROOT/04_示例项目B
- ~/Projects/project-a
```

## 隐私边界

- 不后台监听
- 不读取浏览器历史
- 不读取聊天记录
- 不读取系统日志
- 不扫描全盘
- 不上传外部服务
- 不记录 API key、密码、token 等敏感信息
- 默认不扫描 `projects-index.md` 列表之外的项目目录
- 全局扫描也只看明确候选目录中的路径、时间和 git 摘要

## 不适用场景

- 后台自动监控电脑
- 替代 ai-os 记忆系统
- 替代项目级 `项目开发日志.md`
- 在用户没有原始日记时凭空"生成"日记内容 —— skill 不为用户代写主观经历
- 用 AI 润色用户的日记原话 —— 手动补充区一律保真
- 记录敏感客户资料、账号、密码、token
- 自动上传、同步或发布日志

## 输出示例

```markdown
# 2026-05-17 日志

## 今天做了什么
- 更新个人 skills 仓库,新增 daily-log skill
- 修正 Daily-Logs,补充多个项目和 ai-os 相关进展

## 项目进展
### 示例:个人 skills 仓库
- `882d99e` feat: add daily log skill
- 详情见: $WORK_ROOT/个人skills仓库

## 工作区新增/修改
- `$WORK_ROOT/某内容目录/` 出现多篇 `.docx` 草稿

## 想法
- [手动补充]

## 自动采集说明
- 扫描范围: projects-index.md 授权项目 + 工作区顶层近期文件/目录 + Downloads 顶层当天文件
- 生成时间: 2026-05-17 07:30
```

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

如果要保持 README 一起同步:

```bash
rsync -a --delete daily-log/ ~/.claude/skills/daily-log/
rsync -a --delete daily-log/ ~/.agents/skills/daily-log/
```

## 配合使用

- **`project-init`** — 给单个项目建立 `项目开发日志.md`;daily-log 只做跨项目摘要和链接。
- **`github-analyzer`** — 分析外部 GitHub 项目后,重要结论可进入当天日志。
- **ai-os** — daily-log 遵守 ai-os 权限和协作边界,但不写入永久规则或长期记忆。

## 更新日志

- **v3**(2026-05-19)
  - 新增「手动补充区保真」核心原则:用户口述/写入内容原文保留,不重写不润色不"结论先行",最多去口癖。结论先行规范只适用于自动采集区。
  - 新增工作流模式 6「周/月回顾」:读已写日记找规律(情绪模式 / 反复关键词与焦虑触发 / 自我感知与事实偏差),输入必须是用户已写的日记原文,不允许用 git commit 拼凑替代。报告默认存到 `Daily-Logs/weekly-reviews/`。
  - 灵感来自数字生命卡兹克《AI 时代,为什么我极力推荐你开始写日记》对 AI 润色和回顾用法的论述。

- **v2**(2026-05-17)
  - 增加 `工作区` 顶层/二层轻量扫描
  - 增加「全局扫描」模式,可在用户明确授权时扫描 ai-os、Codex/Claude skill 安装目录和 git 子仓库
  - 增加 `工作区新增/修改` 日志章节
  - 修正示例项目路径
  - 明确不读取浏览器历史、聊天记录、系统日志和敏感配置

- **v1**(2026-05-17)
  - 初版 daily-log skill
  - 支持当日日志、历史补写、已有日志追加更新
  - 基于 `projects-index.md` 授权项目采集 git commit 和项目日志链接
