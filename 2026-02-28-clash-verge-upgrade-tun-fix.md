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
Restart=always
RestartSec=5
UMask=0000

[Install]
WantedBy=multi-user.target
EOF
```

**关键改动说明**:

| 项 | 说明 |
|----|------|
| `ExecStart` | 更正为新版路径 `/usr/bin/clash-verge-service` |
| `ExecStartPre` (mkdir) | 确保 `/tmp/verge` 目录在服务启动前存在 |
| `ExecStartPre` (chmod) | 设置目录权限为 777，让普通用户 GUI 可以访问 IPC socket |
| `UMask=0000` | 让服务创建的 socket 文件默认权限为 666（所有用户可读写） |
| `WantedBy=multi-user.target` | 确保开机自启 |

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

# 验证 socket 权限
ls -la /tmp/verge/
# 期望: drwxrwsrwx ... /tmp/verge/
# 期望: srw-rw-rw- ... clash-verge-service.sock

# 验证 TUN 接口
ip link show | grep Meta
# 期望: Meta: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP>

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
                      └─ 添加 ExecStartPre chmod + UMask=0000
                          └─ GUI 连接 service 成功
                              └─ TUN 模式正常，网络连通
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
