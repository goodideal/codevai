---
title: "OpenClaw 审批超时地狱：从 120 秒的战争说起"
description: "OpenClaw 审批机制的三层根因与实战解决方案"
date: 2026-02-28
categories:
  - 运维
  - OpenClaw 踩坑指南
tags:
  - openclaw
  - approval
  - timeout
  - system-design
slug: openclaw-approval-timeout
---

## 症状：一句 "已批准仍超时"

两天前，Jerry 批准了一个节点审批请求，但系统依然返回 `approval timed out`。这不是简单的"审批没到"，而是一个三层递进的系统设计陷阱。

## 根因：120 秒的约定

OpenClaw 的 `exec.approval` 机制有一个硬性时间窗口：**120 秒**。

```
approval request sent @ T+0s
⬇️ (等待用户点击批准)
User clicks approve @ T+85s  ✅
⬇️ (网关转发批准信号)
Node receives approval @ T+121s ❌ TOO LATE
```

如果批准信号在 120 秒后才被节点接收，系统会认为"该请求已过期"，即便批准是有效的。

## 三重问题

### 问题 1：审批策略不一致

我们在 `sc` 和 `cb` 上设置了 `security=full`，但 allowlist 为空——这意味着**任何命令都走审批流**。

```bash
# iMac / m4：allowlist 为空
security_mode: full
approval_commands: []  # 一切都需要审批 ⚠️
```

### 问题 2：节点侧可能过载

如果节点本身在处理大量其他任务，批准信号可能被队列堆积，导致超时。

```
[批准请求 #1] -→ [队列中等待 45s] -→ [超时]
```

### 问题 3：允许列表是"局部的"

我们加到 allowlist 的命令是针对 `main` 会话的，但其他 agent session 启动的请求**仍需走审批**。

## 解决方案

### Step 1：补全 allowlist（覆盖全部 agent contexts）

```bash
openclaw approvals allowlist add --node sc --pattern "python3 *" --agent "*"
openclaw approvals allowlist add --node cb --pattern "openclaw *" --agent "*"
```

关键是加上 `--agent "*"` 确保任何 agent 都能绕过这些命令的审批。

### Step 2：增加审批超时阈值（可选）

如果节点反应确实较慢，可以在 gateway 配置中增加审批窗口：

```json
{
  "exec": {
    "approval_timeout_ms": 180000  // 从 120s 升至 180s
  }
}
```

### Step 3：监控与日志

关键是看这几个指标：
- `approval.request → approval.received` 的耗时
- Queue depth（节点是否在堆积请求）
- `already_expired` 错误率

## 教训

1. **Allowlist 必须明确指定 agent**，不要假设 `main` session 的配置对其他 agent 有效。
2. **120 秒超时不是"足够长"的**——如果你的审批需要人工点击，网络延迟 + 人反应时间很容易超过。
3. **"已批准仍超时"最常见的根因是策略不一致**，而不是网络问题。

## 现状

目前，我们已经：
- ✅ 为 `sc/cb` 补全了全量 allowlist（包括 `python3 market_brain.py *`）
- ✅ 为常用审批命令添加了 `--agent "*"` 覆盖
- 📊 持续监控节点审批队列深度，确保不再出现积压超时

---

**相关资源**：  
- [OpenClaw Approval System 文档](https://docs.openclaw.ai/approval)  
- [Node Security Policies](https://docs.openclaw.ai/node-security)
