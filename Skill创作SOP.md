# Claude Code Skill 创作 SOP

> 本文档基于对 research、research-deep、hv-analysis、dispatching-parallel-agents、skill-creator 等现有 skill 的逆向分析，以及对 17 个高星开源 Skill（184k~37k ⭐）的横向研究、加上 Anthropic 官方 Agent Skills 规范（https://agentskills.io/specification）整理而成。
> 创建日期：2026-05-16 | 最后更新：2026-05-17

---

## 一、Skill 是什么

Skill 就是一份 Markdown 操作手册，告诉 Claude "遇到某类任务时按这个流程执行"。

不需要写代码、不需要框架、不需要配置文件。核心就是一个文件夹 + 一个 `SKILL.md`。

### 文件结构

```
my-skill/
├── SKILL.md              ← 唯一必须文件
└── scripts/              ← 可选：辅助脚本（如 PDF 转换、数据验证）
└── references/           ← 可选：参考资料（按需加载，节省上下文）
└── assets/               ← 可选：模板、字体等静态资源
```

### SKILL.md 结构

```markdown
---
name: my-skill
description: 触发条件描述（最关键的字段，决定 Claude 何时使用这个 skill）
allowed-tools: Bash Read Write WebSearch Task
---

# Skill 标题

## 第一步：...
## 第二步：...
```

**YAML 头部字段**（来自官方 Agent Skills 规范）：

| 字段 | 必填 | 约束 |
|------|------|------|
| `name` | 是 | 1-64 字符；只允许小写字母、数字、短横线；不能以短横线起止；**必须跟父目录名一致** |
| `description` | 是 | 1-1024 字符；写清楚"什么时候用"和"不用来做什么" |
| `license` | 否 | 协议名或指向打包的 LICENSE 文件 |
| `compatibility` | 否 | 1-500 字符；声明环境要求（产品、系统依赖、网络、Python 版本等）。多数 skill 不需要 |
| `metadata` | 否 | 任意 string → string KV map，存非标准字段（如 author、version） |
| `allowed-tools` | 否 | **空格分隔字符串**（不是逗号），列预先批准的工具。实验性，各客户端实现差异 |

> ⚠️ `allowed-tools` 是空格分隔不是逗号分隔。例：`allowed-tools: Bash Read Write WebSearch` 而不是 `Bash, Read, Write`。可以带子作用域：`Bash(git:*) Bash(jq:*) Read`。
>
> ⚠️ 官方没有 `model` 字段。想给 skill 指定模型,只能塞 `metadata.model`,但 Claude Code 当前不一定识别。
>
> ⚠️ Claude.ai 上传自定义 skill 时 description 目前有更短限制（200 字符）；Claude Code / Agent Skills 开放 spec 是 1024 字符。写给 Claude Code 的本地 skill 按 1024 控制即可；准备上传 claude.ai 时再单独压缩。
>
> 验证 frontmatter 是否合规可用 `skills-ref validate ./my-skill`（来自 https://github.com/agentskills/agentskills/tree/main/skills-ref）。
> 如果用 `uvx` 临时运行,实际可执行名是 `agentskills`:
>
> ```bash
> uvx --from skills-ref agentskills validate ./my-skill
> ```

---

## 二、Description 是触发开关

`description` 字段决定了 Claude 在什么场景下会调用你的 skill。写法要点：

1. **写触发词**：列出用户可能会说的关键词和短语
2. **写适用场景**：什么情况下该用
3. **写排除场景**：什么情况下不该用
4. **写得"有点强势"**：Claude 倾向于少触发 skill，所以描述要主动一点

**好的 description 示例**（来自 hv-analysis）：

```yaml
description: |
  横纵分析法深度研究Skill。
  触发词包括但不限于：横纵分析、研究一下、帮我分析、深度研究、做个研究、
  调研一下、竞品分析、帮我看看这个东西怎么样。
  不要用于简单的名词解释、不要用于公众号写作、不要用于纯标题摘要生成。
```

**差的 description**：

```yaml
description: 研究工具
```

### 2.1 MUST 触发 / MUST 不触发 范式

「触发词清单 + 排除场景」是基础。但当你的 skill 跟相邻 skill 有边界争议时，普通"不要用于"语气不够强，需要用 **MUST 范式**明确告诉 Claude 谁优先。

#### MUST 触发（强信号）

适用场景：用户输入很弱（只有一个 URL / 一个名词 / 一个动词），Claude 可能犹豫不触发或被相邻 skill 抢走。

写法：

```yaml
description: |
  [基础说明...]

  **MUST 触发**：当用户消息只包含 [极弱信号，如裸 URL / 一个名词] 时，
  默认按 [轻量入口或完整流程] 启动本 skill。
  **不要 undertrigger**——用户给你这种输入而你只回"想让我做什么?"，
  等于工具白浪费，这就是用户装这个 skill 的本意。
```

实战案例：`github-analyzer` 的裸 GitHub 链接、`aihot` 的"AI 圈"/"今天发生了什么"。

#### MUST 不触发（让位场景）

适用场景：你的 skill 跟另一个 skill 有重叠风险，需要明确划清谁让位。

写法：

```yaml
description: |
  [基础说明...]

  **MUST 不触发**（让位给更专门的 skill）：
  - 场景 A 出现 → 让位给 skill-X（原因：它有 Y 能力）
  - 场景 B 出现 → 让位给 skill-Y
```

实战案例：`idea-to-prompt` 让位给 `brainstorming`（git repo + 软件 feature 改动）、让位给 `project-init`（cwd 已在项目里）。

#### 何时该用 MUST 范式

| 信号 | 用哪种 |
|------|--------|
| 你的 skill 因输入太弱常被错过 | MUST 触发 |
| 你的 skill 常跟另一 skill 撞触发 | MUST 不触发（双向都加，两个 skill 各自的 description 里互相让位） |
| 单独 skill 没竞争 | 不用，普通触发词清单 + 排除项就够 |

⚠️ MUST 范式有副作用：会让 Claude 更确信地触发或让位。**只在边界确实有冲突时用**，滥用会让 Claude 变得僵硬。

---

## 三、Skill 正文写法原则

### 3.1 用祈使句，不用必须句

```
# 差
你 MUST 使用 WebSearch 搜索
你 SHALL NOT 编造信息

# 好
用 WebSearch 搜索，模型内部知识不够用
搜不到的信息标注「暂缺」，不编造
```

### 3.2 解释为什么，不只是规定怎么做

```markdown
# 差
每个字段值必须用中文。

# 好
所有字段值使用中文输出。调研过程可以用英文搜索，但最终输出给用户的内容必须是中文，
因为这份报告的目标读者是中文用户。
```

### 3.3 给示例，但示例要有代表性

在 prompt 模板旁边放一个 one-shot 示例，让 Claude 理解预期输出格式。

### 3.4 控制篇幅

官方推荐 SKILL.md **正文 < 500 行 / < 5000 tokens**（超出影响 token 占用和响应速度）。
**< 200 行是理想值**：加载快、容易整体看清。

超过 500 行的处理：
- 详细 prompt 模板、长 reference 移到 `references/` 目录，运行时按需 `Read`
- 大段示例移到 `examples/` 或 `assets/`
- 多场景的逻辑分支拆成独立 skill（走 9.4 拆分规则）

9.7 节会展开"如何省 token"的实战技巧。

---

## 四、多 Agent 并发模式（核心）

这是 skill 最强大的能力。通过在同一个对话轮次里发起多个 Agent 调用，实现并行执行。

### 4.1 并发的本质

没有特殊框架。就是在 SKILL.md 里告诉 Claude："同一轮次发起多个 Agent 调用"。

当 Claude 在一条回复里同时调用多个 Agent tool 时，它们自动并行执行。主会话等待所有结果返回后再继续。

### 4.2 三种并发模式

#### 模式 A：单 Agent 外包

适合：简单的一块独立工作。

```markdown
## Step 2: Web Search 补充

启动1个 web-search-agent（后台），使用以下 prompt：
[粘贴 prompt 模板]
```

**案例**：`research` skill 只派 1 个 agent 做联网搜索补充。

#### 模式 B：分批并发 + 断点续传

适合：任务量大（10+ 个对象），需要分批控制。

```markdown
## Step 3: 分批执行

- 按 batch_size 分批（完成一批需要得到用户同意才可进行下一批）
- 每个 agent 负责 items_per_agent 个项目
- 启动 web-search-agent（后台并行，禁用 task output）

## Step 2: 断点续传检查
- 检查 output_dir 下已完成的 JSON 文件
- 跳过已完成的 items
```

**案例**：`research-deep` skill 的完整实现。

#### 模式 C：角色分工型并发

适合：同一任务需要不同视角/维度的信息。

```markdown
### 并行搜索策略

使用子 Agent 并行搜索来提高效率：

- **子Agent 1 — 纵向信息**：起源、发展历程、关键事件...
- **子Agent 2 — 横向信息**：竞品识别、行业对比、市场份额...
- **子Agent 3**（复杂对象才需要）：补充信息...
```

**案例**：`hv-analysis` skill 的横纵双轴信息收集。

### 4.3 子 Agent 的 Prompt 怎么写

子 agent 没有会话上下文，prompt 必须完全自包含。

**关键要素：**

| 要素 | 说明 |
|------|------|
| 任务描述 | 做什么、输出什么 |
| 上下文 | 所有需要的信息直接塞进 prompt |
| 输出要求 | 格式、语言、文件路径 |
| 约束 | 不能做什么 |
| 验证 | 如何自检完成质量 |

**模板：**

```python
prompt = f"""## 任务
[具体任务描述]

## 背景
[所有需要的上下文信息]

## 输出要求
1. [格式要求]
2. [质量要求]
3. [路径要求]

## 约束
- [不能做的事]
- [必须遵守的规则]

## 验证
[完成后如何自检]
"""
```

**实际案例**（来自 research-deep）：

```python
prompt = f"""## 任务
调研 {item_related_info}，输出结构化JSON到 {output_path}

## 字段定义
读取 {fields_path} 获取所有字段定义

## 输出要求
1. 按 fields.yaml 定义的字段输出 JSON
2. 不确定的字段值标注 [不确定]
3. JSON 末尾添加 uncertain 数组，列出所有不确定的字段名
4. 所有字段值必须使用中文输出

## 输出路径
{output_path}

## 验证
完成 JSON 输出后，运行验证脚本：
python ~/.claude/skills/research/validate_json.py -f {fields_path} -j {output_path}
"""
```

### 4.4 主 Agent 的监督逻辑

主 agent（就是用户对话的 Claude）负责：

1. **拆任务**：把大任务分解成子 agent 的独立工作包
2. **派发**：同一轮次发起多个 Agent 调用
3. **等结果**：等所有子 agent 返回
4. **检查**：验证结果质量
5. **决策**：继续下一批 / 重新搜索 / 合并输出
6. **汇报**：向用户展示汇总结果

这些逻辑全部在 SKILL.md 里用自然语言规定，不需要写代码。

---

## 五、创作 SOP（8 步流程）

### Step 0：前置判断——要不要新建 Skill

动手写之前先过三道筛：

| 筛子 | 通过标准 | 不通过怎么办 |
|------|---------|-------------|
| 边界清晰 | 场景有明确的起止点，不会和别的事混在一起 | 归入已有 skill 的内部分支 |
| 高频复现 | 这个场景你会反复遇到，不是一次性的 | 直接在对话里手操，不封装 |
| 无法归属 | 现有 skill 确实覆盖不了这个场景 | 先看能不能扩展已有 skill 的分支逻辑 |

**分类学原则**：不是分得越细越好，是找到最合适的颗粒度。

- 封面图和PPT配图的差异，不值得各自独立占一个 skill，它们只是"图片生成"类别内的变异
- 图片生成和服务器管理的差异，才大到需要各自独立 skill
- 如果新需求只是已有 skill 的一个场景变体，用内部分支处理（skill 内部根据上下文二次判断走哪个子流程），不要新建 skill

### Step 1：明确 Skill 定位

回答这四个问题：

| 问题 | 示例答案 |
|------|---------|
| 这个 skill 做什么？ | 对目标话题进行横纵分析深度研究 |
| 什么时候触发？ | 用户说"研究一下""帮我分析""竞品分析" |
| 输入是什么？ | 一个研究对象名称（产品/公司/概念/人物） |
| 输出是什么？ | 一份 10000-30000 字的 PDF 研究报告 |

### Step 2：设计执行流程

画出步骤，每步标明：
- 谁做（主 agent / 子 agent / 用户确认）
- 用什么工具
- 输入输出是什么

示例：

```
Step 1 [主agent] 确认研究对象 → AskUserQuestion
Step 2 [3个子agent并行] 信息收集 → Agent tool + WebSearch
Step 3 [主agent] 检查信息充分性 → 不够就补搜
Step 4 [主agent] 写纵向分析 → Write
Step 5 [主agent] 写横向分析 → Write
Step 6 [主agent] 写交汇洞察 → Write
Step 7 [主agent] 生成PDF → Bash + scripts/md_to_pdf.py
```

### Step 3：写 SKILL.md 草稿

按以下顺序写：

1. **YAML 头部**：name + description（先写粗版，后面优化）
2. **前置准备**：环境依赖、输入确认
3. **执行步骤**：每步写清楚做什么、怎么做
4. **子 agent 部分**：写 prompt 模板（如果是多 agent skill）
5. **输出格式**：最终产出长什么样
6. **质检清单**：交付前自检项

### Step 4：写 2-3 个测试用例

模拟真实用户会说的话来测试：

```
测试1："帮我研究一下 Cursor 这个编辑器"
测试2："我想了解一下 RAG 技术是怎么回事"
测试3："做一个 Notion vs Obsidian 的竞品分析"
```

### Step 4.5：触发 audit（双向）

写完 description 后，**别急着开测**——先做触发审计。这一步在实操中是 ROI 最高的环节，能避免后续大量返工。

列两份清单：

**should-trigger 清单（5-10 条）** — 用户最可能说的话，全部跑过：
- 直接命名："研究一下 Cursor"
- 弱信号："看看这个 https://github.com/xxx/yyy"
- 自然语言："这个项目咋样"
- 隐含意图："我想了解 RAG 是怎么回事"
- 中文 / 英文混说："Cursor vs Copilot review"

**should-NOT-trigger 清单（与相邻 skill 撞触发的场景）**：
- 邻居 skill 的典型触发词，确认本 skill 不响应
- 闲聊式表达（"我有个想法"在闲聊时不应该被拦）
- 过窄的需求（"写个 hello world" 不该被复杂 skill 拦）
- 已有 spec 的执行任务（应该让位给 writing-plans）

对每条 should-NOT-trigger，**在 description 里加一条排除或 MUST 不触发**（见 2.1）。

跟现有所有 skill 的 description 跑一遍对比，标出重叠/边界模糊处，互相让位。

### Step 5：测试 → 修改 → 重复

1. 用测试用例跑一遍 skill
2. 看输出质量，记录问题
3. 修改 SKILL.md
4. 重新测试
5. 重复直到满意

常见问题及修复：

| 问题 | 修复方向 |
|------|---------|
| Skill 没被触发 | description 写得不够具体或不够"强势"；考虑加 MUST 触发 |
| 经常错触发到本 skill | description 缺排除项；考虑加 MUST 不触发让位给真正合适的 skill |
| 输出格式不对 | 加强输出模板的示例 |
| 子 agent 跑偏了 | prompt 里加更多约束和上下文 |
| 信息不够充分 | 加"信息充分性自检"步骤 |
| 步骤顺序错了 | 调整执行流程，加用户确认节点 |
| 子 agent 失败 / 返回空 | 加容错层（见 9.1），主 agent 用 WebSearch 补 |

### Step 5.5：交付前自检 checklist

跑过测试不等于可以发布。发布前对照下面这份 checklist，逐项打勾：

**Frontmatter 合规**
- [ ] `name` 跟父目录名一致，全小写 + 短横线，长度 ≤ 64
- [ ] `description` 长度 ≤ 1024 字符
- [ ] `allowed-tools`（如有）用**空格分隔**，不是逗号
- [ ] 没有用 `model` 这个非标字段（如必须指定模型，塞 `metadata.model` 并自行测试客户端识别）

**Description 质量**
- [ ] 有触发词清单（5+ 条具体用户表达）
- [ ] 有"不要用于"排除项
- [ ] 与相邻 skill 边界划清（如有冲突，加 MUST 触发 / MUST 不触发）
- [ ] 跑过 Step 4.5 双向触发 audit

**正文质量**
- [ ] 输出格式有 one-shot 示例
- [ ] 多 agent 时，子 agent prompt 自包含 + 注入用户环境摘要
- [ ] 失败容错策略写明（9.1）
- [ ] 篇幅 ≤ 500 行（理想 ≤ 200）；长 reference 移到 `references/`

**生态健康**
- [ ] 跟全局 skill 总数对照（见 9.9），确认本 skill 值得新建，而非扩展已有
- [ ] 跟管道上下游的 skill 接口顺畅（见 9.3）

**可选验证**
- [ ] 用 `skills-ref validate ./my-skill` 跑过官方 schema 验证

### Step 6：安装和使用

Claude Code 的可用位置：

```
~/.claude/skills/my-skill/SKILL.md      ← 全局可用
项目目录/.claude/skills/my-skill/SKILL.md  ← 仅当前项目可用
```

Codex 的可用位置：

```
~/.agents/skills/my-skill/SKILL.md      ← Codex 可见
```

你自己的 skill 不直接在安装目录里长期维护。**先写 `your-skills-repo/{name}/` 仓库版，再同步到 Claude Code / Codex。**

新建并安装一个 skill 的最小命令：

```bash
cd "~/path/to/shaocong-skills/"
mkdir -p my-skill
$EDITOR my-skill/SKILL.md
$EDITOR my-skill/README.md

rsync -a --delete my-skill/ "$HOME/.claude/skills/my-skill/"
rsync -a --delete my-skill/ "$HOME/.agents/skills/my-skill/"
```

放好后，新对话中 Claude / Codex 会自动识别。Claude Code 也可以用 `/skill-name` 手动触发。

### Step 7：修改、同步、push

修改 skill 后的完整流程参考第十一章「Skill 仓库管理 / git 工作流」。要点：
- **主版本在 `your-skills-repo/{name}/`**，这是日常主要修改位置和 git 工作树
- 用户说"更新这个 skill"时，默认含义是：仓库版、Codex 版、Claude Code 版三处都更新
- 从仓库版用 `rsync` 同步到 `~/.agents/skills/{name}/` 和 `~/.claude/skills/{name}/`
- 同步后分别跑仓库版、Codex 版、Claude Code 版验证
- `git add → commit → push` 一气呵成同步到 GitHub

详细命令和 .gitignore 配置见第十一章。

---

## 六、案例拆解

### 案例 1：research（简单流 · 单 agent 外包）

**定位**：对目标话题生成初步调研框架。

**流程**：
```
Step 1 [主agent] 用模型知识生成 items 列表 + 字段框架
Step 2 [1个子agent] 联网搜索补充（后台执行）
Step 3 [主agent] 问用户有没有已有字段文件
Step 4 [主agent] 合并所有信息，生成 outline.yaml + fields.yaml
```

**关键设计**：
- 子 agent 的 prompt 是"硬约束"——必须原样复述模板，只替换变量
- 用 AskUserQuestion 在关键节点让用户确认
- 输出结构化文件（yaml），方便下游 skill 消费

**文件**：`research/SKILL.md`（共 146 行）

---

### 案例 2：research-deep（复杂流 · 分批并发 + 断点续传）

**定位**：按 outline.yaml 逐项深度调研，每个 item 启动独立 agent。

**流程**：
```
Step 1 [主agent] 读取 outline.yaml，获取 items 列表和 execution 配置
Step 2 [主agent] 断点续传检查，跳过已完成的 JSON
Step 3 [主agent] 按 batch_size 分批派发子 agent（并行）
         → 每个子 agent 调研 1-N 个 items，输出 JSON
         → 每个子 agent 完成后自跑验证脚本
Step 4 [主agent] 等待当前批次 → 用户确认 → 下一批
Step 5 [主agent] 全部完成后汇总报告
```

**关键设计**：
- **分批控制**：batch_size 控制并发数，防止一次性派发太多
- **断点续传**：启动时检查 output_dir，跳过已有 JSON 文件
- **验证脚本**：子 agent 写完 JSON 后自跑 `validate_json.py` 确保字段覆盖完整
- **禁用 Task Output**：子 agent 的结果写文件，主 agent 不需要读 stdout
- **不确定标记**：子 agent 对不确定的字段标注 `[不确定]`，留后续补充

**文件**：`research-deep/SKILL.md`（共 101 行）

---

### 案例 3：hv-analysis（角色分工型 · 2-3 个 agent 并行收集）

**定位**：对产品/公司/概念/人物进行横纵分析，输出 PDF 研究报告。

**流程**：
```
Step 1 [2-3个子agent并行] 信息收集
         → Agent 1: 纵轴信息（起源、发展历程、关键事件）
         → Agent 2: 横轴信息（竞品、行业对比、市场份额）
         → Agent 3（可选）: 补充信息
Step 2 [主agent] 纵向分析写作（6000-15000字）
Step 3 [主agent] 横向分析写作（3000-10000字）
Step 4 [主agent] 横纵交汇洞察（1500-3000字）
Step 5 [主agent] 生成 PDF（调用 scripts/md_to_pdf.py）
```

**关键设计**：
- **角色分工**：不是按对象分（一人搞一个产品），而是按维度分（一人搞纵轴、一人搞横轴）
- **联网工具指南**：每个子 agent 的 prompt 里内置联网操作指引（WebSearch + WebFetch + arxiv API）
- **信息充分性自检**：子 agent 返回后主 agent 检查是否够用，不够就补搜
- **写作风格内置**：SKILL.md 里直接定义了完整的写作风格指南，不需要加载外部 skill
- **辅助脚本**：自带 `scripts/md_to_pdf.py`（WeasyPrint）和 `references/schema.json`

**文件**：`hv-analysis/SKILL.md`（共 324 行）

---

### 案例 4：github-analyzer v2（角色分工型 · 量化评分 · 批量汇总）

**定位**：GitHub 开源项目分析与评估，帮助快速判断"对我有没有用、值不值得装"。

**流程**：
```
Step 0 [主agent] 加载用户上下文（ls ~/.claude/skills/）
Step 1 [主agent] 识别项目（链接/截图/文字描述）
Step 2 [2-3个子agent并行] 信息收集
         → Agent 1: 项目基本面（Star/Fork/活跃度/功能/安装方式）
         → Agent 2: 社区评价（口碑/竞品对比/适用场景）
         → Agent 3（可选）: 技术深度（架构/CI/贡献者/扩展性）
Step 3 [主agent] 生成评估报告（含量化评分 + 竞品对比表）
Step 4 [主agent] 用户确认（是否安装/是否深入/是否还有其他项目）
Step 5（3+项目时） [主agent] 汇总排序表
```

**关键设计**：
- **用户环境感知**：启动子 agent 前先读取已安装的 skill/agent 列表，注入环境摘要到每个子 agent prompt，使互补性分析有据可依
- **gh api 替代 curl**：已认证 5000 次/小时 vs 未认证 60 次/小时
- **量化评分**：5 维（需求相关度/成熟度/安装便捷度/互补性/维护信心），多项目可横向对比
- **结构化竞品对比**：强制表格输出（Star/安装难度/核心差异/适合谁）
- **批量汇总排序**：3+ 项目时自动生成排序表 + 行动建议（立即安装/观察等待/暂不考虑）
- **低星项目友好**：子 Agent 3 阈值从 10k 降到 5k，用户关注的项目无论 star 数都深入分析

**文件**：`github-analyzer/SKILL.md`（v2）

---

## 七、常用设计模式速查

| 需求 | 写法 |
|------|------|
| 并发执行 | "在同一轮次发起多个 Agent 调用" |
| 子 agent 隔离 | "子 agent 不继承会话上下文，prompt 必须自包含" |
| 批次控制 | "每批 N 个，完成一批经用户确认再继续" |
| 断点续传 | "启动时检查输出目录已有文件，跳过已完成的" |
| 结果验证 | "完成后运行验证脚本确认输出完整" |
| 不确定标记 | "不确定的值标注 [不确定]" |
| 用户确认节点 | 用 `AskUserQuestion` 在关键步骤暂停 |
| 后台执行 | "在后台启动 agent，不阻塞主流程" |
| 信息补搜 | "检查信息充分性，不够就补搜" |
| 输出文件 | "将结果写入 {output_path}" |
| 角色分工 | "你是基本面分析师" / "你是社区调研员" / "你是技术审计员" |
| 用户环境感知 | "先 ls ~/.claude/skills/ 获取已装 skill，注入子 agent prompt" |
| 量化评分 | "5 维评分（/10），输出汇总排序表" |
| 置信度标注 | "搜索到的事实标来源，推断标 [推断]，搜不到标暂缺" |
| 新鲜上下文 | "子 agent 加'你在全新上下文中工作'" |
| 交互式需求 | "先用 AskUserQuestion 确认方向，再开始执行" |
| 轻量入口 / 重量入口分流 | 重 skill 给一个 5 行速览作软触发，用户决定要不要升级到完整流程 |
| 检测前序产物自动接力 | 启动时 ls 上游 skill 的输出文件（如 `Prompt-*.md`），有就只问差量 |
| 冷启动模拟当交付门槛 | 自己扮演 fresh agent 只读关键文件答三问，任一答不上不交付 |

### 7.1 新增 3 个模式展开

**轻量入口 / 重量入口分流** — 当 skill 是重量级（多 agent 并行 + 写盘 + 报告生成），但触发条件可能很弱（用户只发一个 URL）时，先弹一个 30 秒速览（只跑最小子集，不写盘），让用户决定要不要升级到完整流程。降低误触发的代价，反过来让 description 敢写 MUST 触发。
案例：`github-analyzer` 的「第一步.五」。

**检测前序产物自动接力** — 多 skill 形成管道时，下游 skill 启动时主动扫 cwd 或常用位置（如 `~/Desktop/Prompt-*.md`），找到上游 skill 的输出就直接 read，只问差量字段。这样用户不用手动「这是上次的 prompt 文件」，下游 skill 自己识别。
案例：`project-init` 检测 `Prompt-*.md`，进入"新建+接力"模式。

**冷启动模拟当交付门槛** — 写完产物（如项目骨架、配置文件）后，**不要直接说"完成"**。先自己扮演一个 fresh agent，只读这份产物，试图回答 2-3 个核心问题（这是什么？下一步该做啥？怎么验证？）。任一答不上，回去补字段。这一步是 context engineering 的 acceptance test，比检查文件存在性重要 10 倍。
案例：`project-init` 的「冷启动模拟」步骤。

---

## 八、高星 Skill 设计模式（从 17 个万星项目提炼）

> 分析了 everything-claude-code(184k)、system-prompts(137k)、karpathy-skills(132k)、claude-code 官方(124k)、gstack(98k)、claude-mem(76k)、cc-switch(72k)、get-shit-done(62k)、caveman(61k)、learn-claude-code(60k)、claude-code-best-practice(53k)、ruflo(51k)、graphify(48k)、career-ops(45k)、awesome-claude-code(43k)、open-design(42k)、antigravity-awesome-skills(37k) 共 17 个项目。

### 8.1 高星 Skill 的 7 个共同特征

| 特征 | 说明 | 代表项目 |
|------|------|----------|
| **一个核心卖点** | 不做大而全，只解决一个痛点做到极致 | caveman（只做 token 压缩）、graphify（只做知识图谱） |
| **一条命令安装** | `npx xxx` 或 `curl \| bash` 或复制一个文件，三步以内完成 | 几乎所有高星项目 |
| **即时可见的输出** | 用户跑一次就能看到效果（HTML 报告、进度条、评分表） | graphify 的 `graph.html`、open-design 的沙盒预览 |
| **跨平台兼容** | 支持 Claude Code / Codex / Cursor / Gemini CLI 等多个 agent | caveman(30+)、open-design(16)、get-shit-done(15) |
| **渐进式复杂度** | 提供分档体验：core / standard / full，新手用核心版即可 | get-shit-done(3档)、caveman(4级压缩) |
| **Hook 自动化** | 不靠用户手动触发，用 Hook 在 SessionStart/End 自动执行 | claude-mem(5个 Hook)、everything-claude-code(session 持久化) |
| **置信度标注** | 区分"确信的信息"和"推断的信息"，增强可信度 | graphify(EXTRACTED/INFERRED/AMBIGUOUS) |

### 8.2 高星 Skill 的写作技巧

#### 技巧 1：用角色分工代替步骤堆叠

不要写一个巨长的顺序流程，而是把不同角色分配给不同子 agent。

**来自 gstack (98k ⭐)**：
```
/office-hours  → 6 个强迫性问题，重新定义产品
/plan-ceo-review → CEO 视角评审
/plan-eng-review → 工程经理视角评审
/plan-design-review → 设计师视角评审
/review → Staff Engineer 代码审查
/qa → QA Lead 端到端测试
/ship → Release Engineer 发布
```

每个角色有独立的 SKILL.md，但共同组成一个完整 Sprint 流程。

**怎么用在你的 skill 里**：
- 多 agent skill 不要让每个 agent "什么都查一点"
- 而是让每个 agent 有明确角色："你是基本面分析师"、"你是社区调研员"、"你是技术审计员"
- 在子 agent prompt 开头写明角色身份

#### 技巧 2：用新鲜上下文解决 Context Rot

长会话中 Claude 的上下文会"腐烂"，输出质量下降。

**来自 get-shit-done (62k ⭐)**：
- 重量级工作放在独立 subagent 的 200k token 新鲜上下文中执行
- 主窗口保持 30-40% 占用
- 五层结构化制品跨 session 保持状态：PROJECT.md → REQUIREMENTS.md → ROADMAP.md → STATE.md → CONTEXT.md

**怎么用在你的 skill 里**：
- 子 agent 的 prompt 里加上"你在全新的上下文中工作，不要假设已有任何历史信息"
- 复杂分析任务强制走 subagent，不要在主窗口堆积
- 关键产出写入文件持久化，不依赖上下文记忆

#### 技巧 3：交互式需求收集（先问再干）

不要让 Claude 立刻开始输出，先锁定需求。

**来自 open-design (42k ⭐)**：
生成前先锁定：受众 / 调性 / 品牌 / 尺寸，避免 AI 盲目产出后反复修改。

**怎么用在你的 skill 里**：
- 在第一步和第二步之间加一个 AskUserQuestion 确认节点
- 让用户确认分析方向："你主要关心这个项目的哪方面？实用性/技术架构/社区口碑/安装难度？"
- 根据用户回答调整子 agent 的调查重点

#### 技巧 4：评分量化而非纯定性

"非常好/好/一般/差"难以横向比较。

**来自 github-analyzer v2（你自己的 skill）**：
```
| 维度 | 分数 |
|------|------|
| 需求相关度 | /10 |
| 项目成熟度 | /10 |
| 安装便捷度 | /10 |
| 现有体系互补性 | /10 |
| 长期维护信心 | /10 |
| **加权总分** | **/10** |
```

**怎么用在你的 skill 里**：
- 任何需要"对比多个选项"的场景都加量化评分
- 评分维度要和用户决策直接相关（不是泛泛的"代码质量"）
- 多选项场景必须输出汇总排序表

#### 技巧 5：单文件极致主义

最高星的项目往往核心内容就是一个文件。

**来自 caveman (61k ⭐)**：
整个 skill 的核心就是一个 SKILL.md，4 级压缩模式（lite/full/ultra/wenyan），安装就是一行命令。

**来自 karpathy-skills (132k ⭐)**：
整个仓库 20KB，核心就是一个 50 行的 CLAUDE.md。

**怎么用在你的 skill 里**：
- SKILL.md 控制在 500 行以内
- 辅助脚本放 scripts/，参考资料放 references/，按需加载
- 安装方式设计为"复制一个文件就能用"

#### 技巧 6：给输出加置信度标签

区分"从数据中确认的"和"推断的"信息。

**来自 graphify (48k ⭐)**：
```
EXTRACTED  → 从代码/文档中直接提取的事实
INFERRED   → 基于上下文推断的结论
AMBIGUOUS  → 信息矛盾或不明确
```

**怎么用在你的 skill 里**：
- 子 agent 搜索不到的信息标注"暂缺"
- 推断性结论标注"[推断]"
- 数据来源明确标注（GitHub API / README / 社区搜索 / 模型知识）

#### 技巧 7：设计成可独立安装的原子模块

**来自 claude-code 官方 (124k ⭐)**：
13 个官方插件，每个独立安装，不互相依赖：
```
/plugin install code-review     ← 只装代码审查
/plugin install feature-dev     ← 只装功能开发
/plugin install hookify         ← 只装 hook 生成
```

**来自 antigravity-awesome-skills (37k ⭐)**：
1460+ 个 skill，每个都是独立文件夹，按需取用。

**怎么用在你的 skill 里**：
- 你的 skill 之间不要互相依赖（research 不依赖 research-deep）
- 每个 skill 独立完成一个闭环任务
- 如果 skill 太大，拆成多个（如 research + research-deep + research-report）

### 8.3 高星 Skill 的反模式（避免踩坑）

从 17 个项目中也看到了不少问题，记录下来避免重犯：

| 反模式 | 问题 | 案例 | 你的 skill 如何避免 |
|--------|------|------|---------------------|
| **功能膨胀** | v1→v13 不到一年，功能越来越多但稳定性下降 | claude-mem（v1→v13）、ruflo（alpha.44） | 每个 skill 只做一件事，不加"顺便也做"的功能 |
| **侵入式安装** | 要求改 CLAUDE.md、修改全局配置、安装大量依赖 | gstack（改 CLAUDE.md 路由）、ruflo（Rust+WASM+MongoDB） | 安装就是复制一个文件，不修改用户现有配置 |
| **过度营销** | 术语堆砌、宣传效果无法验证、README 比代码长 | ruflo（SONA/RuVector/Cognitum）、GSD（810x 生产力提升） | README 写清楚做什么、怎么用，不吹嘘效果 |
| **单点维护** | 98%+ 的 commits 来自一人，项目健康度依赖个人 | 几乎所有项目都是单人主导 | skill 设计成模块化，社区可以独立使用和改进 |

### 8.4 偷师方法论：如何从别人的 Skill 中学习并改造

偷师不是复制粘贴，是"读懂设计意图 → 提炼模式 → 用自己的语言重写"。

#### 第一步：找到目标 Skill 的源码

```bash
# 用 gh api 直接读取别人的 SKILL.md
gh api repos/{owner}/{repo}/contents/SKILL.md --jq '.content' | base64 -d

# 如果是插件形式（如 claude-code 官方插件）
gh api repos/anthropics/claude-code/contents/plugins/{plugin-name} --jq '.[].name'
gh api repos/anthropics/claude-code/contents/plugins/{plugin-name}/SKILL.md --jq '.content' | base64 -d
```

#### 第二步：读源码时问自己 4 个问题

| 问题 | 关注什么 |
|------|----------|
| 它解决了什么痛点？ | 一句话总结核心价值 |
| 它的执行流程是怎样的？ | 画出来：Step1→Step2→Step3，标注哪些用子 agent |
| 它的输出格式怎么设计的？ | 表格？JSON？Markdown？为什么选这个格式？ |
| 有什么我没想到的边界处理？ | 空结果、失败重试、用户确认、信息不充分时的策略 |

#### 第三步：提炼设计模式（不是复制代码）

把你读到的内容浓缩成一个设计模式，用一句话描述：

```
# 示例：从 gstack 的 /qa 提炼
"在子 agent 中启动真实浏览器（Playwright），
 对前端页面做端到端测试，自动截图对比，
 发现问题自动修复并生成回归测试。"
```

这个描述不包含任何代码，只有设计意图。下一步才是把意图变成你自己的实现。

#### 第四步：用自己的体系重写

根据你的实际情况决定：

- **融入已有 skill**：把学到的小技巧加到现有 SKILL.md 的某个步骤中
  - 例：在 github-analyzer 的子 agent prompt 里加"你在全新上下文中工作"（来自 get-shit-done）
  - 例：在 research-deep 的输出格式里加置信度标签（来自 graphify）
  - 只改几行，不动整体结构

- **新建独立 skill**：学到的是一个完整的新能力，不是小技巧
  - 例：从 gstack 的 /qa 学到 Playwright 浏览器测试 → 新建 browser-testing skill
  - 按本 SOP 的"五、创作 SOP"完整走一遍

- **只记不吃**：有些设计模式值得记住但暂时用不上
  - 例：career-ops 的 mode 模块化 → 记在本节清单里，等你的 skill 复杂到需要拆分时再用

#### 第五步：测试验证

修改后的 skill 跑一个测试用例，对比修改前后的输出质量。如果没提升就回滚，不强行嫁接。

#### 当前偷师清单

以下是从 17 个项目中提取的、可以逐步应用的设计思路：

| 偷师来源 | 偷什么 | 应用方式 | 状态 |
|----------|--------|----------|------|
| gstack | 真实浏览器测试（Playwright） | 新建 browser-testing skill | 待做 |
| get-shit-done | subagent 新鲜上下文执行 | github-analyzer 子 agent prompt 加提示 | 已做 |
| get-shit-done | STATE.md 跨 session 状态持久化 | 引入项目开发日志体系 | 待评估 |
| caveman | 4 级输出压缩 + wenyan 模式 | 直接装 caveman，或改造为自己的 skill | 待做 |
| learn-claude-code | Skill Loading 按需加载 | 优化 39 个 skill 的体积 | 待做 |
| open-design | 生成前交互式需求表单 | github-analyzer 已用 | 已做 |
| graphify | 置信度标签 | 加到 research-deep 输出格式 | 待做 |
| claude-code 官方 | hookify 自动分析对话模式生成 hook | 新建 hook-suggest skill | 待做 |
| claude-code 官方 | 多 agent 并行审查 | github-analyzer 已用 | 已做 |
| career-ops | mode 模块化 | 等 skill 复杂到需要拆分时再用 | 记住 |
| claude-mem | 渐进式 context disclosure | 优化记忆体系上下文注入 | 待评估 |

---

## 九、容易忽略的 8 个关键问题

### 9.1 子 Agent 失败了怎么办

当前 SOP 假设子 agent 一定会成功返回。实际上子 agent 可能：超时、返回空结果、格式错误、跑偏。

**解决方案：在主 agent 的监督逻辑中加"容错层"**

```markdown
## 第三步：合并结果

等所有子 Agent 返回后：

1. 检查每个子 agent 的返回是否完整
2. 如果某个子 agent 返回为空或明显不完整：
   - 不重新派发整个子 agent（太贵）
   - 用主 agent 自己的能力补上缺失部分（WebSearch 一搜）
   - 在报告中标注"[此部分信息由主 agent 补充，非并行调研结果]"
3. 如果关键信息（如项目定位、核心功能）完全缺失，才重新派发子 agent
```

### 9.2 Token 成本意识

每个子 agent 是一次完整的 API 调用，有自己的上下文窗口（默认 200k token）。3 个子 agent = 3 倍成本。

**控制成本的设计原则：**

| 原则 | 做法 |
|------|------|
| 子 agent 数量最小化 | 2 个够用不派 3 个，子 Agent 3 设条件触发 |
| Prompt 精简 | 子 agent prompt 只包含必要的上下文，不复制整篇 SKILL.md |
| 避免重复搜索 | 多个子 agent 不要搜同一个关键词，分工明确 |
| 结果写文件 | 子 agent 把结果写文件而不是返回长文本，主 agent 按需读取 |
| 用 Haiku 做简单任务 | 简单的格式化、验证用 haiku（便宜），复杂分析用 sonnet/opus |

### 9.3 Skill 管道模式（Skill Pipeline）

你的 research → research-deep → research-report 是一个管道：前一个的输出是后一个的输入。这不是巧合，是设计模式。

**管道模式的三要素：**

1. **标准化接口**：上游 skill 的输出格式必须让下游 skill 能直接消费
   - research 输出 `outline.yaml` + `fields.yaml`
   - research-deep 读取这两个文件，逐项调研输出 JSON
   - research-report 读取 JSON，汇总成 markdown

2. **独立可用**：每个环节的 skill 也能单独使用
   - research 可以独立产出 outline
   - research-deep 也可以用别人写好的 outline
   - research-report 也可以汇总任意来源的 JSON

3. **文件作为管道协议**：不要用"上下文传递"，用文件传递
   - 文件不会丢失、可以断点续传、可以人工编辑

```
research (outline.yaml) → research-deep (JSON files) → research-report (markdown)
```

**怎么在设计新 skill 时应用**：如果你的 skill 需要"分阶段"处理，每个阶段都设计成独立 skill，用文件做接口。

### 9.4 何时拆分、何时合并 Skill

| 信号 | 建议 | 案例 |
|------|------|------|
| SKILL.md 超过 500 行 | 拆！把辅助内容移到 references/ 或拆成独立 skill | hv-analysis 的 schema.json 移到 references/ |
| 同一个 skill 有明显不同阶段 | 拆成管道模式 | research → research-deep → research-report |
| 两个 skill 的 description 有重叠 | **先尝试划清边界**（用 MUST 不触发让位），边界真划不清才合并 | `idea-to-prompt` ↔ `brainstorming` ↔ `project-init` 三角形是边界划清的范例，而不是合并 |
| 用户经常连续触发两个 skill | 考虑合并成一个有子步骤的 skill | 如果用户总是先 research 再 research-deep，可以做一个 auto-research |
| skill 的某部分只偶尔用到 | 拆成可选触发 | github-analyzer 的子 Agent 3（技术分析）设条件触发 |
| 多个场景是同一类任务的变体 | 用内部分支，不各自独立 skill | 公众号封面图/PPT配图/小红书封面图 → 一个图片生成 skill，内部根据上下文走不同分支 |

**内部分支写法示例**：

```markdown
## 场景判断

触发后，根据用户上下文判断具体场景：

- 如果提到"公众号封面"或"封面图" → 走封面图分支（尺寸 900×383，文字居中）
- 如果提到"PPT配图"或"演示配图" → 走 PPT 分支（尺寸 1920×1080，留白多）
- 如果提到"小红书" → 走小红书分支（尺寸 1080×1440，大字标题）
- 其他情况 → 走通用生图分支
```

### 9.5 子 Agent 的 Model 选择

⚠️ **官方 Agent Skills spec 没有 `model` frontmatter 字段**——这是 SOP 早期写错的地方，已修正。
想给 skill 指定模型，只能塞 `metadata.model: opus`，但 Claude Code / 各客户端不一定识别，**当前不可靠**。

实际控制模型的方法：
- 主 agent：由 Claude Code 全局配置决定（用户在 settings 里选 opus/sonnet/haiku）
- 子 agent：通过 Agent tool 的 `model` 参数显式指定（仅在派发 subagent 时控制）

按任务类型推荐：

| 任务类型 | 推荐模型 | 原因 |
|----------|----------|------|
| 简单搜索汇总 | haiku | 快、便宜，搜索+摘要够用 |
| 结构化数据提取 | haiku | 提取 JSON 字段不需要强推理 |
| 多维度分析评估 | sonnet | 需要推理和权衡 |
| 写作、创意、长文 | sonnet/opus | 需要语言能力和深度思考 |
| 代码审查、安全审计 | opus | 需要最强推理能力 |

**SKILL.md 里怎么写**：不写死模型字段，而是在 Agent tool 调用模板里指定，如：

```markdown
启动 Agent 子任务（推荐 model: haiku，简单搜索汇总），prompt 如下：
...
```

让主 agent 在派发时按推荐选模型。

### 9.6 Description 冲突处理

当两个 skill 的 description 都可能被触发时，Claude 会困惑。解决方法：

1. **排除写法**：在 description 中明确写"不要用于 XX"（XX 是另一个 skill 的场景）
2. **优先级说明**：如果两个 skill 有重叠，在各自的 description 里写清楚边界
3. **定期审查**：每次新增 skill 后，检查和已有 skill 的 description 是否冲突

```
# github-analyzer 的排除
"不要用于纯代码审查、不要用于创建新项目、不要用于简单搜索查询"

# security-review 的排除
"不要用于评估外部开源项目（那是 github-analyzer 的事）"
```

### 9.7 SKILL.md 上下文开销

每个 skill 的 SKILL.md 会在会话中占据上下文窗口。你的 N 个 skill 不是每次都全部加载，但加载的每个都会消耗 token。

**优化原则：**

| 做法 | 效果 |
|------|------|
| SKILL.md 控制在 200 行以内（理想值） | 每个加载约 1-2k token |
| 详细的 prompt 模板放 references/ | 只在需要时 `Read` 加载 |
| 子 agent prompt 不要写在主 SKILL.md 里写完整版 | 写模板+变量占位，运行时拼接 |
| 避免大段示例 | 一个精简示例 > 三个冗长示例 |
| 不解释 Claude 已经知道的常识 | "用 WebSearch 搜索" 不需要解释 WebSearch 是什么 |

### 9.8 输出格式怎么让 Claude 真的遵守

写了输出模板但 Claude 不一定严格遵守。几个技巧：

1. **表格比列表更容易遵守** — Claude 对 markdown 表格格式的遵守度高于自由文本
2. **写"输出格式"而不是"输出模板"** — 用"严格按照以下格式输出"比"参考以下模板"有效
3. **给一个具体的 one-shot 示例** — 比描述格式更有效
4. **在最后加自检步骤** — "输出前检查：是否包含所有必需字段？格式是否一致？"
5. **用分隔符固定结构** — `---` 或 `## ` 作为区块边界，Claude 更容易保持结构

```markdown
# 差（Claude 经常跑偏）
输出一份分析报告，包含项目概览、评估、建议。

# 好（Claude 容易遵守）
严格按以下格式输出，不要增减任何章节：

## 一句话结论
（15字以内）

## 综合评分
| 维度 | 分数 |
|------|------|

## 建议
推荐/可选/不推荐 — 一句话原因

输出前自检：是否包含以上所有章节？每个评分是否都有数字？
```

### 9.9 Skill 总量控制

Skill 数量直接影响触发准确率。实验数据：

| 全局 Skill 数 | 触发准确率 | 体验 |
|--------------|-----------|------|
| < 20 | 90%+ | 几乎不会错触发 |
| 20-30 | 开始下降 | 偶尔错触 |
| > 30 | 明显下降 | 经常触发错误 skill |
| > 100 | ~20% | 严重误触 + 速度慢 + token 爆炸 |

**控制原则**：

1. **全局 skill 总数控制在 25 个以内**。超过这个数触发准确率明显下降。每次新建 skill 都先问自己：这个真的非全局不可吗？
2. **项目级 skill 不受此限制**——只在该项目目录下加载，不影响全局触发准确率
3. **定期清理**：每装一个新 skill，检查有没有可以合并或降级为项目级的旧 skill
4. **装了不用的 skill 立刻删**——它会占用渐进式披露的 slot，干扰其他 skill 的触发
5. **以"当前触发准确率"作为信号**：如果你发现 Claude 经常错触发或漏触发，先审查 skill 总数 + description 边界，不要急着加新 skill

**降级策略**（全局 → 项目级 → 删除）：

```
全局 skill（所有项目都用，< 25 个）
  ↓ 只在特定项目用
项目级 skill（.claude/skills/，按项目隔离）
  ↓ 超过一个月没用
删除
```

---

## 十、Skill 安装位置

```
~/.claude/skills/           ← 全局 skill，所有项目可用
├── my-skill/
│   └── SKILL.md
└── another-skill/
    ├── SKILL.md
    └── scripts/

项目目录/.claude/skills/    ← 项目级 skill，仅当前项目可用
├── project-skill/
│   └── SKILL.md
```

创建新 skill 后放在对应位置，新对话中 Claude 自动识别。也可用 `/skill-name` 手动触发。

---

## 十一、Skill 仓库管理 / git 工作流

你的 skill 用三层结构管理：

```
主版本    ~/your-skills-repo/{name}/  ← 日常主要修改位置 + git 工作树
   ├─ rsync → Codex 版       ~/.agents/skills/{name}/
   ├─ rsync → Claude Code 版 ~/.claude/skills/{name}/
   └─ git push → GitHub      github.com/shaocong1987-collab/shaocong-skills
```

### 11.1 三层关系

| 层 | 路径 | 谁来读 | 修改方式 |
|----|------|--------|---------|
| 主版本 / 仓库版 | `your-skills-repo/{name}/` | 人维护、git 版本管理、GitHub 发布源 | **主要在这里 Edit** |
| Codex 版 | `~/.agents/skills/{name}/` | Codex 当前会话 / 后续会话发现 skill | `rsync` 自仓库版 |
| Claude Code 版 | `~/.claude/skills/{name}/` | Claude Code 启动时加载 | `rsync` 自仓库版 |
| 公开仓库 | `shaocong-skills` GitHub | 别人 `curl` 安装 | `git push` 自仓库版 |

**同步原则**：
- 平时主要修改 `your-skills-repo/{name}/` 下的 `SKILL.md` 和 `README.md`。
- 用户说"更新这个 skill"、"把这个 skill 最终版同步"、"推一下这个 skill"时，默认要同时处理四件事：更新仓库版 → 同步 Codex 版 → 同步 Claude Code 版 → commit/push GitHub。
- `SKILL.md` 是 agent 真正执行的文件；功能规则、触发条件、工作流、边界变了必须改它。
- `README.md` 是给人看的 GitHub 文档；用途、功能、触发方式、安装方式或更新日志变了，也要一起改。
- 同步只覆盖目标 skill 目录，不批量覆盖整个 `~/.agents/skills` 或 `~/.claude/skills`。

**为什么不直接把 `~/.claude/skills/` 或 `~/.agents/skills/` git 化？** 因为里面有第三方装的 skill，不全是你写的，不该全部 push 公开。**只有你自己写的 skill 才进 `your-skills-repo` 仓库**。

### 11.2 修改 skill 的完整流程

```bash
# 1. 改主版本（仓库版）
cd "~/path/to/shaocong-skills/"
$EDITOR my-skill/SKILL.md

# 如果功能说明、安装方式、示例或更新日志变了，也改 README
$EDITOR my-skill/README.md

# 2. 同步到 Codex 和 Claude Code
rsync -a --delete my-skill/ "$HOME/.agents/skills/my-skill/"
rsync -a --delete my-skill/ "$HOME/.claude/skills/my-skill/"

# 3. 三处验证
uvx --from skills-ref agentskills validate ./my-skill
uvx --from skills-ref agentskills validate "$HOME/.agents/skills/my-skill"
uvx --from skills-ref agentskills validate "$HOME/.claude/skills/my-skill"

# 4. git push
git add my-skill/
git commit -m "update my-skill: <一句话说改了啥>"
git push origin main
```

### 11.3 新建 skill 的完整流程

```bash
# 1. 在主版本目录创建
cd "~/path/to/shaocong-skills/"
mkdir -p my-skill
$EDITOR my-skill/SKILL.md
$EDITOR my-skill/README.md

# 2. 测试 → 通过自检 checklist → 通过触发 audit

# 3. 同步到 Codex 和 Claude Code
rsync -a --delete my-skill/ "$HOME/.agents/skills/my-skill/"
rsync -a --delete my-skill/ "$HOME/.claude/skills/my-skill/"

# 4. 更新仓库 README（在 Skills 列表表格加一行）
$EDITOR "~/path/to/shaocong-skills/README.md"

# 5. 三处验证
uvx --from skills-ref agentskills validate ./my-skill
uvx --from skills-ref agentskills validate "$HOME/.agents/skills/my-skill"
uvx --from skills-ref agentskills validate "$HOME/.claude/skills/my-skill"

# 6. git push
git add my-skill/ README.md
git commit -m "add my-skill: <一句话说做啥的>"
git push origin main
```

### 11.4 公开仓库的目录约定

```
shaocong-skills/
├── README.md           ← 顶层 Skills 索引表 + 安装方式（每加新 skill 都更新）
├── .gitignore          ← 排除 Skill创作SOP.md（个人方法论，不公开）
├── my-skill-1/
│   ├── SKILL.md        ← Claude Code 直接读取的指令文件
│   └── README.md       ← 给人看的说明（简介/解决什么问题/功能/触发条件/报告示例/安装）
├── my-skill-2/
│   ├── SKILL.md
│   └── README.md
└── ...
```

**两份文件分工**：
- `SKILL.md` — 给 Claude Code 读，包含 YAML frontmatter + 完整指令
- `README.md` — 给人看，GitHub 上点开子目录直接渲染。结构推荐：
  1. **简介**：一句话说清楚是什么
  2. **解决什么问题**：用户真实痛点
  3. **功能**：核心能力清单
  4. **触发条件 / 不适用场景**：跟 SKILL.md description 一致但更易读
  5. **报告 / 输出示例**：let people see what they get
  6. **安装命令**：`curl` 一行装上
  7. **配合使用**：跟其他 skill 的协作关系

`.gitignore` 必含：
```
Skill创作SOP.md
.DS_Store
*.swp
```

> **关于本 SOP**：这是一份"怎么写 skill"的方法论活文档，产品(skill)背后的思考。你若 fork 本仓库自用，可把机器专属的绝对路径、私人风格偏好等改成自己的。

### 11.5 别人怎么装你的 skill

公开仓库 README 给出一行命令：

```bash
# 装单个 skill
mkdir -p ~/.claude/skills/my-skill
curl -o ~/.claude/skills/my-skill/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/my-skill/SKILL.md

# 装齐所有 skill
for s in idea-to-prompt project-init github-analyzer; do
  mkdir -p ~/.claude/skills/$s
  curl -o ~/.claude/skills/$s/SKILL.md \
    https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/$s/SKILL.md
done
```

### 11.6 隐私 / 公开化 checklist

每个 push 到公开仓库的 SKILL.md / README.md，发布前过一遍：

- [ ] 不含具体 skill 数量、绝对路径（`/Users/你的用户名/` 这种）
- [ ] 用户环境感知逻辑用相对路径 / `~` 写法（不硬编码用户名）
- [ ] 不含其他用户、客户、私人项目名
- [ ] README 维护好 Skills 索引表，新加的 skill 在表里有一行
- [ ] 安装命令在干净环境跑过一次，确保能装上

---

*本文档会随着实践持续更新。每个新 skill 创作完成后，在本文档末尾补充案例。*
