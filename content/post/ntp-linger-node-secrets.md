---
title: "NTP 与 Linger：那些让服务器「间歇失踪」的隐秘配置"
description: "systemd 服务间歇离线、时间漂移、节点健康检查的完整指南"
date: 2026-02-28
categories:
  - 运维
  - 系统诊断
tags:
  - systemd
  - ntp
  - linger
  - node-health
slug: ntp-linger-node-secrets
---

## 症状：节点说"离线"就离线

某天凌晨 4 点，监控告诉我 `data` 节点离线了。重启一次它就在线。再过一小时，又离线。这种"幽灵式离线"最让运维头疼。

## 根因 1：systemd Linger = no

如果 OpenClaw Node 是以 **user systemd service** 启动（而非 system service），就面临一个坑爹的设定：

```bash
# ❌ 有问题的配置
~/.config/systemd/user/openclaw-node.service
[Service]
User=root
Type=simple
ExecStart=/usr/bin/openclaw node

# 当用户登出会话时，所有 user services 都会被杀掉！
```

**为什么？** 因为 systemd 有一个参数叫 `Linger`：

```bash
# 查看当前 linger 状态
loginctl show-user root | grep Linger

# 输出示例
Linger=no  # ← 危险信号！
```

`Linger=no` 意味着：
1. 用户登出时，该用户的所有 cgroup（包括所有 service）会被彻底关闭
2. 即便 service 配置了 `Restart=always`，也只能等用户重新登录才能重启

### 修复：启用 Linger

```bash
# 为 root 用户启用 linger
loginctl enable-linger root

# 验证
loginctl show-user root | grep Linger
# Linger=yes  ✅

# 重启 service
systemctl --user restart openclaw-node
```

## 根因 2：NTP 未同步

另一个间接原因是**时间漂移**。如果节点的系统时间与 UTC 出现了显著偏差，某些审批、任务调度机制就会出现"时间验证失败"，导致任务被丢弃。

```bash
# 检查 NTP 同步状态
timedatectl status

# 示例输出
               Local time: Fri 2026-03-06 04:15:30 UTC
           Universal time: Fri 2026-03-06 04:15:30 UTC
                 RTC time: Fri 2026-03-06 04:15:29
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes  ✅
              NTP service: active  ✅
          RTC in local TZ: no
```

如果 `System clock synchronized: no`，说明 NTP 服务异常：

```bash
# Linux：检查 NTP 服务
systemctl status systemd-timesyncd

# macOS：检查时间设置
timedatectl status
```

### 修复：重启 NTP 服务

```bash
# Linux
sudo systemctl restart systemd-timesyncd
sudo timedatectl set-ntp true

# macOS（需要管理员）
sudo systemsetup -setnetworktimeserver time.apple.com
sudo systemsetup -setusingnetworktime on
```

## 时间漂移的三大后果

1. **JWT Token 验证失败** — 如果服务器时间比 token 签发时间**落后太多**，token 会被认为"尚未生效"。
2. **任务调度紊乱** — Cron 和定时任务基于系统时间，时间不准会导致任务被跳过或重复执行。
3. **节点心跳超时** — OpenClaw 节点定期向网关发送心跳，如果网关和节点的时间相差 >5 分钟，连接会被断掉。

## 完整的节点健康检查清单

```bash
#!/bin/bash
# node-health-check.sh

echo "=== System Time ==="
timedatectl status
echo ""

echo "=== NTP Status ==="
systemctl status systemd-timesyncd --no-pager
echo ""

echo "=== Linger Status ==="
loginctl show-user $(whoami) | grep Linger
echo ""

echo "=== OpenClaw Service Status ==="
systemctl --user status openclaw-node --no-pager
echo ""

echo "=== Disk Space ==="
df -h | grep -E '/$|/home'
echo ""

echo "=== Memory ==="
free -h
```

## 预防性维护

### 定期巡检脚本（每小时）

```bash
# /etc/cron.hourly/openclaw-healthcheck
#!/bin/bash
LOGFILE=/var/log/openclaw-health.log

{
  echo "[$(date)]"
  
  # 检查 NTP
  if ! timedatectl status | grep -q "synchronized: yes"; then
    echo "⚠️  NTP not synchronized!"
    systemctl restart systemd-timesyncd
  fi
  
  # 检查 Linger
  if ! loginctl show-user root | grep -q "Linger=yes"; then
    echo "⚠️  Linger not enabled!"
    loginctl enable-linger root
  fi
  
  echo "✓ All checks passed"
  echo ""
} >> $LOGFILE
```

## 教训

1. **User systemd service 必须启用 linger**，否则用户登出时服务会被杀掉。
2. **NTP 同步失败往往是沉默的**，不会有明显的错误信息，但会导致各种诡异的时间验证失败。
3. **定期的节点健康检查胜于事后诊断**——一个简单的 cron job 能避免 99% 的"幽灵离线"。

---

**相关资源**：  
- [systemd-logind 和 Linger](https://www.freedesktop.org/wiki/Software/systemd/UserSessionManagement/)  
- [timedatectl 使用手册](https://man7.org/linux/man-pages/man1/timedatectl.1.html)
