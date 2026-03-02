# [Fix] Clash Verge 旧版升级到 2.4.6 后 Service 无法启动 / TUN 虚拟网卡模式失败

> **关键词**: Clash Verge, clash-verge-rev, clash-verge-service, IPC path not ready, TUN mode, 虚拟网卡, systemd service, ExecStart path, socket permission, device or resource busy, os error 2, Failed to apply config, Linux

## 问题描述

**现象**: 从旧版 Clash Verge（1.x）升级到 2.4.6 后，出现以下一系列问题：

1. **IPC path not ready** — GUI 显示 "IPC path not ready"，无法连接后端服务
2. **Failed to apply config: Connection failed, I/O error: 没有那个文件或目录 (os error 2)** — 开启 TUN 虚拟网卡模式时报错
3. **TUN 开启后无法访问 Google / GitHub 等网站** — TUN 创建失败但 DNS 已被劫持，流量路由断裂

**影响版本**: Clash Verge Rev 2.4.6（从 1.x 升级，Linux deb 安装）

---

## 问题根因分析

### 问题 1: IPC path not ready

旧版 Clash Verge 1.x 的 systemd service 文件中，`ExecStart` 路径指向：

```
/usr/lib/clash-verge/resources/clash-verge-service
```

但新版 2.4.6 将二进制文件安装到了：

```
/usr/bin/clash-verge-service
```

升级 deb 包后，旧的 service 文件未被覆盖，导致 systemd 找不到可执行文件，服务反复崩溃重启（`status=203/EXEC`），GUI 自然无法连接 IPC。

### 问题 2: Socket 权限不可达

`clash-verge-service` 以 root 权限运行，创建的 IPC socket 目录 `/tmp/verge/` 默认权限为 `drwxrws--- root:root`，socket 文件为 `srw-rw---- root:root`。

普通用户运行的 GUI 完全没有访问权限，即使服务正常运行，GUI 仍然报 "IPC path not ready"。

### 问题 3: TUN 需要 Service 模式

TUN 虚拟网卡的创建需要 root 权限。如果 GUI 检测不到 service，会 fallback 到 **sidecar 模式**（以普通用户权限启动 mihomo 内核），此时：

- TUN 创建失败：`Start TUN listening error: configure tun interface: device or resource busy`
- Redir/TProxy 也失败：`operation not permitted`
- 但 DNS 劫持配置已经写入，导致流量黑洞

---

## 解决方案

### 修复 1: 更正 Service 文件的 ExecStart 路径

```bash
# 查看新版二进制文件实际位置
which clash-verge-service
# 输出: /usr/bin/clash-verge-service

# 查看当前 service 文件（旧路径）
cat /etc/systemd/system/clash-verge-service.service
# ExecStart=/usr/lib/clash-verge/resources/clash-verge-service  ← 旧版路径，文件不存在
```

### 修复 2: 重写 Service 文件（解决路径 + 权限问题）

最终稳定版本（`Group=lyy` 比 chmod 777 更安全可靠）：

```bash
sudo tee /etc/systemd/system/clash-verge-service.service << 'EOF'
[Unit]
Description=Clash Verge Service helps to launch Clash Core.
After=network-online.target nftables.service iptables.service

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /tmp/verge
ExecStartPre=/bin/chmod 777 /tmp/verge
ExecStart=/usr/bin/clash-verge-service
Group=lyy
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**关键改动说明**:

| 项 | 说明 |
|----|------|
| `ExecStart` | 更正为新版路径 `/usr/bin/clash-verge-service` |
| `ExecStartPre` (mkdir) | 确保 `/tmp/verge` 目录在服务启动前存在 |
| `ExecStartPre` (chmod) | 设置目录权限为 777，让普通用户 GUI 可以访问 |
| `Group=lyy` | **核心**：service 以 root 用户 + lyy 组运行，创建的 socket 归组 lyy，GUI 天然可读写，无需 chmod socket |
| `WantedBy=multi-user.target` | 确保开机自启 |

> **注意**: `Group=lyy` 需替换为你自己的用户名。可用 `id -gn` 查看。

### 修复 3: 创建 tmpfiles.d 规则（可选，持久化权限）

```bash
echo 'd /tmp/verge 0777 root root -' | sudo tee /etc/tmpfiles.d/clash-verge.conf
```

这确保每次系统启动时 `/tmp/verge` 目录自动以正确权限创建。

### 修复 4: 应用并验证

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 启用开机自启并立即启动
sudo systemctl enable clash-verge-service
sudo systemctl start clash-verge-service

# 验证服务状态
systemctl status clash-verge-service
# 期望: Active: active (running)

# 验证 socket 权限（Group 应为你的用户名）
ls -la /tmp/verge/
# 期望: drwxrws---  root lyy  ... /tmp/verge/
# 期望: srw-rw----  root lyy  ... clash-verge-service.sock

# 验证 TUN 接口
ip link show | grep Meta
# 期望: Meta: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP>

# 验证路由
ip route | grep Meta
# 期望: 198.18.0.0/30 dev Meta proto kernel scope link src 198.18.0.1

# 验证网络连通性
curl -s -o /dev/null -w "%{http_code}" https://www.google.com
# 期望: 200
```

### 修复 5: 清理旧版残留文件

```bash
# 删除旧版缓存目录
rm -rf ~/.cache/clash-verge

# 保留新版数据（不要删除）:
# ~/.local/share/io.github.clash-verge-rev.clash-verge-rev/  ← 新版配置和数据
# ~/.cache/io.github.clash-verge-rev.clash-verge-rev/        ← 新版缓存
```

---

## 排查思路总结

```
IPC path not ready
  └─ systemctl status clash-verge-service → status=203/EXEC
      └─ ExecStart 路径是旧版的 /usr/lib/clash-verge/resources/
          └─ 修正为 /usr/bin/clash-verge-service
              └─ 服务启动成功，但 GUI 仍连不上
                  └─ /tmp/verge/ 权限 700 root:root，普通用户无法访问
                      └─ Service 文件加 Group=lyy，socket 归组用户
                          └─ GUI 连接 service 成功
                              └─ TUN 模式正常，网络连通

TUN 开启后无法上网
  └─ service 日志: "Start TUN listening error: device or resource busy"
      └─ 旧 TUN 接口 Meta 残留未释放
          └─ sudo ip link delete Meta 清除残留
              └─ GUI 关闭再开启 TUN（单次操作，不要连点）
                  └─ TUN 正常，路由恢复，网络连通
```

---

## 常见故障速查表

| 现象 | 原因 | 快速修复 |
|------|------|----------|
| IPC path not ready | service 未启动或 socket 权限不对 | `sudo systemctl restart clash-verge-service` |
| Failed to apply config: os error 2 | GUI 在 sidecar 模式，未连上 service | GUI 设置 → 安装服务 |
| TUN: device or resource busy | 旧 TUN 接口残留 | `sudo ip link delete Meta` 后重新开启 TUN |
| TUN 开启后 DNS 超时 / 无法上网 | TUN 路由表未正确注入 | 同上，清除残留后重新开启 |
| GUI 显示 sidecar 模式 | service 启动时 socket 权限不对 | 检查 service 文件有 `Group=lyy` |

---

## 正常工作状态快照（2026-03-02）

以下是修复完成后 TUN 正常工作时各项配置的基准值，可作为日后排障对照：

### systemd Service 文件

```ini
# /etc/systemd/system/clash-verge-service.service
[Unit]
Description=Clash Verge Service helps to launch Clash Core.
After=network-online.target nftables.service iptables.service

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /tmp/verge
ExecStartPre=/bin/chmod 777 /tmp/verge
ExecStart=/usr/bin/clash-verge-service
Group=lyy
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### tmpfiles.d 规则

```
# /etc/tmpfiles.d/clash-verge.conf
d /tmp/verge 0777 root root -
```

### Socket 权限（正确状态）

```
/tmp/verge/
  drwxrws---  root lyy   clash-verge-service.sock
  srw-rw----  root lyy   clash-verge-service.sock
  srw-rw-rw-  root lyy   verge-mihomo.sock
```

> 关键：socket 归组 `lyy`，GUI 以 lyy 用户运行可直接读写。

### TUN 接口（正确状态）

```bash
$ ip link show Meta
Meta: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN

$ ip route | grep Meta
198.18.0.0/30 dev Meta proto kernel scope link src 198.18.0.1
```

### DNS（正确状态）

```bash
$ nslookup google.com
Server:         127.0.0.53
Name:   google.com
Address: 198.18.0.x    # fake-ip 模式，正常
```

### mihomo 核心配置（config.yaml）

```yaml
redir-port: 7895
tproxy-port: 7896
mixed-port: 7897
socks-port: 7898
port: 7899
log-level: info
allow-lan: false
mode: rule
external-controller: 127.0.0.1:9097
tun:
  stack: gvisor
  device: Meta
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
  - any:53
  strict-route: false
  mtu: 1500
ipv6: true
external-controller-unix: /tmp/verge/verge-mihomo.sock
unified-delay: true
```

### GUI 日志关键行（service 模式正常连接）

```
[Service] 正在尝试通过服务启动核心
[Service] 服务就绪，直接启动
[Service] 服务已运行且版本匹配，直接使用
[Service] 尝试使用现有服务启动核心
[Service] 服务成功启动核心
```

> 如果看到 `Starting core in sidecar mode` 说明没连上 service，需要在 GUI 里点击安装服务。

### 网络连通性基准

```
Google:  200  ~1.0s
GitHub:  301  ~0.8s  (正常跳转)
YouTube: 200  ~1.9s
```

---

## 适用场景

如果你遇到以下任何一种情况，本修复方案可能对你有帮助：

- Clash Verge 升级后 GUI 显示 "IPC path not ready"
- Clash Verge Service 不断重启，日志显示 `status=203/EXEC`
- `journalctl` 显示 `Failed to locate executable /usr/lib/clash-verge/resources/clash-verge-service: No such file or directory`
- TUN 虚拟网卡模式报 "Failed to apply config: Connection failed, I/O error: 没有那个文件或目录 (os error 2)"
- TUN 开启后 DNS 失效、无法上网
- Linux 下从 Clash Verge 1.x 升级到 2.x 后各种异常

---

## 环境信息

| 项目 | 详情 |
|------|------|
| OS | Ubuntu 24.04 (Linux 6.8.0, x86_64) |
| Clash Verge | 2.4.6 (clash-verge-rev) |
| 安装方式 | deb 包 (`clash-verge_2.4.6_amd64.deb`) |
| 旧版本 | clash-verge 1.6.6 |
| 内核 | verge-mihomo (mihomo) |
| Service 二进制 | `/usr/bin/clash-verge-service` |

---

## 参考资料

- Clash Verge Rev 仓库: https://github.com/clash-verge-rev/clash-verge-rev
- systemd service 文档: https://www.freedesktop.org/software/systemd/man/systemd.service.html
- tmpfiles.d 文档: https://www.freedesktop.org/software/systemd/man/tmpfiles.d.html
