# [Fix] Antigravity IDE Agent 执行命令卡在 "checking command status" / 命令输出无法检测

> **关键词**: Antigravity IDE, Google Antigravity, agent stuck, checking command status, terminal command hang, shell integration broken, PROMPT_COMMAND conflict, conda PROMPT_COMMAND, ROS setup.bash, VS Code shell integration, shellIntegration-bash.sh, TerminalShellCommandManager, command detection not working

## 问题描述

**现象**: 在 Antigravity IDE（Google 开发的基于 VS Code 的 AI IDE）中使用 Agent 功能执行终端命令时，Agent 会一直卡在 **"checking command status"** 或 **"等待命令输出"** 状态，即使命令早已执行完毕并产生了输出，Agent 也无法感知到命令已完成，导致整个工作流程卡死。

**具体表现**:
- Agent 让终端执行一条命令（如 `ls`、`pip install`、`git status` 等）
- 命令在终端中已经正常执行完毕，输出已经显示
- 但 Agent 仍然显示 "checking command status"，一直轮询等待
- 必须手动中断 Agent 才能继续操作
- 问题出现频率很高，几乎每次命令执行都会卡住

**影响版本**: Antigravity 1.18.x（基于 VS Code 的 IDE，使用 shellIntegration 机制）

---

## 问题根因分析

### 1. Antigravity 如何检测命令执行状态

Antigravity 使用 VS Code 的 **Shell Integration** 机制来追踪终端命令的生命周期。核心流程如下：

```
Shell Integration 工作原理:

1. Antigravity 通过 shellIntegration-bash.sh 脚本向 bash 注入钩子
2. 钩子修改 PROMPT_COMMAND 和 DEBUG trap
3. 命令开始前: 通过 DEBUG trap 触发 __vsc_preexec()，发送 \e]633;C\a (命令开始信号)
4. 命令结束后: 通过 PROMPT_COMMAND 触发 __vsc_precmd()，发送 \e]633;D;exitcode\a (命令完成信号)
5. TerminalShellCommandManager 类监听这些信号来判断命令状态
```

**关键文件**:
- Shell Integration 脚本: `/usr/share/antigravity/resources/app/out/vs/workbench/contrib/terminal/common/scripts/shellIntegration-bash.sh`
- 核心扩展: `/usr/share/antigravity/resources/app/extensions/antigravity/dist/extension.js`

**关键代码**（从 minified 的 extension.js 中提取）:
```javascript
// TerminalShellCommandManager 类
class _ {
    shellIntegrationDetected = false;

    // 等待 shell integration 就绪，超时 10 秒
    async checkTerminalShellSupport(timeout = 10000) {
        let terminal = this.managedTerminals.values().next().value;
        // ... 等待 shellIntegration 事件 ...
    }

    // 命令执行 - 流式 RPC
    async *executeCommand(signal, commandLine, cwd, terminalId) {
        // 依赖 shellIntegration 来检测命令的开始和结束
        // 如果 shellIntegration 未就绪，将无限等待...
    }
}
```

### 2. 为什么 Shell Integration 会失效

`shellIntegration-bash.sh` 通过设置 `PROMPT_COMMAND=__vsc_prompt_cmd` 和 `trap '__vsc_preexec_only' DEBUG` 来工作。

但在许多开发者的 `.bashrc` 中，以下工具会**在 shell integration 设置之后**再次修改 `PROMPT_COMMAND`：

| 工具 | 行为 | 影响 |
|------|------|------|
| **conda init** | 在 `.bashrc` 末尾注入代码，修改 `PROMPT_COMMAND` 来显示 conda 环境名 | 覆盖 `__vsc_prompt_cmd`，导致命令完成信号 `\e]633;D` 无法发出 |
| **ROS setup.bash** | `source /opt/ros/humble/setup.bash` 可能修改 `PROMPT_COMMAND` | 同上 |
| **bash-preexec** | 某些工具安装的 bash-preexec 会接管 DEBUG trap | 干扰 `__vsc_preexec` 的执行 |
| **starship / oh-my-posh** | 自定义 prompt 工具会完全替换 `PROMPT_COMMAND` | 同上 |

**加载顺序（问题所在）**:

```
1. Antigravity 启动终端
2. shellIntegration-bash.sh 注入 → 设置 VSCODE_INJECTION=1
3. shellIntegration-bash.sh 内部 source ~/.bashrc
4. ~/.bashrc 中 conda init / ROS setup.bash 修改 PROMPT_COMMAND  ← 覆盖了步骤2的钩子！
5. shellIntegration-bash.sh 继续执行，设置 PROMPT_COMMAND=__vsc_prompt_cmd
6. 但如果 Agent 模式下 ANTIGRAVITY_AGENT=1，执行顺序可能不同，导致最终 PROMPT_COMMAND 不是 __vsc_prompt_cmd
```

### 3. Agent 模式的特殊性

当 `ANTIGRAVITY_AGENT=1` 时，`shellIntegration-bash.sh` 会额外执行以下操作（第 65-85 行）：

```bash
if [ -n "${ANTIGRAVITY_AGENT:-}" ]; then
    trap - DEBUG              # 清除 DEBUG trap
    PROMPT_COMMAND=""          # 清空 PROMPT_COMMAND
    unset bash_preexec_imported
    shopt -u extdebug
    unset HISTCONTROL
    shopt -u lithist
    shopt -u cmdhist
fi
```

这段代码清理环境是为了确保 shell integration 能正确工作，但问题是 `.bashrc` 中的 conda/ROS 初始化发生在**之后**，又重新设置了 `PROMPT_COMMAND`，抵消了这里的清理效果。

---

## 解决方案

### 修复 1: 修改 `~/.bashrc`（核心修复）

在 `.bashrc` 文件的**最末尾**（特别是在 conda init 和 ROS setup 之后）添加以下代码：

```bash
# ==============================================================
# Fix: Antigravity IDE Agent shell integration
# Problem: conda/ROS overrides PROMPT_COMMAND after Antigravity's
#          shellIntegration-bash.sh sets it, causing agent to get
#          stuck on "checking command status".
# Solution: Clear PROMPT_COMMAND and DEBUG trap in agent mode,
#           so shellIntegration-bash.sh can properly set its hooks.
# ==============================================================
if [ -n "${ANTIGRAVITY_AGENT:-}" ]; then
    PROMPT_COMMAND=""
    trap - DEBUG 2>/dev/null
    unset bash_preexec_imported 2>/dev/null
fi
```

**为什么这样做有效**:
- `ANTIGRAVITY_AGENT` 环境变量仅在 Antigravity Agent 执行命令时设置，不影响正常终端使用
- 清空 `PROMPT_COMMAND` 后，`shellIntegration-bash.sh` 后续会将其设置为 `__vsc_prompt_cmd`
- 清除 DEBUG trap 后，`shellIntegration-bash.sh` 后续会设置为 `__vsc_preexec_only`
- 这样命令的开始（`\e]633;C`）和结束（`\e]633;D`）信号就能正确发出

**重要**: 这段代码必须放在 `.bashrc` 的**最末尾**，在所有 conda/ROS/nvm 初始化之后。

### 修复 2: 启用 Antigravity Shell Integration 设置

编辑 `~/.config/Antigravity/User/settings.json`，添加：

```json
{
    "terminal.integrated.shellIntegration.enabled": true
}
```

如果你的 `settings.json` 中已有其他设置，在最后一个设置项后面加逗号，然后添加这行。例如：

```json
{
    "workbench.colorTheme": "Tokyo Night",
    "terminal.integrated.shellIntegration.enabled": true
}
```

### 修复后验证

可以在终端中运行以下命令验证修复是否生效：

```bash
# 模拟 Agent 环境，检查 PROMPT_COMMAND 是否被正确清理
ANTIGRAVITY_AGENT=1 bash -c 'source ~/.bashrc; echo "PROMPT_COMMAND: [$PROMPT_COMMAND]"; echo "DEBUG trap: [$(trap -p DEBUG)]"'

# 期望输出:
# PROMPT_COMMAND: []
# DEBUG trap: []
```

修复完成后**重启 Antigravity IDE**，然后让 Agent 执行一条简单命令（如 `ls`）测试是否正常。

---

## 适用场景

如果你遇到以下任何一种情况，本修复方案可能对你有帮助：

- Antigravity Agent 执行命令时卡在 "checking command status"
- Antigravity Agent 无法检测到命令已经执行完毕
- Antigravity 终端中 shell integration 图标显示为禁用状态
- 使用 conda/miniconda/anaconda 且 Antigravity Agent 命令执行异常
- 使用 ROS/ROS2 且 Antigravity Agent 命令执行异常
- 使用 starship/oh-my-posh 等自定义 prompt 工具且 Antigravity Agent 异常
- 其他基于 VS Code 的 IDE（如 Cursor、Windsurf）遇到类似的 shell integration 问题

---

## 环境信息

| 项目 | 详情 |
|------|------|
| OS | Ubuntu 24.04 (Linux 6.8.0-101-generic) |
| Shell | /bin/bash |
| Antigravity 版本 | 1.18.4-1771638098 |
| Antigravity Tools 版本 | 4.1.22 |
| .bashrc 中冲突的组件 | conda init, ROS 2 Humble setup.bash, nvm |

---

## 参考资料

- VS Code Shell Integration 文档: https://code.visualstudio.com/docs/terminal/shell-integration
- VS Code shellIntegration-bash.sh 源码: https://github.com/microsoft/vscode/blob/main/src/vs/workbench/contrib/terminal/common/scripts/shellIntegration-bash.sh
- conda 与 PROMPT_COMMAND 的已知冲突: https://github.com/conda/conda/issues/11885
