# [Fix] Antigravity IDE 1.19.x 更新后的 Shell Integration 隔离修复与并发卡死问题解析

> **关键词**: Antigravity IDE, Google Antigravity, 1.19.x update, checking command status, shell integration broken, PROMPT_COMMAND conflict, conda PROMPT_COMMAND, ROS setup.bash, Agent concurrency, multiple terminals hang, Extension Host RPC

## 问题背景与 1.19.x 版本更新导致的失效

在之前的修复（见 `2026-02-26-antigravity-shell-integration-fix.md`）中，我们通过在 `~/.bashrc` 末尾强行清空 `PROMPT_COMMAND` 的方式，解决了 Antigravity Agent 因 Conda/ROS 干扰而卡在 "checking command status" 的问题。

然而，在 Antigravity 升级到 **1.19.x (例如 1.19.6)** 后，该修复方案**失效了**，Agent 再次出现了卡死现象。

### 失效原因分析

在 1.18.x 版本中，IDE 注入 Shell Integration 钩子的时机或逻辑允许我们在 `.bashrc` 末尾进行清理。

但在 1.19.x 版本中，IDE 的底层注入机制发生了改变（可能是加载顺序变化或依赖了更严格的状态检测）。此时，原先在 `.bashrc` 末尾强行执行的 `PROMPT_COMMAND=""` 逻辑，**会“误杀”掉新版 IDE 自身刚刚设置好的正确钩子**。

简而言之：旧的“事后清理法”在新版本中变成了破坏者，导致 IDE 彻底丢失了监控终端命令状态的能力。

---

## 终极解决方案：环境隔离法 (Isolation)

为了彻底解决版本更新带来的机制变化问题，我们放弃“事后清理”，改为采用**“源头隔离”**的策略。

思路是：当检测到当前终端是由 Agent 启动的（带有 `ANTIGRAVITY_AGENT=1` 环境变量），我们就**直接跳过**所有可能污染 `PROMPT_COMMAND` 的初始化脚本（如 Conda、ROS、NVM 等）。

### 修复步骤

编辑 `~/.bashrc` 文件，将所有可能产生冲突的初始化代码块，用 `if [ -z "${ANTIGRAVITY_AGENT:-}" ]; then ... fi` 包裹起来。

修改后的 `.bashrc` 核心部分应该类似这样：

```bash
# ====================================================================
# [Fix] 只在非 Antigravity Agent 环境下加载可能产生冲突的初始化脚本
# 防止 Agent 终端被污染，确保 Shell Integration 钩子完美注入
# ====================================================================
if [ -z "${ANTIGRAVITY_AGENT:-}" ]; then

    # >>> fishros initialize >>>
    source /opt/ros/humble/setup.bash
    # <<< fishros initialize <<<

    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

    # >>> conda initialize >>>
    # !! Contents within this block are managed by 'conda init' !!
    __conda_setup="$('/home/lyy/miniconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
    if [ $? -eq 0 ]; then
        eval "$__conda_setup"
    else
        if [ -f "/home/lyy/miniconda3/etc/profile.d/conda.sh" ]; then
            . "/home/lyy/miniconda3/etc/profile.d/conda.sh"
        else
            export PATH="/home/lyy/miniconda3/bin:$PATH"
        fi
    fi
    unset __conda_setup
    # <<< conda initialize <<<

fi
```

### 为什么隔离法能一劳永逸？

无论未来 Antigravity 的底层通信钩子机制如何升级，只要它还需要一个纯净的 Bash 终端，这套从源头阻止污染物加载的隔离方案就永远有效。

**注意**：这会导致 Agent 启动的隐藏终端内**默认没有**激活 Conda 或 ROS 环境。如果 Agent 需要执行特定环境的命令，必须在 Prompt 中明确要求它使用**绝对路径**（例如：`/home/lyy/miniconda3/envs/myenv/bin/python script.py`）。

---

## 延伸探讨：为什么多个 Agent 终端并发会卡死？

在排查过程中，我们还确认了一个目前基于 VS Code 架构的 AI IDE（包括 Antigravity、Cursor 等）的普遍痛点：**整个 IDE 的 Agent 通常只能稳定控制一个终端进程。开启多个终端让 Agent 并发执行任务，极易导致状态混乱和集体卡死。**

### 并发卡死的底层架构瓶颈

1. **RPC 通信管道的“单点阻塞”与串台 (Race Condition)**
   Agent 所在的扩展进程（Extension Host）与具体的终端面板之间通过 RPC 管道通信。在多终端高并发输出或同时等待结束信号（`\e]633;D;0\a`）时，全局单例的终端命令管理器如果状态机处理不够健壮，很容易发生消息串台。Agent A 可能会错误地捕获终端 B 的结束钩子。

2. **状态机崩溃**
   一旦因为并发回调导致 `TerminalShellCommandManager` 等核心类的状态机逻辑崩溃，所有正在等待 "checking command status" 的 Agent 任务就会全部陷入永久僵死。

3. **大模型上下文污染**
   不同终端的数据流在组装传递给大模型的 Context 时，如果隔离不严，大模型会看到混合的输出日志，导致后续生成的命令指令完全错乱。

### 避免并发卡死的最佳实践

1. **坚持“串行工作流” (一事一结)**：
   不要在同一个对话中要求 Agent “同时在终端 1 启动服务端，在终端 2 启动客户端”。应先让 Agent 完成并确认服务端启动后，再让它执行客户端命令。

2. **分离长耗时任务**：
   对于需要运行很久的任务（如编译、训练），不要让 Agent 一直“等待输出”。在 Prompt 中明确要求：*“请将该命令放到后台执行（如使用 `nohup` 或 `&`），不要等待它执行完毕，确保启动成功即可。”*

3. **终极恢复手段 (重启扩展宿主)**：
   如果发现终端状态已经彻底混乱（Agent 一直卡在 checking），最快的恢复方法是按下 `Ctrl + Shift + P`，执行 **`Developer: Restart Extension Host`**，强制重置所有卡死的 RPC 管道状态。
