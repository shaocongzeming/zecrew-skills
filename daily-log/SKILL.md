---
name: daily-log
description: |
  每日工作日志 Skill。用于生成、补写或追加个人 Daily-Logs Markdown 日志,记录当天做了什么、项目进展、想法、决策和明天接着做什么。触发词包括:写今天日志、今日复盘、记录一下今天、生成今日日志、补写昨天日志、补 YYYY-MM-DD 的日志、更新今天日志、daily log、work log、今天做了什么。

  适合跨项目全局复盘:按 `projects-index.md` 中授权的项目列表采集当天 git commit 概要、项目开发日志链接、Downloads 当天新增文件,再生成结构化日志草稿。它不是项目级 `项目开发日志.md` 的替代品:项目细节归项目日志,daily-log 只写全局摘要和链接。

  不要用于后台监控、读取浏览器历史、扫描全盘、记录敏感信息、写 weekly review、自动上传同步或替代 ai-os 记忆系统。只在用户明确要求写/补/更新每日日志时触发。
allowed-tools: Bash Read Write Glob
---

# Daily Log — 每日工作日志

你是用户的每日复盘助手。目标不是监控电脑,而是在用户主动要求时,从授权范围内的工作痕迹生成一份可修改的 Markdown 日志草稿。

核心原则:
- **用户主动触发**:不做后台监听,不假装全天看着用户。
- **隐私可控**:只扫描 `projects-index.md` 里列出的项目和 `~/Downloads` 顶层当天新增文件。
- **全局摘要优先**:项目细节留给各项目的 `项目开发日志.md`;每日日志只写一句话进展和链接。
- **不覆盖手写内容**:已有日志保留原文,新线索追加到对应章节或补充区。
- **明文、可迁移**:所有输出都是 Markdown,默认存在 `/Users/sunshaocong/Desktop/Daily-Logs/YYYY/YYYY-MM-DD.md`。

---

## 快速判定

根据用户输入选择模式:

| 用户意图 | 模式 |
|---|---|
| "写今天日志"、"今日复盘"、"记录一下今天" | 生成当日日志 |
| "补写昨天日志"、"补 2026-05-17 的日志" | 补写历史日志 |
| "更新今天日志"、当天日志已存在后再次触发 | 追加更新 |
| "创建 daily log 配置"、配置缺失 | 初始化配置 |

日期规则:
- 默认日期为本地 `date +%Y-%m-%d`。
- "昨天"用本地日期减一天。
- 明确 `YYYY-MM-DD` 时使用该日期。
- 模糊中文日期无法可靠解析时,先问用户确认具体日期。

---

## 文件位置

默认日志根目录:

```text
/Users/sunshaocong/Desktop/Daily-Logs/
├── projects-index.md
├── 2026/
│   ├── 2026-05-17.md
│   └── ...
└── weekly-reviews/     # 预留,本 skill 不自动生成
```

`projects-index.md` 是唯一授权扫描清单。格式:

```markdown
# 活跃项目

- /Users/sunshaocong/Desktop/MacBookpro备份内容/孙韶聪个人skills
- /Users/sunshaocong/Desktop/MacBookpro备份内容/一起AA吧/yiqi-aa
- ~/Projects/project-a
```

读取规则:
- 空行和 `#` 开头的行忽略。
- 每行一个路径,去掉前导 `- `。
- 支持 `~` 展开。
- 路径不存在时在日志里列为"待确认",不要报错中止。

如果 `/Users/sunshaocong/Desktop/Daily-Logs/projects-index.md` 不存在:
1. 创建 `/Users/sunshaocong/Desktop/Daily-Logs/`。
2. 写入最小模板,包含当前工作目录作为候选项目。
3. 告诉用户先补充常用项目路径,本次可继续只扫描当前目录。

---

## 工作流

### 1. 环境扫描

先执行轻量扫描,不要扫全盘:

```bash
LOG_ROOT="/Users/sunshaocong/Desktop/Daily-Logs"
INDEX="$LOG_ROOT/projects-index.md"
TODAY="$(date +%Y-%m-%d)"
```

读取项目列表后,对每个项目只采集这些信息:
- 是否存在 `.git`
- 当日 git commit 一句话概要
- 是否存在 `项目开发日志.md`（存在则链接,不存在则写"暂无项目开发日志"）

Git 时间窗:

```bash
git -C "$PROJECT" log \
  --since="$DATE 00:00" \
  --until="$DATE 23:59:59" \
  --oneline --decorate --all
```

Downloads 只扫顶层当天新增/修改文件:

```bash
find "$HOME/Downloads" -maxdepth 1 -type f \
  -newermt "$DATE 00:00" ! -newermt "$NEXT_DATE 00:00" \
  -print
```

注意:
- 不读取文件内容,只记录文件名和路径。
- 不扫描浏览器历史、聊天记录、系统日志、全盘最近文件。
- 不输出可能像密钥的内容。如果路径或文件名疑似含 token/key/password/secret,只写 `[疑似敏感文件名,已省略]`。

### 2. 生成日志草稿

目标文件:

```text
/Users/sunshaocong/Desktop/Daily-Logs/YYYY/YYYY-MM-DD.md
```

目录不存在就创建。日志不存在时,用下面模板生成:

```markdown
# YYYY-MM-DD 日志

## 今天做了什么
- [自动草稿] ...

## 项目进展
### 项目名
- ...
- 详情见: /absolute/path/项目开发日志.md（如不存在则写"暂无项目开发日志"）

## 下载/新增文件
- ...

## 想法
- [手动补充]

## 决策
- [手动补充]

## 明天接着做
- [手动补充]

## 会话摘要
- [手动补充]

## 自动采集说明
- 扫描范围: projects-index.md 授权项目 + Downloads 顶层当天文件
- 生成时间: YYYY-MM-DD HH:MM
```

写作要求:
- 中文,结论先行。
- commit 不要逐条堆满;每个项目最多先列 3-5 条,再给一句摘要。
- 低价值噪音少写,例如 lockfile 变化、依赖目录、临时文件。
- 如果没有线索,如实写"自动采集暂无明显线索",并提示用户手动补充。

### 3. 追加更新已有日志

如果目标日志已存在:
1. 读取原文。
2. 保留所有手动内容。
3. 对比已有文本,避免重复追加同一 commit 或文件路径。
4. 在文末或对应章节追加:

```markdown
> 补充于 YYYY-MM-DD HH:MM

### 新增线索
- ...
```

不做复杂重排,除非用户明确要求"整理成最终版"。

### 4. 补写历史日志

用户指定历史日期时:
- 使用该日期的 git 时间窗和 Downloads 时间窗。
- 如果日志不存在,生成历史日志草稿。
- 如果日志已存在,走追加更新。
- 明确告诉用户:历史补写只基于文件系统和 git 当前可见记录,无法还原未被记录的主观想法。

---

## 与 ai-os 的边界

`daily-log` 是全局日记,不是 ai-os 规则库。

不要修改:
- `~/.ai-os/INDEX.md`
- `~/.ai-os/shared-protocol.md`
- `~/.ai-os/AGENTS.md`
- 各项目的 `项目开发日志.md`

除非用户明确要求更新项目日志,否则只在每日日志里链接它。这样避免同一事实在多个地方重复、过期、互相冲突。

每日日志可以引用:
- 项目路径
- 项目开发日志路径
- commit hash
- 生成时的自动采集范围

每日日志不应该写入:
- 新的永久规则
- 身份设定
- 敏感账号和客户隐私
- 可放入具体项目日志的详细开发流水账

---

## 自检清单

交付前检查:
- [ ] 使用了正确日期和目标路径。
- [ ] 没有扫描 `projects-index.md` 之外的项目目录。
- [ ] 没有读取浏览器历史、系统日志或聊天记录。
- [ ] 已有日志没有被覆盖。
- [ ] 手动补充区仍然保留。
- [ ] 项目细节没有替代项目级 `项目开发日志.md`。
- [ ] 输出里没有 API key、密码、token 等敏感内容。
- [ ] 最后告诉用户日志文件路径,并给出 1-3 个建议补充的问题。
