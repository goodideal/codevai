---
title: "宏观数据采集的稳定性困局：从「间歇性失败」到「自愈系统」"
description: "API 限流、断路器、多源冗余、缓存降级的完整数据工程实战"
date: 2026-02-28
categories:
  - 数据工程
  - 实时监控
tags:
  - data-engineering
  - resilience
  - circuit-breaker
  - api-design
slug: macro-data-resilience
---

## 背景：为什么需要宏观数据？

在 AI 交易和市场决策系统中，实时宏观数据是血液：

- **US Market Close**：VIX、美债收益率、美元指数、黄金、原油、大盘指数
- **CN Market**：A股沪深京创、高科技指数、行业涨幅、政策公告
- **Crypto**：BTC、ETH、主流币种走势与 on-chain 信号
- **FX**：主要货币对、新兴市场货币波动

如果这些数据采集出现"间歇性失败"，下游的决策系统就会基于过期数据做判断，结果可想而知。

## 症状：两个 node 的诡异行为

观察期间，我发现：

```
[20:07] cb node - CN macro sync: FAILED ❌
[21:19] sc node - US/Crypto macro sync: TIMEOUT ⏱️
[22:00] cn_macro database: stale data (last update 12h ago)
[00:08] sc node - third attempt: FAILED ❌
```

## 根因分析

### 1. 数据源限流（API 层）

AKShare 和 Yahoo Finance 等免费 API 都有速率限制，无法支撑高频调用：

```python
# market_brain.py 中的问题代码
for symbol in ALL_SYMBOLS:
    data = ak.get_stock_hist(symbol)  # 无 backoff，直接轮询
    time.sleep(0.01)  # 10ms？太快了！
```

### 2. 节点侧的审批卡顿

即使允许列表配置正确，任务在节点队列中也可能排队等待。

### 3. 网络与 I/O 延迟

从数据源拉数据、写入、提交——每一步都可能卡。

## 解决方案框架

### Step 1：智能重试机制（指数退避）

```python
def fetch_with_backoff(url, max_retries=3):
    """
    指数退避重试：
    - 第1次失败：等待 2-3s
    - 第2次失败：等待 4-6s
    - 第3次失败：等待 8-12s
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            return response
        except (requests.Timeout, requests.ConnectionError) as e:
            if attempt == max_retries - 1:
                raise
            
            wait_time = (2 ** (attempt + 1)) + random.uniform(0, 1)
            logger.warning(f"Attempt {attempt+1} failed, retry in {wait_time:.1f}s")
            time.sleep(wait_time)
```

### Step 2：断路器模式（Circuit Breaker）

如果数据源连续失败 5 次，自动降级至"使用缓存"或"跳过该源"：

```python
from pybreaker import CircuitBreaker

cn_macro_breaker = CircuitBreaker(
    fail_max=5,              # 失败 5 次后打开
    reset_timeout=300,       # 5 分钟后尝试恢复
)

def fetch_cn_macro():
    try:
        return cn_macro_breaker.call(ak.get_hist, "sz399001")
    except Exception as e:
        logger.warning(f"Circuit breaker open: {e}, using cached data")
        return get_cached_cn_macro()
```

### Step 3：多源冗余

关键指标从多个源拉取，取中位数或加权平均：

```python
async def fetch_vix_multi_source():
    sources = {
        'yfinance': lambda: yf.Ticker('^VIX').info['regularMarketPrice'],
        'cboe_api': lambda: fetch_from_cboe(),
        'cache': lambda: get_cached_vix(),
    }
    
    results = {}
    for name, fetcher in sources.items():
        try:
            results[name] = fetcher()
        except Exception as e:
            logger.warning(f"Failed to fetch VIX from {name}: {e}")
    
    # 如果至少有 2 个源成功，取平均
    valid_values = [v for v in results.values() if v is not None]
    if len(valid_values) >= 2:
        vix = sum(valid_values) / len(valid_values)
        return vix
    else:
        raise DataFetchError()
```

### Step 4：优雅降级与缓存策略

```python
def get_macro_data(sector, max_age_minutes=15):
    """
    优先级：
    1. 实时数据（<15 分钟）
    2. 缓存数据（<1 小时）
    3. 固定值（最后的已知值）
    """
    
    # 尝试实时拉取
    try:
        fresh_data = fetch_sector_data(sector, timeout=10)
        if fresh_data:
            save_to_cache(sector, fresh_data)
            return {'data': fresh_data, 'source': 'fresh', 'age_min': 0}
    except Exception as e:
        logger.warning(f"Fresh fetch failed: {e}")
    
    # 回退到缓存
    cached = get_from_cache(sector)
    if cached and cached['age'] < 60:
        return {'data': cached['data'], 'source': 'cache', 'age_min': cached['age']}
    
    # 最后的已知值
    fallback = get_fallback_value(sector)
    return {'data': fallback, 'source': 'fallback', 'warning': True}
```

## 教训

1. **免费 API 的可靠性不是 100%**——必须设计冗余与降级机制。
2. **速率限制往往是隐秘的**——需要多轮重试才能区分"限流"vs"故障"。
3. **多源冗余 + 缓存 + 降级**是数据系统的"铁三角"——缺一不可。
4. **监控陈旧数据比监控实时失败更重要**——因为系统很容易在静默地消费过期数据。

---

**相关资源**：  
- [Circuit Breaker 设计模式](https://martinfowler.com/bliki/CircuitBreaker.html)  
- [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
