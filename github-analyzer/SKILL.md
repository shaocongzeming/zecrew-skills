---
name: github-analyzer
description: |
  GitHub 开源项目分析与评估 Skill。当用户分享 GitHub 项目截图、链接、仓库名、博客/微博/X thread 等
  提到 GitHub 项目的内容时触发。自动从分享内容中抽取项目列表 → 按用户评估意图(想装/想学/想抄/看赛道)
  生成定制化报告 → 持久化到 `00_全局资产/github-analyses/` 形成可回溯的开源工具记忆库。

  触发词包括但不限于:帮我看看这个项目、分析一下这个GitHub项目、这个开源项目怎么样、评估一下这个工具、
  这个项目值不值得装、帮我研究一下这个repo、这个GitHub仓库有用吗、这个开源工具怎么样、
  评测一下这个项目、帮我分析这个截图里的项目、这篇文章里提到的项目都给我看看、
  这个 thread 提到的工具帮我筛一下。

  当用户直接发来 GitHub 链接(github.com/xxx/xxx 格式)时也应触发。
  当用户发来包含 GitHub 页面、推文、博客的截图时也应触发(需先识别页面类型再提取信息)。
  当用户分享的文章/帖子里提到多个 GitHub 项目时也应触发(先抽取列表再让用户勾选)。

  不要用于纯代码审查(用户要你 review 自己的代码)、不要用于创建新项目(那是开发任务)、
  不要用于简单的搜索查询。

  Star 少不等于差——很多优秀的个人 skill / 小工具只有几十 star。意图匹配优先于 star 数。

  **MUST 触发**:当用户消息**只包含一个 `github.com/owner/repo` URL 或一个 owner/repo 形式的字符串**、
  没有任何其他指令(没有"分析"、"看看"、"评估"、"研究"、"帮我"这类动词)时,**默认按"轻量入口"启动本 skill**——
  先弹 30 秒速览,再问用户要不要升级到完整流程。**不要 undertrigger**——用户发裸链接而你只回
  "想让我做什么?",等于把工具白浪费,这就是用户装这个 skill 的本意。
  轻量入口与完整流程的分流逻辑详见 SKILL.md「第一步.五」。
allowed-tools: Bash Read Write WebSearch Agent AskUserQuestion
---

# GitHub 项目分析

你正在执行一次 GitHub 项目分析。目标:**帮用户判断「这个项目对我有没有用、值不值得花时间装/学/抄」**。

最终产物是一份持久化的报告,落到 `/Users/sunshaocong/Desktop/MacBookpro备份内容/00_全局资产/github-analyses/`,长期积累形成用户的开源工具知识库。

---

## 第零步:环境扫描

并行检查 4 件事:

```bash
# 1. gh CLI 可用性(影响 API 限速,5000/h vs 60/h)
gh auth status >/dev/null 2>&1 && echo "GH_OK" || echo "GH_MISSING"

# 2. 用户工具环境(用于评估互补性)
ls ~/.claude/skills/ 2>/dev/null
ls ~/.claude/agents/ 2>/dev/null
cat /Users/sunshaocong/.ai-os/skills/_SKILL_INDEX.md 2>/dev/null
ls -d "/Users/sunshaocong/Desktop/MacBookpro备份内容/"[0-9][0-9]_* 2>/dev/null | grep -vE '/(00|99)_'

# 3. 历史分析报告
ls "/Users/sunshaocong/Desktop/MacBookpro备份内容/00_全局资产/github-analyses/" 2>/dev/null

# 4. 当前项目上下文(如有)
test -f ./项目开发日志.md && head -20 ./项目开发日志.md
```

整理成 **用户环境摘要**(约 150 字),作为固定前缀注入每个子 agent 的 prompt 末尾。内容包含:
- 已装的 skill / agent 列表(评估重叠/互补)
- 已有的桌面项目类型(评估应用场景)
- 当前项目上下文(如有)
- `gh` 是否可用

**gh 不可用时,所有 `gh api` 调用 fallback 到 curl**:
```bash
# gh 版本
gh api repos/{owner}/{repo} --jq '...'
# curl 版本(未认证,限速 60/h,够单次分析)
curl -s "https://api.github.com/repos/{owner}/{repo}" | jq '...'
```
报告末尾提示用户:`brew install gh && gh auth login` 后续体验更好。

---

## 第一步:输入识别 + 项目抽取

判断用户输入类型,分支处理:

### A. 单一 GitHub 链接 / 项目名
- 直接提取 owner/repo,进入第二步

### B. 截图(先判断页面类型,五选一)

| 页面类型 | 处理方式 |
|---------|---------|
| Repo 主页 | 提取项目名、Stars、描述、README 片段 |
| Issue / PR 页 | 提取 repo 名,跳过 stats |
| User / Org Profile | 提取 user 名,用 AskUserQuestion 问用户具体看哪个 repo |
| Commit / Code 详情 | 提取 repo 名 |
| 第三方转发(微博 / 小红书 / X / 公众号) | 走 C 分支 |

### C. 内容里提到一个或多个 repo(文章 / thread / 帖子)
- 通读全文,把所有 `github.com/owner/repo` 链接、`owner/repo` 引用、被明确提到名字的项目都抽出来
- 用 AskUserQuestion(多选,默认全选)让用户勾选要分析哪些
- ≥ 2 个时,所有项目分析完后走第八步「汇总对比」

### D. 文字描述(「一个叫 XXX 的项目」)
- WebSearch 找候选,用 AskUserQuestion 让用户确认

识别结果模糊时,**先问清楚再继续**,不要硬猜。

---

## 第一步.五:触发模式判断(裸链接的轻量入口)

**判定规则**:

| 用户消息特征 | 模式 |
|------------|------|
| **只含一个 URL / `owner/repo` 字符串**,无动词 | **🪶 轻量入口**(默认) |
| 带了动词(分析 / 看看 / 评估 / 研究 / 帮我...) | **🏋️ 完整流程** |
| 文章 / thread / 截图 / 多项目场景 | **🏋️ 完整流程** |

完整流程 → 直接进入第二步。

### 🪶 轻量入口流程

降低误触发成本:即使 Claude 自动触发了,也不直接跑 3 agent 并行 + 写盘的重操作,而是先给一份 30 秒速览,把"要不要继续"的决定权交给用户。

1. **先查历史**:`ls .../00_全局资产/github-analyzer/ | grep -i "{owner}-{repo}"`,有就直接展示上次结论 → AskUserQuestion(用上次结论 / 重新速览 / 跑完整流程)。无历史继续下一步。

2. **只跑 Agent 1 的最小子集**(不启动 Agent 2 / 3,不写 INDEX,不写完整报告):

   ```bash
   gh api repos/{owner}/{repo} --jq '{name, description, stargazers_count, pushed_at, language, license: .license.spdx_id, archived, disabled, fork}'
   gh api repos/{owner}/{repo}/stats/commit_activity --jq '[.[] | .total] | {last_4w: .[-4:] | add, last_12w: .[-12:] | add}'
   ```

   (gh 不可用时 fallback curl,同第零步规则。)

3. **输出 5 行速览**:

   ```markdown
   ## 🪶 {owner}/{repo} 速览

   {如有风险:⚠️ 横幅一行,例:⚠️ 已归档 / ⚠️ AGPL license}

   - **定位**:{一句话}
   - **Stars / 主语言**:{X} ⭐ · {Language}
   - **活跃度**:{活跃 / 低活跃 / 僵尸}(近 4w: X · 近 12w: Y commit)
   - **License**:{协议 + 一句话翻译}
   - **一句话结论**:{对你有没有用,15 字内}
   ```

4. **用 AskUserQuestion 单选下一步**:

   | 选项 | 行为 |
   |-----|------|
   | 跑完整分析 | 进入第二步,复用上面的 Agent 1 数据,**不重跑** |
   | 直接给安装命令 | 跳到第七步给具体命令 |
   | 不操作 | 结束。**不写盘**(避免给低价值数据污染知识库) |

> **复用规则**:轻量入口跑过的 Agent 1 基本面数据,在升级到完整流程后,Agent 1 不再重跑,直接用速览结果作为输入。Agent 2 / 3 正常启动。

---

## 第二步:评估意图问询

用 AskUserQuestion(单选):

> 「你想拿这个项目干啥?」

| 选项 | 报告侧重 |
|------|---------|
| 🛠 想装来用 | 安装步骤、依赖、与现有工具冲突、上手难度 |
| 📚 想学架构 | 代码结构、文档质量、设计选择、可读性 |
| 💡 想抄思路 | 核心机制、关键算法、独特设计 |
| 🗺 看赛道 | 竞品对比、生态位、社区趋势 |

多项目场景下,如用户对所有项目意图一致,问一次即可;意图差异大时按项目分别问。

意图是后续所有判断的指南针——不同意图,启动哪些子 agent、报告详写哪节,都不一样。

---

## 第三步:历史报告检查

对每个待分析的项目,先查:

```bash
ls "/Users/sunshaocong/Desktop/MacBookpro备份内容/00_全局资产/github-analyses/" \
  2>/dev/null | grep -i "{owner}-{repo}"
```

**如有历史报告**:
1. read 最新一份
2. 给用户看上次「一句话结论 + 总分 + 日期」
3. 用 AskUserQuestion 单选:
   - 直接用上次结论
   - 重新分析(项目可能有变化)
   - 只跑差量(查上次之后的更新)

**如无历史**:进入第四步。

---

## 第四步:并行信息收集

启动 2-3 个子 Agent(`subagent_type: general-purpose`)并行收集。每个子 agent 的 prompt 完全自包含,末尾必须注入第零步的「用户环境摘要」。

### 子 Agent 1 — 项目基本面 + 风险体检

```markdown
## 任务
调研 GitHub 项目 {owner}/{repo} 的基本面 + 风险体检。

## 必查字段
1. 基本面 + 风险标志:
   gh api repos/{owner}/{repo} --jq '{name, description, stargazers_count, forks_count, open_issues_count, language, license: .license.spdx_id, created_at, pushed_at, topics, archived, disabled, fork, default_branch}'

2. 活跃度(过去 52 周周提交数,替代旧的"最近 5 commit"):
   gh api repos/{owner}/{repo}/stats/commit_activity --jq '[.[] | .total] | {last_4w: .[-4:] | add, last_12w: .[-12:] | add, last_52w: add}'

3. README:
   gh api repos/{owner}/{repo}/readme --jq '.content' | base64 -d

4. 贡献者:
   gh api repos/{owner}/{repo}/contributors?per_page=10 --jq '.[] | "\(.login): \(.contributions)"'

5. 安全公告(失败可忽略):
   gh api repos/{owner}/{repo}/security-advisories 2>/dev/null

如 gh 不可用,全部 fallback 到 curl(curl -s "https://api.github.com/repos/{owner}/{repo}/..." | jq ...)。

## 输出结构(每段 < 200 字)
- **项目名 / 定位**:一句话
- **⚠️ 风险横幅**(必给,无风险也要明说"无明显风险"):
  - archived / disabled → 「⚠️ 已归档/已弃用,谨慎使用」
  - license = AGPL-3.0 / GPL-3.0 → 「⚠️ 传染性 license,商用前先看条款」
  - license = null / Other → 「⚠️ 无明确 license,法律上默认禁止再分发」
  - 有公开 security advisories → 「⚠️ 有未修复安全公告」
  - fork=true 且原项目活跃 → 「⚠️ 这是 fork,建议看原项目」
- **Stars / Forks / 主语言**
- **活跃度判定**:活跃(过去 90 天 > 20 commit)/ 低活跃(1-20)/ 僵尸(0)。三个数字都给:近 4w / 近 12w / 近 52w
- **License**:协议名 + 一句话翻译(MIT=随便用 / Apache-2.0=随便用+保留专利条款 / GPL=改了要开源 / AGPL=连 SaaS 用都要开源 / 无 license=默认禁用)
- **核心功能**:3-8 个点
- **安装方式 + 依赖**:具体命令 + 前置条件
- **贡献者分布**:个人 / 小团队 / 大公司 / 社区(附人数)
- **README 摘要**:核心 200 字内

## 用户环境
{用户环境摘要}
```

### 子 Agent 2 — 社区与生态

```markdown
## 任务
调研项目 {owner}/{repo} 的社区评价、生态、竞品。

## 收集
1. WebSearch: "{repo_name} review" / "{repo_name} 怎么样" / "{repo_name} 替代品"
2. WebSearch: "{repo_name} site:reddit.com"
3. WebSearch: "{repo_name} site:v2ex.com OR site:zhihu.com"(中文项目尤其)
4. WebSearch: 同类竞品(至少找 2 个)
5. gh api repos/{owner}/{repo}/issues?state=open&per_page=5 --jq '.[] | "- " + .title'

## 输出(每段 < 200 字)
- **社区口碑**:正面 / 中性 / 负面,附 1-2 条典型评价原文
- **常见问题**(open issues 高频):2-3 个
- **适用场景**:最适合什么人、什么场景
- **不适用场景**:谁/什么场景不该用
- **竞品对比表**(至少 2 个):
  | 维度 | 本项目 | 竞品1 | 竞品2 |
  |------|--------|-------|-------|
  | Star | | | |
  | 最近更新 | | | |
  | 安装难度 | | | |
  | 差异点 | | | |
  | 适合谁 | | | |
- **独特优势**:相对竞品独一份的是什么

## 用户环境
{用户环境摘要}
```

### 子 Agent 3(可选)— 技术深度分析

**启动条件**(合并版,不再矛盾):
- 用户在第二步选了「📚 想学架构」或「💡 想抄思路」→ 必跑
- Star ≥ 5k → 默认跑
- Star < 5k 但用户主动要求 → 跑
- 其他 → 不跑,只在最终报告加一行「需要技术深度分析?告诉我」

```markdown
## 任务
对 {owner}/{repo} 做技术深度分析。

## 收集
1. gh api repos/{owner}/{repo}/contents/ --jq '.[] | "\(.type)\t\(.name)"' 看目录结构
2. 依赖文件(择一):package.json / requirements.txt / Cargo.toml / go.mod / pyproject.toml
3. gh api repos/{owner}/{repo}/contents/.github/workflows --jq '.[].name' 2>/dev/null 检查 CI
4. 主入口文件代码片段(看 README 找入口位置)

## 输出
- **架构复杂度**:简单 / 中等 / 复杂
- **代码质量信号**:有无测试 / CI / 文档质量
- **集成难度**:容易 / 中等 / 困难(安装步骤数 + 依赖数)
- **上手学习曲线**:几小时 / 几天 / 几周
- **二次开发友好度**:好 / 一般 / 差
- **API / MCP / 插件生态**:是否提供扩展点

## 用户环境
{用户环境摘要}
```

---

## 第五步:生成评估报告

等所有子 agent 返回后,综合输出。**报告内容按第二步选择的意图调整侧重**——别什么意图都铺一份大而全。

### 评分维度(满分 10 分)

| 维度 | 说明 |
|------|------|
| 需求相关度 | 与用户意图 + 现有工具的匹配 |
| 项目成熟度 | 活跃度 + 贡献者分散度 + Issue 处理质量 |
| 上手成本(反向) | 安装 + 学习曲线,越低分越高 |
| 现有体系互补性 | 与已装 skill / 工具的重叠度,互补越高分越高 |
| 长期维护信心 | 贡献者分散度 + 商业模式可持续性 |
| 用户视角风险(反向) | License / 安全 / 隐私 / 远程依赖,风险越低分越高 |
| **加权总分** | 六项均值 |

### 报告模板

```markdown
# {项目名} 分析报告

**日期**:{YYYY-MM-DD}
**意图**:{🛠 想装来用 / 📚 想学架构 / 💡 想抄思路 / 🗺 看赛道}

{如有风险:置顶大字号 ⚠️ 横幅,例:}
> ⚠️ 已归档,作者 2 年未维护
> ⚠️ License = AGPL-3.0,商用要小心
> ⚠️ 有未修复 security advisory CVE-XXXX

## 一句话结论
{15 字以内:对你有没有用、值不值得装/学/抄}

## 项目概览
- **定位**:{一句话}
- **Stars** / **Forks** / **语言**
- **活跃度**:{活跃 / 低活跃 / 僵尸} — 近 4w: X / 近 12w: Y / 近 52w: Z commit
- **License**:{协议 + 一句话翻译}

## 核心功能
1. ...

## 评估结果

### 有用程度:{非常高 / 高 / 中 / 低 / 无用}

### 综合评分
| 维度 | 分数 |
|------|------|
| 需求相关度 | /10 |
| 项目成熟度 | /10 |
| 上手成本(反向) | /10 |
| 现有体系互补性 | /10 |
| 长期维护信心 | /10 |
| 用户视角风险(反向) | /10 |
| **加权总分** | **/10** |

### 理由
- **亮点**:{2-3 点}
- **风险**:{1-2 点}
- **与你现有工具的关系**:{重叠了什么、互补了什么,引用具体 skill 名}

### 竞品对比
{Agent 2 的表}

## 按意图定制的深度分析
{根据第二步意图,详写对应一节,其他略写:
 - 🛠 想装 → 详写"如果安装" + 依赖/冲突 + 与已装工具的潜在冲突
 - 📚 想学 → 详写架构 + 推荐阅读顺序 + 关键文件位置
 - 💡 想抄 → 详写核心机制 + 关键算法 + 设计 trade-off
 - 🗺 看赛道 → 详写生态 + 趋势 + 竞品矩阵}

## 建议
**推荐 / 可选 / 不推荐** — {一句话原因}

### 如果安装
1. {具体命令}
2. {配置要点}
3. {注意事项}

### 如果不装
{替代方案或原因}
```

---

## 第六步:持久化

写报告到:
```
/Users/sunshaocong/Desktop/MacBookpro备份内容/00_全局资产/github-analyses/YYYY-MM-DD-{owner}-{repo}.md
```

目录不存在先 `mkdir -p`。

同步维护 INDEX 文件:
```
/Users/sunshaocong/Desktop/MacBookpro备份内容/00_全局资产/github-analyses/INDEX.md
```

格式:

```markdown
# GitHub 项目分析索引

| 日期 | 项目 | 意图 | 总分 | 建议 | 风险 | 一句话理由 |
|------|------|------|------|------|------|-----------|
| 2026-05-17 | owner/repo | 🛠 想装 | 8.2 | 推荐 | 无 | ... |
```

每次分析完追加一行(read INDEX → append → write back)。INDEX 不存在则先创建表头。

---

## 第七步:下一步问询(单选)

用 AskUserQuestion 单选(不要堆三个独立 yes/no):

> 「下一步要做什么?」

| 选项 | 行为 |
|------|------|
| 安装并集成 | 给出具体命令,引导用户跑 |
| 深入某方面 | 再用 AskUserQuestion 问要深入哪一面 |
| 分析下一个项目 | 进入新一轮 |
| 暂不操作,先收藏 | 报告已落盘,结束 |

---

## 第八步:汇总对比(多项目场景)

一次分析 ≥ 2 个项目,所有项目完成后额外输出:

```markdown
## 本次分析汇总

| 排名 | 项目 | ⭐ | 总分 | 建议 | 风险 | 一句话理由 |
|------|------|-----|------|------|------|-----------|
| 1 | xxx | xxk | 8.5 | 推荐 | 无 | ... |
| 2 | yyy | xxk | 7.2 | 可选 | AGPL | ... |

### 行动建议
- **立即安装**:总分 > 8 且无风险横幅
- **观察等待**:总分 6-8 或 alpha / preview 阶段
- **暂不考虑**:总分 < 6 或与现有体系高度重叠
- **谨慎**:有 archived / AGPL / 安全公告横幅
```

这条汇总也 append 到 INDEX.md。

---

## 注意事项

- **信息来源优先级**:gh api / curl 实时数据 > README > 社区搜索 > 模型已有知识。不要只靠记忆。
- **不回避负面**:archived / 维权争议 / 安全问题 / license 陷阱,放报告头部横幅,不能藏。
- **gh fallback**:gh 不可用时降级 curl(限速 60/h),并提示用户安装 + auth。
- **Star 少不等于差**:个人 skill / 小工具往往只有几十 star 但精准。意图匹配优先于 star 数。
- **报告必落盘**:每个项目分析完都写到 `00_全局资产/github-analyses/`,长期形成开源工具记忆库。
- **历史报告先查再做**:重复分析浪费时间,先看上次怎么说,再决定是否重做。
- **意图驱动报告侧重**:第二步选的意图决定报告详写哪节、略写哪节。
- **子 agent 输出要短**:每段 < 200 字,主对话 context 不爆。
- **截图先识别页面类型**:不是所有 GitHub 截图都是 repo 主页。
- **多项目 = 抽取 + 勾选 + 并行 + 汇总**:用户的高频场景是从一篇文章里筛工具,主链路要顺。
