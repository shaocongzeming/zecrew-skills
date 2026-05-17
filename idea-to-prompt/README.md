# idea-to-prompt

## 简介

把模糊想法变成可直接粘贴给任意 AI 工具执行的精确 Prompt。

不限于开发——软件、PPT、海报、视频、活动方案,任何「先有想法再用 AI 工具去做」的场景都适用。

## 解决什么问题

每次想用 AI 做点什么,但说不清楚:
- 「我想做个工具」 → 做啥工具?给谁用?
- 「帮我做个 PPT」 → 给谁看?多长?什么风格?
- 「我想生张图」 → 什么风格?用什么模型?多大尺寸?

结果 AI 出的东西总跟想的不一样,反复改。

这个 skill 用「六心法」帮你先聊明白,再生成一份精确的 Prompt 文件,粘贴给目标 AI 工具(Claude Code / Cursor / Midjourney / Sora / Gamma 等)就能直接执行。

## 功能

### 六心法对话流程

1. **选角** — 选定 1-2 个最适合的专家角色,用专业视角对话
2. **苏格拉底追问** — 一次一个问题,追问到 95% 信心
3. **建设性辩论** — 切换反对者视角找漏洞,每挑一个洞顺手给加固建议
4. **失败预演** — 假设项目已失败,识别真实风险
5. **反向提示** — AI 推荐 1-2 个执行方案
6. **双层解释** — 技术名词用「洗脚城大爷版」+「为什么选它」翻译

### 四类项目适配

- **开发类**:软件/网站/小程序/app/skill/工具/自动化 → Claude Code / Cursor / v0
- **长内容类**:PPT/报告/方案/文案/视频脚本 → ChatGPT / Claude / Gamma
- **视觉素材类**:AI 生图/AI 生视频/3D → Midjourney / Sora / 可灵 / 即梦 / Seedance
- **策划类**:活动/课程/社群/流程 → ChatGPT / Claude / Notion AI

### 三档深度

- ⚡ **闪电(5 分钟)** — 简单需求、单页文案、需求明确
- 🚶 **标准(20-40 分钟)** — 一般功能项目、PPT、活动方案
- 🧗 **深度(1-2 小时)** — 复杂产品、长期项目、大投入

## 触发条件

- 「我想做个东西」、「我有个想法」、「我想开发」
- 「帮我做个 PPT/海报/视频/方案」
- 「帮我理一下思路」、「先聊一下需求」

## 不适用场景

- 纯调研(用 research)
- 纯写作(用 khazix-writer)
- 已有明确 spec 只需要执行(用 writing-plans)
- 已在 git 项目里要做 software feature(让位给 brainstorming——它有 spec → writing-plans 流水线)
- 已有 `Prompt-*.md` 文件 + 开始建项目(让位给 project-init)
- 一句话能说清的简单任务(直接交给对应 skill,不走六心法)

## 输出示例

```markdown
# 项目:个人记账小程序

## 目标 AI 工具
Claude Code + 微信小程序原生

## 一句话描述
按场景自动分类的家庭记账小程序

## 背景与动机
现有记账 app 太复杂,妻子用不下来。想要一个手动记一笔后,
能根据备注关键词自动分类的极简版本。

## 目标用户
夫妻二人,妻子手机操作不熟练

## 核心功能
1. 一键记账 — 输入金额 + 一句话备注
2. 自动分类 — 根据关键词匹配预设类别(餐饮/交通/购物...)
3. 月度总览 — 按类别看花销

## 技术方案
- 前端:微信小程序原生
- 后端:Supabase(免运维)
- 大爷版解释:Supabase 就像一个开箱即用的存档中心,
  你的小程序往里存数据、取数据都靠它,不用自己搭服务器

## 风险与注意事项
(失败预演中识别的关键风险)
```

## 安装

```bash
mkdir -p ~/.claude/skills/idea-to-prompt
curl -o ~/.claude/skills/idea-to-prompt/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/idea-to-prompt/SKILL.md
```

## 配合使用

- **后续接力 `project-init`**:聊完 prompt 后,把 Prompt-*.md 留在 `~/Desktop` 或 cwd,project-init 启动时自动检测并接力,只问差量
- **不与 `brainstorming` 抢**:git 化的软件开发让位给 brainstorming(它有 spec → writing-plans 流水线 + 自动 commit)
