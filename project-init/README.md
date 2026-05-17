# project-init

## 简介

新项目「上下文入口」搭建 Skill。基于 ai-os 落地实践设计。

核心理念:**不是建文件夹,是建上下文入口**。日志是 source of truth,目录从日志的「项目目录地图」拍出来。

## 解决什么问题

开始新项目时经常面临:
- 不知道目录结构怎么搭
- 项目跑到一半,fresh agent(Claude / Codex / 新会话)进来后不知道项目目标、下一步、怎么验证
- 多个项目编号混乱
- 项目级配置文件(CLAUDE.md / AGENTS.md / README)写不写、写多少、放哪、引用哪些全局规则

这个 skill 一次搞定:产物是一份 fresh agent 读完就能直接干活的「项目开发日志.md」,加上最小必要的目录骨架。

## 功能

### 三种模式

- **新建** — 从零开始(默认)
- **补建** — 已在项目里,补缺失的文件(CLAUDE.md / AGENTS.md / README / .gitignore)
- **刷新** — 已有日志但 > 30 天未更新,引导补「当前阶段」

### 四类项目骨架

跟 idea-to-prompt 对齐:

- **开发类**:`docs/ src/ data/ _sandbox/ 99_归档/`
- **长内容类**:`素材/ 草稿/ 成品/ 参考/ 99_归档/`
- **视觉素材类**:`prompts/ 素材/ 成片/ 参考/ 99_归档/`(prompts 是核心资产)
- **策划类**:`方案/ 资源/ 进度/ 参考/ 99_归档/`

### 关键设计

1. **接力 idea-to-prompt** — 启动时自动扫 cwd 和 `~/Desktop` 的 `Prompt-*.md`,有就读、只问差量
2. **官方模板复用** — 项目开发日志.md 直接 cp 自 ai-os 标准模板填空,不二次创作
3. **opt-in 补充文件** — CLAUDE.md / AGENTS.md / README / .gitignore 每个都问用户要不要,不默认全建
4. **协议文件 ≤ 20 行** — 只做"绝对路径指向全局协议 + 加载预算声明 + 项目级覆盖项"
5. **冷启动模拟当门槛** — 自己扮演 fresh agent 只读日志答三问(做什么/下一步/怎么验证),任一答不上不交付
6. **编号自动管理** — 扫描已有项目编号,排除 00/99,提示空位补建

## 触发条件

- 「开始新项目」、「新建项目」、「搭项目骨架」
- 「按这个 prompt 建项目」、「按 prompt 搭骨架」
- **自动接力**:cwd 或 `~/Desktop` 有 `Prompt-*.md` + cwd 不在项目里
- **自动识别补建/刷新**:cwd 已在项目根 + 「日志旧了」、「还缺个 readme」、「补一下」、「看下日志」

## 不适用场景

- 已有项目深度整理(用 neat-freak)
- 深度需求分析(用 idea-to-prompt)
- 代码实现(用 writing-plans)
- 内置 `/init`(只建单个 CLAUDE.md;本 skill 建整套骨架)

## 输出示例

```
05_家庭记账小程序/
├── 项目开发日志.md       ← 唯一上下文入口,fresh agent 读完即可上手
├── docs/                # 设计文档、spec
├── src/                 # 源代码
├── data/                # 数据/配置
├── _sandbox/            # 实验区(> 30 天可清理)
└── 99_归档/             # 历史版本
```

「项目开发日志.md」结构(基于 ai-os 官方模板):

```
- 智能体唯一启动入口(可粘贴给新会话的指令)
- 项目目录地图(每个路径的职责)
- 当前项目原则(目标/阶段/业务底线/技术底线/不做什么)
- 当前功能状态(已完成/部分完成/未开始/已知问题)
- 常用文件(入口、业务规则、数据、验证)
- 开发和验证命令
- 开发记录(YYYY-MM-DD + 改了什么 + 验证方式)
```

## 安装

```bash
mkdir -p ~/.claude/skills/project-init
curl -o ~/.claude/skills/project-init/SKILL.md \
  https://raw.githubusercontent.com/shaocong1987-collab/shaocong-skills/main/project-init/SKILL.md
```

## 配合使用

- **上游 `idea-to-prompt`** — 先聊明白产出 `Prompt-*.md`,接给 project-init 自动建骨架(检测 + 只问差量)
- **跟 `brainstorming` 互补** — brainstorming 管 git 化的软件开发(spec → writing-plans 流水线);本 skill 管所有四类项目的上下文骨架,不假设 git
- **跟内置 `/init` 区分** — `/init` 只初始化单个 CLAUDE.md;本 skill 建整套项目骨架 + 上下文入口
