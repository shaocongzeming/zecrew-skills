# _SKILL_TEMPLATE.md — Skill 模板

> 用途: 创建 Claude/Codex 可复用 Skill 时使用。
> 原则: 一个 Skill 只解决一个清晰工作流。

````markdown
# Skill: <skill-name>

## 什么时候使用

当用户出现以下意图或触发词时使用:

- <触发词 1>
- <触发词 2>
- <典型场景>

不要在以下场景使用:

- <排除场景 1>
- <排除场景 2>

## 目标

这个 Skill 的目标是:

1. <目标 1>
2. <目标 2>
3. <目标 3>

## 输入

需要用户或上下文提供:

- <输入 1>
- <输入 2>
- <输入 3>

缺少关键输入时,只问最关键的 1 个问题。

## 工作流程

1. <步骤 1>
2. <步骤 2>
3. <步骤 3>
4. <验证或复核>

## 输出格式

```text
结论:

依据:

风险:

下一步:
```

## 权限边界

Green:

- <可自主执行>

Yellow:

- <需要简短说明后执行>

Red:

- <必须用户确认>

## 示例

用户输入:

```text
<示例输入>
```

理想输出:

```text
<示例输出>
```

## 失败记录

如果本 Skill 失败,记录到:

```text
~/.ai-os/failures/_ai-use-failure-log.md
```

如果失败来自业务判断,记录到:

```text
~/.ai-os/failures/_business-failure-log.md
```

## Skill 仓库同步

新写的 Skill 建议上传到你的 Skill 仓库:

```text
<你的 skills 仓库>
```

仓库目录结构示例:

```text
your-skills/
├── README.md
├── <skill名 A>/
│   ├── SKILL.md
│   └── README.md
├── <新skill名>/
│   └── SKILL.md
```

同步规则:

- 只上传你自己写的 Skill(SKILL.md + README.md),不上传第三方 Skill。
- 本文件(`_SKILL_TEMPLATE.md`)和 `_SKILL_INDEX.md` 只保存在本地 `~/.ai-os/skills/`,不上传仓库。
````
