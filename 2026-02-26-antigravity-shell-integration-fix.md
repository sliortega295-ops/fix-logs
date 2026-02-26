# Antigravity Shell Integration 修复日志

**日期**: 2026-02-26
**问题**: Antigravity IDE 在执行命令时卡在 "check command status"，即使命令已经有输出也无法检测到命令完成。

---

## 问题分析

Antigravity（基于 VS Code 的 IDE）使用 **Shell Integration** 机制来检测终端命令的开始和结束。该机制通过在 `PROMPT_COMMAND` 和 DEBUG trap 中注入转义序列（`\e]633;...`）来追踪命令状态。

### 根因

1. `.bashrc` 中的 **conda init** 和 **ROS setup.bash** 会修改 `PROMPT_COMMAND`，在 Antigravity agent 的 shell 集成设置好后覆盖了它的钩子函数（`__vsc_prompt_cmd`），导致命令完成信号（`\e]633;D`）无法发出。
2. Antigravity 用户设置中未显式启用 `terminal.integrated.shellIntegration.enabled`。

### 关键代码路径

- Shell Integration 脚本: `/usr/share/antigravity/resources/app/out/vs/workbench/contrib/terminal/common/scripts/shellIntegration-bash.sh`
- 核心扩展: `/usr/share/antigravity/resources/app/extensions/antigravity/dist/extension.js`
- `TerminalShellCommandManager` 类负责命令执行，通过 `shellIntegrationDetected` 标志判断是否可以检测命令完成
- `getTerminalShellIntegration()` 有 10 秒超时等待 shell 集成就绪

---

## 修复内容

### 修复 1: 修改 `~/.bashrc`

在 conda 初始化块之后添加以下代码，确保 Antigravity agent 模式下 shell integration 钩子不被覆盖：

```bash
# Fix Antigravity agent shell integration
# conda/ROS may override PROMPT_COMMAND after shell integration setup.
# Clean up here so shellIntegration-bash.sh can set hooks properly.
if [ -n "${ANTIGRAVITY_AGENT:-}" ]; then
    PROMPT_COMMAND=""
    trap - DEBUG 2>/dev/null
    unset bash_preexec_imported 2>/dev/null
fi
```

**原理**: 当 Antigravity agent 启动终端时，`ANTIGRAVITY_AGENT` 环境变量会被设置。此代码在 conda/ROS 初始化之后清理被覆盖的 `PROMPT_COMMAND` 和 DEBUG trap，让 `shellIntegration-bash.sh` 能正确接管。

### 修复 2: Antigravity 用户设置

在 `~/.config/Antigravity/User/settings.json` 中添加：

```json
"terminal.integrated.shellIntegration.enabled": true
```

---

## 环境信息

- **OS**: Ubuntu (Linux 6.8.0-101-generic)
- **Shell**: /bin/bash
- **Antigravity 版本**: 1.18.4-1771638098
- **Antigravity Tools 版本**: 4.1.22
- **.bashrc 中可能冲突的组件**: conda init, ROS humble setup.bash, nvm
