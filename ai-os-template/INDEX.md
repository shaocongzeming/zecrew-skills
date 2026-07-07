# INDEX.md — AI-OS 入口路由

> 本文件是 Claude Code 和 Codex 的共同入口。
> 只做路由,不做知识库。
> 最后更新: YYYY-MM-DD

## 项目地图(通用示例,按需替换)

在这里维护你的项目清单,让 AI 知道每个项目的入口和触发场景。示例:

| 项目 | 路径 | 入口 | 触发场景 |
|---|---|---|---|
| 项目 A | `~/work/project-a/` | `项目开发日志.md` | 代码 / 数据处理 / 功能开发 |
| 项目 B | `~/work/project-b/` | `README.md` | 内容 / 策划 / 资料整理 |

## 最高治理规则

`shared-protocol.md` 修改权仅限你本人。

Claude / Codex 发现它有问题时:

1. 不直接修改。
2. 写入 `failures/_ai-use-failure-log.md`。
3. 周复盘时由你决定是否更新。

## 每次重要任务的读取顺序

1. 先读本文件。
2. 判断任务歧义度。
3. 默认不要继续全量读取 `~/.ai-os`。
4. 只有任务触发对应场景时,才按下表加载对应文件。
5. 优先读取项目级 `项目开发日志.md`;它通常比全局文件更省 token、更贴近当前任务。
6. 如果发现规则过期,记录到 failure log,不要擅自扩写规则。

## 渐进加载原则

- 轻任务: 只读 `INDEX.md`,直接回答或执行。
- 项目任务: 读 `INDEX.md` + 项目 `项目开发日志.md`。
- Codex 执行任务: 读 `INDEX.md` + `AGENTS.md` + 项目 `项目开发日志.md`;只有涉及身份、权限、合规时才补读 `USER.md` 或 `shared-protocol.md`。
- Claude 判断任务: 读 `INDEX.md` + `CLAUDE.md`;只有涉及你的背景时才补读 `USER.md`,涉及权限/合规/高风险时才补读 `shared-protocol.md`。
- Skill/复盘/失败日志: 只读对应文件,不要顺手读取全部记忆。

## 任务歧义度路由

| 任务类型 | 优先工具 | 加载文件 |
|---|---|---|
| 高歧义:战略、产品、商业判断、架构取舍 | Claude Code | 最小: `CLAUDE.md`; 按需: `USER.md`, `shared-protocol.md` |
| 低歧义:代码修改、文件处理、验证、浏览器/本地操作 | Codex | 最小: `AGENTS.md` + 项目 `项目开发日志.md`; 按需: 项目级 `AGENTS.md`, `shared-protocol.md`, `karpathy-guidelines.md` |
| 中歧义:方向明确但细节不清 | Claude 先收敛,Codex 后落地 | 先 `CLAUDE.md`; 落地时再读 `AGENTS.md` |
| 重复工作第 3 次出现 | 任一工具先识别 | `skills/_SKILL_TEMPLATE.md`, `skills/_SKILL_INDEX.md` |
| AI 用错、工具选错、规则冲突 | 任一工具 | `failures/_ai-use-failure-log.md` |
| 业务判断、客户沟通、选型、报价失败 | 任一工具 | `failures/_business-failure-log.md` |
| 每周复盘 | Claude 或 Codex | `reviews/_weekly-template.md`, 两份 failure log |

## 文件索引

| 文件 | 用途 | 触发词 | 优先级 |
|---|---|---|---|
| `USER.md` | 你是谁、长期身份、业务背景 | 我是谁、背景、身份、长期偏好 | 高 |
| `shared-protocol.md` | 两个模型共同遵守的底层协议 | 规则、权限、沟通、协作方式 | 高 |
| `CLAUDE.md` | Claude 高歧义任务协议 | 战略、判断、架构、取舍、澄清 | 高 |
| `AGENTS.md` | Codex 低歧义执行协议 | 修改、执行、验证、浏览器、自动化 | 高 |
| `skills/_SKILL_TEMPLATE.md` | 新建 Skill 的模板 | 重复三次、沉淀 SOP、造 Skill | 中 |
| `skills/_SKILL_INDEX.md` | Skill 治理表 | Skill 冲突、Skill 审计、触发词 | 中 |
| `failures/_ai-use-failure-log.md` | AI 使用失败日志 | AI 用错了、规则失效、模型误判 | 高 |
| `failures/_business-failure-log.md` | 业务失败日志 | 客户、选型、报价、交付、业务失败 | 中 |
| `reviews/_weekly-template.md` | 周复盘模板 | 复盘、周总结、规则更新、删规则 | 中 |
| `project-templates/project-development-log-template.md` | 项目开发日志模板 | 新项目、项目日志、继续开发、上下文入口 | 中 |
| `karpathy-guidelines.md` | Karpathy 编码行为准则(先思考、简洁优先、精准修改、目标驱动) | 编码规范、过度工程、diff 膨胀、LLM 编码陷阱 | 中 |

> 可选:如果你想拆分长期记忆,可在 `memories/` 下建立 `user-profile.md`(身份详情)、`preferences.md`(沟通/决策偏好)、`external-locations.md`(外部知识库/工具位置),再在本表登记。默认不预建,按需创建。

## 不要做的事

- 不要把长规则塞进 `INDEX.md`。
- 不要因为一次偶然失败就新增永久规则。
- 不要预先创建大量没有真实使用记录的 Skills。
- 不要让 Claude 和 Codex 私下互相覆盖共识协议。
- 不要把方法论当教条。它们是方法来源,不是日常负担。

## 当前系统状态(每周复盘更新)

- 活跃 Skill(过去 4 周用过):
- 候选 Skill(未达使用阈值):
- 待删 Skill(8 周未用):
- 本周失败回流的规则修改:
- 已过期待核验的记忆:

最后更新: YYYY-MM-DD
