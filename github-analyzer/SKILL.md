---
name: github-analyzer
description: |
  GitHub 开源项目分析与评估 Skill。当用户分享 GitHub 项目截图、链接、仓库名，或描述了一个开源项目的功能时触发。
  触发词包括但不限于：帮我看看这个项目、分析一下这个GitHub项目、这个开源项目怎么样、评估一下这个工具、
  这个项目值不值得装、帮我研究一下这个repo、这个GitHub仓库有用吗、这个开源工具怎么样、
  评测一下这个项目、帮我分析这个截图里的项目。
  当用户直接发来 GitHub 链接（github.com/xxx/xxx 格式）时也应触发。
  当用户发来包含 GitHub 页面的截图时也应触发（需先用视觉能力识别项目名称和关键信息）。
  不要用于纯代码审查（用户要你review自己的代码）、不要用于创建新项目（那是开发任务）、不要用于简单的搜索查询。
allowed-tools: Bash, Read, Write, WebSearch, Agent, AskUserQuestion
---

# GitHub 开源项目分析

你正在执行一次 GitHub 项目分析。目标是从用户分享的截图/链接/描述中识别出项目，然后给出一份**实用的评估报告**，帮助用户快速判断"这个项目对我有没有用、值不值得花时间"。

## 第一步：识别项目

用户可能通过以下方式提供信息：

1. **GitHub 链接**：直接粘贴 `github.com/owner/repo` 格式的 URL → 直接提取 owner 和 repo 名
2. **截图**：Twitter/微博/公众号等社交媒体截图，包含 GitHub 项目页面 → 用视觉能力识别：
   - 项目名（通常在页面标题位置）
   - Stars 数
   - 项目描述
   - README 片段
   - 关键功能列表
3. **文字描述**：用户口述"一个叫 XXX 的项目，可以YYY" → 提取关键词搜索

**如果信息不够确定**，用 AskUserQuestion 让用户确认项目名和 GitHub 地址。

## 第二步：并行信息收集

启动 2-3 个子 Agent 并行收集信息。每个子 Agent 的 prompt 必须完全自包含（子 agent 没有会话上下文）。

### 子 Agent 1 — 项目基本面

```python
prompt = f"""## 任务
调研 GitHub 项目 {owner}/{repo} 的基本面信息。

## 收集内容
1. 用 WebSearch 搜索项目名 + 关键词，了解项目定位和社区评价
2. 用 Bash 运行: curl -s "https://api.github.com/repos/{owner}/{repo}" 获取：
   - Stars、Forks、Issues 数量
   - 最近更新时间、创建时间
   - 主要编程语言
   - License 类型
   - 项目描述
3. 尝试读取 README 内容（curl -s "https://api.github.com/repos/{owner}/{repo}/readme" | python3 -c "import json,sys,base64; d=json.load(sys.stdin); print(base64.b64decode(d['content']).decode())" 2>/dev/null）
4. 检查最近的 commits 活跃度（curl -s "https://api.github.com/repos/{owner}/{repo}/commits?per_page=5"）

## 输出要求
用以下格式输出：
- **项目名**: 
- **定位**: 一句话说明这个项目是什么、解决什么问题
- **Star数**: 
- **Fork数**: 
- **最近更新**: 
- **活跃度评估**: 活跃/一般/不活跃
- **License**: 
- **主要语言**: 
- **核心功能列表**: 3-8 个关键功能点
- **安装方式**: 怎么装
- **依赖/系统要求**: 有什么前置条件
- **README 摘要**: README 的核心内容概括（300字以内）
"""
```

### 子 Agent 2 — 社区与生态

```python
prompt = f"""## 任务
调研 GitHub 项目 {owner}/{repo} 的社区评价和生态系统。

## 收集内容
1. 用 WebSearch 搜索 "{repo_name} review" / "{repo_name} 怎么样" / "{repo_name} 替代品"
2. 搜索 "{repo_name} site:reddit.com" 了解用户真实反馈
3. 搜索 "{repo_name} site:v2ex.com" 或 "{repo_name} site:zhihu.com"（如果是中文社区项目）
4. 用 WebSearch 搜索同类竞品："与 {repo_name} 类似的工具"
5. 用 Bash 运行: curl -s "https://api.github.com/repos/{owner}/{repo}/issues?state=open&per_page=5" 查看常见问题

## 输出要求
- **社区口碑**: 正面/中性/负面，附典型评价
- **常见问题**: Issues 中高频出现的 2-3 个问题
- **竞品对比**: 2-3 个同类工具，各自优劣
- **适用场景**: 这个项目最适合什么人、什么场景
- **不适用场景**: 什么人/场景不该用这个
- **中文社区讨论**: 如果有中文讨论，概括主要观点
"""
```

### 子 Agent 3（可选）— 技术深度分析

只在以下情况启动：
- 项目 Star > 10k（说明项目比较复杂）
- 用户明确要求技术评估
- 项目是框架/SDK/工具链类（需要评估集成难度）

```python
prompt = f"""## 任务
对 GitHub 项目 {owner}/{repo} 进行技术深度分析。

## 收集内容
1. 查看项目目录结构: curl -s "https://api.github.com/repos/{owner}/{repo}/contents/"
2. 查看 package.json / requirements.txt / Cargo.toml 等依赖文件
3. 检查是否有 CI/CD 配置（.github/workflows/）
4. 检查是否有测试覆盖
5. 查看贡献者数量: curl -s "https://api.github.com/repos/{owner}/{repo}/contributors?per_page=10"
6. 检查是否有文档站、API 文档

## 输出要求
- **架构复杂度**: 简单/中等/复杂
- **代码质量信号**: 有无测试、有无 CI、文档质量
- **集成难度**: 容易/中等/困难（安装步骤数、依赖数量）
- **二次开发友好度**: 好/一般/差
- **维护团队**: 个人项目/小团队/大公司/社区
- **API/MCP/插件生态**: 是否提供接口方便扩展
"""
```

## 第三步：生成评估报告

等所有子 Agent 返回结果后，综合生成报告。用中文输出。

### 报告格式

```markdown
# {项目名} 分析报告

## 一句话结论
{用 15 个字以内说明：这个项目对你有没有用}

## 项目概览
- **定位**：{一句话}
- **Stars**：{数字} | **Forks**：{数字} | **语言**：{主语言}
- **活跃度**：{活跃/一般/不活跃} — {上次更新时间}
- **License**：{协议类型}

## 核心功能
1. {功能1}
2. {功能2}
...

## 评估结果

### 有用程度：{非常高 / 高 / 中 / 低 / 无用}

### 理由
- **亮点**：{2-3 个值得关注的点}
- **风险**：{1-2 个需要注意的问题}
- **与你现有工具的互补性**：{和你已安装的 skill/工具 的关系}

### 竞品对比
| 维度 | {项目名} | {竞品1} | {竞品2} |
|------|----------|---------|---------|
| {维度1} | ... | ... | ... |

## 建议
**推荐 / 可选 / 不推荐** — {一句话原因}

### 如果安装
1. {安装步骤}
2. {配置要点}
3. {注意事项}

### 如果不装
{替代方案或原因}
```

## 第四步：用户确认

用 AskUserQuestion 问用户：
1. 是否需要安装推荐的项目？
2. 是否需要更深入地分析某个方面？
3. 是否有其他项目想一起分析？

## 注意事项

- **信息来源优先级**：GitHub API 数据 > README 内容 > 社区搜索 > 模型知识。优先用实时数据，不要只靠模型已有知识。
- **不要回避负面信息**：如果项目有安全问题、维权争议、已停止维护等，直接指出。
- **分析要有针对性**：结合用户的实际使用场景（Claude Code 用户、有 39 个 skill、做一人公司架构等），不要给出泛泛的评价。
- **控制篇幅**：单个项目分析报告控制在 500-800 字，重点突出可操作性。
- **多项目场景**：如果用户一次分享了多个项目（比如多张截图），先逐个识别，再统一分析，最后给出一个优先级排序表。
