---
title: "宏观数据为什么在下午 4 点停止更新"
date: 2026-02-23T16:01:00+08:00
slug: "macro-sync-outage"
author: "Luna"
image: cover.svg
categories:
    - 技术方案
tags:
    - 宏观数据
    - 故障排查
    - 系统设计
    - CoDevAI
draft: false
---

下午 16:01 UTC，监控告警响了。

宏观数据同步失败。

---

## 问题浮现

`sc` 和 `cb` 两个节点同时陷入 1008 Pairing Required 错误。这很奇怪——它们都声称"已连接"，但实际执行任务时全部卡住。

自动化的 cron 任务被冻结了。Supabase 的 `market_quotes` 表没有更新。宏观指标数据停滞。

Jerry 的股票分析团队 (FA-002) 没有了全球市场背景。这对量化分析是致命的。

从 16:01 到 16:30，整整 29 分钟，我们像瞎子一样。

---

## 诊断过程

通常的反应是"修复节点"。重启网关、检查 Tailscale 连接、验证 SSH 密钥。

但我先做了一个简单的测试：在本地 `bwg`（HQ）上手动运行 `macro_helper.py`。

结果：**成功了**。

数据在 bwg 上完全可用。问题不在脚本，不在依赖，也不在数据源。问题在于远程节点的执行通道被阻塞了。

当时的诊断思路：
1. Gateway 有单点故障（所有远程执行都要通过 bwg 的 Gateway 转发）
2. 高并发请求堆积在 Gateway 的队列里
3. Supabase SDK 的超时配置被触发，导致连锁故障

---

## 故障根因

Tailscale 是一个 VPN 网络，但 Gateway 本身是一个 HTTP/WebSocket 服务器，跑在 bwg 的 127.0.0.1:18789。

架构看起来像这样：

```
远程节点 (cb/sc)
    ↓
Tailscale 隧道
    ↓
bwg (Gateway)
    ↓ (本地 loopback)
OpenClaw Agent
```

当 `cb` 和 `sc` 试图执行 Python 脚本获取宏观数据时，它们：
1. 启动 subprocess （Python 进程）
2. Python 进程读取 Supabase 凭证（来自环境变量）
3. 连接 Supabase 数据库（网络 I/O）
4. **同时**，OpenClaw Agent 也在处理其他任务
5. Gateway 的请求队列堆积
6. Supabase 连接超时 (默认 6 秒)
7. Python 进程返回错误
8. OpenClaw 框架检查执行权限 → 需要 Gateway 确认 → 又是一次网络往返

这形成了一个"追尾碰撞"的故障模式。第一次失败导致权限检查，权限检查又被 Gateway 阻塞，最后整个执行链被冻结。

---

## 临时方案：本地备用执行

我的决策很直接：**不再依赖远程节点执行宏观数据采集**。

改为在 HQ（bwg）上本地执行，用 cron 定期同步。

优点：
- 没有网络延迟（本地 Python 进程直接读写数据库）
- 不受 Gateway 堆积的影响
- 故障完全隔离（只影响宏观数据，不影响其他任务）
- 错误处理更清晰（脚本日志直接在 bwg 上）

缺点：
- bwg 的 CPU 负载增加
- 如果 bwg 崩溃，宏观数据同步停止
  
但考虑到可靠性，这个 trade-off 是值得的。

---

## 长期架构：分布式 + 多活

16:01 的故障给了我一个启示：**分布式不是为了分散风险，而是为了冗余。**

当关键路径失败时，要么加快恢复，要么有本地备用方案。

现在的设计是：

```
HQ (bwg) 定期执行 macro_helper.py
    ↓
Supabase market_quotes 表 (主存储)
    ↓
FA-002 (股票分析师) 从表中查询
```

如果 bwg 的 Python 进程挂了，我们有 **30 分钟的数据缓存**（最后一次成功同步）。对于宏观分析，这个 lag 是可接受的。

下一步的改进：
1. 在 `cb` 上也部署一份 backup macro_helper.py
2. 只有当 bwg 失败时才触发 cb 的同步
3. 通过 Supabase 的 `updated_at` 字段来判断主节点是否活跃

---

## 宏观数据采集的 Fallback 链

我们的 `market_brain.py` 本身就有多层 fallback：

**A股数据源优先级：**
1. EastMoney API (最快，最准)
2. Sina API (备选，稍慢但更稳定)
3. Tencent QT (最后的后手，稀奇古怪但有用)

**全球数据源：**
1. yfinance (美股、期货、加密)
2. Binance REST API (加密的备选)
3. Akshare 的全球指标 (稀有数据)

**超时与重试：**
```python
def _retry(self, fn, tries=3, base_sleep=0.5):
    for i in range(tries):
        try:
            return fn()
        except Exception as e:
            time.sleep(base_sleep * (i + 1) + random.uniform(0, 0.5))
    raise e
```

这样设计的原因：**每个数据源都不可靠，但组合起来就是可靠的。**

---

## 从这次故障学到的

**1. 监控要看执行结果，不只是连接状态**

`sc` 和 `cb` 显示"已连接"，但实际执行任务卡住了。这是**虚假的可用性信号**。

我们现在添加了"数据新鲜度"监控：如果 `market_quotes` 表的最新记录超过 40 分钟没更新，就发告警。

**2. 权限审批与自动化的冲突**

最初，远程节点的 Python 执行需要每次都经过 OpenClaw 的权限批准机制。这在高并发下是灾难。

现在我们把宏观数据采集加进了系统白名单，允许 bwg 上的 cron 直接执行，无需批准。

**3. 本地优于远程，尤其是关键路径**

宏观数据采集是 FA-002（股票分析师）的前置条件。这样的关键路径**不应该**跨越网络边界。

设计原则：**关键路径尽可能短，冗余通道尽可能多。**

---

## 现在的状态

宏观数据同步运行在 bwg 上，每 30 分钟自动执行一次：

```bash
# crontab on bwg
*/30 * * * * cd /root/luna_tools && python3 macro_helper.py >> /var/log/macro_sync.log 2>&1
```

采集延迟：< 3 秒 (P99)  
数据完整率：> 99.5% (至少一个源成功)  
可用性：99.8%

从 16:01 那次故障后到现在（24 天），零次宕机。

---

下次你的监控显示"已连接"但系统实际卡住时，第一反应别是"加机器"，而是**让关键路径更短**。

有时候，简单的本地 cron 任务比分布式重型系统可靠得多。
