# AI-OS 模板

> 一套让 Claude Code 和 OpenAI Codex 长期协作的"个人 AI 操作系统"骨架。
> 这是脱敏后的开源模板,不含任何真实个人身份或商业信息。

## 这是什么

AI-OS 不是某一个项目的规则,而是一套**机制/骨架**:用一个 `INDEX.md` 做路由、用渐进加载规则控制上下文成本、用一份共同协议约束 Claude 和 Codex 的行为,再配上 Skill 治理、失败复盘和项目日志模板。

目标:把你在工作、创业、写代码、知识管理和个人事务里的**重复劳动**,沉淀成一套可持续迭代的系统,而不是每次都重新交代上下文。

追求可复用、可验证、可迭代,不追求模型表现好看。

## 给谁用

- 长期同时使用 Claude Code 和 Codex(或任意两个 AI 编码/助理工具)的人。
- 想让 AI 记住自己的身份、偏好和协作规则,而不是每次从零解释的人。
- 想把"重复 3 次的动作"系统性沉淀成 Skill / 脚本 / 模板的人。

## 核心思想:按"任务歧义度"分工

Claude 和 Codex 不应被简单分成"大脑"和"执行"。更稳的分工是按任务的**歧义度**:

| 任务类型 | 首选工具 | 原因 |
|---|---|---|
| 高歧义 | Claude Code | 需求澄清、战略取舍、商业判断、架构辩论 |
| 低歧义 | Codex | 文件修改、批量处理、验证、浏览器操作、自动化 |
| 中歧义 | Claude 先收敛,Codex 再落地 | 先减少不确定性,再执行验证 |

`CLAUDE.md` 承载高歧义协议,`AGENTS.md` 承载低歧义协议,`shared-protocol.md` 是两者共同遵守的底层规则。

## 文件结构

```text
ai-os-template/
├── README.md                    # 本文件
├── INDEX.md                     # 入口路由:渐进加载 + 歧义路由 + 文件索引
├── shared-protocol.md           # Claude 与 Codex 共同协议(精华)
├── CLAUDE.md                    # Claude 高歧义任务协议
├── AGENTS.md                    # Codex 低歧义执行协议
├── USER.template.md             # 用户画像模板(填入自己的信息,勿开源真实版)
├── karpathy-guidelines.md       # Karpathy 编码行为准则(英文原样,附来源)
├── project-templates/
│   └── project-development-log-template.md
├── skills/
│   ├── _SKILL_TEMPLATE.md       # 新建 Skill 的模板
│   └── _SKILL_INDEX.template.md # 空的 Skill 治理表(表头 + 门槛规则)
├── failures/
│   ├── _ai-use-failure-log.md   # AI 使用失败日志(仅模板)
│   └── _business-failure-log.md # 业务失败日志(仅模板)
└── reviews/
    └── _weekly-template.md      # 周复盘模板
```

## 怎么装

1. 把本目录内容复制到 `~/.ai-os/`。
2. 把 `USER.template.md` 复制成 `~/.ai-os/USER.md`,填入你自己的信息。
   **重要:** 填好的 `USER.md` 属于个人隐私,若你把 ai-os 开源,记得把它加进 `.gitignore`。
3. 在 `~/.claude/CLAUDE.md` 和 `~/.codex/AGENTS.md` 里放一段轻量入口,要求各自先读 `~/.ai-os/INDEX.md`。这两个全局入口只做路由,不放全部内容。
4. 以后开启重要任务时,对 Claude 或 Codex 说:

   ```text
   先读 ~/.ai-os/INDEX.md,再按任务需要加载相关协议。
   ```

   Claude 应自动偏向读 `~/.ai-os/CLAUDE.md`,Codex 应自动偏向读 `~/.ai-os/AGENTS.md`。

## 演化机制(可选,但建议开启)

- **失败复盘**:AI 用错了写 `failures/_ai-use-failure-log.md`,业务失败写 `failures/_business-failure-log.md`。只记真实失败,不要为系统完整而编。
- **周复盘**:每周用 `reviews/_weekly-template.md` 让系统进化一次——少问、真改、敢删。
- **Skill 治理**:真实重复 3 次或真实失败反复出现,才用 `skills/_SKILL_TEMPLATE.md` 沉淀成 Skill,并登记进 `_SKILL_INDEX.md`。不要预先造一堆空 Skill。
- **修改权**:`shared-protocol.md` 建议只由你本人修改。AI 发现规则有问题时,不直接改,而是写入 failure log,周复盘时你来决定。

## 归属

这套 AI-OS 模板由 shaocong 从个人自用系统抽取脱敏而来,参考了 Claude + Codex 协作实践与 Karpathy 编码准则([github.com/multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills))。
