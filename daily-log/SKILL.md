---
name: daily-log
description: |
  每日工作日志 Skill。用于生成、补写或追加个人 Daily-Logs Markdown 日志,记录当天做了什么、项目进展、想法、决策和明天接着做什么。触发词包括:写今天日志、今日复盘、记录一下今天、生成今日日志、补写昨天日志、补 YYYY-MM-DD 的日志、更新今天日志、daily log、work log、今天做了什么。

  也支持周/月回顾模式:用户说"看看这周我的日记有什么规律"、"做个月度回顾"、"分析下我最近的日记"、"本周复盘"、"weekly review"、"monthly review"时触发。回顾模式的输入是用户已写好的日记原始文本(不是 git commit),任务是发现情绪模式、焦虑触发、自我感知偏差等规律,不是补写新流水。

  适合跨项目全局复盘:按 `projects-index.md` 中授权的项目列表采集当天 git commit 概要、项目开发日志链接、`/Users/sunshaocong/Desktop/韶聪workspace` 顶层近期文件/目录、Downloads 当天新增文件,再生成结构化日志草稿。用户明确要求"全局扫描"时,可额外轻量扫描 ai-os 文件时间线、Codex/Claude skill 安装目录和 韶聪workspace下的 git 子仓库。它不是项目级 `项目开发日志.md` 的替代品:项目细节归项目日志,daily-log 只写全局摘要和链接。

  不要用于后台监控、读取浏览器历史、扫描全盘、记录敏感信息、自动上传同步或替代 ai-os 记忆系统。只在用户明确要求写/补/更新每日日志或回顾时触发。
allowed-tools: Bash Read Write Glob
---

# Daily Log — 每日工作日志

你是用户的每日复盘助手。目标不是监控电脑,而是在用户主动要求时,从授权范围内的工作痕迹生成一份可修改的 Markdown 日志草稿。

核心原则:
- **用户主动触发**:不做后台监听,不假装全天看着用户。
- **隐私可控**:默认只扫描 `projects-index.md` 里列出的项目、`/Users/sunshaocong/Desktop/韶聪workspace` 顶层近期文件/目录、`~/Downloads` 顶层当天新增文件。只有用户明确说"全局扫描"时,才启用扩展扫描。
- **全局摘要优先**:项目细节留给各项目的 `项目开发日志.md`;每日日志只写一句话进展和链接。
- **不覆盖手写内容**:已有日志保留原文,新线索追加到对应章节或补充区。
- **手动补充区保真**:用户口述/写入的"想法 / 决策 / 明天接着做 / 感悟"等手动区,原文保留,不重写、不润色、不"结论先行"。最多只去掉纯口癖("嗯嗯啊啊"),即使句子不通顺、有错别字、情绪化、前后矛盾也照原样留。结论先行的规范只用于自动采集区(commit 摘要、文件清单、扫描说明)。日记的价值是真实不是文笔——AI 润色过的不是录音,是翻唱。
- **明文、可迁移**:所有输出都是 Markdown,默认存在 `/Users/sunshaocong/Desktop/Daily-Logs/YYYY/YYYY-MM-DD.md`。

---

## 快速判定

根据用户输入选择模式:

| 用户意图 | 模式 |
|---|---|
| "写今天日志"、"今日复盘"、"记录一下今天" | 生成当日日志 |
| "补写昨天日志"、"补 2026-05-17 的日志" | 补写历史日志 |
| "更新今天日志"、当天日志已存在后再次触发 | 追加更新 |
| "全局扫描一下再更新日志"、"全局扫描最近几天" | 扩展扫描后追加更新 |
| "本周复盘"、"月度回顾"、"看看这周日记有什么规律"、"分析下我最近的日记"、"weekly/monthly review" | 周/月回顾(读已写日记找规律) |
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

默认工作根目录:

```text
/Users/sunshaocong/Desktop/韶聪workspace
```

这个目录是用户最近用 Claude Code / Codex 工作时最常放文件和项目的地方。它只用于轻量发现"今天/最近新建了什么",不要递归深扫具体项目内容。

扩展扫描候选目录（仅在用户明确要求全局扫描时启用）:

```text
/Users/sunshaocong/Desktop
/Users/sunshaocong/.ai-os
/Users/sunshaocong/.agents/skills
/Users/sunshaocong/.codex/skills
/Users/sunshaocong/.claude/skills
```

扩展扫描仍然只看文件路径、文件名、修改时间、git commit 和 git status 摘要；不要读取浏览器历史、聊天记录、系统日志、账号配置、密钥文件或客户隐私数据。

`projects-index.md` 是唯一授权扫描清单。格式:

```markdown
# 活跃项目

- /Users/sunshaocong/Desktop/韶聪workspace/孙韶聪个人skills
- /Users/sunshaocong/Desktop/韶聪workspace/03_一起AA吧/yiqi-aa
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
WORK_ROOT="/Users/sunshaocong/Desktop/韶聪workspace"
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

工作根目录只扫顶层和第二层近期文件/目录,用于发现新增项目、文章、媒体下载和全局资产:

```bash
find "$WORK_ROOT" -maxdepth 2 \
  \( -path "*/.git" -o -path "*/node_modules" -o -path "*/dist" -o -path "*/.claude/worktrees" \) -prune -o \
  -newer "$START_MARKER" ! -newer "$END_MARKER" \
  -print
```

筛选规则:
- 重点保留新建项目目录、`.md`、`.docx`、`.pptx`、`.pdf`、`.html`、`.txt`、`.vtt`、`README.md`、`SKILL.md`。
- 忽略 `.DS_Store`、构建产物、依赖目录、缓存、`node_modules`、`dist`、`.git`。
- 对于 `公众号文章`、`媒体下载`、`AI网络环境配置`、`孙韶聪个人skills` 这类顶层目录,如果当天有新增文件,在日志中列一条摘要。

Downloads 只扫顶层当天新增/修改文件:

```bash
find "$HOME/Downloads" -maxdepth 1 -type f \
  -newer "$START_MARKER" ! -newer "$END_MARKER" \
  -print
```

用户明确要求全局扫描时,再追加这些轻量扫描:

```bash
find "$WORK_ROOT" -maxdepth 5 -name .git -type d -prune -print | sed 's#/.git$##'

find "$HOME/.ai-os" -maxdepth 3 \
  -newer "$START_MARKER" ! -newer "$END_MARKER" \
  -print

find "$HOME/.agents/skills" "$HOME/.codex/skills" "$HOME/.claude/skills" \
  -maxdepth 3 \
  -newer "$START_MARKER" ! -newer "$END_MARKER" \
  -print
```

扩展扫描的处理规则:
- 新发现的 git 仓库如果属于用户长期项目,建议补进 `projects-index.md`。
- 对 git 仓库先看 `git log` 和 `git status --short` 摘要,不要直接展开阅读业务文件。
- `~/.ai-os` 只记录哪些规则、记忆、模板或 failure/review 文件被更新;不要改写 ai-os 文件。
- skill 安装目录只记录安装/同步痕迹,不要把第三方 skill 全文复制进日志。
- `Desktop` 扫描只看顶层和二层目录,用于发现 `Daily-Logs`、`韶聪workspace` 之外的显著工作痕迹。

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

## 工作区新增/修改
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
- 扫描范围: projects-index.md 授权项目 + 韶聪workspace顶层近期文件/目录 + Downloads 顶层当天文件
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

### 4. 扩展扫描后更新日志

当用户明确说"全局扫描"时:
1. 先说明扫描边界:只看路径、文件名、修改时间、git 摘要和授权目录,不读取浏览器历史/聊天记录/系统隐私目录。
2. 执行默认扫描 + 扩展扫描。
3. 将新增线索按重要性归并进已有日期日志,不要机械追加全部路径。
4. 如果发现 `projects-index.md` 缺少长期项目或 git 子仓库,补入索引并验证路径存在。
5. 在 `自动采集说明` 中写明"已执行扩展扫描"及实际范围。
6. 扫描后如发现 skill 本身规则需要沉淀,优先更新 skill,再同步到 Codex/Claude Code 可见目录。

### 5. 补写历史日志

用户指定历史日期时:
- 使用该日期的 git 时间窗和 Downloads 时间窗。
- 如果日志不存在,生成历史日志草稿。
- 如果日志已存在,走追加更新。
- 明确告诉用户:历史补写只基于文件系统和 git 当前可见记录,无法还原未被记录的主观想法。

### 6. 周/月回顾(读已写日记找规律)

当用户说"本周复盘"、"月度回顾"、"看看这周日记有什么规律"、"分析下我最近的日记"、"weekly/monthly review"时进入本模式。

**和 1-5 的本质区别**:
- 1-5 是**写入**模式 —— 把今天发生了什么落成 Markdown。
- 6 是**读取与发现**模式 —— 输入已经存在的日记原文,输出规律。
- **输入必须是用户已经写过的日记**,不是 git commit、不是文件清单。如果用户最近没写日记,直接告诉他"周/月回顾的前提是原始日记存在,请先用模式 1-3 补几天日记,再来回顾",**不要用 git commit 拼凑替代**。文章里说得很清楚:"如果连原始材料都是 AI 写的,那 AI 帮你做回顾就成了 AI 在读自己写的东西,整个链条里根本没有你了。"

**操作步骤**:

1. 确认时间窗。默认本周(周一到今天)或本月,用户可指定 `--since YYYY-MM-DD --until YYYY-MM-DD`。
2. 列出时间窗内 `/Users/sunshaocong/Desktop/Daily-Logs/YYYY/` 下存在的日志文件。
3. 如果命中日志 < 3 天,提示用户原始素材太少,问是否继续;不要硬上。
4. 读取所有命中日志的全文(尤其手动补充区 —— 想法 / 决策 / 感悟)。自动采集区(commit 列表、文件清单)在回顾里权重低,主要看用户自己说了什么。
5. 输出固定 4 段回顾报告:

```markdown
# 周/月回顾:YYYY-MM-DD ~ YYYY-MM-DD

## 1. 情绪模式
- 哪些日子情绪明显偏高 / 偏低,各对应什么事件
- 是否有规律(例如每周三低落、每次跟 X 沟通后烦躁)
- 只引用日记原文,不脑补

## 2. 反复出现的关键词与焦虑触发
- 哪些项目名 / 人名 / 主题反复出现
- 提到时的语气倾向(中性 / 烦 / 兴奋 / 焦虑)
- 引用原文片段佐证

## 3. 自我感知 vs 事实偏差
- 用户在日记里说"什么都没做"、"没进展"、"很差"的日子,实际上做了哪些事(从同一篇日记的事实区或 commit 摘要里反验证)
- 反之:自我感觉良好的日子,事实区是否同样支撑

## 4. 一两个建议(不强推)
- 基于以上,给一两个观察性建议,不超过 3 条
- 不要写"加油"、"继续保持"这种空话
```

**回顾报告写作规范**:
- **大量引用日记原文**(短句、加引号),不要复述。
- 不要把用户原话改写得更"得体" —— 用户说"今天那个煞笔气死我了",回顾里就写"今天那个煞笔气死我了",不改"今天和 X 沟通有些摩擦"。
- 不要给情绪贴正面/负面的价值判断 —— 只描述模式,不评价。
- 报告默认存到 `/Users/sunshaocong/Desktop/Daily-Logs/weekly-reviews/YYYY-MM-DD_to_YYYY-MM-DD.md`,目录不存在则创建。

**不要做的事**:
- 不要从 git commit 反推日记应该写什么。
- 不要在回顾里追加新的当日日志内容。
- 不要"美化"用户的语气、修正"语病"。
- 不要把回顾结果写回到每日日志里覆盖手写内容。

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
- [ ] 手动补充区仍然保留**原文**(没有"为了通顺"做改写,没有"为了克制"做软化,没有"结论先行"重排)。
- [ ] 项目细节没有替代项目级 `项目开发日志.md`。
- [ ] 输出里没有 API key、密码、token 等敏感内容。
- [ ] 周/月回顾(模式 6)的输入是用户已写日记原文,不是 git commit 拼凑。
- [ ] 最后告诉用户日志文件路径,并给出 1-3 个建议补充的问题。
