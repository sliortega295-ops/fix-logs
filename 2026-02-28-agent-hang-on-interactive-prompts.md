# [Fix] Antigravity Agent 执行 `gh pr create` 等交互式命令卡死挂起

> **关键词**: Antigravity IDE, Google Antigravity, agent stuck, gh pr create, GitHub CLI, interactive prompt hang, TTY, non-interactive mode, GH_PROMPT_DISABLED, Agent 死锁, 终端挂起

## 问题描述

**现象**:
在 Antigravity IDE 中让 Agent 帮忙创建 GitHub Pull Request（执行 `gh pr create` 命令），或者执行 `npm init`、未加 `-y` 的 `apt-get install` 时，Agent 会长时间卡住没有响应，一直处于 "checking command status" 或等待输出状态，挂起数分钟甚至无限期卡死。

**环境**:
- 工具：`gh` (GitHub CLI) 或其他带有交互式提示符的命令行工具
- 运行场景：Antigravity Agent 的后台终端

---

## 问题根因分析

### 1. 交互式提示符（Interactive Prompt）陷阱

CLI 工具（如 `gh`）在设计时默认是面向**人类用户**的交互式程序。

当你仅仅输入 `gh pr create` 时，它会发现你没有提供 PR 的标题（Title）和正文（Body）。于是，它会在底层执行以下操作之一：
1. **弹出 TTY 选择菜单**：要求你使用键盘的上下方向键选择将 PR 提交到哪个仓库（Fork 还是 Upstream）。
2. **打开文本编辑器**：调用系统默认的编辑器（如 `nano`、`vim`，或者弹出 VS Code 的一个新标签页），等待你手动输入 PR 详情、保存并关闭编辑器。

### 2. Agent 的“视觉盲区”与死锁

Antigravity 的 Agent 是一个 AI 实体，它：
- **没有物理键盘**：无法按下上下方向键或回车键来响应 TTY 菜单。
- **“看”不到外部编辑器弹窗**：如果 `gh` 试图打开一个文本编辑器，Agent 只能看到终端停止输出了，它不知道系统正在等待人类编辑文件。

**死锁形成**：
- `gh` 进程挂起，死死地等待来自标准输入（stdin）的按键或编辑器的关闭信号。
- Agent 死死地盯着终端的标准输出（stdout），等待命令执行完毕的结束符。
- 双方都在等对方行动，导致进程永久挂起。

---

## 解决方案

要让 Agent 顺利执行这类命令，必须强迫 CLI 工具进入**非交互模式（静默模式 / 批处理模式）**，通过一次性提供所有必要参数来避免它提问。

### 方案 1：完整提供所有必需参数（最常用）

不要给 `gh` 任何要求交互的机会。直接在命令中用 `--title` 和 `--body` 参数将内容写死。对于多行的正文，强烈建议 Agent 使用 `EOF`（Heredoc）语法：

**❌ 错误写法（Agent 必定卡死）：**
```bash
gh pr create
```

**✅ 正确写法（Agent 专用）：**
```bash
gh pr create --title "修复：更新了登录逻辑" --body "$(cat <<'EOF'
## 描述
修复了用户登录时可能导致崩溃的问题。

## 测试
- [x] 本地单元测试通过
EOF
)"
```

### 方案 2：使用 `--fill` 参数自动填充

如果不需要定制标题和正文，只想用当前分支的 Git commit 信息快速创建 PR，可以使用 `--fill` 参数。它会自动抓取 commit 信息并提交，不会弹窗询问。

**✅ 正确写法：**
```bash
gh pr create --fill
```

### 方案 3：终极防御 —— 注入 `GH_PROMPT_DISABLED=1`

为了彻底防止 `gh` 在任何不可预见的边缘情况（例如：遇到未知的远程仓库配置冲突）下弹出交互式菜单导致 Agent 卡死，可以设置专用的环境变量。

当设置 `GH_PROMPT_DISABLED=1` 时，如果 `gh` 发现缺少必需参数或需要用户抉择，它会**直接报错并退出（Exit Code > 0）**，并打印缺少什么，而不是挂起死等。

**这样 Agent 就能“看见”错误日志，分析报错原因，并修改命令重试，而不是傻等。**

**✅ 终极防御写法：**
```bash
GH_PROMPT_DISABLED=1 gh pr create --title "我的标题" --body "我的正文"
```

---

## 最佳实践：如何指导 Agent 避免挂起

在向 Agent 下达终端任务时，尤其是涉及可能产生交互的工具时，可以遵循以下原则指导它：

1. **“请使用静默模式”**：告诉 Agent “执行命令时请确保加上 `-y` 或相关静默参数（如 `apt-get install -y`），不要出现交互式提示符”。
2. **提前规划文本内容**：让 Agent 将复杂的描述信息通过 `cat <<EOF` 写入临时文件，或作为参数一次性传入，避免工具触发系统默认编辑器。
3. **设置非交互环境变量**：在执行自动化脚本时，Agent 可以在命令前加上类似 `DEBIAN_FRONTEND=noninteractive` 或 `GH_PROMPT_DISABLED=1` 的环境变量前缀。
