# Anti-Hallucination Skill

[English](./README.en.md) | 简体中文

一个用于减少 AI Agent 幻觉的通用指令型 Skill,通过「提问补全背景 + 锚点句自检」双防线机制,让幻觉发生时信号可见。

## 这个 Skill 解决什么问题

AI Agent 的幻觉主要来自两个源头:

1. **背景缺失型幻觉** — 用户一句话下单,Agent 凭猜测填充空白,凭空捏造需求、文件名、函数名
2. **上下文遗失型幻觉** — 长对话中 Agent 忘记前文约束,输出与已确认信息冲突

本 Skill 针对这两类幻觉各设一道防线。

## 核心机制

### 第一道防线:开工前提问补全背景

按任务复杂度分级(L1/L2/L3),决定提问轮数上限,把「猜测」变成「确认」。

| 级别 | 判断特征 | 提问轮数上限 |
|---|---|---|
| L1 简单 | 单文件改动、改个值、需求明确无歧义 | 0 轮(直接开工) |
| L2 中等 | 多文件、有 1-2 个待澄清点 | 1-2 轮 |
| L3 复杂 | 跨模块、需求模糊、不可逆操作 | 3-5 轮 |

### 第二道防线:锚点句自检 + 人工复核

每次回复必须以固定标记包裹,让「上下文遗失」变得肉眼可见:

```
[CTX-LOCK]
<回复正文>
[CTX-VERIFIED]
```

- `[CTX-LOCK]` 开头锚点 = 「我已读取并锁定当前上下文」
- `[CTX-VERIFIED]` 结尾锚点 = 「本次输出已过自检清单」

**为什么是「自检 + 人工复核」而非纯 Agent 自检**:纯 Agent 自检不可靠——幻觉是生成时偏离,自检是同一个模型再生成一次,会一起坏。所以本 Skill 不追求「Agent 自检 100% 可靠」,而是追求「出错时信号可见」:即使 Agent 完全幻觉,用户也能凭锚点缺失一秒识别。

## 文件说明

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 中文版 Skill 主体定义(通用参考规范) |
| `SKILL.en.md` | 英文版 Skill 主体定义 |
| `platforms/trae/` | Trae 平台适配版(符合 Trae frontmatter 规范) |
| `platforms/claude/` | Claude 平台适配版(符合 Agent Skills 开放标准) |
| `platforms/generic/` | 通用 system prompt 版(任意 agent 可用) |
| `LICENSE` | MIT 协议 |

## 安装与使用

本 Skill 已适配三个平台,选择你使用的平台按说明安装:

### Trae

```bash
# 复制到 Trae skills 目录
cp -r platforms/trae/anti-hallucination ~/.trae-cn/skills/
```

验证:在 Trae 中输入任务,观察回复是否以 `[CTX-LOCK]` 开头、`[CTX-VERIFIED]` 结尾。

### Claude(Claude Code / Claude.ai)

```bash
# Claude Code:复制到 .claude/skills 目录
cp -r platforms/claude/anti-hallucination ~/.claude/skills/
```

Claude.ai 用户:将 `platforms/claude/anti-hallucination/SKILL.md` 的内容粘贴到 Project 的自定义指令中。

验证:输入"你有哪些 skill",应能看到 `anti-hallucination`;输入任务后观察锚点标记。

### 通用 system prompt(ChatGPT / Gemini / 本地 LLM 等)

将 `platforms/generic/system-prompt.md` 的全部内容粘贴到 agent 的 system prompt / 自定义指令 / 首条消息中。

验证:发送任意任务,观察回复是否带 `[CTX-LOCK]` / `[CTX-VERIFIED]` 双锚点。

## 技术栈

- **语言**:Markdown(100%)
- **可执行代码**:无
- **依赖**:无

本 Skill 是一份 Prompt/指令定义,不含任何编程语言代码。GitHub Linguist 会将仓库识别为 Markdown。

## 已知局限

- **模型兼容性差异**:不同模型对"强制开头/结尾输出固定标记"的遵守度不同,锚点机制在小模型或经过 RLHF 重度对齐的模型上可能被忽略
- **未做大规模效果验证**:本 Skill 的防幻觉效果未经大规模量化测试,建议在自己的使用场景中验证后再依赖
- **自检非绝对可靠**:agent 自检本身也可能一起幻觉,人工复核(看锚点是否缺失)是最终兜底

## 协议

[MIT](./LICENSE)

## 作者

艾坤 ([@forkseek](https://github.com/forkseek))
